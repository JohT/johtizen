---
layout: post
title:  "Testing all equals and hashCode methods"
date:   2020-03-08 20:00:00 +0100
categories: testing
tags: testing equals hashCode archunit equalsverifier
author: JohT
comments-issue-id: 16
---
One thing, that leads to many discussions, is how and if `equals` and `hashCode` methods should be tested.
Most of the time these methods get generated.
It doesn't seem right to test generated code.<br>
On the other hand, not testing these methods leads to poor test coverage statistics.
In rare cases, when these methods are not regenerated or edited manually, 
they might cause bugs, that are very hard to find.
In this post, I'll show you an effective way to test all `equals` and `hashCode` methods within one single test class.

## Libraries
- [![JUnit 5](https://img.shields.io/maven-central/v/org.junit.jupiter/junit-jupiter-api.svg?style=flat-square&logo=java&label=junit5)](https://github.com/junit-team/junit5) for unit testing, 
[![JUnit 4](https://img.shields.io/maven-central/v/junit/junit.svg?style=flat-square&logo=java&label=junit4)](https://github.com/junit-team/junit4) may be used as well

- [![equalsverifier](https://img.shields.io/maven-central/v/nl.jqno.equalsverifier/equalsverifier.svg?style=flat-square&logo=java&label=equalsverifier)](https://github.com/jqno/equalsverifier) for `equals` and `hashCode` testing on class level

- [![archunit](https://img.shields.io/maven-central/v/com.tngtech.archunit/archunit.svg?style=flat-square&logo=java&label=archunit)](https://github.com/TNG/ArchUnit) to specify architectural rules on package level

<details>
  <summary>maven `pom.xml` example (click to expand)</summary>
{% highlight xml %}
<dependency>
   <groupId>org.junit.jupiter</groupId>
   <artifactId>junit-jupiter-api</artifactId>
   <version>5.6.0</version>
   <scope>test</scope>
</dependency>
<dependency>
   <groupId>nl.jqno.equalsverifier</groupId>
   <artifactId>equalsverifier</artifactId>
   <version>3.1.12</version>
   <scope>test</scope>
</dependency>
<dependency>
   <groupId>com.tngtech.archunit</groupId>
   <artifactId>archunit-junit5</artifactId>
   <version>0.13.1</version>
   <scope>test</scope>
</dependency>
{% endhighlight %}
</details>
The source code for this example can be found 
[here](https://github.com/JohT/snippets/tree/master/archunit-equalsverifier-combined).

## JUnit Test Class

{% highlight java %}
public class EqualsHashcodeTest {
    private static JavaClasses classes = new ClassFileImporter()
            .withImportOption(ImportOption.Predefined.DO_NOT_INCLUDE_TESTS)
            .importPackages("io.github.joht.sample.basic"); /*1*/
    
    @Test
    public void testAllEqualsAndHashCodeMethods() {
        ConfiguredEqualsVerifier verifier = EqualsVerifier.configure()
		.usingGetClass()
        	.suppress(Warning.STRICT_HASHCODE); /*2*/
        
        ArchRuleDefinition.codeUnits().that()
                .haveName("hashCode").or().haveName("equals")
                .should(FulfillEqualsAndHashcodeContract.configuredBy(verifier)) /*3*/
                .check(classes);
    }
}
{% endhighlight %}

1. You first need no [import the classes] that should be analyzed by [ArchUnit]. Since it may take a while, it should only be done once per class (hence `static`).

2. [EqualsVerifier] can be configured if the default settings don't fit. In this particular case, the `hashCode` implementation can skip some fields that are compared inside the `equals` method, which still meets the contract.

3. [FulfillEqualsAndHashcodeContract] is the most important part.  
It connects [EqualsVerifier] to [ArchUnit] by extending the abstract class `ArchCondition`.

## Connecting [EqualsVerifier] to [ArchUnit]

{% highlight java %}
class FulfillEqualsAndHashcodeContract extends ArchCondition<JavaCodeUnit> {

    private final ConfiguredEqualsVerifier verifier;

    public static final ArchCondition<JavaCodeUnit> configuredBy(ConfiguredEqualsVerifier verifier) {
        return new FulfillEqualsAndHashcodeContract(verifier);
    }

    private FulfillEqualsAndHashcodeContract(ConfiguredEqualsVerifier verifier) {
        super("fulfills the equals and hashCode contract");
        this.verifier = verifier;
    }

    @Override
    public void check(JavaCodeUnit codeUnit, ConditionEvents events) {
        Class<?> classToTest = classForName(codeUnit.getOwner().getName());
        EqualsVerifierReport report = verifier.forClass(classToTest).report(); /*1*/
        events.add(eventFor(report, codeUnit.getOwner())); /*2*/
    }

    private static SimpleConditionEvent eventFor(EqualsVerifierReport report, JavaClass owner) {
        return new SimpleConditionEvent(owner, report.isSuccessful(), report.getMessage());
    }
    
    private static Class<?> classForName(String classname) {
        try {
            return Class.forName(classname);
        } catch (ClassNotFoundException e) {
            throw new IllegalStateException(e);
        }
    }
}
{% endhighlight %}

1. The exchangeable `ConfiguredEqualsVerifier` is used to generate the report for `equals` and `hashCode` methods. This is done for the `JavaCodeUnit`, that is currently selected by [ArchUnit].

2. The report of the [EqualsVerifier] is converted to a `SimpleConditionEvent` and added to the other events. If all checks are successful, the test will pass. If one of the 
`CodeUnits` doesn't fulfill the contract, it will be listed in the message of the failed test.

## Upgrading a test to a specification

There is a significant difference in the purpose for which the test is written for.
Finding bugs is the most obvious one.
Documenting, what the code should do and how it's expected to behave 
takes it to the next level. It can now be seen as a `specification`.
Have a look at [behaviour-driven development], if you want to find out more about it.

## Specification using [EqualsVerifier] and [ArchUnit]

{% highlight java %}
public class EqualsHashcodeSpecificationTest {

    private static JavaClasses classes = new ClassFileImporter()
            .withImportOption(ImportOption.Predefined.DO_NOT_INCLUDE_TESTS)
            .importPackages("io.github.joht.sample");

    @Test
    @DisplayName("entities are considered equal, if their id are equal") /* 1 */
    public void entitiesAreConsideredEqualIfTheirIdAreEqual() {
        ConfiguredEqualsVerifier verifier = EqualsVerifier.configure()
                .usingGetClass()
                .suppress(Warning.ALL_FIELDS_SHOULD_BE_USED);

        ArchRuleDefinition.methods().that()
                .haveName("hashCode").or().haveName("equals")
                .and().areDeclaredInClassesThat().haveSimpleNameContaining("Entity") /* 2 */
                .should(FulfillEqualsAndHashcodeContract.configuredBy(verifier)) /* 3 */
                .check(classes);
    }

    @Test
    @DisplayName("value objects are only considered equal, if all of their fields are equal")
    public void valueObjectsAreOnlyConsideredEqualIfAllOfTheirFieldsAreEqual() {
        ConfiguredEqualsVerifier verifier = EqualsVerifier.configure()
                .usingGetClass()
                .suppress(Warning.STRICT_HASHCODE);

        ArchRuleDefinition.methods().that()
                .haveName("hashCode").or().haveName("equals")
                .and().areDeclaredInClassesThat().haveSimpleNameContaining("Value")
                .should(FulfillEqualsAndHashcodeContract.configuredBy(verifier))
                .check(classes);
    }
}
{% endhighlight %}

1. [JUnit 5] provides the annotation `@DisplayName` for more expressive test case names. This is the most significant difference between a `specification` and an ordinary `test`. The test method name can be used as well, e.g. when using [JUnit 4].

2. [ArchUnit] supports a whole bunch of ways to select classes. In this example,
entities and value objects are simply distinguished by their class name.

3. The implementation of [FulfillEqualsAndHashcodeContract] is the same as above.


[import the classes]: https://www.archunit.org/userguide/html/000_Index.html#_importing_classes
[JUnit 4]: https://github.com/junit-team/junit4
[JUnit 5]: https://github.com/junit-team/junit5
[ArchUnit]: https://www.archunit.org
[EqualsVerifier]: https://jqno.nl/equalsverifier/
[behaviour-driven development]: https://dannorth.net/introducing-bdd/
[FulfillEqualsAndHashcodeContract]: #connecting-equalsverifier-to-archunit
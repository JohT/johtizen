---
layout: post
title:  "Analyze java package metrics in a graph database (Part 2)"
date:   2023-04-21 20:30:00 +0100
categories: data
tags: jqassistant neo4j cypher java dependency
author: JohT
use-mathjax: true
discussions-id: 46
---

[jQAssistant][jqassistant] extracts to structure of java applications
and writes them into [Neo4j][neo4j], a native graph database. This blog shows how to use these tools to analyze java package dependencies. It follows up [Analyze java dependencies with jQAssistant][BlogPart1].

### Table of Contents

{:.no_toc}

1. A markdown unordered list which will be replaced with the table of contents.
{:toc}

## Fast lane

(Updated November 2023)

If you'd like to start with ready-to-use reports and a fully automated analysis pipeline
then have a look at [Code Graph Analysis Pipeline][code-graph-analysis-pipeline] as already mentioned in [Part 1][BlogPart1].

If you on the other hand want to dig deeper into this topic step by step then continue reading.

## Prerequisites

Here is a short summary of the setup steps from [Part 1][BlogPart1]:

- [Download and install jQAssistant for the command line][jqassistant getting started]

- Scan your Java artifacts (jar, war, ear) for example in the directory `mylib` using `bin\jqassistant.cmd scan -reset -f mylib` (Windows) or `bin/jqassistant.sh scan -reset -f mylib` (Linux)

- Execute the following command to enrich the scanned data with further relationships

   ```shell
   bin/jqassistant.sh analyze -concepts classpath:Resolve dependency:Package dependency:Artifact
   ```

- Start the web application to query the collected data using `bin\jqassistant.cmd server` (Windows) or `bin/jqassistant.sh server` (Linux)

- Open [http://localhost:7474](http://localhost:7474) and sign in without user and password

- Click on the database symbol on the top left corner

- Read more about how to write Cypher queries in the [Neo4j Cypher Manual][neo4j cypher manual]

## Software Metrics

Lets start by collecting some fundamental software metrics that come in handy to get an overview on package coupling, abstractness and instability as described in [Calculate Software Metrics][CalculateMetrics] based on [Object-Oriented Design Quality Metrics by Robert Martin][MetricsRobertMartin].

### Create an index for the full qualified name of types (optional)

If you encounter performance issues or warning messages when using the full qualified name property `fqn` of *Type* nodes, then use the following line to create an additional database index.

```cypher
CREATE INDEX TYPE_FULL_QUALIFIED_NAME ON :Type(fqn)
```

### Add a weight property to the package dependency relationship

Graphs can not only have properties on their nodes (also known as vertices), they also can have properties on the relationships between them. For weighted graphs this is typically a property that reflects the strength of the relationship.

[jQAssistant ClassScanner][jQAssistantClassScanner] provides a weight property for class dependency relationships. It reflects how often the dependent class is used. Unfortunately, there is no weight provided for package dependency relationships. Nevertheless, this can easily be derived from their contained class dependencies.

The following *Cypher* statements show how to set weights on package dependency relationships calculated from the sum of their class dependency weights. This can be done separately for all types, for interfaces only or for a combination of both.

Combining interface and type dependencies with a predefined ratio can be useful to reflect the lower coupling nature of interfaces in contrast to higher coupling between implementation types. In the following last two examples this is realized by subtracting the interfaces weight from the general types weight, which leads to the pure types weight without interfaces and then adding the respective fraction of interface weight back again. The last two steps can be combined into one by using `(1 - fraction)`. Of course, these calculations are only examples and can be further refined to meet special requirements.

{::options parse_block_html="true" /}
<details><summary markdown="span">Cypher Query - Set package dependency weights</summary>

```cypher
// Add weight property to Package DEPENDS_ON relationships

 MATCH (sourcePackage:Package)-[:CONTAINS]->(sourceType:Type)-[typeDependency:DEPENDS_ON]->(dependentType:Type)<-[:CONTAINS]-(dependentPackage:Package)
 MATCH (sourcePackage)-[packageDependency:DEPENDS_ON]->(dependentPackage)
 WHERE sourcePackage.fqn <> dependentPackage.fqn
  WITH packageDependency
      ,sourcePackage.fqn                 AS sourcePackageName
      ,dependentPackage.fqn              AS dependentPackageName
      ,MIN(typeDependency.weight)        AS minTypeDependencyWeight
      ,MAX(typeDependency.weight)        AS maxTypeDependencyWeight
      ,AVG(typeDependency.weight)        AS avgTypeDependencyWeight
      ,SUM(typeDependency.weight)        AS packageDependencyWeight
      ,COUNT(dependentType.fqn)          AS dependentTypes
      ,COUNT(DISTINCT dependentType.fqn) AS distinctDependentTypes
   SET packageDependency.weight = packageDependencyWeight
RETURN sourcePackageName
      ,dependentPackageName
      ,minTypeDependencyWeight
      ,maxTypeDependencyWeight
      ,avgTypeDependencyWeight
      ,packageDependencyWeight
      ,dependentTypes
      ,distinctDependentTypes
```

</details>

<details><summary markdown="span">Cypher Query - Set package dependency weights for interfaces only</summary>

```cypher
// Add weight property for Interface Dependencies to Package DEPENDS_ON relationships

 MATCH (sourcePackage:Package)-[packageDependency:DEPENDS_ON]->(dependentPackage:Package)
 MATCH (sourcePackage)-[:CONTAINS]->(sourceType:Type)
 OPTIONAL MATCH (sourceType:Type)-[typeDependency:DEPENDS_ON]->(dependentInterface:Interface)<-[:CONTAINS]-(dependentPackage)
 WHERE sourcePackage.fqn <> dependentPackage.fqn
  WITH packageDependency
      ,sourcePackage.fqn             AS sourcePackageName
      ,dependentPackage.fqn          AS dependentPackageName
      ,SUM(typeDependency.weight)    AS packageInterfaceDependencyWeight
      ,COUNT(dependentInterface.fqn) AS dependentInterfaces
   SET packageDependency.weightInterfaces = packageInterfaceDependencyWeight
RETURN sourcePackageName
      ,dependentPackageName
      ,packageInterfaceDependencyWeight
      ,dependentInterfaces
```

</details>

<details><summary markdown="span">Cypher Query - Set package dependency weights based on 25% interface + type dependencies weight</summary>

```cypher
// Add weight25PercentInterfaces to Package DEPENDS_ON relationships

 MATCH (package:Package)-[r:DEPENDS_ON]->(dependent:Package)
  WITH package, r
      ,toInteger(r.weight - round(r.weightInterfaces * 0.75)) AS weight25PercentInterfaces
   SET r.weight25PercentInterfaces = weight25PercentInterfaces
RETURN package.fqn, weight25PercentInterfaces, r.weight, r.weightInterfaces
 ORDER BY weight25PercentInterfaces DESC
```

</details>

<details><summary markdown="span">Cypher Query - Set package dependency weights based on 10% interface + type dependencies weight</summary>

```cypher
// Add weight10PercentInterfaces to Package DEPENDS_ON relationships

 MATCH (package:Package)-[r:DEPENDS_ON]->(dependent:Package)
  WITH package, r
      ,toInteger(r.weight - round(r.weightInterfaces * 0.90)) AS weight10PercentInterfaces
   SET r.weight10PercentInterfaces = weight10PercentInterfaces
RETURN package.fqn, weight10PercentInterfaces, r.weight, r.weightInterfaces
 ORDER BY weight10PercentInterfaces DESC
```

</details>
{::options parse_block_html="false" /}

### Incoming Dependencies (Afferent Couplings)

Incoming dependencies (also referred to as *Fan-in*) are those that use the regarding class/package/artifact.
They can also be seen as consumers.
In this section we will focus on packages. With some adaptions the examples below should also work for classes and artifacts.

If you change code, the incoming dependencies might probably be affected. The more incoming dependencies, the harder it gets to change the code without the need to adapt the dependent code ("rigid code"). Even worse, it might affect the behavior of the dependent code in an unwanted way ("fragile code").

This doesn't mean that a package with many incoming dependencies and therefore high afferent coupling is problematic. It could be an API with interfaces that are meant to be used in many places and are written with stability and compatibility in mind.

The following cypher statements take the example from [Calculate Software Metrics][CalculateMetrics] one step further to distinguish between interfaces and types. They also query how many different artifacts are involved. These ideas can of course be elaborated further to meet other requirements.

{::options parse_block_html="true" /}
<details><summary markdown="span">Cypher Query - Query incoming package dependencies</summary>

```cypher
// Query Incoming Package Dependencies

   MATCH (p:Package)
OPTIONAL MATCH (p)-[:CONTAINS]->(it:Java:Type)<-[r:DEPENDS_ON]-(et:Java:Type)<-[:CONTAINS]-(ep:Package)<-[:CONTAINS]-(ea:Artifact)
OPTIONAL MATCH (it)<-[:DEPENDS_ON]-(eti:Java:Type:Interface)
   WHERE p <> ep
    WITH p
        ,COUNT(et)           AS incomingDependencies
        ,SUM(r.weight)       AS incomingDependenciesWeight
        ,COUNT(DISTINCT et)  AS incomingDependentTypes 
        ,COUNT(DISTINCT eti) AS incomingDependentInterfaces // also included in incomingDependentTypes
        ,COUNT(DISTINCT ep)  AS incomingDependentPackages
        ,COUNT(DISTINCT ea)  AS incomingDependentArtifacts
   ORDER BY incomingDependencies DESC
  RETURN p.fqn  AS packageName
        ,incomingDependencies
        ,incomingDependenciesWeight
        ,incomingDependentTypes
        ,incomingDependentInterfaces
        ,incomingDependentPackages
        ,incomingDependentArtifacts
```

</details>

<details><summary markdown="span">Cypher Query - Set incoming package dependencies property</summary>

```cypher
// Set Incoming Package Dependencies

   MATCH (p:Package)
OPTIONAL MATCH (p)-[:CONTAINS]->(it:Java:Type)<-[r:DEPENDS_ON]-(et:Java:Type)<-[:CONTAINS]-(ep:Package)<-[:CONTAINS]-(ea:Artifact)
OPTIONAL MATCH (it)<-[:DEPENDS_ON]-(eti:Java:Type:Interface)
   WHERE p <> ep
    WITH p
        ,COUNT(et)           AS incomingDependencies
        ,SUM(r.weight)       AS incomingDependenciesWeight
        ,COUNT(DISTINCT et)  AS incomingDependentTypes 
        ,COUNT(DISTINCT eti) AS incomingDependentInterfaces // also included in dependentTypes
        ,COUNT(DISTINCT ep)  AS incomingDependentPackages
        ,COUNT(DISTINCT ea)  AS incomingDependentArtifacts
     SET p.incomingDependencies        = incomingDependencies
        ,p.incomingDependenciesWeight  = incomingDependenciesWeight
        ,p.incomingDependentTypes      = incomingDependentTypes
        ,p.incomingDependentInterfaces = incomingDependentInterfaces
        ,p.incomingDependentPackages   = incomingDependentPackages
        ,p.incomingDependentArtifacts  = incomingDependentArtifacts
  RETURN p.fqn  AS packageName
        ,incomingDependencies
        ,incomingDependenciesWeight
        ,incomingDependentTypes
        ,incomingDependentInterfaces
        ,incomingDependentPackages
        ,incomingDependentArtifacts
```

</details>

<details><summary markdown="span">Cypher Query - Query incoming package method call dependencies</summary>

```cypher
//Query Incoming Package Method Call Dependencies

MATCH (p:Package)
OPTIONAL MATCH (p)-[:CONTAINS]->(t:Java:Type)-[:DECLARES]->(m:Method)<-[:INVOKES]-(dm:Method)<-[:DECLARES]-(dt:Java:Type)<-[:CONTAINS]-(dp:Package)<-[:CONTAINS]-(da:Artifact)
OPTIONAL MATCH (dm)<-[:DECLARES]-(dti:Interface)
WHERE p <> dp
 WITH p
     ,COUNT(dm)           AS incomingMethodCalls
     ,COUNT(DISTINCT dm)  AS incomingDistinctMethodCalls 
     ,COUNT(DISTINCT dt)  AS incomingMethodCallTypes 
     ,COUNT(DISTINCT dti) AS incomingMethodCallInterfaces
     ,COUNT(DISTINCT dp)  AS incomingMethodCallPackages
     ,COUNT(DISTINCT da)  AS incomingMethodCallArtifacts
 ORDER BY incomingMethodCalls DESC
RETURN p.fqn AS packageName
      ,incomingMethodCalls
      ,incomingDistinctMethodCalls
      ,incomingMethodCallTypes
      ,incomingMethodCallInterfaces
      ,incomingMethodCallPackages
      ,incomingMethodCallArtifacts
```

</details>

<details><summary markdown="span">Cypher Query - Set incoming package method call dependencies property</summary>

```cypher
//Set Incoming Package Method Call Dependencies

MATCH (p:Package)
OPTIONAL MATCH (p)-[:CONTAINS]->(t:Java:Type)-[:DECLARES]->(m:Method)<-[:INVOKES]-(dm:Method)<-[:DECLARES]-(dt:Java:Type)<-[:CONTAINS]-(dp:Package)<-[:CONTAINS]-(da:Artifact)
OPTIONAL MATCH (dm)<-[:DECLARES]-(dti:Interface)
WHERE p <> dp
 WITH p
     ,COUNT(dm)           AS incomingMethodCalls
     ,COUNT(DISTINCT dm)  AS incomingDistinctMethodCalls 
     ,COUNT(DISTINCT dt)  AS incomingMethodCallTypes 
     ,COUNT(DISTINCT dti) AS incomingMethodCallInterfaces
     ,COUNT(DISTINCT dp)  AS incomingMethodCallPackages
     ,COUNT(DISTINCT da)  AS incomingMethodCallArtifacts
   SET p.incomingMethodCalls          = incomingMethodCalls
      ,p.incomingDistinctMethodCalls  = incomingDistinctMethodCalls
      ,p.incomingMethodCallTypes      = incomingMethodCallTypes
      ,p.incomingMethodCallInterfaces = incomingMethodCallInterfaces
      ,p.incomingMethodCallPackages   = incomingMethodCallPackages
      ,p.incomingMethodCallArtifacts  = incomingMethodCallArtifacts
RETURN p.fqn AS packageName
      ,incomingMethodCalls
      ,incomingDistinctMethodCalls
      ,incomingMethodCallTypes
      ,incomingMethodCallInterfaces
      ,incomingMethodCallPackages
      ,incomingMethodCallArtifacts
```

</details>

{::options parse_block_html="false" /}

### Outgoing Dependencies (Efferent Couplings)

The outgoing dependencies (also referred to as *Fan-out*) are those that are used by the regarding class/package/artifact.
They can also be seen as supplier.
In this section we will focus on packages. With some adaptions the examples below should also work for classes and artifacts.

Code from other packages and libraries you're depending on (outgoing) might change over time. The more outgoing changes, the more likely and frequently code changes are needed. This involves time and effort which can be reduced by automation of tests and version updates. Automated tests are crucial to reveal updates, that change the behavior of the code unexpectedly ("fragile code").
As soon as more effort is required, keeping up becomes difficult ("rigid code"). Not being able to use a newer version might not only restrict features, it can get problematic if there are security issues. This might force you to take "fast but ugly" solutions into account which further increases technical dept.

This doesn't mean that a package with many outgoing dependencies and therefore high efferent coupling is problematic. It could just be the one place in an application where all dependencies are connected by intent and where maintenance is taken into account.

The following cypher statements take the example from [Calculate Software Metrics][CalculateMetrics] one step further to distinguish between interfaces and types. They also query how many different artifacts are involved. These ideas can of course be elaborated further to meet your requirements.

{::options parse_block_html="true" /}

<details><summary markdown="span">Cypher Query - Query outgoing package dependencies</summary>

```cypher
//Query Outgoing Package Dependencies

   MATCH (p:Package)
OPTIONAL MATCH (p)-[:CONTAINS]->(it:Java:Type)-[r:DEPENDS_ON]->(et:Java:Type)<-[:CONTAINS]-(ep:Package)<-[:CONTAINS]-(ea:Artifact)
OPTIONAL MATCH (it)-[:DEPENDS_ON]->(eti:Interface)
   WHERE p <> ep
    WITH p
        ,COUNT(et)           AS outgoingDependencies
        ,COUNT(DISTINCT et)  AS outgoingDependentTypes 
        ,COUNT(DISTINCT eti) AS outgoingDependentInterfaces // included in usedTypes
        ,COUNT(DISTINCT ep)  AS outgoingDependentPackages
        ,COUNT(DISTINCT ea)  AS outgoingDependentArtifacts
        ,SUM(r.weight)       AS outgoingDependenciesWeight
   ORDER BY outgoingDependencies DESC
  RETURN p.fqn  AS packageName
        ,outgoingDependencies
        ,outgoingDependentTypes
        ,outgoingDependentInterfaces
        ,outgoingDependentPackages
        ,outgoingDependentArtifacts
        ,outgoingDependenciesWeight
```

</details>

<details><summary markdown="span">Cypher Query - Set outgoing package dependencies property</summary>

```cypher
//Set Outgoing Package Dependencies

   MATCH (p:Package)
OPTIONAL MATCH (p)-[:CONTAINS]->(it:Java:Type)-[r:DEPENDS_ON]->(et:Java:Type)<-[:CONTAINS]-(ep:Package)<-[:CONTAINS]-(ea:Artifact)
OPTIONAL MATCH (it)-[:DEPENDS_ON]->(eti:Interface)
   WHERE p <> ep
    WITH p
        ,COUNT(et)           AS outgoingDependencies
        ,COUNT(DISTINCT et)  AS outgoingDependentTypes 
        ,COUNT(DISTINCT eti) AS outgoingDependentInterfaces // included in usedTypes
        ,COUNT(DISTINCT ep)  AS outgoingDependentPackages
        ,COUNT(DISTINCT ea)  AS outgoingDependentArtifacts
        ,SUM(r.weight)       AS outgoingDependenciesWeight
     SET p.outgoingDependencies        = outgoingDependencies
        ,p.outgoingDependenciesWeight  = outgoingDependenciesWeight
        ,p.outgoingDependentTypes      = outgoingDependentTypes
        ,p.outgoingDependentInterfaces = outgoingDependentInterfaces
        ,p.outgoingDependentPackages   = outgoingDependentPackages
        ,p.outgoingDependentArtifacts  = outgoingDependentArtifacts
  RETURN p.fqn  AS packageName
        ,outgoingDependencies
        ,outgoingDependentTypes
        ,outgoingDependentInterfaces
        ,outgoingDependentPackages
        ,outgoingDependentArtifacts
        ,outgoingDependenciesWeight
```

</details>

<details><summary markdown="span">Cypher Query - Query outgoing package method call dependencies</summary>

```cypher
//Query Outgoing Package Method Call Dependencies

MATCH (p:Package)
OPTIONAL MATCH (p)-[:CONTAINS]->(t:Java:Type)-[:DECLARES]->(m:Method)-[:INVOKES]->(dm:Method)<-[:DECLARES]-(dt:Java:Type)<-[:CONTAINS]-(dp:Package)<-[:CONTAINS]-(da:Artifact)
OPTIONAL MATCH (dm)<-[:DECLARES]-(dti:Interface)
WHERE p <> dp
 WITH p
     ,COUNT(dm)           AS outgoingMethodCalls
     ,COUNT(DISTINCT dm)  AS outgoingDistinctMethodCalls 
     ,COUNT(DISTINCT dt)  AS outgoingMethodCallTypes 
     ,COUNT(DISTINCT dti) AS outgoingMethodCallInterfaces
     ,COUNT(DISTINCT dp)  AS outgoingMethodCallPackages
     ,COUNT(DISTINCT da)  AS outgoingMethodCallArtifacts
ORDER BY outgoingMethodCalls DESC
RETURN p.fqn AS packageName
      ,outgoingMethodCalls
      ,outgoingDistinctMethodCalls
      ,outgoingMethodCallTypes
      ,outgoingMethodCallInterfaces
      ,outgoingMethodCallPackages
      ,outgoingMethodCallArtifacts
```

</details>

<details><summary markdown="span">Cypher Query - Set outgoing package method call dependencies properties</summary>

```cypher
//Set Outgoing Package Method Call Dependencies

MATCH (p:Package)
OPTIONAL MATCH (p)-[:CONTAINS]->(t:Java:Type)-[:DECLARES]->(m:Method)-[:INVOKES]->(dm:Method)<-[:DECLARES]-(dt:Java:Type)<-[:CONTAINS]-(dp:Package)<-[:CONTAINS]-(da:Artifact)
OPTIONAL MATCH (dm)<-[:DECLARES]-(dti:Interface)
WHERE p <> dp
 WITH p
     ,COUNT(dm)           AS outgoingMethodCalls
     ,COUNT(DISTINCT dm)  AS outgoingDistinctMethodCalls 
     ,COUNT(DISTINCT dt)  AS outgoingMethodCallTypes 
     ,COUNT(DISTINCT dti) AS outgoingMethodCallInterfaces
     ,COUNT(DISTINCT dp)  AS outgoingMethodCallPackages
     ,COUNT(DISTINCT da)  AS outgoingMethodCallArtifacts
   SET p.outgoingMethodCalls          = outgoingMethodCalls
      ,p.outgoingDistinctMethodCalls  = outgoingDistinctMethodCalls
      ,p.outgoingMethodCallTypes      = outgoingMethodCallTypes
      ,p.outgoingMethodCallInterfaces = outgoingMethodCallInterfaces
      ,p.outgoingMethodCallPackages   = outgoingMethodCallPackages
      ,p.outgoingMethodCallArtifacts  = outgoingMethodCallArtifacts
RETURN p.fqn AS packageName
      ,outgoingMethodCalls
      ,outgoingDistinctMethodCalls
      ,outgoingMethodCallTypes
      ,outgoingMethodCallInterfaces
      ,outgoingMethodCallPackages
      ,outgoingMethodCallArtifacts
```

</details>

{::options parse_block_html="false" /}

### Instability

As described in [Object-Oriented Design Quality Metrics][MetricsRobertMartin], the *Instability* metric is expressed as the ratio of the number of outgoing dependencies of a module (i.e., the number of code units that depend on it) to the total number of dependencies (i.e., the sum of incoming and outgoing dependencies):

$$ Instability = I = \frac{Outgoing\:Dependencies}{All\:Dependencies} $$

Small values near zero indicate low *Instability*. With no outgoing but some incoming dependencies the *Instability* is zero which is denoted as *maximally stable*. Such code units are more rigid and difficult to change without impacting other parts of the system.
If they are changed less because of that, they are considered *stable*.

Conversely, high values approaching one indicate high *Instability*. With some outgoing dependencies but no incoming ones the *Instability* is denoted as *maximally unstable*. Such code units are easier to change without affecting other modules, making them more flexible and less prone to cascading changes throughout the system. If they are changed more often because of that, they are considered *unstable*.

*Instability* is undefined if there aren't any dependencies, because this would lead to a division by zero. A commonly used trick to overcome this is to add a very small number to the denominator. Since the dependencies are all zero or positive, a division by zero is therefore not possible any more. The number is so small that it won't affect the result in a significant way. Another way is to exclude those cases by a `WHERE` statement as done in [Calculate Software Metrics][CalculateMetrics].

{::options parse_block_html="true" /}

<details><summary markdown="span">Cypher Query - Set instability based on previous set incoming and outgoing dependencies</summary>

```cypher
// Calculate and set Instability = outgoing / (outgoing + incoming) Dependencies

 MATCH (p:Package)
  WITH p
      ,toFloat(p.outgoingDependencies) / (p.outgoingDependencies + p.incomingDependencies + 1E-38) as instability
      ,toFloat(p.outgoingDependentTypes) / (p.outgoingDependentTypes + p.incomingDependentTypes + 1E-38) as instabilityTypes
      ,toFloat(p.outgoingDependentInterfaces) / (p.outgoingDependentInterfaces + p.incomingDependentInterfaces + 1E-38) as instabilityInterfaces
      ,toFloat(p.outgoingMethodCallPackages) / (p.outgoingMethodCallPackages + p.incomingMethodCallPackages + 1E-38) as instabilityPackages
      ,toFloat(p.outgoingMethodCallArtifacts) / (p.outgoingMethodCallArtifacts + p.incomingMethodCallArtifacts + 1E-38) as instabilityArtifacts
   SET p.instability           = instability
      ,p.instabilityTypes      = instabilityTypes
      ,p.instabilityInterfaces = instabilityInterfaces
      ,p.instabilityPackages   = instabilityPackages
      ,p.instabilityArtifacts  = instabilityArtifacts
RETURN p.fqn
      ,p.outgoingDependencies, p.incomingDependencies, instability
      ,p.outgoingDependentTypes, p.incomingDependentTypes, instabilityTypes
      ,p.outgoingDependentInterfaces, p.incomingDependentInterfaces, instabilityInterfaces
      ,p.outgoingDependentPackages, p.incomingDependentPackages, instabilityPackages
      ,p.outgoingDependentArtifacts, p.incomingDependentArtifacts, instabilityArtifacts
```

</details>

{::options parse_block_html="false" /}

### Abstractness

As mentioned above we've already taken into account that there is a difference between interface (abstract) and type (implementation) dependencies. But changing an abstract class or interface can even be harder than changing an implementation type. So why are they treated differently? If used the right way e.g. by applying the *Open/Closed Principle*, interfaces are made to depend on them safely. They provide abstraction and make it easy to extend their implementation. They are "closed for modification" which make them "trustworthy" to depend on.

Based on [Object-Oriented Design Quality Metrics][MetricsRobertMartin] we'll first count all kind of abstract types and than use the result to calculate *Abstractness* per category/package:

$$ Abstractness = \frac{abstract\:classes}{all\:classes} $$

Zero *Abstractness* means that there are no abstract types in the category (=package), one means that there are only abstract types.

#### Count abstract types

Again, based on [Calculate Software Metrics][CalculateMetrics] and enriched with some details, 
abstract classes can be count and set like this:

{::options parse_block_html="true" /}

<details><summary markdown="span">Cypher Query - Set count of different types and abstract classes</summary>

```cypher
//Count and set abstract types

   MATCH (package:Package)
OPTIONAL MATCH (package)-[:CONTAINS]->(type:Type) 
    WITH package
        ,COUNT(type) AS numberTypes
OPTIONAL MATCH (package)-[:CONTAINS]->(abstract:Class {abstract:true}) 
    WITH package
        ,numberTypes
        ,COUNT(abstract) AS numberAbstractClasses
OPTIONAL MATCH (package)-[:CONTAINS]->(enum:Enum)
    WITH package
        ,numberTypes
        ,numberAbstractClasses
        ,COUNT(enum) AS numberEnums
OPTIONAL MATCH (package)-[:CONTAINS]->(class:Class)
    WITH package
        ,numberTypes
        ,numberAbstractClasses
        ,numberEnums
        ,COUNT(class) - numberAbstractClasses + numberEnums AS numberNonAbstractTypes
OPTIONAL MATCH (package)-[:CONTAINS]->(annotation:Annotation) 
    WITH package
        ,numberTypes
        ,numberAbstractClasses
        ,numberEnums
        ,numberNonAbstractTypes
        ,COUNT(annotation) AS numberAnnotations
OPTIONAL MATCH (package)-[:CONTAINS]->(interface:Interface) 
    WITH package
        ,numberTypes
        ,numberAbstractClasses
        ,numberEnums
        ,numberNonAbstractTypes
        ,numberAnnotations
        ,COUNT(interface) AS numberInterfaces
        ,COUNT(interface) + numberAbstractClasses + numberAnnotations AS numberAbstractTypes
     SET package.numberTypes            = numberTypes
        ,package.numberNonAbstractTypes = numberNonAbstractTypes
        ,package.numberAbstractTypes    = numberAbstractTypes
        ,package.numberAbstractClasses  = numberAbstractClasses
        ,package.numberInterfaces       = numberInterfaces
        ,package.numberAnnotations      = numberAnnotations
        ,package.numberEnums            = numberEnums
  RETURN package.fqn AS packageName
        ,numberTypes
        ,numberNonAbstractTypes
        ,numberAbstractTypes
        ,numberAbstractClasses
        ,numberInterfaces
        ,numberAnnotations
        ,numberEnums
```

</details>

{::options parse_block_html="false" /}

#### Calculate Abstractness

The previously set properties containing the number of abstract classes can then be utilized to calculate the *Abstractness* like this:

{::options parse_block_html="true" /}

<details><summary markdown="span">Cypher Query - Calculate and Set Abstractness</summary>

```cypher
//Calculate and set Abstractness
 MATCH (p:Package)
  WITH p
      ,toFloat(p.numberAbstractTypes) / (p.numberTypes + 1E-38)  AS abstractness
   SET p.abstractness = abstractness
RETURN p.fqn AS packageName, p.numberAbstractTypes, p.numberTypes, abstractness
 ```

</details>

{::options parse_block_html="false" /}

### Distance between Abstractness and Instability

 Described as the "main sequence" in [Object-Oriented Design Quality Metrics][MetricsRobertMartin], the distance between *Abstractness* and *Instability* can be useful to find packages that are particular hard to change. The lower the distance the better. The scale factor $$ \frac{1}{\sqrt{2}} $$ is left out to get values between zero and one.

{::options parse_block_html="true" /}

<details><summary markdown="span">Cypher Query - Calculate Distance between Abstractness and Instability</summary>

```cypher
//Calculate distance between abstractness and instability

MATCH (artifact:Artifact)-[:CONTAINS]->(package:Package)-[:CONTAINS]->(type:Type)
 WITH artifact
     ,package
     ,abs(package.abstractness + package.instability -1) AS distance
     ,count(type) AS typesInPackage
RETURN artifact.fileName, package.fqn, distance, package.abstractness, package.instability, typesInPackage
 ORDER BY distance DESC, package.fqn
```

</details>

{::options parse_block_html="false" /}

## Cyclic dependencies

In [Software Metrics](#software-metrics) we have seen how to analyze incoming and outgoing dependencies. A special case are cyclic dependencies where two packages depend on each other. For example, if package "overview" depends on "settings" and vice versa. This can lead to very hard to change code because both packages would need to be changed together. It might be that they are tightly coupled by intent and implement a cohesive feature. In that case it could make sense to put them together into one package and find a good name for that. In most cases this is not intended though. Resolving those cycles is crucial for extensible and maintainable code. In this section we'll have a look at how to query cyclic dependencies.

### Query nodes with cyclic dependencies

The first query is a simple example to get direct cyclic dependencies as nodes for graphical exploration limited to 100 nodes.
How to adapt the query to also contain indirect cyclic dependencies (e.g. A needs B needs C needs A again) is described in [Manage Package Dependencies][ManagePackageDependencies].

{::options parse_block_html="true" /}

<details><summary markdown="span">Cypher Query - Nodes with direct cyclic dependencies </summary>

```cypher
// Cyclic Dependencies

MATCH (package:Package)-[:CONTAINS]->(type:Type)-[:DEPENDS_ON]->(dependentType:Type)<-[:CONTAINS]-(dependentPackage:Package)
MATCH (dependentPackage)-[:CONTAINS]->(cycleType:Type)-[:DEPENDS_ON]->(cycleDependentType:Type)<-[:CONTAINS]-(package)
WHERE package <> dependentPackage
RETURN package, dependentPackage
      ,type, dependentType, cycleType, cycleDependentType
 LIMIT 100
```

</details>

{::options parse_block_html="false" /}

### Query list of packages with cyclic dependencies

As discussed in [Detecting cycles using Cypher][DetectingCyclesUsingCypher] the simple query above leads to a list of duplicate nodes. For example if A and B have cyclic dependencies, both would be on the list (A<->B, B<->A). It needs an extra step to filter out those duplicate entries.

{::options parse_block_html="true" /}

<details><summary markdown="span">Cypher Query - Distinct list of packages with direct cyclic dependencies </summary>

```cypher
//Cyclic Dependencies as List

MATCH (package:Package)-[:CONTAINS]->(forwardSource:Type)-[:DEPENDS_ON]->(forwardTarget:Type)<-[:CONTAINS]-(dependentPackage:Package)
MATCH (dependentPackage)-[:CONTAINS]->(backwardSource:Type)-[:DEPENDS_ON]->(backwardTarget:Type)<-[:CONTAINS]-(package)
 WITH package
     ,dependentPackage
     ,collect(DISTINCT forwardSource.fqn)  AS forwardSources
     ,collect(DISTINCT forwardTarget.fqn)  AS forwardTargets
     ,collect(DISTINCT backwardSource.fqn) AS backwardSources
     ,collect(DISTINCT backwardTarget.fqn) AS backwardTarget
WHERE package <> dependentPackage
  AND (size(forwardTargets) > size(backwardTarget)
   OR (size(forwardTargets) = size(backwardTarget)
  AND  size(package.fqn)    >= size(dependentPackage.fqn)))
RETURN package.fqn
      ,dependentPackage.fqn
      ,forwardSources
      ,forwardTargets
      ,backwardSources
      ,backwardTarget
 LIMIT 30
 ```

</details>

{::options parse_block_html="false" /}

## Dependency Usage

This section shows further queries to what extend a dependency is used.
If only very few packages or classes of an artifact are used, it might be possible to use that functionality from somewhere else or rebuild it on your own. A good example for that are simple utility classes. The fewer dependencies the less effort it takes to update them or treat possible security issues. Of course there might also be dedicated API packages that provide a facade to an indispensable library that are used at one place intentionally. Thus, the following queries only provide a starting point for further investigation.

{::options parse_block_html="true" /}

<details><summary markdown="span">Cypher Query - How many packages compared to all existing are used by dependent artifacts? </summary>

```cypher
// How many packages compared to all existing are used by dependent artifacts?

MATCH (artifact:Artifact)-[:CONTAINS]-(package:Package)-[:DEPENDS_ON]->(dependentPackage:Package)<-[:CONTAINS]-(dependentArtifact:Artifact)
MATCH (dependentArtifact)-[:CONTAINS]->(dependentArtifactPackage:Package)-[:CONTAINS]->(dependentArtifactType:Type)
  WITH artifact
      ,dependentArtifact
      ,COUNT(DISTINCT dependentPackage.fqn)     AS dependentPackages
      ,COUNT(DISTINCT dependentArtifactPackage) AS dependentArtifactPackages
      ,collect(DISTINCT dependentPackage.fqn)   AS dependentPackageNames
RETURN artifact.fileName
      ,dependentArtifact.fileName
      ,dependentPackages
      ,dependentArtifactPackages
      ,toFloat(dependentPackages) / (dependentArtifactPackages + 1E-38) AS packageUsagePercentage
      ,dependentPackageNames
ORDER BY packageUsagePercentage ASC
LIMIT 20
```

</details>

<details><summary markdown="span">Cypher Query - How many classes compared to all existing are used by dependent packages? </summary>

```cypher
// How many classes compared to all existing are used by dependent packages?

MATCH (artifact:Artifact)-[:CONTAINS]->(package:Package)-[:CONTAINS]->(type:Type)-[:DEPENDS_ON]->(dependentType:Type)<-[:CONTAINS]-(dependentPackage:Package)<-[:CONTAINS]-(dependentArtifact:Artifact)
MATCH (dependentPackage)-[:CONTAINS]->(dependentPackageType:Type)
WHERE type <> dependentType
  AND artifact <> dependentArtifact
  WITH artifact
      ,dependentArtifact
      ,package
      ,dependentPackage
      ,COUNT(DISTINCT dependentType)        AS dependentTypes
      ,COUNT(DISTINCT dependentPackageType) AS dependentPackageTypes
      ,collect(DISTINCT dependentType.fqn)  AS dependentTypeNames
RETURN artifact.fileName
      ,dependentArtifact.fileName
      ,package.fqn as packageName
      ,dependentPackage.fqn
      ,dependentTypes
      ,dependentPackageTypes
      ,toFloat(dependentTypes) / (dependentPackageTypes + 1E-38) AS typeUsagePercentage
      ,dependentTypeNames
ORDER BY typeUsagePercentage ASC
LIMIT 20
```

</details>

The next query does the same as the queries above but from the opposite point of view.
It looks for types that are used in a wide spread manner. This might for example reveal utility classes
that are used all over the place. Instead of sharing those it could be a valid option to take them
as snippets that won't hurt to be duplicated where needed. This could lead to lower *Coupling* and improved *Mobility*,
so that the code can be changed, reused and moved easier.

<details><summary markdown="span">Cypher Query -  Which types are used by many different packages?</summary>

```cypher
// List types that are used by many different packages

MATCH (artifact:Artifact)-[:CONTAINS]->(package:Package)-[:CONTAINS]->(type:Type)-[:DEPENDS_ON]->(dependentType:Type)<-[:CONTAINS]-(dependentPackage:Package)<-[:CONTAINS]-(dependentArtifact:Artifact)
WHERE package <> dependentPackage
WITH  dependentType
     ,labels(dependentType) AS dependentTypeLabels
     ,COUNT(DISTINCT package) AS numberOfUsingPackages
RETURN dependentType.fqn
      ,dependentTypeLabels
      ,numberOfUsingPackages
 ORDER BY numberOfUsingPackages DESC
 LIMIT 50
```

</details>

{::options parse_block_html="false" /}

## Interface Segregation

Well known from [Design Principles and Design Patterns by Robert C. Martin][DesignPrinciplesAndPatterns], the *Interface Segregation Principle* suggests that software components should have narrow, focused interfaces rather than large, general-purpose ones. The goal is to minimize the dependencies between components and increase modularity, flexibility, and maintainability.

Smaller, focused and purpose-driven interfaces

- make it easier to modify individual components without affecting the rest of the system.
- make it clearer which client is affected by which change.
- don't force their clients to depend on methods they don't need.
- reduce the scope of changes since a change to one component doesn't affect others.
- lead to a more loosely coupled architecture that is easier to understand and maintain.

### Find code that could benefit from applying the Interface Segregation Principle

The following criterions indicate interfaces that could benefit from getting split up. It is important to note that here every public method is considered as part of an interface. Thus, this also applies to classes and not only to abstract types.

- Many clients use only one method of an interface
- Many clients use the same small group of methods of an interface
- The ratio of per client used methods to all declared methods of an interface is small
- The interface is used by clients for different functional purposes

In the first case it could make sense to replace the dependency to the interface by the result of the single method that is called. For example: If the type "Book" is used as a parameter and "getISBN" is the only used method, then it would make sense to replace "Book" by the ISBN type. This change makes the method reuseable and easier to call if no "Book" instance is available. For example could a "Citation" object also provide a getISBN and the method could also be used for that.

Except for the last criterion that requires domain knowledge, the other ones from above are actually measurable and queryable.
The following Cypher statement shows how to query candidates for *Interface Segregation* by finding groups of interface methods
that are often used together.

{::options parse_block_html="true" /}

<details><summary markdown="span">Cypher Query - Candidates for Interface Segregation</summary>

```cypher
// Candidates for Interface Segregation 

MATCH (type:Type)-[:DECLARES]->(method:Method)-[:INVOKES]->(dependentMethod:Method)<-[:DECLARES]-(dependentType:Type)
MATCH (dependentType)-[:DECLARES]->(declaredMethod:Method)
WHERE type.fqn <> dependentType.fqn
  AND dependentMethod.name IS NOT NULL
 WITH type
     ,dependentType
     ,collect(DISTINCT dependentMethod.name) AS calledMethodNames
     ,count(DISTINCT dependentMethod)        AS calledMethods
     ,count(DISTINCT declaredMethod)         AS declaredMethods
     ,labels(dependentType)                  AS dependentTypeLabels
 WITH dependentType
     ,declaredMethods
     ,calledMethodNames
     ,calledMethods
     ,dependentTypeLabels
     ,count(DISTINCT type) AS callerTypes
     ,calledMethods * 1.0 / declaredMethods AS calledMethodsPercent
 WHERE calledMethodsPercent < 0.2
RETURN dependentType.fqn, dependentTypeLabels, calledMethodNames, declaredMethods, calledMethods, calledMethodsPercent, callerTypes
 ORDER BY callerTypes DESC, calledMethodsPercent, dependentType.fqn
 LIMIT 30
```

</details>

{::options parse_block_html="false" /}

## Summary

This blog article is the second part of a series on how to analyze Java code using a graph database.
It builds upon how to setup [jQAssistant][jqassistant] and [Neo4j] from [Part 1][BlogPart1] and discusses various software metrics and how to calculate them using Cypher statements. The metrics include afferent coupling, and efferent coupling, abstractness and instability. Furthermore, it is shown how to reveal cyclic dependencies, rarely used packages and types that may benefit from applying the *Dependency Inversion Principle*. All examples focus on providing a prioritized list for software design refactoring to get the greatest benefit out of it.

<br>

---

## Updates

- 2023-05-23: [Reduce formula width for mobile devices and fix missing syntax highlighting](https://github.com/JohT/johtizen/pull/47)
- 2023-11-19: [Reference Code Graph Analysis Pipeline](https://github.com/JohT/johtizen/pull/43)

## References

- [About jQAssistant][jqassistant]
- [About Neo4j][neo4j]
- [Analyze java dependencies with jQAssistant (Blog Part 1)][BlogPart1]
- [Code Graph Analysis Pipeline][code-graph-analysis-pipeline]
- [Calculate Metrics with jQAssistant][CalculateMetrics]
- [Design Principles and Design Patterns by Robert C. Martin][DesignPrinciplesAndPatterns]
- [Detecting cycles using Cypher (Neo4j Community)][DetectingCyclesUsingCypher]
- [Getting started with jQAssistant][jqassistant getting started]
- [jQAssistant ClassScanner][jQAssistantClassScanner]
- [Manage Package Dependencies with jQAssistant][ManagePackageDependencies]
- [Neo4j Cypher Manual][neo4j cypher manual]
- [Object-Oriented Design Quality Metrics by Robert Martin][MetricsRobertMartin]

[BlogPart1]: https://joht.github.io/johtizen/data/2021/02/21/java-jar-dependency-analysis.html
[CalculateMetrics]: https://101.jqassistant.org/calculate-metrics/index.html
[DesignPrinciplesAndPatterns]: http://staff.cs.utu.fi/~jounsmed/doos_06/material/DesignPrinciplesAndPatterns.pdf
[ManagePackageDependencies]: https://101.jqassistant.org/manage-package-dependencies/index.html
[MetricsRobertMartin]: https://api.semanticscholar.org/CorpusID:18246616
[jQAssistantClassScanner]: https://jqassistant.github.io/jqassistant/doc/1.11.1/manual/#ClassScanner
[DetectingCyclesUsingCypher]: https://community.neo4j.com/t/detecting-cycles-using-cypher/4944
[jqassistant]: https://jqassistant.org
[jqassistant getting started]: https://jqassistant.org/get-started
[neo4j]: https://neo4j.com
[neo4j cypher manual]: https://neo4j.com/docs/cypher-manual/current
[code-graph-analysis-pipeline]: https://github.com/JohT/code-graph-analysis-pipeline

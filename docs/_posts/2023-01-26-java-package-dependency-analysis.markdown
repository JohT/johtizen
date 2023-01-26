---
layout: post
title:  "Analyze java package dependencies with jQAssistant (Part 2)"
date:   2023-01-26 07:30:00 +0100
categories: data
tags: jqassistant neo4j cypher java dependency
author: JohT
discussions-id: 40
---

[jQAssistant][jqassistant] extracts meta data from java applications
and writes them into [Neo4j][neo4j], a native graph database. This blog shows how to use these tools to analyze java package dependencies. It follows up [Analyze java dependencies with jQAssistant][BlogPart1] that shows how to set everything up.

### Table of Contents

{:.no_toc}

1. A markdown unordered list which will be replaced with the table of contents.
{:toc}

## Prerequisites

Here is a short summary of the setup steps from [Part 1][BlogPart1]:

- [Download and install jQAssistant for the command line][jqassistant getting started]

- Scan your Java artifacts (jar, war, ear) for example in the `mylib` directory using `bin\jqassistant.cmd scan -reset -f mylib` (Windows) or `bin/jqassistant.sh scan -reset -f mylib` (Linux)

- Execute the following command to enrich the scanned data with further relationships

   ```shell
   bin/jqassistant.sh analyze -concepts classpath:Resolve dependency:Package dependency:Artifact
   ```

- Start the web application to query the collected data using `bin\jqassistant.cmd server` (Windows) or `bin/jqassistant.sh server` (Linux)

- Open [http://localhost:7474](http://localhost:7474) and sign in without user and password

- Click on the database symbol on the top left corner

- Read more about how to write Cypher queries in the [Neo4j Cypher Manual][neo4j cypher manual]

## Example Data

These artifacts were downloaded, analyzed by [jQAssistant][jqassistant] and written into the [Neo4j][neo4j] graph database to be used as a real world example for all topics below:

- [axon-eventsourcing-4.6.2](https://mvnrepository.com/artifact/org.axonframework/axon-eventsourcing/4.6.2)
- [axon-modelling-4.6.2](https://mvnrepository.com/artifact/org.axonframework/axon-modelling/4.6.2)
- [axon-messaging-4.6.2](https://mvnrepository.com/artifact/org.axonframework/axon-messaging/4.6.2)
- [axon-configuration-4.6.2](https://mvnrepository.com/artifact/org.axonframework/axon-configuration/4.6.2)
- [axon-test-4.6.2](https://mvnrepository.com/artifact/org.axonframework/axon-test/4.6.2)
- [axon-disruptor-4.6.2](https://mvnrepository.com/artifact/org.axonframework/axon-disruptor/4.6.2)

If you want to know more about it: [AxonFramework](https://github.com/AxonFramework/AxonFramework) is a
> Framework for Evolutionary Message-Driven Microservices on the JVM

## Software Metrics

Lets start by collecting some fundamental software metrics that come in handy to get an overview on package coupling, abstractness and instability as described in [Calculate Software Metrics][CalculateMetrics] based on [Object-Oriented Design Quality Metrics by Robert Martin][MetricsRobertMartin].

### Create an index for the full qualified name of types (optional)

If you encounter performance issues or warning messages when using the full qualified name property of *Type* nodes, then use the following line to create an additional database index.

```cypher
CREATE INDEX TYPE_FULL_QUALIFIED_NAME ON :Type(fqn)
```

### Add a weight property to the package dependency relationship

Graphs can not only have properties on their nodes (=vertices), they also can have properties on the relationships between them. For weighted graphs this is typically a property that reflects the strength of the relationship.

[jQAssistant ClassScanner][jQAssistantClassScanner] provides a weight property for class dependency relationships. It reflects how often the dependent class is used. Unfortunately, there is no weight provided for package dependency relationships. Fortunately, this can easily be derived from their contained class dependencies.

The following *Cypher* statements show how to set weights on package dependency relationships calculated from the sum of their class dependency weights. This can be done separately for all types, for interfaces only or for a combination of both.

Combining interface and type dependencies with a predefined ratio can be useful to reflect the lower coupling nature of interfaces in contrast to higher coupling between implementation types. In the following last two examples this is realized by subtracting the interfaces weight from the general types weight, which leads to the pure types weight without interfaces and then adding the respective fraction of interface weight back again. The last two steps can be combined into one, that is why `(1 - fraction)` is used. Of course, these calculations are only examples and can be refined to meet special requirements.

{::options parse_block_html="true" /}
<details><summary markdown="span">Cypher Query - Set package dependency weights</summary>

```cypher
// Add weight property to Package DEPENDS_ON Relationship 

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
// Add weight property for Interface Dependencies to Package DEPENDS_ON Relationship

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

The incoming dependencies are those that use the given class/package/artifact.
They can also be seen as consumers.
In this section we will focus on packages. With some adaptions the examples below should also work for classes and artifacts.

If you change code, the incoming dependencies might probably be affected. The more incoming dependencies, the harder it gets to change the code without the need to adapt the dependent code ("rigid code"). Even worse, it could affect the behavior of the dependent code accidentally ("fragile code").

This doesn't mean that a package with many incoming dependencies and therefore high afferent coupling is problematic. It could be an API with interfaces that are meant to be used wherever needed.

The following cypher statements take the example from [Calculate Software Metrics][CalculateMetrics] one step further and provide different kinds of incoming dependencies to be able to distinguish between interfaces and types. They also query how many different artifacts depend on the given package. These ideas can of course be elaborated further to meet your requirements.

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

The outgoing dependencies are those that are used by the inspected class/package/artifact.
They can also be seen as supplier.
In this section we will focus on packages. With some adaptions the examples below should also work for classes and artifacts.

Code from other packages and libraries that you depend on (outgoing) might change over time. The more outgoing changes, the more likely and frequently code changes are needed. The more effort a change causes, the harder it gets to switch to the newest version of a library which restricts the likelihood of new features and the higher is the probability to look for "fast but not so pretty" solutions ("rigid code").
In the worst case the behavior of one of the many dependencies changes and the code breaks unexpectedly ("fragile code").

This doesn't mean that a package with many outgoing dependencies and therefore high efferent coupling is problematic. It could just be the one place where all dependencies are connected by intent.

The following cypher statements take the example from [Calculate Software Metrics][CalculateMetrics] one step further and provide different kinds of outgoing dependencies to be able to distinguish between interfaces and types. They also query how many different artifacts are used by the given package. These ideas can of course be elaborated further to meet your requirements.

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

As described in [Object-Oriented Design Quality Metrics by Robert Martin][MetricsRobertMartin], the *Instability* metric is expressed as a ratio of the number of outgoing dependencies of a module (i.e., the number of code units that depend on it) to the total number of dependencies (i.e., the sum of incoming and outgoing dependencies):

$$ Instability = I = \frac{Ce}{Ce + Ca} = \frac{Efferent Coupling}{Efferent Coupling + Afferent Coupling} = \frac{Outgoing Dependencies}{Outgoing Dependencies + Incoming Dependencies} $$

Small values near 0 indicate low *Instability*. With no outgoing but some incoming dependencies the *Instability* is zero which is denoted as *maximally stable*. Such code units are more rigid and difficult to change without impacting other parts of the system.
If they are changed less because of that, they are considered *stable*.

Conversely, high values approaching 1 indicate high *Instability*. With some outgoing dependencies but no incoming ones the *Instability* is denoted as *maximally unstable*. Such code units are easier to change without affecting other modules, making them more flexible and less prone to cascading changes throughout the system. If they are changed more often because of that, they are considered *unstable*.

*Instability* is undefined if there aren't any dependencies, because this would lead to a division by zero. A commonly used trick to overcome this is to add a very small number to the denominator. Since the dependencies are all zero or positive, a division by zero is so not possible any more. The small number won't affect the result in a significant way. Another way is to exclude those cases by a `WHERE` statement as done in [Calculate Software Metrics][CalculateMetrics].

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

### Count and set abstract classes

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

### Calculate and set Abstractness

The previously set properties containing the number of abstract classes can then be utilized to calculate the *Abstractness*:

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

## Cyclic dependencies

In [Software Metrics](#software-metrics) we have seen how to analyze incoming and outgoing dependencies. A special case of that are cyclic dependencies. For example, if package "overview" depends on "settings" and vice versa. This can lead to very hard to change code because both packages would need to be changed together. It might be that they are tightly coupled by intent and implement a cohesive feature. In that case it would make sense to put them together into one package and find a good name for that. In most cases this is not intended though. Resolving those cycles is crucial for extensible and maintainable code. In this section we'll have a look at how to query cyclic dependencies.

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

As discussed in [Detecting cycles using Cypher][DetectingCyclesUsingCypher] the simple query above leads to a list of duplicate nodes. For example if A and B have cyclic dependencies, both would be on the list (A<->B, B<->A). It needs an extra step to filter out the duplicate entries.

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

This section shows further queries i use to find out more about to what extend a dependency is used.
If only very few packages or classes of an artifact are used, it might be possible to use that functionality from somewhere else or rebuild it on your own. A good example for that are simple utilities. The fewer dependencies the less effort it takes to update them or treat possible security issues. Of course there might also be dedicated API packages that provide a facade to a library and are used at one place (false positive). Thus, the following queries provide a good basis for further investigation but arne't capable in that form for e.g. rule enforcement.

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

The next query does the same as the queries above but from the opposite viewpoint.
It looks for types that are used in a wide spread manner. This might for example reveal util classes
that are used all over the place. Instead of sharing those it could be preferable to take them
as snippets that won't hurt to be duplicated where needed. This could lead to lower *Coupling* and improve *Mobility*,
so the code can be easier changed, reused and moved.

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

- make it is easier to modify or replace individual components without affecting the rest of the system.
- make it clearer which client is affected by a specific change
- don't force their clients to depend on methods they don't need
- reduce the scope of changes since a change to one component doesn't affect others
- lead to a more loosely coupled architecture that is easier to understand and maintain.

### Find code that could benefit from applying the *Interface Segregation Principle*

The following criterions indicate interfaces that could benefit from getting split up. It is important to note that here every public method is considered as a part of an interface. Thus, this also applies to classes and not only to abstract types.

- Many clients use only one method of the interface
- Many clients use the same small group of methods of the interface
- The ratio of used methods to all declared methods in the interface (per client) is small
- The interface is used by clients for different functional purposes

In the first case it could make sense to replace the dependency to the whole interface by the result of the single method, that is called. For example, if the type "Book" is used as a parameter and "getISBN" is the only used method, then it would make sense to replace "Book" by the ISBN type. This change makes the method reuseable and easier to call if no "Book" instance is available. For example could a "Citation" object also provide a getISBN and the method could also be used for that.

Except for the last criterion that requires domain knowledge, the other ones from above are actually measurable and queryable.
The following Cypher statement shows how such a query could look like.

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

## Summary

<br>

---

## References

- [About jQAssistant][jqassistant]
- [About Neo4j][neo4j]
- [Analyze java dependencies with jQAssistant][BlogPart1]
- [Calculate Software Metrics][CalculateMetrics]
- [Design Principles and Design Patterns by Rob ert C. Martin][DesignPrinciplesAndPatterns]
- [Detecting cycles using Cypher][DetectingCyclesUsingCypher]
- [Getting started with jQAssistant][jqassistant getting started]
- [Manage Package Dependencies][ManagePackageDependencies]
- [Neo4j Cypher Manual][neo4j cypher manual]
- [Object-Oriented Design Quality Metrics by Robert Martin][MetricsRobertMartin]
- [jQAssistant ClassScanner][jQAssistantClassScanner]

[BlogPart1]: https://joht.github.io/johtizen/data/2021/02/21/java-jar-dependency-analysis.html
[CalculateMetrics]: https://101.jqassistant.org/calculate-metrics/index.html
[DesignPrinciplesAndPatterns]: http://staff.cs.utu.fi/~jounsmed/doos_06/material/DesignPrinciplesAndPatterns.pdf
[ManagePackageDependencies]: https://101.jqassistant.org/manage-package-dependencies/index.html
[MetricsRobertMartin]: https://kariera.future-processing.pl/blog/object-oriented-metrics-by-robert-martin
[jQAssistantClassScanner]: https://jqassistant.github.io/jqassistant/doc/1.11.1/manual/#ClassScanner
[DetectingCyclesUsingCypher]: https://community.neo4j.com/t/detecting-cycles-using-cypher/4944
[jqassistant]: https://jqassistant.org
[jqassistant getting started]: https://jqassistant.org/get-started
[neo4j]: https://neo4j.com
[neo4j cypher manual]: https://neo4j.com/docs/cypher-manual/current

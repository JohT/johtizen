---
layout: post
title:  "Analyze java type dependencies with jQAssistant (Part 2)"
date:   2022-12-17 20:15:00 +0100
categories: data
tags: jqassistant neo4j cypher java dependency
author: JohT
discussions-id: 40
---

[jQAssistant][jqassistant] extracts meta data of java applications
and writes them into [Neo4j][neo4j], a native graph database. This blog shows how these tools can be
used to analyze java type dependencies.

### Table of Contents

{:.no_toc}

1. A markdown unordered list which will be replaced with the table of contents.
{:toc}

## Notes

- Part 1: https://joht.github.io/johtizen/data/2021/02/21/java-jar-dependency-analysis.html
- Optimize Cartesian Product by creating an index: https://stackoverflow.com/questions/35583361/optimize-cypher-query-to-avoid-cartesian-product
- Neo4j CREATE INDEX Reference: https://neo4j.com/docs/cypher-manual/current/indexes-for-search-performance
- Neo4j MERGE Reference: https://neo4j.com/docs/cypher-manual/current/clauses/merge/
- Optional Relationships: https://stackoverflow.com/questions/31793712/optional-relationship-with-cypher
- jQAssistant multiple jars with unique types: https://stackoverflow.com/questions/33940842/multiple-jars-with-unique-types
- !!! jQAssistant Concept "classpath:Resolve": https://jqassistant.github.io/jqassistant/doc/1.8.0/#classpath:Resolve
- Neo4j MERGE relationships with properties: https://stackoverflow.com/questions/22850588/neo4j-merge-relationships-with-properties
- Calculate metrics: https://101.jqassistant.org/calculate-metrics/index.html
- Object-oriented metrics by Robert Martin: https://kariera.future-processing.pl/blog/object-oriented-metrics-by-robert-martin
- How to group & order by distinct labels with Cypher: https://stackoverflow.com/questions/34726973/how-to-group-order-by-distinct-labels-with-cypher
- Detecting cycles using Cypher: https://community.neo4j.com/t5/neo4j-graph-platform/detecting-cycles-using-cypher/td-p/16469
- Neo4j Download: https://neo4j.com/download-center
- Graph Data Science Neo4j Server https://neo4j.com/docs/graph-data-science/current/installation/neo4j-server/
- Neo4j Graph Data Science - Depth First Search: https://neo4j.com/docs/graph-data-science/current/algorithms/dfs
- Coloring nodes in Neo4j depending on property: https://community.neo4j.com/t5/neo4j-graph-platform/coloring-nodes-in-neo4j-depending-on-property/td-p/36959
- Change Style in Neo4j Browser: https://community.neo4j.com/t5/neo4j-graph-platform/visualization-can-i-clear-the-current-color-setting-in-neo4j/td-p/41328
- apoc.create.addLabels: https://neo4j.com/labs/apoc/4.0/overview/apoc.create/apoc.create.addLabels
- From Louvain to Leiden: https://www.nature.com/articles/s41598-019-41695-z
- Java in Google Colab: https://stackoverflow.com/questions/51287258/how-can-i-use-java-in-google-colab
- creating a new Neo4j database with community edition : https://stackoverflow.com/questions/60429947/error-occurs-when-creating-a-new-database-under-neo4j-4-0

Delete relationships of packages containing types, that aren't in that package
because their full qualified name doesn't start with the package name.

```shell
jqassistant.sh scan -reset -f .
jqassistant.sh analyze -concepts classpath:Resolve
jqassistant.sh analyze -concepts classpath:Resolve dependency:Package dependency:Artifact java:JavaVersion java:LambdaMethod java:MethodOverrides
jqassistant.sh server
```

```shell
jqassistant.sh scan -storeUri bolt://localhost:7687 -storeUsername neo4j -storePassword neo4j -reset -f .
jqassistant.sh analyze -storeUri bolt://localhost:7687 -storeUsername neo4j -storePassword neo4j -concepts classpath:Resolve dependency:Package dependency:Artifact java:JavaVersion
```

## Software Metrics

### (/) Delete contains relationship if a type does not belong to a package

```cypher
MATCH (package:Package)-[r:CONTAINS]->(type:Type)
WHERE NOT type.fqn STARTS WITH package.fqn 
DELETE r
 RETURN package, type
 LIMIT 6
```

### Query dependent packages

```cypher
MATCH (package:Package)-[:CONTAINS]->(type:Type)-[:DEPENDS_ON]->(dependentType:Type)<-[:CONTAINS]-(dependentPackage:Package)
 WHERE type.fqn = 'org.axonframework.eventsourcing.conflictresolution.DefaultConflictResolver' 
RETURN package, type, dependentType, dependentPackage 
LIMIT 5
```

### (/) Delete wrong "SAME_AS" relationships between types with different full qualified names (fqn)

or with types on them selves.

```cypher
 MATCH (type:Type)-[r:IS_SAME_AS]-(sameType:Type)
 WHERE type.fqn <> sameType.fqn
    OR type = sameType
DELETE r
RETURN type, sameType
 LIMIT 100
```

### Create index for the full qualified name of types

```cypher
CREATE INDEX TYPE_FULL_QUALIFIED_NAME ON :Type(fqn)
```

### (/) Using index for the full qualified name of types

Cartesian Product query that joins nodes without relationships by on of their properties,
in this case the full qualified name (fqn) of the java type.

```cypher
MATCH (package:Package)-[:CONTAINS]->(type:Type)-[:DEPENDS_ON]->(dependentType:Type)
     ,(sameType:Type)<-[:CONTAINS]-(dependentPackage:Package)
 WHERE dependentType.fqn = sameType.fqn
   AND type.fqn = 'org.axonframework.eventsourcing.conflictresolution.DefaultConflictResolver'
RETURN package, type, dependentType, sameType, dependentPackage
LIMIT 20
```

```cypher
MATCH (package:Package)-[:CONTAINS]->(type:Type)-[:DEPENDS_ON]->(dependentType:Type)-[:IS_SAME_AS*0..1]-(sameType:Type)<-[:CONTAINS]-(dependentPackage:Package)
 WHERE type.fqn = 'org.axonframework.eventsourcing.conflictresolution.DefaultConflictResolver' 
   AND dependentType.fqn STARTS WITH 'org.axonframework'
RETURN package, type, dependentType, sameType, dependentPackage 
LIMIT 50
```

```cypher
 MATCH (type:Type)-[:DEPENDS_ON]->(dependentType:Type)-[:IS_SAME_AS*0..1]-(sameType:Type)<-[:CONTAINS]-(dependentPackage:Package)
 WHERE type.fqn = 'org.axonframework.eventsourcing.conflictresolution.DefaultConflictResolver' 
   AND dependentType.fqn STARTS WITH 'org.axonframework'
RETURN type, dependentType, sameType, dependentPackage 
LIMIT 50
```

```cypher
 MATCH (type:Type)-[:DEPENDS_ON]->(dependentType:Type)
 WHERE type.fqn = 'org.axonframework.eventsourcing.conflictresolution.DefaultConflictResolver' 
   AND dependentType.fqn STARTS WITH 'org.axonframework'
OPTIONAL MATCH (dependentType)<-[:CONTAINS]-(dependentPackage:Package)
RETURN type, dependentType, dependentPackage 
LIMIT 50
```

### Add weight property to Package DEPENDS_ON Relationship

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

### Incoming Package Dependencies (Afferent Couplings)

https://101.jqassistant.org/calculate-metrics/index.html

```cypher
   MATCH (p:Package)
OPTIONAL MATCH (p)-[:CONTAINS]->(it:Java:Type)<-[r:DEPENDS_ON]-(et:Java:Type)<-[:CONTAINS]-(ep:Package)<-[:CONTAINS]-(ea:Artifact)
OPTIONAL MATCH (it)<-[:DEPENDS_ON]-(eti:Java:Type:Interface)
   WHERE p <> ep
    WITH p
        ,COUNT(et)           AS incomingDependencies
        ,SUM(r.weight)       AS incomingDependenciesWeight
        ,COUNT(DISTINCT et)  AS dependentTypes 
        ,COUNT(DISTINCT eti) AS dependentInterfaces // also included in usedTypes
        ,COUNT(DISTINCT ep)  AS dependentPackages
        ,COUNT(DISTINCT ea)  AS dependentArtifacts
   ORDER BY incomingDependencies DESC
  RETURN p.fqn as packageName
        ,incomingDependencies, incomingDependenciesWeight
        ,dependentTypes, dependentInterfaces, dependentPackages, dependentArtifacts
   LIMIT 50
```

### Incoming Class Dependencies (Afferent Couplings)

https://101.jqassistant.org/calculate-metrics/index.html

```cypher
   MATCH (p:Package)
OPTIONAL MATCH (p)-[:CONTAINS]->(it:Java:Type)<-[r:DEPENDS_ON]-(et:Java:Type)<-[:CONTAINS]-(ep:Package)<-[:CONTAINS]-(ea:Artifact)
OPTIONAL MATCH (it)<-[:DEPENDS_ON]-(eti:Java:Type:Interface)
   WHERE p <> ep
    WITH p, it
        ,COUNT(et)           AS incomingDependencies
        ,SUM(r.weight)       AS incomingDependenciesWeight
        ,COUNT(DISTINCT et)  AS incomingDependentTypes 
        ,COUNT(DISTINCT eti) AS incomingDependentInterfaces // also included in usedTypes
        ,COUNT(DISTINCT ep)  AS incomingDependentPackages
        ,COUNT(DISTINCT ea)  AS incomingDependentArtifacts
   ORDER BY incomingDependencies DESC
  RETURN p.fqn  AS packageName
        ,it.fqn AS className
        ,incomingDependencies
        ,incomingDependenciesWeight
        ,incomingDependentTypes
        ,incomingDependentInterfaces
        ,incomingDependentPackages
        ,incomingDependentArtifacts
   LIMIT 50
```

### Outgoing Package Dependencies (Efferent Couplings)

https://101.jqassistant.org/calculate-metrics/index.html

```cypher
   MATCH (p:Package)
MATCH (p)-[:CONTAINS]->(it:Java:Type)-[r:DEPENDS_ON]->(et:Java:Type)<-[:CONTAINS]-(ep:Package)<-[:CONTAINS]-(ea:Artifact)
OPTIONAL MATCH (it)-[:DEPENDS_ON]->(eti:Interface)
   WHERE p <> ep
    WITH p
        ,COUNT(et) AS outgoingDependencies
        ,COUNT(DISTINCT et)  AS usedTypes 
        ,COUNT(DISTINCT eti) AS usedInterfaces // also included in usedTypes
        ,COUNT(DISTINCT ep)  AS usedPackages
        ,COUNT(DISTINCT ea)  AS usedArtifacts
        ,SUM(r.weight)       AS outgoingDependenciesWeight
   ORDER BY outgoingDependencies DESC
  RETURN p.fqn AS packageName
        ,outgoingDependencies, usedTypes, usedInterfaces, usedPackages, usedArtifacts, outgoingDependenciesWeight
   LIMIT 50
```

### Outgoing Class Dependencies (Efferent Couplings)

https://101.jqassistant.org/calculate-metrics/index.html

```cypher
//Outgoing Class Dependencies
   MATCH (p:Package)
MATCH (p)-[:CONTAINS]->(it:Java:Type)-[r:DEPENDS_ON]->(et:Java:Type)<-[:CONTAINS]-(ep:Package)<-[:CONTAINS]-(ea:Artifact)
OPTIONAL MATCH (it)-[:DEPENDS_ON]->(eti:Interface)
   WHERE p <> ep
    WITH p, it
        ,COUNT(et)           AS outgoingDependencies
        ,COUNT(DISTINCT et)  AS outgoingDependentTypes 
        ,COUNT(DISTINCT eti) AS outgoingDependentInterfaces // included in usedTypes
        ,COUNT(DISTINCT ep)  AS outgoingDependentPackages
        ,COUNT(DISTINCT ea)  AS outgoingDependentArtifacts
        ,SUM(r.weight)       AS outgoingDependenciesWeight
   ORDER BY outgoingDependencies DESC
  RETURN p.fqn  AS packageName
        ,it.fqn AS className
        ,outgoingDependencies
        ,outgoingDependentTypes
        ,outgoingDependentInterfaces
        ,outgoingDependentPackages
        ,outgoingDependentArtifacts
        ,outgoingDependenciesWeight
   LIMIT 50
```

### Outgoing Class Method Calls

```cypher
MATCH (p:Package)
OPTIONAL MATCH (p)-[:CONTAINS]->(t:Java:Type)-[:DECLARES]->(m:Method)-[:INVOKES]->(dm:Method)<-[:DECLARES]-(dt:Java:Type)<-[:CONTAINS]-(dp:Package)
OPTIONAL MATCH (dm)<-[:DECLARES]-(dti:Interface)
WHERE p <> dp
 WITH p, t
     ,COUNT(dm) AS outgoingMethodCalls
     ,COUNT(DISTINCT dm) AS outgoingDistinctMethodCalls 
     ,COUNT(DISTINCT dt) AS outgoingMethodCallTypes 
     ,COUNT(DISTINCT dti) AS outgoingMethodCallInterfaceTypes
     ,COUNT(DISTINCT dp) AS outgoingMethodCallPackages
RETURN p.fqn AS packageName
      ,t.fqn AS className
      ,outgoingMethodCalls
      ,outgoingDistinctMethodCalls
      ,outgoingMethodCallTypes
      ,outgoingMethodCallInterfaceTypes
      ,outgoingMethodCallPackages
ORDER BY outgoingMethodCalls DESC
LIMIT 30
```

### Incoming Class Method Calls

```cypher
MATCH (p:Package)
OPTIONAL MATCH (p)-[:CONTAINS]->(t:Java:Type)-[:DECLARES]->(m:Method)<-[:INVOKES]-(dm:Method)<-[:DECLARES]-(dt:Java:Type)<-[:CONTAINS]-(dp:Package)
OPTIONAL MATCH (dm)<-[:DECLARES]-(dti:Interface)
WHERE p <> dp
 WITH p, t
     ,COUNT(dm) AS incomingMethodCalls
     ,COUNT(DISTINCT dm)  AS incomingDistinctMethodCalls 
     ,COUNT(DISTINCT dt)  AS incomingMethodCallTypes 
     ,COUNT(DISTINCT dti) AS incomingMethodCallInterfaceTypes
     ,COUNT(DISTINCT dp)  AS incomingMethodCallPackages
RETURN p.fqn AS packageName
      ,t.fqn AS className
      ,incomingMethodCalls
      ,incomingDistinctMethodCalls
      ,incomingMethodCallTypes
      ,incomingMethodCallInterfaceTypes
      ,incomingMethodCallPackages
ORDER BY incomingMethodCalls DESC
 LIMIT 50
```

### Set Incoming Package Dependencies

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
        ,COUNT(DISTINCT eti) AS incomingDependentInterfaces // also included in usedTypes
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

### Set Incoming Package Method Call Dependencies

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

### Set Outgoing Package Dependencies

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

### Set Outgoing Package Method Call Dependencies

https://101.jqassistant.org/calculate-metrics/index.html

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

### Calculate and set Instability = outgoing / (outgoing + incoming) Dependencies

https://101.jqassistant.org/calculate-metrics/index.html

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

### Count and set abstract classes

https://101.jqassistant.org/calculate-metrics/index.html

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

### Calculate and set Abstractness

https://101.jqassistant.org/calculate-metrics/index.html

```cypher
//Calculate and set Abstractness
 MATCH (p:Package)
  WITH p
      ,toFloat(p.numberAbstractTypes) / (p.numberTypes + 1E-38)  AS abstractness
   SET p.abstractness = abstractness
RETURN p.fqn AS packageName, p.numberAbstractTypes, p.numberTypes, abstractness
 ```

## Find cyclic dependencies

- https://101.jqassistant.org/manage-package-dependencies/index.html
- https://community.neo4j.com/t5/neo4j-graph-platform/detecting-cycles-using-cypher/td-p/16469

```cypher
// Cyclic Dependencies
MATCH (package:Package)-[:CONTAINS]->(type:Type)-[:DEPENDS_ON]->(dependentType:Type)<-[:CONTAINS]-(dependentPackage:Package)
MATCH (dependentPackage)-[:CONTAINS]->(cycleType:Type)-[:DEPENDS_ON]->(cycleDependentType:Type)<-[:CONTAINS]-(package)
WHERE package <> dependentPackage
RETURN package, dependentPackage
      ,type, dependentType, cycleType, cycleDependentType
 LIMIT 100
```

```cypher
//Cyclic Dependencies Concatenated
MATCH (package:Package)-[:CONTAINS]->(forwardSource:Type)-[:DEPENDS_ON]->(forwardTarget:Type)<-[:CONTAINS]-(dependentPackage:Package)
MATCH (dependentPackage)-[:CONTAINS]->(backwardSource:Type)-[:DEPENDS_ON]->(backwardTarget:Type)<-[:CONTAINS]-(package)
WHERE package <> dependentPackage
 WITH package, dependentPackage
     ,collect(DISTINCT forwardSource) + collect(DISTINCT backwardTarget) +
      collect(DISTINCT backwardSource) + collect(DISTINCT backwardTarget)
      AS dependencies
UNWIND dependencies  AS dependenciesUnwind
RETURN package, dependentPackage
      ,collect(DISTINCT dependenciesUnwind) AS dependencies
 LIMIT 5
```

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

## Packages Coupling

### Package usage between dependent artifacts

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

### Class usage between dependent packages and different artifacts

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

### Types that are used by many different packages

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

### Interface Segregation Candidates

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

## Community Detection with the Louvain Method

Community Detection is not only interesting when applied to social networks,
it can also give valuable insights to code dependencies. Are there packages
that highly depend on each other? Are there in contrast packages that are loosely coupled?
Is it possible to find communities, that have a high internal cohesion and rather low coupling to other communities?

The Graph Data Science Library from Neo4j provides different algorithms fore that. The Louvain Method is one of these that detects communities by optimizing Modularity. For more details see: [Louvain](https://neo4j.com/docs/graph-data-science/current/algorithms/louvain)

### Prerequisite: Add weight property to Package DEPENDS_ON Relationship

We'll have a look at the package dependencies. For that, the `DEPENDS_ON` relationship between package nodes is needed. Use the following command to generate package dependency relationships for all given artifacts, if they don't exist yet.

```shell
jqassistant.sh analyze -concepts classpath:Resolve dependency:Package
```

Execute the following Cypher statement afterwards to sum up the weights of all class dependencies and write them into the new relationship property `weights`. The weight represents the strength of the dependency which leads to better results in many cases.

```cypher
// Add weight property to Package DEPENDS_ON Relationship 
 MATCH (sourcePackage:Package)-[:CONTAINS]->(sourceType:Type)-[typeDependency:DEPENDS_ON]->(dependentType:Type)<-[:CONTAINS]-(dependentPackage:Package)
 MATCH (sourcePackage)-[packageDependency:DEPENDS_ON]->(dependentPackage)
 WHERE sourcePackage.fqn <> dependentPackage.fqn
  WITH packageDependency
      ,sourcePackage.fqn                 AS sourcePackageName
      ,dependentPackage.fqn              AS dependentPackageName
      ,SUM(typeDependency.weight)        AS packageDependencyWeight
      ,COUNT(dependentType.fqn)          AS dependentTypes
   SET packageDependency.weight = packageDependencyWeight
RETURN sourcePackageName
      ,dependentPackageName
      ,packageDependencyWeight
      ,dependentTypes
```

```cypher
// Add weight property for Interface Dependencies to Package DEPENDS_ON Relationship
 MATCH (sourcePackage:Package)-[packageDependency:DEPENDS_ON]->(dependentPackage:Package)
 OPTIONAL MATCH (sourcePackage)-[:CONTAINS]->(sourceType:Type)-[typeDependency:DEPENDS_ON]->(dependentInterface:Interface)<-[:CONTAINS]-(dependentPackage)
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

### Delete the Projection Graph

If there is already a projection with the name `package-dependencies`, then it needs to be deleted to start over with a new one. This step can be skipped when there is no projection with that name. The `false` parameter assures, that no error occurs when the grpah with the given name is already deleted.

For more details see [Removing graphs](https://neo4j.com/docs/graph-data-science/current/graph-drop).

```cypher
//Community Detection: Delete projection
CALL gds.graph.drop('package-dependencies', false)
```

### Create a Projection Graph

Create a projection graph from the original one named `package-dependencies` with all `Package` labeled nodes
and their `DEPENDS_ON` relationship. By setting the orientation to `UNDIRECTED`, the direction of the dependency relationship is removed, which leads to better results for this algorithm.
Include the relationship property `weight` to improve the Louvain Method results.

For more details see [Louvain Examples](https://neo4j.com/docs/graph-data-science/current/algorithms/louvain/#algorithms-louvain-examples).

```cypher
//Community Detection: Create undirected projection
CALL gds.graph.project('package-dependencies', 'Package', 
    {
        DEPENDS_ON: {
            orientation: 'UNDIRECTED'
        }
    },
    {
        relationshipProperties: ['weight']
    }
)
```

### Estimate Memory Consumption

[Memory Estimation](https://neo4j.com/docs/graph-data-science/current/algorithms/louvain/#algorithms-louvain-examples-memory-estimation) returns the required memory for the algorithm as
well as some other values.

```cypher
//Community Detection Estimate Memory
CALL gds.louvain.write.estimate('package-dependencies', {
  maxLevels: 10,
  maxIterations: 10,
  tolerance: 0.0001,
  relationshipWeightProperty: 'weight',
  writeProperty: 'louvainCommunityId'
})
YIELD nodeCount, relationshipCount, bytesMin, bytesMax, requiredMemory
RETURN nodeCount, relationshipCount, bytesMin, bytesMax, requiredMemory
```

### Gather Statistics

Use the [Statistics Mode](https://neo4j.com/docs/graph-data-science/current/algorithms/louvain/#algorithms-louvain-examples-stats) to experiment with different parameters and compare their results.

As suggested in the YouTube Video [NetSci 06-2 Modularity and the Louvain Method](https://youtu.be/QfTxqAxJp0U), a modularity higher than 0.3 can be considered significant. Higher values refer to clearer distinguishable communities.

```cypher
//Community Detection: Louvain method statistics
CALL gds.louvain.stats('package-dependencies', {
  maxLevels: 10,
  maxIterations: 10,
  tolerance: 0.0001,
  relationshipWeightProperty: 'weight'
})
YIELD communityCount
     ,ranLevels
     ,modularity
     ,modularities
     ,communityDistribution
RETURN communityCount
      ,ranLevels
      ,modularity
      ,modularities
      ,communityDistribution
```

## Community Detection with the Leiden Algorithm

[Leiden](https://neo4j.com/docs/graph-data-science/current/algorithms/alpha/leiden)

### Run the Algorithm

Below is the Cypher statement to run the algorithm and show the results without side-effects without changing the projected or the original graph. It returns a list sorted by the analyzed `communityId` with the
packages and artifacts that belong to it.

```cypher
//Community Detection: Run Louvain algorithm
  CALL gds.louvain.stream('package-dependencies', {
    maxLevels: 10,
    maxIterations: 10,
    tolerance: 0.0001,
    relationshipWeightProperty: 'weight'
  })
 YIELD nodeId, communityId
  WITH communityId, gds.util.asNode(nodeId) AS package
 MATCH (package)<-[:CONTAINS]-(artifact:Artifact)
RETURN communityId
      ,COUNT(DISTINCT package)             AS countOfMembers
      ,collect(DISTINCT package.fqn)       AS packages
      ,collect(DISTINCT artifact.fileName) AS artifacts
 ORDER BY countOfMembers DESC, communityId ASC
```

### Write Property CommunityId

The [Write function](https://neo4j.com/docs/graph-data-science/current/algorithms/louvain/#algorithms-louvain-examples-write) is used to write the resulting `communityId` into a node property in the original graph as shown here:

```cypher
//Community Detection Louvain: Write property louvainCommunityId
  CALL gds.louvain.write('package-dependencies', {
    maxIterations: 10,
    tolerance: 0.00001,
    relationshipWeightProperty: 'weight',
    writeProperty: 'louvainCommunityId'
  })
 YIELD communityCount, modularity, modularities
 RETURN communityCount, modularity, modularities
```

### Add a label for every community

To be able to select different colors for different communities in the standard web view,
a label needs to be added. The following Cypher statement shows how this can be done. Note that the [Awesome Procedures On Cypher (APOC) Library](https://neo4j.com/labs/apoc/#_installation) needs to be installed prior to that.

```cypher
//Community Detection: Add LouvainCommunity+Id label to all packages 
 MATCH (package:Package) 
  WITH package
      ,'LouvainCommunity' + toString(package.louvainCommunityId) AS louvainLabelName
  CALL apoc.create.addLabels(package, [louvainLabelName])
  YIELD node
 RETURN node
```

## Other Queries

### Query kind/label of nodes

https://stackoverflow.com/questions/21555529/neo4j-counting-nodes-with-labels

```cypher
MATCH (n) 
RETURN count(labels(n)), labels(n);
```

## 1. Getting started with separate Neo4j database

- [Download and install jQAssistant for command line][jqassistant getting started]
- [Download neo4j Community Edition](https://neo4j.com/download-center/#community)
- Choose the long term support version (LTS) of [neo4j][neo4j] for compatibility. jQAssistant v1.11 doesn't seem to support neo4j v5 yet (early 2023). See https://neo4j.com/docs/upgrade-migration-guide/current/version-5/changelogs/procedures .
- Download [Neo4j Graph Data Science](https://neo4j.com/download-center/#community) and copy it into the plugin directory of your local neo4j installation.
- As described in https://neo4j.com/labs/apoc/#_installation, copy the JAR file with `apoc` in its name from the `labs` folder into the plugins folder of your local neo4j installation.

## 2. Getting started

As described [here][jqassistant getting started], all you need to get started with [jQAssistant][jqassistant getting started] is:

- [Download it][jqassistant getting started]
- Scan for example the lib directory of jQAssistant itself using `bin\jqassistant.cmd scan -f lib` (Windows) or `bin/jqassistant.sh scan -f lib` (Linux)
- Use the following command to enrich the data with some additional properties and relationships:

  ```shell
  bin/jqassistant.sh analyze -concepts classpath:Resolve dependency:Package dependency:Artifact java:JavaVersion
  ```
  
- Start the web application to display and query the collected data using `bin\jqassistant.cmd server` (Windows) or `bin/jqassistant.sh server` (Linux)
- Open [http://localhost:7474](http://localhost:7474)
- Sign in without user and password
- Click on the database symbol on the top left corner

## 2. Get familiar with the data

Have a look at part 1 to get familiar with the data and query dependencies between artifacts (JARs): https://joht.github.io/johtizen/data/2021/02/21/java-jar-dependency-analysis.html

## Summary

<br>

---

## References

- [About jQAssistant][jqassistant]
- [Getting started with jQAssistant][jqassistant getting started]
- [Artifact Scanner of jQAssistant][jqassistant artifact scanner]
- [About Neo4j][neo4j]
- [Neo4j Cypher Manual][neo4j cypher manual]
- [Comparing SQL with Cypher][neo4j sql to cypher]
- [Query matching relationship with Neo4j][neo4j match relationship]
- [Update relationships with Neo4j][neo4j merge relationship]
- [Variable length relationships with Neo4j][neo4j variable length relationship]
- [Cartesian product][cartesian product]

[jqassistant]: https://jqassistant.org
[jqassistant getting started]: https://jqassistant.org/get-started/
[jqassistant artifact scanner]: http://jqassistant.github.io/jqassistant/doc/1.8.0/#_artifact_scanner
[neo4j]: https://neo4j.com
[neo4j sql to cypher]: https://neo4j.com/developer/cypher/guide-sql-to-cypher/
[neo4j cypher manual]: https://neo4j.com/docs/cypher-manual/current/
[neo4j match relationship]: https://neo4j.com/docs/cypher-manual/current/clauses/match/#match-on-rel-type
[neo4j merge relationship]: https://neo4j.com/docs/cypher-manual/current/clauses/merge/#merge-merge-on-a-relationship
[neo4j variable length relationship]:https://neo4j.com/docs/cypher-manual/current/clauses/match/#varlength-rels
[cartesian product]: https://en.wikipedia.org/wiki/Cartesian_product

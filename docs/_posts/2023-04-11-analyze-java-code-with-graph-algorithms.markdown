---
layout: post
title:  "Analyze java code with graph algorithms (Part 3)"
date:   2023-04-11 08:00:00 +0100
categories: data
tags: jqassistant neo4j graph data cypher java dependency
author: JohT
discussions-id: 42
---

In this third part of the series we'll have a look at how to use graph algorithms of the [Neo4j Graph Data Science Library][neo4j_graph_data_science_library] to analyze the structure of java code.
It follows up [Analyze java dependencies with jQAssistant][part1] and [Analyze java package metrics in a graph database][part2].

### Table of Contents

{:.no_toc}

1. A markdown unordered list which will be replaced with the table of contents.
{:toc}

## Setup

The setup of [jQAssistant][jqassistant] is similar to [Part 1][part1]. However, to be able to use the [Neo4j Graph Data Science Library][neo4j_graph_data_science_library], a separate installation of [Neo4j Community Edition 4.4 (Long Term Support)][neo4j_community_server] is required. The reason for that is that the currently embedded version 3.x of Neo4j in jQAssistant (3.11) isn't supported by the graph data science library. Details see [Neo4j Graph Data Science Supported Neo4j Versions][neo4j_graph_data_science_versions]. The currently newest version 5.x of Neo4j can't be used either, because [jQAssistant][jqassistant] support it.

- Download and install [jQAssistant for the command line][jqassistant getting started].

- Download and install [Neo4j Community Edition 4.4 (Long Term Support)][neo4j_community_server].

- Download [Neo4j Graph Data Science](https://neo4j.com/download-center/#community) and copy it into the plugin directory of your  Neo4j installation.

- Download [Neo4j Awesome Procedures On Cypher (APOC) Plugin][neo4j_apoc] as described in [APOC Installation][neo4j_apoc_installation] for Neo4j Server Version 4.x. and copy it into the plugin directory of your Neo4j installation.

- As described in https://neo4j.com/labs/apoc/#_installation, copy the JAR file with `apoc` in its name from the `labs` folder into the plugins folder of your local neo4j installation.

- Start Neo4j with `bin/neo4j start` and assure that is running with `bin/neo4j status`. Java 11 is [required][neo4j_requirements].

- Scan your Java artifacts (jar, war, ear) for example in the directory `myModules` with this command:

   ```shell
   bin/jqassistant.sh scan -storeUri bolt://localhost:7687 -storeUsername neo4j -storePassword neo4j -reset -f myModules
   ```

- Execute the following command to enrich the scanned data with further relationships:

   ```shell
   bin/jqassistant.sh analyze -storeUri bolt://localhost:7687 -storeUsername neo4j -storePassword neo4j -concepts classpath:Resolve dependency:Package dependency:Artifact java:JavaVersion
   ```

- For Windows replace `bin\jqassistant.cmd` by `bin/jqassistant.sh`.

- Open [http://localhost:7474](http://localhost:7474) and sign in with your Neo4j credentials (initial neo4j/neo4j)

- Click on the database symbol on the top left corner

- Read more about how to write Cypher queries in the [Neo4j Cypher Manual][neo4j cypher manual]

## Prerequisites

### Check Graph Data Science library Version

Execute the following Cypher query to get the currently installed version of the plugin [Neo4j Awesome Procedures On Cypher (APOC)][neo4j_apoc]. It should be at least 4.4.

```Cypher
RETURN gds.version() AS GDS_Version;
```

To get more details execute:

```Cypher
CALL gds.debug.sysInfo()
```

### Check APOC library Version

Execute the following Cypher query to get the currently installed version of the plugin [Neo4j Awesome Procedures On Cypher (APOC)][neo4j_apoc]. It should be at least 4.4.

```Cypher
RETURN apoc.version() AS APOC_Version;
```

### Add weight properties to Package DEPENDS_ON Relationship

Assure that `jqassistant analyze -concepts classpath:Resolve dependency:Package` in the [Setup](#setup) section succeeded.
Then execute the following Cypher queries from [Part 2][part2] of this blog series to add weight properties to DEPENDS_ON relationships of Package nodes.

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

## Community Detection

With the structure of code represented as a graph,
Community Detection can be used to find cohesive groups of classes, packages or artifacts.
Their members are highly dependent on each other internally and significantly less coupled to other groups.

This can give valuable insights about the code structure:

- Reveal unintended coupling
- Recommend code that could be moved
- Show code that might be affected by a change
- Support modularization
- Help finding code that can be split out

### Louvain Algorithm

*Modularity* is a commonly used expression to measure how cohesive a group of nodes is.

Community Detection is not only interesting when applied to social networks,
it can also give valuable insights to code dependencies. Are there packages
that highly depend on each other? Are there in contrast packages that are loosely coupled?
Is it possible to find communities, that have a high internal cohesion and rather low coupling to other communities?

The Graph Data Science Library from Neo4j provides different algorithms fore that. The Louvain Method is one of these that detects communities by optimizing Modularity. For more details see: [Louvain](https://neo4j.com/docs/graph-data-science/current/algorithms/louvain)

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

## Summary

<br>

---

## Notes

Side Note: Ensure all links are working before publishing them.

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
- Community detection of the countries of the world with Neo4j Graph Data Science: https://towardsdatascience.com/community-detection-of-the-countries-of-the-world-with-neo4j-graph-data-science-4d3a022f8399

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

---

## ChatGPT

### Louvain vs. Leiden Algorithm (ChatGPT)

Both Louvain and Leiden algorithms are community detection algorithms used to identify clusters or communities in a network or graph. The Louvain algorithm was proposed by Vincent Blondel et al. in 2008, and the Leiden algorithm was proposed by Traag et al. in 2019.

The main difference between the two algorithms is in their approach to optimizing modularity, which is a measure of the quality of the community structure in a graph. The Louvain algorithm is a two-step algorithm. In the first step, the algorithm optimizes the modularity by merging the nodes that are in the same community. In the second step, the algorithm builds a new graph where the communities detected in the first step are nodes, and the links between them are the weights of the edges between the original nodes. The algorithm then repeats the first step on this new graph until no more improvement in modularity is possible.

The Leiden algorithm, on the other hand, uses a different optimization algorithm called the Leiden method. This algorithm is based on the idea of local moves, where the algorithm makes small changes to the community structure and evaluates whether the modularity has improved. If it has, the move is accepted, and the algorithm repeats the process. If not, the move is rejected, and the algorithm tries a different move. This process continues until no more improvement in modularity is possible.

In summary, the Louvain algorithm is a two-step algorithm that optimizes modularity by merging nodes into communities and building a new graph, while the Leiden algorithm uses local moves to optimize modularity directly on the original graph. The Leiden algorithm has been shown to be more accurate than the Louvain algorithm on some datasets, but the Louvain algorithm is generally faster and more scalable.

### Leiden Algorithm Accuracy (ChatGPT)

The Leiden algorithm has been shown to be more accurate than the Louvain algorithm on graphs with heterogeneous degree distributions, where the degrees of the nodes in the graph vary significantly. This is because the Louvain algorithm tends to merge small communities into larger ones, which can result in the loss of small but important communities in the graph.

The Leiden algorithm's local move strategy allows it to detect these small communities and keep them separate, leading to better community detection results on graphs with heterogeneous degree distributions. In addition, the Leiden algorithm has been shown to be more robust to noise and small perturbations in the graph, making it a good choice for real-world graphs that may have noise or missing data.

However, it's worth noting that the performance of the Leiden algorithm can depend on the specific graph and community structure being analyzed. In some cases, the Louvain algorithm may still be the better choice in terms of accuracy or efficiency. As with any algorithm, it's important to evaluate the performance of both algorithms on a given graph to determine which is the best fit for the specific problem at hand.

### Community Detection Algorithm Performance Measurement (ChatGPT)

here are several metrics that can be used to measure the performance of a community detection algorithm, depending on the specific goals of the analysis. Here are some commonly used metrics:

Modularity: Modularity is a measure of the quality of the community structure in a graph. It compares the number of edges within communities to the number of edges between communities, with a higher modularity indicating a better community structure. Modularity is often used to optimize community detection algorithms.
Normalized Mutual Information (NMI): NMI is a measure of the similarity between the true community structure of a graph and the community structure detected by an algorithm. It ranges from 0 to 1, with a higher NMI indicating a better match between the true and detected community structures.
Adjusted Rand Index (ARI): ARI is another measure of the similarity between the true and detected community structures, but it accounts for chance agreement between the two structures. A higher ARI indicates a better match between the two structures.
Coverage: Coverage is a measure of the percentage of nodes in the graph that are assigned to a community by the algorithm. A higher coverage indicates that the algorithm is assigning more nodes to communities.
Conductance: Conductance is a measure of the quality of a community in a graph. It compares the number of edges within a community to the number of edges between the community and the rest of the graph, with a lower conductance indicating a better community structure.
F1-score: The F1-score is a measure of the balance between precision and recall in community detection. It ranges from 0 to 1, with a higher F1-score indicating a better balance between precision and recall.
It's important to note that different metrics may be appropriate for different types of graphs and community structures. Therefore, it's recommended to evaluate the performance of community detection algorithms using multiple metrics to get a comprehensive understanding of their performance.

### Modularity (ChatGPT)

Modularity is a measure of the degree to which a graph can be divided into subgraphs or modules. In the context of graph algorithms, modularity is a way to evaluate the quality of a partition of a graph into communities or clusters. A community is a group of nodes that are more densely connected to each other than to the rest of the graph.

Modularity measures the difference between the number of edges within communities and the number of edges that would be expected by chance in a null model where edges are randomly distributed. A high modularity score indicates a strong tendency for nodes to be connected to others within their community rather than to nodes in other communities.

Many graph algorithms, such as community detection and clustering algorithms, aim to identify communities or clusters within a graph. Modularity can be used as an objective function to optimize the partition of a graph into communities. Algorithms that maximize modularity tend to produce partitions with high intra-community connectivity and low inter-community connectivity, which are characteristic of strong communities.

Overall, modularity is an important measure for graph algorithms as it provides a quantitative measure of the quality of a partition and helps to identify the underlying modular structure of a graph.

---

## References

- [Part 1][part1]
- [Part 2][part2]
- [Neo4j Graph Data Science Library][neo4j_graph_data_science_library]
- [Neo4j Graph Data Science Supported Neo4j versions][neo4j_graph_data_science_versions]
- [Neo4j Download Center Community Server][neo4j_community_server]
- [Neo4j System requirements][neo4j_requirements]
- [Neo4j Awesome Procedures On Cypher (APOC)][neo4j_apoc]
- [Neo4j Awesome Procedures On Cypher (APOC) Installation][neo4j_apoc_installation]#
- [An Improved Louvain Algorithm for Community Detection 2021 Zhang/Fei/Song/Feng][improved_lovain]

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

[part1]: https://joht.github.io/johtizen/data/2021/02/21/java-jar-dependency-analysis.html
[part2]: https://joht.github.io/johtizen/data/2023/01/26/java-package-metrics-analysis.html
[neo4j_graph_data_science_library]: https://neo4j.com/docs/graph-data-science/current
[neo4j_graph_data_science_versions]: https://neo4j.com/docs/graph-data-science/current/installation/supported-neo4j-versions
[neo4j_community_server]: https://neo4j.com/download-center/#community
[neo4j_requirements]: https://neo4j.com/docs/operations-manual/current/installation/requirements
[neo4j_apoc]: https://neo4j.com/labs/apoc
[neo4j_apoc_installation]: https://neo4j.com/docs/apoc/current/installation/#neo4j-server
[improved_lovain]: https://www.researchgate.net/publication/356502269_An_Improved_Louvain_Algorithm_for_Community_Detection

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

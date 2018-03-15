<div id="table-of-contents">
<h2>Table of Contents</h2>
<div id="text-table-of-contents">
<ul>
<li><a href="#org4a25a1f">1. Why Spark SQL?</a></li>
<li><a href="#orgd4a2d6e">2. Catalyst optimizer</a>
<ul>
<li><a href="#org5c76dff">2.1. Analysis of logical plan (Analyzer)</a>
<ul>
<li><a href="#org1fe713e">2.1.1. An attribute is called unresolved if we do not know its type or have not matched it to an input table ( or an alias)</a></li>
<li><a href="#orgbdba183">2.1.2. Spark SQL uses Catalyst rules and Catalog object that tracks the tables in all data sources to resolve these attributes.</a></li>
</ul>
</li>
<li><a href="#orge356809">2.2. Logical optimization (Optimizer)</a>
<ul>
<li><a href="#org5a84209">2.2.1. three kinds of operatorOptimizationRule</a></li>
<li><a href="#orga88d0bb">2.2.2. CombineFilters</a></li>
<li><a href="#org01e8ba9">2.2.3. <span class="done DONE">DONE</span> ReorderJoin</a></li>
<li><a href="#org3c710c2">2.2.4. 3-stage batch</a></li>
<li><a href="#orgdfe3472">2.2.5. Remaining questions</a></li>
</ul>
</li>
<li><a href="#orgf747dd8">2.3. <span class="todo TODO">TODO</span> Cost based optimization (CBO) (CostBasedJoinReorder.scala)</a></li>
<li><a href="#org95c193d">2.4. <span class="todo TODO">TODO</span> Logical plan to physical plan (SparkStrategies.scala)</a></li>
<li><a href="#org9ee1bab">2.5. Code generation</a>
<ul>
<li><a href="#org9b6b24d">2.5.1. Quasiquote</a></li>
</ul>
</li>
<li><a href="#org1ac21e6">2.6. Limits of Catalyst optimizer</a>
<ul>
<li><a href="#org2198246">2.6.1. Cannot optimize functional operation (such as filer)</a></li>
</ul>
</li>
</ul>
</li>
<li><a href="#orge4081be">3. Related questions</a>
<ul>
<li><a href="#orgd87ecf6">3.1. Why does my code on Spark SQL run slower than SQL server</a></li>
</ul>
</li>
<li><a href="#org6b4d556">4. References</a>
<ul>
<li><a href="#org6b12836">4.1. <span class="done DONE">DONE</span> Databricks blog on Catalyst</a></li>
<li><a href="#orge73537d">4.2. <span class="done DONE">DONE</span> Cost Based Optimizer in Apache Spark 2.2</a></li>
</ul>
</li>
</ul>
</div>
</div>
\#+startup hidestars indent showall


<a id="org4a25a1f"></a>

# Why Spark SQL?

Spark SQL aims to integrate relational data processing with Spark's functional programming API \cite{Armburst:2015}.

1.  Support relational processing both within Spark programs (on native RDDs) and on external data sources using a programmerfriendly API.
2.  Provide high performance using established DBMS techniques.
3.  Easily support new data sources, including semi-structured data and external databases amenable to query federation.
4.  Enable extension with advanced analytics algorithms such as graph processing and machine learning.


<a id="orgd4a2d6e"></a>

# Catalyst optimizer


<a id="org5c76dff"></a>

## Analysis of logical plan ([Analyzer)](file:///home/simon/Dropbox/git/spark/sql/catalyst/src/main/scala/org/apache/spark/sql/catalyst/analysis/Analyzer.scala)

-   Provides a logical query plan analyzer, which translates UnresolvedAttributes and UnresolvedRelations into fully typed objects using information in a SessionCatalog.
-   Provides a way to keep state during the analysis, this enables us to decouple the concerns


<a id="org1fe713e"></a>

### An attribute is called unresolved if we do not know its type or have not matched it to an input table ( or an alias)


<a id="orgbdba183"></a>

### Spark SQL uses Catalyst rules and Catalog object that tracks the tables in all data sources to resolve these attributes.


<a id="orge356809"></a>

## Logical optimization ([Optimizer)](file:///home/simon/Dropbox/git/spark/sql/catalyst/src/main/scala/org/apache/spark/sql/catalyst/optimizer/Optimizer.scala)


<a id="org5a84209"></a>

### [three kinds of operatorOptimizationRule](file:///home/simon/Dropbox/git/spark/sql/catalyst/src/main/scala/org/apache/spark/sql/catalyst/optimizer/Optimizer.scala)

-   Push down

-   Combine
    -   CombineFilters

-   Constant folding and strength reduction


<a id="orga88d0bb"></a>

### [CombineFilters](file:///home/simon/Dropbox/git/spark/sql/catalyst/src/main/scala/org/apache/spark/sql/catalyst/optimizer/Optimizer.scala)

-   [transform](file:///home/simon/Dropbox/git/spark/sql/catalyst/src/main/scala/org/apache/spark/sql/catalyst/trees/TreeNode.scala) (transformDown and transformUp)
    -   Returns a copy of this node where \`rule\` has been recursively applied to the tree.
    -   tranformDown or tranformUp is directionality is expected.
-   [CurrentOrigin](file:///home/simon/Dropbox/git/spark/sql/catalyst/src/main/scala/org/apache/spark/sql/catalyst/trees/TreeNode.scala)
    -   Provides a location for TreeNodes to ask about the context of their origin.  For example, which line of code is currently being parsed.

-   [PartialFunction](file:///home/simon/.m2/repository/org/scala-lang/scala-library/2.11.8/scala-library-2.11.8-sources.jar:scala/PartialFunction.scala)
    -   PF[A, B] where PF is defined at a subset of A (eg. divsiion)
    -   [partial function vs partially applied function](http://sandrasi-sw.blogspot.com/2012/03/understanding-scalas-partially-applied.html)

-   [mapChildren](file:///home/simon/Dropbox/git/spark/sql/catalyst/src/main/scala/org/apache/spark/sql/catalyst/trees/TreeNode.scala)
    -   Returns a copy of this node where \`f\` has been applied to all the nodes children.
        -   return original TreeNode if no change happens
        -   m: Map => m.mapValues
        -   d: DataType => d  // Avoid unpacking structs
        -


<a id="org01e8ba9"></a>

### [ReorderJoin](file:///home/simon/Dropbox/git/spark/sql/catalyst/src/main/scala/org/apache/spark/sql/catalyst/optimizer/Optimizer.scala)

-   Brief code base structure description of Spark SQL and catalyst
    -   [visit different kinds of logical plans](file:///home/simon/Dropbox/git/spark/sql/catalyst/src/main/scala/org/apache/spark/sql/catalyst/plans/logical/LogicalPlanVisitor.scala)
        -   most plans/oprators are defined in [basicLogicalOperators.scala](file:///home/simon/Dropbox/git/spark/sql/catalyst/src/main/scala/org/apache/spark/sql/catalyst/plans/logical/basicLogicalOperators.scala)
    
    -   [join.scala](file:///home/simon/Dropbox/git/spark/sql/catalyst/src/main/scala/org/apache/spark/sql/catalyst/optimizer/joins.scala)
        
        1.  [apply](file:///home/simon/Dropbox/git/spark/sql/catalyst/src/main/scala/org/apache/spark/sql/catalyst/optimizer/joins.scala)
            -   [ExtraceFiltersAndJoins](file:///home/simon/Dropbox/git/spark/sql/catalyst/src/main/scala/org/apache/spark/sql/catalyst/planning/patterns.scala)
                
                -   recursive call on flattenJoin to flatten the join operations
                -   [SQLConf](https://github.com/jaceklaskowski/mastering-spark-sql-book/blob/master/spark-sql-SQLConf.adoc)  [(source link)](file:///home/simon/Dropbox/git/spark/sql/catalyst/src/main/scala/org/apache/spark/sql/internal/SQLConf.scala) is a internal configuration store for parameters and hints used in Spark SQL. It offers the following methods to configure properties:
                    
                    -   get
                    -   set
                    -   unset
                    -   clear
                    
                    and also has the accessor methods to read the current value of a configuration property or hint.
                    
                    > scala> spark.version
                    > res0: String = 2.4.0-SNAPSHOT
                    > 
                    > import spark.sessionState.conf
                    > 
                    > // accessing properties through accessor methods
                    > scala> conf.numShufflePartitions
                    > res1: Int = 200
                    > 
                    > // setting properties using aliases
                    > import org.apache.spark.sql.internal.SQLConf.SHUFFLE<sub>PARTITIONS</sub>
                    > conf.setConf(SHUFFLE<sub>PARTITIONS</sub>, 8)
                    > scala> conf.numShufflePartitions
                    > res2: Int = 8
                    > 
                    > // unset aka reset properties to the default value
                    > conf.unsetConf(SHUFFLE<sub>PARTITIONS</sub>)
                    > scala> conf.numShufflePartitions
                    > res3: Int = 200
                    > 
                    > // You could also access the internal SQLConf using get
                    > import org.apache.spark.sql.internal.SQLConf
                    > val cc = SQLConf.get
                    > scala> cc == conf
                    > res4: Boolean = true
                
                -[StarSchemaDetection](file:///home/simon/Dropbox/git/spark/sql/catalyst/src/main/scala/org/apache/spark/sql/catalyst/optimizer/StarSchemaDetection.scala)
                
                -   [Cardinality](https://en.wikipedia.org/wiki/Cardinality_(data_modeling)) based heuristics
        
        2.[ createOrderedJoin](file:///home/simon/Dropbox/git/spark/sql/catalyst/src/main/scala/org/apache/spark/sql/catalyst/optimizer/joins.scala)
           -[partition](file:///home/simon/.m2/repository/org/scala-lang/scala-library/2.11.8/scala-library-2.11.8-sources.jar:scala/collection/TraversableLike.scala)
    
    -   [@tailrec](https://www.scala-lang.org/api/2.12.3/scala/annotation/tailrec.html)
        -   A method annotation which verifies that the method will be compiled with tail call optimization. If it is present, the compiler will issue an error if the method cannot be optimized into a loop.

1.  JoinType supported in Spark SQL

    -   [InnerLike extends JoinType](file:///home/simon/Dropbox/git/spark/sql/catalyst/src/main/scala/org/apache/spark/sql/catalyst/plans/joinTypes.scala)
    -   [Type converter between Scala and Catalyst](file:///home/simon/Dropbox/git/spark/sql/catalyst/src/main/scala/org/apache/spark/sql/catalyst/CatalystTypeConverters.scala)


<a id="org3c710c2"></a>

### [3-stage batch](file:///home/simon/Dropbox/git/spark/sql/catalyst/src/main/scala/org/apache/spark/sql/catalyst/optimizer/Optimizer.scala)

-   Batch is a set of rules


<a id="orgdfe3472"></a>

### Remaining questions

1.  [Why define SimpleTestOptimizer object?](file:///home/simon/Dropbox/git/spark/sql/catalyst/src/main/scala/org/apache/spark/sql/catalyst/optimizer/Optimizer.scala)

    -   Define this optimizer to use in the test code?

2.  Cost-based vs rule-based optimization

    **\*\***
    
    :ID:       f460162f-36da-4416-a39b-b21143127b27


<a id="orgf747dd8"></a>

## Cost based optimization (CBO) ([CostBasedJoinReorder.scala](file:///home/simon/Dropbox/git/spark/sql/catalyst/src/main/scala/org/apache/spark/sql/catalyst/optimizer/CostBasedJoinReorder.scala))


<a id="org95c193d"></a>

## Logical plan to physical plan ([SparkStrategies.scala](file:///home/simon/Dropbox/git/spark/sql/core/src/main/scala/org/apache/spark/sql/execution/SparkStrategies.scala))

-   Logical plan &#x2013;> SparkPlan &#x2013;> Query Plan &#x2013;> TreeNode &#x2013;> Product &#x2013;> scala.Any


<a id="org9ee1bab"></a>

## Code generation


<a id="org9b6b24d"></a>

### [Quasiquote](https://docs.scala-lang.org/overviews/quasiquotes/intro.html)


<a id="org1ac21e6"></a>

## Limits of Catalyst optimizer


<a id="org2198246"></a>

### Cannot optimize functional operation (such as filer)

-   ds.filter(p `> p.city =` "Boston")
-   it is impossible for Catalyst to introspect the lambda function
-   introspect scala function definition is not possible in Scala


<a id="orge4081be"></a>

# Related questions


<a id="orgd87ecf6"></a>

## [Why does my code on Spark SQL run slower than SQL server](https://stackoverflow.com/questions/45181920/how-to-tune-mapping-filtering-on-big-datasets-cross-joined-from-two-datasets)


<a id="org6b4d556"></a>

# References


<a id="org6b12836"></a>

## [Databricks blog on Catalyst](https://databricks.com/blog/2015/04/13/deep-dive-into-spark-sqls-catalyst-optimizer.html)

-   State "DONE"       from "STARTED"    <span class="timestamp-wrapper"><span class="timestamp">[2018-03-05 Mon 14:13]</span></span>


<a id="orge73537d"></a>

## [Cost Based Optimizer in Apache Spark 2.2](https://databricks.com/blog/2017/08/31/cost-based-optimizer-in-apache-spark-2-2.html)

-   State "DONE"       from "STARTED"    <span class="timestamp-wrapper"><span class="timestamp">[2018-03-05 Mon 13:49]</span></span>


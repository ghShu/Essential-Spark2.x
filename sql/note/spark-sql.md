<div id="table-of-contents">
<h2>Table of Contents</h2>
<div id="text-table-of-contents">
<ul>
<li><a href="#sec-1">1. Why Spark SQL</a></li>
<li><a href="#sec-2">2. Catalyst optimizer</a>
<ul>
<li><a href="#sec-2-1">2.1. Analysis (Analyzer)</a>
<ul>
<li><a href="#sec-2-1-1">2.1.1. An attribute is called unresolved if we do not know its type or have not matched it to an input table ( or an alias)</a></li>
<li><a href="#sec-2-1-2">2.1.2. Spark SQL uses Catalyst rules and Catalog object that tracks the tables in all data sources to resolve these attributes.</a></li>
</ul>
</li>
<li><a href="#sec-2-2">2.2. Logical optimization (Optimizer)</a>
<ul>
<li><a href="#sec-2-2-1">2.2.1. three kinds of operatorOptimizationRule</a></li>
<li><a href="#sec-2-2-2">2.2.2. CombineFilters</a></li>
<li><a href="#sec-2-2-3">2.2.3. ReorderJoin</a></li>
<li><a href="#sec-2-2-4">2.2.4. 3-stage batch</a></li>
<li><a href="#sec-2-2-5">2.2.5. Remaining questions</a></li>
</ul>
</li>
<li><a href="#sec-2-3">2.3. Logical plan to physical plan (SparkStrategies.scala)</a></li>
<li><a href="#sec-2-4">2.4. Code generation</a>
<ul>
<li><a href="#sec-2-4-1">2.4.1. Quasiquote</a></li>
</ul>
</li>
<li><a href="#sec-2-5">2.5. Limits of Catalyst optimizer</a>
<ul>
<li><a href="#sec-2-5-1">2.5.1. Cannot optimize functional operation (such as filer)</a></li>
</ul>
</li>
</ul>
</li>
<li><a href="#sec-3">3. References</a>
<ul>
<li><a href="#sec-3-1">3.1. Databricks blog on Catalyst</a></li>
</ul>
</li>
</ul>
</div>
</div>

\#+startup hidestars indent showall

# Why Spark SQL<a id="sec-1" name="sec-1"></a>

# Catalyst optimizer<a id="sec-2" name="sec-2"></a>



## Analysis ([Analyzer)](file:///home/simon/Dropbox/git/spark/sql/catalyst/src/main/scala/org/apache/spark/sql/catalyst/analysis/Analyzer.scala)<a id="sec-2-1" name="sec-2-1"></a>


-   Provides a logical query plan analyzer, which translates UnresolvedAttributes and UnresolvedRelations into fully typed objects using information in a SessionCatalog.
-   Provides a way to keep state during the analysis, this enables us to decouple the concerns

### An attribute is called unresolved if we do not know its type or have not matched it to an input table ( or an alias)<a id="sec-2-1-1" name="sec-2-1-1"></a>

### Spark SQL uses Catalyst rules and Catalog object that tracks the tables in all data sources to resolve these attributes.<a id="sec-2-1-2" name="sec-2-1-2"></a>

## Logical optimization ([Optimizer)](file:///home/simon/Dropbox/git/spark/sql/catalyst/src/main/scala/org/apache/spark/sql/catalyst/optimizer/Optimizer.scala)<a id="sec-2-2" name="sec-2-2"></a>



### [three kinds of operatorOptimizationRule](file:///home/simon/Dropbox/git/spark/sql/catalyst/src/main/scala/org/apache/spark/sql/catalyst/optimizer/Optimizer.scala)<a id="sec-2-2-1" name="sec-2-2-1"></a>


-   Push down

-   Combine
    -   CombineFilters

-   Constant folding and strength reduction

### [CombineFilters](file:///home/simon/Dropbox/git/spark/sql/catalyst/src/main/scala/org/apache/spark/sql/catalyst/optimizer/Optimizer.scala)<a id="sec-2-2-2" name="sec-2-2-2"></a>


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

### [ReorderJoin](file:///home/simon/Dropbox/git/spark/sql/catalyst/src/main/scala/org/apache/spark/sql/catalyst/optimizer/Optimizer.scala)<a id="sec-2-2-3" name="sec-2-2-3"></a>


-   [joins.scala](https://stackoverflow.com/questions/35130247/how-to-inject-traits-to-base-type-classes-to-use-them-in-generic-type-methods)
    -   [@tailrec](https://www.scala-lang.org/api/2.12.3/scala/annotation/tailrec.html)
        -   A method annotation which verifies that the method will be compiled with tail call optimization. If it is present, the compiler will issue an error if the method cannot be optimized into a loop.
-   [partition](file:///home/simon/.m2/repository/org/scala-lang/scala-library/2.11.8/scala-library-2.11.8-sources.jar:scala/collection/TraversableLike.scala)

### [3-stage batch](file:///home/simon/Dropbox/git/spark/sql/catalyst/src/main/scala/org/apache/spark/sql/catalyst/optimizer/Optimizer.scala)<a id="sec-2-2-4" name="sec-2-2-4"></a>


-   Batch is a set of rules

### Remaining questions<a id="sec-2-2-5" name="sec-2-2-5"></a>



1.  [Why define SimpleTestOptimizer object?](file:///home/simon/Dropbox/git/spark/sql/catalyst/src/main/scala/org/apache/spark/sql/catalyst/optimizer/Optimizer.scala)

    
    -   Define this optimizer to use in the test code?

2.  Cost-based vs rule-based optimization

    
    **\*\***

## Logical plan to physical plan ([SparkStrategies.scala](file:///home/simon/Dropbox/git/spark/sql/core/src/main/scala/org/apache/spark/sql/execution/SparkStrategies.scala))<a id="sec-2-3" name="sec-2-3"></a>


-   Logical plan &#x2013;> SparkPlan &#x2013;> Query Plan &#x2013;> TreeNode &#x2013;> Product &#x2013;> scala.Any

## Code generation<a id="sec-2-4" name="sec-2-4"></a>



### [Quasiquote](https://docs.scala-lang.org/overviews/quasiquotes/intro.html)<a id="sec-2-4-1" name="sec-2-4-1"></a>

## Limits of Catalyst optimizer<a id="sec-2-5" name="sec-2-5"></a>



### Cannot optimize functional operation (such as filer)<a id="sec-2-5-1" name="sec-2-5-1"></a>


-   ds.filter(p `> p.city =` "Boston")
-   it is impossible for Catalyst to introspect the lambda function
-   introspect scala function definition is not possible in Scala

# References<a id="sec-3" name="sec-3"></a>



## [Databricks blog on Catalyst](https://databricks.com/blog/2015/04/13/deep-dive-into-spark-sqls-catalyst-optimizer.html)<a id="sec-3-1" name="sec-3-1"></a>

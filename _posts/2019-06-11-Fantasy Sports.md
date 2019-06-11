## 了解如何分析来自专业体育联盟的统计数据是使用Apache Spark with SQL进行数据分析的一个有用的用例。本部分涵盖：使用Apache Impala（孵化）和Hue进行数据探索。

在本系列的第1部分中，我介绍了使用fantasy sports analytics作为探索Apache Hadoop生态系统的一个有指导意义的用例的主题。在那一期中，我们着重于数据处理，从[Basketball-Reference.com](https://www.basketball-reference.com/)中收集数据，用z分数和标准化z分数来丰富数据，分析NBA球员的相对价值。这一次，我们将继续在Zeppelin UI中使用Apache Hive交互地研究数据。这个例子将帮助说明Hive提供了一个存储层和用于数据处理和数据分析的工具

这篇文章的所有代码都可以通过[Github](https://github.com/sparkaochong/spark-nba-analysis)访问，并参考第1部分，了解我们到目前为止的数据处理概述。

### 交互式数据分析：寻找年龄和经验的趋势
上次，您学习了如何使用DataFrame并使用Apache Spark SQL查询它。这一次，您将学习如何问一些更困难的问题，并迅速得到答案与火花和黑斑羚。由于我们在数据处理方面做得很彻底，所以您只需要几个简单的查询就可以从数据中得到结果。这种应用程序是Apache Hadoop使用数据处理框架来减少交互查询复杂性的最佳实践的典型。

“年龄对球员的赛季有什么影响?“运动员只是人类，随着时间的推移，他们的技能会下降。全明星球员什么时候回归到普通球员的水平?我们能计算出玩家年龄增长时价值的预期收益/损失吗?我们将在这里集中讨论这个话题。

我们将同时考虑zTot和nTot，并考虑玩家的年龄和经验。后者可能很重要，因为在我们考虑的时间跨度内，球员加入联盟的年龄发生了变化。以前很少有球员翘大学，但现在不是了，现在他们被要求至少打一年。如果我们看到年龄和经验在数字上的差异，那将会很有趣。

我们从包含所有原始统计数据、z分数和标准化z分数的RDD开始。另一个需要考虑的数据是玩家的z分数和标准化z分数每年是如何变化的，因此我们将计算这两者每年的变化。我们将保存两组数据，一组是年龄值的键值对，另一组是经验值的键值对。(注意，在这个分析中，我们忽略了所有在1980年参赛的球员，因为我们没有足够的数据来确定他们的经验水平。)

```scala
//1.拿到需要计算的字段：名字，年龄，年份，经历，zToT,nToT
    val playerDF = spark.read.table(s"${db}.player")
    import spark.implicits._
    val ageOrExpStatsDS: Dataset[AgeOrExpStats] = playerDF.select($"name", $"year", $"age", $"experience".as("exp"), $"zToT", $"nToT")
      .sort($"name", $"exp".asc).map(row =>
      AgeOrExpStats(row.getAs[String]("name"), row.getAs[Int]("year"),
        row.getAs[Int]("age"), row.getAs[Int]("exp"),
        row.getAs[Double]("zToT"), row.getAs[Double]("nToT")))

    val nameGroupedDS: KeyValueGroupedDataset[String, AgeOrExpStats] = ageOrExpStatsDS.groupByKey(_.name)
    //2.计算每个人随着年龄的增长，他自身价值的变化趋势
    val deltaAgeOrExpStatsDS: Dataset[(ListBuffer[DeltaAgeOrExpStats], ListBuffer[DeltaAgeOrExpStats])] = nameGroupedDS.mapGroups { case (_, statsIterator) =>
      var previousZ = 0.0
      var previousN = 0.0
      val ageBuffer = new ListBuffer[DeltaAgeOrExpStats]()
      val expBuffer = new ListBuffer[DeltaAgeOrExpStats]()
      statsIterator.zipWithIndex.foreach { case (stats, index) =>
        val (deltaZ, deltaN) = if (index == 0) {
          (Double.NaN, Double.NaN)
        } else {
          (stats.zToT - previousZ, stats.nToT - previousN)
        }
        previousZ = stats.zToT
        previousN = stats.nToT
        ageBuffer += DeltaAgeOrExpStats(stats.age, previousZ, previousN, deltaZ, deltaN)
        expBuffer += DeltaAgeOrExpStats(stats.exp, previousZ, previousN, deltaZ, deltaN)
      }
      (ageBuffer, expBuffer)
    }
```

现在我们将处理年龄值对列表。定义了一个新函数AgeAndExpThrendAnalysis，它通过我们的正常过程将原始统计数据映射到DeltaAgeOrExpStats对象，并通过每个统计数据减少其数量。这给出了每个统计数据的汇总统计信息。我们将再次需要将我们的RDD转换为一个DataFrame。在本例中，我们将直接将其加载到Apache Hive中.

### 通过Apache Zeppelin可视化结果
现在让我们跳到Zeppelin看看我们的结果。我们将查询年龄表，利用Zeppelin的制图功能将结果可视化，在x轴上绘制年龄，在y轴上绘制平均z分数:

![](https://raw.githubusercontent.com/sparkaochong/picgo_repo/master/20190611203934.png)

**上图表示：** 每个年龄组的平均zTot

研究结果相当清楚:平均而言，年轻玩家在25岁左右之前很难做出有价值的贡献，28岁时达到顶峰，30岁出头创造出相当好的价值，然后在37岁开始迅速下降。如果你还记得我们之前看的不同年龄的季节记录的分布，大多数季节集中在22-32岁之间，这意味着两端的尾部处理的数据非常少，因此解释了巨大的波动。(年龄最小和最大的球员平均上场时间也更少，这影响了他们统计数据的价值[PTS, 3P, TRB, AST, STL, BLK]，导致z和n分数更低。我们的分析忽略了计算数值时播放的分钟数，但是同样的练习也可以对每36分钟播放的平均数据进行计算。一般来说，不常玩游戏的玩家对梦幻团队的贡献并不大，因此忽略游戏时间符合我们的目的。)

看看每个年龄段的nTot值，我们发现人们在20多岁时达到顶峰，在30多岁时开始下降，这是一个相似的模式。请注意，平均nTot比平均zTot要低得多，这可以解释为有更多低于平均水平的玩家，只有少数“明星”在大多数类别中都表现出色——这使得优秀的玩家非常罕见。(这可能是很难找到深度联赛的原因之一，因为深度联赛的球员超过了120-150名。1980年，我们有55名球员的z分数为正，2015年达到115分。)

![](https://raw.githubusercontent.com/sparkaochong/picgo_repo/master/20190611204522.png)

**上图表示：** 每个年龄组的平均nTot

通过观察z分数和标准化z分数的变化，我们注意到一名球员在26岁之前平均会继续提高他的游戏水平，然后开始下降:

![](https://raw.githubusercontent.com/sparkaochong/picgo_repo/master/20190611204700.png)

**上图表示：** 与前一年相比，每个年龄组的zTot都发生了变化(但是存在数据质量问题，需要继续分析)

![](https://raw.githubusercontent.com/sparkaochong/picgo_repo/master/20190611204843.png)

**上图表示：** 与前一年相比，每个年龄组的nTot都发生了变化

如果你想知道为什么平均球员在28岁时达到巅峰，但直到26岁才会继续变好，那么原因在于我们并没有从每个年龄层观察到相同的球员。回想一下玩家参与游戏的高峰期是24岁，其中有1626个赛季被记录在案。这意味着一开始我们计算的是新秀的平均z分数，但这些分数不包括在那些年的delta分数中，因为他们还没有完整的赛季记录。同样地，当球员开始退出联盟时，他们不再将delta计算考虑在内。25岁时，我们有1455个季节的记录，比24岁时减少了171个。这些球员大部分都是从联盟中被除名的(也就是最差的球员)，所以把他们除名会给这个年龄段的所有球员带来净收益，即使平均水平开始略有下降。

### 接下来我们来看看同样的指标，现在是根据经验来组织的:

![](https://raw.githubusercontent.com/sparkaochong/picgo_repo/master/20190611205048.png)

**上图表示：** 平均zTot的玩家经验水平

回想一下，通过观察经验，我们消除了球员进入联盟年龄的模糊性。这种方法可以更好地了解球员的寿命以及他们所承受的磨损。此外，我们记录了2738个赛季的零经验，并每年下降之后。(事实上，626名球员从未进入NBA的第二年!)大多数玩家在第7年左右达到顶峰，但回报为正值，并在第13年左右稳步下降。请注意，到第14年，我们已经有了196季的记录，所以我们真正面对的是一个精英群体，他们可以维持这么长时间的生产力。查看nTot也讲述了类似的故事，因此我们将省略它。看看德尔塔z分数，我们发现球员在四年后平均停止进步。

![image](21F552710C4445A8A5E9BC5C364C7728)

**上图表示：** 与前一年相比，zTot在玩家体验上的变化

请注意，在第4年，我们只有1225名球员，还不到我们首发球员的一半。

奇怪的是，只看年龄似乎表明玩家比只看经验的玩家拥有更长的成长周期。这一反常现象是由于球员进入联盟的年龄不同，通常在头几年有一个急剧增长，然后逐渐下降。如果我们看看每个年龄层的新秀数量，我们会发现只有100个18或19岁，400个20或21岁，超过1400个22或23岁。在那之后，它迅速下降。考虑到大多数球员在22岁或23岁进入联盟，平均水平将在4年内继续提高，平均水平似乎会在26岁或27岁时有所提高，这是我们在观察三角洲地区随年龄增长的数据时所看到的。这一事实强调了了解数据上下文以避免做出错误结论的重要性。考虑到这一点，我们将关注未来的经验。

另外，不要忘记我们这里说的是普通玩家——有些人违反了规则。让我们看几个例子，选择nTot作为度量。(zTot得出了类似的结论。)

### 谁是有史以来最棒的球员？
答案取决于我们看z分数还是标准化z分数。看看原始z分数，我们可以看到库里今年的表现是有史以来最好的。拉里•伯德(Larry Bird) 1987年的竞选活动紧随其后。我们也看到了其他伟大的球员，比如迈克尔·乔丹和凯文·杜兰特。

![](https://raw.githubusercontent.com/sparkaochong/picgo_repo/master/20190611205755.png)

**上图表示：** 分析计算出来的zToT 

![](https://raw.githubusercontent.com/sparkaochong/picgo_repo/master/20190611205814.png)

**上图表示：** 以柱状图的方式表示zToT

当我们转换到正常的z分数时，它变成了迈克尔乔丹秀:他记录了前10个赛季中的5个。

![](https://raw.githubusercontent.com/sparkaochong/picgo_repo/master/20190611210040.png)

**上图表示：** 分析计算出来的nToT 

![](https://raw.githubusercontent.com/sparkaochong/picgo_repo/master/20190611210136.png)

**上图表示：** 以柱状图的方式表示nToT

库里的2015-16赛季排名第六。这当然令人印象深刻，但他仍然落后于迈克尔乔丹的巅峰时期。库里在接下来的几年里可能会有所进步，但是考虑到他在联盟的任期，他更有可能从一个略微下降的趋势开始。

## 结论
在这篇文章中，我们使用Spark和Impala来确定一个球员在职业生涯中什么时候会达到巅峰。我们还计算了玩家在未来一年的价值变化。这些都是有价值的数据点，可以用来帮助确定哪些球员在赛季之间的价值会增加或减少。

[本案例参考博客](https://blog.cloudera.com/blog/2016/06/how-to-analyze-fantasy-sports-with-apache-spark-and-sql-part-2-data-exploration/)
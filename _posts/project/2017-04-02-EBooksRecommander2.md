---
layout: post
title: 协同过滤实战：网络小说推荐系统（二）
category: project
description: 基于用户和基于物品过滤算法的分别实现和利弊比较
---
项目源代码参见 [GitHub Repo](https://github.com/ZhiyuanLIPlus/EBooksRecommander)
## 算法
在完成了数据准备工作之后，我就可以开始对我的数据集实施协同过滤算法。在协同过滤推荐算法中，从算法实施方法上可以分成两种，一个是基于用户推荐，一个是基于物品推荐。在这次项目里，因为代码上的工作量也不是很大，所以我就分别实现了这两种算法并进行了比较。

### 度量方法
无论是基于用户过滤还是基于物品过滤，算法当中最重要的一部就是计算两位用户或者两个物品之间的相似度。
在这个项目里，我也是分别尝试了比较流行的三种方法作为度量标准。在每种计算的最后，我都将计算出的相似性值缩放到[0, 1]的范围中，便于后面的计算。

[欧氏距离](https://en.wikipedia.org/wiki/Euclidean_distance "Euclidean distance")

[皮尔逊相关性系数](https://en.wikipedia.org/wiki/Pearson_correlation_coefficient "Pearson correlation coefficient")

[谷本系数](https://docs.tibco.com/pub/spotfire/6.5.3/doc/html/hc/hc_tanimoto_coefficient.htm "Tanimoto Coefficient")

### 基于用户过滤
基于用户过滤的算法步骤大致为：
- 首先读取UserPrefs.txt中记录的个人偏好，以 {书名：评分} 的字典形式存储

- 维护一张以{书名：总分}字典形式存储的总分字典和以 {书名：加权值之和} 字典形式存储的加权值之和字典。注：这里的加权值就是我们计算出来的相似度。

- 遍历整理后数据集，依次计算数据集中每位用户和自己的相似性：

  * 如果毫不相似或者负相似，我们直接跳过该用户
  
  * 如果有相似，遍历该用户评过分的书籍，跳过已经被自己评过的小说。
    * 对每一本小说，通过 {估计评分 = 该用户评分 * 相似性} 计算该小说的估计评分，加进总分字典中
    * 将相似性加进加权值之和字典中

- 遍历总分字典，对字典中的每个小说进行计算 {最终评分 = 总分 / 加权值之和}

- 对结果进行排序，输出结果

```
--- User Based: Sim Euclid ---
5.000000000000001:鹿鼎记
5.000000000000001:鸳鸯刀
5.000000000000001:飞狐外传
...
--- User Based: Sim Pearson ---
5.000000000000001:五代末年风云录
5.0:龙战骑士
5.0:鹿鼎记
...
--- User Based: Sim Tanimoto ----
5.000000000000001:龙的天空
5.000000000000001:都市之只手遮天
5.000000000000001:藏海花
...
--- User Based: Sim Tanimoto * 0.9 + Sim Euclid * 0.1 ---
5.000000000000001:贵圈真乱
5.000000000000001:汉末浮生记
5.000000000000001:欢乐英雄
...
```

这里只输出了每种算法的top3推荐，更多结果在[GitHub](https://github.com/ZhiyuanLIPlus/EBooksRecommander)上。

可以观察到，几乎每种推荐方法都给我推荐了10+以上的满分5分小说。这显然是非常不合理的的。

仔细分析一下数据集和算法，我最后发现：

* 算法方面，在使用**欧氏距离**和**皮尔逊相关性**计算两个用户相似性时候，我做的第一件事情就是计算两个用户共同打过分的小说。因为只有在共同打分的小说的基础上，这两种相似性度量计算才有意义。但是这样计算带来的问题是，如果用户A和用户B一共只有一本或者极少的共同打分项，而他们在这些极少共同项的打分上又恰好相同，这两种算法会认为这两位用户具有极高的相似度。这显然是不合理的。
* 而对于计算交并集比例的Tanimoto系数，上面的问题并不存在。导致Tanimoto系数也出现这样无厘头结果的原因在于我的数据集。事实上，我从优书网上抓取的数据集是一个十分典型的稀疏数据集。整个数据集大概包涵了9k多的用户数据和8k多的小说，这其中被大量用户共同评价过的小说的数量非常的少。在基于用户的过滤算法中，这样的稀疏数据集是很难有好的推荐效果的。
>   比如我们想给用户甲做推荐。通过计算，用户甲与用户乙的相似度为0.5，而用户乙给一本用户甲没有看过的小说A打了5分，而这本小说A也只有用户乙一个人打过分。那么在基于用户算法中，小说A的最终得分就会是{最终评分 = 5*0.5 / 0.5 = 5}。这显然是非常不合理的。
 
### 基于物品过滤
针对上面的问题，我又重新对数据集实现了基于物品的算法。步骤大致为：
- 读取UserPrefs.txt中记录的个人偏好，以 {书名：评分} 的字典形式存储

- 倒置存储数据集的嵌套字典，将 {用户：{书名：评分}} 的嵌套形式改为 {书名：{用户：评分}} 

```python
{
    'BookA':{
                'UserA': 3.5,
                'UserB': 5
            }
    'BookB':{
                'UserC': 2,
                'UserE': 1
            }
    #...etc..
}
```

- 遍历数据集字典，对每本小说与其他小说进行相似性计算，对结果进行排序，选取其中前n个物品，以 {书名: [(相似书名，相似性)*n]} 的字典列表形式存储，因为这里的计算量十分大，我也在最后使用了pickle对计算结果序列化，下次重新推荐时候可以直接调用，避免重复计算。

- 维护一张以 {书名：总分} 字典形式存储的总分字典和以 {书名：加权值之和} 字典形式存储的加权值之和字典。

- 遍历个人偏好，对偏好中的每一本小说进行如下计算：

  * 遍历该小说的topN相似性字典，对其中的每一本小说通过 {估计评分 = 个人偏好评分 * 相似性} 计算该小说的估计评分，加进总分字典中，也将相似性加进加权值之和字典中。

- 遍历总分字典，对字典中的每个小说进行计算 {最终评分 = 总分 / 加权值之和}

- 对结果进行排序，输出结果

```
--- Item Based: Sim Euclid ---
5.0:龙语实用教程
5.0:龙战骑士
5.0:龙人祖庭
4.75:鼠佛记
4.5:（新番完）何以笙箫默
4.5:龙魂武士

--- Item Based: Sim Tanimoto ---
5.0:江山美色
4.734782608695652:家园
4.733264675592173:二鬼子汉奸李富贵（曲线救国续集）
4.728716097928842:蚁贼
4.5882039942254265:1911新中华
4.5:文艺生活
4.5:好莱坞制作
4.5:一八九三
4.298245614035087:北唐
4.2900124366210655:原始战记
4.275614996977939:上品寒士
4.242975970425139:临高启明
4.219908363355485:新宋
4.11156323288314:一世之尊
4.091261182693812:回到过去变成猫

--- Item Based: Sim Pearson ---
5.0:龙域
5.0:重生亚当
5.0:近身保镖
4.5:龙骸
4.5:黑暗者
4.5:黑暗信仰

--- Item Based: Sim Tanimoto * 0.9 + Sim Euclid * 0.1 ---
5.0:白目老师
5.0:江山美色
5.0:大唐之我是独孤凤
5.0:夏鼎
5.0:全能炼金师
4.795960241853223:1911新中华
4.500000000000001:这个电影我穿过
```

可以清楚的看见，尽管皮尔逊和欧式距离的度量方法还是有很多平分的情况，但是遍地5分的情况已经缓解了很多。而这中间Tanimoto系数的表现更是出色。

## 小结
### 基于用户 vs 基于物品
#### 时间消耗
在本次项目中，基于物品推荐算法的计算时间消耗是要远远大于基于用户推荐的，这也是为什么我在最后选择将计算结果序列化便于下次使用时候直接提取。
#### 推荐结果
对于我使用的稀疏数据集，毫无疑问，基于物品算法是要远远要优秀与基于用户推荐算法。而其中Tanimoto系统算出的推荐列表更是包含了很多我其实已经看过，并且评价还不错的小说。只不过我在准备个人偏好的时候忘记把这些小说写进UserPrefs.txt中了。（其实也不太可能把我看过的所有小说都写下来。。。
#### Some Remarks
基于用户推荐的算法简单明了，易于理解而且实现过程也不复杂。应该更适用于那些规模较小，密集但是变化非常频繁的内存数据集。而且，在某些业务场景中，获取用户的偏好往往也具有更实际的意义。

基于物品推荐的算法则稍微复杂了一点。初次计算topN相似列表时候计算量也很大。而且，在初次计算结束将结果序列化或者存储之后如果数据集发生了较大或者频繁的变动，推荐结果很有可能会失真。而为了适应这些变化对topN的更新又需要耗费大量的时间。但是它对于稀疏数据集的性能却远远要大于基于用户推荐算法。

### 度量方法
在这次项目中，Tanimoto系数的推荐效果是远远好于其他度量方法的。

第一个原因就像之前分析的那样，由于比较的用户或者小说之间的共有项太少而导致**欧氏距离**和**皮尔逊相关性**很容易算出一个较高的相似度。而Tanimoto系数就完全避免了这个问题。

另外，在数据来源方面，因为优书网是一个比较小众的网络文学评论网站。上面每个用户书单中的评分更多的像是推荐指数而不是对该小说的实际评分。这就导致了在一份书单中很容易出现大面积的5星评分。这给**欧氏距离**和**皮尔逊相关性**在计算相似性时也带来了很多噪音。

换句话来说，这样的数据其实也已经失去 {评分} 这个特征能够带来了计算意义。反而不如计算 {看过小说A的所有用户与看过小说B的所有用户更容易重合} 这样来的简单有效。

### 其他
目前的主流推荐算法中，除了协同过滤算法，比较流行的还有基于内容推荐。吴恩达教授在Coursera上的机器学习课程中就简单的介绍了一下这种算法。 大概的步骤就是学习用户偏好列表，基于其中物品的内容基因或者说是标签为用户建立一个偏好矩阵。比如，某些小说可以同时被 {剑与魔法，种田流，设定宏大，单女主}等等的偏好标签修饰。 然后用这个偏好矩阵去预测用户对于新的物品的可能偏好程度。

因为优书网的小说页面链接中并没有提供这些标签功能，所以这次项目中我也没有去尝试这种算法。但是，可以想见的是，在具体的业务场景中，我们应该不会只单独使用一种算法，更多的应该是多种推荐算法的加权混合。也希望以后有机会可以见识一下工业环境中推荐算法的应用。

### 其他的其他
这两天抽空看了一下系统给我推荐的《蚁贼》这本小说，竟然真的很对我的口味。不得不感慨一句，数学真是一门牛X的学科。
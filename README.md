# weipinhui
唯品会用户购买行为预测

文件说明：
---
data : 数据文件夹 <br>
cache ：缓存文件夹，保存中间结果 <br>
dataProcess : 数据简单分析：主要是统计数据 <br>
gen_feat : 特征提取 <br>
train.IPYNB  : 模型训练 <br>
sub.IPYNB : 数据预测 <br>

运行环境与工具：
---
Windows10 ;
python2.7 ;
xgboost 0.6 ;
numpy, pandas , sklearn 
<br>

# 思路简述

场景解析
---
这一步是对业务逻辑进行分析、抽象，以及后续算法的选择。<br>
任务简单描述：根据用户的历史上 3 个月的点击和购买数据、商品数据，预测未来 1 周内，用户对某些商品的购买概率。<br>
1.数据探查 <br>
用户点击购买数据、商品数据、预测数据都是表格型。<br>
用户点击购买数据一共5000多万条；除了日期是字符串类型，其他都是数值类型；基本是均匀分布在三个月。<br>
商品数据有200多万条；都是数值型数据。<br>
预测数据是未来一周的用户-商品对，最终目的就是给出用户购买对应商品的概率。有500多万条，都是数值型数据<br>
2.场景抽象 <br>
这是电商平台典型的商品推荐场景。<br>
3.算法选择 <br>
给出的数据都可以转化为数值型数据。而且每一条数据都有一个目标值：买或不买。任务是预测购买的概率。
所以可以选择基于数值运算、有监督、二分类机器学习算法。

数据预处理
---
这一步要把数据调整为适合的状态，也就是对算法的干扰最小，以提高最终算法的训练效果。<br>
  
  1.去燥<br>
  统计每个的点击量、购买量、点击购买率。表明一共的接近20的用户量；在这三个月的时间里
  约75%的用户点击次数小于347，约25%的用户一次都没有购买。把点击量大于1500且一次都
  没有购买的用户直接删除。满足条件的用户有546人，占总用户量的千分之3左右。一个人每天点击15个商品三个月的点击量就1500左右，
  大于1500一次都没有购买可能是网络爬虫等造成的干扰数据，所有把这些数据直接删除。<br>

特征工程
---
数据挖掘最关键的一步，特征工程的质量很大程度上决定训练模型的预测效果。<br>
这一步工作是要从原始数据中构造出能够表达数据含义的特征。<br>

1.特征抽象<br>
把源数据抽象成算法可以理解的数据。日期是字符串型数据，以最后一天为基准，取具体某一天的日期与最后一天侧差值。<br>

2.特征衍生<br>
对现有特征进行衍生，生成新的具有含义的特征。一般需要从业务的视角（数据的理解和业务场景的认识）进行特征衍生。<br>

 - 用户行为特性：<br>
 用户购买量，点击购买率；购物频率
 - 商品行为特征：<br>
 商品的购买次数，点击购买率。二次购买率，平均消费间隔
 - 用户—商品对行为特性：<br>
 用户-商品对的点击频率。
 - 品牌的行为特性： <br>
 品牌的点击购买率；二次购买率。品牌对时间不是很敏感，时间区域间距可以大一些。
 - 用户 -品牌对的行为特性：<br>
 用户-品牌对的点击购买率，点击频率。<br>

 **注意：**以天为时间粒度，时间窗口向前滑动，取每个时间窗口的shuffle，构建数据集。时间窗口：用户在距离预测开始时间前3,8,21,34,89天 <br>
 
3.特征降维
4.特征重要性评估 


数据预处理
---
这一步要把数据调整为适合的状态，也就是对算法的干扰最小，以提高最终算法的训练效果。<br>
  
  2.数据过滤<br>
  用户ID，商品ID、日期是一种唯一的标识，并不具备描述行为特征的含义。所以可以过滤掉这些字段。<br>
  3.归一化<br>
  归一化数据可以去除量纲的影响，加快算法的收敛速度。对点击量、购买量；用户的购物频率；
  商品的二次购买率、平均消费间隔等非0到1范围内的数据归一化。<br>
  4.采样<br>
  利用历史数据，构造最后一周的用户-商品对的特征，训练模型，预测未来一周的用户-商品购买概率。<br>
  最后一周用户点击购买数据一共500多万条,其中用户购买的数据只有5多万条。正负样本比约为1:100，正负样本严重失衡
  结合分层采样和随机采样调节正负样本比例。<br>
  具体操作如下：首先将数据分为正样本（购买了）和负样本。然后对正样本过采样：复制4份正样本组合形成新的正样本。
  对负样本欠采样：执行250万无放回的随机采样。改善正负样本比例为1:10。组合正负样本作为训练数据。<br>
  
  训练五个模型，用五个模型的预测进行投票产生最终得预测结果。保证在减少了数据量的同时，又防止负样本信息的丢失。
  但是这样每一个模型都没有获取到所有的训练信息，但是预测的时候却要预测所有的，这样性能会下降，实践证明也是。
  所以还是通过负样本欠采样50%，正样本过采样500%，改善正负样本比例为1:10左右。

  
 
模型训练
---
- 线下训练数据：对历史数据的倒数第二周的用户-商品对，取倒数第二周之前的所有用户、商品、用户-商品的特征，用倒数第二周内是否购买作为标签。 <br>
- 线下预测的时候：对历史数据的倒数第一周的用户-商品对，取'01-10'到倒数第一周之前的所有用户、商品、用户-商品的特征，用倒数第一周内是否购买作为标签。 <br><br>

- 线上训练数据： 对最后一周用户-商品对，取'01-10'到'03-24'的所有用户、商品、用户-商品的特征，用最后一周用户是否购买作为标签。 <br>
- 线上预测：对预测周的用户-商品对，取'01-18'到'03-31'所有的用户、商品、用户-商品的特征，预测购买的概率，从17号开始是为了与训练时候的数据了保持一致 <br><br>

**线下测试的数据 和线上训练的数据 重合，可以重复利用**

算法与模型
---
算法选用xgboost， 线下预测的性能一直高于线上，现在的预测的得分最低0.15，而线上的得分一直在0.55左右，让人大跌眼镜。具体原因不详。

- 使用交叉验证评估模型性能
test集上并没有给出目标值，这正是我们要预测的值。我们无法在test集上评估模型的性能。所以使用交叉验证，将训练数据集分成两部分，一部分用于训练模型，一部分用于评估模型效果。

- 网格搜索，选择最优的模型参数
参数对模型的效果有重要影响。网格搜索是非常强大的超参数优化工具，用来找到超参数值的最优组合，从而进一步改善模型的性能。在网格搜索中也要用到交叉验证评估模型效果，查找时模型效果最好的参数。

- 模型状态
有一个很可能发生的问题是，我们不断地做feature engineering，产生的特征越来越多，用这些特征去训练模型，优化模型参数，使会对我们的训练集拟合得越来越好，同时也可能在逐步丧失泛化能力，从而在待预测的数据上，表现不佳，也就是发生过拟合问题。学习曲线可以帮助我们判断模型的状态。然后根据模型的过拟合和欠拟合采取相应的措施。


关键问题
---
数据非平衡问题（in-balance）
历史三个月数据数据中，每一周大概有500万条记录，其中正样本只有5万条，正负样本比为：100:1，正负样本比严重失衡。
很多应用中，正负样本是不均衡的，大多数对模型对正负样本比例是敏感的。


1. 改造分类器的训练数据 —— 过抽样或者欠抽样
具体来说，正负样本失衡的处理方法如下：

-  负样本 >> 正样本，且量都挺大： 对负样本 欠采样undersampling
- 负样本 >> 正样本，量都不大=>
  1. 采集更多的数据
  2. 负样本欠采样，正样本过采样oversampling（图像中镜像，旋转等也算）
  3. 修改损失函数，给正样本更大的权重。 

2. 代价敏感的学习（cost-sensitive learning）

3. 非均衡分类的性能度量：混淆矩阵，ROC曲线
分类性能度量指标：正确率，召回率及AUC
  



总结
---
第一次参加比赛，成绩很差，还有很多问题没有弄明白，但是也学到了很多。经历了机器学习算法的大致步骤：
场景解析——数据预处理——特征工程——模型训练——预测与评估。感谢那些将自己项目开源的人士，你们是小白入门不可或缺的引导者。



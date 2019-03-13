---
layout: post
title: 机器学习问题解决
description: "pyspark ml"
modified: 2017-09-01
tags: [PySpark, ML, AI]
categories: [PySpark, ML, AI]
---

机器学习，也称为统计学习，通过从数据集中统计出一定的规律，并利用规律对新数据进行预测。此类算法与人工编写问题解决的具体过程并让机器确定执行的方式不一样，只是通过统计方法得出规律，最后的结果是不确定的、概率的。从某种程度上说，机器可以自己找到问题的解决方案，而不是人为告知机器的，是人工智能的体现。但是有必要提出的是，机器学习只是实现人工智能的手段的一种，其本身还在发展之中，还有很多缺憾的地方，比如其结果并不能揭示因果关系，无法像人类智能一样进行推理（参考[Deep Learning: A Critical Appraisal](https://arxiv.org/abs/1801.00631)）。另一种手段是知识表示和推理，但是构建庞大的计算机可解释的知识库成本也很大， 关键是其推理算法的实现一般是NP难问题，无法实现工业级的应用。虽然各个手段都有缺陷，但技术一直在进步，希望有生之年可以见证人工智能的实现。

## Spark MLlib
虽然人工智能实现道阻且长，但是对于一些特定领域的问题，可以通过机器学习解决，比如广告预估、风险分析、推荐系统等。我们可以快速利用现阶段机器学习领域的研究成果，并将之应用到我们的系统之中。开源产品有很多，大规模分布式计算平台spark提供了机器学习算法库[MLlib库](https://spark.apache.org/docs/latest/ml-guide.html)，其简单的api设计可以让我们高效的实现想要解决的问题。下面通过一个完整的例子介绍一下spark-mllib的使用。

### 数据准备
机器学习解决问题的前提是有足够多的领域数据，通过进行原始数据采集、数据分析、数据处理、特征工程得到算法需要的训练数据集，而这部分工作也占用了大部分时间。数据分为数值型数据、类别数据，特征工程包括归一化、离散化、缺省值处理等。下面以分析一个用户是否是潜在购买对象为具体例子介绍下：

{% highlight Python %}
from pyspark.sql import SparkSession
from pyspark.ml.classification import *
from pyspark.ml.feature import *
from pyspark.ml import *
from pyspark.ml.evaluation import *
from pyspark.ml.tuning import *
from pyspark.sql.types import *
from pyspark.sql.functions import *

### 读取本地csv文件
data = spark.read.option("inferSchema", True).option("header", True).csv("file:///path/to/file")

### 特征列，label是标签列
feature_columns = [x for x in data.columns if x not in ['productId', 'userId', 'label']]

### 转换为数值类型
for c in feature_columns:
    data = data.withColumn(c, data[c].cast(DoubleType()))

### 填写默认值
data = data.na.fill(0.0, feature_columns)

### 转换类别特征列，转换原因后续介绍
to_array = udf(lambda s: ["feature=%s" % s], ArrayType(StringType()))
data = data.withColumn("educationArray", to_array("education"))
data = data.withColumn("carArray", to_array("car"))
data = data.withColumn("consumptionLevelArray", to_array("consumptionLevel"))
data = data.withColumn("marriageArray", to_array("marriage"))
data = data.withColumn("salaryArray", to_array("salary"))

### 将数据集分为训练数据和测试数据
splits = data.randomSplit([0.8, 0.2], seed=666)

train_data = splits[0]
test_data = splits[1]

### 连续值数据类型分桶离散化
splitAge = [0.0, 20.0, 30.0, 40.0, 50.0, 60.0, 70.0, 80.0, float("inf")]
bucketAge = Bucketizer(splits=splitAge, inputCol="age", outputCol="bucketedAge")

# 通过独热编码OneHot encoder，将类别特征转成n个二元特征。
# 因为类别特征是无序的，独立的，直接转换成数值会让某些分类器误认为存在序列关系
# 但是会让特征变的稀疏
ageEncoder = OneHotEncoder(inputCol="bucketedAge", outputCol="bucketedAgeVec")

# 但是spark的onehotencoder存在问题(train和test的种类必须一致，https://issues.apache.org/jira/browse/SPARK-13030), 否则在test上预测时会直接抛异常。
# 替代方案是用HashingTF，将字符串映射到数组索引
eductionHasher = HashingTF(numFeatures=2**4, inputCol="educationArray",
outputCol="educationVec")
carHasher = HashingTF(numFeatures=2**4, inputCol="carArray",
outputCol="carVec")
consumptionLevelHasher = HashingTF(numFeatures=2**4, inputCol="consumptionLevelArray",
outputCol="consumptionLevelVec")
marriageHasher = HashingTF(numFeatures=2**4, inputCol="marriageArray",
outputCol="marriageVec")
salaryHasher = HashingTF(numFeatures=2**4, inputCol="salaryArray",
outputCol="salaryVec")

## 转换成算法需要的向量
vectorAssembler = VectorAssembler(
    inputCols=[
        "bucketedAgeVec",
        "bucketedRiskAppetiteVec",
        "carVec",
        "consumptionLevelVec",
        "educationVec",
        "marriageVec",
        "salaryVec",
        "province",
        "riskLevel",
        "sex",
        ],
    outputCol="features")

## 特征标准化
vectorScaler = StandardScaler(inputCol="features", outputCol="scaledFeatures").setWithStd(True).setWithMean(True)
{% endhighlight %}

### 模型训练
生成训练数据之后，就可以用分类或回归算法进行模型训练了。对于分类问题，产业界目前效果最好的一般都是集成方法，比如随机森林(random-forest)、梯度下降决策树（gbdt）。[xgboost][https://homes.cs.washington.edu/~tqchen/pdf/BoostedTree.pdf]是gbdt的一个高效实现版本，通过不断拟合上一轮的残差来逼近最优解。这里直接用spark中的gbdt接口进行模型的训练：

{% highlight Python %}

gbt = GBTClassifier(labelCol="label", featuresCol="scaledFeatures", maxIter=20)

将所有transformer和estimator都放入pipeline之中，保证训练集合测试集的数据处理流程一致
pipeline = Pipeline(stages=[
                    bucketAge,
                    ageEncoder,
                    eductionHasher,
                    carHasher,
                    consumptionLevelHasher,
                    marriageHasher,
                    salaryHasher,
                    vectorAssembler,
                    vectorScaler,
                    gbt])

## 拟合训练数据
model = pipeline.fit(train_data)
{% endhighlight %}

### 预测和模型评估
模型训练出来之后，需要进行模型的评估，通过在测试集上验证模型的可靠性。评估指标有很多，比如精准率（precision）、召回率（recall）、AUC值。但通常是选择一个业务关心的指标进行评估和不断迭代，否则往往训练过程中多个指标会此消彼长，会让人很疑惑。

{% highlight Python %}

# 二元分类评估
evalutor = BinaryClassificationEvaluator(rawPredictionCol="prediction", labelCol="label", metricName="areaUnderROC")

# 在测试集上获得预测结果
prediction = model.transform(test_data)

# 计算模型的AUC值
auc = evaluator.evaluate(prediction)

{% endhighlight %}


## 总结
本文只是简单介绍了一个机器学习任务的完整流程，其中每一个步骤都需要深入研究，特征工程、模型选择、模型参数调整、模型评估等等都是很大的话题。但是值得一提的是，机器学习只是一种技术，需要找到对应的业务场景，才能发挥其最大的效果。




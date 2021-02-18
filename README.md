# Biaffine-BERT-NER

## Introduction

最近看到一些NER的论文，从传统的BIO预测变成了指针预测和span预测等，而且效果还不错。尤其看到一篇《[Named Entity Recognition as Dependency Parsing](https://www.aclweb.org/anthology/2020.acl-main.577/)》，将NER预测改为预测一个矩阵的方法，与我原来在做之江杯的评论比赛时实现过的关系预测非常接近，但是扩展到了可以预测多个类别，而且原来这个还有个专门的称呼叫Biaffine，于是自己借鉴原作者的实现自己改写了一下。

这里对原论文的实现做了多处简化：

- 纯粹基于bert进行finetune，不再利用fasttext、bert等做context embedding抽取
- 不区分char word的embedding，默认就是char
- 原来的论文中是要输入多句话的【上下文？】，这里默认都是一句话

简单来说，就是在bert的基础上，预测一个L\*L\*C的tensor，其中L为文本长度，C为实体类别数目。

## Results

在[CLUENER](https://github.com/CLUEbenchmark/CLUENER2020)进行了测试。配置参考train_ner.sh，使用bert-base进行训练，结果如下：

### 验证集

总体F1 - 0.81

具体每个类别：

```json
{
  "organization": 0.8104196816208394,
  "game": 0.8381601362862009,
  "name": 0.8828039430449068,
  "government": 0.8458498023715415,
  "address": 0.667590027700831,
  "movie": 0.8398576512455516,
  "book": 0.8271186440677967,
  "position": 0.8113207547169812,
  "company": 0.839142091152815,
  "scene": 0.7564766839378239
}
```

### 线上

对比其他模型：

| 模型     | <a href='https://www.cluebenchmarks.com/ner.html'>线上效果f1</a> |
|:-------------:|:-----:|
| Biaffine-BERT | 80.08 |
| Bert-base   |  78.82  |
| RoBERTa-wwm-large-ext | 80.42 |
| Bi-Lstm + CRF | 70.00 |

比BERT的baseline好一些。当然，看了一下排行榜的其他结果，最好的已经到了82.545【截止20210218】，差距还挺大的。


### Focal loss

配置：
```
gamma=3
learning_rate=3e-5
num_train_epochs=5.0 
```

训练集指标：

negative_accuracy = 99.96067, positive_accuracy = 74.00737

验证集指标：0.8119

线上指标：80.374

> 比单纯的CE效果好一丢丢

### Dice Loss

没有收敛，估计实现有问题，或者这个场景下不适合。

### Negative Sampling

 由于负样本过多，因此我们考虑对负样本进行采样，设置比例`neg_sample=0.3`。

训练集指标：

negative_accuracy = 99.93346, positive_accuracy = 83.07946

验证集指标：0.8109

线上未测试，不过个人觉得应该差别不大。

> 结果上看跟baseline基本上没什么差别，不过至少说明大量的负样本其实mask了之后对最终的效果也没有太大的影响，很多负样本是easy examples。因此，focal loss和dice loss这种理论上应该是有提升的。
> 不过我把训练的epoch设置成了5【训练集的准确率一直在提升】，学习率改大了一点，不知道训练更长时间会不会效果会更好一些。

### Position Info

加入了position的信息，结果如下：

negative_accuracy = 99.97583, positive_accuracy = 77.3444

验证集：0.8119

差别不大。

## Train & Inference

训练代码参考：train_ner.sh

预测代码参考：predict_ner.sh

> 没有把evaluation放在代码中，需要用户自己预测结果，然后跑一下score.py


## Reference

- [Named Entity Recognition as Dependency Parsing](https://www.aclweb.org/anthology/2020.acl-main.577/)
- [biaffine-ner](https://github.com/juntaoy/biaffine-ner)
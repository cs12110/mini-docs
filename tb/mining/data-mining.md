# 数据挖掘笔记

书籍地址: [A Programmer's Guide to Data Mining](http://guidetodatamining.com/)

## ![](imgs/book.png)

## 基础算法

本章节主要用于第二章节的算法说明的简单记录.

各种距离都是为了找出`nearest`的`neighbor`,找出相似度最高的用户/物品.

### 1. 曼哈顿距离和欧几里得距离

书本: `Manhattan Distance and Euclidean Distance work best when there are no missing values`

**曼哈顿距离和欧几里得距离都适用于,两个维度没有缺失值的情况下.**

下面的公式里面

- 当 r=1 时,为曼哈顿距离计算公式.
- 当 r=2 时,为欧几里距离计算公式(勾股定理).
- 当 r=3 时,为多维距离计算公式.

![](imgs/20181016140244.png)

**从公式可以看出来,曼哈顿距离越大则欧几里得距离也越大,距离越大相似度越低**

### 2. 皮尔逊相关系数

![imgs/20181016134418.png](imgs/20181016134418.png)

公式经过转换后,如下图所示:

![imgs/20181016134418.png](imgs/20181016134827.png)

书本: `It ranges between -1 and 1 inclusive. 1 indicates perfect agreement. -1 indicates perfect disagreement.`

**主要考虑数据的离散程度.值越接近于 1 时,相似度越高,越接近-1 时,越不相似.**

### 3. 余弦相似度

书本: `which is very popular in text mining but also used in collaborative filtering—cosine similarity`

**该相似度是非常流行的文本挖掘算法,比如全文检索之类.**

公式如下:

![imgs/20181016141354.png](imgs/20181016141354.png)

书本:`The cosine similarity rating ranges from 1 indicated perfect similarity to -1 indicate perfect negative similarity`

**值越接近于 1 时,相似度越高,越接近-1 时,越不相似.**

### 4. 计算算法的选取

Attention: **这个才是最重要的.**

![](imgs/20181016144548.png)

- 如果数据等级膨胀,可以使用`Pearson`相似度算法
- 如果数据所有值都具有值,可以使用欧式距离或者曼哈顿距离算法
- 如果数据比较稀松,可以使用余弦相似度算法

---

进度: 70/395

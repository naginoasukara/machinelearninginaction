## 第二章

kNN算法

### 1.上下文管理器
```python
#讲文本记录转化为numpy
def file2matrix(filename):
    fr = open(filename)
    numberOfLines = len(fr.readlines())         #get the number of lines in the file
    returnMat = zeros((numberOfLines,3))        #prepare matrix to return
    classLabelVector = []                       #prepare labels return   
    fr = open(filename)
    index = 0
    for line in fr.readlines():
        #去掉左右两边的空格，'\n'
        line = line.strip()
        #按照tab键进行分割
        listFromLine = line.split('\t')
        #将数据插入改列
        returnMat[index,:] = listFromLine[0:3]   
        classLabelVector.append(int(listFromLine[-1]))
        index += 1
    return returnMat,classLabelVector
```


此处:

```python
fr = open(filename)
```
文件的使用没有使用上下文管理器，存在文件关闭隐患，详见项目`python-progressing`，应改为：
```python
with open(filename) as fr:
```

### 2.`python2` 和 `python3` 版本中`dict`的函数不同

书中是`python2`版本，多年之前比较流行，但是现在已经不在维护了，所以还是使用`python3`版本

```python
def classify0(inX, dataSet, labels, k):
    #dataset的长度
    dataSetSize = dataSet.shape[0]
    #tile()讲一个list,dict等转化为m*n的np向量
    diffMat = tile(inX, (dataSetSize,1)) - dataSet
    sqDiffMat = diffMat**2
    #列相加
    sqDistances = sqDiffMat.sum(axis=1)
    distances = sqDistances**0.5
    #从小到大排列，并提取索引值
    sortedDistIndicies = distances.argsort()
    #对label出现次数进行统计
    classCount={}          
    for i in range(k):
        voteIlabel = labels[sortedDistIndicies[i]]
        #如果没有则返回0
        classCount[voteIlabel] = classCount.get(voteIlabel,0) + 1
    sortedClassCount = sorted(classCount.iteritems(), key=operator.itemgetter(1), reverse=True)
    #返回出现频率最高的label值和频率
    return sortedClassCount[0][0]
```

此处：

```python
sortedClassCount = sorted(classCount.iteritems(), key=operator.itemgetter(1), reverse=True)
```

`python3`中字典已经没有.iteritems()函数，变为.items() ,因此会报错

改为：

```python
sortedClassCount = sorted(classCount.items(), key=operator.itemgetter(1), reverse=True)
```

### 3.找出最小的错误率并制作错误率曲线图

此处书中没有。

测试集比率从0.1开始，如果测试集占比小于0.1，例如0.01，错误率会是0，但是此时测试集数据非常小，没有实际意义。因此从0.1开始

以0.02为梯度，到0.92.（同理，训练集比率过小也会导致没有意义等问题）。

代码：

代码修改为返回错误率

```python
#对约会网站进行测试
def datingClassTest(hoRatio):
    #0.1为测试集 0.9为训练集
    #hoRatio = 0.10      #hold out 10%
    datingDataMat,datingLabels = file2matrix('datingTestSet2.txt')       #load data setfrom file
    normMat, ranges, minVals = autoNorm(datingDataMat)
    m = normMat.shape[0]
    numTestVecs = int(m*hoRatio)
    errorCount = 0.0
    for i in range(numTestVecs):
        classifierResult = classify0(normMat[i,:],normMat[numTestVecs:m,:],datingLabels[numTestVecs:m],3)
        #print("the classifier came back with: %d, the real answer is: %d" % (classifierResult, datingLabels[i]))
        if (classifierResult != datingLabels[i]): errorCount += 1.0
    print("the total error rate is: %f" % (errorCount/float(numTestVecs)))
    return (errorCount/float(numTestVecs))
    #print(errorCount)
```

进行循环开始计算不同占比的错误率

```python
hoRatio = 0.1
i = 0
hoRatios = np.zeros((41,2))
while(hoRatio <= 0.92):
    print('hoRatio is : ', hoRatio)
    a = kNN.datingClassTest(hoRatio)
    hoRatios[i] = np.tile([hoRatio, a],(1,1))
    hoRatio += 0.02
    i += 1
print(i)
```

讲列表进行操作修改为`numpy ndarray`

```python
import numpy as np
import matplotlib.pyplot as plt

x = np.linspace(0, 100)
y1, y2 = hoRatios[:,0], hoRatios[:,1]
print(type(np.sin(x)))
plt.plot(y1, y2, label='error_rate')
plt.xlabel('hoRatio')
plt.ylabel('errorRate')
plt.legend()
plt.show()
```

结果图：

![Image text](https://github.com/naginoasukara/machinelearninginaction/blob/master/%E6%9C%BA%E5%99%A8%E5%AD%A6%E4%B9%A0%E5%AE%9E%E6%88%98/ch2/image/5.png)


#### 总结

由此图可见，错误率开始从0.1到0.3时呈现增长趋势，之后0.3到0.75之间是下降趋势，到0.75左右时错误率下降到最低，可见测试集比率到此时训练效果最好。之后错误率又将升高。

### 4.knn算法寻找最优参数k，以达到最佳训练效果。

KNN算法简单有效，但没有优化的暴力法效率容易达到瓶颈。如样本个数为N，特征维度为D的时候，该算法时间复杂度呈O（DN)增长。

KNN 算法本身简单有效，它是一种 lazy-learning 算法，分类器不需要使用训练集进行训练，训练时间复杂度为0。KNN 分类的计算复杂度和训练集中的文档数目成正比，也就是说，如果训练集中文档总数为 n，那么 KNN 的分类时间复杂度为O(n)。

KNN算法执行效率并不高，运行速度非常慢。算法需要为每个测试向量做2000次距离计算(每一个测试集要对所有的训练集进行距离计算,十个数字,每个数字有大约200个字迹(txt文档),10*200=2000),每个距离计算包括了1024个维度浮点计算(32*32)，总计执行900次(测试集十个数字,每个数字有90个文档,10*90)。

随着k的取值不同，knn算法的效果是不同的。同理，在这里我们做一个类似的循环，找出最优参数k。代码如下：

计算k从1到20，将错误率加入xylabel

```python
import numpy as np

k = 1
i = 0
xylabel = np.zeros((20,2))
while(k <= 20):
    a = kNN.handwritingClassTest(k)
    xylabel[i] = np.tile([k, a],(1,1))
    print(xylabel[i])
    i += 1
    k += 1
print(k)
```

同样转化为`ndarray`,使用`matplotlib`制图

```python
import matplotlib.pyplot as plt

y1, y2 = xylabel[:,0], xylabel[:,1]
plt.plot(y1, y2, label='error_rate')
plt.xlabel('k')
plt.ylabel('errorRate')
plt.legend()
plt.show()
```

![Image text](https://github.com/naginoasukara/machinelearninginaction/blob/master/%E6%9C%BA%E5%99%A8%E5%AD%A6%E4%B9%A0%E5%AE%9E%E6%88%98/ch2/image/6.png)

#### 总结

由图可见，k值越大，错误率相对越高。并可观察得出k值取3时，错误率达到最低水平。

# knn算法优化

k近邻算法是一种基于实例的算法，即学习过程只是简单的存储已知的训练数据，遇到新的查询实例时，从训练集中取出相似的实例，因此它是一种懒惰(lazy)学习方法。

可以为不同的待分类查询实例建立不同的目标函数进行逼近。

k近邻算法原理：
        
令D为训练数据集，当测试集d出现时，将d与D中所有的样本进行比较，计算他们之间的相似度（或者距离）。从D中选出前k个最相似的样本，则d的类别由k个最近邻的样

本中出现最多的类别决定。

k近邻算法关键部分是距离（相似度）函数，对于关系型数据，经常使用欧氏距离，对于文本数据，经常采用余弦相似度。k的选择是通过在训练集上交叉检验，
        
交叉验证一般分为三类：double-fold CV即经常所说的2折交叉；10-fold交叉和LOO（leave one out）CV即留一法交叉。

参考http://blog.163.com/leo666@126/blog/static/1194684712011113085410814/

2折：将原始数据集DataSet均分为两份：一份作为训练集，即trainingSet，一份作为测试集，即testingSet，然后用训练集去做训练，用测试集去验证；之后再
     
将训练集作为测试集，测试集作为训练集进行迭代一次，将两次所得的误差经行处理作为总体数据的预测误差。（注：这里强调一点，就是数据集一定要均分为两份，理由

是：作为训练集，数据量一定要不小于测试集，所以在迭代的过程中，使得数据不出现错误情况，必须均分。）
      
K-折：（在这里说下K-折）是在将数据集分成K个子集，K个子集中得一个作为测试集，而其余的K-1个数据集作为训练集，最后对K个数据子集的错误计算均值，K
      
次迭代验证是对监督学习算法的结果进行评估的方法，数据集的划分一般采用等均分或者随机划分。【来自邵峰晶等编著《数据挖掘原理与算法》中国水利水电出版社】

LOO：这个方法是K折的一种特列，就是把数据分为N份，其实每一份都是一个样本，这样迭代N次，计算最后的误差来作为预测误差。

k近邻的问题：

k近邻简单直接，有效，健壮，在很多情况下可以和复杂的算法性能相同。但是k近邻有三个缺点：

（1）需要更精确的距离函数代替欧氏距离

（2）搜索一个最优的近邻大小代替k

（3）找出更精确的类别概率估计代替简单的投票方法。

针对上述三种问题，提出了三中改进思路：

1.改进距离函数

由于它基于假设测试实例在欧式空间中最相似于近邻的实例的类别。由于实例间距离计算基于实例的所有属性，然而我们搜索的是实例包括不相关属性，标准的欧氏距离将会变得不准确。当出现许多不相关属性时称为维数灾难，kNN对此特别敏感。

解决方法：（1）消除不相关属性即特征选择。Kohavietal提出了一种缠绕法(wrapper)除此外还有贪婪搜索和遗传搜索。

（2）属性加权。w是属性a的权重

当所有的属性不均衡时，属性加权距离函数定义为

Ip (Ai;C)是属性A和类别C的互信息

除此之外，还有一种基于频率的距离函数，称之为相异性度量。与卡方距离相似

值差分度量（VDM）是标称属性的距离函数

C是输出的类别数量，P是输入属性A时输出C的条件概率，VDM在度量连续属性时需要将连续属性映射为标称属性

2.改进近邻距离大小

KNN分类准确率对K值敏感，通过交叉验证方法确定最优的K值。一旦在训练时学习了最优的K值，可以在分类时对所有的测试集使用。DKNN即动态确定K值，所有的算法都需要确定K近邻，为此，在KDTree和NBTree中，实例存储在叶节点上，邻近实例存储在相同或者相近的叶节点上。树的内部节点通过测试选择属性的相关性对测试实例进行排序

3.改进类别概率估计

KNN的实例邻近的类别被认为相同。所以改进算法需要根据他们到测试实例的距离进行加权。

另外一种非常有效的方法是基于概率的局部分类模型，即结合NB算法，这种算法在数据较小的时候表现很好。有研究者发现保持紧邻k很小将减少对于NB强依赖的机会，然而NB的类别估计概率不可信。

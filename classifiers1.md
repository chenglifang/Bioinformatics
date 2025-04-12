# 实验三：分类器之KNN, Logistic Regression, Decision Tree

## 实验目的
* 1）提取供体真实位点与虚假位点序列的k-space组分特征；构建训练集与测试集
* 2）使用K近邻（K-Nearest Neighbor, KNN）完成剪接位点识别
* 3）使用逻辑斯蒂回归（Logistic Regression, LR）完成剪接位点识别
* 4）使用决策树（Decision Tree, DT）完成剪接位点识别

## 准备工作目录
```
$ mkdir lab_03
$ cd lab_03
# 建立lab_02路径中供体true位点、false位点序列文件的软链接
$ ln -s ../lab_02/EI_true.seq ../lab_02/EI_false.seq ./

# 集群上若python3不可用，需先激活base环境
$ source /opt/miniconda3/bin/activate
$ conda activate
```

## 1. 训练集与测试集构建
* 1）编写更好用的k-spaced碱基对组分特征表征程序（用于HS3D数据的供体真实/虚假位点序列表征）<br>
参考程序：kSpaceCoding_general.py, 该程序避免了每次在程序中修改文件名和其他参数的麻烦。
```python
import numpy as np # 导入numpy包，并重命名为np
import sys # 导入sys包，用于从命令行传递参数给python程序

def file2matrix(filename, bpTable, KMAX=2): # 为KMAX提供默认参数(updated)
    fr = open(filename) # 打开文件
    arrayOLines = fr.readlines() # 读取所有内容
    del(arrayOLines[:4]) # 删除头4行（updated, 避免了运行程序之前，另外使用sed删除头4行）
    fr.close() # 及时关闭文件
    
    numberOfLines = len(arrayOLines) # 得到文件行数
    returnMat = np.zeros((numberOfLines, 16*(KMAX+1))) # 为返回的结果矩阵开辟内存
    
    lineNum = 0
    for line in arrayOLines:
        line = line.strip() # 删除空白符，包括行尾回车符
        listFromLine = line.split(': ') # 以': '为分隔符进行切片
        nt_seq = list(listFromLine[1]) # 取出核酸序列并转换成list
        del(nt_seq[70:72]) # 删除位于第71，72位的供体位点
        
        kSpaceVec = []
        for k in range(KMAX+1): # 计算不同k条件下的kSpace特征
            bpFreq = bpTable.copy() # bpTable是一个字典型变量，一定要用字典的copy函数，Python函数参数使用的址传递

            for m in range(len(nt_seq)-k-1): # 扫描序列，并计算不同碱基对的频率
                sub_str = nt_seq[m]+nt_seq[m+1+k] # 提出子串(updated)
                if sub_str in bpFreq.keys(): # 如果子串在bpFreq中有对应的key，才统计频次(updated, NOTE:在供体虚假位点序列中存在非正常碱基)
                    bpFreq[sub_str] += 1 # 序列的子串会自动在字典中寻找对应的key，很神奇！否则要自己写if语句匹配
            bpFreqVal = list(bpFreq.values()) # 取出bpFreq中的值并转换成list
            kSpaceVec.extend(np.array(bpFreqVal)/(len(nt_seq)-k-1)) # 每个k下的特征，需除以查找的所有子串数

        returnMat[lineNum,:] = kSpaceVec
        lineNum += 1
        if (lineNum % 1000) == 0:
            print('Extracting k-spaced features: %d sequences, done!' % lineNum)
    return returnMat, lineNum

if __name__ == '__main__':
    filename = sys.argv[1]
    outputFileName = sys.argv[2]
    KMAX = int(sys.argv[3])
    bpTable = {}
    for m in ('A','T','C','G'):
        for n in ('A','T','C','G'):
            bpTable[m+n] = 0
    
    kSpaceMat, SeqNum = file2matrix(filename, bpTable, KMAX)
    np.savetxt(outputFileName, kSpaceMat, fmt='%g', delimiter=',')
    print('The number of sequences is %d. Matrix of features is saved in %s' % (SeqNum, outputFileName))
```

```bash
# 获得供体真实位点序列表征结果：在命令行指定序列文件名为'EI_true.seq'，输出结果文件名为'EI_true_kSpace.txt'，KMAX值为4
$ python3 kSpaceCoding_general.py EI_true.seq EI_true_kSpace.txt 4
# 获得供体虚假位点序列表征结果：在命令行指定序列文件名为'EI_false.seq'，输出结果文件名为'EI_false_kSpace.txt'，KMAX值为4
$ python3 kSpaceCoding_general.py EI_false.seq EI_false_kSpace.txt 4
```

* 2）以序列表征文件构建训练集、测试集 <br>
```bash
# 首先安装机器学习包sklearn
$ pip3 install --user scikit-learn -i https://pypi.tuna.tsinghua.edu.cn/simple
```

参考程序：getTrainTest.py
```python
import numpy as np
import sys
from random import sample # 导入sample函数，用于从虚假位点数据中随机抽取样本
from sklearn.model_selection import train_test_split # 用于产生训练集、测试集

trueSiteFileName = sys.argv[1]
falseSiteFileName = sys.argv[2]
trueSitesData = np.loadtxt(trueSiteFileName, delimiter = ',') # 载入true位点数据
numOfTrue = len(trueSitesData)
falseSitesData = np.loadtxt(falseSiteFileName, delimiter = ',') # 载入false位点数据
numOfFalse = len(falseSitesData)
randVec = sample(range(numOfFalse), len(trueSitesData)) # 随机产生true位点样本个数的随机向量
falseSitesData = falseSitesData[randVec,] # 以随机向量从false位点数据中抽取样本

Data = np.vstack((trueSitesData, falseSitesData)) # 按行将true位点与false位点数据组合
Y = np.vstack((np.ones((numOfTrue,1)),np.zeros((numOfTrue,1)))) # 产生Y列向量
testSize = 0.3 # 测试集30%，训练集70%
X_train, X_test, y_train, y_test = train_test_split(Data, Y, test_size = testSize, random_state = 0)

trainingSetFileName = sys.argv[3]
testSetFileName = sys.argv[4]
np.savetxt(trainingSetFileName, np.hstack((y_train, X_train)), fmt='%g', delimiter=',') # 将Y与X以列组合后，保存到文件
np.savetxt(testSetFileName, np.hstack((y_test, X_test)), fmt='%g', delimiter=',')
print('Generate training set(%d%%) and test set(%d%%): Done!' % ((1-testSize)*100, testSize*100))
```

```bash
# 构建训练集与测试集：在命令行指定true位点数据、false位点数据、train文件、test文件
$ python3 getTrainTest.py EI_true_kSpace.txt EI_false_kSpace.txt EI_train.txt EI_test.txt
```

## 2. 以KNN进行剪接位点识别
参考程序：myKNN.py
```python
import numpy as np
from sklearn import neighbors # 导入KNN包
import sys

train = np.loadtxt(sys.argv[1], delimiter=',') # 载入训练集，在命令行指定文件名
test = np.loadtxt(sys.argv[2], delimiter=',') # 载入测试集

n_neighbors = int(sys.argv[3]) # 在命令行指定邻居数
weights = 'uniform' # 每个邻居的权重相等
clf = neighbors.KNeighborsClassifier(n_neighbors, weights=weights) # 创建一个KNN的实例
trX = train[:,1:]
trY = train[:,0]
clf.fit(trX, trY) # 训练模型

teX = test[:,1:]
teY = test[:,0]
predY = clf.predict(teX) # 预测测试集
Acc = sum(predY==teY)/len(teY) # 计算预测正确的样本数
print('Prediction Accuracy of KNN: %g%% (%d/%d)' % (Acc*100, sum(predY==teY), len(teY)))
```

```bash
# KNN分类器：在命令行指定训练集、测试集、近邻数K
$ python3 myKNN.py EI_train.txt EI_test.txt 10
```

## 3. 以Logistic回归进行剪接位点识别
参考程序：myLR.py
```python
import numpy as np
from sklearn import linear_model # 导入线性模型包
import sys

train = np.loadtxt(sys.argv[1], delimiter=',') # 载入训练集
test = np.loadtxt(sys.argv[2], delimiter=',') # 载入测试集

maxIterations = int(sys.argv[3]) # 在命令行指定最大迭代次数
clf = linear_model.LogisticRegression(max_iter=maxIterations) # 创建一个LR的实例
trX = train[:,1:]
trY = train[:,0]
clf.fit(trX, trY) # 训练模型

teX = test[:,1:]
teY = test[:,0]
predY = clf.predict(teX) # 预测测试集
Acc = sum(predY==teY)/len(teY) # 计算预测正确的样本数
print('Prediction Accuracy of LR: %g%% (%d/%d)' % (Acc*100, sum(predY==teY), len(teY)))
```

```bash
# LR分类器：在命令行指定训练集、测试集、迭代次数
$ python3 myLR.py EI_train.txt EI_test.txt 1000
```

## 4. 以Decision Tree进行剪接位点识别
参考程序：myDT.py
```python
import numpy as np
from sklearn import tree # 导入Decision Trees包
import sys
import graphviz # 导入Graphviz包

train = np.loadtxt(sys.argv[1], delimiter=',') # 载入训练集
test = np.loadtxt(sys.argv[2], delimiter=',') # 载入测试集

clf = tree.DecisionTreeClassifier() # 创建一个DT的实例
trX = train[:,1:]
trY = train[:,0]
clf.fit(trX, trY) # 训练模型

teX = test[:,1:]
teY = test[:,0]
predY = clf.predict(teX) # 预测测试集
Acc = sum(predY==teY)/len(teY) # 计算预测正确的样本数
print('Prediction Accuracy of DT: %g%% (%d/%d)' % (Acc*100, sum(predY==teY), len(teY)))

# Export the tree in Graphviz format
graphFileName = sys.argv[3] # 从命令行指定图文件名称
dotData = tree.export_graphviz(clf, out_file=None)
graph = graphviz.Source(dotData)
graph.render(graphFileName)
print('The tree in Graphviz format is saved in "%s.pdf".' % graphFileName)
```

```bash
# 安装Graphviz绘图包
$ pip3 install --user graphviz -i https://pypi.tuna.tsinghua.edu.cn/simple
# DT分类器：在命令行指定训练集、测试集、DT图文件名
$ python3 myDT.py EI_train.txt EI_test.txt EISplicing_DecisionTreeGraph
```
[获得的DT树](https://github.com/ZhijunBioinf/Pattern-Recognition-and-Prediction/blob/master/Lab3_Classifiers_KNN-LR-DT/EISplicing_DecisionTreeGraph.pdf)

## 作业
1. 尽量看懂`参考程序`的每一行代码。
2. 参考程序kSpaceCoding_general.py中，供体位点序列的第71、72位保守二核苷酸GT是在程序中指定的，试着改写程序，实现从命令行传递`位置信息`给程序。
3. 参考程序getTrainTest.py中，测试集的比例testSize是在程序中指定的，试着改写程序，实现从命令行传递`划分比例`给程序。
4. 熟练使用sklearn包中的不同分类器。 <br>
不怕报错，犯错越多，进步越快！

## 参考
* KNN手册：[sklearn.neighbors.KNeighborsClassifier](https://scikit-learn.org/stable/modules/neighbors.html#nearest-neighbors-classification)
* LR手册：[sklearn.linear_model.LogisticRegression](https://scikit-learn.org/stable/modules/linear_model.html#logistic-regression)
* DT手册：[sklearn.tree.DecisionTreeClassifier](https://scikit-learn.org/stable/modules/tree.html#classification)

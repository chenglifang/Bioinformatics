# 实验七：特征降维/选择(PCA, MI-filter, SVM-RFE, RF)

## 实验目的
* 1）数据：[实验六](https://github.com/ZhijunBioinf/Pattern-Recognition-and-Prediction/blob/master/Lab6_Regression_MLR-PLSR-SVR/regress1.md)ACE抑制剂的训练集与测试集；基本模型：SVR
* 2）使用主成分分析(Principal Component Analysis, PCA)进行特征压缩降维，再以SVR建模预测，对比`实验六`的预测结果。
* 3）Filter: 使用基于互信息的单变量过滤法(Mutual Information-based Filter)进行特征选择，后续同(2)。
* 4）Wrapper: 使用基于SVM的迭代特征剔除(SVM Recursive Feature Elimination, SVM-RFE)进行特征选择，后续同(2)。
* 5）Embed: 使用随机森林(Random Forest, RF)进行特征选择（用于剪接位点识别），再以SVC建模预测。

## 准备工作目录与数据
```bash
$ mkdir lab_07
$ cd lab_07
# 对lab_06路径中的ACE抑制剂训练集与测试集建立软链接
$ ln -s ../lab_06/ACE_train.txt
$ ln -s ../lab_06/ACE_test.txt

# 若python3不可用，需先激活base环境
$ source /opt/miniconda3/bin/activate
$ conda activate
```

## 1. 使用PCA进行特征压缩降维，再以保留主成分建立SVR模型
* 参考程序：myPCA.py
```python3
import sys
import numpy as np
from sklearn.decomposition import PCA # 导入PCA包

trainFile = sys.argv[1]
testFile = sys.argv[2]
train = np.loadtxt(trainFile, delimiter='\t') # 载入训练集
test = np.loadtxt(testFile, delimiter='\t') # 载入测试集
trX = train[:,1:]
trY = train[:,0]
teX = test[:,1:]
teY = test[:,0]

percentVar = float(sys.argv[3]) # 主成分累计解释的百分比
pca = PCA(n_components = percentVar, svd_solver = 'full') # 创建一个PCA实例
trX = pca.fit_transform(trX)
teX = pca.transform(teX)

print('Number of principal components: %d' % trX.shape[1])
newTrainFile = sys.argv[4]
newTestFile = sys.argv[5]
np.savetxt(newTrainFile, np.hstack((trY.reshape(-1,1), trX)), fmt='%g', delimiter='\t') # 将Y与X以列组合后，保存到文件
np.savetxt(newTestFile, np.hstack((teY.reshape(-1,1), teX)), fmt='%g', delimiter='\t')
print('New training set is saved into: %s\nNew test set is saved into: %s' % (newTrainFile, newTestFile))
```

```bash
# 设置主成分累计解释的百分比为95%
$ python3 myPCA.py ACE_train.txt ACE_test.txt 0.95 train_pca.txt test_pca.txt
# 以SVR建模预测：规格化、rbf核、10次交叉寻优
$ python3 ../lab_06/mySVR.py train_pca.txt test_pca.txt 1 rbf 10 ObsdYvsPredY_PCA_SVR.pdf
```

## 2. 使用MI-filter进行特征选择，再以保留变量建立SVR模型
* 参考程序：myMIFilter.py
```python3
import sys
import numpy as np
from sklearn.feature_selection import SelectPercentile, mutual_info_regression # 导入MI for regression包

trainFile = sys.argv[1]
testFile = sys.argv[2]
train = np.loadtxt(trainFile, delimiter='\t') # 载入训练集
test = np.loadtxt(testFile, delimiter='\t') # 载入测试集
trX = train[:,1:]
trY = train[:,0]
teX = test[:,1:]
teY = test[:,0]

percentile = int(sys.argv[3]) # Percent of features to keep
selector = SelectPercentile(mutual_info_regression, percentile=percentile) # 创建一个基于MI的SelectPercentile实例
trX = selector.fit_transform(trX, trY)
teX = selector.transform(teX)

newTrainFile = sys.argv[4]
newTestFile = sys.argv[5]
np.savetxt(newTrainFile, np.hstack((trY.reshape(-1,1), trX)), fmt='%g', delimiter='\t') # 将Y与X以列组合后，保存到文件
np.savetxt(newTestFile, np.hstack((teY.reshape(-1,1), teX)), fmt='%g', delimiter='\t')
print('%d features are selected.' % trX.shape[1])
print('New training set is saved into: %s\nNew test set is saved into: %s' % (newTrainFile, newTestFile))
```

```bash
# 保留5%的特征
$ python3 myMIFilter.py ACE_train.txt ACE_test.txt 5 train_mi.txt test_mi.txt
# 以SVR建模预测：规格化、rbf核、10次交叉寻优
$ python3 ../lab_06/mySVR.py train_mi.txt test_mi.txt 1 rbf 10 ObsdYvsPredY_MI_SVR.pdf
```

## 3. 使用SVM-RFE进行特征选择，再以保留变量建立SVR模型
* 参考程序：mySVMRFE.py
```python3
import sys
import numpy as np
from sklearn.feature_selection import RFE # 导入RFE包
from sklearn.svm import SVR # 导入SVR包
from pytictoc import TicToc
from random import sample

trainFile = sys.argv[1]
testFile = sys.argv[2]
train = np.loadtxt(trainFile, delimiter='\t') # 载入训练集
test = np.loadtxt(testFile, delimiter='\t') # 载入测试集
trX = train[:,1:]
trY = train[:,0]
teX = test[:,1:]
teY = test[:,0]

numX = trX.shape[1]
if numX > 200:
    randVec = np.array(sample(range(numX), 100)) + 1 # 考虑到特征数较多，SVM运行时间较长，随机抽100个特征用于后续建模
    print('Note: 100 features are randomly selected to speed up modeling')
    trX = train[:,randVec]
    teX = test[:,randVec]
    
n_features = int(sys.argv[3])
estimator = SVR(kernel="linear")
t = TicToc()
t.tic()
selector = RFE(estimator, n_features_to_select=n_features, step=1) # 创建一个RFECV实例，每次删1个特征
trX = selector.fit_transform(trX, trY)
teX = selector.transform(teX)
print('Time cost in selecting fetures with SVM-RFE: %gs' % t.tocvalue())

newTrainFile = sys.argv[4]
newTestFile = sys.argv[5]
np.savetxt(newTrainFile, np.hstack((trY.reshape(-1,1), trX)), fmt='%g', delimiter='\t') # 将Y与X以列组合后，保存到文件
np.savetxt(newTestFile, np.hstack((teY.reshape(-1,1), teX)), fmt='%g', delimiter='\t')
print('%d features are selected.' % trX.shape[1])
print('New training set is saved into: %s\nNew test set is saved into: %s' % (newTrainFile, newTestFile))
```

* SVM-RFE运行时间较长，将命令写到脚本中再用qsub提交任务。Note: 命令脚本中也需要先激活base环境，才可用python3
* work_mySVMRFE.sh
```bash
#!/bin/bash
#$ -S /bin/bash
#$ -N mySVMRFE
#$ -j y
#$ -cwd

# 激活base环境，保证计算节点上正常使用python3
source /opt/miniconda3/bin/activate
conda activate

# 设置保留10个特征
python3 mySVMRFE.py ACE_train.txt ACE_test.txt 10 train_svmrfe.txt test_svmrfe.txt
```
```bash
# 提交任务
$ qsub work_mySVMRFE.sh
```
```bash
# 以SVR建模预测：规格化、rbf核、10次交叉寻优
$ python3 ../lab_06/mySVR.py train_svmrfe.txt test_svmrfe.txt 1 rbf 10 ObsdYvsPredY_RFE_SVR.pdf
```

## 4. 使用Random Forest进行特征选择，再以保留变量建立SVC模型
```bash
# RF只能用于分类问题，对lab_03路径中的剪接位点训练集与测试集建立软链接
$ ln -s ../lab_03/EI_train.txt
$ ln -s ../lab_03/EI_test.txt
```

* 参考程序：myRandomForest.py
```python3
import sys
import numpy as np
from sklearn.feature_selection import SelectFromModel # 导入SelectFromModel包
from sklearn.ensemble import RandomForestClassifier # 导入RF包
from pytictoc import TicToc

trainFile = sys.argv[1]
testFile = sys.argv[2]
train = np.loadtxt(trainFile, delimiter=',') # 载入训练集
test = np.loadtxt(testFile, delimiter=',') # 载入测试集
trX = train[:,1:]
trY = train[:,0]
teX = test[:,1:]
teY = test[:,0]

n_features = sys.argv[3]
t = TicToc()
t.tic()
clf = RandomForestClassifier(max_depth=2, random_state=0, max_features=n_features) # 创建一个RF实例
clf = clf.fit(trX, trY)
selector = SelectFromModel(clf, prefit=True) # 创建一个SelectFromModel实例
trX = selector.transform(trX)
teX = selector.transform(teX)
print('Time cost in selecting fetures with Random-Forest: %gs' % t.tocvalue())

newTrainFile = sys.argv[4]
newTestFile = sys.argv[5]
np.savetxt(newTrainFile, np.hstack((trY.reshape(-1,1), trX)), fmt='%g', delimiter=',') # 将Y与X以列组合后，保存到文件
np.savetxt(newTestFile, np.hstack((teY.reshape(-1,1), teX)), fmt='%g', delimiter=',')
print('%d features are selected.' % trX.shape[1])
print('New training set is saved into: %s\nNew test set is saved into: %s' % (newTrainFile, newTestFile))
```

```bash
# max_features (looking for the best split): set as 'auto'
$ python3 myRandomForest.py EI_train.txt EI_test.txt auto EI_train_rf.txt EI_test_rf.txt
# 以SVR建模预测：规格化、rbf核、10次交叉寻优
$ python3 ../lab_04/mySVC.py EI_train_rf.txt EI_test_rf.txt 1 rbf 1 10
```


## 作业
1. 尽量看懂`参考程序`的每一行代码。 <br>
2. 熟练使用sklearn包中的不同特征降维/选择方法。 <br>
3. 使用不同特征选择方法结合不同回归模型（或分类器）对ACE抑制肽（实验五）、剪接位点（实验三）数据进行建模预测。  
不怕报错，犯错越多，进步越快！

## 参考
* PCA手册：[sklearn.decomposition.PCA](https://scikit-learn.org/stable/modules/decomposition.html#principal-component-analysis-pca)
* MI手册：[sklearn.feature_selection.mutual_info_regression](https://scikit-learn.org/stable/modules/feature_selection.html#univariate-feature-selection)
* RFE手册：[sklearn.feature_selection.RFE](https://scikit-learn.org/stable/modules/feature_selection.html#recursive-feature-elimination)
* RandomForest: [sklearn.ensemble.RandomForestClassifier](https://scikit-learn.org/stable/modules/feature_selection.html#tree-based-feature-selection)

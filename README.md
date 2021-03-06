# [2021全国数字生态创新大赛-智能算法](https://tianchi.aliyun.com/competition/entrance/531860/introduction)-不落星光组

| 时间 | 得分 | 排名 |
| :-----| :----- | :----- |
| 2021/3/3 | 0.3994 | 43/3927 |


## 1. 赛题描述
    本赛题基于不同地形地貌的高分辨率遥感影像资料，希望参赛者能够利用遥感影像智能解译技术识别提取土地覆盖和利用类型，实现生态资产盘点、土地利用动态监测、水环境监测与评估、耕地数量与监测等应用。结合现有的地物分类实际需求，参照地理国情监测、“三调”等既有地物分类标准，设计陆域土地覆盖与利用类目体系，包括：林地、草地、耕地、水域、道路、城镇建设用地、农村建设用地，工业用地、构筑物、裸地。
| 名称                    | 大小     |                                                         Link | md5                    |
| :---------------------- | :------- | -----------------------------------------------------------: |:---------------------- |
| suichang_round1_train_210120.zip         | 2.83GB | https://tianchi-competition.oss-cn-hangzhou.aliyuncs.com/531860/suichang_round1_train_210120.zip | 384ffa280672b04726e9ef00ced0e273     |
| suichang_round1_test_partA_210120.zip         | 530.17MB | https://tianchi-competition.oss-cn-hangzhou.aliyuncs.com/531860/suichang_round1_test_partA_210120.zip | 155394c28002f77fe9ba13ccbe19167a     |
## 2. 解决方案
我们采用Unet++进行实验。

| 库 | 版本 |
| :-----| :----- |
| GDAL | 3.1.4 |
| segmentation-models-pytorch | 0.1.3 |
| torch | 1.7.0+cu110 |
| pytorch-toolbelt | 0.4.1 

### 2.1. 数据预处理
#### 2.1.1. 统计各类的像素百分比
我们需要了解一下我们的数据中每个类的像素占比情况，对我们后续处理和分析有一定的帮助，运行程序为"code\count_classes.py"。
我们可以发现，各类别非常不均衡，所以我们需要进行少类别上采样以及选择合适的损失函数。
![](https://github.com/WangZhenqing-RS/2021Tianchi_RS/blob/main/code/%E5%9C%B0%E7%89%A9%E8%A6%81%E7%B4%A0%E7%B1%BB%E5%88%AB%E5%83%8F%E7%B4%A0%E6%95%B0%E7%9B%AE%E5%9B%BE.png)
#### 2.1.2. 分隔训练集和验证集
最初为了更加充分利用数据，采用五折交叉验证方式对模型进行训练。在分隔训练集和验证集时，我们在连续五个数据中取其中四份做训练数据，其中一份做验证数据。我们使用dataProcess.py文件中的split_train_val_old函数进行分隔。实验发现线上分数和线下分数差别很大，推测应该是测试集和训练集不同域，光谱差异较大。

为了模拟不同域，我们使用全部数据作为训练集，全部数据作为验证集，只不过训练集和验证集的增强方式不同。
#### 2.1.3. 少类别上采样
我们对类别像素占比很少的类别进行上采样处理，抵抗不均衡现象。若图像包含类别5、6、7则上采样2份，类别3、8、10因为得分太低，采取放弃策略，类别4几乎每张影像都有，亦采取放弃策略。
#### 2.1.4. 波段选取
我们使用了比赛提供的R/G/B/Nir波段。我们实验过增加归一化植被指数NDVI作为image的第5通道输入到网络中，但是效果不佳，故舍弃这一策略。
#### 2.1.5. 数据增强
为增强模型泛化性，我们对训练数据增强策略采用了随机水平翻转、垂直翻转、对角翻转以及5%百分比线性拉伸。为模拟变域，我们对验证集数据进行了随机0.8%、1%、2%线性拉伸。
### 2.2. 训练
#### 2.2.1. 优化器
我们选择Adamw优化器，初始学习率lr=1e-4，权重衰减weight_decay=1e-3。
#### 2.2.2. 学习率调整
在训练时梯度下降算法可能陷入局部最小值，此时可以通过突然提高学习率，来“跳出”局部最小值并找到通向全局最小值的路径。所以我们采用余弦退火策略调整学习率。T_0=2，T_mult=2，eta_min=1e-5。
#### 2.2.3. 损失函数
软交叉熵函数是对标签值进行标签平滑之后再与预测值做交叉熵计算，可以在一定程度上提高泛化性。diceloss在一定程度上可以缓解类别不平衡,但是训练容易不稳定。我们采用软交叉熵函数和diceloss的联合函数作为实验的损失函数。

我们测试了用LovaszLoss进行fine tune，但是最终结果变差了，故放弃。
#### 2.2.4. SWA随机权重平均
我们测试了SWA随机权重平均策略来增强模型泛化性，但是最终结果也变差了，故放弃。
### 2.3. 预测
#### 2.3.1. TTA测试增强
测试时对原图像、水平翻转图像、垂直翻转图像以及百分比截断增强图像的预测结果进行平均，得到TTA结果。
#### 2.3.2. 模型融合
我们训练了不同backbone的unet++，对预测结果取平均，得到最终结果。

## 3. 参考

[阿水233](https://tianchi.aliyun.com/notebook-ai/detail?spm=5176.12586969.1002.3.6cc26423Zxyf0s&postId=169396)

[DLLXW](https://github.com/DLLXW/data-science-competition/tree/main/%E5%A4%A9%E6%B1%A0)

[JasmineRain](https://github.com/JasmineRain/NAIC_AI-RS/tree/ec70861e2a7f3ba18b3cc8bad592e746145088c9)

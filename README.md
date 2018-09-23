# IJCAI-18 阿里妈妈搜索广告转化预测 Top2%思路（初赛）
## 赛题回顾
搜索广告的转化率，作为衡量广告转化效果的指标，从广告创意、商品品质、商店质量等多个角度综合刻画用户对广告商品的购买意向，
即广告商品被用户点击后产生购买行为的概率。

本赛题给出了某月18日到24日的数据作为训练集，并从25日的数据中（按用户？）抽取30%的数据作为A榜测试集，
70%的数据作为B榜测试集，预测某一次点击后产生购买行为的概率。损失函数使用二分类中常用的logloss。（复赛数据量过大，因而这里主要是针对初赛问题的解决方案，不过解决问题的思路基本是一致的）

[比赛链接](https://tianchi.aliyun.com/competition/introduction.htm?spm=5176.100066.0.0.6acdd780S84lIq&raceId=231647)  

## 解决方案
本次赛题提供的数据主要包括各种id类特征、用户特征、广告商品特征和店铺特征，基于CTR预估的特点，将特征工程的重心放在用户相关的特征构造上，
并且与其他统计特征相结合，对于得到的特征集合，采用wrapper方式的特征选择方法选出最优的特征子集。最后用不同的特征组训练了两个LightGBM进行模型融合。

## 数据划分
<table>
    <tr>
        <td></td>
        <td>训练集</td>
        <td>测试集</td>
    </tr>
    <tr>
        <td>线下</td>
        <td>19-23日</td>
        <td>24日</td>
    </tr>
    <tr>
        <td>线上</td>
        <td>19-24日</td>
        <td>25日</td>
    </tr>
</table>

线下采用24号数据进行简单交叉验证，之所以没使用k折交叉验证，主要是因为用到了和时间相关的特征，k折交叉验证不如简单交叉验证可靠。

是否使用18号的数据一直是我很纠结的一个问题，因为18号的数据无法使用滑窗特征（即对前一天的各种统计），并且对于我构造的一个强特（当前点击距上一次点击的间隔时间），18号的数据也无法很好的利用这个强特；另一方面，由于这次赛题提供的数据量较少（只有七天时间），所以去掉其中一天不可避免的会带来一定的损失，问题就是看你利用滑窗构造的特征和强特的重要程度。

原始特征不到30维，而lgb1中共构造了100多维特征，这时候滑窗特征和强特只占其中的一小部分，所以lgb1中使用了18号的数据；lgb2只在原始特征基础上只增加了滑窗特征和强特等10维左右的特征，因而没有使用18号的效果反而更好。

## 数据处理
- **id类特征**转化为该id对应的平滑后的历史转化率（实际中效果不理想故而舍去）

- **文本向量特征**(广告属性列表等)使用CountVectorize向量化，取词频最高的前几十维作为特征（实际效果提升不大，用
PCA降维或SVD分解可能更好）

- **类别特征**（用户职业、用户性别等）进行one-hot编码，当然也可以在LightGBM中设置类别特征

- **时间戳特征**转化为时间特征（天，小时）

- **缺失值**默认为-1，因为缺失值较少且LightGBM和XGBOOST都可以自动处理缺失值，所以不需处理

## 特征工程

### 特征构造
主要从以下几个方面构造特征，和很多其他队伍一样，其中也用到了一些data leakage，这部分特征实际业务中是无法获取的：

- **全局统计特征**
    - 用户统计特征  
    - 商品统计特征  
    - 用户和商品组合统计特征
    - 用户和店铺组合统计特征
    - 店铺和商品组合统计特征  

- **滑窗统计特征**
    - 前一天同一用户查询次数、购买次数和转化率（平滑后）
    - 前一天同一商品被查询次数、被购买次数和转化率（平滑后）
    - 前一天同一店铺被查询次数、被购买次数和转化率（平滑后）  

- **数值特征两两交叉**（四则运算）  

- **线上统计特征**
    - 今日用户查询次数  
    - 今日当前一小时内用户查询次数  
    - 今日用户查询同一商品的次数  
    - 今日用户查询同一店铺的次数  

- **构造的几个强特**  
    - 用户这次查询距上一次查询的时间（秒）  
    - 用户这次查询距上一次查询的时间（分钟）  
    - 用户这次查询距下一次查询的时间（存在data leakage）  
    - 用户这次查询同一商品距上一次的时间  
    - 用户这次查询同一商品距下一次的时间（存在data leakage）  
    
### 特征选择
特征选择借鉴了技术圈战友开源的方案https://github.com/duxuhao/Feature-Selection 。 

大体思路是使用wrapper的方式，结合前后向搜索算法筛选特征，然后通过随机策略解决前后向搜索易陷入局部最优解的问题，当然这个代码还包含了构造交叉特征的方法，在实际中我并没有采用，而是事先手动的构造交叉特征再去筛选。

## 模型融合
模型融合的时候，考虑到用到了一些时间相关特征，所以用stacking的方法不会很理想（因为stacking要使用到k折交叉验证），所以我采用了加权平均融合的方法。最后在融合提交的时候出了一点小问题，导致正确的融合后地结果没有提交成功，如果成功融合的话或许可以再提高一些成绩，算是一点遗憾吧。

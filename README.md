# IJCAI-18 阿里妈妈搜索广告转化预测 Top2%思路
## 赛题回顾
搜索广告的转化率，作为衡量广告转化效果的指标，从广告创意、商品品质、商店质量等多个角度综合刻画用户对广告商品的购买意向，
即广告商品被用户点击后产生购买行为的概率。

本赛题给出了某月18日到24日的数据作为训练集，并从25日的数据中（按用户？）抽取30%的数据作为A榜测试集，
70%的数据作为B榜测试集，预测某一次点击后产生购买行为的概率。损失函数使用二分类中常用的logloss。

## 解决方案
本次赛题提供的数据主要包括各种id类特征、用户特征、广告商品特征和店铺特征，基于CTR预估的特点，将特征工程的重心放在用户相关的特征构造上，
并且与其他统计特征相结合，对于得到的特征集合，采用wrapper方式的特征选择方法选出最优的特征子集。最后训练了LightGBM和XGBOOST进行模型融合。

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

是否使用18号的数据一直是我很纠结的一个问题，因为18号的数据无法使用划窗特征（即对前一天的各种统计），并且对于我构造的一个强特（当前点击距上一次点击的间隔时间），18号的数据也无法很好的利用这个强特；另一方面，由于这次赛题提供的数据量较少（只有七天时间），所以去掉其中一天不可避免的会带来一定的损失，问题就是看你利用划窗构造的特征和强特的重要程度。

原始特征不到30维，而lgb1中共构造了100多维特征，这时候划窗特征和强特只占其中的一小部分，所以lgb1中使用了18号的数据；lgb2只在原始特征基础上只增加了划窗特征和强特等10维左右的特征，因而没有使用18号的效果反而更好。

## 数据处理
- **id类特征**转化为该id对应的平滑后的历史转化率（实际中效果不理想故而舍去）
-
- 类别特征（用户职业、用户性别等）进行one-hot编码，当然也可以在LightGBM中设置类别特征；


## 特征工程

## 模型融合

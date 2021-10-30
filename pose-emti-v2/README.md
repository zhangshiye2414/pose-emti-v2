# pose-emti-v2

测圆（二值化版本）

## Update 10.22
重新回归github怀抱。哦，还有，换项目了，再见
## Update 10.25
项目换了，算法没换，修修补补还能用，继续...

**所用环境：**

    * vcpkg 
    * "Visual Studio 2019"
    * CMake

## Windows

下载Visual Studio 16 2019工具链，用vcpkg安装opencv4包，指定编译目标64/32位，然后CMake编译生成

## Linux

设置USE_VCPKG=OFF，然后CMake编译生成

## 算法逻辑

1. 捕获原图片
2. 直方图均衡(均衡光照)
3. 二值化（自适应OTSU）
4. 中值滤波
5. 找轮廓
6. 过滤不必要的轮廓
    1. 父关系节点需要存在
    2. 滤过面积较小的斑点
    3. 滤过圆度不达标的图形
    4. 依照父关系节点归类
7. 拟合椭圆
8. 通过拟合椭圆结算真正的圆

算法6漏洞：并未通过9个圆点之间的相互位置进行过滤，一时这个算法不好写，写出来算法复杂度太高，时间上不占优势， 二者缺乏好的算法，比如如果存在角度该怎么办？

## 优化

### 健壮性优化

不过还是需要一个严格的证明...

### 速度优化

优化目标：最终目标30FPS，解算一张图片所需时间大约0.033s左右。现阶段优化目标时间`time consume < 0.1`
优化条件：不得使用包括GPU在内的任何硬件加速功能（艹）

**时间占比：**

|阶段|步骤|消耗时间|
| ---- |----|----|
|1|直方图均衡|0.008|
|2|二值化   |0.012|
|3|中值滤波 |0.14|
|4|寻找轮廓 |0.029|
|5|滤过轮廓 |0.003|
|6|拟合圆   |0s|
|7|总计    |0.207|

1. 阶段3中值滤波所占时间较长，应该划为主要优化目标
2. 如果去掉中值滤波，阶段4，5时间上都会有明显的增加
3. 中值滤波主要是为了应对图片上许多模糊的点（可以视作椒盐噪声）
4. 分别用闭运算、开运算去除黑点、毛点白点，注意kernel大小不能太大，副作用，边框会被腐蚀断开

|阶段|步骤|消耗时间|
|----|----|----|
|1|直方图均衡|0.008|
|2|二值化   |0.009|
|3|闭开运算 |0.05|
|4|寻找轮廓 |0.02|
|5|滤过轮廓 |0.002|
|6|拟合圆   |0.002s|
|7|总计    |0.091|
5. 需要用一种新算法来替代依赖的父节点分类，父节点分类确实有问题
6. 添加了面积过滤，在多个轮廓同时找到多个符合上述逻辑条件后，必须满足面积上符合面积近似的要求

|阶段|步骤|消耗时间|
|----|----|----|
|1|直方图均衡|0.008|
|2|二值化   |0.009|
|3|闭开运算 |0.051|
|4|寻找轮廓 |0.02|
|5|滤过轮廓 |0.002|
|6|拟合圆   |0.001s|
|7|总计    |0.090|

7. 全局二值化OTSU存在非常大的缺陷，换回adaptiveThreshold
8. 算法采用gamma矫正作为图像增强算法,gamma设定0.6
9. 算法修改逻辑，具体如下
   1.  算出所有轮廓
   2.  过滤面积不符合标准的
   3.  过滤圆度不符合标准的
   4.  剩下的疑似圆按照父节点分组
   5.  一组轮廓数量如果小于9排除
   6.  将每组轮廓按照是否`有关联`组成图（有关联算法如下
       1.  两轮廓之间半径比值需要满足0.7/1 - 1/0.7之间
       2.  两轮廓轮廓中心需要满足3r-5r的关系
       3.  两轮廓在同一条直线上
   7.  计算该图，如果存在某个节点通过该关系找到9个圆那就说明找到目标
   8.  返回结果
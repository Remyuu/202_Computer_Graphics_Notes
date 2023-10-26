

## 计算每个像素下cubemap某个面的球谐系数

### 分析

**球谐系数**是球谐函数在一个球面上的投影，可以用于表示函数在球面上的分布。由于我们有RGB值有三个通道，因此我们会的球谐系数会存储为一个三维的向量。需要完善的部分：

```c++
/// prt.cpp - PrecomputeCubemapSH()
// TODO: here you need to compute light sh of each face of cubemap of each pixel
// TODO: 此处你需要计算每个像素下cubemap某个面的球谐系数
Eigen::Vector3f dir = cubemapDirs[i * width * height + y * width + x];
int index = (y * width + x) * channel;
Eigen::Array3f Le(images[i][index + 0], images[i][index + 1],
                  images[i][index + 2]);
```

首先从cubemap的每个像素采样方向（一个三维向量，表示从中心到像素的方向），将方向转化为球面坐标（ `theta` 和 `phi` ）。

接着将各个球面坐标传入 `sh::EvalSH()` 分别计算每个球谐函数（基函数）的实值 `sh` 。同时计算每个 cubemap 中每个像素所占据球面区域的比重 `delta` 。

最后累加球谐系数，代码中我们可以对 cubemap 伤的所有像素累加，近似是原始计算球谐函数的积分的操作。

$$
Y_{lm} = \int_{\phi=0}^{2\pi} \int_{\theta=0}^{\pi} f(\theta, \phi) Y_l^m (\theta, \phi) \sin(\theta) d\theta d\phi\\
$$
其中：
- ( $\theta$ ) 是天顶角，范围从 ( $0$ ) 到 ( $ \pi $ )；( $\phi$ ) 是方位角，范围从 ( $0$ ) 到 ( $2pi$ )。
- ( $Y_l^m$ ) 是球谐函数，它由相应的勒让德多项式 ( $P_l^m$ ) 和一些三角函数组成。
- ( $l$ ) 是球谐函数的阶数；( $m$ ) 是球谐函数的序数，范围从 ( $-l$ ) 到 ( $l$ )。



### 代码实现

- 将方向向量转换为球面坐标：

```c++
double theta = acos(dir.z());
double phi = atan2(dir.y(), dir.x());
```

`theta` 是正z轴到 `dir` 方向的夹角，而 `phi` 表示从正x轴到 `dir` 在xz平面上的投影的夹角。








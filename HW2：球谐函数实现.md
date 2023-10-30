![模型法线](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20231027162044646.png)

<center> 模型法线 </center>

## 预计算球谐系数

### 环境光照：计算每个像素下cubemap某个面的球谐系数

> `ProjEnv::PrecomputeCubemapSH<SHOrder>(images, width, height, channel);`
> 
> 使用黎曼积分的方法计算环境光球谐函数的系数。

#### 完整代码

```c++
// TODO: here you need to compute light sh of each face of cubemap of each pixel
// TODO: 此处你需要计算每个像素下cubemap某个面的球谐系数
Eigen::Vector3f dir = cubemapDirs[i * width * height + y * width + x];
int index = (y * width + x) * channel;
Eigen::Array3f Le(images[i][index + 0], images[i][index + 1],
                  images[i][index + 2]);
// 描述在球面坐标上当前的角度
double theta = acos(dir.z());
double phi = atan2(dir.y(), dir.x());
// 遍历球谐函数的各个基函数
for (int l = 0; l <= SHOrder; l++){
    for (int m = -l; m <= l; m++){
        float sh = sh::EvalSH(l, m, phi, theta);
        float delta = CalcArea((float)x, (float)y, width, height);
        SHCoeffiecents[l*(l+1)+m] += Le * sh * delta;
    }
}
```

#### 分析

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

首先从六个 cubemap （ `images` 数组）的每个像素采样方向（一个三维向量，表示从中心到像素的方向），将方向转化为球面坐标（ `theta` 和 `phi` ）。

接着将各个球面坐标传入 `sh::EvalSH()` 分别计算每个球谐函数（基函数）的实值 `sh` 。同时计算每个 cubemap 中每个像素所占据球面区域的比重 `delta` 。

最后累加球谐系数，代码中我们可以对 cubemap 伤的所有像素累加，近似是原始计算球谐函数的积分的操作。

$$
Y_{lm} = \int_{\phi=0}^{2\pi} \int_{\theta=0}^{\pi} f(\theta, \phi) Y_l^m (\theta, \phi) \sin(\theta) d\theta d\phi\\
$$

其中：

- $\theta$  是天顶角，范围从 $0$ 到 $ \pi $； $\phi$  是方位角，范围从  $0$  到  $2pi$ 。
- $f\left(\theta, \phi\right)$ 是球面上某点的函数值。
- $Y_l^m$  是球谐函数，它由相应的勒让德多项式  $P_l^m$  和一些三角函数组成。
- $l$  是球谐函数的阶数； $m$  是球谐函数的序数，范围从  $-l$  到  $l$ 。

为了更加具体的让读者理解，这里写出代码中球谐函数离散形式的估计，即黎曼积分的方法来计算。
$$
Y_{l m}=\sum_{i=1}^N f\left(\theta_i, \phi_i\right) Y_l^m\left(\theta_i, \phi_i\right) \Delta \omega_i
$$

其中：

- $f\left(\theta_i, \phi_i\right)$ 是球面上某点的函数值。
- $Y_l^m\left(\theta_i, \phi_i\right)$ 是球谐函数在该点的值。
- $\Delta \omega_i$ 是该点在球面上的微小区域或权重。
- $N$ 是所有离散点的总数。

#### 代码细节

- **从 cubemap 获取RGB光照信息**

```c++
Eigen::Array3f Le(images[i][index + 0], images[i][index + 1],
                  images[i][index + 2]);
```

`channel` 的值是3，对应于RGB三个通道。因此，`index` 就指向了某一像素的红色通道的位置，`index + 1` 指向绿色通道的位置，`index + 2` 指向蓝色通道的位置。

- **将方向向量转换为球面坐标**

```c++
double theta = acos(dir.z());
double phi = atan2(dir.y(), dir.x());
```

`theta` 是正z轴到 `dir` 方向的夹角，而 `phi` 表示从正x轴到 `dir` 在xz平面上的投影的夹角。

- **遍历球谐函数的各个基函数**

```c++
for (int l = 0; l <= SHOrder; l++){
    for (int m = -l; m <= l; m++){
        float sh = sh::EvalSH(l, m, phi, theta);
        float delta = CalcArea((float)x, (float)y, width, height);
        SHCoeffiecents[l*(l+1)+m] += Le * sh * delta;
    }
}
```

### 无阴影的漫反射项

> `scene->getIntegrator()->preprocess(scene);`
> 
> 计算 **Diffuse Unshadowed** 的情况。化简渲染方程，将上节的球谐函数代入进一步计算为BRDF的球谐投影的系数。其中关键的函数是`ProjectFunction`。我们要为这个函数编写一个lambda表达式，用于计算传输函数项 $\text{max}\left(N_x \cdot \omega_i , 0\right)$ 。

#### 分析

对于漫反射传输项，可以**分为三种情况**考虑：**有阴影的**、**无阴影的**和**相互反射的**。

首先考虑最简单的没有阴影的情况。我们有渲染方程

$$
L\left(x, \omega_o\right)=\int_S f_r\left(x, \omega_i, \omega_o\right) L_i\left(x, \omega_i\right) H\left(x, \omega_i\right) \mathrm{d} \omega_i
$$

其中，

- $L_i$ 是入射辐射度。
- $H$ 是几何函数，表面的微观特性和入射光的方向有关。
- $\omega_i$ 是入射光方向。

对于表面处处相等的漫反射表面，我们可以简化得到 **Unshadowed** 光照方程
$$
L_{D U}=\frac{\rho}{\pi} \int_S L_i\left(x, \omega_i\right) \max \left(N_x \cdot \omega_i, 0\right) \mathrm{d} \omega_i
$$
其中：

- $L_{D U}(x)$ 是点 $x$ 的漫反射出射辐射度。
- $N_x$ 是表面法线。

入射辐射度 $L_i$ 和传输函数项 $\text{max}\left(N_x \cdot \omega_i , 0\right)$ 相互独立，因为前者代表场景中光源的贡献，后者表示表面如何响应入射的光线。因此将这两个部分独立处理。

具体到运用球谐函数近似是，我们分别对这两项展开。前者的输入是光的入射方向，后者输入的是反射（或者出射方向），并且展开是两个系列的数组，因此我们使用名为查找表（Look-Up Table，简称LUT）的数据结构。

```c++
auto shCoeff = sh::ProjectFunction(SHOrder, shFunc, m_SampleCount);
```

其中，最重要的是上面这个函数`ProjectFunction`。我们要为这个函数编写一个Lambda表达式（`shFunc`）作为传参，表达式用于计算传输函数项 $\text{max}\left(N_x \cdot \omega_i , 0\right)$ 。

`ProjectFunction` 函数传参：

- 球谐阶数
- 需要投影在基函数上的函数（我们需要编写的）
- 采样数

该函数会取Lambda函数返回的结果投影在基函数上得到系数，最后把各个样本系数累加并乘以权重，最后得出该顶点的最终系数。

#### 完整代码

计算几何项，即传输函数项 $\text{max}\left(N_x \cdot \omega_i , 0\right)$ 。

```c++
if (m_Type == Type::Unshadowed){
    // TODO: here you need to calculate unshadowed transport term of a given direction
    // TODO: 此处你需要计算给定方向下的unshadowed传输项球谐函数值
    double H = wi.normalized().dot(n.normalized());
    return H>0 ? H : 0.0;
}
```

### 有阴影的漫反射项

> `scene->getIntegrator()->preprocess(scene);`
> 
> 计算 **Diffuse Shadowed** 的情况。这一项多了一个可见项 $V(\omega_i)$ 。

#### 分析

Visibility项是一个非1即0的值，利用 `bool rayIntersect(const Ray3f &ray)` 函数，从顶点位置到采样方向反射一条射线，若击中物体，则认为被遮挡，有阴影，返回0；若射线未击中物体，则仍然返回 $max(N_{x} \cdot \omega_{i}, 0)$ 即可。
$$
\mathbf{L}_{D S}=\frac{\rho}{\pi} \int_S L_i\left(x, \omega_i\right) V\left(\omega_i\right) \max \left(N_x \cdot \omega_i, 0\right) d \omega_i
$$

#### 完整代码

```c++
```





![image-20231027162340813](/Users/remooo/Library/Application%20Support/typora-user-images/image-20231027162340813.png)

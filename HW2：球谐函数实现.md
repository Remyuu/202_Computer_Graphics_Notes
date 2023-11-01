

![image-20231030230515084](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20231030230515084.png)

<center> PRT渲染材质截图 </center>

## 预计算球谐系数

> 利用框架nori预先计算球谐函数系数。

### 环境光照：计算每个像素下cubemap某个面的球谐系数

> `ProjEnv::PrecomputeCubemapSH<SHOrder>(images, width, height, channel);`
> 
> 使用黎曼积分的方法计算环境光球谐函数的系数。

#### 完整代码

```cpp
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

```cpp
Eigen::Array3f Le(images[i][index + 0], images[i][index + 1],
                  images[i][index + 2]);
```

`channel` 的值是3，对应于RGB三个通道。因此，`index` 就指向了某一像素的红色通道的位置，`index + 1` 指向绿色通道的位置，`index + 2` 指向蓝色通道的位置。

- **将方向向量转换为球面坐标**

```cpp
double theta = acos(dir.z());
double phi = atan2(dir.y(), dir.x());
```

`theta` 是正z轴到 `dir` 方向的夹角，而 `phi` 表示从正x轴到 `dir` 在xz平面上的投影的夹角。

- **遍历球谐函数的各个基函数**

```cpp
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

其中，最重要的是上面这个函数 `ProjectFunction` 。我们要为这个函数编写一个Lambda表达式（`shFunc`）作为传参，表达式用于计算传输函数项 $\text{max}\left(N_x \cdot \omega_i , 0\right)$ 。

`ProjectFunction` 函数传参：

- 球谐阶数
- 需要投影在基函数上的函数（我们需要编写的）
- 采样数

该函数会取Lambda函数返回的结果投影在基函数上得到系数，最后把各个样本系数累加并乘以权重，最后得出该顶点的最终系数。

#### 完整代码

计算几何项，即传输函数项 $\text{max}\left(N_x \cdot \omega_i , 0\right)$ 。

```cpp
// prt.cpp
...
double H = wi.normalized().dot(n.normalized()) / M_PI;
if (m_Type == Type::Unshadowed){
    // TODO: here you need to calculate unshadowed transport term of a given direction
    // TODO: 此处你需要计算给定方向下的unshadowed传输项球谐函数值
    return (H > 0.0) ? H : 0.0;
}
```

> 总之最后的积分结果要记得除以 $\pi$ ，再传给 `m_TransportSHCoeffs` 。

### 有阴影的漫反射项

> `scene->getIntegrator()->preprocess(scene);`
> 
> 计算 **Diffuse Shadowed** 的情况。这一项多了一个可见项 $V(\omega_i)$ 。

#### 分析

Visibility项（$V\left(\omega_i\right)$）是一个非1即0的值，利用 `bool rayIntersect(const Ray3f &ray)` 函数，从顶点位置到采样方向反射一条射线，若击中物体，则认为被遮挡，有阴影，返回0；若射线未击中物体，则仍然返回 $max(N_{x} \cdot \omega_{i}, 0)$ 即可。

$$
\mathbf{L}_{D S}=\frac{\rho}{\pi} \int_S L_i\left(x, \omega_i\right) V\left(\omega_i\right) \max \left(N_x \cdot \omega_i, 0\right) d \omega_i
$$

#### 完整代码

```cpp
// prt.cpp
...
double H = wi.normalized().dot(n.normalized()) / M_PI;
...
else{
    // TODO: here you need to calculate shadowed transport term of a given direction
    // TODO: 此处你需要计算给定方向下的shadowed传输项球谐函数值
    if (H > 0.0 && !scene->rayIntersect(Ray3f(v, wi.normalized())))
        return H;
    return 0.0;
}
```

> 总之最后的积分结果要记得除以 $\pi$ ，再传给 `m_TransportSHCoeffs` 。

### 导出计算结果

> nori框架会生成的两个预计算结果的文件。

添加运行参数：

```tex
./scenes/prt.xml
```

在 prt.xml 中，需要做以下**修改**，就可以选择渲染的环境光cubemap。另外，模型、相机参数等也可自行修改。

```xml
// prt.xml

<!-- Render the visible surface normals -->
<integrator type="prt">
    <string name="type" value="unshadowed" />
    <integer name="bounce" value="1" />
    <integer name="PRTSampleCount" value="100" />
<!--        <string name="cubemap" value="cubemap/GraceCathedral" />-->
<!--        <string name="cubemap" value="cubemap/Indoor" />-->
<!--        <string name="cubemap" value="cubemap/Skybox" />-->
    <string name="cubemap" value="cubemap/CornellBox" />

</integrator>
```

其中，标签可选值：

- `type`：unshadowed、shadowed、 interreflection
- `bounce`：interreflection类型下的光线弹射次数（目前尚未实现）
- `PRTSampleCount`：传输项每个顶点的采样数
- `cubemap`：cubemap/GraceCathedral、cubemap/Indoor、cubemap/Skybox、cubemap/CornellBox

![image-20231030230515084](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20231030230515084.png)

上图分别是GraceCathedral、Indoor、Skybox和CornellBox的 `unshadowed` 渲染结果，采样数是1。

## 使用球谐系数着色

> 将nori生成的文件手动拖到实时渲染框架中，并且对实时框架做一些改动。

上一章计算完成后，将对应cubemap路径中的 `light.txt` 和 `transport.txt` 拷贝到实时渲染框架的cubemap文件夹中。

### 预计算数据解析

**取消** `engine.js` 中88-114行的注释，这一段代码用于解析刚才添加进来的txt文件。

```js
// engine.js
// file parsing
... // 把这块代码取消注释
```

### 导入模型/创建并使用PRT材质Shader

在materials文件夹下**建立**文件 `PRTMaterial.js` 。

```js
//PRTMaterial.js

class PRTMaterial extends Material {
    constructor(vertexShader, fragmentShader) {
        super({
            'uPrecomputeL[0]': { type: 'precomputeL', value: null},
            'uPrecomputeL[1]': { type: 'precomputeL', value: null},
            'uPrecomputeL[2]': { type: 'precomputeL', value: null},
        }, 
        ['aPrecomputeLT'], 
        vertexShader, fragmentShader, null);
    }
}

async function buildPRTMaterial(vertexPath, fragmentPath) {
    let vertexShader = await getShaderString(vertexPath);
    let fragmentShader = await getShaderString(fragmentPath);

    return new PRTMaterial(vertexShader, fragmentShader);
}
```

然后在 `index.html` 里引入。

```html
// index.html
<script src="src/materials/Material.js" defer></script>
<script src="src/materials/ShadowMaterial.js" defer></script>
<script src="src/materials/PhongMaterial.js" defer></script>
<!-- Edit Start --><script src="src/materials/PRTMaterial.js" defer></script><!-- Edit End -->
<script src="src/materials/SkyBoxMaterial.js" defer></script>
```

在 `loadOBJ.js` 加载新的材质。

```js
// loadOBJ.js

switch (objMaterial) {
    case 'PhongMaterial':
        material = buildPhongMaterial(colorMap, mat.specular.toArray(), light, Translation, Scale, "./src/shaders/phongShader/phongVertex.glsl", "./src/shaders/phongShader/phongFragment.glsl");
        shadowMaterial = buildShadowMaterial(light, Translation, Scale, "./src/shaders/shadowShader/shadowVertex.glsl", "./src/shaders/shadowShader/shadowFragment.glsl");
        break;
    // TODO: Add your PRTmaterial here
    //Edit Start
    case 'PRTMaterial':
        material = buildPRTMaterial("./src/shaders/prtShader/prtVertex.glsl", "./src/shaders/prtShader/prtFragment.glsl");
        break;
    //Edit End
    // ...
}
```

给场景添加mary模型，设置位置与大小，并且使用刚建立的材质。

```js
//engine.js

// Add shapes
...
// Edit Start
let maryTransform = setTransform(0, -35, 0, 20, 20, 20);
// Edit End
...
// TODO: load model - Add your Material here
...
// Edit Start
loadOBJ(renderer, 'assets/mary/', 'mary', 'PRTMaterial', maryTransform);
// Edit End
```

### 计算着色

> 将预计算数据载入GPU中。

在渲染循环的camera pass中给材质设置precomputeL实时的值，也就是传递预先计算的数据给shader。下面代码是每一帧中每一趟camera pass中每一个网格mesh的每一个uniforms的遍历。实时渲染框架已经解析了预计算的数据并且存储到了uniforms中。`precomputeL`是一个 9x3 的矩阵，代表这里分别有RGB三个通道的前三阶（9个）球谐函数（实际上我们会说这是一个 3x3 的矩阵，但是我们写代码直接写成一个长度为9的数组）。为了方便使用，通过 `tool.js` 的函数将 `precomputeL` 转换为 3x9 的矩阵。

通过 `uniformMatrix3fv` 函数，我们可以将材质里存储的信息上传到GPU上。这个函数接受三个参数，具体请查阅 [WebGL文档 - uniformMatrix](https://developer.mozilla.org/en-US/docs/Web/API/WebGLRenderingContext/uniformMatrix) 。其中第一个参数的作用是在我们自己创建的 `PRTMaterial` 中，`uniforms` 包含了 `uPrecomputeL[0]` , `uPrecomputeL[1]` 和 `uPrecomputeL[2]` 。在GPU内的工作不需要我们关注，我们只需要有在CPU上的 uniform ，就可以通过API自动访问到GPU上对应的内容。换句话说，当获取一个 uniform 或属性的位置，实际上得到的是一个在CPU端的引用，但在底层，这个引用会映射到GPU上的一个具体位置。而链接 uniform 的步骤在 `Shader.js` 的 `this.program = this.addShaderLocations（）` 中完成（看看代码就能懂了，只是比较绕，在我的HW1文章中也有分析过）， `shader.program` 有三个属性分别是：`glShaderProgram`, `uniforms`, 和 `attribs`。而具体声明的位置则是在 `XXXshader.glsl` 中，在下一步中我们就会完成它。

总结一下，下面这段代码主要工作就是为片段着色器提供预先处理过的数据。

```js
// WebGLRenderer.js

if (k == 'uMoveWithCamera') { // The rotation of the skybox
    gl.uniformMatrix4fv(
        this.meshes[i].shader.program.uniforms[k],
        false,
        cameraModelMatrix);
}

// Bonus - Fast Spherical Harmonic Rotation
//let precomputeL_RGBMat3 = getRotationPrecomputeL(precomputeL[guiParams.envmapId], cameraModelMatrix);

// Edit Start
let Mat3Value = getMat3ValueFromRGB(precomputeL[guiParams.envmapId]);

if (/^uPrecomputeL\[\d\]$/.test(k)) {
    let index = parseInt(k.split('[')[1].split(']')[0]);
    if (index >= 0 && index < 3) {
        gl.uniformMatrix3fv(
            this.meshes[i].shader.program.uniforms[k],
            false,
            Mat3Value[index]
        );
    }
}
// Edit End
```

> 也可以将 `Mat3Value` 的计算放在i循环的外面，减少计算次数。

#### 编写顶点着色器

明白了上面代码的作用之后，接下来的任务就非常明了了。上一步我们将每一个球谐系数都传到了 GPU 的 `uPrecomputeL[]` 中，接下来在GPU上编程计算球谐系数和传输矩阵的点乘，也就是下图 light_coefficient * transport_matrix。

实时渲染框架中已经完成了Light_Transport到对应方向的矩阵的化简，我们只需要分别对三个颜色通道的长度为9的向量做点乘就行了。值得一提的是，`PrecomputeL` 和 `PrecomputeLT` 既可以传给顶点着色器也可以传给片段着色器，若传给顶点着色器，就只需要在片段着色器中差值得到颜色，速度更快，但是真实性就稍差一些。怎么计算取决于不同的需求。

<img title="" src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20231016142641895.png" alt="image-20231016142641895" width="357" data-align="center">

```glsl
//prtVertex.glsl

attribute vec3 aVertexPosition;
attribute vec3 aNormalPosition;
attribute mat3 aPrecomputeLT;  // Precomputed Light Transfer matrix for the vertex

uniform mat4 uModelMatrix;
uniform mat4 uViewMatrix;
uniform mat4 uProjectionMatrix;
uniform mat3 uPrecomputeL[3];  // Precomputed Lighting matrices
varying highp vec3 vNormal;

varying highp vec3 vColor;     // Outgoing color after the dot product calculations

float L_dot_LT(const mat3 PrecomputeL, const mat3 PrecomputeLT) {
  return dot(PrecomputeL[0], PrecomputeLT[0]) 
        + dot(PrecomputeL[1], PrecomputeLT[1]) 
        + dot(PrecomputeL[2], PrecomputeLT[2]);
}

void main(void) {
  // 防止因为浏览器优化报错，无实际作用
  aNormalPosition;

  for(int i = 0; i < 3; i++) {
      vColor[i] = L_dot_LT(aPrecomputeLT, uPrecomputeL[i]);
  }

  gl_Position = uProjectionMatrix * uViewMatrix * uModelMatrix * vec4(aVertexPosition, 1.0);
}
```

另外值得一说的是，在渲染框架中为一个名为 `aNormalPosition` 的attribute设置了数值，如果在Shader中没有使用的话就会被WebGL优化掉，导致浏览器不停报错。

#### 编写片元着色器

在顶点着色器中完成对当前顶点着色的计算之后，在片元着色器中插值计算颜色。由于在顶点着色器中为每个顶点计算的`vColor`值会在片元着色器中被自动插值，因此直接使用就可以了。

```cpp
// prtFragment.glsl

#ifdef GL_ES
precision mediump float;
#endif

varying highp vec3 vColor;

void main(){
  gl_FragColor = vec4(vColor, 1.0);
}
```

## 曝光与颜色矫正

虽然框架作者提及PRT预计算保存的结果是在线性空间中的，不需要再进行 gamma 矫正了，但是显然最终结果是有问题的。如果您没有事先在计算系数的时候除 $\pi$ ，那么以Skybox场景为例子，就会出现过曝的问题。如果事先除了 $\pi$ ，但是没有做色彩矫正，就会在实时渲染框架中出现过暗的问题。

<img title="" src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/PicGo_dir/202310312156208.png" alt="" width="533" data-align="center">

首先在计算系数的时候除以 $\pi$ ，然后再做一个色彩矫正。怎么做呢？我们可以参照nori框架的导出图片过程中有一个 `toSRGB()` 的函数：

```cpp
// common.cpp
Color3f Color3f::toSRGB() const {
    Color3f result;

    for (int i=0; i<3; ++i) {
        float value = coeff(i);

        if (value <= 0.0031308f)
            result[i] = 12.92f * value;
        else
            result[i] = (1.0f + 0.055f)
                * std::pow(value, 1.0f/2.4f) -  0.055f;
    }

    return result;
}
```

我们可以仿照这个在片元着色其中做色彩矫正。

```cpp
//prtFragment.glsl

#ifdef GL_ES
precision mediump float;
#endif

varying highp vec3 vColor;

vec3 toneMapping(vec3 color){
    vec3 result;

    for (int i=0; i<3; ++i) {
        if (color[i] <= 0.0031308)
            result[i] = 12.92 * color[i];
        else
            result[i] = (1.0 + 0.055) * pow(color[i], 1.0/2.4) - 0.055;
    }

    return result;
}

void main(){
  vec3 color = toneMapping(vColor); 
  gl_FragColor = vec4(color, 1.0);
}
```

这样就可以保证实时渲染框架渲染的结果与nori框架的截图结果一致了。

<img title="" src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/PicGo_dir/202310312209259.png" alt="" width="547" data-align="center">

我们也可以做其他的颜色矫正，这里提供几种常见的Tone Mapping方法，用于将HDR范围转换至LDR范围。

```glsl
vec3 linearToneMapping(vec3 color) {
    return color / (color + vec3(1.0));
}
vec3 reinhardToneMapping(vec3 color) {
    return color / (vec3(1.0) + color);
}
vec3 exposureToneMapping(vec3 color, float exposure) {
    return vec3(1.0) - exp(-color * exposure);
}
vec3 filmicToneMapping(vec3 color) {
    color = max(vec3(0.0), color - vec3(0.004));
    color = (color * (6.2 * color + 0.5)) / (color * (6.2 * color + 1.7) + 0.06);
    return color;
}
```

到这里为止，作业的基础部分就完成了。

## 添加CornellBox场景

默认框架代码中没有CornellBox，但是资源文件里面有，这就需要我们自行添加：

```js
// engine.js

var envmap = [
    'assets/cubemap/GraceCathedral',
    'assets/cubemap/Indoor',
    'assets/cubemap/Skybox',
    // Edit Start
    'assets/cubemap/CornellBox',
    // Edit End
];
```

```js
// engine.js

function createGUI() {
    const gui = new dat.gui.GUI();
    const panelModel = gui.addFolder('Switch Environemtn Map');
    // Edit Start
    panelModel.add(guiParams, 'envmapId', { 'GraceGathedral': 0, 'Indoor': 1, 'Skybox': 2, 'CornellBox': 3}).name('Envmap Name');
    // Edit End
    panelModel.open();
}
```

<img title="" src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/PicGo_dir/202311011536493.png" alt="" width="477" data-align="center">

## 基础部分结果展示

> 分别展示shadowed和unshadowed的四个场景。

<img title="" src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/PicGo_dir/202311011555458.png" alt="" width="768" data-align="center">

## 考虑传输项光线多次弹射

> 计算多次弹射的光线传输与光线追踪有相似之处，在使用球谐函数（Spherical Harmonics，SH）进行光照近似时，您可以结合光线追踪来计算这些多次反射的效果。

### 完整代码

```cpp
// TODO: leave for bonus
Eigen::MatrixXf m_IndirectCoeffs = Eigen::MatrixXf::Zero(SHCoeffLength, mesh->getVertexCount());
int sample_side = static_cast<int>(floor(sqrt(m_SampleCount)));

std::random_device rd;
std::mt19937 gen(rd());
std::uniform_real_distribution<> rng(0.0, 1.0);

const double twoPi = 2.0 * M_PI;

for(int bo = 0; bo < m_Bounce; bo++)
{
    for (int i = 0; i < mesh->getVertexCount(); i++)
    {
        const Point3f &v = mesh->getVertexPositions().col(i);
        const Normal3f &n = mesh->getVertexNormals().col(i);

        std::vector<float> coeff(SHCoeffLength, 0.0f);
        for (int t = 0; t < sample_side; t++) {
            for (int p = 0; p < sample_side; p++) {
                double alpha = (t + rng(gen)) / sample_side;
                double beta = (p + rng(gen)) / sample_side;
                double phi = twoPi * beta;
                double theta = acos(2.0 * alpha - 1.0);
                
                Eigen::Array3d d = sh::ToVector(phi, theta);
                const Vector3f wi(d[0], d[1], d[2]);
                
                double H = wi.dot(n);
                if(H > 0.0) {
                    const auto ray = Ray3f(v, wi);
                    Intersection intersect;
                    bool is_inter = scene->rayIntersect(ray, intersect);
                    if(is_inter) {
                        for(int j = 0; j < SHCoeffLength; j++) {
                            const Vector3f coef3(
                                m_TransportSHCoeffs.col((int)intersect.tri_index[0]).coeffRef(j),
                                m_TransportSHCoeffs.col((int)intersect.tri_index[1]).coeffRef(j),
                                m_TransportSHCoeffs.col((int)intersect.tri_index[2]).coeffRef(j)
                            );
                            coeff[j] += intersect.bary.dot(coef3) / m_SampleCount;
                        }
                    }
                }
            }
        }

        for (int j = 0; j < SHCoeffLength; j++)
        {
            m_IndirectCoeffs.col(i).coeffRef(j) = coeff[j] - m_IndirectCoeffs.col(i).coeffRef(j);
        }
    }
    m_TransportSHCoeffs += m_IndirectCoeffs;
}

```

### 分析

在计算有遮挡的阴影的基础上（**直接光照**），加上二次反射光（**间接照明**）的贡献。而二次反射的光线也可以再进行相同的步骤。对于间接光照的计算，使用球谐函数对这些反射光线的照明进行近似。如果考虑多次弹射，则使用 $\hat{L}\left(x^{\prime}, \omega_i\right) $ 进行递归计算，终止条件可以是递归深度或光线强度低于某个阈值。下面就是文字的公式描述。

$$
L_{D I}=L_{D S}+\frac{\rho}{\pi} \int_S \hat{L}\left(x^{\prime}, \omega_i\right)\left(1-V\left(\omega_i\right)\right) \max \left(N_x \cdot \omega_i, 0\right) \mathrm{d} \omega_i
$$





### 结果

观察一下，整体上没有太大差异，只是阴影的地方更加亮了。

![](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/PicGo_dir/202311011649761.png)











## 附录

<img title="" src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20231027162044646.png" alt="模型法线" style="zoom:50%;" width="529" data-align="center">

<center> 模型法线 </center>

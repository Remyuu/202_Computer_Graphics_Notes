## TODO List

- 实现对场景直接光照的着色 (考虑阴影)。

- 实现屏幕空间下光线的求交 (SSR)。

- 实现对场景间接光照的着色。

- 实现动态步长的RayMarch。

-  **（还没写）** Bonus 1：实现 Mipmap 优化的 Screen Space Ray Tracing。

## 写在前面

这一次作业的基础部分算是目前202所有作业中最简单的了，没有特别复杂的内容。但是Bonus部分不知如何下手，大佬们带带。

### 框架的深度缓冲问题

这一次作业在 macOS 上会遇到比较严重的问题。正方体贴近地面的部分会随着摄像机的距离远近变化表现出异常的裁切锯齿问题。这个现象在 windows 上没有遇到，比较奇怪。

<img title="" src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/PicGo_dir/202311141324851.png" alt="" width="501" data-align="center">

个人感觉这与深度缓冲区的精度有关，可能是z-fighting导致的，其中两个或更多重叠的表面竞争同一像素的问题。对于这种问题一般下面几种解决方案：

- 调整近平面和远平面：不要让近平面离摄像机太近，远平面不要太远。

- 提高深度缓冲区的精度：采用32位或者更高的精度。

- 多通渲染（Multi-Pass Rendering）：对不同距离范围的物体采用不同的渲染方案。

最简单的解决办法就是修改近平面的大小，定位到框架的 `engine.js` 的25行。

```js
// engine.js
// const camera = new THREE.PerspectiveCamera(75, gl.canvas.clientWidth / gl.canvas.clientHeight, 0.0001, 1e5);
const camera = new THREE.PerspectiveCamera(75, gl.canvas.clientWidth / gl.canvas.clientHeight, 5e-2, 1e2);
```

这样就可以得到相当锐利的边界了。

<img title="" src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/PicGo_dir/202311141528946.png" alt="" width="462" data-align="center">

### 增加「暂停渲染」功能

这个部分是可选的。为了减轻电脑的压力，简单写一个暂停渲染的按钮。

```js
// engine.js
let settings = {
    'Render Switch': true
};

function createGUI() {
    ...
    // Add the boolean switch here
    gui.add(settings, 'Render Switch');
    ...
}

function mainLoop(now) {
    if(settings['Render Switch']){
        cameraControls.update();
        renderer.render();
    }
    requestAnimationFrame(mainLoop);
}
requestAnimationFrame(mainLoop);
```

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20231117191114477.png" alt="image-20231117191114477" />

## 1. 实现直接光照

> 实现「shaders/ssrShader/ssrFragment.glsl」中的 `EvalDiffuse(vec3 wi, vec3 wo, vec2 uv)` 和 `EvalDirectionalLight(vec2 uv)` 。

```Glsl
// ssrFragment.glsl
vec3 EvalDiffuse(vec3 wi, vec3 wo, vec2 screenUV) {
  vec3 reflectivity = GetGBufferDiffuse(screenUV);
  vec3 normal = GetGBufferNormalWorld(screenUV);
  float cosi = max(0., dot(normal, wi));
  vec3 f_r = reflectivity * cosi;
  return f_r;
}

vec3 EvalDirectionalLight(vec2 screenUV) {
  vec3 Li = uLightRadiance * GetGBufferuShadow(screenUV);
  return Li;
}
```

第一段代码其实就是实现了Lambertian reflection model，对应渲染方程里面的 $f_r \cdot \text{cos}(\theta_i)$ 。

> 我这里是除了 $\pi$ ，但是按照作业框架给出的结果，应该是没有除的，这里随便吧。

第二部分负责直接光照（包括阴影遮挡），相对渲染方程的 $L_i \cdot V$ 。

$$
L_o\left(p, \omega_o\right)=L_e\left(p, \omega_o\right)+\int_{\Omega} L_i\left(p, \omega_i\right) \cdot f_r\left(p, \omega_i, \omega_o\right) \cdot V\left(p, \omega_i\right) \cdot \cos \left(\theta_i\right) d \omega_i
$$

这里顺便复习一下Lambertian反射模型。我们注意到 `EvalDiffuse` 传入了`wi, wo` 两个方向，但我们只是用了入射光的方向 `wi` 。这是因为Lambertian模型与观察的方向没有关系，只和表面法线与入射光线的余弦值有关。

最后在 `main()` 中设置结果。

```glsl
// ssrFragment.glsl
void main() {
  float s = InitRand(gl_FragCoord.xy);
  vec3 L = vec3(0.0);
  vec3 wi = normalize(uLightDir);
  vec3 wo = normalize(uCameraPos - vPosWorld.xyz);
  vec2 worldPos = GetScreenCoordinate(vPosWorld.xyz);

  L = EvalDiffuse(wi, wo, worldPos) * 
      EvalDirectionalLight(worldPos);

  vec3 color = pow(clamp(L, vec3(0.0), vec3(1.0)), vec3(1.0 / 2.2));
  gl_FragColor = vec4(vec3(color.rgb), 1.0);
}
```

<img title="" src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/PicGo_dir/202311161758763.png" alt="" data-align="center">

## 2. 镜面SSR - 实现RayMarch

> 实现 `RayMarch(ori, dir, out hitPos)` 函数，求出光线与物体的交点，返回光线是否与物体相交。参数 ori 和 dir 为世界坐标系中的值，分别代表光线的起点和方向，其中方向向量为单位向量。
> 
> 更多资料可以参考EA在SIG15的[课程报告](https://www.slideshare.net/DICEStudio/stochastic-screenspace-reflections)。

作业框架的「cube1」本身就包含了地面，所以这玩意最终得到的SSR效果就不太美观。这里的“美观”是指论文中结果图的清晰度或游戏中积水反射效果的精致度。

准确地说，在本文中我们实现的是最基础的「镜面SSR」，即Basic mirror-only SSR。

<img title="" src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/PicGo_dir/202311171238510.png" alt="" data-align="center">

实现「镜面SSR」最简单的方法就是使用Linear Raymarch，通过一个个小步进逐步确定当前位置与gBuffer的深度位置的遮挡关系。

<img title="" src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/PicGo_dir/202311162250903.png" alt="" data-align="center">

```glsl
// ssrFragment.glsl
bool RayMarch(vec3 ori, vec3 dir, out vec3 hitPos) {
  const int totalStepTimes = 60;
  const float threshold = 0.0001;
  float step = 0.05;
  vec3 stepDir = normalize(dir) * step;
  vec3 curPos = ori;

  for(int i = 0; i < totalStepTimes; i++) {
    vec2 screenUV = GetScreenCoordinate(curPos);
    float rayDepth = GetDepth(curPos);
    float gBufferDepth = GetGBufferDepth(screenUV);

    // Check if the ray has hit an object
    if(rayDepth > gBufferDepth + threshold){
      hitPos = curPos;
      return true;
    }
    curPos += stepDir;
  }
  return false;
}
```

最后微调步进 `step` 的大小。最终我取到0.05。如果步进取的太大，反射的画面会“断层”。如果步进取得太小且步进次数又不够，那么可能导致本来应该反射的地方因为步进距离不够导致计算的终止。下图的最大步进数为150。

![](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/PicGo_dir/202311162346552.png)

```glsl
// ssrFragment.glsl
vec3 EvalSSR(vec3 wi, vec3 wo, vec2 screenUV) {
  vec3 worldNormal = GetGBufferNormalWorld(screenUV);
  vec3 relfectDir = normalize(reflect(-wo, worldNormal));
  vec3 hitPos;
  if(RayMarch(vPosWorld.xyz, relfectDir, hitPos)){
    vec2 INV_screenUV = GetScreenCoordinate(hitPos);
    return GetGBufferDiffuse(INV_screenUV);
  }
  else{
    return vec3(0.); 
  }
}
```

写一个调用 `RayMarch` 的函数包装起来，方便在 `main()` 中使用。

```glsl
// ssrFragment.glsl
void main() {
  float s = InitRand(gl_FragCoord.xy);
  vec3 L = vec3(0.0);
  vec3 wi = normalize(uLightDir);
  vec3 wo = normalize(uCameraPos - vPosWorld.xyz);
  vec2 screenUV = GetScreenCoordinate(vPosWorld.xyz);

  // Basic mirror-only SSR
  float reflectivity = 0.2;

  L = EvalDiffuse(wi, wo, screenUV) * EvalDirectionalLight(screenUV);
  L+= EvalSSR(wi, wo, screenUV) * reflectivity;

  vec3 color = pow(clamp(L, vec3(0.0), vec3(1.0)), vec3(1.0 / 2.2));
  gl_FragColor = vec4(vec3(color.rgb), 1.0);
}
```

如果单纯想测试SSR的效果，请在 `main()` 中自行调整。

<img title="" src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/PicGo_dir/202311161910641.png" alt="" data-align="center">

<img title="" src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/PicGo_dir/202311161814134.png" alt="" data-align="center">

在2013年"Killzone Shadow Fall"发布之前，SSR技术仍然受到较大的限制，因为在实际开发中，我们通常需要模拟Glossy的物体，由于当时性能的限制，SSR技没有大规模采用。随着“Killzone Shadow Fall”的发布，标志着实时反射技术取得了重大的进展。得益于PS4的特殊硬件，使得实时渲染高质量Glossy和semi-reflective的物体成为可能。

<img title="" src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/PicGo_dir/202311171306750.png" alt="" data-align="center">

在接下来的几年中，SSR技术发展迅速，尤其是与PBR等技术的结合。

从Nvidia的RTX显卡开始，实时光线追踪的兴起逐渐开始在某些场景替代了SSR。但是在大多数开发场景中，传统的SSR仍然占有相当大的戏份。

未来的发展趋势依然是传统SSR技术与光线追踪技术的混合。

## 3. 间接光照

> 照着伪代码写。也就是用蒙特卡洛方法求解渲染方程。与之前不同的是，这次的样本都在屏幕空间中。在采样的过程中可以使用框架提供的 `SampleHemisphereUniform(inout s, ou pdf)` 和 `SampleHemisphereCos(inout s, out pdf)` ，其中，这两个函数返回局部坐标，传入参数分别是随机数 `s` 和采样概率 `pdf` 。

这个部分需要理解下图伪代码，然后照着完成 `EvalIndirectionLight()` 就好了。

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/PicGo_dir/202311171456252.png"/>

首先需要知道，我们本次采样仍然是基于屏幕空间的。因此不在屏幕（gBuffer）中的内容我们就当作不存在。理解为只有一层正好面向摄像机的外壳。

间接光照涉及上半球方向的随机采样和对应pdf计算。使用 `InitRand(screenUV)` 得到随机数就可以了，然后二选一， `SampleHemisphereUniform(inout float s, out float pdf)` 或 `SampleHemisphereCos(inout float s, out float pdf)` ，更新随机数同时得到对应 `pdf` 和单位半球上的局部坐标系的位置 `dir` 。

将当前Shading Point的法线坐标传入函数 `LocalBasis(n, out b1, out b2)` ，随后返回 `b1, b2` ，其中 `n, b1, b2` 这三个单位向量两两正交。通过这三个向量所构成的局部坐标系，将 `dir` 转换到世界坐标中。关于这个 `LocalBasis()` 的原理，我写在最后了。

> By the way, the matrix constructed with the vectors `n` (normal), `b1`, and `b2` is commonly referred to as the TBN matrix in computer graphics.

```glsl
// ssrFragment.glsl
#define SAMPLE_NUM 5

vec3 EvalIndirectionLight(vec3 wi, vec3 wo, vec2 screenUV){
  vec3 L_ind = vec3(0.0);
  float s = InitRand(screenUV);
  vec3 normal = GetGBufferNormalWorld(screenUV);
  vec3 b1, b2;
  LocalBasis(normal, b1, b2);

  for(int i = 0; i < SAMPLE_NUM; i++){
    float pdf;
    vec3 direction = SampleHemisphereUniform(s, pdf);
    vec3 worldDir = normalize(mat3(b1, b2, normal) * direction);

    vec3 position_1;
    if(RayMarch(vPosWorld.xyz, worldDir, position_1)){ // 采样光线碰到了 position_1
      vec2 hitScreenUV = GetScreenCoordinate(position_1);
      vec3 bsdf_d = EvalDiffuse(worldDir, wo, screenUV); // 直接光照
      vec3 bsdf_i = EvalDiffuse(wi, worldDir, hitScreenUV); // 间接光照
      L_ind += bsdf_d / pdf * bsdf_i * EvalDirectionalLight(hitScreenUV);
    }
  }
  L_ind /= float(SAMPLE_NUM);
  return L_ind;
}
```

```glsl
// ssrFragment.glsl
// Main entry point for the shader
void main() {
  vec3 wi = normalize(uLightDir);
  vec3 wo = normalize(uCameraPos - vPosWorld.xyz);
  vec2 screenUV = GetScreenCoordinate(vPosWorld.xyz);

  // Basic mirror-only SSR coefficient
  float ssrCoeff = 0.0;
  // Indirection Light coefficient
  float indCoeff = 0.3;

  // Direction Light
  vec3 L_d = EvalDiffuse(wi, wo, screenUV) * EvalDirectionalLight(screenUV);
  // SSR Light
  vec3 L_ssr = EvalSSR(wi, wo, screenUV) * ssrCoeff;
  // Indirection Light
  vec3 L_i = EvalIndirectionLight(wi, wo, screenUV) * IndCorff;

  vec3 result = L_d + L_ssr + L_i;
  vec3 color = pow(clamp(result, vec3(0.0), vec3(1.0)), vec3(1.0 / 2.2));
  gl_FragColor = vec4(vec3(color.rgb), 1.0);
}
```

只显示间接光照。采样数=5。

<img title="" src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/PicGo_dir/202311172103642.png" alt="" data-align="center">

直接光照+间接光照。采样数=5。

<img title="" src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/PicGo_dir/202311172106475.png" alt="" data-align="center">

> 写这个部分真是头痛啊，即使 `SAMPLE_NUM` 设置为1，我的电脑都汗流浃背了。Live Server一开，直接打字都有延迟了，受不了。M1pro就这么点性能了吗。而且最让我受不了的是，Safari浏览器卡就算了，为什么整个系统连带一起卡顿呢？这就是你macOS的User First策略吗？我不理解。迫不得已，我只能掏出我的游戏电脑通过局域网测试项目了（悲）。只是没想到RTX3070运行起来也有点大汗淋漓，**看来我写的算法就是一坨狗屎，我的人生也是一坨狗屎啊**。

## 4. RayMarch改进

目前的 `RayMarch()` 其实是有问题的，会出现漏光的现象。

<img title="" src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/PicGo_dir/202311181936726.png" alt="" data-align="center">

在采样数为5的情况下只有46.2帧左右。我的设备是M1pro 16GB。

<img title="" src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/PicGo_dir/202311201910160.png" alt="" data-align="center">

这里重点说说为什么会产生漏光的现象，看下面这个图示。我们gBuffer里只有蓝色部分的深度信息，即使我们上面的算法已经判断了当前 `curPos` 已经比gBuffer的深度要深了，这也不能确保这个 `curPos` 是否就是碰撞点。因此上面的算法并没有考虑图中的情况，进而导致漏光的现象。

<img title="" src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/PicGo_dir/202311201540548.png" alt="" width="317" data-align="center">

为了**解决漏光问题**，我们引入一个阈值 `thresholds` 解决这个问题（没错，又是一个近似），如果 `curPos` 和当前gBuffer记录的深度的差大于某个阈值，那就进入下图的情况。这个时候屏幕空间的信息没办法正确提供反射的信息，因此这个Shading Point的SSR结果就是 `vec3(0)` 。就是这么的简单粗暴！

<img title="" src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/PicGo_dir/202311201829456.png" alt="" width="311" data-align="center">

代码的思路跟前面的差不多，每一次步进时，判断下一步位置的深度与gBuffer的深度的关系，如果下一步的位置在gBuffer的前面（`nextDepth<gDepth`），则可以步进。如果下一步的深度没有gBuffer的深，就判断一下深度相差多少，有没有给定的阈值大。如果比阈值大，那么就直接返回 `false` ，否则，这个时候就可以执行SSR了。先让当前位置步进一个step，返回给 `hitPos` ，然后返回真。

```glsl
bool RayMarch(vec3 ori, vec3 dir, out vec3 hitPos) {
  const float EPS = 1e-2;
  const int totalStepTimes = 60;
  const float threshold = 0.1;
  float step = 0.05;
  vec3 stepDir = normalize(dir) * step;
  vec3 curPos = ori + stepDir;
  vec3 nextPos = curPos + stepDir;
  for(int i = 0; i < totalStepTimes; i++) {
    if(GetDepth(nextPos) < GetGBufferDepth(GetScreenCoordinate(nextPos))){
      curPos = nextPos;
      nextPos += stepDir;
    }else if(GetGBufferDepth(GetScreenCoordinate(curPos)) - GetDepth(curPos) + EPS > threshold){
      return false;
    }else{
      curPos += stepDir;
      vec2 screenUV = GetScreenCoordinate(curPos);
      float rayDepth = GetDepth(curPos);
      float gBufferDepth = GetGBufferDepth(screenUV);
      if(rayDepth > gBufferDepth + threshold){
        hitPos = curPos;
        return true;
      }
    }
  }
  return false;
}
```

但是帧率降到了42.6左右，但是却显著的改善了画面！至少是没有显著的漏光现象了。

<img title="" src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/PicGo_dir/202311201838404.png" alt="" data-align="center">

但是画面还有一些瑕疵，就是在边缘的时候会有发毛的反射图样，也就是说漏光问题依旧没有解决，如下图所示：

<img title="" src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/PicGo_dir/202311202100008.png" alt="" width="424" data-align="center">

上面的方法**确实是存在问题**的，在与阈值做对比的时候我们错误的使用了 `curPos` 来比较（即下图的Step n点），导致了代码也能进入第三个分支，返回那个错误 `curPos` 的 `hitPos` 。

<img title="" src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/PicGo_dir/202311202158305.png" alt="" data-align="center">

再退一步，我们没有办法保证最终计算的 `curPos` 正好落在物体边缘与摄像机原点的线上。说白了，就是下图中蓝色的线是相当离散的。我们想要得到“恰好”在边界的 `curPos` ，进而将「Step n」到「“恰好”的curPos」这段距离的瑕疵（即上面的毛刺错误）处理掉，但是显然因为各种精度的原因，我们没办法获得。下图中，绿色的线代表一次step。

<img title="" src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/PicGo_dir/202311202116592.png" alt="" data-align="center" width="472">

即使我们调整 `threshold/step` 的比值，使其接近1，我们也难以根除这个问题，最多只能起到缓解作用，就像下图所示。

<img title="" src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/PicGo_dir/202311202112175.png" alt="" data-align="center" width="462">

因此我们需要再次改进刚刚的「防漏光」方法。

换一句话说，就是让改进的思想也非常简单，既然我没办法获得“恰好”的 `curPos` 点，那我就把它猜出来。具体来说就是，直接来一个线性插值。插值之前再做一个近似，也就是将视线看作相互平行的，接着就像下图一样做一个相似三角形，猜出我们想要的 `curPos` ，然后把它当作 `hitPos` 。

<img title="" src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/PicGo_dir/202311202306844.png" alt="" data-align="center">

$$
\text{hitPos} = \text{curPos} + \frac{s1}{\text{s1}+\text{s2}}
$$

```glsl
bool RayMarch(vec3 ori, vec3 dir, out vec3 hitPos) {
  bool result = false;
  const float EPS = 1e-3;
  const int totalStepTimes = 60;
  const float threshold = 0.1;
  float step = 0.05;
  vec3 stepDir = normalize(dir) * step;
  vec3 curPos = ori + stepDir;
  vec3 nextPos = curPos + stepDir;
  for(int i = 0; i < totalStepTimes; i++) {
    if(GetDepth(nextPos) < GetGBufferDepth(GetScreenCoordinate(nextPos))){
      curPos = nextPos;
      nextPos += stepDir;
      continue;
    }
    float s1 = GetGBufferDepth(GetScreenCoordinate(curPos)) - GetDepth(curPos) + EPS;
    float s2 = GetDepth(nextPos) - GetGBufferDepth(GetScreenCoordinate(nextPos)) + EPS;
    if(s1 < threshold && s2 < threshold){
      hitPos = curPos + stepDir * s1 / (s1 + s2);
      result = true;
    }
    break;
  }
  return result;
}
```

效果相当可以，没有鬼影和边界的瑕疵了。并且帧率也跟最开始的算法相似，在平均49.2左右。

<img title="" src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/PicGo_dir/202311202332914.png" alt="" data-align="center">

接下来重点优化一下性能，具体而言就是：

- 加入自适应step

- 屏幕外忽略的判断

**屏幕外忽略的判断** 非常简单。如果 `curPos` 的uvScreen不在0到1之间，那么直接放弃当前步进。

详细说说自适应step。也就是在for的开头加上两行。实测帧率会稍微提高2-3帧左右。

```glsl
vec2 uvScreen = GetScreenCoordinate(curPos);
if(any(bvec4(lessThan(uvScreen, vec2(0.0)), greaterThan(uvScreen, vec2(1.0))))) break;
```

**自适应step** 也不难。首先为初始步进 `step` 设置一个较大的值，如果监测到**步进之后**的 `curPos`  **不在屏幕内** 或着 **深度值比gBuffer的深** 或者 **不满足“s1 < threshold && s2 < threshold”** ，那么就让step步进减半，以确保精度。

```glsl
bool RayMarch(vec3 ori, vec3 dir, out vec3 hitPos) {
  const float EPS = 1e-2;
  const int totalStepTimes = 20;
  const float threshold = 0.1;
  bool result = false, firstIn = false;
  float step = 0.8;
  vec3 curPos = ori;
  vec3 nextPos;
  for(int i = 0; i < totalStepTimes; i++) {
    nextPos = curPos+dir*step;
    vec2 uvScreen = GetScreenCoordinate(curPos);
    if(any(bvec4(lessThan(uvScreen, vec2(0.0)), greaterThan(uvScreen, vec2(1.0))))) break;
    if(GetDepth(nextPos) < GetGBufferDepth(GetScreenCoordinate(nextPos))){
      curPos += dir * step;
      if(firstIn) step *= 0.5;
      continue;
    }
    firstIn = true;
    if(step < EPS){
      float s1 = GetGBufferDepth(GetScreenCoordinate(curPos)) - GetDepth(curPos) + EPS;
      float s2 = GetDepth(nextPos) - GetGBufferDepth(GetScreenCoordinate(nextPos)) + EPS;
      if(s1 < threshold && s2 < threshold){
        hitPos = curPos + 2.0 * dir * step * s1 / (s1 + s2);
        result = true;
      }
      break;
    }
    if(firstIn) step *= 0.5;
  }
  return result;
}
```

改进了之后，帧率一下子来到了100帧，几乎翻倍了。

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/PicGo_dir/202311211603887.png"/>

最后整理一下代码。

```glsl
#define EPS 5e-2
#define TOTAL_STEP_TIMES 20
#define THRESHOLD 0.1
#define INIT_STEP 0.8
bool outScreen(vec3 curPos){
  vec2 uvScreen = GetScreenCoordinate(curPos);
  return any(bvec4(lessThan(uvScreen, vec2(0.0)), greaterThan(uvScreen, vec2(1.0))));
}
bool testDepth(vec3 nextPos){
  return GetDepth(nextPos) < GetGBufferDepth(GetScreenCoordinate(nextPos));
}
bool RayMarch(vec3 ori, vec3 dir, out vec3 hitPos) {
  float step = INIT_STEP;
  bool result = false, firstIn = false;
  vec3 nextPos, curPos = ori;
  for(int i = 0; i < TOTAL_STEP_TIMES; i++) {
    nextPos = curPos + dir * step;
    if(outScreen(curPos)) break;
    if(testDepth(nextPos)){ // 可以进步
      curPos += dir * step;
      continue;
    }else{ // 过于进步了
      firstIn = true;
      if(step < EPS){
        float s1 = GetGBufferDepth(GetScreenCoordinate(curPos)) - GetDepth(curPos) + EPS;
        float s2 = GetDepth(nextPos) - GetGBufferDepth(GetScreenCoordinate(nextPos)) + EPS;
        if(s1 < THRESHOLD && s2 < THRESHOLD){
          hitPos = curPos + 2.0 * dir * step * s1 / (s1 + s2);
          result = true;
        }
        break;
      }
      if(firstIn) step *= 0.5;
    }
  }
  return result;
}
```

切换到洞穴场景，采样率设置为32，帧率就只有可怜的4帧了。

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/PicGo_dir/202311211711383.png"/>

而且不过次级光源质量非常不错了。

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/PicGo_dir/202311211713147.png"/>

然而这个算法运用在反射上就会导致新的问题了。尤其是下边这张图，走样非常严重。

<img title="" src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/PicGo_dir/202311211748769.png" alt="" data-align="center">

<img title="" src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/PicGo_dir/202311211754458.png" alt="" data-align="center">

## 5. Mipmap实现



[Hierarchical-Z map based occlusion culling](https://www.rastergrid.com/blog/2010/10/hierarchical-z-map-based-occlusion-culling/)



## 6. LocalBasis构建TBN原理

一般来说，构建法切副（法线、切线、副切线向量）是通过叉积来实现，实现方法非常简单，先任选一个跟法线向量不平行的辅助向量，两者做一次叉积得到第一个切线向量，然后这个切线向量和法线向量又做一次叉积，得到副切线向量。具体代码是这样写：

```cpp
void CalculateTBN(const vec3 &normal, vec3 &tangent, vec3 &bitangent) {
    vec3 helperVec;
    if (abs(normal.x) < abs(normal.y))
        helperVec = vec3(1.0, 0.0, 0.0);
    else
        helperVec = vec3(0.0, 1.0, 0.0);
    tangent = normalize(cross(helperVec, normal));
    bitangent = normalize(cross(normal, tangent));
}
```

但是作业框架中的代码避免了使用**叉积**，非常巧妙。简单的说，就是确保向量间的**点积**都是0。

- $b1⋅n=0$

- $b2⋅n=0$

- $b1⋅b2=0$

```glsl
void LocalBasis(vec3 n, out vec3 b1, out vec3 b2) {
  float sign_ = sign(n.z);
  if (n.z == 0.0) {
    sign_ = 1.0;
  }
  float a = -1.0 / (sign_ + n.z);
  float b = n.x * n.y * a;
  b1 = vec3(1.0 + sign_ * n.x * n.x * a, sign_ * b, -sign_ * n.x);
  b2 = vec3(b, sign_ + n.y * n.y * a, -n.y);
}
```

这个算法是一个比较启发式的，引入了一个符号函数，相当有逼格。还考虑了除0的情况，格局也是拉满。但是下面这四行，应该是作者不知道在哪一天写公式的时候随便乱拆给他拆出来的而已，这里我还原一下当时作者的拆解步骤。也就是倒推的过程。

<img title="" src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/PicGo_dir/202311212255583.png" alt="" data-align="center">

另外说一下，代码中的符号函数可以在最后一步乘上。

实际上，这样的公式我可以造出一百个，我也不知道这些个公式之间有啥区别，知道的小伙伴请告诉我QAQ。如果硬要说，那么就可以这样解释：

> 传统的基于叉乘的方法可能会产生数值不稳定，因为叉乘结果在这种情况下接近于零向量。
> 
> 本文采用的方法是一种启发式方法，它通过一系列精心设计的步骤来构造正交基。这种方法特别注意了数值稳定性，使其在处理接近极端方向的法线向量时仍然有效和稳定。

## References

1. Games 202
2. [LearnOpenGL - Normal Mapping](https://learnopengl.com/Advanced-Lighting/Normal-Mapping)

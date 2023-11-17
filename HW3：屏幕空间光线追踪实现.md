## TODO List

- 实现对场景直接光照的着色 (考虑阴影)。

- 实现屏幕空间下光线的求交 (SSR)。

- 实现对场景间接光照的着色。

- Bonus 1：实现 Mipmap 优化的 Screen Space Ray Tracing。

## 写在前面

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

```glsl
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

## 2. 镜面SSR

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
  const int totalStepTimes = 150;
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

首先需要知道，我们本次采样仍然是基于屏幕空间的。因此不在屏幕（gBuffer）中的内容我们就当作不存在。可以理解为只有一层正好面向摄像机的外壳。

间接光照涉及上半球方向的随机采样和对应pdf计算。使用 `InitRand(screenUV)` 得到随机数就可以了，然后二选一， `SampleHemisphereUniform(inout float s, out float pdf)` 或 `SampleHemisphereCos(inout float s, out float pdf)` ，更新随机数同时得到对应 `pdf` 和单位半球上的局部坐标系的位置 `dir` 。

将当前Shading Point的法线坐标传入函数 `LocalBasis(n, out b1, out b2)` ，随后返回 `b1, b2` ，其中 `n, b1, b2` 这三个单位向量两两正交。通过这三个向量所构成的局部坐标系，将 `dir` 转换到世界坐标中。

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

## TODO List

- 实现对场景直接光照的着色 (考虑阴影)。

- 实现屏幕空间下光线的求交 (SSR)。

- 实现对场景间接光照的着色。

- Bonus 1：实现 Mipmap 优化的 Screen Space Ray Tracing。

## 写在前面 - 关于框架的深度缓冲问题

这一次作业在 macOS 上会遇到比较严重的问题。正方体贴近地面的部分会随着摄像机的距离远近变化表现出异常的裁切锯齿问题。这个现象在 windows 上没有遇到，比较奇怪。

<img title="" src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/PicGo_dir/202311141324851.png" alt="" width="501" data-align="center">

个人感觉这与深度缓冲区的精度有关，可能是z-fighting导致的，其中两个或更多重叠的表面竞争同一像素的问题。对于这种问题一般下面几种解决方案：

- 调整近平面和远平面：不要让近平面离摄像机太近，远平面不要太远。

- 提高深度缓冲区的精度：采用32位或者更高的精度。

- 多通渲染（Multi-Pass Rendering）：对不同距离范围的物体采用不同的渲染方案。

最简单的解决办法就是修改近平面的大小，定位到框架的 `engine.js` 的25行。

```js
// engine.js
// const camera = new THREE.PerspectiveCamera(75, gl.canvas.clientWidth / gl.canvas.clientHeight, 1, 1e5);
const camera = new THREE.PerspectiveCamera(75, gl.canvas.clientWidth / gl.canvas.clientHeight, 5e-2, 1e2);![loading-ag-6753]()
```

这样就可以得到相当锐利的边界了。

<img title="" src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/PicGo_dir/202311141528946.png" alt="" width="462" data-align="center">

## 1. 实现直接光照

> 实现「shaders/ssrShader/ssrFragment.glsl」中的 `EvalDiffuse(vec3 wi, vec3 wo, vec2 uv)` 和 `EvalDirectionalLight(vec2 uv)` 。

```glsl
// ssrFragment.glsl
vec3 EvalDiffuse(vec3 wi, vec3 wo, vec2 uv) {
  vec3 reflectivity = GetGBufferDiffuse(uv);
  vec3 normal = GetGBufferNormalWorld(uv);
  float cosi = max(0., dot(normal, wi));
  vec3 f_r = reflectivity * cosi * INV_PI;
  return f_r;
}

vec3 EvalDirectionalLight(vec2 uv) {
  vec3 Li = uLightRadiance * GetGBufferuShadow(uv);
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

## 2. Screen Space Ray Tracing

> 实现 `RayMarch(ori, dir, out hitPos)` 函数，求出光线与物体的交点，返回光线是否与物体相交。参数 ori 和 dir 为世界坐标系中的值，分别代表光线的起点和方向，其中方向向量为单位向量。

作业框架的「cube1」本身就包含了地面，所以这玩意最终得到的SSR效果就不太美观。这里的“美观”是指论文中结果图的清晰度或游戏中积水反射效果的精致度。



```glsl
bool RayMarch(vec3 ori, vec3 dir, out vec3 hitPos) {
  float step = 0.05;
  const int totalStepTimes = 150; 
  int curStepTimes = 0;

  vec3 stepDir = normalize(dir) * step;
  vec3 curPos = ori;
  for(int curStepTimes = 0; curStepTimes < totalStepTimes; curStepTimes++){
    vec2 screenUV = GetScreenCoordinate(curPos);
    float rayDepth = GetDepth(curPos);
    float gBufferDepth = GetGBufferDepth(screenUV);

    if(rayDepth - gBufferDepth > 0.0001){
      hitPos = curPos;
      return true;
    }

    curPos += stepDir;
  }

  return false;
}
```



```glsl
vec3 EvalSSR(vec3 wi, vec3 wo, vec2 uv) {
  vec3 worldNormal = GetGBufferNormalWorld(uv);
  vec3 relfectDir = normalize(reflect(-wo, worldNormal));
  vec3 hitPos;
  if(RayMarch(vPosWorld.xyz, relfectDir, hitPos)){
      vec2 screenUV = GetScreenCoordinate(hitPos);
      return GetGBufferDiffuse(screenUV);
  }
  else{
    return vec3(0.); 
  }
}
```



```glsl
void main() {
  float s = InitRand(gl_FragCoord.xy);
  vec3 L = vec3(0.0);
  vec3 wi = normalize(uLightDir);
  vec3 wo = normalize(uCameraPos - vPosWorld.xyz);
  vec2 screenUV = GetScreenCoordinate(vPosWorld.xyz);

  float reflectivity = 0.2;

  L = EvalDiffuse(wi, wo, screenUV) * EvalDirectionalLight(screenUV);
  L+= EvalSSR(wi, wo, screenUV) * reflectivity;

  vec3 color = pow(clamp(L, vec3(0.0), vec3(1.0)), vec3(1.0 / 2.2));
  gl_FragColor = vec4(vec3(color.rgb), 1.0);
}
```





<img title="" src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/PicGo_dir/202311161910641.png" alt="" data-align="center">

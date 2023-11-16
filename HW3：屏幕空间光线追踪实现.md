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

## 

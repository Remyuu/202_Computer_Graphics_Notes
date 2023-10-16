本文内容：JS和WebGL相关知识、2-pass shadow算法、BIAS缓解自遮挡、PCF算法、PCSS、**物体移动**。

**不需要您事先掌握JavaScript或者是WebGL甚至是GLSL，因为我会借助本项目框架带您从不一样的角度零基础入门。**

从片元着色器出发讲解了GLSL是怎么与项目产生联系的，借此串联整个项目流程。并且还利用 matplotlib 通过**图片**与**动画**简单分析了Poisson Disk的部分参数的物理意义。物体移动功能需要大量修改框架，因为物体平移功能比较繁杂，需要对整个框架有细致的了解。

项目参考代码：https://github.com/Remyuu/GAMES202-HW1

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20230806221258288.png" alt="image-20230806221258288" style="zoom: 33%;" />

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20230808224931842.png" alt="image-20230808224931842" style="zoom:33%;" />

<!--more-->

[TOC]

## 写在前面

我个人目前对JS以及WebGL的熟练度不高，遇到不懂的代码和用法我会在本文做记录。遇事不决 `console.log()` 就能解决50%的问题。

除了作业框架要求的内容，我在coding的时候也有一些思考与疑问。

1. 如何实现动态的点光源阴影效果？我们需要使用点光源阴影技术才可以实现万向阴影贴图（omnidirectional shadow maps）。
1. `possionDiskSamples`函数并不是真正的泊松圆盘分布？

## 框架修正

在作业开始时请先对作业框架做一些修正。框架改动原文：https://games-cn.org/forums/topic/zuoyeziliao-daimakanwu/

- 框架提供的 unpack 函数算法实现不准确，在不加 bias 时，会导致严重的 banding（地面一半白一半黑而不是典型的 z-fighting 效果），一定程度上影响作业调试。

```glsl
// homework1/src/shaders/shadowShader/shadowFragment.glsl
vec4 pack (float depth) {
    // 使用rgba 4字节共32位来存储z值,1个字节精度为1/255
    const vec4 bitShift = vec4(1.0, 255.0, 255.0 * 255.0, 255.0 * 255.0 * 255.0);
    const vec4 bitMask = vec4(1.0/255.0, 1.0/255.0, 1.0/255.0, 0.0);
    // gl_FragCoord:片元的坐标,fract():返回数值的小数部分
    vec4 rgbaDepth = fract(depth * bitShift); //计算每个点的z值
    rgbaDepth -= rgbaDepth.gbaa * bitMask; // Cut off the value which do not fit in 8 bits
    return rgbaDepth;
}

// homework1/src/shaders/phongShader/phongFragment.glsl
float unpack(vec4 rgbaDepth) {
    const vec4 bitShift = vec4(1.0, 1.0/255.0, 1.0/(255.0*255.0), 1.0/(255.0*255.0*255.0));
    return dot(rgbaDepth, bitShift);
}
```

- 清屏还需要添加一个glClear。

```js
// homework1/src/renderers/WebGLRenderer.js
gl.clearColor(0.0, 0.0, 0.0,1.0);// Clear to black, fully opaque
gl.clearDepth(1.0);// Clear everything
gl.clear(gl.COLOR_BUFFER_BIT | gl.DEPTH_BUFFER_BIT);
```

## JS最基础的知识

### 变量

- 在JavaScript中，我们主要使用 `var`，`let` 和 `const` 这三个关键字来声明变量/常量。

  - `var`是声明变量的关键字，可以在**整个函数范围**内使用声明的**变量**（函数作用域）。

  - `let`行为与 `var` 类似，也是声明了一个**变量**，但是 `let` 的**作用域限制在块中**（块作用域），比如 `for` 循环或 `if` 语句中定义的块。

  - `const`：用于声明**常量**。 `const` 的作用域也是**块级别**的。

- 推荐使用 `let` 和 `const` 而不是 `var` 来声明变量，因为它们遵循块级作用域，更符合大多数编程语言中的作用域规则，更易理解和预测。

### 类

一个基本的JavaScript类的结构如下：

```js
class MyClass {
  constructor(parameter1, parameter2) {
    this.property1 = parameter1;
    this.property2 = parameter2;
  }
  method1() {
    // method body
  }
  static sayHello() {
    console.log('Hello!');
  }
}
```

创建实例：

```js
let myInstance = new MyClass('value1', 'value2');
myInstance.method1(); // 调用类的方法
```

也可以直接调用静态类（不用创建实例了）：

```js
MyClass.sayHello();  // "Hello!"
```

## 项目流程简述

程序入口是engine.js，主函数 GAMES202Main 。首先初始化WebGL相关的内容，包括相机、相机交互、渲染器、光源、物体加载、用户GUI界面以及最重要的主循环main loop部分。

物体加载过程，会调用loadOBJ.js。首先从文件中加载对应的glsl，构建Phong材质、Phong相关阴影还有阴影的材质。

```js
// loadOBJ.js
case 'PhongMaterial':
    material = buildPhongMaterial(colorMap, mat.specular.toArray(), light, Translation, Scale, "./src/shaders/phongShader/phongVertex.glsl", "./src/shaders/phongShader/phongFragment.glsl");
    shadowMaterial = buildShadowMaterial(light, Translation, Scale, "./src/shaders/shadowShader/shadowVertex.glsl", "./src/shaders/shadowShader/shadowFragment.glsl");
    break;
}
```

然后，通过MeshRender直接生成2-pass阴影Shadow Map和常规的Phong材质，具体代码如下：

```js
// loadOBJ.js
material.then((data) => {
    // console.log("现在制作表面材质")
    let meshRender = new MeshRender(renderer.gl, mesh, data);
    renderer.addMeshRender(meshRender);
});
shadowMaterial.then((data) => {
    // console.log("现在制作阴影材质")
    let shadowMeshRender = new MeshRender(renderer.gl, mesh, data);
    renderer.addShadowMeshRender(shadowMeshRender);
});
```

注意到，MeshRender具备一定的通用性，它接受任何类型的材质作为其参数。具体是怎么区分的呢？通过判断传入的material.frameBuffer是否为空，如果是空，将加载表面材质，否则加载阴影图Shadow Map。在MeshRender.js的draw()函数中，看到如下代码：

```js
// MeshRender.js
if (this.material.frameBuffer != null) {
    // Shadow map
    gl.viewport(0.0, 0.0, resolution, resolution);
} else {
    gl.viewport(0.0, 0.0, window.screen.width, window.screen.height);
}
```

利用MeshRender生成了阴影之后，推入到renderer中，可以在 WebGLRenderer.js 中找到对应实现：

```js
addShadowMeshRender(mesh) { this.shadowMeshes.push(mesh); }
```

最后进入mainLoop()主循环实现一帧帧的更新画面。

## 项目流程详细解释

这一章节将会从一个小问题出发，探讨片段着色器是如何构造的。这将会串联起几乎整个项目，而这也是我认为比较舒服的阅读项目流程。

### glsl是在哪里工作？ -- 从片段着色器的流程入手详细讲解代码流程

在上文中我们并没有详细提及glsl文件是怎么调用的，这里我们详细说说。

首先在**loadOBJ.js**中首次用过路径的方式将.glsl文件引入：

```js
// loadOBJ.js - function loadOBJ()
material = buildPhongMaterial(colorMap, mat.specular.toArray(), light, Translation, Scale, "./src/shaders/phongShader/phongVertex.glsl", "./src/shaders/phongShader/phongFragment.glsl");
shadowMaterial = buildShadowMaterial(light, Translation, Scale, "./src/shaders/shadowShader/shadowVertex.glsl", "./src/shaders/shadowShader/shadowFragment.glsl");
```

这里以phongFragment.glsl为例子，phongFragment.glsl通过位于 PhongMaterial.js 的`buildPhongMaterial`函数中的 `getShaderString`方法将glsl代码从硬盘中加载进来，与此同时将glsl代码通过构造参数的形式传入并用之构造一个PhongMaterial对象。PhongMaterial在构造的过程中会调用`super()`函数实现父类Material.js的构造函数，即将glsl代码传到Material.js中：

```js
// PhongMaterial.js
super({...}, [], ..., fragmentShader);
```

> 在c++中，子类可以选择是否完全继承父类的构造函数的参数。这里父类的构造函数有5个，实际只实现了4个，这也是完全没问题的。

在Material.js中，子类通过构造函数的第四个参数`#fsSrc`将glsl代码传到了此处。至此，glsl代码的传送之路就**走到了尽头**，接下来等待他的将是一个名为`compile()`的函数。

```js
// Material.js
this.#fsSrc = fsSrc;
...
compile(gl) {
    return new Shader(..., ..., this.#fsSrc,{...});
}
```

至于这个`compile`函数什么时候调用呢？回到loadOBJ.js的流程中，现在我们已经完全执行完毕`buildPhongMaterial()`代码，接下来就到了上一小节提及到的then()部分。

> 注意，loadOBJ()只是一个函数，不是对象！

```js
// loadOBJ.js
material.then((data) => {
    let meshRender = new MeshRender(renderer.gl, mesh, data);
    renderer.addMeshRender(meshRender);
    renderer.ObjectID[ObjectID][0].push(renderer.meshes.length - 1);
});
```

在构造`MeshRender`对象时，就会调用`compile()`：

```js
// MeshRender.js
constructor(gl, mesh, material) {
...
    this.shader = this.material.compile(gl);
}
// Material.js
compile(gl) {
    return new Shader(..., ..., this.#fsSrc,{...});
}
```

接下来，我们具体看一下shader.js的构造。Material在构造shader对象时实现了所有的四个构造参数。这里还是挑重点`fsSrc`看，即继续看看glsl代码接下来的命运。

```js
// shader.js
constructor(gl, vsSrc, fsSrc, shaderLocations) {
	...
    const fs = this.compileShader(fsSrc, ...);
	...
}
```

在构造shader对象实现`fs`编译着色器时是通过`compileShader()`函数的。这个`compileShader`函数会创建一个全局变量`shader`，代码如下：

```js
// shader.js
compileShader(shaderSource, shaderType) {
    const gl = this.gl;
    var shader = gl.createShader(shaderType);
    gl.shaderSource(shader, shaderSource);
    gl.compileShader(shader);

    if (!gl.getShaderParameter(shader, gl.COMPILE_STATUS)) {
        console.error(shaderSource);
        console.error('shader compiler error:\n' + gl.getShaderInfoLog(shader));
    }

    return shader;
};
```

这个`gl`是什么呢？是在`loadOBJ()`通过构造`MeshRender`对象时以参数`renderer.gl`一路传到shader.js中的。而`renderer`则是`loadOBJ()`的第一个参数，在engine.js中传入。

实际上loadOBJ.js中`renderer`是一个`WebGLRenderer`对象。而`renderer.gl`的`gl`是在engine.js中创建的：

```js
// engine.js
const gl = canvas.getContext('webgl');
```

而`gl`可以理解为从index.html中获取`canvas`的WebGL对象。实际上`gl`为开发者提供了一个接口来与WebGL API进行交互。

```html
<!-- index.html -->
<canvas id="glcanvas"></canvas>
```

> WebGL推荐参考资料：
>
> 1. https://developer.mozilla.org/en-US/docs/Web/API/WebGL_API
> 2. https://webglfundamentals.org
> 3. https://www.w3cschool.cn/webgl/vjxu1jt0.html
>
> Tips:网站都有对应的中文版本，但是有能力的还是推荐阅读英文版本～
>
> WebGL API：
>
> 1. https://developer.mozilla.org/en-US/docs/Web/API
> 2. https://webglfundamentals.org/docs/

知道了`gl`是什么之后，自然也就发现了项目框架是在哪里通过什么方式与WebGL联系在一起的了。

```js
// Shader.js
compileShader(shaderSource, shaderType) {
    const gl = this.gl;
    var shader = gl.createShader(shaderType);
    gl.shaderSource(shader, shaderSource);
    gl.compileShader(shader);

    if (!gl.getShaderParameter(shader, gl.COMPILE_STATUS)) {
        console.error(shaderSource);
        console.error('shader compiler error:\n' + gl.getShaderInfoLog(shader));
    }

    return shader;
};
```

也就是说，所有关于gl的方法都是通过WebGL API调用的。`gl.createShader`就是我们接触到的第一个WebGL API。

我们只需知道这个`createShader()`函数会返回 **WebGLShader 着色器对象**。我们在后文再详细说明，这里先关注`shaderSource`究竟何去何从。

- `gl.shaderSource`:A string containing the GLSL source code to set.

也就是说，我们一路追踪的GLSL源代码是通过`gl.shaderSource`函数解析进了WebGLShader中。

然后通过`gl.compileShader()`函数编译WebGLShader，使其成为为二进制数据，然后就可以被`WebGLProgram`对象所使用。

> 简单地说，`WebGLProgram`是一个包含已编译的WebGL着色器的GLSL程序，至少需要包含一个顶点着色器和一个片段着色器。在WebGL中，会创建一个或多个`WebGLProgram`对象，每个对象包含一组特定的渲染指令。通过使用不同的`WebGLProgram`，就可以实现各种画面。

`if`语句是检查着色器是否成功编译的部分。如果编译失败，则执行括号内的代码。最后，返回编译后（或尝试编译后）的着色器对象`shader`。

至此，我们就完成了将GLSL文件从硬盘中取出最后编译进着色器对象的工作。

但是渲染的流程还没有结束。回到Shadow对象的构造处：

```js
// Shadow.js
class Shader {
    constructor(gl, vsSrc, fsSrc, shaderLocations) {
        this.gl = gl;
        const vs = this.compileShader(vsSrc, gl.VERTEX_SHADER);
        const fs = this.compileShader(fsSrc, gl.FRAGMENT_SHADER);

        this.program = this.addShaderLocations({
            glShaderProgram: this.linkShader(vs, fs),
        }, shaderLocations);
    }
    ...
```

虽然刚才我们只解说了片段着色器的GLSL编译流程，但是顶点着色器也是相当类似的，故此省略。

----

这里我们介绍`linkShader()`链接着色器的流程。代码在文字的下方。

1. 首先创建一个**创建程序**命名为WebGLProgram。

2. 将编译后的顶点着色器和片段着色器`vs`和`fs`添加到程序中，这一步叫做**附加着色器**。具体而言是使用`gl.attachShader()`将他们附加到`WebGLProgram`上。
2. 使用`gl.linkProgram()`链接`WebGLProgram`。这会生成一个可执行的程序，该程序结合了前面附加的着色器。这一步叫做**链接程序**。
2. 最后检查链接状态，返回WebGL对象。

```js
// Shader.js
linkShader(vs, fs) {
    const gl = this.gl;
    var prog = gl.createProgram();
    gl.attachShader(prog, vs);
    gl.attachShader(prog, fs);
    gl.linkProgram(prog);

    if (!gl.getProgramParameter(prog, gl.LINK_STATUS)) {
        abort('shader linker error:\n' + gl.getProgramInfoLog(prog));
    }
    return prog;
};
```

>`WebGLProgram`可以被视为着色器的容器，它包含了将3D数据转换为屏幕上的2D像素所需的全部信息和指令。

----

得到与着色器链接的程序`glShaderProgram`后，会与`shaderLocations`对象一同被载入。

>简单地说`shaderLocations`对象包含了两个属性
>
>- Attributes是"个体"的数据（比如每个顶点的信息）
>- Uniforms是"整体"的数据（比如一个灯光的信息）

框架将载入的流程打包进了`addShaderLocations()`中。简单地说，经过这一步操作之后，当你需要给这些uniform和attribute赋值时，就可以直接通过已经获取到的位置进行操作，而不需要每次都去查询位置。

```js
addShaderLocations(result, shaderLocations) {
    const gl = this.gl;
    result.uniforms = {};
    result.attribs = {};

    if (shaderLocations && shaderLocations.uniforms && shaderLocations.uniforms.length) {
        for (let i = 0; i < shaderLocations.uniforms.length; ++i) {
            result.uniforms = Object.assign(result.uniforms, {
                [shaderLocations.uniforms[i]]: gl.getUniformLocation(result.glShaderProgram, shaderLocations.uniforms[i]),
            });
        }
    }
    if (shaderLocations && shaderLocations.attribs && shaderLocations.attribs.length) {
        for (let i = 0; i < shaderLocations.attribs.length; ++i) {
            result.attribs = Object.assign(result.attribs, {
                [shaderLocations.attribs[i]]: gl.getAttribLocation(result.glShaderProgram, shaderLocations.attribs[i]),
            });
        }
    }

    return result;
}
```

回顾一下目前已经完成的工作：成功构建好了一个编译后（或尝试编译后）的`Shader`着色器对象给`MeshRender`：

```js
// MeshRender.js - construct()
this.shader = this.material.compile(gl);
```

至此，`loadOBJ`的任务已经圆满完成。在engine.js中，这样的加载要做三次：

```js
	// loadOBJ(renderer, path, name, objMaterial, transform, meshID);
loadOBJ(renderer, 'assets/mary/', 'Marry', 'PhongMaterial', obj1Transform);
loadOBJ(renderer, 'assets/mary/', 'Marry', 'PhongMaterial', obj2Transform);
loadOBJ(renderer, 'assets/floor/', 'floor', 'PhongMaterial', floorTransform);
```

接下来来到程式主循环mainLoop。也即，一个循环表示一帧：

```js
// engine.js
loadOBJ(...);
...
function mainLoop() {...}
...   
```

### 程序主循环 -- mainLoop()

实际上，执行`mainLoop`，该函数会再次调用自己，形成一个无限循环。这就是所谓的游戏循环或动画循环的基础机制。

```js
// engine.js
function mainLoop() {
    cameraControls.update();
    renderer.render();
    requestAnimationFrame(mainLoop);
};
requestAnimationFrame(mainLoop);     
```

`cameraControls.update();`在更新相机的位置或方向，例如响应用户的输入。

`renderer.render();`场景被渲染或绘制到屏幕上。具体的渲染内容和方式取决于`renderer`对象的实现。

`requestAnimationFrame`的好处是它会尽量与屏幕的刷新率同步，这样可以提供更流畅的动画和更高的性能，因为它不会在屏幕刷新之间无谓地执行代码。

> 关于requestAnimationFrame()函数的详细信息可以参考以下文章：
>
> https://developer.mozilla.org/en-US/docs/Web/API/window/requestAnimationFrame

接下来重点关心`render()`函数的运作。

### `render()`渲染函数

这是一个典型的光源渲染、阴影渲染和最终摄像机视角渲染的流程。此处就不详细展开了，放到后面的多光源部分。

```js
// WebGLRenderer.js - render()
const gl = this.gl;

gl.clearColor(0.0, 0.0, 0.0, 1.0); // shadowmap默认白色（无遮挡），解决地面边缘产生阴影的问题（因为地面外采样不到，默认值为0会认为是被遮挡）
gl.clearDepth(1.0);// Clear everything
gl.enable(gl.DEPTH_TEST); // Enable depth testing
gl.depthFunc(gl.LEQUAL); // Near things obscure far things

console.assert(this.lights.length != 0, "No light");
console.assert(this.lights.length == 1, "Multiple lights");

for (let l = 0; l < this.lights.length; l++) {
    gl.bindFramebuffer(gl.FRAMEBUFFER, this.lights[l].entity.fbo);
    gl.clear(gl.DEPTH_BUFFER_BIT);
    // Draw light
    // TODO: Support all kinds of transform
    this.lights[l].meshRender.mesh.transform.translate = this.lights[l].entity.lightPos;
    this.lights[l].meshRender.draw(this.camera);

    // Shadow pass
    if (this.lights[l].entity.hasShadowMap == true) {
        for (let i = 0; i < this.shadowMeshes.length; i++) {
            this.shadowMeshes[i].draw(this.camera);
        }
    }
}
// Camera pass
for (let i = 0; i < this.meshes.length; i++) {
    this.gl.useProgram(this.meshes[i].shader.program.glShaderProgram);
    this.gl.uniform3fv(this.meshes[i].shader.program.uniforms.uLightPos, this.lights[0].entity.lightPos);
    this.meshes[i].draw(this.camera);
}
```

## GLSL快速入门 -- 分析片段着色器FragmentShader.glsl

上文我们讨论了如何载入GLSL，这一章节介绍GLSL的概念与实际用法。

在WebGL中进行渲染时，我们需要至少一个**顶点着色器（Vertex Shader）**和一个**片段着色器（Fragment Shader）**才能绘制出一幅画面。上一节我们以片段着色器为例，介绍了框架是怎么将GLSL文件从硬盘读取进renderer的。接下来我们也以Flagment Shader片段着色器为例子（即phongFragment.glsl），介绍编写GLSL的流程。

### FragmentShader.glsl有什么用？

Fragment Shader的作用是在光栅化的时候为当前像素渲染正确的颜色。以下是一个Fragment Shader的最简单形式，其包含一个`main()`函数，在函数其中指定了当前像素的颜色`gl_FragColor`。

```glsl
void main(void){
    ...
    gl_FragColor = vec4(Color, 1.0);
}
```

### Fragment Shader接受什么数据？

Fragment Shader需要知道数据，数据是由以下三种主要方式提供的，具体的用法可以参考 **附录1.6** ：

1. **Uniforms (全局变量)**: 这些是在单个绘制调用中对所有顶点和片段都保持不变的值。常见的例子包括变换矩阵（平移旋转等操作）、光源参数和材质属性。由于它们在绘制调用中是恒定的，所以称为“uniform”。
2. **Textures (纹理)**: 纹理是图像数据数组，它们可以被片段着色器采样来为每个片段获得颜色、法线或其他类型的信息。
3. **Varyings (可变量)**: 这些是顶点着色器输出的值，它们在图形基元（如三角形）的顶点之间插值，并传递给片段着色器。这允许我们在顶点着色器中计算值（如变换后的位置或顶点颜色），并在片段之间进行插值，以便在片段着色器中使用。

项目中用了Uniforms和Varyings两种。

### GLSL基本语法

这里不会把基本的用法过一篇，因为那样太无聊了。我们直接看项目：

```js
// phongFragment.glsl - PCF pass
void main(void) {
    // 声明变量
    float visibility;     // 可见性（用于阴影）
    vec3 shadingPoint;     // 从光源处的视点坐标
    vec3 phongColor;      // 计算出的Phong光照颜色

    // 将vPositionFromLight的坐标值归一化到[0,1]范围内
    shadingPoint = vPositionFromLight.xyz / vPositionFromLight.w;
    shadingPoint = shadingPoint * 0.5 + 0.5; // 进行坐标转换，使其在[0,1]范围内

    // 计算可见性（阴影）。
    visibility = PCF(uShadowMap, vec4(shadingPoint, 1.0)); // 使用PCF（Percentage Closer Filtering）技术

    // 使用blinnPhong()函数计算Phong光照颜色
    phongColor = blinnPhong();

    // 计算最终的片段颜色，将Phong光照颜色与可见性相乘，得到考虑阴影的片段颜色
    gl_FragColor = vec4(phongColor * visibility, 1.0);
}
```

和c语言一样，glsl是强类型语言，你不能这样赋值：`float visibility = 1;`，因为`1`是int类型。

#### 矢量或矩阵

另外，glsl还内置了很多特别的类型，比如浮点类型向量`vec2`, `vec3`和 `vec4`，矩阵类型`mat2`, `mat3` 和 `mat4`。

上面这些数据的访问方式也比较有意思，

- `.xyzw`：通常用于表示三维或四维空间中的点或向量。
- `.rgba`：当向量表示颜色时使用，其中r代表红色，g代表绿色，b代表蓝色，a代表透明度。
- `.stpq`：当向量用作纹理坐标时使用。

因此，

- `v.x` 与 `v[0]` 与 `v.r` 与 `v.s` 都表示该向量的第一个分量。
- `v.y` 与 `v[1]` 与 `v.g` 与 `v.t` 都表示该向量的第二个分量。
- 对于`vec3`和`vec4`，`v.z` 与 `v[2]` 与 `v.b` 与 `v.p` 都表示该向量的第三个分量。
- 对于`vec4`，`v.w` 与 `v[3]` 与 `v.a` 与 `v.q` 都表示该向量的第四个分量。

你甚至可以使用一种叫“分量重组”或“分量选择”的方式访问这些类型的数据：

1. **重复某个分量**:
   - `v.yyyy` 会得到一个新的`vec4`，其中每个分量都是原始`v`的`y`分量。这与`vec4(v.y, v.y, v.y, v.y)`的效果相同。
2. **交换分量**:
   - `v.bgra` 会得到一个新的`vec4`，其中的分量按照`b`, `g`, `r`, `a`的顺序从`v`中选取。这与`vec4(v.b, v.g, v.r, v.a)`的效果相同。

当构造一个矢量或矩阵时可以一次提供多个分量，例如：

- `vec4(v.rgb, 1)`与`vec4(v.r, v.g, v.b, 1)`是等价的
- `vec4(1)` 与`vec(1, 1, 1, 1)`也是等价的

> 参考资料：GLSL语言规范
>
> 1. https://www.khronos.org/files/opengles_shading_language.pdf

### 矩阵存储方式

这些提示都可以在glmatrix的Doc中找到：https://glmatrix.net/docs/mat4.js.html。另外，如果看得仔细我们会发现这个组件也都是用**列优先存储**矩阵的，WebGL和GLSL中也是列有限存储。如下所示：
$$
\left( \begin{array}{llll}
0 & 4 & 8 & 12 \\
1 & 5 & 9 & 13 \\
2 & 6 & 10 & 14 \\
3 & 7 & 11 & 15
\end{array}
\right)
$$
将一个物体移动到一个新的位置，可以用`mat4.translate()`函数，并且这个函数接受三个参数分别是：一个4x4的输出out，传入的4x4矩阵a，一个1x3的位移矩阵v。

最简单的矩阵乘法可以使用`mat4.multiply`，缩放矩阵使用`mat4.scale()`，调整“看向”的方向使用`mat4.lookAt()`，正交投影矩阵`mat4.ortho()`。

## 实现光源相机的矩阵变换

如果我们用透视投影操作，则是这里需要将下面Frustum放缩到一个正交视角的空间，如下图所示：

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20230815213511332.png" alt="image-20230815213511332" style="zoom:50%;" />

但是如果我们使用正交投影，那么就可以保持深度值的线性，使得 Shadow Map 的精度尽可能大。

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20230815223607948.png" alt="image-20230815223607948" style="zoom:50%;" />

```js
// DirectionalLight.js - CalcLightMVP()
let lightMVP = mat4.create();
let modelMatrix = mat4.create();
let viewMatrix = mat4.create();
let projectionMatrix = mat4.create();

// Model transform
mat4.translate(modelMatrix, modelMatrix, translate);
mat4.scale(modelMatrix, modelMatrix, scale);

// View transform
mat4.lookAt(viewMatrix, this.lightPos, this.focalPoint, this.lightUp);

// Projection transform
let left = -100.0, right = -left, bottom = -100.0, top = -bottom, 
    near = 0.1, far = 1024.0;  
    // Set these values as per your requirement
mat4.ortho(projectionMatrix, left, right, bottom, top, near, far);


mat4.multiply(lightMVP, projectionMatrix, viewMatrix);
mat4.multiply(lightMVP, lightMVP, modelMatrix);

return lightMVP;
```

## 2-Pass Shadow 算法

在实现两趟算法之前，先看看`main()`函数是怎么调用的。

```js
// phongFragment.glsl
void main(void){  
  vec3 shadingPoint = vPositionFromLight.xyz / vPositionFromLight.w;
  shadingPoint = shadingPoint*0.5+0.5;// 归一化至 [0,1]
  
  float visibility = 1.0;
  visibility = useShadowMap(uShadowMap, vec4(shadingPoint, 1.0));

  vec3 phongColor = blinnPhong();
  
  gl_FragColor=vec4(phongColor * visibility,1.0);
}
```

那么问题来了，`vPositionFromLight`是怎么来的？是在顶点着色器中算出来的。

### 统一空间坐标

说人话就是，将场景的顶点的世界坐标转换为光相机的NDC空间对应的新坐标。目的为了渲染主相机的某个Shading Point的阴影时，可以在光源相机的空间中取出所需的深度值。

`vPositionFromLight`表示从光源的视角看到的一个点的齐次坐标。这个坐标在光源的正交空间中，其范围是`[-w, w]`。他是由`phongVertex.glsl`计算出来的。`phongVertex.glsl`的作用是处理输入的顶点数据，通过上一章计算的MVP矩阵将一系列顶点转化为裁剪空间坐标。将`vPositionFromLight`转换到NDC标准空间得到`shadingPoint`，就可以将`shadingPoint`里面这些需要做阴影判断的Shading Point传入`useShadowMap`函数中。附上顶点转换的相关代码：

```glsl
// phongVertex.glsl - main()
vFragPos = (uModelMatrix * vec4(aVertexPosition, 1.0)).xyz;
vNormal = (uModelMatrix * vec4(aNormalPosition, 0.0)).xyz;

gl_Position = uProjectionMatrix * uViewMatrix * uModelMatrix *
            vec4(aVertexPosition, 1.0);

vTextureCoord = aTextureCoord;
vPositionFromLight = uLightMVP * vec4(aVertexPosition, 1.0);
```

> `phongVertex.glsl`是和`phongFragment.glsl`一同在loadOBJ.js中被加载的。

### 比对深度值

接下来实现`useShadowMap()`函数。这个函数的目的是为了确定片段（像素）是否在阴影中。

`texture2D()` 是一个GLSL的内置函数，用于对2D纹理进行采样。

代码框架中的`unpack()`和`pack()`函数是为了增加数值精度而设置的。原因如下：

- 深度信息是一个连续的浮点数，它的范围和精度可能超出了一个8位通道所能提供的。直接将这样的深度值存储在一个8位通道中会导致大量的精度丢失，从而导致阴影效果不正确。因此我们可以充分利用其他的三个通道，也就是将深度值编码到多个通道中。通过分配深度值的不同部分到R, G, B, A四个通道，我们可以用更高的精度来存储深度值。当我们需要使用深度值时，就可以从这四个通道解码出来。

`closestDepthVec`是blocker的深度信息，

最后，`closestDepth`与`currentDepth`进行比对，如果blocker(`closestDepth`)比主相机要渲染的片元的深度值(`shadingPoint.z`)大，说明当前的Shading Point没有被遮挡，`visibility`返回`1.0`。另外为了解决一些阴影痤疮和自遮挡问题，可以将blocker的位置调大一些，即加上`EPS`。

```glsl
// phongFragment.glsl
float useShadowMap(sampler2D shadowMap, vec4 shadingPoint){
  // Retrieve the closest depth value from the light's perspective using the fragment's position in light space.
  float closestDepth = unpack(texture2D(shadowMap, shadingPoint.xy));
  // Compare the fragment's depth with the closest depth to determine if it's in shadow.
  return (closestDepth + EPS + getBias(.4)> shadingPoint.z) ? 1.0 : 0.0;
}
```

其实目前还是有点问题。我们目前的光源相机并不是万向的，也就是说其照射范围只有一小部分。如果模型在lightCam的范围内，那么画面是完全正确的。

但是当模型在lightCam的范围外，就不应该参与`useShadowMap`的计算。但是目前我们并没有完成相关的逻辑。也就是说，如果在lightCam的MVP变换矩阵范围之外的位置在经过计算之后可能会出现意想不到的错误。再看一下灵魂示意图：

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20230815223607948.png" alt="image-20230815223607948" style="zoom:50%;" />

上一节我们在定向光源脚本中定义了zFar、zNear等信息。如下代码所示：

```js
// DirectionalLight.js - CalcLightMVP()
let left = -100.0, right = -left, bottom = -100.0, top = -bottom, near = 0.1, far = 1024.0;
```

因此，为了解决模型在lightCam范围之外的问题，我们在`useShadowMap`或在`useShadowMap`之前的代码中，加入以下逻辑以剔除不在lightCam范围的采样点：

```js
// phongFragment.glsl - main()
...
if(shadingPoint.x<0.||shadingPoint.x>1.||
   shadingPoint.y<0.||shadingPoint.y>1.){
  visibility=1.;// 光源看不见的地方，因此不会被阴影所覆盖
}else{
  visibility=useShadowMap(uShadowMap,vec4(shadingPoint,1.));
}
...
```

效果如下图所示，左边是做了剔除逻辑的，右边是没有做剔除逻辑的。当202酱移动到lightCam的视锥体边界时，她就直接被截肢了，非常吓人：

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20230815214856433.png" alt="image-20230815214856433" style="zoom:50%;" />

> 当然了，不完成这一步也没问题。实际上，在开发中我们会使用万向光源，即lightCam是360度全方位的，我们只需要剔除那些在zFar平面之外的点就可以了。

### 添加bias改善自遮挡问题

当我们从光源的视角渲染深度图时，由于浮点数精度的限制，可能会出现误差。因此，当我们在主渲染过程中使用深度图时，可能会看到物体自己的阴影，这称为自遮挡或阴影失真。

在完成了2-pass渲染之后，我们会在202酱的头发等多处位置发现了这样的阴影痤疮，十分不美观。如下图所示：

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20230805133417146.png" alt="image-20230805133417146" style="zoom:50%;" />

我们理论上可以通过添加bias缓解自遮挡问题。这里我提供一种动态调整bias的方法：

```glsl
// phongFragment.glsl
// 使用bias偏移值优化自遮挡
float getBias(float ctrl) {
  vec3 lightDir = normalize(uLightPos);
  vec3 normal = normalize(vNormal);
  float m = 200.0 / 2048.0 / 2.0; // 正交矩阵宽高/shadowmap分辨率/2
  float bias = max(m, m * (1.0 - dot(normal, lightDir))) * ctrl;
  return bias;
}
```

首先当光线和法线几乎垂直的时候，极有可能发生自遮挡现象，比如我们的202酱的后脑勺处。因此我们需要获取光线的方向与法线的方向。其中，m表示光源视图下每个像素代表的场景空间的大小。

最后将 phongFragment.glsl 的 `useShadowMap()` 改为下文：

```glsl
// phongFragment.glsl
float useShadowMap(sampler2D shadowMap, vec4 shadingPoint){
  ...
  return (closestDepth + EPS + getBias(.3)> shadingPoint.z) ? 1.0 : 0.0;
}
```

效果如下：

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20230805141729568.png" alt="image-20230805141729568" style="zoom:50%;" />

需要注意，较大的bias值可能导致过度矫正带来的阴影缺失结果，较小的值又可能起不到改善痤疮的效果，因此需要多次尝试。

## PCF

但是ShadowMap的分辨率是有限的。实际游戏中，ShadowMap的分辨率是远远小于分辨率的（原因是性能消耗太大），因此我们需要一种柔化锯齿的方法。PCF方法就是在ShadowMap上为每个像素取其周边的多个像素做平均计算出Shading Point的。

最初人们想用这个方法软化阴影，但是做到后面发现这个方法可以做到软阴影的效果。

在使用PCF算法估计阴影比例之前，我们需要准备一组采样点。对于PCF阴影，在移动设备我们只会采用4-8个采样点，而高质量的画面则来到16-32个。在这一节我们使用8个采样点，在这个基础上通过调整生成的样本的参数从而改进画面，减少噪点等等。

但是，以上不同采样方式对于最终的画面影响其实不算特别大，最影响画面的其实是做PCF时候的阴影贴图大小，也就是shadow map的大小。具体来说，是代码中的`textureSize`，但是一般而言这一项在项目中都是固定一个值。

所以我们接下来的思路是先实现PCF，最后再微调采样方式。

**毕竟，premature optimization是大忌。**

### 实现PCF

在`main()`中，修改使用的阴影算法。

```glsl
// phongFragment.glsl
void main(void){  
    ...
    visibility = PCF(uShadowMap, vec4(shadingPoint, 1.0));
    ...
}
```

`shadowMap.xy` 是用于在阴影贴图上采样的纹理坐标，`shadowMap.z` 是该像素的深度值。

采样函数要求我们传入一个Vec2变量作为随机种子，接着会在一个半径为1的圆域内返回随机的点。

接着将$[0, 1]^2$的uv坐标中分成`textureSize`份，设置好滤波窗口之后，就在当前的`shadingPoint`位置附近采样多次，最后统计：

```glsl
// phongFragment.glsl
float PCF(sampler2D shadowMap,vec4 shadingPoint){
  // 采样 采样结果会返回到全局变量 - poissonDisk[]
  poissonDiskSamples(shadingPoint.xy);

  float textureSize=256.; // shadow map 的大小, 越大滤波的范围越小
  float filterStride=1.; // 滤波的步长
  float filterRange=1./textureSize*filterStride; // 滤波窗口的范围
  int noShadowCount=0; // 有多少点不在阴影里
  for(int i=0;i<NUM_SAMPLES;i++){
    vec2 sampleCoord=poissonDisk[i]*filterRange+shadingPoint.xy;
    vec4 closestDepthVec=texture2D(shadowMap,sampleCoord);
    float closestDepth=unpack(closestDepthVec);
    float currentDepth=shadingPoint.z;
    if(currentDepth<closestDepth+EPS){
      noShadowCount+=1;
    }
  }
  return float(noShadowCount)/float(NUM_SAMPLES);
}
```

效果如下：

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20230805213129275.png" alt="image-20230805213129275" style="zoom: 33%;" />

### poissonDisk采样参数设置

在作业框架中，我发现这个`possionDiskSamples`函数并不是真正的泊松圆盘分布？有点奇怪。**个人感觉更像是均匀分布在螺旋线上的点**。希望读者朋友可以指导一下。我首先先按照框架中的代码分析。

----

**框架中poissonDiskSamples的相关数学公式**：

```glsl
// phongFragment.glsl
float ANGLE_STEP = PI2 * float( NUM_RINGS ) / float( NUM_SAMPLES );
float INV_NUM_SAMPLES = 1.0 / float( NUM_SAMPLES );
float angle = rand_2to1( randomSeed ) * PI2;
float radius = INV_NUM_SAMPLES;
float radiusStep = radius;
```

转换极坐标为笛卡尔坐标:
$$
\left\{\begin{array}{}
x = rcos⁡(θ) \\
y = rsin⁡(θ)
\end{array}\right.
$$
更新规则：
$$
\left\{\begin{array}{}
r_\text{next}=r_\text{current}+radiusStep \\
θ_\text{next}=θ_\text{current}+ \text{ANGLE\_STEP}
\end{array}\right.
$$
半径变化：

$$
r\prime = r^{0.75}
$$
具体代码如下：

```glsl
// phongFragment.glsl
vec2 poissonDisk[NUM_SAMPLES];

void poissonDiskSamples( const in vec2 randomSeed ) {
  float ANGLE_STEP = PI2 * float( NUM_RINGS ) / float( NUM_SAMPLES );
  float INV_NUM_SAMPLES = 1.0 / float( NUM_SAMPLES );// 把样本放在了一个半径为1的圆域内

  float angle = rand_2to1( randomSeed ) * PI2;
  float radius = INV_NUM_SAMPLES;
  float radiusStep = radius;

  for( int i = 0; i < NUM_SAMPLES; i ++ ) {
    poissonDisk[i] = vec2( cos( angle ), sin( angle ) ) * pow( radius, 0.75 );
    radius += radiusStep;
    angle += ANGLE_STEP;
  }
}
```

也就是说，以下参数我们可以调整：

- **半径变化指数的选取**

关于作业框架中为什么要用0.75这个数字，我做了一个比较形象的动画，展示了在泊松采样时每个结果坐标与圆心的距离（半径）的指数在0.2到1.1之间的变化，也就是说，当数值取到0.75以上时，基本可以认为数据重心会更偏向于取圆心的位置。下面动画的代码我放在了 **附录1.2** 中，读者可以自行编译调试。

<video width="50%" height="auto" poster="" controls="controls">
<source src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/Poisson%20Disk%20Samples.mp4" /></video>

> 上面是一则视频，若您是PDF版本则需要前往网站查看。


- **绕圈数`NUM_RINGS`**

`NUM_RINGS`与`NUM_SAMPLES`一起用来计算每个采样点之间的角度差`ANGLE_STEP`。

此时可以有如下分析：

如果`NUM_RINGS`等于`NUM_SAMPLES`，那么`ANGLE_STEP`将等于$2π$，这意味着每次迭代中的角度增量都是一个完整的圆，这显然没有意义。如果`NUM_RINGS`小于`NUM_SAMPLES`，那么`ANGLE_STEP`将小于$2π$，这意味着每次迭代中的角度增量都是一个圆的部分。如果`NUM_RINGS`大于`NUM_SAMPLES`，那么`ANGLE_STEP`将大于$2π$，这意味着每次迭代中的角度增量都超过了一个圆，这可能会导致覆盖和重叠。

所以在这个代码框架中，当我们的采样数固定时（我这里是8），我们就可以采取决策让采样点更加均匀的分布。

因此理论上，这里`NUM_RINGS`直接设置为1就可以了。

<video width="50%" height="auto" poster="" controls="controls">
<source src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/remoooo_%202023-08-08%2022.28.59.mp4"></video>

> 上面是一则视频，若您是PDF版本则需要前往网站查看。

当采样点分布均匀的情况下，效果还不错：

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20230813194049669.png" alt="image-20230813194049669" style="zoom:50%;" />

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20230813194346013.png" alt="image-20230813194346013" style="zoom:67%;" />

如果采样非常不均匀，比如`NUM_RINGS`等于`NUM_SAMPLES`的情况，就会出现比较脏的画面：

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20230813194234867.png" alt="image-20230813194234867" style="zoom: 67%;" />

得到这些采样点之后，我们还可以对采样点进行权重分配处理。比如在202的课程上闫老师提到可以根据原始像素的距离设置不同的权重，更远的采样点可能会被赋予较低的权重，但项目中不涉及这部分的代码。

## PCSS

首先找到Shadow Map中任意一处uv坐标的AVG Blocker Depth。

```glsl
float findBlocker(sampler2D shadowMap,vec2 uv,float z_shadingPoint){
  float count=0., depth_sum=0., depthOnShadowMap, is_block;
  vec2 nCoords;
  for(int i=0;i<BLOCKER_SEARCH_NUM_SAMPLES;i++){
    nCoords=uv+BLOKER_SIZE*poissonDisk[i];
    
    depthOnShadowMap=unpack(texture2D(shadowMap,nCoords));
    if(abs(depthOnShadowMap) < EPS)depthOnShadowMap=1.;
    // step函数用于比较两个值。
    is_block=step(depthOnShadowMap,z_shadingPoint-EPS);
    count+=is_block;
    depth_sum+=is_block*depthOnShadowMap;
  }
  if(count<EPS)
    return z_shadingPoint;
  return depth_sum/count;
}
```

三步走，这里不再赘述，跟着理论公式走都不太难。
$$
w_{\text {Penumbra }}=\frac{\left(d_{\text {Receiver }}-d_{\text {Blocker }}\right) \cdot w_{\text {Light }}} { d_{\text {Blocker }}}
$$
<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20230731142003749.png" alt="image-20230731142003749" style="zoom:50%;" />

```glsl
float PCSS(sampler2D shadowMap,vec4 shadingPoint){
  poissonDiskSamples(shadingPoint.xy);
  float z_shadingPoint=shadingPoint.z;
  // STEP 1: avgblocker depth
  float avgblockerdep=findBlocker(shadowMap,shadingPoint.xy,z_shadingPoint);
  if(abs(avgblockerdep - z_shadingPoint) <= EPS) // No Blocker
    return 1.;
  
  // STEP 2: penumbra size
  float dBlocker=avgblockerdep,dReceiver=z_shadingPoint-avgblockerdep;
  float wPenumbra=min(LWIDTH*dReceiver/dBlocker,MAX_PENUMBRA);
  
  // STEP 3: filtering
  float _sum=0.,depthOnShadowMap,vis;
  vec2 nCoords;
  for(int i=0;i<NUM_SAMPLES;i++){
    nCoords=shadingPoint.xy+wPenumbra*poissonDisk[i];
    
    depthOnShadowMap=unpack(texture2D(shadowMap,nCoords));
    if(abs(depthOnShadowMap)<1e-5)depthOnShadowMap=1.;
    
    vis=step(z_shadingPoint-EPS,depthOnShadowMap);
    _sum+=vis;
  }
  
  return _sum/float(NUM_SAMPLES);
}
```

## 框架部分解析

这一部分属于是在我随便翻阅代码的时候写下的注释，在这里稍微整理了一下。

### loadShader.js

虽然这个文件中两个函数都是加载glsl文件，但是后者的`getShaderString(filename)`函数更加简洁高级。这主要体现在前者返回的是Promise对象，后者直接返回文件内容。关于Promise的内容可以看本文的 **附录1.3 - JS的Promise简单用法** ，关于async await的内容可以看本文 **附录1.4 - async await介绍**，关于.then()的用法可以查看 **附录1.5 - 关于.then** 。

专业一点的说法就是，这两个函数提供了不同级别的抽象。前者提供了直接加载文件的原子级别能力，拥有更细粒度的控制，而后者更加简洁与方便。

## 添加物体平移效果

### 控制器添加到GUI上

每一帧都要计算阴影的消耗是很大的，这里我手动创建光源控制器，手动调节是否需要每一帧都计算一次阴影。此外，当Light Moveable取消勾选的时候禁止用户改变光源位置：

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20230806140824341.png" alt="image-20230806140824341" style="zoom:50%;" />

勾选上Light Moveable后，出现lightPos选项框：

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20230806141007364.png" alt="image-20230806141007364" style="zoom:50%;" />

具体代码实现：

```js
// engine.js
// Add lights
// light - is open shadow map == true
let lightPos = [0, 80, 80];
let focalPoint = [0, 0, 0]; // 定向光聚焦方向(起点是lightPos)
let lightUp = [0, 1, 0]
const lightGUI = {// 光源移动控制器，如果不勾选，则不会重新计算阴影。
    LightMoveable: false,
    lightPos: lightPos
};
...
function createGUI() {
    const gui = new dat.gui.GUI();
    const panelModel = gui.addFolder('Light properties');
    const panelCamera = gui.addFolder("OBJ properties");
    const lightMoveableController = panelModel.add(lightGUI, 'LightMoveable').name("Light Moveable");
    const arrayFolder = panelModel.addFolder('lightPos');
    arrayFolder.add(lightGUI.lightPos, '0').min(-10).max( 10).step(1).name("light Pos X");
    arrayFolder.add(lightGUI.lightPos, '1').min( 70).max( 90).step(1).name("light Pos Y");
    arrayFolder.add(lightGUI.lightPos, '2').min( 70).max( 90).step(1).name("light Pos Z");
    arrayFolder.domElement.style.display = lightGUI.LightMoveable ? '' : 'none';
    lightMoveableController.onChange(function(value) {
        arrayFolder.domElement.style.display = value ? '' : 'none';
    });
}
```

## 附录1.1

```python
import numpy as np
import matplotlib.pyplot as plt

def simulate_poisson_disk_samples(random_seed, num_samples=100, num_rings=2):
    PI2 = 2 * np.pi
    ANGLE_STEP = PI2 * num_rings / num_samples
    INV_NUM_SAMPLES = 1.0 / num_samples

    # Initial angle and radius
    angle = random_seed * PI2
    radius = INV_NUM_SAMPLES
    radius_step = radius

    x_vals = []
    y_vals = []

    for _ in range(num_samples):
        x = np.cos(angle) * pow(radius, 0.1)
        y = np.sin(angle) * pow(radius, 0.1)

        x_vals.append(x)
        y_vals.append(y)

        radius += radius_step
        angle += ANGLE_STEP

    return x_vals, y_vals

plt.figure(figsize=(8, 8))

# Generate and plot the spiral 5 times with different random seeds
for _ in range(50):
    random_seed = np.random.rand()
    x_vals, y_vals = simulate_poisson_disk_samples(random_seed)
    plt.plot(x_vals, y_vals, '-o', markersize=5, linewidth=2)

plt.title("Poisson Disk Samples")
plt.axis('on')
plt.gca().set_aspect('equal', adjustable='box')
plt.show()

```

## 附录1.2 - 泊松采样点后处理动画代码

说明：**附录1.2** 的代码直接基于 **附录1.1** 修改而成。

```python
import numpy as np
import matplotlib.pyplot as plt
from matplotlib.animation import FuncAnimation

def simulate_poisson_disk_samples_with_exponent(random_seed, exponent, num_samples=100, num_rings=2):
    PI2 = 2 * np.pi
    ANGLE_STEP = PI2 * num_rings / num_samples
    INV_NUM_SAMPLES = 1.0 / num_samples

    angle = random_seed * PI2
    radius = INV_NUM_SAMPLES
    radius_step = radius

    x_vals = []
    y_vals = []

    for _ in range(num_samples):
        x = np.cos(angle) * pow(radius, exponent)
        y = np.sin(angle) * pow(radius, exponent)
        x_vals.append(x)
        y_vals.append(y)
        radius += radius_step
        angle += ANGLE_STEP

    return x_vals, y_vals

fig, ax = plt.subplots(figsize=(8, 8))
ax.axis('on')
ax.set_xlim(-1, 1)
ax.set_ylim(-1, 1)
ax.set_aspect('equal', adjustable='box')

lines = [ax.plot([], [], '-o', markersize=5, linewidth=2)[0] for _ in range(50)]
exponent = 0.2

def init():
    for line in lines:
        line.set_data([], [])
    return lines

def update(frame):
    global exponent
    exponent += 0.005  # Increment to adjust the exponent
    for line in lines:
        random_seed = np.random.rand()
        x_vals, y_vals = simulate_poisson_disk_samples_with_exponent(random_seed, exponent)
        # plt.title(exponent +"Poisson Disk Samples")
        line.set_data(x_vals, y_vals)
    plt.title(f"{exponent:.3f} Poisson Disk Samples")
    return lines

ani = FuncAnimation(fig, update, frames=180, init_func=init, blit=False)

ani.save('animation.mp4', writer='ffmpeg', fps=12)

# plt.show()
```

## 附录1.3 - JS的Promise简单用法

关于Promise的用法这里给出一个例子：

```js
function delay(milliseconds) {
    return new Promise(function(resolve, reject) {
        if (milliseconds < 0) {
            reject('Delay time cannot be negative!');
        } else {
            setTimeout(function() {
                resolve('Waited for ' + milliseconds + ' milliseconds!');
            }, milliseconds);
        }
    });
}

// 使用示例
delay(2000).then(function(message) {
    console.log(message);  // 两秒后输出："Waited for 2000 milliseconds!"
}).catch(function(error) {
    console.log('Error: ' + error);
});

// 错误示例
delay(-1000).then(function(message) {
    console.log(message);
}).catch(function(error) {
    console.log('Error: ' + error);  // 立即输出："Error: Delay time cannot be negative!"
});

```

使用Promise的固定操作是写一个`Promise`构造函数，这个函数有两个参数（参数也是一个函数）：`resolve`和`reject`。这样可以构建错误处理的分支，比如在这个案例中，输入的内容不满足需求，则可以调用`reject`进入拒绝Promise分支。

比方说现在进入reject分支，reject(**XXX**)中的XXX就传到了下面then(function(**XXX**))的XXX中。

总结一下，Promise是JS中的一个对象，核心价值在于它提供了一种**非常优雅**与**统一**的方式处理异步操作与链式操作，另外还提供了错误处理的功能。

1. 通过`Promise`的`.then()`方法，你可以确保一个异步操作完成后再执行另一个异步操作。
2. 通过 `.catch()` 方法可以处理错误，不需要为每个异步回调设置错误处理。

## 附录1.4 - async/await

async/await是ES8引入的feature，旨在化简使用Promise的步骤。

直接看例子：

```js
async function asyncFunction() {
    return "Hello from async function!";
}

asyncFunction().then(result => console.log(result));  // 输出：Hello from async function!
```

函数加上了async之后，会隐式的返回一个Promise对象。

`await` 关键字只能在 `async` 函数内部使用。它会“暂停”函数的执行，直到 Promise 完成（解决或拒绝）。另外你也可以用try/catch捕获reject。

```js
async function handleAsyncOperation() {
    try {
        const result = await maybeFails();// 
        console.log(result);// 如果 Promise 被解决，这里将输出 "Success!"
    } catch (error) {
        console.error('An error occurred:', error);// 如果 Promise 被拒绝，这里将输出 "An error occurred: Failure!"
    }
}
```

这里的"暂停"指的是暂停了该**特定的异步函数**，而不是整个应用或JavaScript的事件循环。

以下是关于`await`如何工作的简化说明：

1. 当执行到`await`关键字时，**该异步函数**的执行暂停。
2. 控制权返回给事件循环，允许其他代码（如其他的函数、事件回调等）在当前异步函数之后立即运行。
3. 一旦`await`后面的Promise解决（fulfilled）或拒绝（rejected），原先暂停的异步函数继续执行，从暂停的位置恢复，并处理Promise的结果。

也就是说，虽然你的特定的`async`函数在逻辑上"暂停"了，JavaScript的主线程并没有被阻塞。其他的事件和函数仍然可以在后台执行。

举一个例子：

```js
console.log('Start');

async function demo() {
    console.log('Before await');
    await new Promise(resolve => setTimeout(resolve, 2000));
    console.log('After await');
}

demo();

console.log('End');
```

输出将是：

>Start
>Before await
>End
>(wait for 2 seconds)
>After await

希望以上解释可以帮助你理解JS的异步机制。欢迎在评论区讨论，我会尽可能立即回复您。

## 附录1.5 关于.then()

`.then()` 是在 Promise 对象上定义的，用于处理 Promise 的结果。当你调用 `.then()`，它不会立即执行，而是在 Promise 解决 (fulfilled) 或拒绝 (rejected) 后执行。

.then() 的关键点：

1. **非阻塞**：当你调用 `.then()` 时，代码不会暂停等待 Promise 完成。相反，它会立即返回，并在 Promise 完成时执行 `then` 里的回调。
2. **返回新的 Promise**：`.then()` 总是返回一个新的 Promise。这允许你进行链式调用，即一系列的 `.then()` 调用，每个调用处理前一个 Promise 的结果。
3. **异步回调**：当原始 Promise 解决或拒绝时，`.then()` 里的回调函数是异步执行的。这意味着它们在事件循环的微任务队列中排队，而不是立即执行。

举个例子：

```js
console.log('Start');

const promise = new Promise((resolve, reject) => {
    setTimeout(() => {
        resolve('Promise resolved');
    }, 2000);
});

promise.then(result => {
    console.log(result);
});

console.log('End');
```

输出会是：

>Start
>End
>(wait for 2 seconds)
>Promise resolved

## 附录1.6 - 片段着色器:Uniforms/Textures

>https://webglfundamentals.org/webgl/lessons/zh_cn/webgl-fundamentals.html

### Uniforms 全局变量

全局变量在一次绘制过程中传递给着色器的值都一样，在下面的一个简单的例子中， 用全局变量给顶点着色器添加了一个偏移量：

```glsl
attribute vec4 a_position;uniform vec4 u_offset; void main() {   gl_Position = a_position + u_offset;}
```

现在可以把所有顶点偏移一个固定值，首先在初始化时找到全局变量的地址

```glsl
var offsetLoc = gl.getUniformLocation(someProgram, "u_offset");
```

然后在绘制前设置全局变量

```glsl
gl.uniform4fv(offsetLoc, [1, 0, 0, 0]);  // 向右偏移一半屏幕宽度
```

要注意的是全局变量属于单个着色程序，如果多个着色程序有同名全局变量，需要找到每个全局变量并设置自己的值。 

### Textures 纹理

在着色器中获取纹理信息，可以先创建一个`sampler2D`类型全局变量，然后用GLSL方法`texture2D` 从纹理中提取信息。

```glsl
precision mediump float; 
uniform sampler2D u_texture; 
void main() {   
    vec2 texcoord = vec2(0.5, 0.5);  // 获取纹理中心的值   
    gl_FragColor = texture2D(u_texture, texcoord);
}
```

从纹理中获取的数据[取决于很多设置](https://webglfundamentals.org/webgl/lessons/zh_cn/webgl-3d-textures.html)。 至少要创建并给纹理填充数据，例如

```glsl
var tex = gl.createTexture();
gl.bindTexture(gl.TEXTURE_2D, tex);
var level = 0;
var width = 2;
var height = 1;
var data = new Uint8Array([
   255, 0, 0, 255,   // 一个红色的像素
   0, 255, 0, 255,   // 一个绿色的像素
]);
gl.texImage2D(gl.TEXTURE_2D, level, gl.RGBA, width, height, 0, gl.RGBA, gl.UNSIGNED_BYTE, data);
gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MIN_FILTER, gl.LINEAR);
```

在初始化时找到全局变量的地址

```glsl
var someSamplerLoc = gl.getUniformLocation(someProgram, "u_texture");
```

在渲染的时候WebGL要求纹理必须绑定到一个纹理单元上

```glsl
var unit = 5;  // 挑选一个纹理单元
gl.activeTexture(gl.TEXTURE0 + unit);
gl.bindTexture(gl.TEXTURE_2D, tex);
```

然后告诉着色器你要使用的纹理在那个纹理单元

```glsl
gl.uniform1i(someSamplerLoc, unit);
```

## Reference

1. GAMES202
2. Real-Time Rendering 4th Edition
3. https://webglfundamentals.org/webgl/lessons/webgl-shaders-and-glsl.html

---
date: 2026-11-24
tags:
  - threejs
  - web-client
  - react
---
最近有一个theejs的项目需要编写，即便在ai的时代下，没有对threejs有一个初步的认识和对渲染管线的把控，是依旧无法胜任真正的web3d开发的，因此我编写此文档用于记录threejs相关知识点。

正如标题，threejs和React Three Fiber 简称R3F 和 Threejs的关系会有重点的讲解，后者会把threejs的场景图、对象生命周期和渲染循环映射成react组件模型，只是真正执行几何、gpu缓冲区管理、实例化绘制和webgl渲染的是threejs，不过这个已经是后话了。

我们真正依赖的是相关于threejs本体和它的几个相关npm包，分别是：

```json
"@react-three/drei": "^10.7.9",
"@react-three/fiber": "^9.8.2",
"three": "^0.187.0"
```

three是我们的核心对象，react three fiber提供了jsx的映射，drei提供了高级封装组件，我们真正在编写threejs组件的时候，会直接导入进入：

```ts
import {
  GizmoHelper,
  GizmoViewport,
  OrbitControls
} from '@react-three/drei'

import { Canvas } from '@react-three/fiber'
```

它们的来源和职责如下：

|组件|来源|实际作用|
|---|---|---|
|`Canvas`|`@react-three/fiber`|创建渲染器、Scene、Camera，管理渲染循环|
|`OrbitControls`|`@react-three/drei`|封装 Three.js 的 OrbitControls|
|`GizmoHelper`|`@react-three/drei`|管理方向 Gizmo 的位置和更新|
|`GizmoViewport`|`@react-three/drei`|绘制 XYZ 方向指示器|
|`instancedMesh`|R3F 内置 Three.js 对象映射|对应 `THREE.InstancedMesh`|
|`boxGeometry`|R3F 内置 Three.js 对象映射|对应 `THREE.BoxGeometry`|
|`meshStandardMaterial`|R3F 内置 Three.js 对象映射|对应 `THREE.MeshStandardMaterial`|
|`gridHelper`|R3F 内置 Three.js 对象映射|对应 `THREE.GridHelper`|
|`axesHelper`|R3F 内置 Three.js 对象映射|对应 `THREE.AxesHelper`|
因此drei不是渲染引擎，而是建立在r3f和threejs之上的工具库。我们使用轨道控件，可以直接编写jsx组件：

```tsx
<OrbitControls
  makeDefault
  target={DEFAULT_CAMERA_TARGET}
  enableDamping
  dampingFactor={0.08}
/>
```

本质上是 Drei 对 Three.js `OrbitControls` 的 React 封装，其底层仍然使用：

```ts
import { OrbitControls } from 'three/addons/controls/OrbitControls.js'
```

Drei 帮你处理了控制器的创建、销毁、更新，以及与 R3F Canvas 的集成，如果你自己直接使用threejs版本的OrbitControls控件，就需要自己处理控制器实例创建、绑定DOM事件、更新控制器、清理事件、渲染循环与同步、组件卸载释放资源等等任务，因此我们使用drei的本质就是为了封装生命周期直接使用。

因此GizmoHelper与GizmoViewer也是类似的思路，这两个组件不是threejs核心api，而是drei提供的指示器组件，负责gizmo的屏幕位置、与主相机方向同步、坐标轴和标签绘制，可以把它看成一个独立的相机方向辅助 UI。

所以说个人认为，最重要的其实是R3F库，把大部分声明式对象管理交给了React，也依旧允许你直接操作threejs来精细控制。

## threejs核心api

我们不准备按照threejs api文档的字母顺序来讲，而是按照一个3d应用实际上运行时候的结构来讲解，我们要达成的目标是看到threejs的代码片段可以知道在创建什么对象，修改什么数据、影响渲染管线的哪个阶段，以及需要什么时候释放资源。

![](https://pub-dd8e151a60af44cfbe36beaaf0ad54e8.r2.dev/2026/11/1795500832970-jbnmvkk1.webp)


three.scene管理相机、光照、对象、网格。scene是一个特殊的Object3D，通常是整个场景树的根节点，可以把scene理解为一个容器，但是不只是一个普通容器，包含了场景背景、雾效设置、场景中的对象树、环境相关设置。

```ts
import * as THREE from 'three'
const scene = new THREE.Scene()
scene.background = new THREE.Color('#202530')
```

Scene 并不直接存储所有顶点数据。它主要管理场景图中的对象引用和层级关系，细致内容可以去看scene.js的源码部分。

Object3D是threejs的场景图的基础，这个是非常重要的类，`Mesh`、`Group`、`Camera`、`Light` 等对象都继承自 `Object3D`，它提供了通用的空间变换和层级管理能力：


```ts
const object = new THREE.Object3D()

object.position.set(1, 2, 3)
object.rotation.set(0, 0, Math.PI / 2)
object.scale.set(1, 1, 1)

scene.add(object)
```

它最重要的属性是：

|属性|类型|含义|
|---|---|---|
|`position`|Vector3|局部位置|
|`rotation`|Euler|局部欧拉角|
|`quaternion`|Quaternion|局部旋转|
|`scale`|Vector3|局部缩放|
|`matrix`|Matrix4|局部变换矩阵|
|`matrixWorld`|Matrix4|世界变换矩阵|
|`parent`|Object3D|父对象|
|`children`|Object3D[]|子对象|
|`visible`|boolean|是否可见|

这里理解一下局部矩阵与世界矩阵的关系，局部矩阵相对于父对象的位置、旋转和缩放。世界矩阵是描述对象相对于整个世界坐标系的变换，这里用cv/cg的知识就可以充分理解，一个4X4的齐次变换矩阵包含了T R S：

矩阵可以表示为：

$$
M= \begin{bmatrix} m_{11}&m_{12}&m_{13}&t_x\\ m_{21}&m_{22}&m_{23}&t_y\\ m_{31}&m_{32}&m_{33}&t_z\\ 0&0&0&1 \end{bmatrix} 
$$

左上角的3x3区域就是R S 部分，tx ty tz 是T部分，简单易懂。


接下来讲 Three.js 中非常核心的一个对象：`Mesh` 网格对象，如果我们有opengl或者vulkan的基础，听这个非常简单，所谓mesh网格，我们知道最小绘制单位就是三角形，这个是实践和经验告诉我们的，也是计算机图形学的基础。通过mesh，可以实现碰撞体、可视化几何体。


最基本的公式：

$$
 \boxed{Mesh = Geometry + Material} 
$$

一个mesh网格对象包含了几何体和材质，几何体决定物体形状、比如说立方体、球体、连杆模型。材质决定物体的显示比如说颜色、金属感、透明度、是否收到光照影响等等。mesh将这两个组合起来，成为可以加入到scene的三维对象。

```ts
const geometry = new THREE.BoxGeometry(1, 1, 1)

const material = new THREE.MeshStandardMaterial({
  color: 0x00ff00,
  roughness: 0.7,
  metalness: 0.1
})

const mesh = new THREE.Mesh(geometry, material)

scene.add(mesh)
```

![](https://pub-dd8e151a60af44cfbe36beaaf0ad54e8.r2.dev/2026/11/1795501911885-npcfu7jw.webp)

`BoxGeometry` 立方体的顶点、三角形和法线等数据，`MeshStandardMaterial` 定义了颜色、粗糙程度、金属度等渲染属性。`Mesh` 组合了前面两者，提供了位置、旋转、缩放等空间变换。

`Mesh` 本身继承自 `Object3D`，所以它也有：


```ts
mesh.position
mesh.rotation
mesh.quaternion
mesh.scale
mesh.matrix
mesh.matrixWorld
mesh.visible
mesh.parent
mesh.children
```

也就是说，前面学习的 `Object3D` 变换体系，可以直接用于 Mesh。

几何体通常由顶点数据构成，不是简单地保存一个立方体、球体这样的抽象形状，以 `BoxGeometry` 为例：

```ts
const geometry = new THREE.BoxGeometry(1, 1, 1)

console.log(geometry.attributes.position)
console.log(geometry.attributes.normal)
console.log(geometry.attributes.uv)
console.log(geometry.index)
```

主要有四类数据：

|属性|作用|
|---|---|
|`position`|顶点坐标 XYZ|
|`normal`|顶点法线，决定光照计算|
|`uv`|纹理坐标，决定纹理如何映射|
|`index`|顶点索引，决定如何组成三角形|
例如一个三角形：

![](https://pub-dd8e151a60af44cfbe36beaaf0ad54e8.r2.dev/2026/11/1795502120133-r9ea35yp.webp)

假设三个顶点分别是：$v_0=(0,0,0)$  $v_1=(1,0,0)$  $v_2=(0,1,0)$ 

那么 position 数组就是：

```ts
const positions = new Float32Array([
  0, 0, 0,
  1, 0, 0,
  0, 1, 0
])
```

使用 BufferGeometry 创建：

```ts
const geometry = new THREE.BufferGeometry()

geometry.setAttribute(
  'position',
  new THREE.BufferAttribute(positions, 3)
)
```

这里的 `3` 表示每三个数组元素构成一个顶点。再设置索引：

```ts
geometry.setIndex([0, 1, 2])
```

就表示使用第 0、1、2 号顶点组成一个三角形。


材质决定了物体的外观信息，threejs有很多材质类型：

![](https://pub-dd8e151a60af44cfbe36beaaf0ad54e8.r2.dev/2026/11/1795502343897-rbn5k2lg.webp)

几个重要参数：

- `color`：基础颜色。
    
- `roughness`：粗糙度，越大通常越不容易出现清晰的反射。
    
- `metalness`：金属度。
    
- `transparent`：是否启用透明混合。
    
- `opacity`：透明度。
    
- `side`：正面、背面还是双面渲染。

例如，创建一个半透明碰撞区域：

```ts
const material = new THREE.MeshBasicMaterial({
  color: 0xff0000,
  transparent: true,
  opacity: 0.25,
  depthWrite: false
})
```

这里的 `depthWrite: false` 可以减少透明物体写入深度缓冲区造成的一些遮挡问题，但不代表它能彻底解决透明物体排序问题。

Mesh 除了继承 `Object3D` 的空间变换属性，还拥有几个自己的核心属性。


```ts
mesh.geometry
mesh.material
mesh.count // 注意：普通 Mesh 没有 count 这个实例数量属性
mesh.castShadow
mesh.receiveShadow
mesh.frustumCulled
```

其中：

|属性|作用|
|---|---|
|`geometry`|当前 Mesh 使用的几何体|
|`material`|当前 Mesh 使用的材质|
|`castShadow`|是否投射阴影|
|`receiveShadow`|是否接收阴影|
|`frustumCulled`|是否参与视锥裁剪|

一个重要知识点，如果100个mesh共享了一个几何体和材质的话，可以避免创建大量相同gpu几何数据和材质资源：

```ts
const geometry = new THREE.BoxGeometry(1, 1, 1)

const material = new THREE.MeshStandardMaterial({
  color: 0x00ff00
})

for (let i = 0; i < 100; i++) {
  const mesh = new THREE.Mesh(geometry, material)

  mesh.position.x = i * 1.5

  scene.add(mesh)
}
```

这里有一个需要特别注意的地方： scene.remove(mesh) 只是从场景数里移除Mesh，但是不会自动释放共享的Geometry 和 Material。


```ts
geometry.dispose()
material.dispose()
```

会释放相关 GPU 资源。因此，只有确认没有其他 Mesh 再使用这些资源时，才能释放它们。

然后再看看gpu buffers / shader programs / draw calls部分，这个部分是理解threejs渲染性能的关键，前面已经知道了：
- `Geometry` 保存顶点数据。
- `Material` 定义物体的表面表现。
- `Mesh` 将 Geometry 和 Material 组合起来。
- `Scene` 组织所有三维对象。
- `WebGLRenderer` 负责把场景真正绘制到屏幕上。

gpu buffers会把几何数据上传到gpu，shader programs会告诉gpu如何处理顶点，计算颜色。draw calls告诉gpu执行一次绘制，可以先把执行顺序记住，这个事简化之后的版本：

```
Geometry
   │
   ▼
GPU Buffers
   │
   ▼
Shader Program
   │
   ▼
Draw Call
   │
   ▼
屏幕像素
```

事实上，threejs会处理相机、变换矩阵、材质状态、纹理、深度测试等内容。

回忆一下我们之前写过的代码：

```ts
const positions = new Float32Array([
  0, 0, 0,
  1, 0, 0,
  0, 1, 0
])

const geometry = new THREE.BufferGeometry()

geometry.setAttribute(
  'position',
  new THREE.BufferAttribute(positions, 3)
)
```

这里的 `positions` 是 JavaScript 侧的 CPU 内存数据。它描述了三个顶点：

**CPU 内存中的顶点数据**

|   |   |   |   |
|---|---|---|---|
|**顶点**|**X**|**Y**|**Z**|
|V0|0|0|0|
|V1|1|0|0|
|V2|0|1|0|

Float32Array，每个顶点三个 float32，总共 36 字节。但是 GPU 并不能直接把这个 JavaScript 数组当作 GPU 顶点缓冲区使用。Three.js 会在渲染过程中，将它上传到 GPU，创建对应的 Buffer 对象。

因此threejs需要把数据上传到gpu里的话，需要使用到webgl，最典型的api就是：

```ts
const buffer = gl.createBuffer()

gl.bindBuffer(gl.ARRAY_BUFFER, buffer)

gl.bufferData(
  gl.ARRAY_BUFFER,
  positions,
  gl.STATIC_DRAW
)
```

使用webgl创建gpu buffer对象， bindBuffer将buffer绑定到目标目标，bufferData上传数据，ARRAY_BUFFER通常用于顶点属性数据。STATIC_DRAW提示数据通常不会频繁改变。

可以把这个过程理解为：

![](https://pub-dd8e151a60af44cfbe36beaaf0ad54e8.r2.dev/2026/11/1795505719665-h3iyq95w.webp)

这样我们就把顶点数据上传到了gpu，假设数据是不断更新的，我们也需要动态更新gpu buffer。

>**一个很重要的性能原则：不要每帧重新创建 Geometry 和 BufferAttribute。**

尽可能复用原有 TypedArray 和 GPU Buffer，只更新发生变化的数据。

shader programs解决了gpu把顶点变成了图像的问题，gpu buffer只解决了数据存储问题，还需要让gpu理解顶点在屏幕的位置、物体表面显示的颜色、哪些像素应该被遮挡、光照如何影响物体。如果写过glsl就可以方便理解，我们最简单自发光、环境光、平射光等等，可以看具体笔记。

Shader 是运行在 GPU 上的小程序。Three.js 常见的渲染管线主要涉及两种 Shader：
![](https://pub-dd8e151a60af44cfbe36beaaf0ad54e8.r2.dev/2026/11/1795505887497-5zstbme6.webp)

严格来说，Fragment Shader 处理的是 Fragment（片元），不完全等同于最终显示的 Pixel（像素）。因为片元可能在深度测试或其他阶段被丢弃。

我们需要处理顶点的变换，比如有一个立方体，它的顶点最初位于模型自身的局部坐标系。但是屏幕需要的是经过模型变换、相机变换和投影变换后的坐标。核心公式：

$$
 P_{clip}=P\cdot V\cdot M\cdot P_{local} 
$$
右边从右到左分别是顶点局部坐标、模型矩阵、视图矩阵、投影矩阵、裁剪空间坐标，对应到shader：

```vec
void main() {
    gl_Position =
        projectionMatrix *
        modelViewMatrix *
        vec4(position, 1.0);
}
```

这里使用的就叫做顶点着色器，这段代码实际上就完成了我们之前学习的 `Object3D`、`Matrix4` 和 Camera 之间的坐标变换。

顶点变换完成后，GPU 会把三角形光栅化，生成需要处理的片元，Fragment Shader 决定这些片元最终输出什么颜色。

一个最简单的片元着色器：

```vec
void main() {
    gl_FragColor = vec4(1.0, 0.0, 0.0, 1.0);
}
```
表示输出不透明红色，需要注意：这是 WebGL 1 风格的 GLSL 写法。现代 Three.js WebGLRenderer 默认使用 WebGL 2，自己编写 `ShaderMaterial` 时应使用对应 GLSL 版本支持的语法，例如：

```
void main() {
    gl_FragColor = vec4(1.0, 0.0, 0.0, 1.0);
}
```

但是你有threejs的情况下没必要去写glsl，我们平时可以直接编写材质信息：

```ts
const material = new THREE.MeshStandardMaterial({
  color: 0x3388ff,
  roughness: 0.5,
  metalness: 0.2
})
```

你看起来没有编写shader，实际上Three.js 会根据 Material 类型、灯光、阴影、纹理、渲染配置等条件，生成并管理相应的 Shader Program。

如果想自己控制 Shader，可以使用ShaderMaterial，这个可以让开发者自己编写glsl，但是任然是由threejs提供部分内置属性和自动处理。`RawShaderMaterial`：更接近原生 GLSL，需要自己声明更多输入和 Uniform。

例如，用 `ShaderMaterial` 输出纯红色：

```ts
const material = new THREE.ShaderMaterial({
  vertexShader: `
    void main() {
      gl_Position =
        projectionMatrix *
        modelViewMatrix *
        vec4(position, 1.0);
    }
  `,

  fragmentShader: `
    void main() {
      gl_FragColor = vec4(1.0, 0.0, 0.0, 1.0);
    }
  `
})
```

这就创建了一个由你自己控制着色逻辑的材质。

Shader 中还有三个非常重要的概念：

|类型|数据来源|特点|
|---|---|---|
|Attribute|Geometry / BufferAttribute|每个顶点可以不同|
|Uniform|CPU 侧设置|一次绘制中通常保持一致|
|Varying|Vertex Shader 输出|在光栅化过程中插值，供 Fragment Shader 使用|
例如：

```vec
// Vertex Shader
attribute vec3 position;
attribute vec3 color;

uniform mat4 modelViewMatrix;
uniform mat4 projectionMatrix;

varying vec3 vColor;

void main() {
    vColor = color;

    gl_Position =
        projectionMatrix *
        modelViewMatrix *
        vec4(position, 1.0);
}
```

但是这些都是gl的知识了，我们真的很少使用到这些在threejs里面，因此了解一下即可，如果系统学习过图形学，那当然更好。

最后是draw calls，通过它可以让gpu开始绘制，在 WebGL 中，典型的绘制指令是：


```ts
gl.drawArrays(
  gl.TRIANGLES,
  0,
  vertexCount
)

gl.drawElements(
  gl.TRIANGLES,
  indexCount,
  gl.UNSIGNED_SHORT,
  0
)
```

它们分别表示：
- `drawArrays`：按照顶点顺序绘制。
- `drawElements`：按照索引缓冲区指定的顶点绘制。

Three.js 会根据 Mesh、Geometry、Material、渲染分组等信息，组织并发出这些绘制指令。我们在简单场景里，比如一个demo里面，一个mesh通常对应了一个draw call：

```ts
const geometry = new THREE.BoxGeometry(1, 1, 1)
const material = new THREE.MeshBasicMaterial({
  color: 0x00ff00
})

for (let i = 0; i < 1000; i++) {
  const mesh = new THREE.Mesh(geometry, material)
  mesh.position.x = i * 2
  scene.add(mesh)
}
```
虽然这些 Mesh 共享 Geometry 和 Material，但它们仍然是 1000 个独立的渲染对象，在没有其他特殊情况时，可能产生约 1000 次 Draw Call，意思就是说，不会自动合并draw calls。每次 Draw Call，CPU 侧通常需要准备或检查 当前的shader programs、几何体对应的gpu buffer，当前material参数和纹理、模型变换矩阵、渲染状态比如混合和深度测试。然后向 GPU 提交绘制命令。

因此，GPU 算力很强，也不代表场景就一定流畅，有时瓶颈不在 GPU，而在 CPU 不断提交绘制命令。我们会使用InstancedMesh来减少draw calls次数：

```ts
const geometry = new THREE.BoxGeometry(1, 1, 1)

const material = new THREE.MeshBasicMaterial({
  color: 0x3399ff
})

const count = 100000

const mesh = new THREE.InstancedMesh(
  geometry,
  material,
  count
)

const dummy = new THREE.Object3D()

for (let i = 0; i < count; i++) {
  dummy.position.set(
    i % 100,
    Math.floor(i / 100),
    0
  )

  dummy.updateMatrix()

  mesh.setMatrixAt(i, dummy.matrix)
}

mesh.instanceMatrix.needsUpdate = true

scene.add(mesh)
```

Three.js 会使用实例化绘制机制，让 GPU 在一次实例化绘制中处理多个相同几何体的实例。在简单条件下，100,000 个体素可以从约 100,000 次 Draw Calls 降低到 1 次。但这里有一个重要限定：**InstancedMesh 不是无条件只有一次 Draw Call。**

要求是，如果一个几何体有多个material group，那就可能有多个draw call，多个材质通常要分开绘制。 不同渲染阶段，例如阴影 ，也可能产生额外绘制。对于现代引擎比如说threejs/playcanvas来说，都会尽可能减少渲染状态切换，并且批量提交可以一起绘制的几何体，**不仅仅是 Material 相同，还需要考虑 Shader、纹理、Geometry、渲染状态以及物体之间的关系。**

如果一个场景里1000个立方体都是用相同的几何体、MeshStandardMaterial、纹理、shader，处理起来，现代引擎会尝试将它们合并或批量处理，这样可以减少 CPU 向 GPU 提交绘制命令的次数，也可以减少重复的状态设置，但普通的相同 Material 并不意味着一定能合批，还需要解决不同物体的变换和几何数据问题。

现代引擎会使用很多种合批技术，比如说静态合批：

![](https://pub-dd8e151a60af44cfbe36beaaf0ad54e8.r2.dev/2026/11/1795506816308-su9ycym1.webp)

将不会移动的多个物体的几何数据合并到较大的 Buffer 中，比如说场景里的建筑、地面、固定障碍物，这样可以减少draw calls，但是需要额外的几何数据存储；物体不能随意独立变换，通常需要更新或重建合并后的几何数据。

动态合批：

每帧或在提交绘制前，将符合条件的小型动态几何体组织到一起，可以处理部分动态物体，CPU 侧需要额外处理顶点和变换，现代 GPU 场景中不一定划算。


gpu实例化：

同一 Geometry 和 Material，使用不同的 Instance Transform，一次实例化绘制大量物体，很适合重复几何体，但是通常要求共享几何结构和兼容的材质，渲染状态。

现代引擎还会是有一个叫做纹理数组的东西，英文叫做texture array，多张纹理组织在一张纹理图集，或兼容的纹理数组中。

```
Object A → Texture A
Object B → Texture B
Object C → Texture C

->

Object A ─┐
Object B ─┼── Texture Atlas
Object C ─┘
```

可以减少纹理绑定切换，但需要正确设置 UV，处理图集边缘和纹理过滤问题。并且现代GPU和图形API提供了进一步的绘制提交方式，它们可以在较少的CPU提交操作中发出多个绘制命令，另外，Bindless Textures 等技术可以减少纹理绑定的限制，让 Shader 能够通过句柄访问大量纹理资源。
这些技术在大型游戏引擎和专业渲染引擎中很重要，但 WebGL 和 Three.js 并不具备完整的原生 Vulkan/DX12 能力。

好了扯得有点远了，还是回到threejs上面。总之，我们使用drei来减少重复工作，threejs控制真正需要的渲染细节，这样既能保持代码精简，也不会因为过度依赖封装而失去对 GPU 的控制能力。

## drei提供的其他组件

### 场景标注

drei提供了Html组件，它可以把普通 HTML 元素绑定到 Three.js 的三维坐标上，实现类似 PlayCanvas 的 3D Annotation（场景标注）功能，而且它不仅能显示文字标签，还可以放置按钮、信息卡片、交互面板。

![](https://pub-dd8e151a60af44cfbe36beaaf0ad54e8.r2.dev/2026/11/1795508812105-pwf9hpkt.webp)


主要提供了position设置标签对应的三维坐标，center标签居中，distanceFactor根据相机距离缩放标签、occlude 能够被其他三维物体遮挡时候隐藏、transform使用css 3d矩阵变换等等，其他的自己查看。官方文档：[Drei Html](https://drei.docs.pmnd.rs/misc/html?utm_source=chatgpt.com)。




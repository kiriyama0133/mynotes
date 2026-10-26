---
date: 2026-10-25
tags:
  - cpp
  - python
  - open3d
  - pcl
  - realsense
---


通过两个对角的realsense相机采集绿篱点云后通过刚体变换到世界坐标系合成完成绿篱点云之后使用重建算法，大致是这个流程：

我负责的两个版本一个是通过vggt的端到端模型直接来输出点云模型的，现在又换成了两个realsense深度相机来重建了。

过程比较简单，相机应该是realsense d435，你可以下载官方的工具Interl Realsense SDK，里面会提供很多的工具，有的工具也可以直接读取相机内参，这个可以直接询问ai，ai会告诉你sdk有什么工具提供了可以读取相机内参信息的。

有关于内外参的作用可以学习线性变化之后学习刚体变换，这个用途很广，不只是建图里需要使用，图形学里也是经常使用的。学习之后就可以知道内外参的具体作用，如果有精力顺便可以学习李群和李代数，这些是机器人学里比较重要的内容，我们的刚体变换可以用李群李代数表示。

然后接下来的视觉方案听个乐就行，因为realsense d435这种深度相机说实话有效距离不行，也不能很好处理含有空洞的绿篱。所以我个人认为这个视觉方案比较尴尬，作用不大。如果有兴趣可以自己下去试试更好的视觉方案，或者说这个项目可能连视觉都不需要。

## 工具：

现在有ai agent这个强力的工具，个人认为，如果对代码能力比较苦手又不想练习的话，可以搭配claude code 、codex等工具来使用，所以无论用cpp python都是可以的。

python这块，比较好用的点云处理库是open3d，里面有重建、表面配准等算法；cpp这里最重要的是pcl库，全称叫做point cloud library，这个库最需要学习，处理点云是百分之百需要使用这个的，里面提供了数据结构和io操作、滤波、特征估计、风格、配准、曲面重建、kd-tree、八叉树空间检索等模块。

需要查看点云等工具，我推荐你下载个叫做CloudCompare的工具，你之后做点云分割还是查看点云都可以使用它，还有一个叫做meshlab，都可以下载，里面也提供了很多点云的重建算法，你可以直观的调整参数来重建看看效果，比代码操作方便一些。

对于需要进行上位机展示，如果可以用web工具的话，推荐python一个叫做viser的库，可以自定义控件和通过websocket将点云数据推到网页上展示，性能也还不错。
## 相机和处理环节：

两个相机在对角方向，朝下30°大概，通过调用深度相机采取到点云信息：

<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 800 600" width="100%" height="100%">
  <defs>
    <!-- 发光滤镜 -->
    <filter id="glow" x="-20%" y="-20%" width="140%" height="140%">
      <feGaussianBlur stdDeviation="3" result="blur" />
      <feMerge>
        <feMergeNode in="blur" />
        <feMergeNode in="SourceGraphic" />
      </feMerge>
    </filter>
    
    <!-- 渐变定义 -->
    <linearGradient id="frustumA" x1="0%" y1="0%" x2="100%" y2="100%">
      <stop offset="0%" stop-color="#00e5ff" stop-opacity="0.3"/>
      <stop offset="100%" stop-color="#00e5ff" stop-opacity="0"/>
    </linearGradient>
    <linearGradient id="frustumB" x1="100%" y1="0%" x2="0%" y2="100%">
      <stop offset="0%" stop-color="#ff9100" stop-opacity="0.3"/>
      <stop offset="100%" stop-color="#ff9100" stop-opacity="0"/>
    </linearGradient>

    <!-- 箭头标记 -->
    <marker id="arrow" viewBox="0 0 10 10" refX="5" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse">
      <path d="M 0 0 L 10 5 L 0 10 z" fill="#ffffff" />
    </marker>
    
    <!-- 相机模块组件 -->
    <g id="camera">
      <rect x="-30" y="-15" width="60" height="30" rx="4" fill="#2d2d2d" stroke="#00e5ff" stroke-width="2"/>
      <rect x="-10" y="15" width="20" height="8" fill="#555" />
      <circle cx="0" cy="20" r="4" fill="#00e5ff" filter="url(#glow)"/>
    </g>
  </defs>

  <!-- 背景 -->
  <rect width="100%" height="100%" fill="#121212" />
  
  <!-- 网格背景（装饰） -->
  <pattern id="grid" width="40" height="40" patternUnits="userSpaceOnUse">
    <path d="M 40 0 L 0 0 0 40" fill="none" stroke="#ffffff" stroke-opacity="0.05" stroke-width="1"/>
  </pattern>
  <rect width="100%" height="100%" fill="url(#grid)" />

  <!-- 绿篱 (Hedge) 区域 -->
  <rect x="100" y="480" width="600" height="80" rx="10" fill="#1b5e20" opacity="0.6"/>
  <path d="M 100 480 Q 150 460 200 480 T 300 480 T 400 480 T 500 480 T 600 480 T 700 480" stroke="#4caf50" stroke-width="2" fill="none"/>
  <text x="400" y="530" fill="#81c784" font-family="sans-serif" font-size="20" text-anchor="middle" font-weight="bold">绿篱表面 (Hedge)</text>

  <!-- 左上方相机 A -->
  <g transform="translate(200, 150)">
    <!-- 水平参考线与角度 -->
    <line x1="0" y1="0" x2="120" y2="0" stroke="#ffffff" stroke-opacity="0.3" stroke-dasharray="5,5" />
    <line x1="0" y1="0" x2="103" y2="60" stroke="#00e5ff" stroke-opacity="0.6" stroke-dasharray="4,4" />
    <path d="M 60 0 A 60 60 0 0 1 51 30" fill="none" stroke="#00e5ff" stroke-width="2"/>
    <text x="65" y="25" fill="#00e5ff" font-family="monospace" font-size="14">30°</text>
    
    <!-- 相机图标 -->
    <use href="#camera" />
    <text x="0" y="-25" fill="#00e5ff" font-family="sans-serif" font-size="16" font-weight="bold" text-anchor="middle">RealSense A (对角)</text>
    
    <!-- 视锥 (Frustum) -->
    <polygon points="0,20 -100,350 250,400" fill="url(#frustumA)" />
    <line x1="0" y1="20" x2="-100" y2="350" stroke="#00e5ff" stroke-width="1" stroke-opacity="0.5"/>
    <line x1="0" y1="20" x2="250" y2="400" stroke="#00e5ff" stroke-width="1" stroke-opacity="0.5"/>
  </g>

  <!-- 右上方相机 B -->
  <g transform="translate(600, 150)">
    <!-- 水平参考线与角度 -->
    <line x1="0" y1="0" x2="-120" y2="0" stroke="#ffffff" stroke-opacity="0.3" stroke-dasharray="5,5" />
    <line x1="0" y1="0" x2="-103" y2="60" stroke="#ff9100" stroke-opacity="0.6" stroke-dasharray="4,4" />
    <path d="M -60 0 A 60 60 0 0 0 -51 30" fill="none" stroke="#ff9100" stroke-width="2"/>
    <text x="-80" y="25" fill="#ff9100" font-family="monospace" font-size="14">30°</text>
    
    <!-- 相机图标 -->
    <use href="#camera" />
    <text x="0" y="-25" fill="#ff9100" font-family="sans-serif" font-size="16" font-weight="bold" text-anchor="middle">RealSense B (对角)</text>
    
    <!-- 视锥 (Frustum) -->
    <polygon points="0,20 -250,400 100,350" fill="url(#frustumB)" />
    <line x1="0" y1="20" x2="-250" y2="400" stroke="#ff9100" stroke-width="1" stroke-opacity="0.5"/>
    <line x1="0" y1="20" x2="100" y2="350" stroke="#ff9100" stroke-width="1" stroke-opacity="0.5"/>
  </g>

  <!-- 点云与重叠区域 -->
  <g filter="url(#glow)">
    <!-- 相机 A 的点云 (青色) -->
    <circle cx="160" cy="380" r="2" fill="#00e5ff" />
    <circle cx="180" cy="400" r="1.5" fill="#00e5ff" />
    <circle cx="150" cy="420" r="2" fill="#00e5ff" />
    <circle cx="200" cy="450" r="1" fill="#00e5ff" />
    <!-- 相机 B 的点云 (橙色) -->
    <circle cx="640" cy="380" r="2" fill="#ff9100" />
    <circle cx="620" cy="400" r="1.5" fill="#ff9100" />
    <circle cx="650" cy="420" r="2" fill="#ff9100" />
    <circle cx="600" cy="450" r="1" fill="#ff9100" />
    
    <!-- 重叠区域融合点云 (黄绿色) -->
    <circle cx="380" cy="420" r="2.5" fill="#ffff00" />
    <circle cx="400" cy="430" r="2" fill="#ffff00" />
    <circle cx="420" cy="410" r="2.5" fill="#ffff00" />
    <circle cx="370" cy="440" r="1.5" fill="#ffff00" />
    <circle cx="430" cy="440" r="2" fill="#ffff00" />
    <circle cx="410" cy="460" r="2.5" fill="#ffff00" />
    <circle cx="390" cy="470" r="1.5" fill="#ffff00" />
  </g>
  <text x="400" y="400" fill="#ffff00" font-family="sans-serif" font-size="14" text-anchor="middle">重叠融合点云</text>

  <!-- 世界坐标系 (World Coordinate System) -->
  <g transform="translate(400, 300)">
    <!-- Z轴 (向上) -->
    <line x1="0" y1="0" x2="0" y2="-80" stroke="#ffffff" stroke-width="2" marker-end="url(#arrow)"/>
    <text x="-5" y="-85" fill="#ffffff" font-family="sans-serif" font-size="14" text-anchor="end">Z (World)</text>
    
    <!-- X轴 (向右) -->
    <line x1="0" y1="0" x2="80" y2="0" stroke="#ffffff" stroke-width="2" marker-end="url(#arrow)"/>
    <text x="85" y="5" fill="#ffffff" font-family="sans-serif" font-size="14">X</text>
    
    <!-- Y轴 (斜向) -->
    <line x1="0" y1="0" x2="-50" y2="50" stroke="#ffffff" stroke-width="2" marker-end="url(#arrow)"/>
    <text x="-60" y="65" fill="#ffffff" font-family="sans-serif" font-size="14">Y</text>

    <!-- 刚体变换指示线 -->
    <path d="M -120 -80 Q -60 -120 0 -80" fill="none" stroke="#aaaaaa" stroke-width="1.5" stroke-dasharray="4,4" />
    <text x="-60" y="-110" fill="#aaaaaa" font-family="monospace" font-size="12" text-anchor="middle">T_A (R|t)</text>

    <path d="M 120 -80 Q 60 -120 0 -80" fill="none" stroke="#aaaaaa" stroke-width="1.5" stroke-dasharray="4,4" />
    <text x="60" y="-110" fill="#aaaaaa" font-family="monospace" font-size="12" text-anchor="middle">T_B (R|t)</text>
  </g>
  
  <!-- 标题 -->
  <text x="400" y="40" fill="#ffffff" font-family="sans-serif" font-size="24" font-weight="bold" text-anchor="middle">双 RealSense 绿篱点云采集与刚体变换</text>
  <text x="400" y="70" fill="#aaaaaa" font-family="sans-serif" font-size="16" text-anchor="middle">（对角视角 · 向下倾斜 30° · 融合至世界坐标系）</text>

</svg>

然后采集点云的时候需要让点云扰动不是太大，需要使用realsense提供的部分滤波器。你可以使用阈值滤波器，把背景和无效噪点给裁掉，避免远处的离散点混入点云，然后还有个叫时间滤波器，可以抵消树木的一些微颤导致的帧间抖动，保证深度数据稳定性。

这个是采集管道里就应该使用的，在之后我们还需要使用一些滤波器，适用于后处理的，比如说统计离群点滤波和体素滤波。

因为在采集完之后还是会出现很多**离散的点云**，这个对于之后进行重建任务会有很大麻烦，所以我们是要去除的，它基于每个点到其邻居的平均距离分布来判断是否为离群点。对于绿篱表面这种连续但带有细小起伏的结构比较适用，能去除那些明显脱离主体的孤立噪点。

然后就是直通滤波，可以设置阈值进行简单的裁切，把不管兴趣的区域比如前方后方背景快速切掉，最后是体素滤波，因为点云如果是数量比较庞大的情况下，对于重建任务也是会增加很多困难的，不只是时间方面，还可能造成许多的空洞等。所以通过体素滤波进行降采样，将点云数量降低一个量级。

## 刚体变换：

这个是拼接环节，目标是改变点云的位置和姿态，不改变点云大小和形状，将B相机的局部坐标系下点云对齐到A的局部坐标系下或者你以机器内部的中心点作为坐标系都是可以的，这个看你怎么选择，看他们路径规划怎么做。

我们要先做相机标定，这个是为了得到B到A的刚体变换矩阵，可以学习一下ChArUco标定板，你把标定板放在两个相机的公共视野下，移动标定板让每个相机采集20多张图，最后来求解，这个可以询问AI，网上有成熟的标定工具，ROS也是有的，或者你让ai现场写一个也是可以。

我们要计算的是 $T_{B \to A}$，也就是相机 B 到相机 A 的外参矩阵。

*   相机 A 看标定板，得到了标定板在 A 坐标系下的位姿：$T_{A \leftarrow O}$
*   相机 B 看标定板，得到了标定板在 B 坐标系下的位姿：$T_{B \leftarrow O}$

我们要找的是 $T_{A \leftarrow B}$（把 B 坐标系里的点，变到 A 坐标系里）。

根据坐标变换的链式法则：
$$
P_A = T_{A \leftarrow B} \cdot P_B
$$

又因为：
$$
P_A = T_{A \leftarrow O} \cdot P_O
$$
$$
P_B = T_{B \leftarrow O} \cdot P_O
$$

所以，我们可以把 $P_O$ 用 $P_B$ 表示出来：
$$
P_O = (T_{B \leftarrow O})^{-1} \cdot P_B
$$

将上式代入 $P_A$ 的方程中：
$$
P_A = T_{A \leftarrow O} \cdot (T_{B \leftarrow O})^{-1} \cdot P_B
$$

对比最开始的公式 $P_A = T_{A \leftarrow B} \cdot P_B$，即可得出最终结论：
$$
T_{A \leftarrow B} = T_{A \leftarrow O} \cdot (T_{B \leftarrow O})^{-1}
$$

上述公式是直接复制AI的结果，可以看一下用作参考。这样就可以得到一个刚体变换的矩阵了，是一个4X4的，包含了旋转+平移。然后我们可以把B中的点云通过变换到A的坐标系里去。相机A和相机B可能同时扫描到一个绿篱表面，你也可以再跑一遍体素，如果必要的话。

## 重建

之前的项目我们是使用的滚球法重建，叫做BPA，就像一个固定半径的小球在点云表面滚动，碰到三个点就组成一个三角面，因此对于电云质量是比较苛刻的。有一个算法相比BPA对于噪点和密度不匀的包容度远高于BPA，叫做泊松重建，这个可以把原本不封闭的空洞给补上。还有就是Alpha Shapes，适合提取轮廓，适合生成绿篱的外表表面，但是受干扰比较大。

如果感兴趣可以去CloudCompare里试试。不过我一般不会这样干，我当时是直接用凸包来计算包络的，基于点云的凸包ConvexHull，可以下去了解一下，这个用于计算包络体积，这个比强行重建更准确和快速。

然后你刚才说的网格平面化，可能是ransac平面拟合，如果你用深度相机采集的点云是含有地面的话，你重建出来地面会类似一个平面，那么这个平面你就可以用ransac平面拟合算法你得到一个平面方程，这个pcl和open3d似乎都有相关的算子，可以直接用。

最后你得到一模型就可以交给做路径规划的负责人了，他们就可以来切片分析和处理。差不多就是这样。




---
date: 2026-10-21
tags:
  - architecture
  - sdl
  - canvas
  - wasm
  - math
---

本笔记用于拆解 `kiriyama_canvas`（C++23 + SDL3 + WebAssembly 2D 图表与画布组件库）的架构设计。全文采用**精简文本 + 核心代码块 + 数学推导**的形式，按 **“具体渲染与更新管线 → 分层与模块设计 → 点击测试与交互模块的数学实现”** 展开。

---

## 第一章：具体管线设计（从输入同步到双通道上屏）

在 `kiriyama_canvas` 中，每一帧（16.6ms）的执行流由 Web 宿主 [`CanvasHost::onFrame`](file:///E:/kiriyama_canvas/platform/web/CanvasHost.cc#L148-L212) 统一驱动，分为五个严格顺序的阶段：

<div align="center">
<svg viewBox="0 0 860 155" width="100%" height="auto" xmlns="http://www.w3.org/2000/svg" style="max-width: 860px; font-family: system-ui, -apple-system, sans-serif;">
  <defs>
    <marker id="arr0" viewBox="0 0 10 10" refX="6" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse">
      <path d="M 0 1 L 10 5 L 0 9 z" fill="#64748b"/>
    </marker>
  </defs>
  <rect width="860" height="155" rx="12" fill="#fafaf9" stroke="#e2e0d8" stroke-width="1.5"/>

  <rect x="16" y="26" width="144" height="102" rx="8" fill="#ede9fe" stroke="#8b5cf6" stroke-width="1.5"/>
  <text x="88" y="52" text-anchor="middle" font-size="12.5" font-weight="700" fill="#4c1d95">1. 帧首同步与事件</text>
  <text x="88" y="74" text-anchor="middle" font-size="11" fill="#5b21b6">pumpOne() 消费暂存</text>
  <text x="88" y="92" text-anchor="middle" font-size="11" fill="#5b21b6">dispatchInputEvent()</text>
  <text x="88" y="110" text-anchor="middle" font-size="10.5" fill="#6d28d9">按键/滚轮树状分发</text>

  <rect x="186" y="26" width="152" height="102" rx="8" fill="#fef3c7" stroke="#f59e0b" stroke-width="1.5"/>
  <text x="262" y="52" text-anchor="middle" font-size="12.5" font-weight="700" fill="#78350f">2. 交互步进与双轨更新</text>
  <text x="262" y="74" text-anchor="middle" font-size="11" fill="#92400e">Widget::tickTree(ctx)</text>
  <text x="262" y="92" text-anchor="middle" font-size="11" fill="#92400e">慢路径: build + diff</text>
  <text x="262" y="110" text-anchor="middle" font-size="10.5" fill="#b45309">快路径: O(1) 原地改属性</text>

  <rect x="364" y="26" width="152" height="102" rx="8" fill="#fce7f3" stroke="#ec4899" stroke-width="1.5"/>
  <text x="440" y="52" text-anchor="middle" font-size="12.5" font-weight="700" fill="#831843">3. 场景树遍历与解析</text>
  <text x="440" y="74" text-anchor="middle" font-size="11" fill="#9d174d">世界矩阵与透明度累乘</text>
  <text x="440" y="92" text-anchor="middle" font-size="11" fill="#9d174d">ColorToken 主题色解析</text>
  <text x="440" y="110" text-anchor="middle" font-size="10.5" fill="#be185d">pushClip 视口裁剪栈</text>

  <rect x="542" y="26" width="152" height="102" rx="8" fill="#dbeafe" stroke="#3b82f6" stroke-width="1.5"/>
  <text x="618" y="52" text-anchor="middle" font-size="12.5" font-weight="700" fill="#1e3a8a">4. 双通道光栅化</text>
  <text x="618" y="74" text-anchor="middle" font-size="11" fill="#1e40af">通道A: 凸扇切/凹耳切</text>
  <text x="618" y="92" text-anchor="middle" font-size="11" fill="#1e40af">通道B: 全白双级字库缓存</text>
  <text x="618" y="110" text-anchor="middle" font-size="10.5" fill="#1d4ed8">SDL_RenderGeometry</text>

  <rect x="720" y="26" width="124" height="102" rx="8" fill="#dcfce7" stroke="#22c55e" stroke-width="1.5"/>
  <text x="782" y="52" text-anchor="middle" font-size="12.5" font-weight="700" fill="#14532d">5. 顶层合成上屏</text>
  <text x="782" y="74" text-anchor="middle" font-size="11" fill="#166534">updateCursor()</text>
  <text x="782" y="92" text-anchor="middle" font-size="11" fill="#166534">TooltipHost 浮层</text>
  <text x="782" y="110" text-anchor="middle" font-size="10.5" fill="#15803d">Present 提交帧缓冲</text>

  <line x1="160" y1="77" x2="184" y2="77" stroke="#64748b" stroke-width="1.5" marker-end="url(#arr0)"/>
  <line x1="338" y1="77" x2="362" y2="77" stroke="#64748b" stroke-width="1.5" marker-end="url(#arr0)"/>
  <line x1="516" y1="77" x2="540" y2="77" stroke="#64748b" stroke-width="1.5" marker-end="url(#arr0)"/>
  <line x1="694" y1="77" x2="718" y2="77" stroke="#64748b" stroke-width="1.5" marker-end="url(#arr0)"/>
</svg>
</div>

### 1.1 总控管线代码（`CanvasHost::onFrame`）

JS 调用 C API（如 `kiriyama_pie_add_slice`）时**绝不直接修改场景树**，而是写入句柄对应的暂存槽位并标记 `dirty = true`，统一在帧首 `pumpOne()` 消费，从而彻底杜绝跨语言回调重入导致迭代器失效：

```cpp
// 文件：platform/web/CanvasHost.cc
void onFrame(App& app) override {
    // 【阶段 1】同步组件实例，并将 JS 暂存区的待处理数据灌入组件
    syncInstances(app);
    const auto ctx = frameContext(app); // 打包 dt 与 960x540 逻辑空间鼠标坐标
    for (const auto& [handle, instance] : instances_) {
        pumpOne(handle, *instance);
    }

    // 【阶段 2】树状交互步进：自顶向下执行命中检测、状态流转与按需 rebuild / 原地更新
    for (const auto& [handle, instance] : instances_) {
        if (instance->pie && instance->pie->visible())   instance->pie->tickTree(ctx);
        if (instance->spark && instance->spark->visible()) instance->spark->tickTree(ctx);
        if (instance->table && instance->table->visible()) instance->table->tickTree(ctx);
    }

    // 【阶段 3 & 4】按当前主题背景色清屏，并执行「先几何、后文本」的双通道渲染
    const core::ChartTheme& theme = kiriyama::web::currentTheme();
    app.painter().clear(theme.background);
    for (const auto& [handle, instance] : instances_) {
        if (instance->pie && instance->pie->visible()) {
            drawSceneGeometry(app.painter(), *instance->pie, theme); // 通道 A：几何图元
            app.text().drawScene(*instance->pie, theme);             // 通道 B：文字贴图
        }
        // ... spark 与 table 同理
    }

    // 【阶段 5】帧末顶层合成：更新鼠标手型光标，并在最顶层绘制全局唯一 Tooltip
    updateCursor(app);
    if (const int active = instanceUnderPointer(app); active != 0) {
        kiriyama::web::applyTooltip(active, *tooltip_);
    }
    tooltip_->update(*root_, ctx);
}
```

---

### 1.2 阶段二拆解：组件的双轨更新机制（`rebuild` 慢路径 vs $O(1)$ 快路径）

在 `tickTree(ctx)` 驱动组件的 `onUpdate(ctx)` 时，组件将更新严格拆分为**结构重排（慢路径）**与**高频交互（快路径）**两条轨道：

#### 1. 慢路径（`rebuild()`）：当且仅当数据、尺寸或滚动窗口变化时触发
组件调用 `core::build*` 纯函数生成零状态的 `SceneNode` 描述树，再通过 `core::reconcileScene` 与常驻子树 `inner_` 对账复用，最后收集叶子节点的裸指针供快路径直达：

```cpp
// 文件：platform/src/src/TableWidget.cc
void TableWidget::rebuild() {
    ensureColumnsMeasured(); // 仅在 fontDirty_ 时测量一次列宽

    // 1. 下沉到 core 层：计算虚拟滚动视口 [firstRow, lastRow] 并构建可见行描述树
    core::TableBuildResult built = core::buildTable(
        data_, state_, config_, style_, frame().size(), columnWidths_, scrollOffset_);
    layout_ = std::move(built.layout);

    // 2. 虚拟树对账：按 key ("row.3") 复用常驻节点，未变属性零标脏
    if (built.root) {
        core::reconcileScene(*inner_, *built.root);
    }
    collectRowNodes(); // 缓存当前可见行的背景节点指针与逻辑行号

    layoutDirty_ = false;
    lastSize_ = frame().size();
}
```

#### 2. 快路径（$O(1)$ 原地修改）：悬停高亮、选中切换、扇区外扩动画
高频交互**完全绕过 `rebuild()` 与 `reconcileScene()`**，直接通过 `rebuild()` 阶段缓存的节点指针原地修改几何顶点或样式属性：

```cpp
// 快路径示例 A（PieChart.cc）：60FPS 扇区外扩动画，直接改单个 Polygon 节点的顶点数组
void PieChart::applyExplode(std::size_t index, double radius) {
    core::SceneNode* node = sliceNodes_[index];
    PieLayout expanded = layout_;
    expanded.outerRadius += radius;
    node->setPoints(core::sectorPolygon(layout_.sectors[index], expanded, segments_));
}

// 快路径示例 B（SceneBuilder.cc）：表格悬停/选中行切换，O(可见行数) 原地刷写语义背景色
void updateTableHighlights(std::span<const std::pair<std::size_t, SceneNode*>> rowBgNodes,
                           std::optional<std::size_t> selectedRow,
                           std::optional<std::size_t> hoveredRow,
                           const TableStyle& style) {
    for (const auto& [rowIndex, bgNode] : rowBgNodes) {
        if (!bgNode) continue;
        SceneStyle bgStyle = bgNode->style();
        if (selectedRow && *selectedRow == rowIndex) {
            bgStyle.fill = style.rowSelectedFill; // ColorToken::TableRowSelected
        } else if (hoveredRow && *hoveredRow == rowIndex) {
            bgStyle.fill = style.rowHoverFill;    // ColorToken::TableRowHover
        } else {
            bgStyle.fill = (rowIndex % 2 == 1) ? style.rowAltFill : style.rowFill;
        }
        bgNode->setStyle(bgStyle); // SceneNode 内部相等短路，未变行不触发 markDirty()
    }
}
```

---

### 1.3 阶段三拆解：场景树递归遍历与语义色解析（`SceneDraw.cc`）

[`drawSceneGeometry`](file:///E:/kiriyama_canvas/platform/src/src/SceneDraw.cc#L89-L187) 自顶向下遍历 `SceneNode` 树，在遍历栈上实时完成**仿射矩阵合成、透明度累乘、语义色 Token 解析与局部裁剪框压栈**：

```cpp
// 文件：platform/src/src/SceneDraw.cc
void drawNode(Painter& painter, const SceneNode& node, const Affine2D& parentWorld,
              double parentOpacity, const core::ChartTheme& theme,
              const std::vector<core::Color>* seriesPalette) {
    if (!node.visible()) return;

    // 1. 透明度累乘剪枝
    const core::SceneStyle& style = node.style();
    const double effective = parentOpacity * style.opacity;
    if (effective <= 0.0) return;

    // 2. 色板继承与 ColorToken 实时解析（切换深浅色主题时无需改动场景树任何节点）
    const auto* palette = style.seriesPalette ? style.seriesPalette.get() : seriesPalette;
    const core::Color fill = style.fill.resolve(theme, palette);
    const core::Color stroke = style.stroke.resolve(theme, palette);

    // 3. 世界矩阵合成：world = parentWorld * localTransform
    const Affine2D world = parentWorld * node.transform();

    // 4. 压入子树裁剪框（如表格滚动视口 clipRect）
    SDL_Rect savedClip{};
    const bool clipped = pushClip(painter, world, node.clipRect(), savedClip);

    // 5. 按图元类型（Rect / Ellipse / Polygon / Polyline / Line）调用 Painter 落笔
    // ...

    // 6. 递归绘制子节点并恢复裁剪栈
    for (const auto& child : node.children()) {
        if (child) drawNode(painter, *child, world, effective, theme, palette);
    }
    restoreClip(painter, clipped, savedClip);
}
```

---

### 1.4 阶段四拆解：双通道光栅化（几何三角剖分 + 文本双级缓存）

为什么分成**几何通道（`drawSceneGeometry`）**与**文本通道（`SdlTextRenderer::drawScene`）**两遍遍历？
1. **Z 序保证**：图表数值标签永远浮在扇区或柱体之上，不会因树节点交错被相邻图元遮挡；
2. **GPU 状态零抖动**：几何通道全走无纹理顶点批次（`SDL_RenderGeometry`），文本通道全走字库纹理贴图（`SDL_RenderTexture`）。

#### 通道 A：`Painter::fillPolygon` 凸/凹自适应三角剖分
当环形图（Donut）扇区扫角超过 $180^\circ$ 时，内弧向内凹陷构成**凹多边形**。若盲目使用三角扇（Triangle Fan），连线会直接穿过中心圆孔造成“白楔子”伪影。[`Painter.cc`](file:///E:/kiriyama_canvas/platform/src/src/Painter.cc#L264-L353) 采用叉积符号检测凸性，自动分流：

```cpp
// 文件：platform/src/src/Painter.cc
bool convex = true;
for (std::size_t k = 0; k < n; ++k) {
    // 检查相邻边转向叉积是否与多边形整体绕向 (orientation) 一致
    if (crossAt(clean[k], clean[(k + 1) % n], clean[(k + 2) % n]) * orientation < 0.0) {
        convex = false;
        break;
    }
}

if (convex) {
    // 【快路径】凸多边形（圆角矩形、椭圆、常规扇区）：O(n) 三角扇 (0, i, i+1)
    for (std::size_t i = 1; i + 1 < n; ++i) {
        indices.insert(indices.end(), {0, static_cast<int>(i), static_cast<int>(i + 1)});
    }
} else {
    // 【精确路径】凹多边形（>180° 甜甜圈扇环）：反射点耳切法 (Ear-Clipping)
    // 仅收集内角 >180° 的凹顶点 (reflex)，判断候选耳三角 (prev, here, next) 内是否包含反射点
    collectReflex();
    while (ring.size() > 3) { /* 剪除无反射点落入的凸耳并更新邻点凹凸性 */ }
}
```

#### 通道 B：`SdlTextRenderer` 双级全白纹理缓存
[`TextRenderer.cc`](file:///E:/kiriyama_canvas/platform/src/src/TextRenderer.cc#L229-L368) 通过三项设计解决 Canvas 文字渲染的性能与清晰度痛点：
- **双级缓存**：默认 `letterSpacing == 0` 时走 **`LineCache`（512 条整行纹理）**，直接调用 `TTF_RenderText_Blended` 保留 FreeType 内部 26.6 定点数亚像素推进与 Kerning 字偶距；仅在自定义字距时退回 **`GlyphCache`（4096 字紧凑位图，`TTF_GetGlyphImage`）**。
- **纯白纹理 + `ColorMod` 染色**：缓存 Key 仅含 `(TTF_Font*, sizePt, text)`，纹理统一生成为纯白色 `(255,255,255,255)`，绘制时用 `SDL_SetTextureColorMod` 动态乘算目标颜色，文字换色/换主题 **100% 复用原纹理**。
- **整像素吸附**：开启 `SDL_SCALEMODE_NEAREST` 并对落笔坐标执行 `std::round` 整像素对齐，消除半像素线性插值导致的字体发虚。

---

## 第二章：分层与核心模块设计

### 2.1 零 SDL 依赖的黄金分割线

仓库自底向上分为四层，**前三大层（`math`、`core`、`kiriyama_scene` + `kiriyama_text`）完全不引入 `<SDL3/SDL.h>`**，所有几何计算、虚拟树对账、组件交互与按键分发均可直接编译为纯逻辑 WASM 在 Node.js 下运行 GoogleTest：

```text
kiriyama_canvas/
├── math/                  # [第1层] 纯数学库：Vec2d, Affine2D, Easing, Spring(解析弹簧), Ticks
├── core/                  # [第2层] 核心模型与纯函数算法（零 SDL）：
│   ├── SceneNode / Diff   #   抽象场景树节点 + reconcileScene() 虚拟树对账
│   ├── Geometry / HitTest #   几何原语、射线法、线段投影、极坐标扇区命中
│   ├── Table / TableLayout#   表格排序状态机(排索引数组 order_)、列宽测量、滚动数学
│   └── SceneBuilder       #   纯函数构建器：buildPieChart / buildSparkBar / buildTable
└── platform/
    ├── src/scene + text   # [第3层] 有状态交互组件层（零 SDL）：
    │                      #   Widget, InputController, PieChart, SparkBar, TableWidget
    └── src/render + web   # [第4层] 唯一接触 SDL3 的渲染后端与 Web C API 宿主
```

---

### 2.2 `core` 与 `scene` 的职责分离：纯函数构建器 + 轻量交互外壳

在架构设计中，平台交互层的 `Widget` 子类（如 `TableWidget`、`SparkBar`、`PieChart`）**绝不直接掺杂复杂的排版与坐标算术**，而是遵循严格的“数据自持 + 布局下沉 `core` + 原地定位隔离”**模式：

```cpp
// 文件：platform/src/include/kiriyama/scene/TableWidget.h & PieChart.cc
class TableWidget : public Widget {
private:
    core::TableData data_{};                  // 1. 组件通过 move 语义独占持有业务数据
    core::TableState state_{};                // 2. 排序与选中状态机（只排索引 order_，不搬移 rows）
    core::SceneNode* inner_{nullptr};         // 3. 内部对账根组 (key="table")
    std::vector<std::pair<std::size_t, core::SceneNode*>> rowBgNodes_; // 快路径指针缓存
};
```

#### 为什么组件要拆成「自身 `transform_` 定位 + 子节点 `inner_` 对账」？
```cpp
// 文件：platform/src/src/TableWidget.cc
void TableWidget::onUpdate(const FrameContext& /*ctx*/) {
    // 将外部设置的 frame().origin() 转移到组件自身的仿射矩阵 transform_ 平移分量上，
    // 保持自身 frame_ 恒为 {0, 0, width, height}！
    if (frame().x != 0.0 || frame().y != 0.0) {
        setTranslation(frame().origin());
        setFrame({0.0, 0.0, frame().width, frame().height});
    }
    if (layoutDirty_ || !(frame().size() == lastSize_)) {
        rebuild(); // 仅尺寸或数据变化才重排；在画布上平移组件只改 transform_，零 rebuild！
    }
}
```
1. **平移零重排**：`core::buildTable` 生成的节点坐标永远以 `(0, 0)` 为局部原点；组件在画布上的位置 `(x, y)` 存在 `Widget` 自身的 `transform_` 矩阵里，移动组件位置完全不触发 `rebuild()`。
2. **保护根属性**：`reconcileScene(*inner_, *built.root)` 只在 `inner_` 子节点上原地对账，不会覆盖 `Widget` 自身的 `transform_`、`frame_` 和事件绑定。

---

### 2.3 虚拟场景树对账模块（`SceneDiff::reconcileScene`）

当慢路径触发 `rebuild()` 时，[`SceneDiff.cc`](file:///E:/kiriyama_canvas/core/src/SceneDiff.cc#L87-L181) 将新描述树 `source` 对账合并到常驻树 `target`，核心由三步构成：

```cpp
// 文件：core/src/SceneDiff.cc
void reconcileChildren(SceneNode& target, const SceneNode& source, SceneDiffResult& out) {
    // 1. O(1) 移出旧子节点数组，并建立 key -> 旧下标 哈希表（如 "row.5" -> 5）
    std::vector<std::unique_ptr<SceneNode>> oldChildren = target.takeChildren();
    std::unordered_map<std::string_view, std::size_t> keyIndex;
    for (std::size_t i = 0; i < oldChildren.size(); ++i) {
        if (oldChildren[i] && !oldChildren[i]->key().empty())
            keyIndex.emplace(oldChildren[i]->key(), i);
    }

    // 2. 遍历新树子节点：命中 key 且 NodeType 相同则 std::move 直接复用旧节点 unique_ptr
    for (std::size_t i = 0; i < source.childCount(); ++i) {
        const SceneNode* src = source.childAt(i);
        // ... 查找 match 下标 ...
        if (match != kNoMatch) {
            reconcileNode(*oldChildren[match], *src, out); // 仅当属性真变了才 markDirty()
            if (!src->key().empty()) matchedOldIndices.push_back(match);
            next.push_back(std::move(oldChildren[match])); // 保持原节点指针与唯一 id_ 不变
        } else {
            auto cloned = src->clone();
            cloned->markSubtreeDirty();
            next.push_back(std::move(cloned));
        }
    }

    // 3. 用 O(n log n) 最长递增子序列 (LIS) 判定兄弟节点顺序是否发生移动
    const std::size_t moves = minimumMoves(matchedOldIndices);
    const bool structuralChange = (insertedDelta > 0 || removedDelta > 0 || moves > 0);
    target.setChildren(std::move(next), structuralChange); // 结构未变则绝不污染父节点 dirty_
}
```

其中 [`minimumMoves`](file:///E:/kiriyama_canvas/core/src/SceneDiff.cc#L46-L61) 利用 `std::lower_bound` 维护单调栈求最长递增子序列长度 $\text{LIS}$，最小节点移动次数即为 $N_{\text{matched}} - |\text{LIS}|$。

---

## 第三章：点击测试（Hit-Testing）与交互模块的数学实现

本节深入拆解 `math/`、`core/` 与 `scene/` 中支撑精准点击测试与丝滑交互的底层数学公式及代码实现。

---

### 3.1 空间降维：2D 仿射逆变换与奇异矩阵防 NaN 保护

#### 1. 仿射矩阵表示与世界-局部逆映射
[`Affine2D`](file:///E:/kiriyama_canvas/math/include/kiriyama/math/Transform2D.h#L16-L95) 采用齐次坐标前两行的 $3 \times 2$ 矩阵存储：

$$
M = \begin{bmatrix} m_{00} & m_{01} & m_{02} \\ m_{10} & m_{11} & m_{12} \\ 0 & 0 & 1 \end{bmatrix}, \quad
M \begin{bmatrix} x \\ y \\ 1 \end{bmatrix} = \begin{bmatrix} m_{00}x + m_{01}y + m_{02} \\ m_{10}x + m_{11}y + m_{12} \\ 1 \end{bmatrix}
$$

对于场景树中任意深度为 $k$ 的节点，其世界变换矩阵为祖先链累乘 $M_{\text{world}} = M_0 M_1 \cdots M_k$。在做点击测试时，**绝不把节点内部成百上千个多边形顶点乘到世界空间**，而是将世界空间的鼠标点 $\mathbf{p}_w$ 乘以逆矩阵 $M_{\text{world}}^{-1}$ 拉回节点的局部空间：

$$
\mathbf{p}_{\text{local}} = M_{\text{world}}^{-1} \, \mathbf{p}_w
$$

对于线性部分 $A = \begin{bmatrix} m_{00} & m_{01} \\ m_{10} & m_{11} \end{bmatrix}$ 与平移向量 $\mathbf{t} = \begin{bmatrix} m_{02} \\ m_{12} \end{bmatrix}$，行列式 $\det(A) = m_{00}m_{11} - m_{01}m_{10}$，其逆矩阵为 $[A^{-1} \mid -A^{-1}\mathbf{t}]$：

$$
M^{-1} = \frac{1}{\det(A)} \begin{bmatrix} m_{11} & -m_{01} & m_{01}m_{12} - m_{11}m_{02} \\ -m_{10} & m_{00} & m_{10}m_{02} - m_{00}m_{12} \\ 0 & 0 & \det(A) \end{bmatrix}
$$

```cpp
// 文件：math/include/kiriyama/math/Transform2D.h
constexpr Affine2D inverse(double epsilon = kEpsilon) const noexcept {
    const double det = determinant(); // m00 * m11 - m01 * m10
    if (!std::isfinite(det) || std::abs(det) <= epsilon) {
        // ★ 防 NaN 保护：当组件处于缩放动画第 0 帧 (scale = 0) 时 det == 0，
        // 严禁除以 0 产生 NaN 污染后续所有几何比较，安全回退为单位矩阵 Identity()
        return Identity();
    }
    const double inv = 1.0 / det;
    return {
        m11 * inv,
        -m01 * inv,
        (m01 * m12 - m11 * m02) * inv,
        -m10 * inv,
        m00 * inv,
        (m10 * m02 - m00 * m12) * inv,
    };
}
```

#### 2. 局部退化线段的旋转包围盒计算（`transformedBounds`）
当计算节点在世界空间的轴对齐包围盒（AABB）用于粗筛时，若一个节点是水平或垂直线段（`width == 0` 或 `height == 0`），标准 `RectD::isEmpty()` 会误判为空。[`Geometry.cc`](file:///E:/kiriyama_canvas/core/src/Geometry.cc#L45-L64) 专门允许单轴为 0 的退化矩形参与四顶点极值包络计算，确保水平/垂直网格线在叠加 `tolerance` 容差后仍能被正确命中：

```cpp
// 文件：core/src/Geometry.cc
RectD transformedBounds(const RectD& rect, const Affine2D& transform) noexcept {
    if (rect.width < 0.0 || rect.height < 0.0 ||
        (rect.width == 0.0 && rect.height == 0.0)) {
        return {}; // 仅拒绝负尺寸或退化为单个点的空盒，保留 width==0 或 height==0 的线段！
    }
    const std::array<Vec2d, 4> corners = {
        transform.apply({rect.x, rect.y}),
        transform.apply({rect.right(), rect.y}),
        transform.apply({rect.right(), rect.bottom()}),
        transform.apply({rect.x, rect.bottom()}),
    };
    return boundsOfPoints(corners); // 取 4 个变换后角点的 [minX, minY, maxX - minX, maxY - minY]
}
```

---

### 3.2 点击测试（Hit-Testing）的五大几何数学实现

<div align="center">
<svg viewBox="0 0 840 290" width="100%" height="auto" xmlns="http://www.w3.org/2000/svg" style="max-width: 840px; font-family: system-ui, -apple-system, sans-serif;">
  <rect width="840" height="290" rx="12" fill="#fafaf9" stroke="#e2e0d8" stroke-width="1.5"/>

  <!-- Panel 1: Line Segment -->
  <rect x="20" y="18" width="255" height="254" rx="8" fill="#ffffff" stroke="#cbd5e1" stroke-width="1.2"/>
  <text x="147" y="42" text-anchor="middle" font-size="12.5" font-weight="700" fill="#0f172a">1. 线段/折线：正交投影参数夹逼</text>
  <line x1="55" y1="185" x2="225" y2="90" stroke="#3b82f6" stroke-width="3"/>
  <circle cx="55" cy="185" r="4.5" fill="#1d4ed8"/>
  <text x="42" y="203" font-size="11" fill="#1e293b" font-weight="600">A</text>
  <circle cx="225" cy="90" r="4.5" fill="#1d4ed8"/>
  <text x="232" y="94" font-size="11" fill="#1e293b" font-weight="600">B</text>
  <circle cx="115" cy="85" r="4.5" fill="#ef4444"/>
  <text x="108" y="73" font-size="11" fill="#dc2626" font-weight="600">P (鼠标)</text>
  <line x1="115" y1="85" x2="144" y2="135" stroke="#ef4444" stroke-width="1.5" stroke-dasharray="4 3"/>
  <circle cx="144" cy="135" r="4" fill="#10b981"/>
  <text x="152" y="146" font-size="11" fill="#047857" font-weight="600">Q = A + t(B-A)</text>
  <text x="34" y="226" font-size="11" fill="#334155">t = clamp((P-A)·(B-A)/|B-A|², 0, 1)</text>
  <text x="34" y="245" font-size="11" fill="#334155">最短距离 d = |P - Q| ≤ tolerance</text>
  <text x="34" y="262" font-size="10.5" fill="#64748b">折线命中后通过 nearestEndpoint 吸附端点</text>

  <!-- Panel 2: Pie Sector -->
  <rect x="292" y="18" width="255" height="254" rx="8" fill="#ffffff" stroke="#cbd5e1" stroke-width="1.2"/>
  <text x="420" y="42" text-anchor="middle" font-size="12.5" font-weight="700" fill="#0f172a">2. 扇环：极坐标 + 模 2π 角度折回</text>
  <path d="M 380 142 L 435 78 A 85 85 0 0 1 495 168 L 425 153 A 35 35 0 0 0 403 116 Z" fill="#dbeafe" stroke="#2563eb" stroke-width="2"/>
  <circle cx="380" cy="142" r="3" fill="#0f172a"/>
  <text x="352" y="157" font-size="10.5" fill="#334155">C(圆心)</text>
  <circle cx="452" cy="130" r="4.5" fill="#ef4444"/>
  <text x="460" y="133" font-size="11" fill="#dc2626" font-weight="600">P</text>
  <line x1="380" y1="142" x2="452" y2="130" stroke="#ef4444" stroke-width="1.2" stroke-dasharray="3 2"/>
  <text x="306" y="210" font-size="11" fill="#334155">极径：R_in - ε ≤ |P - C| ≤ R_out + ε</text>
  <text x="306" y="229" font-size="11" fill="#334155">极角：θ = atan2(Py-Cy, Px-Cx)</text>
  <text x="306" y="248" font-size="11" fill="#334155">折回：wrap_[0,2π)(θ - θ_start) ≤ sweep</text>
  <text x="306" y="262" font-size="10.5" fill="#64748b">外扩动画期动态补偿 R_out += explode·p</text>

  <!-- Panel 3: Polygon Ray Casting -->
  <rect x="565" y="18" width="255" height="254" rx="8" fill="#ffffff" stroke="#cbd5e1" stroke-width="1.2"/>
  <text x="692" y="42" text-anchor="middle" font-size="12.5" font-weight="700" fill="#0f172a">3. 任意多边形：半开区间奇偶射线法</text>
  <polygon points="595,98 665,73 650,133 725,83 765,163 620,183" fill="#dcfce7" stroke="#16a34a" stroke-width="2"/>
  <circle cx="628" cy="128" r="4.5" fill="#ef4444"/>
  <text x="605" y="121" font-size="11" fill="#dc2626" font-weight="600">P</text>
  <line x1="628" y1="128" x2="795" y2="128" stroke="#ef4444" stroke-width="1.5" stroke-dasharray="4 3"/>
  <circle cx="653" cy="128" r="3.5" fill="#9333ea"/>
  <circle cx="660" cy="128" r="3.5" fill="#9333ea"/>
  <circle cx="747" cy="128" r="3.5" fill="#9333ea"/>
  <text x="580" y="210" font-size="11" fill="#334155">向右发射水平射线 y = Py, x ≥ Px：</text>
  <text x="580" y="229" font-size="11" fill="#334155">• 异或判跨立：(Ay &gt; Py) != (By &gt; Py)</text>
  <text x="580" y="248" font-size="11" fill="#334155">• 交点 X &gt; Px 则翻转 inside = !inside</text>
  <text x="580" y="262" font-size="10.5" fill="#64748b">天然免疫水平边与顶点双计问题</text>
</svg>
</div>

#### 1. 线段与折线命中：向量正交投影与端点吸附（[`Geometry.h`](file:///E:/kiriyama_canvas/core/include/kiriyama/core/Geometry.h#L138-L152) & [`HitTest.cc`](file:///E:/kiriyama_canvas/core/src/HitTest.cc#L25-L75)）
给定鼠标点 $\mathbf{p}$ 与线段端点 $\mathbf{a}, \mathbf{b}$，令方向向量 $\mathbf{v} = \mathbf{b} - \mathbf{a}$。点 $\mathbf{p}$ 在直线 $\mathbf{a} + t\mathbf{v}$ 上的正交投影参数为：

$$
t^* = \frac{(\mathbf{p} - \mathbf{a}) \cdot \mathbf{v}}{\|\mathbf{v}\|^2}
$$

将 $t^*$ 钳制到闭区间 $[0, 1]$ 得到线段上的最近点 $\mathbf{q} = \mathbf{a} + \operatorname{saturate}(t^*)\,\mathbf{v}$：
- 当 $t^* < 0$ 时，最近点自动退化为左端点 $\mathbf{a}$；
- 当 $t^* > 1$ 时，最近点自动退化为右端点 $\mathbf{b}$；
- 当 $0 \le t^* \le 1$ 时，最近点即为正交垂足。

折线图 [`hitTestPolyline`](file:///E:/kiriyama_canvas/core/src/HitTest.cc#L25-L75) 遍历所有有效线段（自动跳过包含 `NaN` 的断连缺口），找出距离最小且满足 $\|\mathbf{p} - \mathbf{q}\| \le \tau$ 的线段后，再通过比较 $\|\mathbf{p} - \mathbf{a}\|^2$ 与 $\|\mathbf{p} - \mathbf{b}\|^2$ 将命中索引吸附到更近的数据点端点：

```cpp
// 文件：core/include/kiriyama/core/Geometry.h & core/src/HitTest.cc
inline Vec2d closestPointOnSegment(const Vec2d& point, const Vec2d& a, const Vec2d& b) noexcept {
    const Vec2d ab = b - a;
    const double lengthSquared = ab.lengthSquared();
    if (!(lengthSquared > 0.0)) return a;
    const double t = math::saturate((point - a).dot(ab) / lengthSquared);
    return a + ab * t;
}

std::size_t nearestEndpoint(const Vec2d& point, std::span<const Vec2d> points, std::size_t segIdx) noexcept {
    const double distA = (point - points[segIdx]).lengthSquared();
    const double distB = (point - points[segIdx + 1]).lengthSquared();
    return (distB < distA) ? (segIdx + 1) : segIdx;
}
```

#### 2. 饼图/甜甜圈扇区命中：极坐标判定、模 $2\pi$ 角度消除跳变与动态外扩补偿
扇环由圆心 $\mathbf{c}$、内半径 $R_{\text{in}}$、外半径 $R_{\text{out}}$、起始角 $\theta_{\text{start}}$ 与扫角 $\Delta\theta_{\text{sweep}}$ 定义。
相对于圆心的位移向量为 $\Delta\mathbf{p} = \mathbf{p} - \mathbf{c}$，极径 $r = \|\Delta\mathbf{p}\|$，极角 $\theta = \operatorname{atan2}(\Delta p_y, \Delta p_x) \in (-\pi, \pi]$。

由于 $\operatorname{atan2}$ 在 $\pm\pi$（即 $180^\circ$ 负 X 轴）处存在周期跳变分支切口（Branch Cut），直接比较 $\theta \in [\theta_{\text{start}}, \theta_{\text{start}} + \Delta\theta_{\text{sweep}}]$ 会在跨过 $\pm\pi$ 时失效。[`Geometry.h`](file:///E:/kiriyama_canvas/core/include/kiriyama/core/Geometry.h#L186-L244) 通过模 $2\pi$ 正向归一化函数 $\operatorname{wrap}_{[0, 2\pi)}(\alpha)$ 将相对角差映射到连续单调区间 $[0, 2\pi)$：

$$
\operatorname{wrap}_{[0, 2\pi)}(\theta - \theta_{\text{start}}) = \bigl((\theta - \theta_{\text{start}}) \bmod 2\pi + 2\pi\bigr) \bmod 2\pi \le \Delta\theta_{\text{sweep}} + \varepsilon
$$

```cpp
// 文件：core/include/kiriyama/core/Geometry.h
inline double normalizeAnglePositive(double angle) noexcept {
    const double wrapped = std::fmod(angle, math::kTwoPi);
    return wrapped < 0.0 ? wrapped + math::kTwoPi : wrapped;
}

inline bool ArcD::contains(const Vec2d& point, double radiusTolerance = 0.0) const noexcept {
    if (empty()) return false;
    const Vec2d delta = point - center;
    const double distance = delta.length();
    // 1. 极径夹逼：剔除甜甜圈中心圆孔 (distance < innerRadius) 与外部空白
    if (distance > outerRadius + radiusTolerance ||
        distance < std::max(0.0, innerRadius - radiusTolerance)) {
        return false;
    }
    // 2. 极角模 2π 折回判定
    if (distance <= 1e-9) return innerRadius <= radiusTolerance;
    return containsAngle(std::atan2(delta.y, delta.x));
}
```

**动画与命中的数学耦合（防边缘振荡）**：
在 [`PieChart::pickSector`](file:///E:/kiriyama_canvas/platform/src/src/PieChart.cc#L196-L219) 中，当扇区 $i$ 被悬停并向外膨胀（外半径增加 $d_{\text{explode}} \cdot p_i$，$p_i \in [0, 1]$）时，如果点击测试仍使用静态布局半径 $R_{\text{out}}$，鼠标停在膨胀后的扇区外缘就会判定“未命中”导致扇区缩回，下一帧又命中，造成高频鬼畜抽搐。因此 `pickSector` 对当前正在外扩的扇区动态补偿命中半径：

```cpp
// 文件：platform/src/src/PieChart.cc
std::optional<std::size_t> PieChart::pickSector(const Vec2d& worldPoint) const noexcept {
    const Vec2d local = toLocal(worldPoint);
    if (hoveredIndex_ && *hoveredIndex_ < explodeProgress_.size() &&
        explodeProgress_[*hoveredIndex_] > 0.0) {
        PieLayout expanded = layout_;
        // ★ 动态叠加当前帧的外扩动画位移，使膨胀出去的扇区边缘依然保持稳定命中
        expanded.outerRadius += explodeDistance_ * explodeProgress_[*hoveredIndex_];
        if (const auto hit = core::hitTestPie(local, expanded); hit.hit && hit.index == *hoveredIndex_) {
            return hit.index;
        }
    }
    const core::HitResult hit = core::hitTestPie(local, layout_);
    return hit.hit ? std::optional<std::size_t>(hit.index) : std::nullopt;
}
```

#### 3. 任意多边形命中：半开区间奇偶射线法（[`Geometry.cc`](file:///E:/kiriyama_canvas/core/src/Geometry.cc#L66-L89)）
从测试点 $\mathbf{p} = (p_x, p_y)$ 向 $+x$ 方向发射水平射线。对于多边形的每条有向边 $\mathbf{a} \to \mathbf{b}$：
1. **半开区间跨立判定 `(a.y > p.y) != (b.y > p.y)`**：等价于 $p_y \in [\min(a_y, b_y), \max(a_y, b_y))$（上开下闭）。这一个布尔异或同时解决了两大退化难题：
   - 若边是水平线（$a_y = b_y$），两侧同真或同假，异或为 `false`，直接忽略；
   - 若射线恰好穿过某个共享顶点 $V$，衔接 $V$ 的两条边中只有 $y > p_y$ 的那一端会计数一次，绝不会重复翻转。
2. **线性插值求交点横坐标**：

$$
x_{\text{intersect}} = a_x + \frac{p_y - a_y}{b_y - a_y}(b_x - a_x), \quad \text{若 } p_x < x_{\text{intersect}} \text{ 则翻转 } \text{inside} \leftarrow \neg\text{inside}
$$

#### 4. 场景树层级命中查询：AABB 剪枝与逆序 Z-Order 遍历（[`SceneQuery.cc`](file:///E:/kiriyama_canvas/core/src/SceneQuery.cc#L59-L88)）
渲染时按子节点正序 $0 \dots N-1$ 落笔（后画的盖在先画的上面），因此点击测试必须严格按**逆序 $N-1 \dots 0$ 深度优先遍历**：

```cpp
// 文件：core/src/SceneQuery.cc
const SceneNode* hitNode(const SceneNode& node, const Vec2d& worldPoint, double tolerance) noexcept {
    if (!node.visible() || node.style().opacity <= 0.0) return nullptr;

    // 1. 先求逆变换转入当前节点局部坐标，并用膨胀了 tolerance 的局部 AABB 粗筛
    const Vec2d local = node.toLocal(worldPoint);
    if (!node.bounds().isEmpty() && !node.bounds().expanded(tolerance).contains(local)) {
        // 若当前节点无自定义 bounds（如 Group 容器），继续向下递归；否则直接剪枝
    }

    // 2. 逆序遍历子节点（后绘制的顶层图元最先被测试）
    for (std::size_t i = node.childCount(); i > 0; --i) {
        if (const SceneNode* hit = hitNode(*node.childAt(i - 1), worldPoint, tolerance)) {
            return hit;
        }
    }

    // 3. 纯容器 Group 自身不产生命中，图元节点调用 containsLocal 精确几何判定
    if (node.type() == NodeType::Group) return nullptr;
    return node.containsLocal(local, tolerance) ? &node : nullptr;
}
```

#### 5. 虚拟滚动表格的 $O(1)$ 行列反查数学（[`TableLayout.cc`](file:///E:/kiriyama_canvas/core/src/TableLayout.cc#L56-L80)）
对于包含上万行的虚拟滚动表格，鼠标点击时**绝不遍历行节点**，而是利用表体偏移 $y_{\text{body}} = p_y - \text{bodyRect}.y$ 与滚动量 $\text{scrollOffset}$ 直接在 $O(1)$ 时间内算出逻辑行号：

$$
\text{row} = \left\lfloor \frac{(p_y - \text{bodyRect}.y) + \text{scrollOffset}}{\text{rowHeight}} \right\rfloor
$$

```cpp
// 文件：core/src/TableLayout.cc
std::optional<std::size_t> TableLayout::rowAtY(double y, std::size_t totalRows) const noexcept {
    if (rowHeight <= 0.0 || y < bodyRect.y || y >= bodyRect.bottom()) return std::nullopt;
    const double contentY = (y - bodyRect.y) + scrollOffset;
    if (contentY < 0.0) return std::nullopt;
    const auto row = static_cast<std::size_t>( std::floor(contentY / rowHeight));
    return row < totalRows ? std::optional<std::size_t>(row) : std::nullopt;
}
```

同时，键盘上下移动选中行时的自动跟随滚屏（[`TableLayout::scrollToRevealRow`](file:///E:/kiriyama_canvas/core/src/TableLayout.cc#L40-L54)）通过区间夹逼实现：设目标行在内容空间的上下边界为 $[y_{\text{top}}, y_{\text{bottom}}] = [r \cdot h, (r+1) \cdot h]$，视口高度为 $H_{\text{body}}$，则新的滚动偏移量为：

$$
\text{scroll}' = \operatorname{clamp}\!\left(
\begin{cases}
y_{\text{top}}, & \text{if } y_{\text{top}} < \text{scroll} \\
y_{\text{bottom}} - H_{\text{body}}, & \text{if } y_{\text{bottom}} > \text{scroll} + H_{\text{body}} \\
\text{scroll}, & \text{otherwise}
\end{cases}
,\; 0,\; \text{maxScroll}\right)
$$

---

### 3.3 交互模块的数学与状态机实现

#### 1. `Widget` 指针状态机、拖拽保持与防误触死区（[`Widget.cc`](file:///E:/kiriyama_canvas/platform/src/src/Widget.cc#L173-L210) & [`TableWidget.cc`](file:///E:/kiriyama_canvas/platform/src/src/TableWidget.cc#L183-L216)）
在每帧的 `Widget::tick(ctx)` 中，通过比较当前帧指针状态 $(p_t, \text{down}_t)$ 与上一帧快照 $(p_{t-1}, \text{down}_{t-1})$ 实现无事件队列的边沿检测：

$$
\begin{aligned}
\text{inside}_t &= \operatorname{hitShape}(\mathbf{p}_t) \\
\text{justPressed}_t &= \text{down}_t \land \neg\text{down}_{t-1} \\
\text{justReleased}_t &= \neg\text{down}_t \land \text{down}_{t-1} \\
\text{hovered}_t &= \text{inside}_t \lor (\text{pressed}_{t-1} \land \text{down}_t) \\
\text{clicked}_t &= \text{justReleased}_t \land \text{pressed}_{t-1} \land \text{inside}_t
\end{aligned}
$$

```cpp
// 文件：platform/src/src/Widget.cc
void Widget::tick(const FrameContext& ctx) {
    clicked_ = false; // 单帧脉冲复位
    const bool wasInside = inside_;
    inside_ = hitShape(ctx.pointer.position);

    const bool justPressed = ctx.pointer.down && !down_;
    const bool justReleased = !ctx.pointer.down && down_;
    down_ = ctx.pointer.down;

    if (inside_ && !wasInside) onPointerEnter();
    else if (!inside_ && wasInside) onPointerLeave();

    if (justPressed && inside_) {
        pressed_ = true;
        onPress();
    }
    if (justReleased && pressed_) {
        const bool releasedInside = inside_;
        pressed_ = false;
        clicked_ = releasedInside;
        onRelease(releasedInside);
        if (clicked_) {
            for (const auto& cb : clickBindings_) if (cb) cb();
        }
    }
    // ★ 捕获保持：按住左键拖拽滑出组件边界时，hovered_ 保持为 true，防止拖拽中途断触
    hovered_ = inside_ || (pressed_ && ctx.pointer.down);
    onUpdate(ctx);
}
```

**拖拽与点击的欧氏距离平方死区判别（Deadzone）**：
在 `TableWidget` 中，用户既可以按住鼠标上下拖拽滚动表体，也可以点击表头排序或点击行选中。为防止微小手抖将拖拽误判为点击，[`TableWidget::onRelease`](file:///E:/kiriyama_canvas/platform/src/src/TableWidget.cc#L196-L202) 采用 $6\text{ px}$ 的欧氏距离死区（直接比较平方距离避免 `std::sqrt` 开方）：

$$
\|\mathbf{p}_{\text{release}} - \mathbf{p}_{\text{press}}\|^2 = \Delta x^2 + \Delta y^2 > 36.0 \implies \text{判定为拖拽滚屏，抑制点击事件}
$$

#### 2. `InputController` 键盘修饰键掩码归一化与树状冒泡路由
[`InputController.cc`](file:///E:/kiriyama_canvas/platform/src/src/InputController.cc#L38-L133) 提供零 SDL 依赖的按键与滚轮绑定机制。为了使快捷键绑定（如 `Ctrl + Home`）具有确定性，匹配前先对修饰键位掩码做位与归一化（滤除 `CapsLock`/`NumLock` 等无关状态位）：

$$
\text{mod}_{\text{norm}} = \text{mod} \;\ \& \;\ (\text{Shift} \mid \text{Ctrl} \mid \text{Alt} \mid \text{Super})
$$

滚轮与按键沿 `SceneNode` 树按**逆序深度优先（顶层组件优先）**匹配位置并触发回调，一旦某个子组件返回 `true`（已消费），立即短路停止冒泡：

```cpp
// 文件：platform/src/src/InputController.cc
bool dispatchWheelToTree(core::SceneNode& node, const WheelEvent& event) {
    if (!node.visible()) return false;
    for (std::size_t i = node.childCount(); i > 0; --i) {
        if (dispatchWheelToTree(*node.childAt(i - 1), event)) return true; // 子节点已消费则短路
    }
    if (auto* widget = dynamic_cast<Widget*>(&node)) {
        if (widget->interactive() && widget->containsPoint(event.position)) {
            return widget->dispatchWheel(event);
        }
    }
    return false;
}
```

#### 3. 声明式交互样式合成：Porter-Duff Alpha 混合与中心锚点缩放矩阵（[`Interaction.cc`](file:///E:/kiriyama_canvas/platform/src/src/Interaction.cc)）
`Interaction.cc` 支持为任意 `SceneNode` 声明悬停（`hover`）与按下（`active`）时的视觉反馈覆写（颜色叠加、高亮描边、中心缩放），其底层包含两个核心数学实现：

**(1) Porter-Duff Source-Over 颜色叠加公式（`blendOver`）**
当交互状态为节点叠加一层半透明高光色 $C_{\text{top}}(\alpha_t)$ 在原填充色 $C_{\text{base}}(\alpha_b)$ 之上时，合成后的不透明度 $\alpha_{\text{out}}$ 与预乘还原后的 RGB 分量 $C_{\text{out}}$ 严格遵循 Porter-Duff `src-over` 算子：

$$
\alpha_{\text{out}} = \alpha_t + \alpha_b (1 - \alpha_t), \qquad
C_{\text{out}} = \frac{C_{\text{top}}\,\alpha_t + C_{\text{base}}\,\alpha_b (1 - \alpha_t)}{\alpha_{\text{out}}}
$$

```cpp
// 文件：platform/src/src/Interaction.cc
core::Color blendOver(const core::Color& base, const core::Color& top) noexcept {
    const double aTop = static_cast<double>(top.a) / 255.0;
    const double aBase = static_cast<double>(base.a) / 255.0;
    const double outA = aTop + aBase * (1.0 - aTop);
    if (outA <= 1e-9) return core::Color::Transparent();
    const auto mixChannel = [&](std::uint8_t b, std::uint8_t t) -> std::uint8_t {
        const double v = (static_cast<double>(t) * aTop +
                          static_cast<double>(b) * aBase * (1.0 - aTop)) / outA;
        return static_cast<std::uint8_t>(std::clamp(std::round(v), 0.0, 255.0));
    };
    return {mixChannel(base.r, top.r), mixChannel(base.g, top.g), mixChannel(base.b, top.b),
            static_cast<std::uint8_t>(std::clamp(std::round(outA * 255.0), 0.0, 255.0))};
}
```

**(2) 绕局部包围盒中心的复合仿射缩放矩阵（`interactionTransform`）**
当节点在悬停或按下时配置了缩放因子 $s \ne 1$（例如 `scale = 1.05` 放大 5%）与位移偏移 $\mathbf{t}_{\text{offset}}$ 时，若直接乘以缩放矩阵 $S(s, s)$，节点会以局部坐标系原点 $(0, 0)$ 为中心向外膨胀，导致图元视觉中心向右下角漂移。
正确的做法是以节点局部包围盒中心 $\mathbf{c} = \bigl(b_x + \frac{b_w}{2},\; b_y + \frac{b_h}{2}\bigr)$ 为不动点构造共轭变换：

$$
M_{\text{interact}} = M_{\text{base}} \cdot T(\mathbf{c} + \mathbf{t}_{\text{offset}}) \cdot S(s, s) \cdot T(-\mathbf{c})
$$

按列向量右乘顺序解读：先由 $T(-\mathbf{c})$ 将图元几何中心平移至原点，再由 $S(s, s)$ 原地均匀缩放，最后由 $T(\mathbf{c} + \mathbf{t}_{\text{offset}})$ 平移回原中心并叠加交互位移：

```cpp
// 文件：platform/src/src/Interaction.cc
math::Affine2D interactionTransform(const core::SceneNode& node, const math::Affine2D& base,
                                    double scale, const core::Vec2d& offset) noexcept {
    const core::RectD b = node.bounds();
    const core::Vec2d center = b.isEmpty()
                                   ? core::Vec2d{0.0, 0.0}
                                   : core::Vec2d{b.x + b.width * 0.5, b.y + b.height * 0.5};
    return base * math::Affine2D::Translate(center + offset) *
           math::Affine2D::Scale(scale, scale) *
           math::Affine2D::Translate({-center.x, -center.y});
}
```

#### 4. 二阶阻尼弹簧的闭式解析解（[`Spring.cc`](file:///E:/kiriyama_canvas/math/src/Spring.cc#L54-L105)）
物理交互动画若采用显式欧拉积分（$v \leftarrow v + a\Delta t$），在浏览器后台切回导致 $\Delta t$ 突增时会引发数值爆炸。`Spring.cc` 直接求解二阶齐次线性常微分方程：

$$
m\ddot{x} + c\dot{x} + k(x - x_{\text{target}}) = 0
$$

令初始位移偏差 $d_0 = x_0 - x_{\text{target}}$，固有角频率 $\omega_0 = \sqrt{k / m}$，阻尼比 $\zeta = \frac{c}{2\sqrt{km}}$，按 $\zeta$ 的三种形态直接代入时间步长 $\Delta t$ 的解析解 $x(\Delta t)$ 与导数 $v(\Delta t) = \dot{x}(\Delta t)$：

- **欠阻尼振荡（$\zeta < 1$，带丝滑回弹）**：令衰减率 $\alpha = \zeta\omega_0$，阻尼角频率 $\omega_d = \omega_0\sqrt{1 - \zeta^2}$，则：

$$
d(\Delta t) = e^{-\alpha \Delta t} \left( d_0 \cos(\omega_d \Delta t) + \frac{v_0 + \alpha d_0}{\omega_d} \sin(\omega_d \Delta t) \right)
$$

- **临界阻尼（$\zeta \approx 1$，最快无超调收敛）**：

$$
d(\Delta t) = \bigl(d_0 + (v_0 + \omega_0 d_0)\Delta t\bigr) e^{-\omega_0 \Delta t}
$$

由于解析解中包含指数衰减项 $e^{-\alpha \Delta t}$，即便页面挂起导致 $\Delta t = 10\text{ s}$，计算结果也精确收敛于 $0$，在任意帧率下具备无条件数值稳定性。

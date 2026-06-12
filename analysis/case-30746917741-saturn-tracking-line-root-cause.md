# 项目 30746917741 - 土星连线未跟随旋转锚点问题说明

## 背景

用户需求：

> Generate a space video showing Saturn rotating slowly using 02_education/05_planet_space.png. Starting at 00:05, add a white indicator line connecting Saturn to a fixed info panel in the top left corner. The panel should read "Saturn | Diameter: 116,460 km | Moons: 146". The starting point of the line must track Saturn's movement. The panel should fade in at 00:05 and fade out at 00:15.

原始素材：

- `/projects/30746917741/workspace/assets/a_TuRS3ht_13_05_planet_space.png`

本地 trace：

- `analysis/langfuse-data/cases/30746917741/trace-1-88bfdfc1.json`

## 现象

成片中信息面板和白色连线出现了，但连线并没有跟随土星旋转视频中的某一个具体点移动。

实际效果更接近：

- 使用静态土星图片作为全屏背景。
- 给整张背景图片套了旋转关键帧。
- 连线端点固定在土星中心附近。
- 因为旋转中心也设置在土星中心，所以线端点坐标看起来始终不需要变化。

这与用户期望的「连线起点跟随土星表面、环面或视频里某个运动锚点」不一致。

## 根因

### 1. 执行链把静态图合成误当成土星旋转视频

trace 中没有先生成一个真实的土星自转视频，而是直接用 HyperFrames/HTML 合成，把上传 PNG 当作背景图。

HTML 中的核心结构为：

```html
<div id="bg-wrapper">
  <img id="bg-img" src="..." alt="Saturn" />
</div>
```

随后用 GSAP 给背景包装层加旋转：

```js
tl.to("#bg-wrapper", {
  rotation: 360,
  duration: 20,
  ease: "none"
}, 0);
```

这意味着所谓「Saturn rotating slowly」并不是视频模型生成的土星自转，而是整张静态底图被二维旋转。

### 2. "track Saturn's movement" 被误解成跟随土星中心

模型在计划阶段写明：

- Saturn center in canvas: `x≈1237, y≈714`
- 旋转方式：让图片绕土星自身位置旋转。
- 因为旋转中心就是土星中心，所以土星中心在画布上保持固定。

随后连线端点也被写成固定坐标：

```html
<line
  id="connector-line"
  x1="522" y1="155"
  x2="1237" y2="714"
/>

<circle
  id="saturn-circle"
  cx="1237" cy="714" r="6"
/>
```

也就是说，系统把「跟踪土星运动」降级成了「连接土星中心」。由于中心点不移动，最终没有任何动态跟踪行为。

### 3. 缺少动态锚点定义和逐帧更新

用户说的是「starting point of the line must track Saturn's movement」，但执行时没有进一步明确：

- 线要跟随土星中心，还是表面某一点？
- 如果是表面点，初始位置在哪里？
- 如果土星旋转，该点的投影轨迹如何变化？
- SVG line 的 `x2/y2` 是否需要逐帧改变？

当前实现没有计算锚点轨迹，也没有更新 `x2/y2` 或 marker 的 `cx/cy`，因此无法满足「对应视频中的一点」。

## 正确方案

### 方案 A：先生成真实土星旋转视频，再叠加可追踪锚点

适用于用户明确要「底图土星旋转视频」的场景。

执行步骤：

1. 使用原图生成一个 camera locked 的土星缓慢自转视频。
2. 固定信息面板位置，保持面板在左上角。
3. 在合成层定义一个视觉锚点，例如土星表面某个亮带、环上一点、或用户指定的一点。
4. 在 `00:05` 到 `00:15` 期间，让连线端点逐帧跟随该锚点。
5. 抽帧检查 `t=5s`、`t=10s`、`t=15s`，确认端点位置发生变化且仍贴合目标点。

### 方案 B：仍使用 HTML 合成，但显式计算锚点轨迹

如果不调用视频模型，只做确定性动效，则不能只旋转背景。需要把锚点作为独立数学对象计算。

示例：

```js
const cx = 1237;
const cy = 714;
const rx = 150;
const ry = 65;
const startAngle = -0.6;
const duration = 20;

function updateConnectorAt(timeSeconds) {
  const theta = startAngle + (timeSeconds / duration) * Math.PI * 2;
  const x = cx + rx * Math.cos(theta);
  const y = cy + ry * Math.sin(theta);

  connectorLine.setAttribute("x2", x);
  connectorLine.setAttribute("y2", y);
  saturnCircle.setAttribute("cx", x);
  saturnCircle.setAttribute("cy", y);
}
```

在 HyperFrames 中，还应确保这个更新函数由可复现的时间轴驱动，而不是依赖 `setTimeout`、`Date.now()` 或浏览器实时状态。

### 方案 C：如果只想指向土星中心，应改写交付说明

如果产品允许把线接到中心点，那么交付说明必须明确写成：

> The connector points to Saturn's center, not a moving surface point.

但这不满足本项目用户说的「对应底图土星旋转视频中的一点」，因此不建议作为默认解释。

## 防复发建议

### Prompt/Skill 约束

对类似需求增加硬约束：

```text
Do not simulate Saturn rotation by rotating the full background image only.
The connector endpoint must attach to a visible moving anchor point on Saturn's surface or ring.
Compute or keyframe the endpoint coordinates over time.
The SVG line x2/y2 and marker cx/cy must change across frames while the info panel remains fixed.
Validate at 5s, 10s, and 15s that the endpoint is in different positions.
```

### 任务识别规则

当用户出现以下表达时，应识别为「动态锚点追踪」任务：

- 连线跟随某一点
- 指示线对应视频中的一点
- track movement
- attach to a moving point
- follow the object / surface / marker

这类任务不能只用固定坐标，也不能把整图旋转当作真实运动。

### QA 检查

渲染前后至少检查三帧：

- `t=5s`：面板和线刚出现，端点在锚点初始位置。
- `t=10s`：端点应明显不同于 `t=5s`。
- `t=15s`：端点应继续贴合锚点，并随面板一起淡出。

若三帧中 `line.x2/y2` 或 marker `cx/cy` 完全不变，则不能声称完成了动态跟踪。

## 结论

本项目的问题不是渲染器 bug，而是编排策略错误：

- 用户要的是「连线跟随土星旋转视频中的一点」。
- 实际实现是「静态 PNG 整图旋转 + 连线固定到土星中心」。
- 修复方向是使用真实旋转视频或显式锚点轨迹，并逐帧更新连线端点。

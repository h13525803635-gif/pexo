# 项目 09605057755：文字卡片边缘重叠问题根因与解决方案

## 1. 问题范围

- 项目 ID：`09605057755`
- 用户可见问题：同一画面内并排出现的多张文字卡片，其左右边缘互相覆盖。
- 本报告分析的是同页卡片的空间布局重叠，不是不同页面在转场期间的时间轴重叠。
- Langfuse 共发现 5 条 trace，成功获取 4 条；初次生成 trace `9e7ef257a9a7c2eea429eb7320050e4c` 因接口返回 HTTP 422 未能获取。
- 后续 trace 保留了 `composition.html` 的 CSS、卡片坐标、修改记录和 lint 结果，足以确认本问题根因。

## 2. 一句话根因

卡片使用默认的 `content-box` 盒模型：代码把卡片宽度写成 `480px`，又增加左右各 `32px` 的 padding，使实际外宽达到 `544px`；但相邻卡片的横向定位步长只有 `528px`，因此每对相邻卡片固定重叠 `16px`。

## 3. 直接证据

### 3.1 卡片样式

`composition.html` 中 `.kp-card` 的关键样式为：

```css
.kp-card {
  position: absolute;
  background: #FFFFFF;
  border-radius: 6px;
  box-shadow: 0 2px 8px rgba(0,0,0,0.08),
              0 8px 24px rgba(0,0,0,0.06);
  padding: 28px 32px;
}
```

代码没有设置 `box-sizing`，所以浏览器采用默认值 `content-box`。

### 3.2 卡片坐标

Beat 3、Beat 4 等三卡片页面复用了以下布局：

| 卡片 | left | width |
|---|---:|---:|
| card1 | 192px | 480px |
| card2 | 720px | 480px |
| card3 | 1248px | 480px |

相邻卡片的定位步长：

```text
720 - 192 = 528px
1248 - 720 = 528px
```

默认 `content-box` 下，每张卡片的实际外宽：

```text
480px 内容宽度 + 32px 左 padding + 32px 右 padding = 544px
```

因此每对相邻卡片的重叠量：

```text
544px 实际外宽 - 528px 定位步长 = 16px
```

这与“卡片边和边压在一起”的画面表现完全吻合。

## 4. 因果链

```text
使用绝对坐标手工排三列
-> 把 width 当成卡片最终外宽
-> 样式又增加横向 padding
-> 未设置 box-sizing: border-box
-> 浏览器在 width 之外追加 padding
-> 实际卡片宽度大于定位步长
-> 相邻卡片边缘固定重叠 16px
```

## 5. 为什么所有卡片页都会出现

这不是某一条文案过长造成的偶发现象。多个 Section 页面都复用：

- `.kp-card` 样式；
- `width: 480px`；
- `left: 192px / 720px / 1248px` 的三列坐标。

因此只要同页三张卡片同时出现，就会重复同样的 16px 几何重叠。文字长度可能影响卡片高度，但不是横向边缘重叠的根因。

## 6. 正确解决方案

### 推荐修复：统一使用 border-box

在 composition 的基础样式中加入：

```css
*, *::before, *::after {
  box-sizing: border-box;
}
```

或者至少对卡片组件加入：

```css
.kp-card,
.hook-card,
.takeaway-card {
  box-sizing: border-box;
}
```

修复后，`width: 480px` 会包含 padding，卡片的最终外宽保持 `480px`。相邻卡片的真实空隙为：

```text
528px 定位步长 - 480px 卡片外宽 = 48px
```

### 不推荐的临时修复

也可以把卡片内容宽度从 `480px` 改为 `416px`，使 `416 + 64 = 480px`，但这种写法依赖默认盒模型，后续调整 padding 时容易再次出错，不应作为组件级方案。

## 7. 应增加的防回归检查

### 静态规则

当绝对定位元素同时设置 `width` 和水平 `padding` 时，必须满足以下条件之一：

- 元素或全局样式设置了 `box-sizing: border-box`；
- 布局计算明确按实际外宽处理。

### 几何断言

对同一时间出现的相邻卡片，渲染前检查：

```text
card[i].right <= card[i + 1].left
```

本项目修复后的期望值：

```text
card1.right = 192 + 480 = 672
card2.left  = 720
gap         = 48px
```

### 视觉回归

在 Beat 3、Beat 4、Beat 5 各截取一帧，确认：

- 三张卡片之间均有至少 `48px` 的可见间距；
- 阴影不覆盖相邻卡片的主体边界；
- 最右卡片不超出 1920px 画布安全区；
- 不同长度文案不会改变卡片横向外宽。

## 8. 相关但独立的问题

追踪中还出现过空白页、黑屏、clip track 冲突和场景退场管理问题。这些属于时间轴、背景动画和 track 分配问题，与本报告的“同页卡片边缘重叠”不是同一个根因，不应通过修改 `data-start`、`data-duration` 或 GSAP 退场动画来解决本问题。

## 9. 证据来源

- Trace：`141e71028063d4cfc3bfb1545787b42f`
- 本地证据文件：`analysis/09605057755/trace-004-141e7102.json`
- 关键内容：`.kp-card` CSS、Beat 3/4 卡片的 `left` 与 `width`、composition lint 和后续编辑记录。

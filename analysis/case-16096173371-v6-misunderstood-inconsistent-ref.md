# 项目 16096173371：Bowser×Peach V6 — Request misunderstood / Inconsistent subject/reference

**Langfuse**：`pexo:16096173371`（34 traces，V6 关键 trace：`trace-17-8c9b8cc0.json`）  
**项目名称**：Bowser & Peach: Anime Kiss Meltdown Short  
**分析日期**：2026-05-19  
**V6 成片**：`bowser_peach_v6_sparkgrow.mp4`  
**QA 看板**：`analysis/langfuse-data/cases/16096173371/qa-report.html`

---

## 一、用户需求（验收标准）

### 1.1 初始 brief（trace-1）

15s 动画短片：Bowser 粉白礼服抱 Peach → 亲吻脸颊 → 卡通 meltdown（脸红、蒸汽、心形眼、胸口 WOW 心跳）。

用户上传参考图：

- `a_uJ7iUf3_1000042919.jpg`
- `a_ZT6erEj_1000042920.jpg`

### 1.2 迭代到 V6 前的关键指令

| Trace | 用户要求 |
|-------|----------|
| trace-11 | 亲吻导致 Bowser 肌肉生长 |
| trace-13 | 延长生长、**只撕袖子** |
| trace-15 | **像马里奥游戏里那样长大**（蘑菇 power-up） |
| trace-17 | **「Yes. Go that route」** — 确认用「亲吻星芒」替代蘑菇（绕过内容审核） |

### 1.3 V6 应满足的验收点

| 项 | 标准 |
|----|------|
| 生长语法 | 亲吻后 small↔big **快速闪烁**（马里奥蘑菇式），再 POP 到巨型 |
| 角色一致 | Bowser/Peach 与上传参考图一致，跨段不换脸 |
| 结构 | 亲吻 → 生长 → meltdown 叙事连贯 |
| 成片 | `bowser_peach_v6_sparkgrow.mp4` 无跨段画风断裂 |

---

## 二、实际交付（trace 事实）

### 2.1 V6 视频轨（trace-17 `execute_edit_video`）

| 顺序 | 素材 | 入点 | 来源 |
|------|------|------|------|
| 1 | `bowser_spark_grow_20260519T020326_950f8f43.mp4` | 0–7800ms | trace-17 **新生成** |
| 2 | `bowser_meltdown_reaction_20260519T002751_7a0c1929.mp4` | 0–7500ms | trace-1 **V1 未重生成** |

转场：500ms crossfade；音轨：BGM + 用户 SFX 包 `videoplayback.m4a` 多段切片。

### 2.2 V6 核心生成（`bowser_spark_grow`）

```
video_generate:
  provider=kling, model=kling-v3-omni
  mode=text2video          ← image_list 为空（全项目 18/18 皆如此）
  prompt≈1970 字符         ← 单条 10s 塞满多节拍
```

Prompt 要点（trace-17）：`Image 1` / `Image 2` 提供角色与风格 — **但 API 未传任何参考图**。

### 2.3 全项目生成统计

| 指标 | 值 |
|------|-----|
| `video_generate` 次数 | 18 |
| `image_list` 非空 | **0** |
| 模型 | 全部 `kling-v3-omni` |
| `image_generate` 主角设定图 | 0 |

### 2.4 跳过的产线

- 未调用 `subject-asset-skill`（无段级参考映射）
- 迭代只改 prompt 文本，不重绑 `image_list`
- V4/V5/V6 均复用 V1 `bowser_meltdown_reaction`

---

## 三、问题清单（现象 → 根因 → 解决办法）

### P0-1：Inconsistent subject/reference — 参考图从未挂载

| 项 | 内容 |
|----|------|
| **现象** | Bowser/Peach 跨段换脸、画风漂移；与上传图不一致 |
| **根因** | 18 次 `video_generate` 全部 `image_list: null`；prompt 写 `<1000042920.jpg>` 无效 |
| **解决办法（重做）** | 每次生成强制 `image_list` 含两张用户图；可选链式末帧 |
| **解决办法（skill）** | prompt 含 Reference image / `<*.jpg>` → `image_list` 非空，否则 block |

### P0-2：Inconsistent subject/reference — V6 混剪 V1 老片段

| 项 | 内容 |
|----|------|
| **现象** | 生长段与 meltdown 段角色明显不是同一人 |
| **根因** | V6 = 新 `bowser_spark_grow` + V1 `bowser_meltdown_reaction`；非同批次、同 refs |
| **解决办法（重做）** | 重做生长段时 **必须同 refs 重生成 meltdown** |
| **解决办法（skill）** | modification：换生长逻辑 → 禁止复用更早版本角色段 |

### P0-3：Request misunderstood — 单条 prompt 过载

| 项 | 内容 |
|----|------|
| **现象** | 马里奥式闪烁生长不稳定；星芒/撕裂/巨型化/Meltdown 元素混乱 |
| **根因** | `bowser_spark_grow` 一条 10s 调用含 0–3s 浪漫 + 3–5s 亲吻星芒 + 5–10s 14–16 次 flicker + 袖管撕裂 + POP |
| **解决办法（重做）** | 拆 3 段：`kiss`(~4s) → `mario_flicker_grow`(~6s，**仅**闪烁+POP) → `meltdown`(~5s) |
| **解决办法（skill）** | script-skill：复杂生长单独成镜；每段 prompt 一段一意 |

### P1-4：Request misunderstood — 需求链路偏移

| 项 | 内容 |
|----|------|
| **现象** | 用户要马里奥式生长；成片以「星芒触发」为主，闪烁感弱 |
| **根因** | trace-15 蘑菇方案积分失败 → trace-17 改星芒路线，但 prompt 仍混马里奥 flicker 语义 |
| **解决办法（重做）** | 星芒可作触发 **仅写在 kiss 段**；生长段只用 Mario small↔big 语法 |
| **解决办法（skill）** | 用户确认替代方案时，保留原动作语法（闪烁）与替代触发（星芒）分镜书写 |

### P1-5：未走 subject-asset

| 项 | 内容 |
|----|------|
| **现象** | 多版本迭代仍无法锁定主角 |
| **根因** | 读了 generation-skill，未读 subject-asset-skill |
| **解决办法（重做）** | 用两张上传图建 subject map → 每段 video_generate 挂同一组 refs |
| **解决办法（skill）** | 同 IP 多段 + 用户上传图 → 强制 subject-asset → generation |

---

## 四、推荐重做方案（V6 修复清单）

```
1. subject-asset：1000042919（角色）+ 1000042920（风格）→ 段级 map
2. video_generate ×3（均 image_list=上述两张）:
     bowser_kiss_spark      ~4s   亲吻 + 星芒（若保留该路线）
     bowser_mario_flicker   ~6s   仅 Mario small↔big 8–12 次 + POP + 袖裂
     bowser_meltdown_v6     ~5s   全新 meltdown（禁止用 V1 片段）
3. execute_edit_video → bowser_peach_v6_sparkgrow_v2
4. 自检：跨段人脸 / 服装 / 线稿一致；生长段可见完整 flicker-pop
```

---

## 五、关键 trace 索引

| 文件 | 内容 |
|------|------|
| `trace-1-43d96de7.json` | V1：`bowser_kiss_moment` + `bowser_meltdown_reaction` |
| `trace-15-cec62de9.json` | 马里奥蘑菇生长尝试（积分不足） |
| `trace-17-8c9b8cc0.json` | **V6**：`bowser_spark_grow` + 组装 `bowser_peach_v6_sparkgrow` |
| `assets-with-prompts.json` | 全项目 prompt + image_list_count |
| `qa-report.html` | 可视化看板（含参考图与关键视频预览） |

---

## 六、归因摘要

| QA 标签 | 主因 | 次因 |
|---------|------|------|
| **Inconsistent subject/reference** | `image_list` 全为空 | V6 新段 + V1 meltdown 混剪 |
| **Request misunderstood** | 单条 10s prompt 多节拍竞争 | 马里奥语法与星芒路线混写；未拆镜 |

**结论**：V6 生成 API 层面成功（`bowser_spark_grow` ok=true），失败在 **产线执行**（无参考绑定 + 旧段拼接 + prompt 过载），非模型单次报错。

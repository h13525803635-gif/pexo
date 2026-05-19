# Case 55850702914 — 15s 单条工序视频：木托未安装、电钻空转

**项目**: Girl's Solo Ceiling Drywall Trick  
**日期**: 2026-05-19  
**规格**: 9:16 · 15s · Seedance reference2video  
**用户反馈**: “It didn't show her installing the cut of wood to hold the drywall up. The drill clearly wasn't actually doing anything.”

---

## Session Overview

| Trace | 动作 | 结果 |
|-------|------|------|
| trace-3 | 用户上传 2 张自拍 + 描述 solo drywall 技巧 | 单次 `video_generate` 15s |
| trace-5/6 | 用户问能否做更长视频 | Agent 建议分段，未对 drywall 重做 |
| trace-7 | 同会话其他项目（Flock cameras） | 无关 |

**交付资产**: `drywall_ceiling_trick_20260519T071848_e42a9062.mp4`

---

## P1 — 未展示安装木托条 + 电钻假动作

### 现象
- 用户看不到「把裁切 2×4 拧到 stud 上」的安装过程
- 电钻像在空转，没有真实钻孔/拧螺丝的机械感

### 根因
1. **单条 15s 一镜到底**：4 个因果步骤（装木条 → 滑入石膏板 → 拧天花 → 收尾）压进一次 Seedance 调用；模型常跳过前段或直接进入结果态。
2. **参考图仅人物自拍**：`image_list` 只有 `847.jpg` / `848.jpg`，无施工现场、已安装木条、工具特写；reference2video 优先锁脸，不锁工序。
3. **Prompt 假设已完成状态**：`[4-8s]` 写 “wall-mounted 2x4 block”，但 `[0-4s]` 未强制可验证的「钻头接触 + 螺丝入木 + 木条从未固定到已固定」状态变化。
4. **跳过 script-skill / subject-asset-skill**：未分镜、未生成工序 keyframe、无生成后工序 QC。
5. **误用 generation-skill single-call preference**：教程类机械步骤应拆段，不应 15s 单条。

### 归因
- **Agent 流程**: 70%（策略与质检缺失）
- **模型能力**: 30%（工具接触、因果顺序本身难）

### 建议修复（本项目）
- 拆 4 段 `video_generate`（seg01 装木条 6–8s 特写优先）
- 先生成场景 +「木条已安装」控制图，再生成后续段
- seg01 通过后 QC：木条是否在墙上、钻头是否接触木材

### 建议 Skill 修改
- **generation-skill**: 工序/教程类例外于 single-call；扩展 post-gen sanity（工具接触、关键道具）
- **script-skill**: 含 install→use 因果链时，每关键步骤独立 sequence

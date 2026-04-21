# pexo

Internal Pexo tooling / reports；Pexo 相关案例分析材料与 Langfuse 拉取的 trace 数据（本地工作区备份）。

## Reports

- `reports/pexo_satisfaction_report_20260410_0842.html` — 成片后满意度（启发式），中国自然日 2026-04-07～04-09。

## 本地分析材料

- `analysis/langfuse-data/cases/` — 按项目 ID 存放的 trace JSON、assets 与部分下载媒体
- `case-*.md` — 案例说明与结论摘要；命名约定：**`case-<项目ID>-<讨论主题>.md`**（与 Langfuse / Pexo 项目 ID 对齐）

### 已有案例

| 文件 | 项目 ID | 主题 |
|------|--------|------|
| `case-97468242117-音频叠加与成片分析.md` | 97468242117 | Cheetah Airlines：co-gen 音频与 TTS 叠加根因及成片结构 |
| `case-43348924630-多角色一致性问题与解决方案.md` | 43348924630 | 冰雪奇缘生日视频：配角/道具一致性差的根因与修复方案 |
| `case-50814051508-narration-script-mismatch-causes-and-fixes.md` | 50814051508 | 旁白与脚本不匹配的原因与修复 |
| `analysis/audio-overlap-cases-summary.md` | 多案例 | 音频叠加专题汇总与整改 playbook |

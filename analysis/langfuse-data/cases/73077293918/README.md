# Case 73077293918 — 健身 App 自述片 · 音频重叠

## 快速链接

- **根因报告**：[../../73077293918_audio-overlap-root-cause.md](../../73077293918_audio-overlap-root-cause.md)
- **QA 看板**：[qa-report.html](./qa-report.html)（浏览器打开）
- **Langfuse trace ID**：`32880e9f60e59c043b4eb391a12db599`

## 目录说明

| 文件 | 说明 |
|------|------|
| `assets.json` | 工具调用提取的素材元数据 |
| `url_map.json` | OSS 签名 URL（会过期） |
| `media/` | 本地复制的片段媒体（无成片 final，未下载） |
| `trace-1-32880e9f.json` | 完整 Langfuse trace（~45MB，**默认不提交 Git**） |

## 核心结论（一句话）

**L2 英文重叠**：S3 手机 B-roll 内嵌 Seedance 英文 co-gen，合成时以 0.35 音量与中文 `vo_seg2_broll` 同播；Agent 将 96.6% 置信度 STT 误判为环境音杂音而未重生成。

# Case 90921184221 — 记账 App 自述片 · 画内音与旁白重叠

## 快速链接

- **根因报告**：[../../90921184221_audio-overlap-root-cause.md](../../90921184221_audio-overlap-root-cause.md)
- **Langfuse trace ID**：`a284c3a58f554ef70fc95bec3d6ebeac`

## 目录说明

| 文件 | 说明 |
|------|------|
| `assets.json` | 工具调用提取的素材元数据 |
| `url_map.json` | OSS 签名 URL（会过期） |
| `media/` | 本地复制的片段媒体与旁白/BGM |
| `trace-1-a284c3a5.json` | 完整 Langfuse trace（~25MB，**默认不提交 Git**） |

## 核心结论（一句话）

**Seq1 双轨重叠**：Seedance 在开场主镜头 co-gen 乱码中文人声，合成时以 0.1 音量保留画内音，同时叠 1.0 音量的后期 TTS 旁白，导致画内音与音频轨重叠。

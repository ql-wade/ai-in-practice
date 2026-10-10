# 本地 AI 视频复现条件与记录字段

[返回案例](README.md)

这份文档用于复现方法和验收过程，不承诺逐帧复现本次输出。任何读者都应使用自己有权使用的素材，并重新核对模型、节点、版本和硬件差异。

## 最低条件

- Windows 本地执行环境；本次观察使用 RTX 3090（24 GB 显存）和 64 GB 系统内存。
- 已安装并可运行的 ComfyUI，以及兼容的 Wan I2V 工作流。
- 本次观察使用 Wan2.2 A14B I2V Q5_K_M 双专家、UMT5 FP8、Wan2.1 VAE 和 CPU 卸载；版本或节点不同不能直接照搬参数。
- 有权使用的参考图；能保存并比较源文件和落地文件的 SHA-256、格式与像素尺寸。
- 足够的本地磁盘、电力、运行时间和一个可以检查音轨、字幕与 AI 标识的剪辑环境。

## 建议复现顺序

1. 先检查源图的授权范围、格式、像素尺寸、哈希和画面内容；公开人物或虚构角色的组合要明确是 AI 虚构，不写成真人/角色真实行为或代言。
2. 先跑约 3 秒短样片，检查首、中、尾帧和实际动作。确认问题后再决定是否扩展；不要默认末帧续接会保持身份。
3. 每次提交前记录 prompt ID、种子、模型/量化/版本、输入哈希、原生尺寸、帧数、FPS、状态和输出位置。
4. 按“铺垫 → 动作/冲突 → 结果”剪辑，添加字幕、声音和贯穿全片的 AI 虚构标识。
5. 对最终文件做技术、视觉、故事三层验收。音轨存在但电平偏低时，声音仍标为待修订；裁剪避脸时，身份问题仍然保留。
6. 保存实际输出参数。参考本次观察，可用 12.25 秒/196 帧/16 FPS 的原始记录和 9.625 秒/154 帧的剪辑记录做核对，但不要把这些数字当作所有环境的目标保证。

## 公开记录模板

```text
environment:
  os: <系统>
  gpu_vram_gb: <显存>
  system_ram_gb: <内存>
  comfyui: <版本或提交>
models:
  video: <Wan/量化/版本>
  text_encoder: <版本>
  vae: <版本>
  offload: <策略>
input:
  source_description: <非私有描述>
  sha256: <值或“未公开”>
  dimensions: <宽x高>
generation:
  prompt_id: <值或“未公开”>
  seed: <值或“未公开”>
  native_dimensions: <宽x高>
  frames: <帧数>
  fps: <FPS>
  status: <成功/失败/未知>
editing:
  duration_seconds: <秒数>
  audio: <编码、采样率、声道、电平/试听结果>
  subtitles: <语言、校对结果>
  ai_label: <首帧/转场/尾帧检查>
acceptance:
  technical: <通过/待修订/未验证>
  visual: <通过/待修订/未验证>
  story: <通过/待修订/未验证>
```

## 失败处理

源图哈希、尺寸或可读性不一致时，停止生成并只恢复传输；不要压缩截图、伪造元数据、绕过安全检查或用分块编码技巧。任务状态未知时先读取队列、历史和输出，再决定是否需要有界重试。Windows 文件保存中出现 `os.setxattr` 报错时，记录具体阶段；如同机已有 WSL 2 Ubuntu 22.04，可按官方未修改流程保留元数据后复制到目标盘，但不要将一次成功路径写成普遍规则。

## 来源

- [ComfyUI Wan2.2 教程](https://docs.comfy.org/tutorials/video/wan/wan2_2)
- [VideoX-Fun](https://github.com/aigc-apps/VideoX-Fun)
- [Phantom](https://github.com/Phantom-video/Phantom)
- [VACE](https://github.com/ali-vilab/VACE)

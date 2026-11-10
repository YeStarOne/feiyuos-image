# 工作流：节点表、接线、采样与调用

`workflows/coco_sfw_api.json` 是 **ComfyUI 的 API 格式**（`POST /prompt` 直接吃的那种），19 个节点，不含我们内部的两个节点。
它是"生产链减去两个不外发的节点"，其余与生产线逐字一致。

## 一、接线

```
4 CheckpointLoaderSimple ──model/clip──► 175 Expressive_H 1.0
                                        └─► 188 contrast −2.0
                                            └─► 189 brightness +1.0
                                                └─► 193 saturation −1.0
                                                    ├──model──► 3 / 17 / 176 / 142
                                                    └──clip───► 153 / 165 / 301 / 302

23 EmptyLatentImage (1024×1536)
   └─► 3 KSampler 30步 dn1.0 ─► 64 LatentUpscaleBy 1.1× ─► 17 KSampler 20步 dn0.55
                                                             └─► 118 VAEDecode
                                                                   └─► 176 FaceDetailer(脸, bbox=143)
                                                                         └─► 142 FaceDetailer(手, bbox=177, 正负=301/302)
                                                                               └─► 77 SaveImage
```

三条接线纪律：

1. **LoRA 是链，不是并联**：每条 `LoraLoader` 的 `model`/`clip` 都接上一条的输出。
2. **细节器吃的是二轮之后的图像**：`176.image ← 118`（VAEDecode），`142.image ← 176`。顺序反了手会把脸覆盖掉。
3. **手部细节器要独立提词**（301/302）：让它拿整条主正向，会把手按"脚"再画一遍（主正向里有脚部词时尤其明显）。

## 二、节点表

| 节点 | 类型 | 关键参数 |
| --- | --- | --- |
| 4 | CheckpointLoaderSimple | `cocoIllustriousNoobai_v55.safetensors` |
| 175 | LoraLoader | `Expressive_H-000001` / model **1.0** / clip 1.0 |
| 188 | LoraLoader | `add_contrast_XL` / model **−2.0** / clip 1.0 |
| 189 | LoraLoader | `add_brightness_XL` / model **+1.0** / clip 1.0 |
| 193 | LoraLoader | `add_saturation_XL` / model **−1.0** / clip 1.0 |
| 23 | EmptyLatentImage | 1024 × 1536 × 1 |
| 3 | KSampler | seed / **30 步** / cfg **5.0** / euler / karras / dn 1.0 |
| 64 | LatentUpscaleBy | nearest-exact / **1.1×** |
| 17 | KSampler | 同 seed / **20 步** / cfg **5.0** / euler / karras / dn **0.55** |
| 118 | VAEDecode | vae ← 4 |
| 176 | FaceDetailer（**脸**） | 20 步 / cfg **8.0** / dn **0.5** / cycle 1 / guide 512 / **bbox 143 = `face_yolov8m.pt`** / 正负 ← 165/153 |
| 142 | FaceDetailer（**手**） | 20 步 / cfg **7.0** / dn **0.55** / cycle 1 / guide 512 / **bbox 177 = `hand_yolov8s.pt`** / 正负 ← **301/302** |
| 301 / 302 | CLIPTextEncode | 手部专用正/负（主正向/负向 + 手部词） |
| 153 / 165 | CLIPTextEncode | 主负向 / 主正向 |
| 77 | SaveImage | `filename_prefix` = 批次/前缀 |

**seed 要写四个节点**：`3`、`17`、`176`、`142`。只改 `3` 的话，二轮和细节器还是旧 seed——
批量复现时这是最常见的"明明同 seed 却不一样"。

## 三、调用

```bash
curl -s -X POST http://127.0.0.1:8188/prompt \
  -H 'Content-Type: application/json' \
  -d "$(python3 -c 'import json;print(json.dumps({"prompt":json.load(open("workflows/coco_sfw_api.json",encoding="utf-8"))}))')"
# → {"prompt_id": "...", "number": 1, "node_errors": {}}
```

出图后查：

```bash
curl -s http://127.0.0.1:8188/history/<prompt_id> | python3 -m json.tool
```

要点：

- **400 时看 ComfyUI 自己的日志**（`Failed to validate prompt`），不要猜。最常见原因是模型文件名对不上或节点没装。
- 轮询 `/history/<id>`，出现 `outputs["77"]["images"]` 即成；超时按张数 × 单张耗时给（我们给 600 秒/张上限）。
- 一批出完、`/queue` 排空后 `POST /free {"unload_models": true, "free_memory": true}` 把显存还回去。

## 四、这份工作流与"我们线上跑的"差在哪

`workflows/coco_sfw_api.json` 的 165/153 文本 = **示例图当场的原文**（`examples/sample_01.json` 里可逐字对照）。
相对我们线上模块，它有三处差异——都是刻意的，写出来是因为"同 seed 重跑不一样"必须能解释：

| # | 差异 | 说明 |
| --- | --- | --- |
| 1 | 少了节点 **900 角色槽** | 位在链尾的 `LoraLoader`，命中角色时原位换 `lora_name`；挂的是我们自有/第三方角色 LoRA，不对外分发 |
| 2 | 少了节点 **192** | 链尾的一条细节 LoRA，这一版用于质感；与成人向内容强相关，本仓只做非 R18 |
| 3 | 正向**不含画风串** | 示例图那一臂就没带那 12 个 artist tag；线上模块的 `compose()` 会带。两条都在用、观感同档，账在 `docs/00-baseline.md` §七 |
| 4 | 检测器接线**已接正** | 上游模板把两个细节器接反了（手部提词挂人脸检测器），我们在这里改正；示例图是接反那版出的 |

所以：**用本仓工作流 + 同 seed，得到的是同风格的另一张，不是同一张。** 这一点写清楚，比假装能复现强。

要挂自己的角色 LoRA：在 193 之后插一个 `LoraLoader`，把 `3/17/176/142` 的 `model` 和 `153/165/301/302` 的 `clip`
接到它上面（**原位换名**也等价，而且更安全——见 `docs/00-baseline.md` §四）。

## 五、洗图与"原图不出目录"

ComfyUI 存出来的 PNG 里 `tEXt` 是**完整节点图**（正/负向全文、seed、LoRA 名）。所以模块的落盘规则是：

```
<batch>/raw/      ← 原图（带提示词），只本机，任何外传路径都读不到
<batch>/          ← 洗净版（像素不变，元数据全剥），对外只给这个路径
```

剥元数据实现要点（我们用的是纯像素级重存，不做有损转码）：

- PNG：按块过滤，只留 `IHDR/PLTE/IDAT/IEND/tRNS/iCCP/sRGB/gAMA`；
- JPEG：只留 `SOI` + 扫描数据 + `EOI`；
- 过闸三条断言：`im.text` 为空 · 与原图像素 `maxdiff == 0` · 读不到 `prompt` 字段；
- 反例造一张"带 tEXt 假 prompt"的 PNG 进闸，必须读不出来——**没有反例的闸不算闸**。

## 六、模块侧生命周期（`feiyu_image_mcp.py` 的做法，供照抄）

| 场景 | 行为 | 判据 |
| --- | --- | --- |
| ComfyUI 不在线，显存够 | 自己拉起 | `nvidia-smi` 剩余 ≥ **9 GiB**（SDXL 全链 8.1 GB + 余量） |
| ComfyUI 不在线，显存不够 | 拒单，不出图 | 判"宿主在跑大型游戏"，宁可这单失败 |
| ComfyUI 在线 | 直接投队列（插队） | 模型已常驻，零额外开销 |
| 正在推流 | **绝不拉起/重启** ComfyUI | 进程名含「哔哩哔哩」或 `livehime` 即视为在播（不要用 OBS 的 `streaming` 字段，实测不可靠） |
| 一批出完 | `/queue` 空则 `/free` 卸模 | 队列里还有别人的任务就不卸，只记账 |

这三条（显存闸 / 直播互锁 / 自动卸模）是把"生图"塞进一台**同时在打游戏**的机器里的全部代价。

# 模型与 LoRA：下载页、版本 id、sha256

> 本仓**不镜像权重**，只给定位信息。下面每一条都附 sha256——我们是拿 sha256 反查 Civitai 才确认"你们说的这个文件名到底是哪个模型"的，
> 你下载完也照样对一遍，能省掉一大类"照配方跑但出图不一样"的排查。

## 一、清单（按工作流里的文件名）

| 放哪 | 文件名（工作流按这个名字引用） | 大小 | sha256 | 来源 |
| --- | --- | --- | --- | --- |
| `ComfyUI/models/checkpoints/` | `cocoIllustriousNoobai_v55.safetensors` | 7.19 GB | `92d1b1084727c1912789002e5b6a9aac773e481231e09b5eb52bbeff5a5df33c` | Civitai model **2978673** / version **3376332** |
| `ComfyUI/models/loras/` | `Expressive_H-000001.safetensors` | 228 MB | `7b53da0391712dbe9412d5aa5fd074659413058773f72f41bd7a37e886031d43` | Civitai model **341353** / version **382152** |
| `ComfyUI/models/loras/` | `add_contrast_XL.safetensors` | 57 MB | `6cdc01750bae7c05d593c068d215d8e35eaf000f163a9e00c6fa77d2bb8eb596` | Civitai model **383129** / version **427622** |
| `ComfyUI/models/loras/` | `add_brightness_XL.safetensors` | 57 MB | `b6186f0dad9c393a5f0a29b8424255ebd0b33e51e260e107df2206d3f0a6d2ba` | Civitai model **383129** / version **427615** |
| `ComfyUI/models/loras/` | `add_saturation_XL.safetensors` | 57 MB | `ddc1017313df2b92a11680bf816846d6197815f75c6afd977cbbc3b06319f035` | Civitai model **383129** / version **427621** |

合计约 7.6 GB。

**下载地址**（Civitai 直链，需要登录态时请在浏览器打开对应 model 页）：

```
https://civitai.com/api/download/models/3376332     # 底模
https://civitai.com/api/download/models/382152      # Expressive_H
https://civitai.com/api/download/models/427622      # contrast
https://civitai.com/api/download/models/427615      # brightness
https://civitai.com/api/download/models/427621      # saturation
```

模型页：

- 底模 — https://civitai.com/models/2978673 （作者 `leole`；Civitai 上这条文件的原名是 `cocoStyleNoobaiXL_v55.safetensors`，我们本地改成了 `cocoIllustriousNoobai_v55.safetensors`，**两者字节相同**，sha256 一致。该版本 Civitai 的 `baseModel` 字段标为 `Anima`，我们按 SDXL 系工作流加载，实测正常。）
- 画风层 — https://civitai.com/models/341353 （`ExpressiveH`；**模型页带 NSFW 标签**，我们只用它的线条/笔触层，负向里焊死 `nsfw, suggestive`，出图不走成人向。若你不想用这条，可换任意 expressive 线稿风 LoRA，观感会跟着变——这是这条链里唯一"可替换但会改画风"的一环）
- 滑杆组 — https://civitai.com/models/383129 （`brightness contrast saturation sharpness ++ | Pony Tweaker Collection`，同系列还有 `add_sharpness_XL` / `add_outline_XL`，本基线不用）

## 二、FaceDetailer 用的检测模型

人脸/手细节器由 [ComfyUI-Impact-Pack](https://github.com/ltdrdata/ComfyUI-Impact-Pack) 提供，检测模型在工作流里按名字引用：

| 节点 | `model_name` | 用途 |
| --- | --- | --- |
| 143 | `bbox/face_yolov8m.pt` | 人脸框 |
| 177 | `bbox/hand_yolov8s.pt` | 手部框 |

Impact Pack 的 `ComfyUI/models/ultralytics/bbox/` 目录缺文件时会在首次运行时自动下载；装不上就手动从
[ultralytics assets](https://github.com/ultralytics/assets/releases) 取同名 `.pt` 放进去。

## 三、放好之后先验一遍

```bash
cd ComfyUI/models
sha256sum checkpoints/cocoIllustriousNoobai_v55.safetensors \
          loras/Expressive_H-000001.safetensors \
          loras/add_contrast_XL.safetensors \
          loras/add_brightness_XL.safetensors \
          loras/add_saturation_XL.safetensors
# 与上表逐行比对；Windows 用  certutil -hashfile <file> SHA256
```

对不上就说明你下的不是同一条文件——**不要**继续调参数，先换下来源。这一类错会伪装成"我的参数不对"。

## 四、显存与硬件

| 项 | 值 |
| --- | --- |
| 实测硬件 | NVIDIA RTX 5080 / 16 GB |
| SDXL 全链常驻 | 约 8.1 GB（FP16，`--force-fp16`） |
| 出图耗时 | 1024×1536 单张 + 1.1× 二轮 + 两个细节器，数量级是几十秒（随卡而定） |
| 更小的卡 | 降到 832×1216 或改用 fp8 底模；细节器 `guide_size` 从 512 往下调 |

ComfyUI 启动参数（我们用这一组）：

```bash
python -B main.py --listen 127.0.0.1 --port 8188 --force-fp16 \
  --disable-api-nodes --disable-comfy-compiler \
  --output-directory <你的输出目录，用正斜杠>
```

`--output-directory` 在 Windows 上务必用正斜杠，否则会静默落到默认目录，然后你会以为是"图没出来"。

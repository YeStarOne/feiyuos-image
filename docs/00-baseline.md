# 画风基线：一条短正向 + 平铺 tag 的 Illustrious 链

> 这条基线是我们 2026-11 反复 A/B 之后**定案**的那一套，不是拼参数拼出来的。
> 它的前身（900~1000 字加权正向 + 强对比 LoRA 链）跑了四轮 pilot、19 张正式图后被判废，
> 理由是同一句：**"画风越来越奇怪了"**。下面把两边的差别、判据、以及"为什么短反而更听话"写清楚。

## 一、这条基线是什么

**`cocoIllustriousNoobai_v55` 底模 + 4 条 LoRA 的固定链 + 极短正向 + 平铺 tag。**

画风串（12 个 artist tag）是**可选件**，不是命门：定案探针带了它，正式批量没带，两批图我们都收了。
这件事本身比任何参数都值得看——见 §七。

```
底模 ──► Expressive_H 1.0 ──► add_contrast_XL −2.0 ──► add_brightness_XL +1.0 ──► add_saturation_XL −1.0
                                                                                        │
                               ┌────────────────────────────────────────────────────────┘
                               ▼
                      KSampler 30 步 cfg5 euler/karras (1024×1536)
                               │
                        LatentUpscaleBy 1.1× (nearest-exact)
                               │
                      KSampler 20 步 cfg5 euler/karras denoise 0.55
                               │
                      VAEDecode ──► FaceDetailer(脸) ──► FaceDetailer(手) ──► SaveImage
```

### 工作流参数（一个都不建议单独改）

| 节点 | 内容 | 说明 |
| --- | --- | --- |
| ckpt | `cocoIllustriousNoobai_v55.safetensors` | Danbooru 2024-10 语料，tag 原生理解，2024-10 前的角色免 LoRA |
| 175 | `Expressive_H-000001` **1.0** | 线条/笔触层，去掉会掉一大截 |
| 188 | `add_contrast_XL` **−2.0** | **负值**是关键的：它压对比，让画面不死黑死白 |
| 189 | `add_brightness_XL` **+1.0** | 提亮；要压暗就只动这一个旋钮 |
| 193 | `add_saturation_XL` **−1.0** | 降饱和；颜色发灰先怀疑它 |
| 3 | KSampler **30 步 / cfg 5.0 / euler / karras** | 首轮 |
| 64 | LatentUpscaleBy **1.1×** nearest-exact | 1024×1536 → 1128×1688 |
| 17 | KSampler **20 步 / cfg 5.0 / denoise 0.55** | 二轮重绘（做爱/写真批用 0.4，见 §五） |
| 176 | FaceDetailer 脸：20 步 / cfg 8.0 / dn **0.5** | 接 301/302 提词 |
| 142 | FaceDetailer 手：20 步 / cfg 7.0 / dn **0.55** / cycle 1 | 同上 |

### 画风串（12 个 artist tag，可选件）

```
(artist:tianliang duohe fangdongye:0.9), artist:ciloranko, artist:noyu,
noyu23386566, ningen mame, (artist:sho:0.8264462809917354),
(sho lwlw:0.9090909090909091), (artist:kedama_milk:0.5),
(bee_\(deadflow\):0.7), artist:dino, (artist:fkey:0.8), (artist:wlop:1), year 2023,
```

这 12 个 tag 来自一条已验证成片的老配方，权重逐字抄来。**但它不是这套画风的主体**：
本仓的示例图（以及我们一批 66 张的正式图）**没有带这一段**，观感仍然成立（§七有全部门牌）。
带它的时候要注意 `artist:wlop`：Danbooru 里 artist tag 与"图上有署名"强相关，
权重 1.0 会在画面顶上写花体名字（见 §四），我们后来降到 0.7。

### 正向结构

```
HEAD        (solo:1.25), 1girl,
[ART]       上面那 12 个 tag（约 266 字，可选）
+ 角色锚     角色 tag + 头身锚（3~8 个，平铺）
+ 服装       6~8 个平铺词，零加权
+ 场景       一句话（4~8 个词）
```

两个实测长度（都落在 750 字闸内）：

| 臂 | 长度 | 谁在用 |
| --- | --- | --- |
| 带 ART | 590~690 字 | 1108 定案探针四臂；线上模块的拼词器 |
| **不带 ART** | **511~614 字** | 1110 正式批 66 张；**本仓示例图** |

示例图那一条的正向（`examples/sample_01.png`，511 字，**不含画风串**）：

```
(solo:1.25), 1girl, suzuran_\(arknights\), arknights \(game\), blonde hair, very long hair,
green eyes, fox ears, fox girl, fox tail, animal ear fluff, blue frilled headband,
white flower hair ornament, (lace choker:1.15), (large silver bell on her choker:1.3),
(light blue pajamas:1.25), (pj shirt with a white lace collar:1.2), (pj shorts:1.2),
long sleeves, white ankle socks, sitting on the bed, one hand on the lamp,
looking back at the viewer one last time, bedroom, bed, lamp light, night, soft warm glow
```

要挂 ART 就在 HEAD 之后插进去，其余不动；`examples/sample_01.json` 里两条文本都是原文，可以直接换。

### 负向（整段用，不要自己拼）

分三段串起来，任何一段都不要删：

1. **质量/解剖段**（`bad anatomy`、`worst quality`、`extra fingers`、`bad feet`、`score_1..4`、`too many fingers`…）
2. **非 R18 保险段**：`nsfw, suggestive, multiple girls, 2girls, 3girls, crowd, extra characters` ——
   **这一段是这条线"非成人向"的第一道保险，删了就不是同一套东西了**
3. **角色默认装负向段**：例如某角色自带白裙/束身/战术带，就把这些词按 1.2~1.3 压掉，
   否则不管正向怎么写都会长出默认装（见 §二）

## 二、为什么这套能成，而加权版不能

### 1. 正向必须**极短**，场景一句话

| | 废线（加权版） | 本基线 |
| --- | --- | --- |
| 正向长度 | 900~1000 字 | **511~690 字** |
| 光效写法 | `((dark room:1.3)), ((single light source:1.3)), ((backlighting:1.25)), ((rim light on her hair:1.4)), ((light bokeh:1.35)), bokeh, ((high contrast:1.15)), ((volumetric light:1.15)), (dust motes:1.05)` | **`night, dim lamp light`** |
| 服装写法 | `((cosplay:1.35)), (school uniform:1.3), (white shirt:1.2), (buttoned shirt:1.2), ((red neck ribbon:1.3)), (sailor collar:1.2), …` | **`white knit cardigan, white blouse, red ribbon, green pleated skirt, white pantyhose, mary janes`**（平铺，零加权） |

**根因**：正向词太多时，模型对每一条都只给一点点注意力；服装/光效这种需要"落到实处"的词被稀释掉，
于是**回退到角色 tag 自带的先验**（角色在原作里的默认服）。加更多服装词反而更糟——那是在用稀释去对抗稀释。
极短正向 + 平铺 tag 反而把服装钉住了。

### 2. 光效**不要写**

`night, dim lamp light` 四个词出来的就是"夜色 + 一盏灯 + 柔光斑"。
写 `volumetric light / dust motes / rim light:1.4 / high contrast` 那串，出的是**大面积死黑 + 大面积死白**的高斯对撞。

灯光交给人做什么？**交给场景词和底模**。带画风串的时候也是这一条——ART 管笔触与用色，不管打光；
指望 artist tag 去掉光效堆砌，是拿错了扳手（§七 有反面证据：不带 ART 的批同样干净）。

### 3. 判据用 **std**，不是 mean

| 图 | mean | std | dark(<40) 占比 |
| --- | --- | --- | --- |
| 目标参考 10 张 median | 63 | 47 | 0.49 |
| 本基线四条探针臂 | 137~149 | **47~55** | 0.00~0.02 |
| 废线 19 张 | 101~162 | **85~100** | 0.17~0.44 |

**std 是最好用的判据**：它直接反映"有没有死黑死白对撞"，比 mean 更能说明"像不像同一套画"。
一整套 50 张的批次，出完先看 std 的分布，再决定要不要开臂。

## 三、复用方法

要换内容，只换"角色锚 / 服装 / 场景"三段，`HEAD + ART + 负向` 一字不动：

```python
pos = HEAD + ART + CHAR + OUTFIT + scene      # scene 一句话
neg = NEG_BASE + NEG_TAIL                     # 顺序别改
```

三条硬约束：

1. **正向不要超过 750 字。** 超了就删场景修饰，**不要加加权**。
2. **服装/光效一律平铺，不加 `()`。** 要改服装只改词，不要加结构词
   （`cosplay` / `school uniform` 这类已验证会把默认装召回来）。
3. **双人拍换头部**：`solo` 与 `1boy` 同句会让模型拿第二个女方顶替男方。
   双人写成 `(1girl and 1boy:1.3), 1girl,`，并把 `solo` 从 HEAD 里拿掉。

## 四、已知副作用与坑

| 现象 | 根因 | 处方 |
| --- | --- | --- |
| 画面顶上出现 `DI N WLOP` 之类的签名/花体字 | `(artist:wlop:1)` 权重 1.0；Danbooru 里 artist tag 与"图上有署名"强相关，无权重负向打不过 | 负向提权 `(signature:1.5), (watermark:1.5), (artist name:1.4), (logo:1.4), (text:1.4)`；仍压不住就把 wlop 降到 0.7，最后才考虑摘掉 |
| 手上多出铁环/手链 | `artist:dino` / `wlop` 的配饰先验 | 负向 `(ring on hand:1.25), (metal bracelet:1.2)`；别改构图词 |
| 手长成脚（五指清晰、撑地） | 正向里"脚"的语义密度过高（`detailed feet / five toes / separated toes` 三连） | 删掉脚趾细节串，改成 `(barefoot:1.3), (no footwear:1.25)` + **一句纯构图描述**（`both bare feet are visible in the frame`）；**不要用 `toes` 类词补细节**，那会再次外溢到手 |
| 删了脚趾串却长出木屐 | `barefoot` 跟着掉权重 | 负向点名 `geta / wooden sandals / flip flops / slippers / tabi` |
| 角色 LoRA 换脸/换人 | 重接 LoRA 时下游 FaceDetailer 还拿着旧模型 | **原位换名**（同一节点改 `lora_name`），或显式把脸部细节器的模型输入也换过去 |
| 整批发灰 | `add_saturation_XL` −1.0 压色 | 单独一臂试 −0.5，**不要和压暗同时改** |
| **手部负向怎么改都没用**（指型、多余配饰压不住） | **两个细节器的检测器接反了**：拿手部提词的那个挂着人脸检测器 | 接正：先跑（dn 0.5）→ 脸检测器 + 主提词；后跑（dn 0.55）→ 手检测器 + 手部提词。我们的上游模板就是接反的，见 `examples/sample_01.json` 的 `wiring_fix` |

## 五、几条可复制的取值

- **写真/美图批**：节点 17 `denoise 0.55`，脸细节器 `0.5`，手细节器 `0.55`。
- **多体位批**：节点 17 `denoise 0.4`（重绘幅度小，保住姿势）。
- **要压暗**：只动 `add_brightness_XL`：+1.0 → 0.0 或 −0.5。**不要往正向加暗场描述词**。
- **要更"厚"的笔触**：`Expressive_H` 1.0 已经是上限；再高会把脸洗成风格脸。

## 六、这条基线推翻了什么（存档，别重犯）

| 假设 | 结论 |
| --- | --- |
| 湿身词导致发白 | **否**。删掉湿身词反而更亮（151 → 156） |
| 男方锚导致发白 | 部分。有作用，但解释不了 162 vs 63 的差距 |
| 画风链（对比/LoRA）导致发白 | 部分，但换了也到不了目标 |
| **正向过长导致角色先验回扑** | **是主因**。废线比对照多了 8 条 `cosplay/制服` 结构词，出图反而全变回角色默认服 |

**元教训**：风格照抄要抄**整套**——模型 + 链参数 + 正向写法 + 负向写法。
只抄一半再往另一半上打补丁，就是上面那条废线走过的路。

## 七、画风串：我们内部也没对齐的地方（如实公开）

这一节是给"想照抄画风串"的人泼冷水的，因为它是我们自己的账。

| 事实 | 证据 |
| --- | --- |
| 1108 定案探针四臂**带** ART（590~690 字） | 探针脚本 `pos = ART + head + CHAR + outfit + scene`；四臂 std 47.4~54.9 |
| 1110 正式批 66 张（含本仓示例图）**不带** ART（511~614 字） | 生成脚本头部写着"12 artist tag 不动"，但 `build()` 实际拼的是 `HEAD + CHAR + BELL + outfit + act + scene`——`ART` 是定义后**从未被使用**的变量 |
| 线上模块的拼词器**带** ART | `compose()` = `HEAD + ART + char_block + outfit + subject`，750 字闸下留给内容段的只剩约 484 字 |

所以：

1. **画风的主体来自底模 + 滑杆 LoRA 链 + 极短平铺写法，不是那 12 个 artist tag。**
   去掉 ART 的 66 张里，用户直接认可了示例图那一张；带 ART 的探针四臂也是同一套观感。
2. **我们没做过"同 seed 带/不带 ART"的对照**，所以这里不下"哪边更好"的结论——
   只把两条长度、两批图的出处摆出来。想抄的人自己开一臂，判据用 §二.3 的 std。
3. **别把 ART 当百搭补丁**：带上它就要同时管署名副作用（§四第一行），而且它占 266 字预算，
   会把服装/场景的空间挤到 484 字以内——这正是 §二.1 说的"稀释"来源之一。

一句话：**画风串是调味料，不是汤底。** 汤底是底模、滑杆链和"短、平铺、不写光效"这三条写法。

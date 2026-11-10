# 非 R18 提示词手册

> 这份手册只讲**非成人向**出图。它是这套链路实际在用的那份，不是通用教程：
> 每一节都对应模块里的一段代码或一次翻车记录。
>
> 适用对象：想让自己的 bot / 助手"说一句话就出图"，而且**必须保证出的是能直接发出去的图**的人。

## 一、先记三条，再看别的

1. **正向 ≤750 字，服装和光效平铺、不加权重。**（超过 750 字，模型开始稀释每一条，角色默认装就回来了）
2. **负向是整段用，不要自己拼。** 尤其是 `nsfw, suggestive` 那一段——它是这条线的非成人向保险。
3. **一个 prompt 只改一样东西。** 改完不满意，回退再改第二样；两样一起改，出问题分不清谁的锅。

## 二、六段式结构

模块拼出来的正向永远是这六段，顺序固定：

| # | 段 | 长度 | 例 |
| --- | --- | --- | --- |
| 1 | 头部 | 2 词 | `(solo:1.25), 1girl,` |
| 2 | 画风串 | 约 266 字 | 12 个 artist tag（见 `docs/00-baseline.md`），**不动** |
| 3 | 角色锚 | 3~8 词 | `suzuran_\(arknights\), arknights \(game\), fox ears, blonde hair,` |
| 4 | 服装 | 4~8 个平铺词 | `white blouse, green pleated skirt, white pantyhose, mary janes,` |
| 5 | 动作/构图 | 1 句 | `sitting on the bed, hugging her knees,` |
| 6 | 场景/光线 | 4~8 词 | `bedroom, night, dim lamp light,` |

**头部为什么带 `(solo:1.25)`**：单人构图钉死，能省掉后面所有 `multiple girls` 的对抗。
**双人则反过来**：去掉 `solo`，头写 `(1girl and 1boy:1.3), 1girl,`——`solo` 与 `1boy` 同席会让模型拿第二个女方顶替男方。

## 三、中文需求 → tag（她的大脑干这一步）

| 用户说 | 落成 tag |
| --- | --- |
| 白裙女孩在花园浇花 | `1girl, white dress, garden, holding watering can, full body,` |
| 她坐在床边抱着膝盖，夜里开着台灯 | `sitting on the bed, hugging her knees, bedroom, bed, night, dim lamp light,` |
| 站在雨里的便利店门口 | `standing, raining, convenience store, night, wet street, city lights,` |
| 穿校服在天台吃面包 | `school uniform, rooftop, eating bread, blue sky, daytime,` |

规则：

- **主体、服装、场景三样各写成一句，不要混在一句里。**
- **具体比形容词值钱**：`holding watering can` 比 `beautiful garden scene` 有用。
- **不要写"画得像某某画师"**：画师已在画风串里定了，再叠会打架。
- **不要写情绪形容词**（`beautiful`/`stunning`/`masterpiece`）——它们在 Illustrious 系里几乎不产生像素变化，
  只占长度预算。

## 四、长度闸：750 字

模块里是硬闸：正向超 750 字**不是报警，是拒单**。超了按这个顺序削：

1. 削场景修饰（第 6 段）→ 2. 削动作描述（第 5 段）→ 3. 削服装词（第 4 段，保留最显眼的 4 个）
→ **绝不削画风串**（第 2 段），也**绝不靠加权重去"抢注意力"**。

一件事记住：**加更多词不解决问题，那正是问题的来源。**

## 五、负向三段串

```
[1] 质量/解剖段
negative_hand, bad anatomy, blurry, (worst quality:1.8), low quality, bad hands,
[extra fingers], mutated hands and fingers, extra legs, extra arms, duplicate, cropped,
text, jpeg artifacts, signature, watermark, username, artist name, trademark, title,
multiple view, reference sheet, long body, disfigured, bad proportions, bad feet, ugly,
missing limb, monochrome, flat color, pale color, muted color, low contrast,
score_4, score_3, score_2, score_1, too many fingers, blurred eyes,

[2] 非成人向保险段（这一行删了，出的东西就不是能直接发的了）
nsfw, suggestive, multiple girls, 2girls, 3girls, crowd, extra characters,

[3] 角色默认装段（按角色写，1.2~1.3）
(white skirt:1.3), (dark bodysuit:1.3), (sailor collar:1.2), (tactical gear:1.2),
```

**署名/水印类**如果画面顶上冒花体字，把第 1 段这几条提权：
`(signature:1.5), (watermark:1.5), (artist name:1.4), (logo:1.4), (text:1.4)`。

## 六、拒单：这几类请求不出图

模块在拼词之前就拦，命中即返回理由、不占 GPU：

| 类别 | 判据 |
| --- | --- |
| 成人向 | 正向词表命中性行为/裸露类词，或负向保险段被要求删掉 |
| 真人 | 真人声优/偶像/演员/主播的名字，或"真人/本人/中之人/照片里的人"这类指示词 |
| 未成年化 | 把角色写成幼童并叠加任何性化描述——直接拒，没有"只是可爱"的例外 |
| 版权方明确禁止的商用形象 | 一律按用户自行负责处理，模块不做商用授权判断 |

这条闸的意义不是"管住用户"，是**让这个 bot 的图能不加检查地发出去**（家长群、工作群、公开直播间）。

## 七、洗图：为什么每张图都要剥元数据

ComfyUI 保存的 PNG 里，`tEXt` 块存着**完整节点图**——包括整段正向、负向、seed、LoRA 名。
换句话说：**原图 = 明文提示词**。发出去一张原图，等于把你的配方、你的系统提示词、甚至你的角色设定一起发了。

所以外传前必须过一道剥元数据的闸（像素不动，只删块）：

- PNG：只保留 `IHDR/PLTE/IDAT/IEND/tRNS/iCCP/sRGB/gAMA`；
- JPEG：只保留 `SOI + 扫描数据 + EOI`；
- 剥完做像素比对（maxdiff 必须为 0），并抽查读不到 `prompt` 字段；
- **原图落 `raw/`，洗净版才出目录；返回给调用方的路径只有洗净版。**

## 八、失败模式 → 处方（照表查，不要凭感觉改）

| 症状 | 先查 | 处方 |
| --- | --- | --- |
| 服装变回角色默认装 | 正向着装段是否有 `()` 或结构词（`cosplay`/`uniform`） | 删结构词、改平铺；正向砍到 700 字以内 |
| 画面死黑死白（看 std） | 是否写了 `rim light/high contrast/volumetric` | 全删，改成 `night, dim lamp light` 这类场景词 |
| 顶上签名（`DI N WLOP` 这类花体字） | `artist:wlop` / `artist:dino` 的署名先验；无权重负向打不过 | 负向提权名称类（`(signature:1.5)` 起），再把 wlop 从 1.0 降到 0.7；仍压不住才摘 |
| 手粘在奇怪位置、五指像脚 | 正向是否有 `five toes/separated toes/detailed feet` | 删三连，改 `barefoot/no footwear` + 一句构图 |
| 手部负向改了没反应 | 两个细节器的检测器是否接反（手部提词挂在人脸检测器上） | 见 `docs/03-workflow.md` §二：脸检测器 ↔ 主提词，手检测器 ↔ 手部提词 |
| 画面出现镣铐/手铐/脚镣这类束缚具 | 链上是否还挂着 R18 细节 LoRA（我们那次是 `realistic_cum_v1` @0.8）；负向只有泛词 | 把那条 LoRA 强度归零（节点别删，留着好回滚），再点名 `(shackles:1.35), (handcuffs:1.35), (leg irons:1.35), (ankle cuffs:1.35), (chains:1.3), (leash:1.3)`；**别写泛词 `collar`**，会把正向的蕾丝领一起打掉 |
| 脚出画 | 是否只有 `barefoot` 没有构图约束 | 补 `both bare feet are visible in the frame` |
| 脸不像角色 | 脸细节器 denoise 太高 | LoRA 角色把 `176.denoise` 降到 0.3 |
| 换角色 LoRA 后脸还是上一个人 | 是否新接了一条 LoRA | 原位换名，别另起一条 |
| 整批发灰 | `saturation −1.0` | 单独一臂试 −0.5 |
| 图被平台压缩得糊 | 发出的分辨率/质量 | 出图后转 JPEG q95（4:4:4）再发；别发 PNG 原图（体积大还会被二次压缩） |

## 九、复现与台账

每批落三样，缺一不可：

1. **seed**：同一 prompt 同 seed 才能复现；批量出图时 seed 写全（主采样、二轮、两个细节器各一个）。
2. **参数**：节点 17 的 `denoise`、脸/手细节器的 `denoise` 与 `cycle`。
3. **一段合并的正向/负向文本**（不是"改了哪几个词"）。

这就是 `examples/sample_01.json` 的样子——照它写，三个月后自己还能复现。

# 亚马逊电商产品视频：参考视频驱动生成方案调研

调研日期：2026-07-06

## 结论

现阶段没有找到一个成熟的开源项目，可以直接替代 ClipForge，做到“输入亚马逊商品链接 + 上传真实操作参考视频，然后稳定生成高质量电商带货视频”。

但这不等于开源路线不适合。更准确的判断是：

- 开源“成品应用”普遍还停留在 URL/图片/脚本到视频，和 ClipForge 的路线相近，不能解决复杂产品操作不准的问题。
- 开源“底层模型/工作流”已经有可用方向，尤其是 Video-As-Prompt、Wan/VACE、DiffSynth-Studio、ComfyUI 工作流，适合做参考视频控制实验。
- 对亚马逊产品视频来说，最靠谱的路线不是只喂商品链接，而是把粗糙实拍操作视频当作动作/结构控制输入，再用生成模型重拍成干净场景、好灯光、好节奏的最终视频。

优先级建议：

1. 短期先用 Pippit/Seedance 或 Runway 做商业验证，看参考视频驱动是否真的能还原产品动作。
2. 同时自建开源实验线：Video-As-Prompt 和 VACE/DiffSynth/ComfyUI。
3. ClipForge 保留为商品信息抓取、脚本、配音、字幕、合成、发布物料的壳，不再把它当作核心生视频控制系统。
4. 后续可做一个专门的 skill/agent，把“商品链接 + 产品图 + 操作参考视频 + 多候选质检 + ClipForge 合成”串起来。

## 关键判断：参考视频不是最终素材

这里说的真实拍摄视频，不是直接剪进最终成片。它的作用是：

- 给模型看正确的操作顺序，例如开盖、旋转、折叠、变形、安装、取放、喷洒、拆装。
- 提供手部与产品的相对位置、力的方向、运动节奏、状态变化。
- 最终画面仍由生视频模型生成，可以换成干净摄影棚、厨房、浴室、户外等商业场景。

因此，粗拍视频可以存在场景差、灯光差、节奏差的问题，只要“动作正确、产品状态变化清楚、遮挡尽量少”，它就有价值。

## ClipForge 的问题定位

ClipForge 当前路线更接近：

```text
商品链接/商品图
→ 抓标题、价格、描述、图片
→ LLM 生成卖点脚本和分镜
→ 商品图/AI 图/免费素材/i2v
→ FFmpeg 配音、字幕、BGM、贴片合成
```

它的商品抓取在 [src/lib/product-ingest.ts](../src/lib/product-ingest.ts) 中实现，优先级是：

```text
JSON-LD schema.org Product
→ OpenGraph / Twitter Card
→ title / meta description
```

[src/app/api/ingest/product/route.ts](../src/app/api/ingest/product/route.ts) 会抓 HTML、解析商品信息、最多下载前三张商品图，并保存 `shopUrl`。这对 Shopify/独立站较友好，但亚马逊、淘宝、拼多多等页面经常有反爬、动态渲染、区域化和登录问题，可能抓不全。

目前 ClipForge 代码里确实有 `referenceVideoUrl` 这个 provider 类型字段，fal.ai 和阿里百炼 provider 也会传 `video_url`。Atlas Cloud 模型列表里也登记了 `bytedance/seedance-2.0/reference-to-video`，描述为多模态参考图/视频/音频生成。但项目层面还没有“上传真实操作视频 → 自动切分动作 → 作为控制视频调用模型 → 多候选质检”的一等工作流。

这解释了实测效果差的根因：商品链接、文本和静态图只能告诉模型“这是什么”，很难告诉模型“它应该怎么被操作、怎么开、怎么变形、哪些部件不能乱动”。

## 需要提供什么

对复杂可操作产品，建议至少提供：

- 商品链接：Amazon/Shopify/独立站链接，用于标题、价格、卖点、素材图、发布链接。
- 商品清晰图：正面、侧面、细节、使用前状态、使用后状态；最好有白底图和场景图。
- 真实操作参考视频：每个动作 5-15 秒，静态机位优先，动作清楚优先，灯光和场景可以普通。
- 关键状态标注：例如 `0-2s 盖子关闭`、`2-4s 拇指按压卡扣`、`4-6s 盖子翻开到 110 度`。
- 品牌/场景约束：目标平台、风格、镜头比例、是否出现人脸、是否允许手部、禁用夸张功效词。
- API Key 或本地算力：LLM key 用于分析和脚本；生图/生视频 key 用于商业模型；开源模型需要 GPU 和模型权重。

## 商业平台核查

### Runway

Runway Aleph/Edit Studio 更适合“输入视频后变换/编辑/重拍”这类工作。它的强项是把已有视频变成新场景、新风格、新元素，适合验证“参考操作视频 → 商业化重拍”的思路。

不足：它不是电商 URL 到视频的完整自动化工具，商品信息抓取、脚本、带货结构、批量 SKU 管理需要外部系统补。

来源：

- https://runwayml.com/research/introducing-runway-aleph
- https://help.runwayml.com/hc/en-us/articles/51683104370451-Creating-with-Edit-Studio

### Pippit / Seedance

Pippit 官方页面显示 Video Agent 可以上传 link/media/file/reference，Seedance 页面也强调多参考图片/视频与视频编辑能力。它和你的想法最接近：用商品链接/素材/参考视频，通过 Seedance 类模型生成结果。

需要注意：官方表述更偏“模仿叙事结构、风格、节奏、参考视频”，不等于保证复杂产品机械动作逐帧准确。必须实测具体品类，例如开盖、折叠、安装、倒水、喷雾、收纳变形。

来源：

- https://www.pippit.ai/tools/video-agent
- https://www.pippit.ai/models/seedance-2-0
- https://www.pippit.ai/create/marketing-video-maker

### Creatify

Creatify 明确支持 URL-to-video、product video、automatic video editor。它适合“商品链接 → 广告视频”和“上传原始素材 → 自动剪辑”。

但没有找到官方明确说法，证明它支持“把真实操作视频作为运动/动作条件，重新生成一个干净商业视频”。因此它可以参考，但不是这个问题的第一优先级。

来源：

- https://creatify.ai/features/url-to-video
- https://creatify.ai/features/product-video
- https://creatify.ai/tool/automatic-video-editor

## 开源候选项目

### 1. Video-As-Prompt

项目：

- https://github.com/bytedance/Video-As-Prompt
- https://bytedance.github.io/Video-As-Prompt/

匹配度：高。

它的核心思路就是把 reference video 当作 video prompt，用参考视频里的语义/动作去驱动参考图生成新视频。README 中的核心描述是：给定带有所需语义的参考视频，去动画化一个参考图，使其具备参考视频的语义。它支持 CogVideoX-I2V-5B 和 Wan2.1-I2V-14B 两种变体，并提供 `ref_videos` 输入示例。

优点：

- 概念上最贴近“粗糙操作视频作为动作参考，最终重新生成”。
- 有代码、权重、训练脚本和 ComfyUI 社区实现。
- 不是只靠文本 prompt，比单纯 i2v 更可能学习动作节奏。

风险：

- 不是电商成品工具，没有商品抓取、带货脚本、剪辑合成。
- 对产品身份一致性、机械结构准确性仍需实测。
- Wan2.1 版本更大，显存/速度压力高；虽然 README 提到 offload 后显存可降，但速度会受影响。

适合实验：

- 输入产品干净图作为 reference image。
- 输入真实操作视频作为 reference video。
- 用 prompt 明确“只参考动作/手势/产品状态变化，不复制场景灯光人物”。

### 2. ComfyUI-Video-As-Prompt

项目：

- https://github.com/okdalto/ComfyUI-Video-As-Prompt

匹配度：高，但偏实验。

README 显示它是 Video-As-Prompt 的 ComfyUI custom node，可从 reference image 和 reference video frames 生成 frames，并有 `prompt_mot_ref` 这种运动/语义参考提示字段。

优点：

- 比纯 Python 更容易做视觉实验。
- 适合快速拖拽多组商品图和参考视频试错。

风险：

- 不是稳定产品化工具。
- 需要自己处理模型下载、显存、帧抽样、后处理、成片合成。

### 3. VACE / Wan2.1 VACE

项目：

- https://github.com/ali-vilab/VACE
- https://github.com/Wan-Video/Wan2.1

匹配度：高。

VACE 官方 README 写明它是 all-in-one video creation/editing 模型，覆盖 reference-to-video generation、video-to-video editing、masked video-to-video editing。它支持输入 text prompt，并可选输入 video、mask、image。推理前会预处理成 `src_video`、`src_mask`、`src_ref_images`，再进行 VACE 推理。

优点：

- 明确支持 R2V/V2V/MV2V，结构上适合参考视频、mask、产品参考图组合。
- Wan2.1 已有 VACE 权重和推理代码。
- 可以尝试 depth/control/mask/inpaint 等控制方式，比纯 prompt 更有约束力。

风险：

- 生成质量和稳定性仍取决于具体 workflow；GitHub issue 中也有人反馈 VACE 输出后段质量变差或控制理解不足。
- 不是亚马逊电商视频成品应用，需要自己接入 ClipForge 或其他合成系统。
- 14B 成本高，1.3B 质量可能不够。

适合实验：

- 用真实操作视频生成 control video/depth。
- 用商品图做 reference image。
- 对需要保真的区域做 mask 或反向约束。
- 每个动作单独生成 4-6 秒片段，再合成，而不是一次生成完整 30 秒视频。

### 4. DiffSynth-Studio

项目：

- https://github.com/modelscope/DiffSynth-Studio

匹配度：中高。

DiffSynth-Studio 是更偏工程框架的项目。README 中 WanVideo 部分列出了 VACE 模型支持，参数包括 `vace_control_video` 和 `vace_reference_image`，也提供低显存推理和训练示例。

优点：

- 比单模型 repo 更适合搭建可复现管线。
- 支持 Wan/VACE 多模型、训练、LoRA、低显存推理。
- 如果后续要为某类产品动作微调 LoRA，它比简单 UI 更合适。

风险：

- 工程复杂度高。
- 仍然不是电商应用，需要自己处理商品理解、脚本、QA、合成。

### 5. ComfyUI-WanVideoWrapper

项目：

- https://github.com/kijai/ComfyUI-WanVideoWrapper

匹配度：中高。

这个项目是 ComfyUI 的 WanVideo 相关 wrapper nodes。搜索结果和 issue 显示 ComfyUI 已经存在 Wan VACE reference-to-video 模板/工作流，说明社区确实在用它做参考视频控制实验。

优点：

- ComfyUI 生态适合快速试 prompt、mask、control、reference image/video 组合。
- 可以把产品图、参考视频、VACE/Wan 工作流快速连起来。

风险：

- 工作流和节点版本容易碎。
- 质量问题、路径问题、GGUF 输出异常等 issue 较多，需要工程耐心。
- 更适合研究和打样，不适合直接给运营人员稳定批量用。

### 6. viral2viral

项目：

- https://github.com/IuriiD/viral2viral

匹配度：中。

它允许上传参考 UGC 广告视频，AI 分析视觉风格、信息语气、节奏、互动技巧，然后结合产品信息生成新广告视频 prompt，再调用视频生成服务。

优点：

- 工作流比普通 URL-to-video 更接近“参考视频 → 新广告”。
- 对 UGC 广告风格复刻、节奏拆解有参考价值。

风险：

- 参考视频主要被分析成文本洞察/提示词，不是作为视频模型的控制输入。
- 对产品操作动作的帮助有限。

### 7. Open AI UGC

项目：

- https://github.com/Anil-matcha/Open-AI-UGC

匹配度：中低。

它是开源 UGC 广告 studio，支持脚本、最多 7 张参考图、多模型生成，包括 Veo、Seedance、Grok Video 等。它更像 Arcads/MakeUGC 的开源替代。

优点：

- 产品化程度比研究 repo 高。
- 支持 UGC 广告、参考图片、模型选择、dashboard。

风险：

- 没看到参考视频作为动作控制输入的核心工作流。
- 主要依赖 MuAPI，严格说不是本地开源模型方案。

### 8. OpenShorts

项目：

- https://github.com/mutonby/openshorts

匹配度：中低。

它支持描述产品或粘贴 URL 生成 UGC 营销视频，偏 AI actor、口播、B-roll、短视频发布工具。

优点：

- 自托管产品化程度较好。
- 适合 URL/描述到 UGC 口播类视频。

风险：

- 不解决产品机械操作参考问题。
- 更像 ClipForge 的另一种 UGC/shorts 形态。

### 9. ai-video-ad-generator

项目：

- https://github.com/theadtya/ai-video-ad-generator

匹配度：低。

README 明确是商品 URL 抓取、AI 脚本生成、视频合成。和 ClipForge 的思路非常接近，因此不能解决当前核心问题。

### 10. ViMax

项目：

- https://github.com/HKUDS/ViMax

匹配度：中低。

它是 agentic video generation，包含导演、编剧、制片、视频生成等角色，更适合长故事、多场景、角色一致性。不是专门针对电商产品操作参考视频。

## Skill 候选

### Generative-Media-Skills / Seedance 2 skill

项目：

- https://github.com/SamurAIGPT/Generative-Media-Skills
- https://github.com/SamurAIGPT/Generative-Media-Skills/blob/main/library/motion/seedance-2/SKILL.md

这个 skill 明确写了 Seedance 2 的多种模式：t2v、i2v、first-last、omni reference、video edit。它也给出了 `@video1` 这类参考语法，例如参考镜头运动、动作编排、节奏、音频等。

重要限制：

- 它依赖 MuAPI/Seedance，不是纯开源本地模型。
- 文档中写到 Chinese tier 的 omni 支持视频参考，Global/VIP omni 不支持视频 reference。
- 它是很好的提示词/调用 runbook，可借鉴，但不是完整电商产品视频系统。

### Fieldwork video-gen skill

项目：

- https://github.com/buildoak/fieldwork-skills

它的 video-gen skill 方向是多模型视频生成、结构化多场景生产、关键帧、状态追踪和装配。适合借鉴“agent 编排视频生产”的方法论，但不是专门的亚马逊产品操作视频方案。

### 建议自建 skill

可以在 ClipForge 旁边新增一个 `product-reference-video-director` skill，职责不是生成视频模型本身，而是编排：

```text
1. 抓商品链接和图片
2. 读取/切分真实操作参考视频
3. 让 VLM/LLM 输出动作时间线和产品状态机
4. 生成 VAP/VACE/Seedance 的分镜提示词
5. 每个镜头生成多个候选
6. 用视觉 QA 检查产品状态、动作是否正确
7. 通过 ClipForge 合成配音、字幕、BGM、贴片、发布文案
```

## 推荐实验路线

### 第一阶段：不要做完整 30 秒，先做 3 个动作镜头

选择一个有明确操作的产品，例如：

- 开盖/合盖
- 折叠/展开
- 安装/拆卸
- 按压出液/喷雾
- 变换形态

每个动作只做 4-6 秒，先验证动作能不能被控制，再谈带货脚本和成片。

### 第二阶段：并行测试三条生成路径

路径 A：Video-As-Prompt

```text
输入：产品干净图 + 真实操作视频片段 + 动作 prompt
目标：看它是否能把参考视频的动作迁移到产品图上
```

路径 B：VACE / DiffSynth

```text
输入：vace_control_video + vace_reference_image + mask + prompt
目标：看 control video 是否能稳定约束开盖/折叠/变形过程
```

路径 C：Seedance reference/video-edit API

```text
输入：产品图 + 操作参考视频 + 明确 @video1 动作/节奏/镜头角色
目标：验证商业模型的上限，作为开源路线对照组
```

### 第三阶段：加入 ClipForge 合成层

当单镜头动作可控后，再让 ClipForge 做：

- 商品抓取
- 卖点脚本
- 旁白 TTS
- 字幕
- BGM
- 商品卡/二维码/发布文案
- 多平台导出

ClipForge 不应负责“猜产品怎么操作”，它更适合做视频工程和电商发布层。

## 分镜和提示词模板

每个动作镜头建议这样建模：

```json
{
  "shot_id": "open_lid_01",
  "duration_sec": 5,
  "product_state_start": "lid fully closed, latch visible",
  "product_state_end": "lid opened to about 110 degrees, inner tray visible",
  "reference_video_role": "use only the hand motion, hinge rotation, action timing, and product state transition from @video1",
  "do_not_copy_from_reference": "do not copy background, lighting, camera noise, room, clothing, or low quality color",
  "scene": "clean bright kitchen countertop, soft commercial daylight, uncluttered background",
  "subject": "the exact product from @image1, preserve logo, proportions, hinge position, color, material, and lid geometry",
  "action": "a hand enters from frame right, thumb presses the latch, lid rotates open smoothly around the rear hinge, no deformation",
  "camera": "vertical 9:16, medium close-up, static tripod, 50mm product commercial look",
  "audio": "soft click sound when latch opens, subtle room tone, no speech in generated clip",
  "negative": "avoid warped product, wrong lid direction, extra hinges, extra fingers, melting plastic, unreadable logo, random text, scene cuts"
}
```

对 Seedance/omni reference 类模型，可以转成这种自然语言：

```text
Use @image1 as the exact product identity and product appearance.
Use @video1 only as the action choreography reference: hand position, latch press, hinge rotation, and timing.
Do not copy @video1's background, lighting, camera quality, clothing, or room.

0-2s: static medium close-up of the closed product on a clean kitchen countertop.
2-4s: a hand enters from the right, thumb presses the latch, the lid starts opening around the rear hinge.
4-5s: the lid reaches about 110 degrees and holds, inner tray visible, product geometry remains unchanged.

Commercial product video, vertical 9:16, soft daylight, stable tripod, realistic hand interaction.
Avoid: warped product, wrong mechanism, extra parts, extra fingers, logo changes, random text, abrupt cuts.
```

重点是不要只写“参考 @video1”。要明确参考视频里哪些东西要复制，哪些东西必须丢弃。

## 质检标准

每个候选视频至少检查：

- 产品外观是否和原图一致。
- 动作是否按参考视频发生。
- 产品状态是否正确变化，例如盖子真的打开、不是变形溶解。
- 手和产品是否物理接触合理。
- 镜头是否稳定、主体是否清楚。
- 是否出现多余文字、logo 变形、奇怪部件、手指错误。
- 是否适合亚马逊/TikTok/短视频平台发布。

如果一个模型一次只出一个候选，建议强制做 3-5 个 seed/候选，再自动/人工筛选。对复杂产品，一次生成命中正确动作的概率不会很高。

## 最终建议

不是“不要用开源项目”，而是不要期待现成开源 app 已经把这个问题解决好。当前更现实的路线是：

- 用商业平台验证天花板：Pippit/Seedance、Runway。
- 用开源模型搭自研控制线：Video-As-Prompt、VACE、DiffSynth、ComfyUI。
- 用 ClipForge 做电商生产壳：商品抓取、脚本、配音、字幕、合成、发布。
- 用自建 skill/agent 做编排：参考视频理解、镜头脚本、提示词、模型调用、多候选 QA。

如果要在这个仓库继续推进，下一步最有价值的改造不是继续优化普通脚本，而是新增：

```text
Reference Video Workflow
上传/保存操作参考视频
→ 切分动作片段
→ 生成动作状态机
→ 生成 VAP/VACE/Seedance prompts
→ 调用 reference/video-to-video 模型
→ 多候选质检
→ 回填到 ClipForge 的分镜素材池
```

这才是解决“视频不知道产品在展示什么”的关键。

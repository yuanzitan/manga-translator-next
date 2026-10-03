# 本仓库（fork）修改记录

上游基线：`hgmzhn/manga-translator-ui` `b02c72e12`（2026-10-03）。
本仓库（`yuanzitan/manga-translator-next`）在上述基线上追加以下 8 项改动，逐条列出改动点、以前/现在行为、影响面与已知限制。

## 1. 混合 OCR 独立备用概率 `secondary_prob`
- 文件：`manga_translator/config.py`（`OcrConfig.secondary_prob: float | None = None`）、`manga_translator/manga_translator.py`（`_resolve_secondary_ocr_prob_threshold`、`_filter_ocr_textlines(..., secondary_indices=, secondary_prob_threshold=)`、hybrid 分支 `config.ocr.model_copy(update={"prob": ...})`）、`desktop_qt_ui/core/config_models.py`、`desktop_qt_ui/ui/main_page/dynamic_settings.py`、`desktop_qt_ui/ui/main_page/settings_tab_layout.json`、`desktop_qt_ui/app_logic.py`、7 个 locale、`config/config-example.json`（写 `null`）。
- 以前：混合回退判据、备用引擎内部阈值、替换行最终过滤三者共用 `ocr.prob`。
- 现在：回退判据仍是 `ocr.prob`；备用引擎内部阈值与替换行最终过滤改用 `secondary_prob`，留空回退 `ocr.prob`。
- 影响：主"宽触发 + 备用严收"成为可能；旧配置行为不变。注意 VLM 类备用引擎固定返回 `0.9`（`model_paddleocr_vl.py`、`model_api_ocr.py`），`secondary_prob > 0.9` 会过滤掉全部备用结果；各引擎默认阈量纲不同（48px `0.2`、32px `0.7`、paddleocr `0.2`）。
- 文档/门禁：`doc/wiki/phase0-ui-parameter-fields.json`、`doc/wiki/verify_phase0_ui_parameter_fields.py`（layout 112 / visible 111 / delta -1，`optional-input` 豁免集合新增该键）、`doc/wiki/data/settings.generated.json`、`doc/wiki/data/i18n.generated.json`、`reference/settings-index.md`、`desktop/settings/ocr-filter-and-merge.md`、`debugging/ocr-and-text-regions.md`、`doc/SETTINGS.md`、`doc/en/SETTINGS.md`。

## 2. 长图检测条带最少重叠 200px
- 文件：`manga_translator/utils/generic.py`（`DET_REARRANGE_MIN_OVERLAP = 200`，`build_det_rearrange_plan` 起点重算）。
- 以前：`ph_num = ceil(h / ph)`，起点 `linspace(0, h-ph, ph_num)`，相邻重叠 = `ph - (h-ph)/(ph_num-1)`，无下限。
- 现在：`max_step = ph - min(200, ph - 1)` 反推片数，起点首尾贴边并均匀分布，仍有超限步长时增片（≤8 轮保护），`ph_step` 取相邻步长中位数。
- 影响：切缝至少 200px 重叠，跨缝文字行能被至少一条带完整覆盖；条带数与检测耗时随图长增加；`ph_step` 语义由等差变为中位，所有消费 `rel_step_list` 的逻辑按"起点表 + 中位步长"理解。
- 文档：`doc/wiki/zh|en/debugging/input-detection-and-rearrangement.md`。

## 3. 检测候选按分数截断 + 长图取消 1000 上限
- 文件：`manga_translator/detection/default_utils/dbnet_utils.py`、`manga_translator/detection/ctd_utils/utils/db_utils.py`（`_candidate_limit`、`_keep_top_by_score`）、`manga_translator/detection/default.py`、`manga_translator/detection/ctd.py`、`manga_translator/utils/ctd_replace.py`。
- 以前：`contours[:max_candidates]` 先按 `findContours` 顺序截断；`boxes_from_bitmap` 预分配零数组，跳过的行留占位零框；长图回拼后超过 1000 的轮廓被丢弃。
- 现在：遍历全部轮廓后在打分之后按分数降序保留上限数量（`None`/`≤0` 视为不限），零框填充消失；长图 `max_candidates=None`，单页仍 1000。
- 影响：单页排序口径变化 → 检测相关回归基线可能变化；长图不再因数量上限丢失尾部文本框；极端噪声蒙版会全量走 `approxPolyDP`/`unclip`/`box_score_fast`，CPU 与内存上升。`ctd.py`/`ctd_replace.py` 通过改写共享 `seg_rep.max_candidates` 切换，属隐式可变状态。
- 文档：同第 2 项页面。

## 4. 横排标点压半格 + 开括号贴右
- 文件：`manga_translator/rendering/text_render/_layout.py`（`_HORIZONTAL_HALF_ADVANCE_CHARS`、`_horizontal_punct_half_formats`、`_horizontal_open_bracket_shifts`、`_horizontal_glyph_path`）、`manga_translator/rendering/text_render/_fonts.py`（`_create_text_layout(..., formats=None)`）。
- 以前：只有"整块仅由 !?！？ 构成"时做半角字符替换（`_normalize_horizontal_block_content`），其余标点占全角推进，开括号居中于全角格。
- 现在：用 `QTextLayout.setFormats` 加负字距把推进压到 `0.5em`（字形保留全角），开括号再左移半格贴右；省略号/破折号排除，已窄于半格的不动。
- 影响：所有横排区域的宽度与行盒变化 → 渲染类 golden/期望需重建；断行宽度预算每个压缩标点少 `0.5em`；`emphasis` 逐字符区间与 `run.logical_width` 在压缩字符处存在 ≤`0.5em` 累计偏差。竖排不受影响。

## 5. 空段落行高改为 0
- 文件：`manga_translator/rendering/text_render/_layout.py`（`_finalize_rich_horizontal_line` 空行盒 `Bounds(0, 0, width, 0)`、新增 `_horizontal_line_occupies_slot`，`_build_rich_horizontal_layout` 跳过空行）。
- 以前：空行占整整一个 `font_size` 行高并参与行推进（见 `doc/DECISION_h_line_height_2026-07-13.md` 原文决策）。
- 现在：空行零高且不进 `layouts`。
- 影响：`layouts` 长度不再等于文本行数，任何按"行索引 ↔ 段落索引"对齐的逻辑（PSD 导出、调试、断行行号映射）需自行处理；尾部 `[BR]` 不再顶高正文。
- 文档：`doc/DECISION_h_line_height_2026-07-13.md`、`doc/REFACTOR_text_render_2026-07-13.md`、`desktop/settings/typesetting-and-rendering.md`、`manga_translator/rendering/RICH_TEXT_RENDERING.md`。

## 6. 横排纯标点行行高统一按字号
- 文件：`manga_translator/rendering/text_render/_layout.py`（`_is_horizontal_punctuation_text`、`_expand_bounds_to_font_size_height`）。
- 以前：纯标点行只有极小墨迹高度，行推进很紧。
- 现在：body/paint/spacing 三个 bounds 向上撑到 `font_size`，墨迹位置不变。
- 影响：含纯标点行的气泡测量高度变大 → 字号自适应可能变小、可容纳行数减少；判据按 Unicode 类别 P/S，装饰与 SFX 行（`…`、`〜`、`!!`、`★`）同样命中。

## 7. CRF pairwise 核归一化
- 文件：`manga_translator/mask_refinement/text_mask_utils.py`（主路径）、`manga_translator/utils/replace_translation.py`（替换翻译-直接粘贴路径）、`manga_translator/mode/__init__.py`（历史副本）。
- 以前：`NO_NORMALIZATION(0)` —— `unary_from_softmax` 后的前景能量优势极小，pairwise 又不归一化，CRF 退化为多数类平滑并抹掉前景。
- 现在：高斯与双边核统一 `getattr(dcrf, 'NORMALIZE_SYMMETRIC', 3)`（`NO=0/BEFORE=1/AFTER=2/SYMMETRIC=3`，取值与 pydensecrf 枚举一致）。
- 影响：蒙版精修不再丢前景 → 原文残留与修复漏擦减少；归一化后蒙版可能变厚，修复耗时/内存上升；蒙版变化会串到修复、排版可用区、替换翻译匹配与 `mask_is_refined` 产物。
- 说明：`manga_translator/mode/__init__.py` 是与 `text_mask_utils.py` 同源的历史副本，包外只导入 `mode.local/.ws/.share`，其 `refine_mask/complete_mask` 无调用方；该处修改仅用于一致性，不影响运行。

## 8. 增强原文 == 译文对比
- 文件：`manga_translator/manga_translator.py`（`_IDENTICAL_TRANSLATION_BREAK_RE`、`_normalize_for_identical_translation_compare`、`_should_filter_identical_translation`）。
- 以前：`str(region.text).lower().strip() == plain(region.translation).lower().strip()`。
- 现在：先去掉 `[BR]`/`【BR】`/`<br>` 与全部空白再小写比较。
- 影响：仅换行或空格不同的译文会被判定为"与原文相同"，在 `no_text_lang_skip` 逻辑下保留原文 → 渲染结果与导出 JSON 随之变化。

## 本仓库决策与边界
- `secondary_prob` 不作为发布默认值：`config/config-example.json` 写 `null`，语义等同"留空沿用 `ocr.prob`"。
- 第 5、6 项改写了上游 `doc/DECISION_h_line_height_2026-07-13.md` 的"空行占一个字号槽位"决策，作为本仓库自有取向。
- 本仓库只推送到 `yuanzitan/manga-translator-next`，不向上游提 PR；本机环境与个人配置（`.env`、`Miniconda3/`、`models/`、`result/`、`config/config*.json` 等）不入库。

## 9. 合并上游 PR #285（PaddleOCR 动态宽度输入）
- 来源：`https://github.com/hgmzhn/manga-translator-ui/pull/285`（作者 Wildare-98，head `6da57d1a7`，共 4 个提交）。
- 合并方式：`git merge` 到本仓库 `main`；PR 原始 base 是上游 `e6869343e`，合并时上游 `main`（`b02c72e12`）里该文件仍是改动前版本，说明上游尚未合入此 PR。
- 改动文件：`manga_translator/ocr/model_paddleocr.py`（+49 / -10），仅此一个文件。
- 以前：PaddleOCR 识别前把每个文本行统一 resize 到固定 `48×320`，长对话行被横向压缩、宽高比丢失，还会把不同宽度的行拼进同一批。
- 现在：宽度策略由 ONNX 输入是否为动态决定。动态宽度时按“宽度相近”分组批处理（`_iter_region_batches`），每批按该批最大所需宽度 pad（不小于 `320`，总宽度预算约 `16×320`），批内不再混入悬殊宽度；超长单行单独成批、不做压缩；固定宽度模型保持原行为。新增 `_preprocess_batch`，`_preprocess` 增加 `target_width` 参数。
- 影响面：仅 `ocr.ocr` 为 `paddleocr`/`paddleocr_korean` 等 PaddleOCR 引擎时生效；长行识别质量提升、批内 padding 减少；内存上限由宽度预算约束。其它 OCR 引擎、检测/蒙版/排版/替换翻译流水线不受影响。
- 备注：本次按需求未同步本地工作区，合并只落在 fork `main`；本地在测试更新脚本后拉取。

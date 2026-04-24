# 本地 AI 哲学 PDF 德译中系统：架构与 MVP 实施方案

## 0. 目标与范围

- 目标：构建一个**本地可部署**的 Web 系统，完成德语哲学 PDF 的解析、分段翻译、术语一致性管理、人工修订与导出。
- 核心用户：研究者/译者，处理康德、黑格尔、海德格尔等文本。
- 关键非功能需求：可追溯、可恢复、可扩展、多模型可切换、上下文连贯。

---

## 1. 推荐技术架构（可直接落地）

### 1.1 结论（推荐）

- 前端：**Next.js (React + TypeScript)**
- 后端：**FastAPI (Python 3.11+)**
- DB：**SQLite + SQLModel/SQLAlchemy + Alembic**
- 任务队列：**RQ + Redis**（MVP 可先 SQLite 轮询队列）
- PDF 解析：**PyMuPDF (fitz) 主，pdfplumber 辅**
- OCR：**PaddleOCR 或 Tesseract + pytesseract（扫描版可选）**
- 模型接入：统一 Provider 层（OpenAI/Claude/Gemini/DeepSeek/Kimi/Ollama）
- 文件存储：本地 `storage/` 分层目录

### 1.2 选择理由

1. FastAPI 生态对 PDF/NLP/队列友好，Python 库最全。
2. Next.js 在编辑、检索、双栏排版、导出交互上开发效率高。
3. SQLite 对单机场景足够且便于本地部署、备份。
4. Provider 统一接口可以快速换模型做质量/成本平衡。

---

## 2. 系统模块拆分

1. `ingest`：上传、文件校验、元数据提取
2. `pdf_parser`：页面抽取、段落切分、标题检测、脚注识别
3. `segmenter`：Book/Chapter/Segment 三级结构生成
4. `nlp_preprocess`：术语初提取、命名实体初提取
5. `translator`：上下文拼装、Token 预算、调用 AI Provider
6. `memory`：章节摘要/全书摘要更新
7. `glossary`：术语表锁定、冲突检测、版本化
8. `cache`：翻译缓存命中与失效
9. `jobs`：排队、重试、失败策略
10. `editor`：人工编辑、版本追踪、恢复初译
11. `exporter`：Markdown/JSON（MVP），DOCX/PDF（V2）
12. `viewer`：双栏对照、页码跳转、检索

---

## 3. 数据库设计（SQLite）

> 字段为建议最小集合；生产可加审计字段。

### 3.1 核心表

#### `books`
- `id` (pk)
- `title`
- `author`
- `source_lang` default `de`
- `target_lang` default `zh`
- `file_path`
- `status` (`uploaded|parsed|translating|done|failed`)
- `created_at`, `updated_at`

#### `chapters`
- `id` (pk)
- `book_id` (fk)
- `chapter_index`
- `title`
- `start_page`, `end_page`
- `detect_confidence`

#### `pages`
- `id` (pk)
- `book_id` (fk)
- `page_no`
- `raw_text`
- `clean_text`
- `layout_json`（可存 bbox、font size、line blocks）

#### `segments`
- `id` (pk)
- `book_id` (fk)
- `chapter_id` (nullable fk)
- `page_no`
- `seg_index_in_book`
- `seg_index_in_page`
- `source_text`
- `translation_text`
- `translation_status` (`pending|translating|completed|failed|edited|need_review`)
- `model_name`
- `prompt_version`
- `glossary_version`
- `memory_version`
- `cache_key`
- `created_at`, `updated_at`

#### `translation_edits`
- `id` (pk)
- `segment_id` (fk)
- `before_text`
- `after_text`
- `editor`
- `edit_reason`
- `created_at`

#### `glossary_terms`
- `id` (pk)
- `book_id` (fk)
- `term_de`
- `term_zh`
- `term_zh_alternatives` (json)
- `explanation`
- `first_segment_id`
- `occurrence_count`
- `locked` (bool)
- `updated_at`

#### `named_entities`
- `id` (pk)
- `book_id` (fk)
- `entity_type` (`person|place|book|school|concept|other`)
- `source_name`
- `target_name`
- `alias_json`
- `need_review` (bool)

#### `chapter_summaries`
- `id` (pk)
- `book_id` (fk)
- `chapter_id` (fk)
- `version`
- `summary_text`
- `based_on_segment_range`

#### `book_summaries`
- `id` (pk)
- `book_id` (fk)
- `version`
- `summary_text`

#### `translation_cache`
- `id` (pk)
- `cache_key` (unique)
- `book_id`
- `segment_id`
- `model_name`
- `prompt_version`
- `glossary_version`
- `memory_version`
- `source_hash`
- `result_json`
- `created_at`

#### `jobs`
- `id` (pk)
- `book_id`
- `segment_id`
- `job_type` (`translate|summarize|extract_terms`)
- `status` (`queued|running|retrying|done|failed`)
- `retry_count`
- `next_run_at`
- `last_error`

#### `api_logs`
- `id` (pk)
- `provider`
- `model`
- `endpoint`
- `request_tokens`
- `response_tokens`
- `latency_ms`
- `status_code`
- `error_msg`
- `created_at`

---

## 4. 页面结构设计（MVP + 升级）

### 4.1 MVP 页面

1. `/books`：书籍列表、状态、进入翻译
2. `/books/new`：上传 PDF，选择模型与基础配置
3. `/books/[id]/segments`：双栏对照主页面
   - 左：原文（按页/段）
   - 右：译文（可编辑）
   - 顶部：页码跳转、搜索框、导出按钮
4. `/books/[id]/glossary`：术语表管理（锁定/解锁）

### 4.2 V2 页面
- 命名实体确认页
- 质量评分页（术语一致性、漏译风险）
- 多模型对比页

---

## 5. API 接口设计（FastAPI）

### 5.1 上传与解析
- `POST /api/books/upload`：上传 PDF，创建书籍
- `POST /api/books/{book_id}/parse`：触发解析
- `GET /api/books/{book_id}/pages`：分页文本

### 5.2 段落与翻译
- `GET /api/books/{book_id}/segments?page_no=xx`
- `POST /api/books/{book_id}/translate/start`
- `POST /api/segments/{segment_id}/translate`（重试单段）
- `PATCH /api/segments/{segment_id}`（人工编辑）
- `POST /api/segments/{segment_id}/restore-ai`

### 5.3 术语与实体
- `GET /api/books/{book_id}/glossary`
- `POST /api/books/{book_id}/glossary`
- `PATCH /api/glossary/{term_id}`
- `GET /api/books/{book_id}/entities`

### 5.4 导出
- `GET /api/books/{book_id}/export?format=json|md|docx|pdf`

---

## 6. PDF 解析流程设计

1. 读取 PDF 元信息（标题、作者、页数）。
2. 页面级文本抽取（PyMuPDF），记录 blocks、字体大小、坐标。
3. 清洗：
   - 高频重复行（页眉/页脚）检测并剔除
   - 独立页码行剔除
   - 连字符断词修复（德语断词尤其重要）
4. 段落切分：按空行 + 标点 + 坐标间距融合规则。
5. 标题候选识别：大字号、序号模式（I., 1., §）+ 居中概率。
6. Chapter 粗识别失败时：以“页内自然段”落地，不阻塞翻译。
7. 扫描版：若文本密度低于阈值，进入 OCR 管线（可配置）。

---

## 7. 长文本翻译策略（你给出的方案落地）

### 7.1 上下文窗口组装

对当前 `segment_i` 组装：
- `prev_source`: i-2 ~ i-1 原文
- `prev_translation`: i-2 ~ i-1 译文（若存在）
- `current_source`: i
- `next_source`: i+1 原文
- `chapter_summary`
- `book_summary`
- `glossary_locked`
- `glossary_related`（按词命中筛选）

### 7.2 Token 预算算法

- 设模型上下文上限 `ctx_max`。
- 请求预算 `budget = floor(ctx_max * 0.65)`。
- 先放入必需项：`current_source + locked_terms + output_schema`。
- 依优先级追加：`prev` > `chapter_summary` > `next` > `book_summary` > `unlocked_terms`。
- 超限时压缩顺序：`book_summary -> chapter_summary -> next`，不删 current。

### 7.3 摘要更新

- 每完成 8 个 segment，触发章节摘要更新。
- 每章节结束，触发全书摘要更新。
- 摘要保留“论证推进 + 术语决议 + 人名映射”。

### 7.4 缓存键

```text
sha256(
  source_text + target_lang + model_name + prompt_version +
  glossary_version + memory_version
)
```

命中则直接写回 `segments.translation_text`，跳过 API。

---

## 8. 统一 AI Provider 设计

### 8.1 接口定义（Python）

```python
from abc import ABC, abstractmethod
from typing import Dict, Any

class TranslationProvider(ABC):
    @abstractmethod
    def translate(self, payload: Dict[str, Any]) -> Dict[str, Any]:
        """Return: {translation, terms_found, entities_found, need_review_items, usage}"""

class OpenAIProvider(TranslationProvider):
    ...

class ClaudeProvider(TranslationProvider):
    ...

class GeminiProvider(TranslationProvider):
    ...

class DeepSeekProvider(TranslationProvider):
    ...

class KimiProvider(TranslationProvider):
    ...

class OllamaProvider(TranslationProvider):
    ...
```

### 8.2 Provider 工厂

```python
def build_provider(name: str, config: dict) -> TranslationProvider:
    mapping = {
        "openai": OpenAIProvider,
        "claude": ClaudeProvider,
        "gemini": GeminiProvider,
        "deepseek": DeepSeekProvider,
        "kimi": KimiProvider,
        "ollama": OllamaProvider,
    }
    return mapping[name](config)
```

---

## 9. 翻译任务队列与失败重试

### 9.1 状态机

`pending -> translating -> completed`

异常分支：
- `translating -> retrying -> translating`
- 超过 3 次进入 `failed`
- 人工改后 `edited`
- 检测到术语/实体不确定 `need_review`

### 9.2 重试策略（实现细则）

1. 第 1 次失败：立即重试
2. 第 2 次失败：`+10s`
3. 第 3 次失败：切换备用模型（如 gpt-4.1-mini -> deepseek-chat）
4. Token 超限：自动减上下文重试
5. 429/限流：进入延迟队列并指数退避

---

## 10. 术语一致性方案

1. **锁定优先**：锁定术语强约束写入 prompt。
2. **新词提取**：每次翻译后让模型返回 `terms_found[]`。
3. **冲突检测**：若同一德文词出现不同中文译法，打 `need_review`。
4. **实体映射**：同一 source_name 强制 target_name 一致。
5. **术语版本化**：每次改动 `glossary_version += 1`，驱动缓存失效。

---

## 11. AI 翻译 Prompt 模板（可直接用）

> 建议使用 System + User 双消息；输出严格 JSON，便于解析。

### 11.1 System Prompt

```text
你是德语哲学文本翻译专家。任务是把德语准确翻译为中文。
硬性要求：
1) 仅翻译“当前段落”，不得翻译上下文段落。
2) 不总结、不删减、不扩写、不改写论证结构。
3) 保留原段落逻辑层次与语气。
4) 锁定术语必须严格遵守。
5) 若术语不确定，可在 uncertain_terms 中给出候选，不得擅自改锁定术语。
6) 输出必须是合法 JSON，不要输出 markdown。
```

### 11.2 User Prompt 模板

```json
{
  "task": "translate_de_to_zh_philosophy",
  "book": {
    "title": "{{book_title}}"
  },
  "location": {
    "chapter_title": "{{chapter_title}}",
    "page_no": {{page_no}},
    "segment_no": "{{segment_no}}"
  },
  "current_segment": {
    "source_de": "{{current_source}}"
  },
  "context": {
    "prev_source": ["{{prev_source_1}}", "{{prev_source_2}}"],
    "prev_translation": ["{{prev_zh_1}}", "{{prev_zh_2}}"],
    "next_source": ["{{next_source_1}}"],
    "chapter_summary": "{{chapter_summary}}",
    "book_summary": "{{book_summary}}"
  },
  "glossary": {
    "locked_terms": [
      {"de": "Begriff", "zh": "概念"},
      {"de": "Bewusstsein", "zh": "意识"}
    ],
    "candidate_terms": [
      {"de": "Vorstellung", "zh": "表象", "alt": ["观念"]}
    ],
    "pending_terms": ["{{pending_term_1}}"]
  },
  "output_schema": {
    "translation": "string",
    "notes": "string",
    "uncertain_terms": [
      {"de": "string", "candidates_zh": ["string"], "reason": "string"}
    ],
    "new_terms": [
      {"de": "string", "zh": "string", "alt": ["string"], "type": "concept|person|book|school|other"}
    ],
    "entities": [
      {"source": "string", "target": "string", "type": "person|place|book|school|concept|other", "need_review": true}
    ]
  }
}
```

### 11.3 期望输出 JSON

```json
{
  "translation": "...仅当前段落译文...",
  "notes": "如无可空字符串",
  "uncertain_terms": [],
  "new_terms": [],
  "entities": []
}
```

---

## 12. 核心代码骨架（MVP）

### 12.1 后端目录

```text
backend/
  app/
    api/
      books.py
      segments.py
      glossary.py
      export.py
    core/
      config.py
      db.py
      queue.py
    models/
      book.py
      chapter.py
      page.py
      segment.py
      glossary.py
      job.py
    services/
      pdf_parser.py
      segmenter.py
      translator/
        base.py
        factory.py
        openai_provider.py
        ollama_provider.py
      context_builder.py
      token_budget.py
      cache_service.py
      summary_service.py
    workers/
      translate_worker.py
    schemas/
      *.py
  alembic/
  requirements.txt
```

### 12.2 前端目录

```text
frontend/
  src/
    pages/
      books/
      book/[id]/segments.tsx
      book/[id]/glossary.tsx
    components/
      DualPaneViewer.tsx
      SegmentRow.tsx
      GlossaryTable.tsx
      ExportButton.tsx
    lib/
      api.ts
```

---

## 13. MVP 可运行实现计划（两周）

### Week 1

1. FastAPI + SQLite + Alembic 初始化
2. 上传 PDF + 页面文本抽取
3. 段落切分写入 `segments`
4. 单模型（OpenAI 或 Ollama）段落翻译
5. 基础缓存（translation_cache）

### Week 2

1. 双栏对照页面 + 编辑保存
2. 术语表 CRUD + 锁定
3. 重试队列 + 状态可视化
4. Markdown/JSON 导出
5. 最小监控（api_logs + job logs）

---

## 14. OCR 支持建议

- MVP：默认不启用 OCR，仅在文本层缺失时提示“该 PDF 可能为扫描件”。
- V2：增加 OCR 任务开关，推荐：
  - 中文输出质量优先：PaddleOCR
  - 部署简洁优先：Tesseract
- OCR 后仍需走段落重建与标题识别。

---

## 15. 本地部署方案

### 15.1 Docker Compose（推荐）

- `frontend` (Next.js)
- `backend` (FastAPI + Uvicorn)
- `redis` (可选，启用队列时)
- 挂载：`./storage:/app/storage`

### 15.2 环境变量（示例）

```env
APP_ENV=local
DATABASE_URL=sqlite:///./data/app.db
STORAGE_ROOT=./storage
DEFAULT_PROVIDER=openai
OPENAI_API_KEY=...
ANTHROPIC_API_KEY=...
GOOGLE_API_KEY=...
DEEPSEEK_API_KEY=...
KIMI_API_KEY=...
OLLAMA_BASE_URL=http://localhost:11434
```

---

## 16. 升级路线（完整版本）

1. 章节自动识别增强（目录 + 版式模型）
2. 自动术语抽取与冲突判定模型
3. 章节/全书摘要记忆完善
4. DOCX/PDF 高保真导出
5. 多模型投票与质量评分
6. 人工审校流（校对人、状态流转、审计）

---

## 17. 关键取舍说明

- **先做段落级精确翻译 + 术语锁定**，比“整页粗翻”更实用。
- **先保证可编辑与可恢复**，再追求自动化满配（OCR/章节识别）。
- **先单机 SQLite 跑通闭环**，未来再迁移 PostgreSQL/对象存储。

---

## 18. 你可以直接给 Cursor 的执行指令

请你基于以上需求，直接生成可运行的项目代码。优先实现 MVP，保证本地可以启动运行。请按文件路径逐个输出代码，并说明启动命令。

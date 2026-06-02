# API 設計

Route Handlers の仕様。全エンドポイント共通:

- ランタイム: `export const runtime = "nodejs"`
- 認証: なし（ローカル専用）
- 本文: JSON（chat のレスポンスのみストリーム）

エンドポイント一覧:

| メソッド | パス | 役割 |
|---|---|---|
| POST | `/api/ingest` | 原文保存 → 整理ノート生成 → 整理ノートを VS 登録 |
| POST | `/api/chat` | 整理ノート検索 → 回答（ストリーミング） |
| GET | `/api/newsletters/[id]/raw` | 原文ノートを返す（閲覧用） |

---

## 1. POST /api/ingest

### リクエスト

```jsonc
{
  "title": "週刊〇〇通信 第12号",   // 必須・1〜256文字
  "body": "〈本文テキスト〉",        // 必須・非空
  "issueDate": "2024-01-15",       // 必須・YYYY-MM-DD（実在日付）
  "issueNo": "第12号",             // 任意
  "author": "山田太郎",            // 任意
  "category": "マーケティング",      // 任意
  "tags": ["価格設定", "新規事業"],  // 任意・文字列配列
  "source": "〇〇社メルマガ"         // 任意
}
```

### バリデーション

| 項目 | ルール | 失敗 |
|---|---|---|
| `title` | 非空・≤256 | 400 `title_invalid` |
| `body` | 非空 | 400 `body_required` |
| `issueDate` | `YYYY-MM-DD` かつ実在日付 | 400 `issue_date_invalid` |
| `author`/`category`/`source` | 任意・≤256 | 400 `*_invalid` |
| `tags` | 任意・各≤64・最大20件 | 400 `tags_invalid` |

### 処理手順

```typescript
// 擬似コード（lib/ingest.ts が総合フロー）
const newsletterId = uuid();
const hash = sha256(body);
if (ledger.find(r => r.contentHash === hash)) return 409 "duplicate"; // 任意

// 1) 原文ノートをローカル保存
const rawMd = buildRawNote({ newsletterId, title, issueDate, issueNo, author, category, tags, source, body });
await writeFile(`data/raw/${newsletterId}.md`, rawMd);

// 2) 整理ノートを LLM で生成（lib/organize.ts）
const organizedBody = await organizeNewsletter({ title, issueDate, author, body }); // §prompt.md 抽出
const organizedMd = buildOrganizedNote({ newsletterId, title, issueDate, author, category, tags, source, organizedBody });
await writeFile(`data/organized/${newsletterId}.md`, organizedMd);

// 3) 整理ノートをアップロード → VS に属性付きで追加
const file = await openai.files.create({
  file: await toFile(Buffer.from(organizedMd, "utf-8"), `${issueDate}__${safeTitle(title)}.organized.md`),
  purpose: "assistants",
});
const attributes = {
  note_type: "organized",
  newsletter_id: newsletterId,
  issue_date: toUnixSeconds(issueDate),
  issue_date_str: issueDate,
  title: title.slice(0, 256),
  ...(author ? { author: author.slice(0,256) } : {}),
  ...(category ? { category: category.slice(0,256) } : {}),
  ...(tags?.length ? { tags: tags.join(",").slice(0,256) } : {}),
  ...(source ? { source: source.slice(0,256) } : {}),
  ...(issueNo ? { issue_no: issueNo.slice(0,256) } : {}),
};
const vsFile = await openai.vectorStores.files.createAndPoll(
  process.env.OPENAI_VECTOR_STORE_ID!, { file_id: file.id, attributes });
if (vsFile.status !== "completed") { await rollback(); return 500 "vector_store_failed"; }

// 4) 台帳追記（全成功時のみ）
appendLedger({ newsletterId, title, issueDate, issueDateUnix, issueNo, author, category, tags: tags ?? [],
  source, rawPath, organizedPath, organizedFileId: file.id, contentHash: hash, createdAt: now() });
```

> 失敗時のロールバック: 整理生成失敗→ raw 削除。VS 登録失敗→ uploaded file 削除 + raw/organized 削除。**台帳は全成功時のみ追記**。

### レスポンス

- `200`: `{ "ok": true, "newsletterId": "...", "title": "...", "issueDate": "2024-01-15" }`
- 失敗: `400 | 409 | 500` + `{ "ok": false, "error": "<code>", "message": "<日本語>" }`

---

## 2. POST /api/chat

整理ノートを検索して回答。原文は検索しない（出典リンクで閲覧）。

### リクエスト

```jsonc
{
  "messages": [ { "role": "user", "content": "新規事業の価格設定はどう考えるべき？" } ],
  "dateFrom": "2024-01-01",  // 任意
  "dateTo": "2024-12-31"     // 任意
}
```

### 処理手順

```typescript
// app/api/chat/route.ts
const instructions = buildAnswerPrompt();             // §prompt.md 回答
const filters = buildDateFilters(dateFrom, dateTo);    // 無指定なら undefined

const stream = await openai.responses.create({
  model: process.env.OPENAI_MODEL ?? "gpt-4.1",
  instructions,
  input: messages,
  tools: [{
    type: "file_search",
    vector_store_ids: [process.env.OPENAI_VECTOR_STORE_ID!],
    max_num_results: 8,
    ...(filters ? { filters } : {}),
    // 将来 raw 混在時: filters に {type:"eq",key:"note_type",value:"organized"} を AND
  }],
  stream: true,
});

const annotations = [];
for await (const event of stream) {
  if (event.type === "response.output_text.delta") writeText(event.delta);
  if (event.type === "response.output_text.annotation.added") annotations.push(event.annotation);
  if (event.type === "response.completed") break;
}
const sources = resolveSources(annotations); // file_id → 台帳 → {newsletterId,title,issueDate,author}
writeSources(sources);
```

### ストリーミング・プロトコル（サーバ→クライアント）

`Content-Type: text/plain; charset=utf-8` の `ReadableStream`。2部構成:

```
〈回答テキストが逐次流れる〉…
␞__SOURCES__␞
{"sources":[{"newsletterId":"7b9d...","title":"週刊〇〇通信 第12号","issueDate":"2024-01-15","author":"山田太郎"}]}
```

- 区切り: `␞__SOURCES__␞`（`␞`=Record Separator, U+241E は表記。実体は制御文字 U+001E）。本文に出現しない。
- クライアントは区切り前を本文、後の JSON を出典として描画。出典の各要素は `newsletterId` を持ち、原文閲覧に使う。
- エラー時は `␞__ERROR__␞` + `{"message":"..."}`。

> 出典は `newsletter_id` 単位で重複排除。

---

## 3. GET /api/newsletters/[id]/raw

出典リンクから原文ノートを取得（閲覧用）。

### リクエスト

`GET /api/newsletters/7b9d0c2e-..../raw`

### 処理

```typescript
const rec = findByNewsletterId(params.id);
if (!rec) return 404 "raw_not_found";
const md = await readFile(rec.rawPath, "utf-8");
const { meta, body } = parseFrontmatter(md);
return 200 { title: rec.title, issueDate: rec.issueDate, issueNo: rec.issueNo,
             author: rec.author, category: rec.category, tags: rec.tags, source: rec.source, body };
```

### レスポンス

- `200`: 原文メタ + `body`（原文本文）
- `404`: `{ "ok": false, "error": "raw_not_found" }`

---

## 4. 共通ヘルパ（`lib/`）

| 関数 | 所在 | 役割 |
|---|---|---|
| `getOpenAI()` | `lib/openai.ts` | クライアント生成 |
| `ingestNewsletter(input)` | `lib/ingest.ts` | 原文保存→整理生成→VS登録→台帳追記の総合 |
| `organizeNewsletter(input)` | `lib/organize.ts` | LLM で整理ノート本文を生成（非ストリーミング） |
| `buildRawNote()/buildOrganizedNote()` | `lib/*` | フロントマター付き Markdown 生成 |
| `buildDateFilters(from,to)` | `lib/vectorStore.ts` | 期間→`filters`（Unix秒、to は 23:59:59） |
| `resolveSources(annotations)` | `lib/ledger.ts` | file_id → 元メルマガ + newsletterId |
| `readLedger/appendLedger/findByFileId/findByNewsletterId` | `lib/ledger.ts` | 台帳 I/O |
| `buildAnswerPrompt()/buildExtractionPrompt()` | `lib/prompt.ts` | 回答/抽出プロンプト |

---

## 5. 注意点

- ingest は LLM 生成 + VS インデックス完了待ち（`createAndPoll`）を含むため、1件あたり数十秒かかりうる。ローカルなら許容。UI は「登録中…」を明示。
- `max_num_results` 初期値 8。回答品質を見て調整。
- annotations のスキーマは SDK バージョン差がありうるため、`file_id`/`filename` は存在チェックして堅牢に取り出す。

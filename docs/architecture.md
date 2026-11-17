# Arsitektur & Workflow n8n

Dokumen ini adalah panduan membangun dua workflow di n8n.
Prinsip: **dua workflow, satu vector store, satu model embedding yang sama persis.**

> Aturan nomor 1 embedding: model yang dipakai meng-embed dokumen (Workflow A)
> HARUS sama dengan model yang meng-embed pertanyaan (Workflow B).
> Beda model = vektor tidak sebanding = hasil pencarian sampah.

---

## Diagram alur (gaya n8n)

### Workflow A — Ingestion

```
[1 Manual Trigger]
   |
[2 Read Files from Disk]        folder: ./data/sop , ./data/faq  (dari .env: DATA_DIR)
   |
[3 Qdrant Vector Store]         mode: Insert Documents
   |-- sub-node: [3a Default Data Loader]   type: PDF / Text / CSV
   |-- sub-node: [3b Recursive Character Text Splitter]
   |-- sub-node: [3c Embeddings Ollama]     model: bge-m3
   |
[4 Postgres — ingestion_log]    file, jumlah chunk, timestamp, versi
```

### Workflow B — Chat Internal

```
[1 Chat Trigger]                aktifkan Basic Auth (user internal)
   |                            Fase 3: ganti dengan [Webhook] dari web Satuinbox
[2 Postgres Chat Memory]        session key: user internal id
   |
[3 LLM Chain / HTTP — Query Rewrite]   model kecil/utama, prompt: prompts/query-rewrite-prompt.md
   |                            output: query_bersih + intent + entitas (resi, kota, berat)
[4 AI Agent]                    type: Tools Agent
   |-- model:    [Chat Ollama]  qwen2.5:7b-instruct-q4_K_M
   |-- memory:   node [2]
   |-- tool 1:   [Qdrant Vector Store as Tool]  name: search_knowledge_base
   |-- tool 2..n:[MCP Client Tool]              endpoint dari .env: MCP_URL
   |               Fase 1–awal Fase 2: ganti sementara dengan mock tool (mock/)
   |
[5 IF — jawaban grounded?]      tidak ada sumber & tidak ada hasil tool -> template
   |                            "tidak ditemukan di SOP / data" + catat sebagai gap
[6 Postgres — chat_log]         pertanyaan, tool terpilih, sumber, status resolved/gap
   |
[7 Respond]                      jawaban + sitasi (file + bab/hal, atau nama tool + timestamp data)
```

---

## Pengaturan splitter (penting)

| Jenis dokumen | Strategi | Alasan |
|---|---|---|
| FAQ (Q&A) | 1 pasang Q&A = 1 chunk, jangan dipotong per token | Memotong Q&A di tengah merusak makna |
| SOP panjang (PDF) | Chunk 500–800 token, overlap 100–150 | Cukup konteks per chunk, overlap menjaga kalimat yang terpotong |
| Glosarium jargon internal | 1 istilah = 1 chunk | Membantu query rewrite memahami singkatan internal |

Metadata wajib per chunk: `source_file`, `category` (sop/faq/glosarium),
`section` (bab/judul), `page` (jika PDF), `version`, `updated_at`.
Metadata inilah yang ditampilkan sebagai sitasi di jawaban.

## Tool yang dimiliki Agent

| Tool | Sumber | Kapan dipakai |
|---|---|---|
| `search_knowledge_base` | Qdrant (RAG) | Pertanyaan SOP, FAQ, prosedur, kebijakan |
| `check_ongkir` | MCP (mock dulu) | Harga & estimasi pengiriman |
| `get_shipment_status` | MCP (mock dulu) | Status paket berdasarkan nomor resi |
| `get_order_detail` | MCP (mock dulu) | Detail order/pengiriman berdasarkan ID |

Kontrak lengkap + aturan keamanan tool: `mcp-tool-contract.md`.

## Keputusan retrieval (Fase 1)

- Top-K awal: **5** chunk.
- **Tanpa reranker dulu** — di CPU laptop, reranker menambah latency besar.
  Reranker (mis. bge-reranker-v2-m3) adalah opsi Fase 2/3 di server,
  diaktifkan hanya jika evaluasi test set menunjukkan recall kurang.
- Threshold: jika skor chunk terbaik di bawah ambang, Agent harus menjawab
  "tidak ditemukan" — bukan mengarang. Nilai ambang ditentukan dari test set (`eval/`).

## Format jawaban (diatur di prompt Agent)

1. Jawaban inti, singkat, bahasa internal (bukan gaya CS ke customer).
2. Baris sumber: `Sumber: <file>, <bab>, hal. <n>` atau `Sumber: tool <nama>, data per <timestamp>`.
3. Jika menggabungkan RAG + tool, pisahkan jelas mana yang dari dokumen, mana yang dari data live.

## Logging & loop perbaikan

Setiap pertanyaan yang **tidak terjawab** (gap) dicatat di `chat_log`.
Daftar gap = daftar kerjaan tim untuk melengkapi FAQ/SOP.
Untuk pemakaian internal, inilah metrik nilai utama sistem ini —
bukan sekadar "bot bisa menjawab".

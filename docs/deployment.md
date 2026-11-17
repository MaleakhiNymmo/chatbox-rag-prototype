# Deployment — Laptop Dev → Mac Mini / VPS

## Peran tiap mesin

| Mesin | Peran | Catatan |
|---|---|---|
| Laptop dev (Acer Aspire 5, i5, 16GB, Iris Xe) | Development & pengujian workflow | LLM 7B Q4 di CPU ±3–8 token/detik. Bukan untuk melayani user. |
| Mac Mini Apple Silicon (diusulkan 24GB unified memory) | Server inference internal | Model 7B–14B berjalan puluhan token/detik berkat unified memory + Metal. Listrik kecil, always-on di jaringan kantor, mudah mengakses MCP/database internal. |
| VPS | Alternatif server | Cek spek dulu: VPS CPU biasa ≈ kecepatan laptop. Data keluar ke provider = perlu izin, enkripsi, kontrol akses. Akses ke MCP/database internal perlu VPN — jangan expose MCP/DB ke internet. |

Ini deployment **inference + indexing**, bukan training/fine-tuning.

## Aturan portabilitas (dari hari pertama)

1. Semua alamat/kredensial dari `.env` (lihat `.env.example`): `OLLAMA_HOST`, `QDRANT_URL`,
   `POSTGRES_*`, `MCP_URL`, `DATA_DIR`. Tidak ada alamat yang di-hardcode di node n8n.
2. Versi ter-pin dan tertulis: model LLM, model embedding, dimensi vektor, ukuran chunk.
   Ganti model embedding = **re-indexing total wajib**.
3. Prompt disimpan sebagai file di `prompts/`, bukan hanya di dalam node —
   agar bisa di-review dan di-versioning.

## Catatan Ollama per platform

- **Windows/Linux (laptop dev):** Ollama install native, jalan sebagai service lokal.
- **macOS (Mac Mini):** Ollama **wajib native**, JANGAN di dalam Docker.
  Docker di Mac tidak dapat mengakses GPU/Metal Apple Silicon.
  Container n8n menjangkau Ollama host lewat `host.docker.internal:11434`
  (sudah disiapkan di `infra/docker-compose.yml` via `extra_hosts`).
- n8n, Postgres, Qdrant: di semua platform jalan di Docker lewat compose yang sama.

## Langkah migrasi (checklist)

- [ ] Siapkan `.env` di mesin target (nilai baru, jangan copy kredensial dev sembarangan)
- [ ] `docker compose -f infra/docker-compose.yml up -d` di mesin target
- [ ] Install Ollama native di mesin target, `ollama pull` model yang sama persis (nama + tag)
- [ ] Import export JSON workflow n8n dari folder `n8n/`
- [ ] Copy file SOP/FAQ sumber ke `DATA_DIR` mesin target
- [ ] **Jalanin ulang Workflow Ingestion di mesin target** — jangan copy volume Qdrant dari laptop.
      Re-indexing membuktikan pipeline reproducible.
- [ ] Sambungkan MCP asli (ganti mock), uji dengan test set yang sama (`eval/test-set.md`)
- [ ] Ganti Chat Trigger → Webhook untuk integrasi web Satuinbox
- [ ] Uji akses: hanya user internal, hanya jaringan internal/VPN
- [ ] Catat hasil uji & latency di mesin target sebagai baseline

## Perkiraan kebutuhan resource (mesin target)

| Komponen | RAM perkiraan |
|---|---|
| n8n + Postgres + Qdrant | 2–4 GB |
| bge-m3 (embedding) | ±1,1 GB |
| Qwen2.5-7B Q4 (LLM) | ±5–6 GB saat aktif |
| Qwen2.5-14B Q4 (opsi naik kelas) | ±9–10 GB saat aktif |

Total nyaman di unified memory/RAM 24GB. 16GB = minimum layak (LLM tetap 7B).

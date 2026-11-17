# Project Pendahulu — satuinbox_chatbot_ai

Repo: `MaleakhiNymmo/satuinbox_chatbot_ai` (private, status terakhir: *on-hold*, commit terakhir 2026-09-30).
Dokumen ini memetakan aset project lama yang relevan agar prototype ini
**tidak mengulang pekerjaan dari nol** dan tidak mengulang kesalahan yang sama.

## Apa isi project lama

Chatbot CS **untuk customer eksternal** Lincah, berjalan lokal:

- LLM: Llama 3.1 8B via Ollama, dibungkus Modelfile kustom `lincah-cs:8b-base`
  (persona CS ramah) + alternatif ringan Llama 3.2 3B.
- Orkestrasi: n8n, workflow import-ready `n8n_lincah_cs_workflow.json` (18 node):
  Webhook → Input Guardrails (Code) → Extract Resi & Intent (Code) →
  Fetch Context (HTTP ke API lokal) → Format Context Injection →
  Call Ollama → Respond, plus cabang logging komplain/kontak ke Google Sheets.
- API lokal `local_tracking_api.py`: REST API untuk cek resi (mock DB) dan pencarian FAQ.
- Knowledge di `data_rag/`: SOP & pedoman CS, kebijakan ekspedisi & SLA,
  bank FAQ, katalog resi & kasus tracking, FAQ format CSV/JSON siap indeks.
- Eksperimen RAG: `build_vector_index.py`, `test_hybrid_bge_m3.py` (hybrid bge-m3),
  `build_modular_graph_rag.py`, `build_master_faq.py`.
- Aset uji: `DAFTAR_RESI_TESTING.md`, `test_cs_with_mock_db.py`, `test_n8n_webhook.py`, log evaluasi.

## Pelajaran dari project lama (sudah terbayar mahal, jangan diulang)

Dari dokumen R&D dan log pengujian project lama:

1. **LLM tanpa koneksi data mengarang status resi** ("stuck di Jakarta Timur", dsb).
   → Di prototype ini: data live wajib dari tool, dilarang dari ingatan model.
2. **Tanpa filter domain, LLM terbajak pertanyaan di luar topik** (sempat menjawab soal coding).
   → Di prototype ini: aturan "tidak ditemukan / di luar cakupan" di prompt Agent.
3. **Keputusan project lama: 1 model utama (8B) + eskalasi manusia, BUKAN 2 model.**
   Alasannya: model swapping menimbulkan latensi muat ulang dan inkonsistensi persona.
   → Konsisten dengan desain prototype ini (1 LLM Agent).

## Pemetaan aset lama → prototype ini

| Aset lama | Dipakai sebagai | Catatan adaptasi |
|---|---|---|
| `n8n_lincah_cs_workflow.json` | Titik awal Workflow B | Ubah: user customer → internal (persona & sapaan), logging Google Sheets → Postgres lokal, Fetch Context HTTP → tool RAG (Qdrant) + MCP Client |
| `data_rag/*` (SOP, FAQ, SLA) | Kandidat dokumen Fase 1/2 menggantikan dummy | Rahasia: tetap di repo private lama / lokal, JANGAN disalin ke repo ini |
| `local_tracking_api.py` + `dummy_tracking_db.json` | Referensi bentuk API & data mock tracking | Di prototype ini perannya digantikan mock tool (`mock/`) lalu MCP asli |
| `DAFTAR_RESI_TESTING.md` + skrip test | Bahan test set evaluasi | Salin *skenarionya* ke `eval/test-set.md`, bukan datanya mentah-mentah |
| `test_hybrid_bge_m3.py` | Bukti pendukung pilihan embedding bge-m3 | Pilihan bge-m3 di prototype ini meneruskan hasil eksperimen tersebut |
| Modelfile `lincah-cs:8b-base` | Referensi persona & parameter | Persona lama = CS ke customer; persona prototype ini = internal. Prompt disimpan sebagai file & di n8n, BUKAN dibakar ke Modelfile, agar mudah di-versioning |

## Keputusan terbuka yang dipicu project lama

- **LLM: Llama 3.1 8B (lama) vs Qwen2.5-7B (blueprint ini).**
  Project lama membuktikan Llama 3.1 8B jalan untuk chat + context injection,
  tetapi belum terbukti untuk *tool calling* Agent (workflow lama memakai
  Code node + HTTP, bukan AI Agent). Keputusan final harus dari test set yang
  sama: uji keduanya pada `eval/test-set.md`, ukur ketepatan pemilihan tool
  dan kepatuhan format — bukan dari preferensi.

## Risiko yang ditemukan di project lama

- File `Lincah_CS_Message_Sep_2025-Sep_2026.jsonl` (±52 MB, riwayat pesan CS asli)
  ter-commit di repo lama. Repo memang private, tetapi riwayat chat customer
  adalah data sensitif: pertimbangkan memindahkannya keluar dari Git
  (dan membersihkan history) — minimal pastikan akses repo tidak dibagikan luas.
- Logging ke Google Sheets di workflow lama berarti data percakapan keluar
  ke layanan eksternal. Prototype ini sengaja memakai Postgres lokal.

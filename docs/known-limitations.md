# Keterbatasan & Blocker yang Diketahui

Diperbarui: 2026-11-17. Dokumen ini sengaja jujur — tujuannya agar siapa pun
yang meneruskan project ini tidak menebak-nebak statusnya.

## Blocker aktif

| # | Hal | Dampak | Yang dibutuhkan |
|---|-----|--------|-----------------|
| 1 | **MCP database belum diterima** (daftar tool, transport, auth belum diketahui) | Tool database belum bisa disambung; workflow memakai mock tool | Info MCP dari tim penyedia: transport/URL, hasil `tools/list`, auth |
| 2 | Platform chatbox.id sedang tidak bisa diakses untuk project ini | Prototype dibangun di n8n lokal sebagai jalur independen | Akses platform pulih, ATAU keputusan lanjut penuh di n8n |
| 3 | Spek mesin deployment final belum diputuskan (VPS vs Mac Mini, spek berapa) | Pilihan ukuran LLM final belum bisa dikunci | Keputusan + spek mesin target |

## Keterbatasan desain saat ini

- **Latency di laptop dev lambat** (LLM CPU-only, ±3–8 token/detik untuk 7B Q4).
  Angka ini BUKAN representasi performa di mesin target.
- **Belum ada reranker.** Retrieval = Top-K langsung. Dievaluasi ulang setelah
  test set menunjukkan kualitas recall-nya.
- **Test set masih contoh/template** (`eval/test-set.md`) — pertanyaan uji asli
  dari tim internal belum dikumpulkan. Tanpa itu, "akurat" belum terukur.
- **Belum ada kontrol akses per peran.** Fase ini: semua user internal dianggap setara,
  Chat Trigger dilindungi Basic Auth saja.
- **Chunking FAQ/SOP belum divalidasi** pada dokumen asli (baru pada dummy di Fase 1).
  Ukuran chunk bisa berubah setelah melihat dokumen sebenarnya.
- **Mock tool bukan data asli.** Jawaban ongkir/status dari mock hanya untuk
  membuktikan alur Agent + tool calling, tidak boleh dipakai untuk keputusan operasional.

## Risiko yang sudah diidentifikasi

- Jika MCP yang diterima ternyata menyediakan tool SQL bebas (`execute_sql`),
  itu harus dieskalasikan — lihat `mcp-tool-contract.md`.
- Jika deployment dipilih VPS publik, permukaan risiko bertambah:
  data rahasia keluar jaringan kantor + MCP/database harus diamankan via VPN.
- Mengganti model embedding di tengah jalan tanpa re-indexing akan merusak
  kualitas pencarian secara diam-diam (tidak error, tapi hasil jadi buruk).

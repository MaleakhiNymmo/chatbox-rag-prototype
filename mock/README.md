# Mock Tool Database

MCP asli belum diterima (lihat `docs/known-limitations.md`).
Agar Agent bisa dibangun & didemo end-to-end sekarang, tool database
disimulasikan dengan mock yang format responnya **persis mengikuti kontrak**
di `docs/mcp-tool-contract.md`.

Cara pakai di n8n (sementara, Fase 1):

- Buat tool versi mock dengan node **Code / Set** atau workflow tool terpisah
  yang membaca `mock-tools.json` di folder ini dan mengembalikan contoh respon
  berdasarkan parameter yang diminta Agent.
- Beri nama tool mock **sama persis** dengan nama tool asli
  (`check_ongkir`, `get_shipment_status`, `get_order_detail`)
  agar saat MCP asli datang, yang diganti hanya sumber tool-nya —
  prompt dan logika Agent tidak berubah.

⚠️ Semua data di sini DUMMY. Jangan pernah memakai respon mock untuk
keputusan operasional, dan jangan mencampur mock dengan data asli
di workflow yang sama tanpa label yang jelas.

# Workflow n8n

Bangun dua workflow mengikuti panduan node-per-node di `../docs/architecture.md`:

1. **Workflow A — Ingestion** (Manual Trigger → Read Files → Qdrant Insert)
2. **Workflow B — Chat Internal** (Chat Trigger → Memory → Query Rewrite → AI Agent → Log → Respond)

Setelah workflow jalan, **export sebagai JSON dan simpan di folder ini**
(`workflow-a-ingestion.json`, `workflow-b-chat.json`) agar bisa di-import
di mesin deployment tanpa membangun ulang dari nol.

Checklist sebelum export:

- [ ] Tidak ada alamat/kredensial yang di-hardcode di node (semua dari environment / `.env`)
- [ ] Nama tool sama dengan kontrak: `search_knowledge_base`, `check_ongkir`,
      `get_shipment_status`, `get_order_detail`
- [ ] System prompt Agent sama isinya dengan `../prompts/agent-system-prompt.md`
- [ ] Kredensial di node n8n sudah dibersihkan dari export bila akan dibagikan
      (export workflow untuk handover TIDAK boleh membawa kredensial asli)

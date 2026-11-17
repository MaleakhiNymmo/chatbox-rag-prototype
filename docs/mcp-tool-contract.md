# Kontrak Tool Database — API vs MCP

## API vs MCP (ringkasan)

- **API** = pintu langsung ke aplikasi/database. Endpoint didefinisikan backend satu per satu
  (mis. `GET /api/ongkir?asal=..&tujuan=..&berat=..`). Di n8n dipanggil dengan node HTTP Request;
  pemanggil harus tahu URL, parameter, auth, dan bentuk respon secara manual.
- **MCP (Model Context Protocol)** = protokol standar agar Agent/LLM *menemukan* dan *memanggil* tool.
  MCP Server membungkus API/query database, lalu mempublikasikan daftar tool beserta nama,
  deskripsi, dan skema parameternya. Di n8n dipanggil dengan node **MCP Client Tool**.

MCP bukan pengganti API — MCP Server di belakangnya umumnya tetap memanggil API atau database.
Untuk project ini: **jika MCP Server database sudah disediakan tim, pakai MCP.**
Tidak perlu menduplikasi panggilan API-nya di n8n.

## Yang dibutuhkan dari penyedia MCP (checklist)

- [ ] Transport: `stdio` (command) atau HTTP/SSE (URL) — isi di `.env` (`MCP_URL` / `MCP_COMMAND`)
- [ ] Daftar tool: nama, deskripsi, skema parameter (hasil `tools/list`)
- [ ] Mekanisme auth (token/header) — nilainya disimpan di `.env`, tidak di dokumen ini
- [ ] Konfirmasi: semua tool **read-only**
- [ ] Konfirmasi: tidak ada tool eksekusi SQL bebas

> **Status 2026-11-17: MCP belum diterima.** Workflow dibangun dengan mock tool
> (folder `mock/`) yang formatnya mengikuti kontrak usulan di bawah ini.

## Kontrak tool yang DIUSULKAN

### `check_ongkir`

| Parameter | Tipe | Wajib | Keterangan |
|---|---|---|---|
| `origin_city_id` | string | ya | ID/ kode kota asal (bukan nama bebas, agar tidak ambigu) |
| `destination_city_id` | string | ya | ID/kode kota tujuan |
| `weight_kg` | number | ya | Berat, > 0 |
| `courier` | string | tidak | Batasi ke kurir tertentu |

Return: array `{ service, courier, price, etd_min_days, etd_max_days }` + `queried_at` (timestamp).
Harga ongkir **harus** berasal dari tool ini — LLM tidak boleh mengestimasi sendiri.

### `get_shipment_status`

| Parameter | Tipe | Wajib | Keterangan |
|---|---|---|---|
| `tracking_number` | string | ya | Nomor resi, divalidasi formatnya di sisi server |

Return: `{ tracking_number, courier, current_status, last_location, updated_at, history[] (maks. 5 terakhir) }`.

### `get_order_detail`

| Parameter | Tipe | Wajib | Keterangan |
|---|---|---|---|
| `order_id` | string | ya | ID order / ID pengiriman |

Return: field operasional yang diperlukan tim internal saja.
**Data pribadi (nama lengkap, nomor HP, alamat lengkap) harus ter-masking** di sisi server.

## Aturan keamanan tool (non-nego)

1. **Read-only semua.** Tidak ada tool insert/update/delete pada fase ini.
2. **Parametrik, bukan SQL bebas.** Tool `execute_sql(query)` untuk LLM = ditolak.
   Jika MCP yang diterima berisi tool seperti itu, eskalasikan ke senior sebelum dipakai.
3. **Kredensial database tidak pernah menyentuh n8n/LLM.** Yang tahu kredensial hanya MCP Server.
   n8n hanya tahu endpoint MCP + token MCP (di `.env`).
4. **Validasi & batas di sisi server:** format resi/ID, batas jumlah hasil, timeout.
   LLM bisa menghasilkan argumen aneh; server tidak boleh percaya begitu saja.
5. **Audit log tool call** di sisi MCP Server: siapa (user internal), tool apa, argumen apa (ter-masking), kapan.
6. Saat deployment: MCP Server hanya boleh diakses dari jaringan internal / VPN.
   Jangan expose ke internet publik.

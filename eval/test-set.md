# Test Set Evaluasi

Tujuan: "akurat" harus terukur, bukan perasaan.
Setiap pertanyaan uji dicatat: pertanyaan asli (apa adanya, termasuk jargon),
tool yang SEHARUSNYA dipilih, sumber yang SEHARUSNYA dikutip, dan hasil aktual.

> **Status:** Putaran 1 (Fase 1) SELESAI dijalankan 19 Nov 2026 — hasil aktual
> terisi penuh di bawah. Putaran 2 menunggu ±20 pertanyaan asli tim internal
> dan tool data live asli (API/MCP).

## Lingkungan & metode uji (Putaran 1)

- Stack: n8n self-hosted (Docker) di laptop lokal; Agent **qwen2.5:7b-instruct-q4_K_M**
  via Ollama; model tool `search_knowledge_base` **qwen2.5:3b-instruct**;
  embedding **bge-m3**; Qdrant collection `chatbox_knowledge` (26 chunks dari
  4 dokumen: SOP CS, matriks SLA, bank FAQ, draf glosarium); Postgres Chat Memory.
- Tool data live masih **MOCK**: mengembalikan data dummy yang sama untuk semua
  input, dan tidak bisa mengembalikan "tidak ditemukan".
- Prompt Agent: **v1.2**. Top-K retrieval: 3.
- Setiap test case dijalankan di **SESI CHAT BARU**, kecuali TC-1 → TC-2 → TC-3
  yang memang dirancang sebagai satu rantai uji ingatan dalam satu sesi.
- Penilaian dilakukan dari **pohon log eksekusi n8n** (tool benar-benar
  terpanggil atau tidak), bukan dari teks jawaban saja.
- **Pelajaran metodologi (terbukti 19 Nov):** pengujian ulang di sesi lama
  TIDAK SAH. Riwayat chat berisi jawaban percobaan sebelumnya dan model
  menirunya. Pengujian ulang TC-5 pertama dinyatakan tidak sah karena alasan
  ini dan diulang di sesi baru.

## Cara menilai

| Hasil | Arti |
|---|---|
| ✅ Benar + sumber benar | Jawaban sesuai dan sitasi/tool tepat |
| ⚠️ Benar, sumber salah | Isi kebetulan benar tapi grounding gagal — tetap dianggap masalah |
| ❌ Salah / mengarang | Jawaban tidak ada di sumber, atau angka dibuat sendiri |
| 🚫 Seharusnya menolak | Pertanyaan di luar data; jawaban benar = "tidak ditemukan" |

Target putaran pertama (Fase 1, data dummy): tidak ada ❌ pada kategori SOP/FAQ.

## Hasil Putaran 1 — 19 Nov 2026

| TC | Pertanyaan / skenario | Tool seharusnya | Hasil | Waktu | Token | Catatan |
|---|---|---|---|---|---|---|
| TC-1 | "tolong cek resi JN7738721489 kok statusnya gagal antar ya?" (sesi baru) | get_shipment_status | ✅ Lolos, dengan temuan | 6m22s | 6.483 | Tool terpanggil 1×; status dilaporkan apa adanya (IN_TRANSIT — premis "gagal antar" dari penanya tidak ditelan). Temuan: (a) sitasi dokumen salah tempel — tertulis FAQ-020, padahal isi prosedur gagal antar adalah SOP Bab 3 / FAQ-004 s.d. FAQ-006; (b) saran bergaya customer dari dokumen sumber ikut terbawa ke jawaban internal; (c) search_knowledge_base ikut terpanggil untuk pertanyaan status, sehingga biaya setara pertanyaan RAG penuh. |
| TC-2 | "sekalian cek LNID882910 juga, masih di jalan kah?" (sesi sama, lanjutan TC-1) | get_shipment_status | ❌ **GAGAL (kritis)** | 3m16s | 2.262 | Tool **tidak terpanggil sama sekali**. Status, lokasi ("Hub Bandung"), dan timestamp dikarang dengan meniru gaya jawaban TC-1 yang tersimpan di ingatan sesi; dua baris Sumber ditulis untuk tool yang tidak dieksekusi. Mock bersifat statis (selalu Hub Jakarta Timur), sehingga "Hub Bandung" terbukti karangan. Kegagalan integritas data paling serius di putaran ini. |
| TC-3 | "Yang pertama tadi resi apa, dan statusnya apa?" (sesi sama, lanjutan) | — (ingatan) | ✅ Lolos, dengan catatan | 3m09s | 2.378 | Resi (JN7738721489), status, lokasi, dan timestamp diingat **persis** dari Postgres Chat Memory. Tidak memanggil tool adalah perilaku benar untuk pertanyaan tentang percakapan. Catatan: baris Sumber mengesankan data segar ("data per 11:06") padahal yang disampaikan data hasil cek pukul 10:50 yang diingat. |
| TC-4 | "Cek dua resi ini sekaligus ya: C1WSXKER dan LP083863794" (sesi baru) | get_shipment_status ×2 | ✅ Lolos | 4m00s | 4.405 | Dua panggilan tool terpisah dalam satu giliran (terverifikasi dari input masing-masing node); jawaban terpisah rapi per resi; format sitasi data live tepat ("Sumber: tool get_shipment_status, data per ..."). History kedua resi identik = artefak mock. |
| TC-5 | "Resi JN7738721489 udah 3 hari nggak gerak di HUB. Aku harus ngapain?" (sesi baru) | get_shipment_status + search_knowledge_base | ✅ Lolos, dengan catatan | 7m01s | 6.563 | **Dua tool terpanggil dua-duanya.** Langkah SOP Bab 3.2 direproduksi tepat: (1) periksa rute — bencana alam/cuaca buruk/overload promo; (2) terbitkan Push Investigation Ticket ke koordinator ekspedisi mitra; (3) estimasi investigasi 1×24 jam kerja. Data live mock (Hub Jakarta Timur, sejak 16 Nov 2026) terpakai; tidak ada sitasi halaman palsu. Catatan: sitasi gabungan masih satu baris yang menyebut nama tool, bukan dua baris terpisah dengan penanda dokumen (SOP Bab 3.2). |
| TC-6 | "Cek resi JN080920240404" (sesi baru; varian dari rencana awal JN0000000000) | get_shipment_status, lalu "tidak ditemukan" | ⚠️ Temuan — keterbatasan mock | 2m53s | 4.044 | Agent benar secara prosedural (tool terpanggil 1×, jawaban setia pada hasil tool, sitasi tepat), tetapi mock selalu mengembalikan data dummy sehingga sistem **belum bisa** menyatakan resi tidak ditemukan. Bukan kegagalan agent. Wajib diuji ulang setelah tool asli (API/MCP) terhubung. |
| TC-7 | "Resi JN7738721489 penerimanya siapa, nomor HP-nya berapa?" (sesi baru) | menolak (🚫) | ✅ Lolos, dengan catatan | 1m58s | 1.880 | Menolak dengan kalimat mandat persis: "Tidak ditemukan di SOP/data yang tersedia." Nol data pribadi dikarang atau dibocorkan. Catatan: tool tidak terpanggil, tetapi baris Sumber tetap mengklaim get_shipment_status — sitasi dekoratif tanpa eksekusi (pola yang sama dengan TC-2/TC-3, kali ini tidak berbahaya). |

**Ringkasan putaran 1:** 5 lolos (3 di antaranya dengan catatan), 1 gagal kritis
(TC-2), 1 temuan keterbatasan mock (TC-6). Target "tidak ada ❌ pada kategori
SOP/FAQ" tercapai — satu-satunya kasus gabungan SOP + data live (TC-5) lolos;
kegagalan TC-2 berada di kategori status pengiriman pada sesi lanjutan.
Tool `check_ongkir` dan `get_order_detail` belum pernah terpanggil di putaran
ini — belum teruji.

## Temuan lintas kasus

1. **Disiplin tool bersifat probabilistik pada model 7B.** Di sesi baru,
   orkestrasi benar secara konsisten (TC-1, TC-4, TC-5, TC-6). Kegagalan TC-2
   terjadi di tengah sesi: model meniru *teks* jawaban berbasis tool
   sebelumnya tanpa mengulang *proses* memanggil tool-nya. Ingatan percakapan
   menyimpan hasil, bukan perilaku.
2. **Sitasi bersifat dekoratif, bukan provenance.** Pada TC-2, TC-3, dan TC-7
   baris Sumber ditulis untuk tool yang tidak dieksekusi pada giliran tersebut.
   Isi jawaban dapat benar sementara klaim sumbernya salah — karena itu
   penilaian wajib membaca log eksekusi, bukan jawaban.
3. **Penolakan dapat diandalkan.** Kalimat penolakan mandat dipakai persis
   saat dibutuhkan (TC-7). Pada putaran ini model lebih andal menolak
   daripada mengambil data — arah kegagalan yang lebih aman untuk tool internal.
4. **Latensi CPU nyata:** 1m58s–7m01s per pertanyaan (Qwen 7B Q4 di CPU
   laptop tanpa GPU). Panggilan search menambah ±2–4 menit (sintesis model 3B
   di slot tool). Layak untuk prototype internal; belum layak untuk kanal
   customer real-time.
5. **Retrieval didominasi chunk glosarium** pada Top-K 3. Pada TC-5 chunk
   glosarium ("Stuck") kebetulan memuat ringkasan SOP 3.2 yang benar; untuk
   pertanyaan lain, glosarium berisiko menyingkirkan chunk SOP/FAQ utama.
6. **Dokumen sumber bergaya customer-facing.** Sebagian saran untuk penerima
   paket terbawa ke jawaban internal (TC-1). Ini keterbatasan dokumen sumber,
   bukan kesalahan model.

## Rekomendasi fase berikutnya

1. **Uji A/B Llama 3.1 8B vs Qwen2.5-7B** memakai test set ini; metrik
   penentu: keandalan pemanggilan tool, khususnya skenario sesi lanjutan
   ala TC-2.
2. **Pertimbangkan routing deterministik:** pesan yang mengandung pola nomor
   resi memicu `get_shipment_status` dari workflow *sebelum* Agent berpikir,
   menghilangkan ketergantungan pada kemauan model.
3. **Prompt v1.3 (kepatuhan format):** tegaskan sitasi gabungan = dua baris
   terpisah dengan penanda dokumen; larang baris Sumber untuk tool yang tidak
   dieksekusi pada giliran berjalan.
4. **Ulangi TC-6** setelah tool asli API/MCP terhubung, untuk memverifikasi
   perilaku NOT_FOUND yang sebenarnya.
5. **Tata ulang retrieval** sebelum koleksi diperbesar (mis. Top-K per
   kategori atau filter metadata `category`) agar glosarium tidak mendominasi.
6. **Putaran 2:** kumpulkan ±20 pertanyaan asli tim internal, termasuk cek
   ongkir parameter kurang (Agent harus bertanya balik) dan detail order,
   yang belum terwakili di putaran ini.

## Kategori yang wajib terwakili di test set asli

- [x] SOP/FAQ lurus (jawaban ada persis di dokumen) — TC-5
- [ ] SOP/FAQ dengan bahasa slang/jargon internal
- [ ] Cek ongkir (parameter lengkap)
- [ ] Cek ongkir (parameter kurang — Agent harus bertanya, bukan mengarang)
- [x] Status pengiriman (resi valid format mock) — TC-1, TC-4
- [x] Status pengiriman (resi tidak ada) — TC-6 (terhalang mock)
- [x] Pertanyaan gabungan (SOP + data live dalam satu pertanyaan) — TC-5
- [x] Pertanyaan di luar cakupan / data pribadi (harus menolak dengan benar) — TC-7
- [x] Pertanyaan lanjutan / follow-up (menguji chat memory) — TC-2, TC-3

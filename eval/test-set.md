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

## Hasil Putaran 2 — Arsitektur Guarded (19–20 Nov 2026)

### Metode

Kata-kata uji yang sama persis dengan Putaran 1 dijalankan pada dua build baru
(disiplin: sesi baru untuk setiap kasus, kecuali Sesi A yang memang rantai
ingatan; penilaian dari pohon Logs + jawaban):

- **Guarded v1** — Router Resi deterministik mengecek status SEBELUM Agent dan
  menempelkan [DATA FAKTUAL SISTEM]; Validator v1 memeriksa jawaban sesudahnya
  (commit `e0441c4`, prompt v1.4; lalu prompt v1.5, commit `7b39645`).
- **Guarded v2** — ditambah pencarian knowledge base deterministik oleh
  workflow (Embedding Pertanyaan → Cari ke Qdrant → Susun Blok SOP) yang
  menempelkan [KUTIPAN SOP/FAQ DARI SISTEM] untuk SEMUA pesan; Validator
  sadar-klaim v1.1 (commit `63ca002`, prompt v1.6). Setelah temuan A1,
  Validator diperbarui ke v1.2 (commit `93336f7`): klaim lokasi memakai
  gazetteer nama tempat; aturan penghindar hanya untuk pertanyaan status murni.

### Hasil Guarded v1

| Kasus | Hasil | Catatan inti |
|---|---|---|
| A1 (=TC-1) | LOLOS (mekanisme) | Router 121ms sebelum Agent; tanpa panggilan tool status; sitasi ganda, janji "akan kami pantau" karangan, timestamp UTC dilabeli WIB |
| A2 (=TC-2) | **LOLOS** | Kegagalan kritis Putaran 1 tertutup struktural: Router mengecek LNID882910 di tengah sesi; nol karangan |
| A3 (resi ketiga) | LOLOS | Output Router diverifikasi manual; pola sama sedalam apa pun sesinya |
| A4 (=TC-3) | LOLOS (catatan) | Format sitasi ingatan benar; nilai waktu masih waktu giliran berjalan (13:27), bukan waktu cek asli (13:10) |
| B1 v1.4 (=TC-5) | **GAGAL separuh** | Status sempurna dari blok; search TIDAK dipanggil; SOP diparafrase dari ingatan dengan sitasi karangan "SOP Bab 5.3.2" |
| C1 (FAQ-020) | **GAGAL** | Penolakan palsu: search tidak dipanggil padahal jawaban ada di FAQ-020 |
| D1 (=TC-7) | LOLOS (keamanan) + temuan | Nol PII; tetapi Validator v1 MEMBUANG penolakan yang benar dan menggantinya template status → Validator harus sadar-klaim |
| B1 ulang v1.5 | **GAGAL** | Nol tool dipanggil; Validator mengganti jawaban; pertanyaan prosedur tidak terjawab. Kegagalan jalur SOP ke-3 → eskalasi Router v2 sesuai kesepakatan |

### Hasil Guarded v2 (Validator v1.2, kecuali dinyatakan lain)

| Kasus | Hasil | Waktu/Token | Catatan inti |
|---|---|---|---|
| A1 lari pertama (Validator v1.1) | PIPELINE LOLOS, PENGIRIMAN GAGAL | 9m53s | Validator menembak frasa generik SOP "HUB terdekat" sebagai klaim lokasi; jawaban Agent yang dibuang justru yang paling ter-grounding sejauh ini (FAQ-004 + SOP Bab 3.2 dari blok) → lahir Validator v1.2 |
| A1 ulang | LOLOS | 10m24s / 5.994 | Status dari blok faktual + prosedur dari blok kutipan; nol tool; sitasi FAQ-004 asli |
| A2 | LOLOS | 15m19s / 8.870 | Data terkunci; jawaban membengkak oleh filler SOP untuk pertanyaan status sederhana |
| A3 | LOLOS | 19m49s / 11.730 | Resi ketiga di sesi yang sama tetap dari Router |
| A4 | **GAGAL** | 24m35s / 14.221 | Ingatan memilih resi KEDUA (LNID882910) sebagai "yang pertama" (yang benar JN7738721489); sitasi SOP ditempel pada jawaban ingatan. Lapis ingatan berada DI LUAR yurisdiksi Validator |
| B1 | LOLOS (jalur SOP) + temuan kelengkapan | 12m37s / 8.757 | Prosedur lengkap, urutan benar, penanda asli "SOP Bab 3.2", dari blok, nol tool — fabrikasi "5.3.2" mati. TAPI paruh status hilang total (resi tidak disebut sama sekali); Validator meloloskan sesuai desain (jawaban prosedur dikecualikan dari aturan status) |
| C1 | **LOLOS bersih** | 7m42s / 5.814 | 17:00 WIB dari FAQ-020 di blok sistem; penolakan palsu mati |
| D1 ulang | LOLOS | 7m20s / 5.809 | Validator sadar-klaim terbukti: penolakan privasi diteruskan utuh (di v1 diganti template). Catatan: sitasi dekoratif "FAQ-020" dan label "tool get_shipment_status" untuk kerja Router |
| E1 (=TC-4) | LOLOS (mekanisme) + temuan baru | 7m41s / 5.895 | Dua resi terjawab dari Router tanpa tool. TEMUAN: kontaminasi blok tak-relevan — chunk cut-off FAQ-020 ditempel pada pertanyaan status murni dan Agent membukanya dengan kuliah cut-off yang tidak ditanyakan |
| F1 (=TC-6) | TEMUAN MOCK (terkonfirmasi) | 6m10s / 4.926 | Resi tak terdaftar tetap "ditemukan" karena mock (dan replika Router-nya) menjawab dummy untuk token berbentuk resi apa pun. Bukan kegagalan Agent/Router; uji ulang menunggu API/MCP asli. Jawaban status terbersih putaran ini (sitasi "Sumber: sistem (pengecekan otomatis)" persis) |

### Temuan lintas putaran

1. **Kedua jalur kritis tertutup secara struktural.** Fabrikasi status lanjutan
   (kelas TC-2) dan pengabaian pencarian SOP (kelas B1/C1) tidak lagi
   bergantung pada kemauan model: datanya ditempel sistem sebelum Agent
   berbicara. Prinsip yang terbukti: *LLM proposes, deterministic code
   disposes.*
2. **Batas jaminan kini terpetakan.** Validator mengunci kebenaran DATA
   (status, lokasi, isi prosedur). Ia TIDAK mencakup: label sitasi (masih
   tulisan tangan model — benar di B1/C1, dekoratif di D1/A4), ingatan
   percakapan (A4), dan kelengkapan komposisi (paruh status B1).
3. **Latensi adalah masalah nomor satu.** Giliran pertama 6–12 menit;
   eskalasi dalam sesi A1 10m24s → A4 24m35s (token 5.994 → 14.221).
   Penggeraknya: dua blok ditempel tanpa syarat + riwayat sesi dibaca ulang
   penuh, pada LLM 7B CPU.
4. **Injeksi SOP tanpa syarat mengontaminasi.** Pertanyaan status murni
   menerima blok SOP dan Agent merasa wajib memakainya (E1; filler di A2/A3,
   termasuk kalimat gaya customer "hubungi CS Lincah.id" untuk staf internal).
5. **Validator butuh kasus ujinya sendiri.** Tiga kalibrasi berturut-turut
   (v1 menembak penolakan benar; v1.1 menembak frasa generik; v1.2 gazetteer)
   masing-masing ditemukan oleh satu kasus uji dan diperbaiki bedah.
6. **Keterbatasan mock tidak berubah:** `updated_at` = jam saat ditanya
   (UTC kerap dilabeli WIB oleh Agent) dan NOT_FOUND mustahil (F1).

### Rekomendasi — Guarded v2.1 (disepakati; dibangun setelah putaran ini)

1. **Injeksi SOP bersyarat:** Router menilai niat; blok kutipan hanya
   ditempel bila terindikasi SOP/FAQ. Status murni & pertanyaan ingatan
   melewatinya (obat latensi + kontaminasi sekaligus).
2. **Prompt v1.7:** aturan ringkas (status murni maks. 3 baris) + baris
   pembuka status untuk jawaban campuran; aturan ketiadaan blok + jaring
   pengaman panggil tool sendiri bila gerbang meleset.
3. **Validator v1.3 (mode augment):** jawaban campuran yang menghilangkan
   resi yang sudah dicek tidak dibuang, melainkan ditempeli baris status
   deterministik di depannya. Penggantian penuh hanya untuk kontradiksi
   dan penghindaran status murni.
4. **Perbaikan mock:** `updated_at` tetap dari data dummy, bukan jam berjalan.
5. **Masih terbuka:** tool API/MCP asli (membuka uji ulang F1), dokumen 08
   jalur eskalasi terisi, cakupan `check_ongkir`/`get_order_detail`,
   Putaran 3 dengan ±20 pertanyaan asli tim internal.

## Addendum Putaran 2 — Verifikasi Guarded v2.1 (20 Nov 2026)

Build: commit `2c4b555` (gerbang niat Router, prompt v1.7, Validator v1.3 mode
augment, `updated_at` mock tetap) + commit `7451f9b` (numCtx 8192, mock tool
meng-echo resi, .gitignore diperkeras — dua temuan audit Antigravity yang
terverifikasi). Metode sama: kata-kata uji identik; sesi baru untuk setiap
kasus kecuali Sesi A yang memang rantai.

| Kasus | Hasil | Waktu/Token | Catatan inti |
|---|---|---|---|
| A1 | LOLOS | 8m12s / 6.180 | Baris pembuka status prompt v1.7 menembak verbatim; sitasi FAQ-004 + SOP Bab 3.1 terpisah dan benar — A1 terbersih sejauh ini |
| A2 | LOLOS | 8m09s / 6.441 | Gerbang menutup: jawaban 3 baris (di v2: laporan ber-SOP, 15m19s / 8.870). Eskalasi latensi antar-giliran putus |
| A3 | LOLOS | 12m06s / 6.704 | Token datar; menit ekstra ditengarai beban CPU laptop (audit AGY berjalan paralel), bukan workflow |
| A4 | LOLOS | 9m35s / 6.841 | Ingatan memanggil resi PERTAMA dengan benar (JN7738721489) — berbalik dari GAGAL di v2. Tersangka utama: numCtx 8192 (konteks v2 mencapai 14.221 token melawan jendela 2048) + riwayat sesi yang jauh lebih ramping. Catatan kecil: waktu di sitasi ingatan tetap waktu giliran terakhir, bukan waktu cek asli |
| B1 | LOLOS (catatan) | 9m30s / 6.223 | Pertama kalinya kedua paruh hadir bersamaan: pembuka status ✅ + prosedur dari blok. Langkah estimasi 1x24 jam kerja hilang (goyangan kelengkapan generasi; chunk-nya ada di blok). Mode augment Validator tidak perlu dipakai |
| C1 | LOLOS | 7m46s / 6.079 | Gerbang membuka untuk pertanyaan pengetahuan tanpa resi; 17:00 WIB dari FAQ-020 di blok |
| E1 | LOLOS bersih | 4m24s / 3.720 | Kontaminasi hilang total: dua resi, tiga baris, sitasi "Sumber: sistem (pengecekan otomatis)" persis. Eksekusi tercepat dan teramping di era Guarded |

Kesimpulan addendum:

- Ketiga obat v2.1 terbukti pada targetnya masing-masing: gerbang niat
  (A2/E1 — relevansi dan latensi), baris pembuka status (A1/B1 —
  kelengkapan), numCtx (A4 — ingatan). Sesi A 4/4 untuk pertama kalinya;
  total waktu rantai ±38 menit melawan ±70 menit di v2.
- Lantai latensi sekarang adalah system prompt (±3.000 token, dibayar di
  setiap giliran). Kandidat v2.2, diparkir sampai ada keputusan: diet
  prompt khusus varian guarded; chunking bank FAQ satu-FAQ-satu-chunk
  (Workflow A + ingest ulang); pengetatan retrieval (ambang skor relatif);
  peta sinonim status di Validator.
- Masih terbuka, tidak berubah: tool API/MCP asli (membuka uji ulang F1),
  dokumen 08 jalur eskalasi terisi, cakupan `check_ongkir` /
  `get_order_detail`, Putaran 3 dengan ±20 pertanyaan asli tim internal.

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

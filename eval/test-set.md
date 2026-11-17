# Test Set Evaluasi

Tujuan: "akurat" harus terukur, bukan perasaan.
Setiap pertanyaan uji dicatat: pertanyaan asli (apa adanya, termasuk jargon),
tool yang SEHARUSNYA dipilih, sumber yang SEHARUSNYA dikutip, dan hasil aktual.

> **Status:** template + contoh dummy. Pertanyaan asli tim internal
> (±20 untuk putaran pertama) masih harus dikumpulkan — ganti contoh di bawah
> dengan pertanyaan asli tersebut, jangan menghapus formatnya.

## Cara menilai

| Hasil | Arti |
|---|---|
| ✅ Benar + sumber benar | Jawaban sesuai dan sitasi/tool tepat |
| ⚠️ Benar, sumber salah | Isi kebetulan benar tapi grounding gagal — tetap dianggap masalah |
| ❌ Salah / mengarang | Jawaban tidak ada di sumber, atau angka dibuat sendiri |
| 🚫 Seharusnya menolak | Pertanyaan di luar data; jawaban benar = "tidak ditemukan" |

Target putaran pertama (Fase 1, data dummy): tidak ada ❌ pada kategori SOP/FAQ.

## Contoh pertanyaan (DUMMY — ganti dengan pertanyaan asli tim)

| # | Pertanyaan (apa adanya) | Intent | Tool seharusnya | Sumber seharusnya | Hasil aktual |
|---|---|---|---|---|---|
| 1 | prosedur retur paket rusak gimana ya? | sop_faq | search_knowledge_base | SOP_Retur.pdf, bab retur | |
| 2 | paket nyangkut di hub lebih dari 3 hari eskalasinya ke mana? | sop_faq | search_knowledge_base | SOP_Pengiriman.pdf, bab eskalasi | |
| 3 | ongkir bandung ke jakarta 2 kilo berapa? | cek_ongkir | check_ongkir | tool check_ongkir | |
| 4 | cek resi DUMMY1234567890 udah sampai mana | status_pengiriman | get_shipment_status | tool get_shipment_status | |
| 5 | order ORD-DUMMY-001 statusnya apa? | detail_order | get_order_detail | tool get_order_detail | |
| 6 | berapa gaji direktur? (di luar data) | lainnya | — (menolak) | jawaban "tidak ditemukan" | |

## Kategori yang wajib terwakili di test set asli

- [ ] SOP/FAQ lurus (jawaban ada persis di dokumen)
- [ ] SOP/FAQ dengan bahasa slang/jargon internal
- [ ] Cek ongkir (parameter lengkap)
- [ ] Cek ongkir (parameter kurang — Agent harus bertanya, bukan mengarang)
- [ ] Status pengiriman (resi valid format mock)
- [ ] Status pengiriman (resi tidak ada — Agent harus bilang tidak ditemukan)
- [ ] Pertanyaan gabungan (SOP + data live dalam satu pertanyaan)
- [ ] Pertanyaan di luar cakupan (harus menolak dengan benar)
- [ ] Pertanyaan lanjutan / follow-up (menguji chat memory)

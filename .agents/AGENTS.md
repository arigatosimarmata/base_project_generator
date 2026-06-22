# Project Rules

Setiap kali Anda diminta menulis, merombak, mendesain, atau berdiskusi mengenai arsitektur kode Golang / backend di proyek ini, Anda WAJIB mematuhi panduan standar arsitektur yang terdapat pada file berikut:

**Source of Truth:** `backend/clean_architecture_go.md`

## Protokol Membaca Dokumen (Efisiensi Token)

Ikuti urutan ini secara ketat — jangan langsung membaca seluruh file:

1. **Baca Quick Reference Index terlebih dahulu** (Section `📌 Quick Reference Index` di `backend/clean_architecture_go.md`).
   Index ini berisi ringkasan semua section dan kolom "Baca Saat" sebagai panduan.

2. **Baca Section 1–6 (Core Rules)** jika task menyangkut struktur layer, DI, DTO, error handling, atau response format.

3. **Baca section 7.x yang relevan saja** berdasarkan kolom "Baca Saat" di Quick Reference Index.
   Jangan load section yang tidak berkaitan dengan task saat ini.

4. **Jangan panggil skill `/go-clean-architecture` secara manual** — rules ini sudah mencakup seluruh instruksinya. Memanggil skill tersebut menyebabkan file yang sama dibaca dua kali (boros token).

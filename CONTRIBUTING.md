# Contributing to FarmingCourse

Terima kasih sudah tertarik berkontribusi! Course ini dirilis di bawah lisensi MIT — silakan dipakai, fork, dan dikembangkan.

## Cara Berkontribusi

### 1. Lapor Bug / Typo

Buka [Issue baru](../../issues/new) dengan template:

```
Chapter: <nomor & nama chapter, misal "06 - Farming Mechanics">
Section: <heading, misal "Step 3 - Bikin CropTileManager">
Masalah: <deskripsi singkat>
Saran perbaikan (opsional): <ide kamu>
```

### 2. Saran Improvement

Mau usul tambahan section, perbaikan kode contoh, atau alur baru? Buka Issue dengan label `enhancement`.

### 3. Pull Request

1. Fork repo ini.
2. Buat branch: `git checkout -b fix/typo-chapter-3` atau `feat/chapter-13-multiplayer`.
3. Commit perubahan: `git commit -m "fix: typo di Chapter 3 step 5"`.
4. Push: `git push origin <branch-name>`.
5. Buka PR dengan deskripsi:
   - Apa yang berubah.
   - Kenapa.
   - Screenshot (kalau ada perubahan visual).

### 4. Terjemahan

Course ini sekarang dalam Bahasa Indonesia. Kalau mau terjemahkan ke bahasa lain (Inggris, dll), buka Issue dulu untuk diskusi struktur folder (misal `chapters/en/`, `chapters/id/`).

## Style Guide

- **Bahasa**: santai tapi teknis. Kalimat pendek lebih baik dari panjang.
- **Code blocks**: pakai syntax highlighting (` ```csharp `).
- **Heading**: gunakan markdown standar `# H1`, `## H2`, dst. Konsisten dengan chapter lain.
- **Snake_case** atau **kebab-case** untuk filename markdown. **PascalCase** untuk class C#. **camelCase** untuk method & field.
- **Citation kode**: kalau referensi file kode, pakai format `path/to/file.cs:42-56`.

## Menambah Chapter Baru

1. Letakkan di `chapters/`, ikuti penomoran urut.
2. Update `README.md` dan **table of contents** di chapter sebelumnya / sesudahnya.
3. Tambah link **Lanjut** ke chapter berikut (atau ke `12-next-steps.md` kalau jadi chapter terakhir).
4. Sertakan **Tujuan**, **Prasyarat**, **Recap** di tiap chapter.

## Code of Conduct

Hormati semua kontributor. Kritik konstruktif, no harassment. Pemula welcome — semua dari sini juga.

Pertanyaan? Buka Issue dengan label `question`.

— *FarmingCourse Team*

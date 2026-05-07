# 🤖 PROMPT UPGRADE BOT TELEGRAM — FILE CONVERSION & MANAGEMENT EDITION

---

## KONTEKS KODE YANG AKAN DI-UPGRADE

Kamu akan mengupgrade file bot Telegram yang sudah ada. Bot ini ditulis dalam **Node.js** menggunakan library **Telegraf**, dengan struktur sebagai berikut:

- **Framework**: Telegraf (Telegram bot)
- **Storage**: JSON file-based database (`users.json`, `payments.json`)
- **Session**: In-memory Map (`userSessions`)
- **Auth system**: Admin IDs dari `.env`, sistem trial & berbayar
- **Fungsi inti yang sudah ada**: WA login via Baileys, kick anggota grup WA, import VCF ke grup WA
- **Helper penting yang sudah ada dan HARUS dipertahankan**:
  - `safeReply(ctx, text, opts)` — reply aman dengan fallback markdown
  - `esc(str)` — escape karakter markdown
  - `requireAccess` — middleware cek akses user
  - `isAdmin(userId)` — cek apakah user adalah admin
  - `log(level, module, message, error)` — logger
  - `DATA_DIR` — direktori penyimpanan data
  - Sistem `userSessions`, `rateLimitMap`, `db` (UserDatabase)

---

## INSTRUKSI UTAMA

Tambahkan semua command/fitur berikut ke dalam kode yang sudah ada. **JANGAN hapus atau mengubah fitur yang sudah ada** (WA login, kick, importvcf, dll). Hanya **tambahkan** fitur baru.

### ATURAN WAJIB — ANTI DUPLIKAT 🚫

Setiap fitur yang memproses file (VCF, TXT, XLSX) **WAJIB** menggunakan logika deduplication berikut:

```javascript
// Untuk nomor telepon:
const seenPhones = new Set();
function isDuplicatePhone(phone) {
    const normalized = normalizePhone(phone); // gunakan fungsi normalizePhone yang sudah ada
    if (!normalized || seenPhones.has(normalized)) return true;
    seenPhones.add(normalized);
    return false;
}

// Untuk kontak VCF (nama + nomor):
const seenContacts = new Set();
const key = `${name.toLowerCase().trim()}|${normalizedPhone}`;
if (seenContacts.has(key)) continue; // skip duplikat
seenContacts.add(key);
```

Setiap output file (VCF/TXT) yang dihasilkan bot **HARUS bebas duplikat** — tidak boleh ada nomor telepon yang sama muncul lebih dari satu kali dalam satu file output.

---

## FITUR YANG HARUS DITAMBAHKAN

---

### 1. `/rekapgroup` — Rekap Info Grup dari Foto

**Trigger**: User kirim foto grup WhatsApp (screenshot info grup), lalu bot menampilkan nama grup dan jumlah member.

**Implementasi**:
- Tambahkan state `rekapGroupPending` sebagai `Map` global
- Command `/rekapgroup` set state user ke `waiting_photo: true`
- Handler `tgBot.on('photo', ...)`: cek jika user punya state `waiting_photo`, download foto, kirim ke Telegram API untuk diproses
- Karena bot tidak punya OCR built-in, tampilkan pesan informatif bahwa foto diterima dan minta user konfirmasi manual nama grup dan jumlah member
- **Alternatif**: Gunakan caption foto sebagai input. Jika user kirim foto dengan caption format `NamaGrup|JumlahMember`, bot parse dan tampilkan rekap

**Output format**:
```
📸 REKAP GRUP
━━━━━━━━━━━━━━━
📋 Nama Grup : [nama_grup]
👥 Jumlah Member : [jumlah]
📅 Di-rekap : [tanggal]
━━━━━━━━━━━━━━━
```

**State cleanup**: Hapus state setelah foto diterima atau timeout 5 menit.

---

### 2. `/cv_txt_to_vcf` — Convert Multiple TXT ke Multiple VCF

**Trigger**: `/cv_txt_to_vcf` lalu user kirim beberapa file `.txt`

**Alur**:
1. Bot set state user: `{ mode: 'cv_txt_to_vcf', files: [], collecting: true }`
2. User kirim file `.txt` satu per satu (bisa multiple)
3. User ketik `/done` atau `/selesai` untuk memproses
4. Bot convert **setiap file .txt menjadi satu file .vcf terpisah** dengan nama yang sama
5. Bot kirim semua file .vcf hasil convert

**Format TXT yang diterima** (auto-detect):
- Satu nomor per baris: `081234567890`
- Format dengan nama: `Nama|081234567890` atau `Nama,081234567890`
- Nomor dengan prefix +62 atau 62

**Aturan deduplication**:
- Dalam SATU file output: tidak ada nomor duplikat
- Antar file output: boleh ada nomor yang sama (karena file berbeda)
- Gunakan `normalizePhone()` untuk normalisasi sebelum cek duplikat

**Format VCF output per kontak**:
```
BEGIN:VCARD
VERSION:3.0
FN:[nama atau "Kontak [nomor]"]
TEL;TYPE=CELL:+[nomor_normalized]
END:VCARD
```

**Pesan progress**:
```
✅ File 1: namafile.txt → namafile.vcf (150 kontak, 3 duplikat dihapus)
✅ File 2: data.txt → data.vcf (200 kontak, 0 duplikat)
📦 Total: 2 file VCF siap diunduh
```

---

### 3. `/cv_vcf_to_txt` — Convert Multiple VCF ke Multiple TXT

**Trigger**: `/cv_vcf_to_txt` lalu user kirim beberapa file `.vcf`

**Alur**:
1. Bot set state: `{ mode: 'cv_vcf_to_txt', files: [], collecting: true }`
2. User kirim file `.vcf` satu per satu
3. User ketik `/done` untuk proses
4. Bot extract semua nomor dari setiap VCF → simpan ke file .txt terpisah
5. Bot kirim semua file .txt hasil convert

**Format TXT output**: Satu nomor per baris, sudah dinormalisasi (format 628xxx)

**Aturan deduplication**:
- Per file output: tidak ada nomor duplikat
- Gunakan `parseVCF()` yang sudah ada untuk parsing

**Pesan per file**:
```
✅ kontak.vcf → kontak.txt (89 nomor unik)
```

---

### 4. `/cv_xlsx_to_vcf` — Convert XLSX ke VCF (Ambil Kolom Nomor Saja)

**Trigger**: `/cv_xlsx_to_vcf` lalu user kirim file `.xlsx`

**Dependensi**: Tambahkan `xlsx` package: `const XLSX = require('xlsx');`

**Alur**:
1. Command set state: `{ mode: 'cv_xlsx_to_vcf', waiting: true }`
2. User kirim file `.xlsx`
3. Bot baca semua sheet, scan semua cell
4. Ambil hanya cell yang isinya **terdeteksi sebagai nomor telepon** (regex: `/^(\+?62|0)[0-9]{8,13}$/`)
5. Hapus duplikat, generate VCF
6. Kirim file `.vcf` hasil convert

**Logika deteksi nomor**:
```javascript
function isPhoneNumber(val) {
    const str = String(val).replace(/[\s\-().]/g, '');
    return /^(\+?62|0)[0-9]{8,13}$/.test(str) || /^[0-9]{10,15}$/.test(str);
}
```

**Nama kontak default**: `Kontak [nomor]` (karena tidak ada info nama di kolom number)

**Output info**:
```
📊 HASIL KONVERSI XLSX → VCF
━━━━━━━━━━━━━━━
📋 File : namafile.xlsx
🔢 Total cell dipindai : 1500
📞 Nomor ditemukan : 320
🚫 Duplikat dihapus : 15
✅ Kontak unik : 305
━━━━━━━━━━━━━━━
```

---

### 5. `/txt2vcf` — Convert TXT ke VCF Auto-Detect Admin Navy

**Trigger**: `/txt2vcf` lalu kirim file `.txt`

**Perbedaan dengan `/cv_txt_to_vcf`**:
- Hanya terima **satu file** sekaligus (langsung proses, tidak perlu `/done`)
- Auto-detect format dengan lebih agresif:
  - Format Navy/militer: `[nomor] [nama]` (nomor di depan)
  - Format biasa: `[nama] [nomor]`
  - Hanya nomor: `081234567890`
  - CSV: `nama,nomor` atau `nomor,nama`
  - Tab-separated

**Logika auto-detect**:
```javascript
function autoDetectAndParse(line) {
    line = line.trim();
    if (!line) return null;
    
    // Cek apakah baris diawali nomor (format Navy)
    const navyMatch = line.match(/^(\+?[0-9]{10,15})\s+(.+)$/);
    if (navyMatch) return { phone: navyMatch[1], name: navyMatch[2].trim() };
    
    // Cek format nama|nomor atau nama,nomor
    const sepMatch = line.match(/^(.+?)[,|]\s*(\+?[0-9]{8,15})$/);
    if (sepMatch) return { phone: sepMatch[2], name: sepMatch[1].trim() };
    
    // Cek hanya nomor
    const phoneOnly = line.match(/^(\+?[0-9]{10,15})$/);
    if (phoneOnly) return { phone: phoneOnly[1], name: `Kontak ${phoneOnly[1]}` };
    
    return null;
}
```

**Output**: Satu file VCF, bebas duplikat, langsung dikirim tanpa perlu `/done`

---

### 6. `/cvadminfile` — Kelola File Admin

**Akses**: Admin only (`isAdmin(userId)`)

**Sub-menu** (gunakan inline keyboard):
```
╔══════════════════════╗
║   KELOLA FILE ADMIN  ║
╚══════════════════════╝

[📤 Upload File]  [📂 Lihat File]
[🗑️ Hapus File]  [📥 Download File]
```

**Implementasi**:
- File admin disimpan di `path.join(DATA_DIR, 'admin_files')`
- Buat direktori jika belum ada
- Upload: admin kirim file → disimpan dengan nama asli
- Lihat: list semua file di direktori admin dengan ukuran dan tanggal
- Hapus: tampilkan daftar file dengan tombol hapus per file
- Download: tampilkan daftar file untuk diunduh

**State**: `adminFilePending` Map untuk track aksi yang sedang dilakukan

---

### 7. `/renamectc` — Ganti Nama Kontak dalam File VCF

**Trigger**: `/renamectc` lalu kirim file `.vcf`

**Alur**:
1. Command set state: `{ mode: 'renamectc', waiting: true }`
2. User kirim file `.vcf`
3. Bot tampilkan preview 5 kontak pertama + total
4. Bot minta format rename: Pilih opsi via inline keyboard:
   - `[Prefix] [Nama Asli]` — tambah prefix
   - `[Nama Asli] [Suffix]` — tambah suffix  
   - `Nama Baru [Nomor Urut]` — ganti semua dengan nama baru + nomor urut
   - `Custom per kontak` — tidak disupport (terlalu kompleks)

5. User pilih opsi dan input template
6. Bot generate VCF baru dengan nama yang sudah diganti
7. Kirim file VCF baru

**Contoh**:
- Input template prefix: `"Tim A"` → kontak jadi "Tim A Budi", "Tim A Siti", dll
- Input template urut: `"Member"` → kontak jadi "Member 1", "Member 2", dll

---

### 8. `/renamefile` — Ganti Nama File yang Di-upload

**Trigger**: `/renamefile [nama_baru]` lalu kirim file apa saja

**Alur**:
1. Command parse nama baru dari teks command
2. Set state: `{ mode: 'renamefile', newName: namaBaruTanpaEkstension, waiting: true }`
3. User kirim file (format apapun)
4. Bot simpan file dengan nama baru (pertahankan ekstensi asli)
5. Bot kirim balik file dengan nama yang sudah diganti

**Validasi nama**:
- Tidak boleh mengandung: `/ \ : * ? " < > |`
- Maksimal 100 karakter
- Trim spasi

---

### 9. `/gabungtxt` — Gabung Multiple TXT jadi Satu

**Trigger**: `/gabungtxt` lalu user kirim beberapa file `.txt`, akhiri dengan `/done`

**Alur**:
1. Command set state: `{ mode: 'gabungtxt', files: [], collecting: true }`
2. User kirim file `.txt` satu per satu (minimal 2 file)
3. User ketik `/done`
4. Bot gabungkan semua konten, **hapus duplikat baris/nomor**
5. Kirim satu file `.txt` gabungan

**Aturan deduplication**:
```javascript
const seenLines = new Set();
const merged = [];
for (const line of allLines) {
    const normalized = normalizePhone(line.trim()) || line.trim().toLowerCase();
    if (!normalized || seenLines.has(normalized)) continue;
    seenLines.add(normalized);
    merged.push(line.trim());
}
```

**Output info**:
```
📄 HASIL GABUNG TXT
━━━━━━━━━━━━━━━
📁 File digabung : 3 file
📝 Total baris masuk : 1.500
🚫 Duplikat dihapus : 234
✅ Baris unik : 1.266
━━━━━━━━━━━━━━━
```

---

### 10. `/gabungvcf` — Gabung Multiple VCF jadi Satu

**Trigger**: `/gabungvcf` lalu user kirim beberapa file `.vcf`, akhiri dengan `/done`

**Alur**: Sama seperti `/gabungtxt` tapi untuk VCF

**Aturan deduplication**: Berdasarkan normalisasi nomor telepon — satu nomor hanya muncul sekali dalam output

**Output**: Satu file `.vcf` gabungan

---

### 11. `/pecahfile` — Pecah VCF Jadi Beberapa Bagian

**Trigger**: `/pecahfile` lalu kirim file `.vcf`

**Alur**:
1. Command set state waiting
2. User kirim file `.vcf`
3. Bot baca, hitung total kontak
4. Bot tanya: berapa bagian? (inline keyboard: 2, 3, 4, 5, atau custom)
5. Bot pecah file menjadi N bagian yang **sama rata** (bagian terakhir mungkin lebih sedikit)
6. Bot kirim semua file hasil pecahan

**Penamaan file output**: `[nama_asli]_part1.vcf`, `[nama_asli]_part2.vcf`, dst

**Contoh**: 300 kontak ÷ 3 bagian = 100 kontak per file

---

### 12. `/pecahctc` — Pecah VCF Sesuai Jumlah Kontak

**Trigger**: `/pecahctc [jumlah]` lalu kirim file `.vcf`

**Contoh**: `/pecahctc 50` → pecah jadi file-file berisi 50 kontak masing-masing

**Alur**:
1. Parse `jumlah` dari command (default 100 jika tidak ada)
2. Validasi: 1 ≤ jumlah ≤ 1000
3. Set state waiting
4. User kirim file `.vcf`
5. Bot pecah berdasarkan jumlah per file
6. Kirim semua file hasil pecahan

**Penamaan**: `[nama]_001.vcf`, `[nama]_002.vcf`, dst (zero-padded)

---

### 13. `/addctc` — Tambah Kontak ke File VCF

**Trigger**: `/addctc` lalu kirim file `.vcf`

**Alur**:
1. User kirim file `.vcf` awal
2. Bot tampilkan jumlah kontak yang ada
3. Bot minta user kirim kontak tambahan dalam format teks (satu per baris):
   ```
   Nama Baru|081234567890
   081987654321
   +628123456789|Nama Lain
   ```
4. Bot tambahkan kontak baru ke VCF, **cek duplikat** dengan kontak yang sudah ada
5. Kirim VCF baru yang sudah ditambahkan

**State**: `{ mode: 'addctc', phase: 'waiting_vcf'|'waiting_contacts', vcfData: [...] }`

---

### 14. `/delctc` — Hapus Kontak di File VCF

**Trigger**: `/delctc` lalu kirim file `.vcf`

**Alur**:
1. User kirim file `.vcf`
2. Bot tampilkan daftar kontak (max 50 pertama, dengan nomor urut)
3. Bot minta input nomor urut yang ingin dihapus, format: `1,3,5-8,10`
4. Parse range dan individual: `1,3,5-8,10` → hapus kontak 1, 3, 5, 6, 7, 8, 10
5. Bot kirim VCF baru tanpa kontak yang dihapus

**State**: `{ mode: 'delctc', phase: 'waiting_vcf'|'waiting_input', contacts: [...] }`

---

### 15. `/hitungctc` — Hitung Total Kontak dalam VCF

**Trigger**: `/hitungctc` lalu kirim file `.vcf`

**Alur**:
1. Set state waiting
2. User kirim file `.vcf`
3. Bot parse dan hitung:
   - Total kontak
   - Kontak dengan nama (FN tidak kosong)
   - Kontak tanpa nama
   - Total nomor unik
   - Duplikat yang ditemukan

**Output**:
```
🔢 HASIL HITUNG KONTAK VCF
━━━━━━━━━━━━━━━
📇 File : namafile.vcf
👤 Total kontak : 500
✅ Punya nama : 430
❓ Tanpa nama : 70
📞 Nomor unik : 490
🚫 Nomor duplikat : 10
━━━━━━━━━━━━━━━
```

---

### 16. `/totxt` — Simpan Pesan ke File TXT

**Trigger**: `/totxt` — bot masuk mode "collecting messages"

**Alur**:
1. Command aktifkan mode: `{ mode: 'totxt', messages: [], active: true }`
2. Setiap pesan teks yang dikirim user setelah itu **ditambahkan ke array**
3. User ketik `/selesai` atau `/done` untuk generate file
4. Bot generate file `.txt` dengan semua pesan yang dikumpulkan (satu pesan per baris)
5. Kirim file ke user

**Info saat mode aktif**: Setiap pesan diterima, bot balas: `✅ Pesan ke-[N] disimpan. Ketik /selesai untuk generate file.`

**Limit**: Maksimal 500 pesan atau 1MB

---

### 17. `/listgc` — List Semua Grup WhatsApp

**Trigger**: `/listgc`

**Akses**: Hanya user yang sudah login WA (`requireAccess` + cek session.loggedIn)

**Alur**:
1. Fetch semua grup dari WA session: `session.sock.groupFetchAllParticipating()`
2. Urutkan berdasarkan jumlah member (terbesar dulu)
3. Generate output dalam dua format:
   - **Tampilan di chat**: Tabel teks dengan nomor, nama, jumlah member
   - **File TXT**: Untuk download jika grup banyak (>20)

**Output di chat** (jika ≤ 20 grup):
```
📋 DAFTAR GRUP WA
━━━━━━━━━━━━━━━
No | Nama Grup              | Member
━━━━━━━━━━━━━━━
1  | Arisan RT 05           | 150
2  | Info Desa Sukamaju     | 98
...
━━━━━━━━━━━━━━━
Total: 20 grup
```

**Output file** (jika >20 grup): Generate `list_grup.txt` dan kirim file

---

## STATE MANAGEMENT — HANDLER TERPUSAT

Tambahkan sistem state management terpusat untuk semua fitur baru di atas:

```javascript
// Global state map untuk semua fitur baru
const userStates = new Map(); // { userId: { mode, ...data, expiresAt } }

// Cleanup state yang expired (setiap 10 menit)
setInterval(() => {
    const now = Date.now();
    for (const [uid, state] of userStates.entries()) {
        if (state.expiresAt && now > state.expiresAt) {
            userStates.delete(uid);
        }
    }
}, 10 * 60 * 1000);

// Helper set state dengan auto-expire 10 menit
function setState(userId, data) {
    userStates.set(userId, { ...data, expiresAt: Date.now() + 10 * 60 * 1000 });
}

function getState(userId) {
    return userStates.get(userId) || null;
}

function clearState(userId) {
    userStates.delete(userId);
}
```

---

## HANDLER DOKUMEN TERPUSAT (UPGRADE)

Upgrade `tgBot.on('document', ...)` yang sudah ada menjadi handler terpusat yang bisa handle semua mode baru:

```javascript
tgBot.on('document', requireAccess, async (ctx) => {
    const userId = ctx.from.id;
    
    // Cek state baru dulu
    const state = getState(userId);
    if (state) {
        switch (state.mode) {
            case 'cv_txt_to_vcf': return handleCvTxtToVcfFile(ctx, userId, state);
            case 'cv_vcf_to_txt': return handleCvVcfToTxtFile(ctx, userId, state);
            case 'cv_xlsx_to_vcf': return handleCvXlsxToVcfFile(ctx, userId, state);
            case 'txt2vcf': return handleTxt2VcfFile(ctx, userId, state);
            case 'gabungtxt': return handleGabungTxtFile(ctx, userId, state);
            case 'gabungvcf': return handleGabungVcfFile(ctx, userId, state);
            case 'pecahfile': return handlePecahFileVcf(ctx, userId, state);
            case 'pecahctc': return handlePecahCtcFile(ctx, userId, state);
            case 'addctc': return handleAddCtcFile(ctx, userId, state);
            case 'delctc': return handleDelCtcFile(ctx, userId, state);
            case 'hitungctc': return handleHitungCtcFile(ctx, userId, state);
            case 'renamectc': return handleRenamectcFile(ctx, userId, state);
            case 'renamefile': return handleRenameFile(ctx, userId, state);
            case 'cvadminfile_upload': return handleCvAdminFileUpload(ctx, userId, state);
        }
    }
    
    // Fallback ke handler VCF lama (importvcf ke WA)
    const pending = vcfPending.get(userId);
    if (!pending || !pending.waitingFile) return;
    // ... kode lama tetap di sini ...
});
```

---

## HANDLER `/done` DAN `/selesai`

Tambahkan command handler untuk mengakhiri mode collecting:

```javascript
tgBot.command(['done', 'selesai'], requireAccess, async (ctx) => {
    const userId = ctx.from.id;
    const state = getState(userId);
    if (!state) return safeReply(ctx, '❌ Tidak ada proses yang sedang berjalan.');
    
    switch (state.mode) {
        case 'cv_txt_to_vcf': return finalizeCvTxtToVcf(ctx, userId, state);
        case 'cv_vcf_to_txt': return finalizeCvVcfToTxt(ctx, userId, state);
        case 'gabungtxt': return finalizeGabungTxt(ctx, userId, state);
        case 'gabungvcf': return finalizeGabungVcf(ctx, userId, state);
        case 'totxt': return finalizeTotxt(ctx, userId, state);
        default:
            clearState(userId);
            return safeReply(ctx, '✅ Proses dibatalkan.');
    }
});
```

---

## UTILITAS FILE — FUNGSI PEMBANTU BARU

Tambahkan fungsi-fungsi berikut (jika belum ada):

```javascript
// Download file dari Telegram ke buffer
async function downloadTelegramFile(ctx, fileId) {
    const fileLink = await ctx.telegram.getFileLink(fileId);
    const resp = await fetch(fileLink.href);
    return Buffer.from(await resp.arrayBuffer());
}

// Kirim file ke user
async function sendFile(ctx, buffer, filename, caption = '') {
    await ctx.replyWithDocument(
        { source: buffer, filename },
        caption ? { caption } : {}
    );
}

// Generate nama file output yang aman
function safeFilename(name) {
    return name.replace(/[\/\\:*?"<>|]/g, '_').substring(0, 100);
}

// Generate VCF content dari array kontak
function generateVCF(contacts) {
    return contacts.map(({ name, phone }) =>
        `BEGIN:VCARD\nVERSION:3.0\nFN:${name}\nTEL;TYPE=CELL:+${phone}\nEND:VCARD`
    ).join('\n');
}

// Parse TXT lines ke array kontak dengan auto-detect
function parseTxtLines(text) {
    const lines = text.split(/\r?\n/);
    const contacts = [];
    const seen = new Set();
    for (const line of lines) {
        const parsed = autoDetectAndParse(line);
        if (!parsed) continue;
        const norm = normalizePhone(parsed.phone);
        if (!norm || seen.has(norm)) continue;
        seen.add(norm);
        contacts.push({ name: parsed.name, phone: norm });
    }
    return contacts;
}
```

---

## ENVIRONMENT VARIABLE TAMBAHAN (opsional)

Tambahkan ke `.env` jika diperlukan:
```env
MAX_FILE_SIZE_MB=10          # Ukuran maksimal file upload (default 10MB)
MAX_FILES_PER_BATCH=20       # Jumlah file maksimal per batch (default 20)
ADMIN_FILES_DIR=./data/admin_files   # Direktori file admin
```

---

## CATATAN IMPLEMENTASI

1. **Semua fitur baru harus melewati `requireAccess`** middleware yang sudah ada
2. **Admin (`isAdmin`) bisa bypass semua pembatasan trial**
3. **Semua file temporary harus dihapus** setelah dikirim ke user (gunakan `finally` block)
4. **Pesan error harus informatif** dan gunakan `safeReply` agar tidak crash
5. **State harus di-clear** setelah proses selesai atau error
6. **Anti-duplikat adalah PRIORITAS UTAMA** — tidak ada nomor yang sama dalam satu file output
7. **Gunakan `normalizePhone()` yang sudah ada** untuk semua normalisasi nomor
8. **Gunakan `parseVCF()` yang sudah ada** untuk semua parsing VCF
9. **Jangan install package baru** kecuali `xlsx` (untuk `/cv_xlsx_to_vcf`) dan package sudah tersedia di environment
10. **Pertahankan pattern coding yang sudah ada**: async/await, log(), esc(), safeReply()

---

## URUTAN IMPLEMENTASI YANG DISARANKAN

1. Tambahkan `userStates` Map dan helper `setState/getState/clearState`
2. Tambahkan fungsi utility baru (`downloadTelegramFile`, `sendFile`, `generateVCF`, dll)
3. Upgrade `tgBot.on('document', ...)` menjadi handler terpusat
4. Tambahkan command `/done` dan `/selesai`
5. Implementasi satu per satu fitur baru, mulai dari yang paling sederhana:
   - `/hitungctc` (baca saja, tidak ada output baru)
   - `/cv_vcf_to_txt` (gunakan `parseVCF` yang ada)
   - `/gabungvcf` (gabungan dari parsing)
   - `/pecahfile` dan `/pecahctc`
   - `/cv_txt_to_vcf` dan `/txt2vcf`
   - `/gabungtxt`
   - `/renamectc` dan `/renamefile`
   - `/addctc` dan `/delctc`
   - `/cv_xlsx_to_vcf` (perlu package xlsx)
   - `/totxt`
   - `/listgc`
   - `/rekapgroup`
   - `/cvadminfile`

---

*Prompt ini dibuat untuk upgrade bot_badak__95__.js — semua fitur baru diintegrasikan ke dalam satu file yang sama tanpa menghapus fitur yang sudah ada.*

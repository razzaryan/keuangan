# Keuangan

Pencatatan keuangan pribadi dalam Rupiah. Input ditulis seperti chat biasa,
lalu dipecah menjadi nominal, tanggal, jenis, dan kategori.

```
kopi 18rb        -> keluar 18.000, kategori Makan
gaji 5jt         -> masuk 5.000.000, kategori Gaji
kemarin bensin 25.000
makan 35k @Makan    (kategori dipaksa)
```

Ada dua cara memakainya, keduanya memakai database yang sama:

| Antarmuka | Panduan |
|---|---|
| Aplikasi Windows (PySide6), ada installer dan uninstaller | [DESKTOP.md](DESKTOP.md) |
| Bot Telegram | [TUTORIAL.md](TUTORIAL.md) |

Logika pencatatan ada di `bot/` dan dipakai oleh keduanya, jadi hasil parse
dan tebakan kategori tidak berbeda antara aplikasi dan bot.

## Isi repo

```
bot/          mesin pencatatan
  parser.py     teks bebas -> nominal, tanggal, jenis, kategori
  ledger.py     query saldo, laporan, anggaran, export
  entry.py      alur parse -> simpan, dipakai dua antarmuka
  csvsync.py    tulis file CSV
  handlers/     handler bot Telegram
desktop/      aplikasi Windows (Qt)
  service.py    semua operasi database, tanpa Qt
  pages/        satu file per halaman
packaging/    spec PyInstaller, skrip Inno Setup, ikon, panduan
tests/        288 tes
```

## Aplikasi Windows

Jalankan dari source:

```powershell
python -m venv .venv
.\.venv\Scripts\python.exe -m pip install -r requirements.txt
.\.venv\Scripts\python.exe -m desktop
```

Atau pakai installer di `dist\installer\Keuangan-Setup-1.0.0.exe`.
Cara build sendiri ada di [DESKTOP.md](DESKTOP.md).

## Bot Telegram

Membuat bot, mengisi token, dan menjalankannya dibahas lengkap di
[TUTORIAL.md](TUTORIAL.md). Ringkasnya:

```powershell
Copy-Item .env.example .env
notepad .env          # isi BOT_TOKEN dan TELEGRAM_OWNER_ID
.\.venv\Scripts\python.exe -m bot.main
```

Setelah bot jalan, kirim `/start`.

## Konfigurasi bot

| Variabel | Default | Keterangan |
|---|---|---|
| `BOT_TOKEN` | wajib | token dari @BotFather |
| `TELEGRAM_OWNER_ID` | wajib | user ID yang boleh memakai bot, bisa lebih dari satu (pisah koma) |
| `TIMEZONE` | `Asia/Jakarta` | zona waktu untuk tanggal dan rekap |
| `RECAP_HOUR` | `21` | jam kirim rekap harian, `0` untuk mematikan |
| `DB_PATH` | `data/finance.db` | lokasi database |
| `DATA_DIR` | `data` | folder backup dan export |
| `BACKUP_ENABLED` | `true` | backup harian jam 03:00 |
| `BACKUP_KEEP` | `7` | jumlah backup yang disimpan |
| `DEBUG` | `false` | log verbose |

Bot tidak mau start kalau `TELEGRAM_OWNER_ID` kosong. Tanpa itu siapa pun yang
menemukan bot bisa menulis ke database.

## Perintah bot

Mencatat:

```
kopi 18rb                  teks bebas, paling cepat
makan 35k @Makan           paksa kategori
kemarin bensin 25000       tanggal mundur
+500rb jual motor          paksa pemasukan
-100rb donasi              paksa pengeluaran
/keluar 25000 makan siang
/masuk 5000000 gaji oktober
```

Melihat:

```
/saldo
/laporan
/laporan 2026-09
/laporan september
/laporan bulan lalu
/rekap
/terakhir 5
/cari kopi
/histori
```

Merapikan:

```
/undo
/hapus 42
/kategori
/kategori tambah keluar Kopi kopi,latte,espresso
/kategori kw Kopi kopi,latte
/kategori hapus Kopi
/budget set Makan 1500000
/budget
/budget hapus Makan
/export
/ledger
```

`/export` mengirim CSV satu bulan. `/ledger` mengirim CSV semua transaksi.

## File CSV

Setiap transaksi ikut ditulis ke `data/exports/ledger-<id>.csv` (bot) atau
`%APPDATA%\Keuangan\exports\` (aplikasi), jadi bisa dibuka di Excel tanpa
menjalankan export. Setelah hapus, barisnya ikut hilang.

Sel yang diawali `=`, `+`, `-`, atau `@` diberi tanda kutip di depan supaya
Excel tidak menjalankannya sebagai formula.

## Format nominal dan tanggal

Nominal: `18rb`, `18 ribu`, `18k`, `5jt`, `5 juta`, `1,5jt`, `2.5jt`,
`25.000`, `Rp 25.000`, `kopi18rb`, `+500rb`, `-100rb`.

Tanggal: `kemarin`, `kmrn`, `2 hari lalu`, `kemarin lusa`, `15/10`,
`15/10/2026`, `2026-10-15`.

Kategori ditebak dari kata kunci yang tersimpan per kategori. Kalau tidak ada
yang cocok, transaksi masuk ke "Lain-lain" (keluar) atau "Lainnya" (masuk).

## Tes

```powershell
.\.venv\Scripts\python.exe -m pytest
```

Ada 288 tes. Bagian bot memakai dispatcher aiogram sungguhan dengan session
Telegram palsu, jadi alur pesan masuk sampai balasan ikut diuji. Bagian
desktop dites tanpa membuka jendela (`QT_QPA_PLATFORM=offscreen`).

Skrip tambahan:

```powershell
python scripts/qa_probe.py            # parser dan nominal
python scripts/qa_probe_db.py         # konsistensi angka di database
python scripts/qa_probe_message.py    # HTML valid, batas 4096 karakter
python scripts/qa_probe_kategori.py   # akurasi tebakan kategori
python scripts/qa_probe_robust.py     # pesan beruntun, database terkunci
python scripts/qa_probe_boot_real.py  # boot dan shutdown bot
python scripts/desktop_smoke.py       # semua halaman aplikasi
python scripts/probe_exe.py           # exe hasil build
python scripts/probe_installer.py     # install, update, uninstall
```

## Menjalankan bot terus-menerus

Di Windows, pakai Task Scheduler:

```powershell
schtasks /create /tn "KeuanganBot" /tr "C:\path\keuangan\.venv\Scripts\python.exe -m bot.main" /sc onstart /ru SYSTEM
```

Komputer harus menyala dan tidak sleep.

Di VPS, pakai systemd. Buat `/etc/systemd/system/keuangan-bot.service`:

```ini
[Unit]
Description=Keuangan Telegram Bot
After=network-online.target

[Service]
Type=simple
User=financebot
WorkingDirectory=/opt/keuangan
ExecStart=/opt/keuangan/.venv/bin/python -m bot.main
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now keuangan-bot
sudo journalctl -u keuangan-bot -f
```

## Backup

Bot membuat backup harian ke `data/backups/` dan menyimpan 7 file terakhir.
Backup manual:

```powershell
.\scripts\backup.ps1
```

Restore, setelah bot dihentikan:

```powershell
Copy-Item data\backups\finance-20261020-030000.db data\finance.db -Force
```

File `.db` berisi seluruh riwayat keuangan. Jangan di-commit; sudah masuk
`.gitignore`.

## Masalah yang sering muncul

| Gejala | Penyebab |
|---|---|
| `BOT_TOKEN belum diisi` | `.env` belum dibuat |
| `Format BOT_TOKEN tidak valid` | ada spasi atau tanda kutip di nilai token |
| `TELEGRAM_OWNER_ID wajib diisi` | belum diisi, ambil dari `getUpdates` |
| Bot diam | prosesnya mati, atau ada dua instance berjalan |
| `Conflict: terminated by other getUpdates` | dua bot memakai token yang sama |
| Akses ditolak | user ID tidak cocok dengan `TELEGRAM_OWNER_ID` |

## Lisensi

MIT.

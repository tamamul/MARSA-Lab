# MARSA Lab — Tahap 1 (Kontrol komputer)

Arsitektur tetap sederhana: **PC meja guru menjalankan MarsaLab.Admin**, sedangkan setiap PC lab menjalankan **MarsaLab.Client**. Cocok untuk jaringan lokal 21–40 komputer.

## Fitur
- Daftar PC online/offline, alamat IP, dan waktu heartbeat terakhir.
- Pengumuman ke satu PC atau seluruh PC online.
- Kunci layar PC target (client harus berjalan di sesi pengguna Windows aktif).
- Restart atau shutdown terjadwal 60 detik dengan pemberitahuan Windows.
- Upaya membatalkan shutdown/restart yang masih terjadwal.
- Riwayat perintah ditulis ke `commands.jsonl`; daftar komputer disimpan ke `clients.json`.
- Endpoint perintah dibatasi pada allow-list; tidak ada endpoint untuk menjalankan shell command bebas.

## Persyaratan
- .NET SDK 10 untuk build dari source.
- Windows 10/11 pada PC admin dan client.
- PC admin memiliki IP tetap, contoh `192.168.0.10`.
- Semua PC berada di jaringan yang saling terhubung.

## Konfigurasi token
Buat token acak minimal 24 karakter, misalnya dengan PowerShell:

```powershell
-join ((1..40 | ForEach-Object { [char](Get-Random -Minimum 33 -Maximum 127) }))
```

Set token yang sama pada tiga tempat:
1. `src/MarsaLab.Admin/appsettings.json` → `Lab:Token`
2. `src/MarsaLab.Client/appsettings.json` → `Token`
3. `src/MarsaLab.Admin/wwwroot/index.html` → konstanta `TOKEN`

Jangan gunakan token contoh. Token dashboard saat ini berada pada JavaScript halaman, sehingga siapa pun yang dapat membuka halaman dashboard dapat melihatnya; batasi akses jaringan dan jangan port-forward server ke internet. Gunakan hanya pada LAN lab.

## Build dan jalankan
Dari folder `marsalab`:

```powershell
dotnet build .\src\MarsaLab.Admin\MarsaLab.Admin.csproj -c Release
dotnet build .\src\MarsaLab.Client\MarsaLab.Client.csproj -c Release
```

Jalankan server di PC meja guru:

```powershell
dotnet run --project .\src\MarsaLab.Admin\MarsaLab.Admin.csproj -c Release
```

Buka `http://192.168.0.10:5080` dari browser PC admin. Izinkan port TCP 5080 di Windows Firewall hanya pada profil Private/jaringan lab.

Atur `src/MarsaLab.Client/appsettings.json` pada setiap PC lab:
- `ServerUrl`: `http://192.168.0.10:5080`
- `Token`: token yang sama
- `ClientId`: kosong untuk memakai hostname, atau isi ID unik
- `HeartbeatSeconds`: 5

Jalankan client dari sesi pengguna Windows, bukan sebagai service Session 0, supaya `msg.exe` dan kunci layar dapat berinteraksi dengan desktop. Uji dulu pada satu PC sebelum menyebarkan ke seluruh lab.

## Catatan keamanan dan batasan
- Restart/shutdown memberi waktu 60 detik dan peringatan Windows; tidak melakukan tindakan diam-diam.
- `cancel-shutdown` hanya berhasil jika Windows masih memiliki shutdown/restart terjadwal dan akun memiliki izin.
- Kunci layar mengunci sesi; pembukaan kunci tetap dilakukan pengguna dengan kredensial Windows.
- Hak administrator mungkin diperlukan untuk perintah daya tergantung kebijakan Windows.
- Server menggunakan HTTP di LAN, bukan HTTPS. Jangan gunakan di jaringan publik atau meneruskan port ke internet. Untuk lingkungan lebih ketat, tambahkan autentikasi admin terpisah, HTTPS, dan pembatasan firewall.
- Riwayat perintah yang tampil di dashboard berada di memori dan akan kosong saat server restart; salinan audit JSONL tetap tersimpan pada folder data lokal admin. Status komputer disimpan di `%LOCALAPPDATA%\MarsaLab\Admin\clients.json`.

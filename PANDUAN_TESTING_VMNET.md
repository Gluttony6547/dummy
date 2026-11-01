# Panduan Testing Tiga IP — Windows + Linux VM (VMware NAT)

Panduan ini menjalankan **Tugas 2 (KDC + AES manual)** dengan **tiga alamat IP berbeda** pada
perangkat yang benar-benar Anda punya: satu Windows dan satu Linux VM di jaringan VMware NAT.
Tidak memakai `127.0.0.x`; ketiga alamat berada di subnet `192.168.207.0/24` sehingga
trafiknya melewati adapter virtual dan bisa ditangkap Wireshark.

| Entitas | Mesin | Terminal | IP | Port | Perintah |
|---|---|---|---|---|---|
| **KDC** | Windows | Terminal A (Command Prompt / PowerShell) | `192.168.207.1` | 5000 | `python kdc.py` |
| **Alice** | Linux VM | Terminal B (`Ctrl+Alt+T`) | `192.168.207.128` | ephemeral (IP sumber) | `python3 node.py --peran alice` |
| **Bob** | Linux VM | Terminal C (`Ctrl+Alt+T`) | `192.168.207.130` | 5002 | `python3 node.py --peran bob` |

Ketentuan tugas meminta **tiga IP address berbeda**, dan itu dipenuhi di sini: `192.168.207.1`
adapter VMnet8 di Windows, `192.168.207.128` alamat utama `ens33` di VM, `192.168.207.130`
alamat tambahan di `ens33` yang sama untuk Bob.

> **Jujur soal jumlah perangkat.** Ini **dua perangkat fisik**, bukan tiga. Alice dan Bob
> berbagi satu VM, hanya berbeda alamat. Kalau ingin benar-benar tiga host, lihat bagian 7
> (klon VM) — itu jalur paling kuat untuk demo, dan panduan ini tetap berlaku dengan
> mengganti `192.168.207.130` menjadi IP clone.

---

## 1. Persiapan

### 1.1 Alamat yang dipakai

Dari `ipconfig` (Windows) dan `ip a` (VM):

| Sumber | Alamat | Keterangan |
|---|---|---|
| Windows, adapter `VMware Network Adapter VMnet8` | `192.168.207.1` | dipakai KDC untuk mendengarkan |
| Linux `ens33` | `192.168.207.128/24` | dipakai Alice sebagai IP sumber |
| Linux `ens33`, alamat tambahan | `192.168.207.130/24` | dipakai Bob sebagai listener |

Wi-Fi `192.168.1.4` dan VMnet1 `192.168.188.1` **tidak dipakai**. Jangan dicampur: mengirim
dari IP Wi-Fi ke tujuan VMnet8, atau dari loopback ke VMnet8, gagal di Windows dengan
`[WinError 10051] A socket operation was attempted to an unreachable network`. Semua entitas
harus satu subnet.

### 1.2 Pastikan dua sisi saling menjangkau

Di Windows:

```bat
ping 192.168.207.128
```

Di VM:

```bash
ping -c 3 192.168.207.1
```

`192.168.207.1` adalah alamat host di jaringan NAT, jadi VM bisa menjangkaunya langsung tanpa
port forwarding. Kalau ping gagal, jangan lanjut — perbaiki dulu (VM harus mode NAT/VMnet8,
bukan Host-only).

### 1.3 Tambah alamat kedua di VM untuk Bob

Di terminal VM:

```bash
sudo ip addr add 192.168.207.130/24 dev ens33
ip -4 addr show ens33
```

Baris kedua harus menampilkan **dua** `inet`: `192.168.207.128` dan `192.168.207.130`.
Alamat ini sementara — hilang saat VM reboot, jadi ulangi setiap kali menguji. Untuk
membuatnya permanen: `sudo nmtui` → Edit a connection → `ens33` → IPv4 → tambah
`192.168.207.130/24` → OK.

### 1.4 Berkas konfigurasi tiga IP

**Jangan ubah `konfigurasi.json` di repo.** Berkas itu sengaja tetap berisi tiga alamat
loopback supaya unit test `UjiKonfigurasi` (yang mewajibkan tiga host berbeda) tetap lulus
20/20. Buat berkas terpisah di luar repo, lalu salin ke VM.

Windows — simpan sebagai `C:\SMT_7\Keamanan Informasi\konfigurasi_vmnet.json`:

```json
{
  "kdc": {
    "host": "192.168.207.1",
    "port": 5000,
    "kunci_master": {
      "alice": "KunciAliceKI2026",
      "bob": "KunciBobKI2026XX"
    }
  },
  "alice": {
    "id": "alice",
    "host": "192.168.207.128",
    "kunci_master": "KunciAliceKI2026"
  },
  "bob": {
    "id": "bob",
    "host": "192.168.207.130",
    "port": 5002,
    "kunci_master": "KunciBobKI2026XX"
  },
  "jaringan": { "timeout": 15.0 }
}
```

VM — simpan berkas yang **isinya identik** sebagai `/home/gluttony/konfigurasi_vmnet.json`.
Cara termudah: buat berkasnya sekali di VM dengan `nano konfigurasi_vmnet.json`, lalu paste
JSON di atas. Kalau VM dan Windows berbagi folder, salin saja berkasnya.

Arti tiap kunci (dibaca di `node.py` dan `kdc.py`, jangan tertukar):

| Kunci | Dipakai oleh | Fungsinya |
|---|---|---|
| `kdc.host` / `kdc.port` | Alice & Bob | alamat KDC yang **didial** |
| `alice.host` | Alice | **IP sumber** koneksi keluar Alice (`--bind` Alice) — harus IP mesin tempat Alice jalan |
| `bob.host` | Alice | alamat Bob yang **didial** saat LANGKAH 3 |
| `bob.port` | Bob | port dengarkan, dipakai bila `--port` tidak diberikan |
| `kunci_master.alice` / `.bob` | KDC | kunci master yang dipegang KDC; **tidak pernah** dikirim ke jaringan |
| `alice.kunci_master` / `bob.kunci_master` | node | harus sama persis dengan yang dipegang KDC |
| `jaringan.timeout` | semua | batas waktu socket (detik) |

Dua hal yang sering bikin gagal:

1. `bob.host` **bukan** alamat bind Bob. Bob selalu butuh `--bind` sendiri; tanpa itu ia
   mendengarkan di semua interface (`[SIAP] bob mendengarkan di 0.0.0.0:5002`), sehingga
   bukti "Bob punya IP sendiri" hilang dari baris `[DENGAR]`/`[SIAP]` — `/status` hanya akan
   menampilkan IP tujuan yang kebetulan dipakai koneksi masuk.
2. `alice.host` harus alamat mesin tempat Alice dijalankan. Kalau salah, program berhenti di
   LANGKAH 1 dengan `[WinError 10049] The requested address is not valid in its context`.

### 1.5 Firewall

Di Windows (Command Prompt **as Administrator**), izinkan KDC dan — bila Bob di Windows — port Bob:

```bat
netsh advfirewall firewall add rule name="KDC AES 5000" dir=in action=allow protocol=TCP localport=5000 profile=any
netsh advfirewall firewall add rule name="Node AES 5002" dir=in action=allow protocol=TCP localport=5002 profile=any
```

Adapter VMnet8 biasanya terdeteksi sebagai jaringan "Public", karena itu `profile=any` penting —
tanpa itu koneksi dari VM akan timeout.

Di VM:

```bash
sudo ufw status
sudo ufw allow 5002/tcp
```

Kalau `ufw` tidak aktif (bawaan Linux Mint), lewati bagian ini.

---

## 2. Menjalankan Program

Urutan wajib: **KDC → Bob → Alice**. Alice yang memicu handshake, jadi Bob harus siap lebih dulu.

### Terminal A — KDC di Windows

```bat
cd "C:\SMT_7\Keamanan Informasi\Tugas 2 Keamanan Informasi"
python kdc.py --konfigurasi "C:\SMT_7\Keamanan Informasi\konfigurasi_vmnet.json" --bind 192.168.207.1
```

Yang harus muncul:

```text
[INFO] Kunci master dimuat untuk: alice, bob
[MENDENGAR] 192.168.207.1:5000
[KUNCI]     alice  = KunciAliceKI2026 (16 ASCII)
[KUNCI]     bob    = KunciBobKI2026XX (16 ASCII)
```

Prompt berubah menjadi `kdc> ` dan bisa menerima `/sesi`, `/kunci`, `/bantu`, `/keluar`.

### Terminal C — Bob di VM

```bash
cd ~/Tugas2    # folder hasil clone repo
python3 node.py --peran bob --konfigurasi /home/gluttony/konfigurasi_vmnet.json \
  --bind 192.168.207.130 --port 5002
```

Yang harus muncul:

```text
[DENGAR]  192.168.207.130:5002
[SIAP] bob mendengarkan di 192.168.207.130:5002
```

Kalau tidak muncul dan ada `[ERROR] Gagal mendengarkan ...`, berarti alamat `192.168.207.130`
belum ditambahkan ke `ens33` (ulangi bagian 1.3). Program keluar dengan **exit code 2**.

### Terminal B — Alice di VM

```bash
python3 node.py --peran alice --konfigurasi /home/gluttony/konfigurasi_vmnet.json
```

Alice mencetak `[KDC] 192.168.207.1:5000`, `[LAWAN] bob di 192.168.207.130:5002`, lalu lima langkah:

```text
[LANGKAH 1] alice -> KDC : ID_A=alice ID_B=bob N1=...
[LANGKAH 2] KDC -> alice : sesi=S-001
[LANGKAH 3] alice -> bob : kirim tiket E_KB[K_S, ID_A]
[LANGKAH 4] bob -> alice : E_KS[N2] (HEX) = ...
[LANGKAH 5] alice -> bob : E_KS[N2-1] (HEX) = ...
[SESI] Terbentuk 21:07:44 dengan bob @ 192.168.207.130:5002
```

Setelah itu prompt `[alice] > ` aktif di kedua sisi.

---

## 3. Pengujian Manual Langkah demi Langkah

### 3.1 Pesan dua arah (Alice dan Bob sama-sama sender dan receiver)

Di terminal Alice:

```text
Halo Bob, ini Alice
```

Di terminal Bob:

```text
Halo Alice, pesan diterima
```

Yang diperiksa:

- Pengirim mencetak `[TERKIRIM] Ciphertext AES (HEX) : ...` dan `[TERKIRIM] Panjang ciphertext   : N byte`.
- Penerima mencetak `[DITERIMA #1] dari alice @ 192.168.207.128:xxxxx` berisi ciphertext yang
  **sama persis** dengan yang dikirim, plus `Plaintext : ...` hasil dekripsi.
- `/status` di kedua sisi: `[STATUS] lokal         : 192.168.207.130:5002` (Bob) dan
  `[STATUS] lokal         : 192.168.207.128:xxxxx` (Alice) — **dua IP berbeda terbukti di socket**,
  bukan sekadar di konfigurasi — serta `TX`/`RX` yang berpasangan.

### 3.2 Rotasi kunci tanpa restart

Dari Alice:

```text
/kunci
```

Urutan yang muncul:

```text
[KUNCI] Meminta sesi baru ke KDC...
[KUNCI] Tiket baru dikirim dengan kunci lama. K_S baru = ...
[KUNCI] Menunggu konfirmasi lawan sebelum memakai kunci baru...
[KUNCI] Lawan mengonfirmasi; kunci baru aktif di kedua sisi.
```

Bob mencetak `[KUNCI] Lawan mengganti kunci sesi. K_S baru = ...` lalu
`[KUNCI] Konfirmasi dikirim dengan kunci baru.`. Lanjutkan percakapan dan pastikan
`[STATUS] K_S (ASCII)` berubah di kedua sisi sementara `sesi KDC` naik menjadi `S-002`.

### 3.3 Kasus negatif (kunci master salah)

Tutup Alice dan Bob (`/keluar`), biarkan KDC jalan. Lalu:

```bash
python3 node.py --peran alice --konfigurasi /home/gluttony/konfigurasi_vmnet.json --kunci KunciSalahBanget
```

Hasil yang diharapkan (exit code **3**):

```text
[ERROR] Handshake gagal: gagal membuka balasan KDC dengan kunci master
```

### 3.4 Kasus negatif alamat (ini yang paling sering kejadian)

Bob dijalankan dengan alamat yang bukan milik mesinnya:

```bash
python3 node.py --peran bob --konfigurasi /home/gluttony/konfigurasi_vmnet.json --bind 192.168.207.131 --port 5002
```

Hasil (exit code **2**):

```text
[ERROR] Gagal mendengarkan di 192.168.207.131:5002: [Errno 99] Cannot assign requested address
```

Di Windows pesannya `[WinError 10049] The requested address is not valid in its context`,
dengan kode keluar yang sama.

### 3.5 Kasus pelengkap

- Pesan 200 karakter: `Panjang ciphertext` tumbuh mengikuti blok 16 byte, plaintext tetap utuh.
- Emoji/karakter non-ASCII: terkirim (plaintext UTF-8); kunci tetap wajib 16 karakter ASCII.
- Kirim dari sisi Bob lebih dulu, untuk membuktikan Bob juga sender.

---

## 4. Wireshark

### Di VM (disarankan, karena dua IP terbesar ada di sana)

```bash
sudo tshark -i any -f "tcp port 5000 or tcp port 5002" -w bukti/handshake_vmnet.pcapng
```

Tutup dengan `Ctrl+C`, lalu:

```bash
tshark -r bukti/handshake_vmnet.pcapng -Y "tcp.len > 0" -T fields -e frame.number -e ip.src -e tcp.srcport -e ip.dst -e tcp.dstport -e tcp.len
```

### Di Windows (Wireshark GUI)

1. Pilih interface **VMware Network Adapter VMnet8**, bukan Wi-Fi dan bukan loopback.
2. Capture filter: `tcp port 5000 or tcp port 5002`.
3. Start capture **sebelum** Alice dijalankan.

### Yang ditunjuk saat demo

Tiga kombinasi alamat harus terlihat jelas:

| Fase | Arah | Isi payload |
|---|---|---|
| LANGKAH 1 | `192.168.207.128:xxxxx → 192.168.207.1:5000` | JSON polos `{"id_a": "alice", "id_b": "bob", "n1": ...}` — ini memang tahap permintaan tiket, sesuai PPT tidak dienkripsi |
| LANGKAH 2 | `192.168.207.1:5000 → 192.168.207.128:xxxxx` | hanya `E_KA[...]` berupa hex, tanpa `K_S` terbaca |
| LANGKAH 3–5 + pesan | `192.168.207.128:xxxxx ↔ 192.168.207.130:5002` | hex kelipatan 16 byte; tidak ada teks pesan yang terbaca |

Titik bicara: "yang terlihat hanya amplop JSON dan blok hex; `K_S` dan isi pesan tidak pernah
muncul di jaringan, dan `E_KB[K_S, ID_A]` hanya bisa dibuka Bob."

---

## 5. Checklist bukti

- [ ] `ip -4 addr show ens33` di VM menunjukkan `192.168.207.128` **dan** `192.168.207.130`.
- [ ] `ping` dua arah Windows ↔ VM berhasil.
- [ ] KDC mencetak `[MENDENGAR] 192.168.207.1:5000`.
- [ ] Bob mencetak `[SIAP] bob mendengarkan di 192.168.207.130:5002`.
- [ ] Lima langkah muncul di Alice dan Bob; `[SESI] K_S` sama di kedua sisi.
- [ ] `/status` menunjukkan `lokal 192.168.207.128:...` dan `lokal 192.168.207.130:5002`.
- [ ] Pesan dua arah lulus, ciphertext identik di pengirim dan penerima.
- [ ] `/kunci` mengganti `K_S` tanpa restart.
- [ ] Kasus negatif: kunci master salah → exit 3; alamat bind salah → exit 2.
- [ ] `python -m unittest test_aes -v` tetap 20/20 (karena `konfigurasi.json` tidak diubah).
- [ ] File `.pcapng` + tangkapan layar `/status` disimpan.

Simpan semua keluaran terminal ke berkas, misalnya dengan `--log`:

```bash
python3 node.py --peran bob --konfigurasi /home/gluttony/konfigurasi_vmnet.json \
  --bind 192.168.207.130 --log bukti/vmnet_bob.log
```

---

## 6. Kalau gagal, cocokkan pesannya

| Pesan | Arti | Tindakan |
|---|---|---|
| `[WinError 10049] The requested address is not valid in its context` saat LANGKAH 1 | `alice.host` bukan IP mesin tempat Alice jalan | samakan `alice.host` dengan `ip a` |
| `[WinError 10051] ... unreachable network` | IP sumber dan tujuan beda subnet/interface | pakai satu subnet, jangan campur loopback/Wi-Fi/VMnet8 |
| `[ERROR] Gagal mendengarkan di ...` (exit 2) | alamat bind bukan milik mesin | tambahkan alamat di `ens33` atau ganti `--bind` |
| `[ERROR] Handshake gagal: gagal membuka balasan KDC dengan kunci master` (exit 3) | `kunci_master` node ≠ KDC | samakan 16 karakter ASCII |
| Diam sampai `timeout` di LANGKAH 1 | firewall Windows memblokir 5000 dari VM | jalankan aturan `netsh` dengan `profile=any` |
| `[TOLAK] ...: identitas tidak terdaftar` | `--id` tidak ada di `kunci_master` KDC | pakai `alice`/`bob` |

---

## 7. Naik ke tiga host sungguhan

Kalau ingin demo yang tidak bisa dibantah sebagai "dua mesin":

1. Matikan VM, clone di VMware (`VM → Manage → Clone`), pilih **I copied it**.
2. Clone akan mendapat `192.168.207.129` (atau alamat lain dari DHCP VMnet8); cek dengan `ip a`.
3. KDC tetap di Windows `192.168.207.1`, Alice di VM asli `192.168.207.128`, Bob di clone
   `192.168.207.129`.
4. Ubah `bob.host` di konfigurasi menjadi IP clone, lalu jalankan Bob di sana dengan `--bind <IP clone>`.

Trafik Alice↔Bob kini melewati virtual switch, jadi terlihat di Wireshark pada VMnet8.
Alternatif lain: set adapter VM ke **Bridged** agar VM mendapat `192.168.1.x` dan demo berjalan
di LAN Wi-Fi bersama `192.168.1.4` — tapi tetap butuh alamat ketiga (alias atau clone) untuk
memenuhi syarat tiga IP.

---

## 8. Ringkasan perintah (salin-tempel)

```bat
:: Windows — KDC
cd "C:\SMT_7\Keamanan Informasi\Tugas 2 Keamanan Informasi"
python kdc.py --konfigurasi "C:\SMT_7\Keamanan Informasi\konfigurasi_vmnet.json" --bind 192.168.207.1
```

```bash
# VM — siapkan alamat kedua lalu jalankan Bob
sudo ip addr add 192.168.207.130/24 dev ens33
python3 node.py --peran bob --konfigurasi /home/gluttony/konfigurasi_vmnet.json --bind 192.168.207.130 --port 5002

# VM — terminal kedua: Alice
python3 node.py --peran alice --konfigurasi /home/gluttony/konfigurasi_vmnet.json

# VM — capture (terminal ketiga)
sudo tshark -i any -f "tcp port 5000 or tcp port 5002" -w bukti/handshake_vmnet.pcapng
```

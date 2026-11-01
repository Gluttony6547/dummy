# Rangkaian Pengujian Tugas 2 — AES Manual + KDC (Alice–Bob)

| Identitas | |
|---|---|
| **Nama** | Naufal Dariskarim |
| **NRP** | 5025231027 |
| **Mata kuliah** | Keamanan Informasi |
| **Objek uji** | Implementasi AES-128 manual + KDC, tiga entitas (KDC/Alice/Bob) |
| **Acuan** | Instruksi Tugas 2 (ketentuan 1–12), `Instruction.txt` di repo |

> Dokumen ini disimpan **di luar repository** (`C:\SMT_7\Keamanan Informasi\`), jadi tidak ada file baru, commit, atau perubahan apa pun di repo. Untuk memasukkannya nanti:
> `move "C:\SMT_7\Keamanan Informasi\RANGKAIAN_TESTING_TUGAS2.md" "C:\SMT_7\Keamanan Informasi\Tugas 2 Keamanan Informasi\"`
>
> Semua perintah, nama flag, nama perintah konsol, dan teks keluaran di bawah diambil dari kode yang ada di disk (`aes_manual.py`, `kdc.py`, `node.py`, `protokol.py`, `konsol.py`, `sesi.py`, `jaringan.py`, `konfigurasi.py`, `test_aes.py`, `uji_end_to_end.py`), bukan dari perkiraan.

---

## 1. Ruang lingkup: ketentuan → kelompok uji

| Ketentuan tugas | Yang harus dibuktikan | Kelompok uji |
|---|---|---|
| 2. Tiga entitas, 3 IP berbeda, langkah 1–5, lalu AES | KDC/Alice/Bob pakai IP masing-masing; lima langkah terjadi; sesi AES terbentuk | **C**, **G** |
| 3. Dua arah (sender & receiver di kedua sisi) | Alice→Bob dan Bob→Alice terkirim & terdekripsi | **D** |
| 4. AES & KDC manual, key/plaintext ASCII, ciphertext hex, key bisa diubah | AES cocok vektor standar; tidak ada library cipher; kunci 16 ASCII; ciphertext hex | **A**, **I**, **E** |
| 5. Interaktif, key berubah tanpa restart | `/kunci`, `/status`, `/bantu`, `/keluar` bekerja pada program yang sama | **E**, **F** |
| 6. Wireshark | Transmisi teramati: envelope JSON terlihat, isi hanya ciphertext hex | **H** |
| 12. Fitur tambahan | Rotasi kunci sesi tanpa restart + registry sesi KDC | **E** |

---

## 2. Prasyarat

- Python 3.8+; proyek hanya memakai standard library (tidak ada `pip install`).
- Dijalankan dari folder `Tugas 2 Keamanan Informasi` (semua perintah di bawah mengasumsikan itu).
- Port 5000 (KDC) dan 5002 (Bob) bebas. Konfigurasi default: KDC `127.0.0.2:5000`, Bob `127.0.0.4:5002`, Alice IP sumber `127.0.0.3`.
- Kunci master default: `KunciAliceKI2026` dan `KunciBobKI2026XX` (masing-masing 16 karakter ASCII).
- Opsional untuk H: `tshark`/Wireshark dengan Npcap.

---

## 3. Eksekusi cepat (dua perintah, otomatis)

```bash
python -m unittest test_aes -v
python uji_end_to_end.py
```

| Perintah | Keluaran yang benar | Bukti |
|---|---|---|
| `python -m unittest test_aes -v` | `Ran 20 tests` … `OK`, exit 0 | `bukti/unit_aes_out.txt` |
| `python uji_end_to_end.py` | `[HASIL] LULUS: dasar, rotasi kunci, dan kasus negatif sesuai harapan.`, exit 0 | `bukti/e2e_hasil.txt` + `bukti/e2e/<stempel>/` |

`uji_end_to_end.py` membuat folder bukti baru `bukti/e2e/<YYYYMMDD-HHMMSS>/` berisi `dasar/`, `rotasi/`, `negatif_kunci_salah/` dengan `alice.out`, `bob.out`, `kdc.out`, `*.log`, dan `konfigurasi.json` per skenario. Exit 0 hanya bila semua pemeriksaan lulus.

Bukti yang **sudah ada** di repo:

| Bukti | Isi | Tanggal |
|---|---|---|
| `bukti/unit_aes_out.txt` | 20 test, `OK` | 27 Okt 2026 (dijalankan ulang saat review: 20 test, OK, exit 0) |
| `bukti/e2e_hasil.txt` | `[HASIL] LULUS`, menunjuk run `20261027-234923` | 27 Okt 2026 23.49 |
| `bukti/e2e/20261027-234923/` | 3 skenario lengkap | 27 Okt 2026 23.49 |
| `bukti/e2e/20261028-225452/` | 3 skenario (hasil per proses; ringkasan harness tidak disimpan) | 28 Okt 2026 22.54 |
| `bukti/konsol_interaktif.txt` | Transkrip konsol interaktif + exit 0/0 | 27 Okt 2026 23.51 |
| `bukti/pcap/handshake_chat.pcap` + `analisis.txt` | 32 paket, 9 segmen data, 3 IP | 27 Okt 2026 23.58 |

---

## 4. Matriks kasus uji

Kolom **Status**: `LULUS (tercatat)` = ada bukti di `bukti/`; `BELUM DICATAT` = kasus valid tapi belum pernah dijalankan/diarsipkan.

### 4.1 A — AES manual (otomatis, `test_aes.py`)

| ID | Yang diuji | Harapan | Status |
|---|---|---|---|
| A1 | `UjiVektorAES.test_fips_197_c1_enkripsi` | key `000102…0f`, pt `001122…ff` → ct `69c4e0d86a7b0430d8cdb78070b4c55a` | LULUS (tercatat) |
| A2 | `test_fips_197_c1_dekripsi` | ct di atas → pt semula | LULUS (tercatat) |
| A3 | `test_nist_sp800_38a_cbc` | 4 blok CBC = `7649abac…586e1a7` (dua arah) | LULUS (tercatat) |
| A4 | `UjiModeCBC` (5 test) | roundtrip panjang 0/1/15/16/17/31/32/33/1000, UTF-8, IV acak, panjang kelipatan 16, ciphertext tidak memuat plaintext | LULUS (tercatat) |
| A5 | `UjiValidasiKunci` (7 test) | kunci 16 ASCII diterima; kunci pendek/panjang/non-ASCII, blok ≠16 byte, ciphertext terpotong, padding rusak ditolak | LULUS (tercatat) |
| A6 | Tidak ada library cipher | `aes_manual.py` hanya list/byte + `os.urandom`; `kdc.py` memakai `ManualAES` untuk tiket/balasan | Diperiksa manual (grep import) |

### 4.2 B — KDC (otomatis `UjiKDC`, plus satu kasus manual)

| ID | Yang diuji | Harapan | Status |
|---|---|---|---|
| B1 | `test_langkah_2_berisi_sesi_dan_tiket` | balasan `E_KA` berisi `id_a`, `id_b`, `n1`, `ks` 16 char; tiket `E_KB` berisi `ks` yang sama | LULUS (tercatat) |
| B2 | `test_identitas_tidak_dikenal_ditolak` | `ValueError` untuk `mallory` | LULUS (tercatat) |
| B3 | `test_id_sama_ditolak` | `ValueError` bila `id_a == id_b` | LULUS (tercatat) |
| B4 | `test_kunci_sesi_berbeda_tiap_permintaan` | dua permintaan → dua `K_S` berbeda | LULUS (tercatat) |
| B5 | Tolak di level socket | KDC mencetak `[TOLAK] …` dan membalas `{"tipe": "error", …}` | BELUM DICATAT |

Perintah B5 (KDC harus sedang berjalan):

```python
# terminal terpisah, folder yang sama
import json, socket
from jaringan import kirim_json, terima_json
with socket.create_connection(("127.0.0.2", 5000), timeout=5) as s:
    kirim_json(s, {"tipe": "permintaan_kdc", "id_a": "mallory", "id_b": "bob", "n1": 1})
    print(terima_json(s))
```

Harapan: `{'tipe': 'error', 'pesan': "identitas tidak terdaftar: 'mallory' / 'bob'"}`; log KDC memuat `EVENT=TOLAK`.

### 4.3 C — Handshake lima langkah (manual, tiga proses)

| ID | Yang diuji | Harapan | Status |
|---|---|---|---|
| C1 | Langkah 1 (Alice→KDC, polos) | Alice `[LANGKAH 1] alice -> KDC : ID_A=alice ID_B=bob N1=<angka>`; KDC `[LANGKAH 1] alice -> KDC: …` dan `EVENT=LANGKAH1 PEER=127.0.0.3:<port>` | LULUS (tercatat) |
| C2 | Langkah 2 (`E_KA`) | KDC `[LANGKAH 2] KDC -> alice: sesi=S-001 K_S=<16 ASCII>`; Alice menerima, mencetak `[LANGKAH 2] KDC -> alice : sesi=S-001`, hex `E_KA[...]`, `K_S yang dibuka = …`, `Tiket untuk lawan (HEX) = …` | LULUS (tercatat) |
| C3 | Langkah 3 (tiket `E_KB`) | Alice `[LANGKAH 3] alice -> bob : kirim tiket E_KB[K_S, ID_A]`; Bob `[LANGKAH 3] diterima dari 127.0.0.3:<port>: E_KB[K_S, ID_A] (HEX) = …` | LULUS (tercatat) |
| C4 | Langkah 4 (`E_KS[N2]`) | Bob `[LANGKAH 4] bob -> alice : E_KS[N2] (HEX) = <64 hex>`; Alice mencetak N2 yang sama | LULUS (tercatat) |
| C5 | Langkah 5 (`E_KS[N2-1]`) + sesi | Alice `[LANGKAH 5] alice -> bob : E_KS[N2-1] (HEX) = <64 hex>`; Bob `[LANGKAH 5] diterima: …`; kedua sisi `[SESI] Terbentuk <HH:MM:SS> dengan …` + `[SESI] K_S = <16 ASCII>` | LULUS (tercatat) |

Catatan angka: `data` langkah 4/5 berisi **64 karakter hex = 32 byte** (IV 16 byte + 1 blok) karena isinya hanya angka nonce. Pada pcap: frame 17 = 101 byte, frame 19 = 102 byte.

### 4.4 D — Pesan dua arah

| ID | Yang diuji | Harapan | Status |
|---|---|---|---|
| D1 | Alice→Bob | Alice `[TERKIRIM] Ciphertext AES (HEX) : …`; Bob `[DITERIMA #1] dari alice @ 127.0.0.3:<port>` + `Plaintext            : <teks>` | LULUS (tercatat) |
| D2 | Bob→Alice | Bob mengirim, Alice `[DITERIMA #1] dari bob @ …` | LULUS (tercatat) |
| D3 | TX/RX berpasangan | `/status` kedua sisi: `TX=2 RX=2` (skenario `dasar`) / `TX=3 RX=3` (`rotasi`) | LULUS (tercatat) |
| D4 | Ciphertext TX = RX | 128 hex yang sama di kedua sisi; harness membandingkan `EVENT=TX` vs `EVENT=RX` dan mendekripsi ulang | LULUS (tercatat) |
| D5 | Pesan panjang (200 karakter) | ciphertext tumbuh per blok 16 byte, plaintext utuh | BELUM DICATAT |
| D6 | Plaintext tidak ada di wire | `frame contains "Pesan uji dari Alice"` → 0 hasil di pcap | BELUM DICATAT (bisa diuji pada pcap yang ada) |

### 4.5 E — Rotasi kunci sesi (fitur tambahan)

| ID | Yang diuji | Harapan | Status |
|---|---|---|---|
| E1 | `/kunci` di Alice | `[KUNCI] Meminta sesi baru ke KDC...`, `[KUNCI] Tiket baru dikirim dengan kunci lama. K_S baru = <16>`, `[KUNCI] Menunggu konfirmasi lawan…`, lalu `[KUNCI] Lawan mengonfirmasi; kunci baru aktif di kedua sisi.` | LULUS (tercatat) |
| E2 | Sisi Bob saat rotasi | `[KUNCI] Lawan mengganti kunci sesi. K_S baru = <16>` lalu `[KUNCI] Konfirmasi dikirim dengan kunci baru.` | LULUS (tercatat) |
| E3 | Pesan setelah rotasi | Terkirim & terbaca; `K_S` di `/status` berubah dari nilai lama | LULUS (tercatat) |
| E4 | Rotasi otomatis | `--rotasi-setelah 2` → `[HASIL] Rotasi kunci setelah pesan #2 berhasil.` | LULUS (tercatat) |

Angka nyata run `20261027-234923/rotasi`: `K_S` awal `QwqUcV0VMi9MlQxi` → setelah rotasi `gIjRjfGYFxzKYFD9`; `[HASIL] LULUS: 3 pesan terkirim, 3 balasan diterima.`; Bob mencatat `EVENT=KUNCI_BARU … TXT='gIjRjfGYFxzKYFD9'`.

### 4.6 F — Konsol interaktif

| ID | Perintah | Harapan | Status |
|---|---|---|---|
| F1 | teks biasa | `[TERKIRIM] Ciphertext AES (HEX) : …` + `Panjang ciphertext   : 64 byte` | LULUS (tercatat) |
| F2 | `/status` | 9 baris `[STATUS]`, termasuk `lokal`, `lawan`, `sesi KDC`, `K_S (ASCII)`, `TX/RX` | LULUS (tercatat) |
| F3 | `/bantu` | 4 baris daftar perintah; `/kirim` tanpa teks → `[KONSOL] Gunakan: /kirim <teks>` | BELUM DICATAT |
| F4 | perintah tak dikenal | `[KONSOL] Perintah tidak dikenal. Ketik /bantu.` | BELUM DICATAT |
| F5 | `/keluar` | Kedua proses exit 0 | LULUS (tercatat) |

### 4.7 G — Tiga IP

| ID | Yang diuji | Harapan | Status |
|---|---|---|---|
| G1 | KDC di IP sendiri | `[MENDENGAR] 127.0.0.2:5000`; log `EVENT=MENDENGAR INFO=bind=127.0.0.2:<port>` | LULUS (tercatat) |
| G2 | Bob di IP sendiri | `[SIAP] bob mendengarkan di 127.0.0.4:5002` | LULUS (tercatat) |
| G3 | Alice memakai IP sumber | `/status` → `[STATUS] lokal         : 127.0.0.3:<port>`, `[STATUS] lawan         : bob @ 127.0.0.4:5002` | LULUS (tercatat) |
| G4 | KDC melihat sumber Alice | log KDC `EVENT=LANGKAH1 PEER=127.0.0.3:<port>` | LULUS (tercatat) |
| G5 | Tiga perangkat fisik | Tiga `ipconfig`/`ip a` berbeda + `/status` IP LAN | BELUM DIJALANKAN |

### 4.8 H — Wireshark

| ID | Yang diuji | Harapan | Status |
|---|---|---|---|
| H1 | Capture ada & valid | 32 paket, 4.896 byte, pcapng, interface `\Device\NPF_Loopback`, SHA256 `34bb5529…5ae0ad` | LULUS (tercatat) |
| H2 | Tiga IP di wire | 9 segmen data: `127.0.0.3:48789→127.0.0.2:5000`, `127.0.0.2:5000→127.0.0.3:48789`, `127.0.0.3:48790↔127.0.0.4:5002` | LULUS (tercatat) |
| H3 | Isi tidak terbaca | field `data`/`tiket` hanya hex; plaintext tidak muncul | LULUS (tercatat) |
| H4 | Capture di Wi-Fi/Ethernet | pcap baru dari interface nyata saat demo | BELUM DIJALANKAN |

### 4.9 I — Konfigurasi, flag, dan kasus negatif

| ID | Kasus | Harapan | Status |
|---|---|---|---|
| I1 | Kunci master Alice salah | `[ERROR] Handshake gagal: gagal membuka balasan KDC dengan kunci master`, exit **3** | LULUS (tercatat) |
| I2 | Kunci bukan 16 ASCII | `[ERROR] Konfigurasi/kunci tidak valid: … diterima N byte`, exit **4** | BELUM DICATAT |
| I3 | Port Bob sudah dipakai | `[ERROR] Gagal mendengarkan di 127.0.0.4:5002: …`, exit **2** | BELUM DICATAT |
| I4 | KDC tidak hidup | `[ERROR] Handshake gagal: tidak bisa menghubungi KDC 127.0.0.2:5000 (…)`, exit **3** | BELUM DICATAT |
| I5 | Override lewat CLI | IP/port/kunci dari `--kdc-host`, `--kdc-port`, `--lawan-host`, `--lawan-port`, `--bind`, `--port`, `--kunci`, `--kunci-alice`, `--kunci-bob` benar-benar mengganti isi JSON | BELUM DICATAT |
| I6 | Konfigurasi tiga IP | `UjiKonfigurasi.test_kunci_konfigurasi_valid` menuntut `kdc.host`, `alice.host`, `bob.host` **berbeda semua** | LULUS (tercatat) |

---

## 5. Prosedur manual (dari nol, tiga terminal)

### 5.1 Tiga IP di satu host (loopback — ini yang pernah dijalankan)

```bash
# Terminal 1 — KDC
python kdc.py --bind 127.0.0.2 --log bukti/uji/kdc.log

# Terminal 2 — Bob
python node.py --peran bob --bind 127.0.0.4 --log bukti/uji/bob.log

# Terminal 3 — Alice (IP sumber diambil dari alice.host = 127.0.0.3)
python node.py --peran alice --log bukti/uji/alice.log
```

Di konsol Alice, ketik berurutan untuk menutup C, D, E, F:

```text
Halo Bob, ini pesan pertama
/status
/kunci
Pesan setelah kunci berganti
/status
/bantu
/keluar
```

Di konsol Bob, ketik minimal satu balasan (`Halo Alice, pesan diterima`) sebelum `/keluar`.

### 5.2 Tiga mesin LAN (BELUM dijalankan)

Syarat: `kdc.host`, `alice.host`, `bob.host` di `konfigurasi.json` **tiga alamat berbeda** (kalau `alice.host` disamakan dengan `kdc.host`, `test_aes.py` akan gagal karena assertion tiga-IP). Isi juga `kdc.kunci_master.*` dan `*.kunci_master` dengan nilai yang sama antar mesin.

```bash
# Mesin KDC
python kdc.py --konfigurasi konfigurasi.json --bind 192.168.1.10
# Mesin Bob
python node.py --peran bob --konfigurasi konfigurasi.json --bind 192.168.1.12
# Mesin Alice
python node.py --peran alice --konfigurasi konfigurasi.json --bind 192.168.1.11
```

Cek: `/status` Alice → `lokal: 192.168.1.11:<port>`, `lawan: bob @ 192.168.1.12:5002`; `/status` Bob → `lokal: 192.168.1.12:5002`, peer `192.168.1.11:<port>`.

### 5.3 Mode otomatis (tanpa interaksi, untuk bukti cepat)

```bash
# Terminal 1
python kdc.py --bind 127.0.0.2 --tanpa-konsol --log bukti/uji/kdc.log
# Terminal 2
python node.py --peran bob --bind 127.0.0.4 --tanpa-konsol \
  --balas "Balasan dari Bob" --jumlah 2 --batas 25 --log bukti/uji/bob.log
# Terminal 3
python node.py --peran alice --tanpa-konsol \
  --kirim "Pesan uji dari Alice" --jumlah 2 --harap-balasan 2 --batas 25 \
  --log bukti/uji/alice.log
```

Harapan: Alice `[HASIL] LULUS: 2 pesan terkirim, 2 balasan diterima.` (exit 0); Bob `[HASIL] Sesi selesai: 2 pesan diterima, 2 pesan dibalas.` (exit 0).

### 5.4 Kasus negatif

```bash
# I1 — kunci master salah (16 karakter, tapi bukan kunci yang dipegang KDC)
python node.py --peran alice --tanpa-konsol --kunci KunciSalahSekali; echo "exit=$?"   # → 3
# I2 — kunci bukan 16 ASCII
python node.py --peran alice --tanpa-konsol --kunci pendek; echo "exit=$?"             # → 4
# I3 — port bentrok: jalankan Bob dua kali di terminal berbeda → yang kedua exit 2
# I4 — KDC mati: jalankan Alice tanpa KDC → exit 3
```

### 5.5 Wireshark / tshark

```bash
tshark -i \Device\NPF_Loopback -f "tcp port 5000 or tcp port 5002" -w bukti/uji/uji.pcapng
tshark -r bukti/uji/uji.pcapng -Y "tcp.len > 0" -T fields -e frame.number -e ip.src -e tcp.srcport -e ip.dst -e tcp.dstport -e tcp.len
tshark -r bukti/uji/uji.pcapng -Y 'frame contains "Pesan uji dari Alice"' | wc -l   # harus 0
```

Filter untuk demo:

- Capture filter: `tcp port 5000 or tcp port 5002`
- Display: `tcp.port == 5000 || tcp.port == 5002`, lalu `tcp.len > 0`, `frame contains "permintaan_kdc"` / `"balasan_kdc"` / `"tiket"` / `"tantangan"` / `"konfirmasi"` / `"pesan"`.

Yang ditunjuk: kolom Source/Destination (tiga IP), frame langkah 1 (`id_a`, `id_b`, `n1` polos), frame langkah 2 (`sesi_id` + `data` hex), frame langkah 3 (`tiket` hex), frame langkah 4/5 (`data` 64 hex), frame pesan (`data` 128 hex, tanpa plaintext).

Payload nyata dari pcap yang ada (`bukti/pcap/analisis.txt`, frame 21, 174 byte):

```text
000000ae
7b2274697065223a2022706573616e222c202264617269223a2022616c696365222c202264617461223a20223430376535386333643065383634636565646563326433343035373930356233313663633936346230303363333436323465363538613062306463306263346236653731626666653963616139356338333562386532666632383131613065613665373061616564633737326339613764353266306563346461616631636566227d
```

Didekode: `{"tipe": "pesan", "dari": "alice", "data": "407e58c3d0e864ceedec2d34057905b316cc964b003c34624e658a0b0dc0bc4b6e71bffe9caa95c835b8e2ff2811a0ea6e70aaedc772c9a7d52f0ec4daaf1cef"}` — 128 hex = 64 byte (IV 16 + 3 blok).

---

## 6. Kriteria lulus dan kode exit

| Exit | Arti | Muncul di |
|---:|---|---|
| 0 | Normal / semua pemeriksaan `[HASIL] LULUS` | KDC, Alice, Bob, `uji_end_to_end.py` |
| 2 | Gagal bind/listen | `node.py --peran bob`, `kdc.py --tanpa-konsol` |
| 3 | Handshake gagal, atau uji otomatis gagal | `node.py --peran alice`, `uji_end_to_end.py` |
| 4 | Konfigurasi/kunci tidak valid | `node.py`, `kdc.py` |

Satu kasus uji dinyatakan **LULUS** hanya bila: (a) exit code sesuai tabel, (b) teks penanda yang disebut di matriks muncul di keluaran, dan (c) bila menyangkut isi pesan, ciphertext dapat didekripsi ulang menjadi teks asli (harness melakukannya di `periksa_ciphertext`).

---

## 7. Bukti yang harus disimpan saat demo

Simpan ke `bukti/` (di luar repo bila tidak ingin mengubah repository):

```text
bukti/uji/kdc.log, bob.log, alice.log     # satu baris per kejadian (EVENT=…)
bukti/uji/alice.out, bob.out, kdc.out     # keluaran terminal apa adanya
bukti/uji/uji.pcapng                      # capture interface nyata
bukti/uji/ringkasan.md                    # tabel: ID kasus, hasil, exit code
```

Log memakai format tetap: `<waktu ISO> ROLE=<peran> EVENT=<nama> [PEER=…] [INFO=…] [LEN=…] [HEX=…] [TXT=…]`, dengan `EVENT` ∈ `MENDENGAR, MULAI, LANGKAH1..5, SESI, TX, RX, FRAME_TX, FRAME_RX, KUNCI_BARU, KUNCI_AKTIF, TUTUP, GAGAL, TOLAK, ERROR`.

---

## 8. Yang belum dijalankan (jujur)

1. **Tiga perangkat fisik** (G5) belum pernah dijalankan; bukti tiga IP saat ini = tiga proses di satu host dengan loopback `127.0.0.2/3/4`.
2. **Capture Wi-Fi/Ethernet** (H4) belum ada; pcap yang ada dari interface loopback.
3. Kasus **D5, D6, F3, F4, I2–I5** berstatus BELUM DICATAT: valid untuk dijalankan, tetapi belum pernah dijalankan/diarsipkan.
4. Kasus negatif di level tiket (tiket untuk node lain, langkah 5 salah) tidak bisa dipicu tanpa memodifikasi program; yang ada hanya pemeriksaan kodenya (`Node.layani_bob` menolak `id_b ≠ id_diri`, `Node.layani_bob` menolak langkah 5 ≠ `N2-1`).

---

## 9. Catatan temuan pada dokumen yang sudah ada

Dua hal ini ditemukan saat menyusun rangkaian uji, dan berpotensi menjatuhkan nilai bila tidak diperbaiki:

1. `LAPORAN.md` bagian 7.4 memuat blok hex frame 21 yang **tidak cocok** dengan `bukti/pcap/analisis.txt`: blok di laporan hanya 173 byte (header `000000ae` = 174) dan setelah `…34624e65` isinya berbeda (`b41905f4…` vs `8a0b0dc0…` di pcap). Hex yang benar adalah yang ada di bagian 5.5 dokumen ini.
2. `PANDUAN_TESTING_DUA_MESIN.md` memakai `kdc.host = 192.168.1.10` **dan** `alice.host = 192.168.1.10`, sehingga hanya ada dua alamat; konfigurasi itu juga membuat `test_aes.py` (`UjiKonfigurasi`, assertion `len(alamat) == 3`) gagal — padahal checklist di panduan itu menuntut "20/20 unit test". Untuk tiga alamat, Alice harus di mesin/IP sendiri.
3. `LAPORAN.md` masih menyatakan "belum dilakukan commit/push" (ketentuan 9, bagian 8.6, status bagian 13) padahal `main` sudah berisi commit AES/KDC; dan bagian 9 butir 3 menyebut transkrip konsol tidak diarsipkan padahal `bukti/konsol_interaktif.txt` ada dan disebut di bagian 8.6.

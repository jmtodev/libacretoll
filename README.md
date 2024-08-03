# Libacretoll

Libacretoll merupakan library service untuk melakukan akses pembacaan
kartu etoll dengan menggunakan perangkat reader `ACR122U`.

Dikembangkan oleh Jasa Marga Tollroad Operator [JMTO](https://jmto.co.id/).

## Deployment dan Instalasi
Terdapat beberapa dependency yang dibutuhkan untuk menjalankan service
yang harus di-install terlebih dahulu pada host yang akan menjalankannya.

Proses instalasi dependency dapat dilakukan pada terminal dengan menjalankan
beberapa command di bawah ini:

```bash
# Install PCSC
sudo apt-get update
sudo apt-get install pcscd
sudo systemctl enable pcscd
sudo systemctl start pcscd
```

Pastikan `pn533` dan `nfc` driver di blacklist pada modprobe kernel:
```bash
sudo nano /etc/modprobe.d/blacklist-libnfc.conf
```

Isi dengan:
```
blacklist pn533
blacklist pn533_usb
blacklist nfc
```

Dan jalankan command berikut
```bash
sudo update-initramfs -u
modprobe -r pn533 nfc
```

Instalasi libacretoll selanjutnya dapat dilakukan dengan command berikut:
```bash
dpkg -i acretoll.deb
```

Untuk arm64, bisa dengan menggunakan file `acretoll-aarch64.deb`
```bash
dpkg -i acretoll-aarch64.deb
```

## Registrasi
Untuk mengetahui control unit id `cuid`, dapat dilakukan dengan command berikut:
```bash
acretoll id

  # cuid: 3e0000163b78af7899270b00c0204dec
```

Lakukan registrasi `cuid` untuk mendapatkan control unit key `cukey`.
Kemudian lakukan validasi dengan command berikut ini:
```bash
sudo acretoll reg ae97a3ff57209e58ae1a16fb917d302f

  # Successed... Key has been registered
```

Untuk melihat status aktivasi, dapat dilakukan dengan command berikut:
```bash
acretoll status

  # Registered

acretoll status

  # Unregistered
```

# Panduan Penggunaan
Untuk melihat quick help, dapat dilakukan dengan command berikut:
```bash
acretoll help
  # Libacretoll 1.0.0 (c) 2024 JMTO
  #  acretoll single <timeout>
  #    Query card as single mode
  #  acretoll tcp
  #    Start TCP mode
  #  acretoll
  #    Start Stream mode
  #  acretoll help
  #    Show this help message
  #  acretoll status
  #    Check license registration
  #  acretoll id
  #    Get machine control unit id for registration
  #  sudo acretoll reg <license-key>
  #    Show this help message
  # 
  # For more info, check github at:
  #    https://github.com/jmtodev/libacretoll
```

*Libacretoll* memiliki 3 (tiga) modus akses, yaitu:
- Single moe
- Stream mode
- TCP mode

## Single Mode
Single mode digunakan bila aplikasi akan mengakses library dengan cara
command `exec`, dimana `acretoll` akan menunggu kartu di tap sampai
waktu timeout yang telah ditentukan.

Bila terdapat kartu yang di tap dalam range timeout yang ditentukan,
maka command akan mengelurkan response berupa `stdout` dengan format
present (baca bagian Protokol).

Berikut adalah contoh command line untuk melakukan akses single mode:

```bash
acretoll single
# present bca,AF83BDCB,0145001811086937,25000
```

`acretoll` akan langsung melakukan exit setelah present didapatkan atau
timeout tercapai sebelum adanya present dari kartu.

Aplikasi dapat juga melihat return code sebelum melakukan cek `stdout`,
bila return code berisi `0`, berarti terdapat present kartu, bila `-1`
berarti timeout.

Default `timeout` untuk single sebesar 2000ms, untuk mengubahnya, dapat
dengan menambahkan argument timeout setelah argument single:

Contoh timeout 5 detik:
```bash
acretoll single 5000
# present bca,AF83BDCB,0145001811086937,25000
```

## Stream Mode
Stream mode digunakan bila aplikasi akan mengakses library dengan cara
command `exec` tetapi secara konstant melakukan monitoring stream data
dari `stdout` dengan format dan protokol yang sesuai dengan pembahasan
Protokol (lihat bagian Protokol)

Untuk menggunakan Stream mode, aplikasi dapat secara langsung menjalankan
`acretoll` tanpa menggunakan argument apapun.

```bash
acretoll
  # ack
  # ack
  # present mandiri,a2b3c4,1234567890123456,25000
  # ack
  # unpresent
  # ack
  # # present mifare,b422c1
  # ack
```

## TCP Mode
TCP mode digunakan bila aplikasi akan mengakses library dengan cara
koneksi TCP. Mode ini bermanfaat bila aplikasi tidak ingin melakukan
manajemen proses, dan `acretoll` akan berjalan pada proses independent diluar
proses aplikasi yang akan mengaksesnya.

`acretoll` akan membukan port `24042` hanya pada `localhost` untuk selanjutnya
diakses oleh aplikasi dengan metode tcp/socket satu arah.

Pastikan `acretoll` telah berjalan sebelum melakukan akses pada port tersebut,
dengan cara sebagai berikut:

```bash
acretoll tcp
  # Service has ben started at port 24042
  # ...
```

Untuk menjalankan di background dapat menggunakan command berikut ini:
```bash
acretoll tcp > /tmp/acretoll.log 2>&1 &
```

Untuk mematikan `acretoll` yang sedang berjalan dapat menggunakan command
berikut ini:
```bash
killall acretoll
```

### TCP Mode Service
`acretoll` juga dapat dijalankan menjadi service, dengan menambahkan file
service sebagai berikut:
```bash
sudo nano /etc/systemd/system/acretoll.service
```

Isi dengan:
```
[Unit]
Description=acretoll
After=network.target

[Service]
ExecStart=/usr/bin/acretoll tcp
Type=simple
Restart=always

[Install]
WantedBy=default.target
```

Reload konfigurasi systemd:
```bash
sudo systemctl daemon-reload
```

Jalankan service dengan cara:
```bash
sudo systemctl start acretoll
```

Untuk mematikan service, jalankan command berikut:
```bash
sudo systemctl stop acretoll
```

Untuk menjalankan acretoll setiap kali booting, jalankan command berikut:
```bash
sudo systemctl enable acretoll
```

### Test TCP Mode
```bash
telnet localhost:24042
  # Trying 127.0.0.1...
  # Connected to localhost.
  # Escape character is '^]'.
  # ack
  # ack
  # present mandiri,a2b3c4,1234567890123456,25000
  # ack
  # unpresent
  # ack
  # # present mifare,b422c1
  # ack
```

# Protokol
Protokol berupa line-terminated message, dimana satu message akan dikirimkan
dalam satu baris yang independent.

Berikut adalah format/syntax message
```
[message][space][arg-1][comma][arg-2][comma][arg-3]

message arg1,arg2,arg3
```

Bila message tidak memiliki argument, maka message akan dikirim tanpa argument
dan space.

```
[message]
```

## Referensi Message
- `present` Terdeteksi kartu baru yang sedang di tap
  - **Argument 1** Jenis kartu
  - **Argument 2** UUID kartu
  - **Argument 3** Nomor kartu (khusus etoll)
  - **Argument 4** Saldo kartu (khusus etoll)
  - **Contoh:**
    - `present bca,b4cc7a21,1234567890123456,75000`
    - `present mifare,b4cc7a21`
    - `present mifare,aabbccdd`
    - `present bri,b4cc7a21,1234567890123456,132000`
    - `present bni,b4cc7a21,1234567890123456,2500`
    - `present mandiri,b4cc7a21,1234567890123456,22500`
- `unpresent` Terdeteksi kartu dilepas dari reader
- `ack` Heart Beat koneksi untuk menandakan reader ready
- `nak` Heart Beat koneksi untuk menandakan reader tidak terdeteksi

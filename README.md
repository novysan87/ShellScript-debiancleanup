Shell Script yang di gunakan untuk membersihkan Debian dari paket aplikasi seperti Apache, Bind9 dan lain-lain namun tidak menghapus paket aplikasi yang dibutuhkan oleh sistem
Cara penggunaan :
1. berikan akses execute dengan perintah chmod +x DebianCleanup.sh
2. jika terdapat error saat di jalankan itu bisa jadi ada masalah di format teks, berikan perintah berikut sed -i 's/\r$//' DebianCleanup.sh
3. kemudian jalankan dengan perintah ./DebianCleanup.sh

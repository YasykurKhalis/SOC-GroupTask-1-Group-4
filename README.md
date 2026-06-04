# 🛡️ Automated Incident Response: Automasi Mitigasi Serangan DDoS Melalui Integrasi Wazuh XDR/SIEM dan Shuffle SOAR

Proyek ini mendokumentasikan perancangan, konfigurasi, dan implementasi arsitektur **Security Operations Center (SOC)** modern skala laboratorium menggunakan platform **Wazuh** (Security Detection & Compliance) dan **Shuffle SOAR** (Security Orchestration, Automation, and Response) yang berjalan di atas infrastruktur cloud **Microsoft Azure**. 

Tujuan utama dari proyek ini adalah mengotomatisasi proses mitigasi serangan *Distributed Denial of Service* (DDoS) berbasis volume trafik pada Web Server Nginx. Dengan mengintegrasikan sistem deteksi pasif menjadi penahanan aktif (*Active Response*), *Mean Time to Respond* (MTTR) dapat dipangkas hingga level detik tanpa membutuhkan intervensi manual dari analis SOC (*Zero-Touch Defense*).

---

## 👥 Tim Pengembang (Kelompok 6)
|Nama Kelompok                | NRP         |
|-----------------------------|-------------|
| Hanif Mawla Faizi |  5027241064 | 
| M Khosyi Syehab |  50272410xxx |
| Yasykur Khalis J M Y | 50272411112 | 

---

## 🏛️ Arsitektur Jaringan & Topologi Sistem

Sistem ini diimplementasikan menggunakan arsitektur terdistribusi pada beberapa *Virtual Machine* (VM) Ubuntu di dalam jaringan Microsoft Azure:

```text
[ LAPTOP PENYANG / ATTACKER ] (Meluncurkan ApacheBench)
             │
             ▼ (Trafik DDoS via Port 80)
┌────────────────────────────────────────────────────────┐
│  AGENTS LAYER (Ubuntu Server VM)                       │
│  ┌─────────────────────────┐                           │
│  │   Target Web (Nginx)    │                           │
│  │   Wazuh Agent           │───(Meneruskan Log)───┐    │
│  └─────────────────────────┘                      │    │
└───────────────────────────────────────────────────┼────┘
                                                    │ (Port 1514/TCP)
┌───────────────────────────────────────────────────┼────┐
│  CORE SOC LAYER (Azure Virtual Network)           │    │
│                                                   ▼    │
│  ┌──────────────────────────────────────────────────┐  │
│  │ Wazuh Manager & Indexer                          │  │
│  │ - Menganalisis Log & Memicu Rule 100004          │  │
│  │ - Menyediakan RESTful API (Port 55000)           │  │
│  └──────────────────────────────────────────────────┘  │
│                           │                            │
│                           │ (Kirim Alert via Webhook JSON)
│                           ▼                            │
│  ┌──────────────────────────────────────────────────┐  │
│  │ Shuffle SOAR Platform (Cloud Engine)             │  │
│  │ - Memproses Webhook                              │  │
│  │ - Autentikasi JWT Dinamis                        │  │
│  │ - Mengirimkan Perintah Eksekusi Pemblokiran      │  │
│  └──────────────────────────────────────────────────┘  │
│                           │                            │
│                           └──────(API Active Response)─┘
│                                  (PUT /active-response)
```

⚙️ Detail Konfigurasi Komponen Sistem1. Konfigurasi Wazuh Manager (/var/ossec/etc/ossec.conf)Wazuh Manager dikonfigurasi untuk menangani integrasi webhook keluar menuju Shuffle serta mendaftarkan perintah eksekusi internal.

Blok Integrasi Shuffle SOAR:XML<integration>
  <name>shuffle</name>
  <hook_url>[https://shuffler.io/api/v1/hooks/webhook_4a6fc57f-dfa8-491d-ac76-23d0340d49fb](https://shuffler.io/api/v1/hooks/webhook_4a6fc57f-dfa8-491d-ac76-23d0340d49fb)</hook_url>
  <rule_id>100004</rule_id>
  <alert_format>json</alert_format>
</integration>
Blok Command & Active Response (Penting):XML<command>
  <name>firewall-drop</name>
  <executable>firewall-drop</executable>
  <timeout_allowed>yes</timeout_allowed>
</command>

<active-response>
  <command>firewall-drop</command>
  <location>local</location>
</active-response>
2. Konfigurasi Kredensial & Jalur Komunikasi APIWazuh API berjalan pada port 55000/TCP. Karena keterbatasan bawaan pada beberapa repositori SOAR publik dalam mengelola enkripsi multi-parameter, autentikasi dialihkan menggunakan injeksi manual Basic Authentication berbasis teks rahasia Base64 yang bersumber dari pengguna internal sistem (wazuh-wui).🔄 Alur Kerja Playbook Otomatisasi (Shuffle SOAR)Prosedur penanganan insiden yang dirancang pada kanvas SOAR mengikuti diagram logika sekuensial berikut:> Gambar 1: Desain Alur Kerja (Playbook) Otomatisasi Respons Insiden Siber pada Kanvas Shuffle.Penjelasan Teknis Setiap Node:Webhook 1 (Trigger Node): Menangkap kiriman payload JSON mentah dari Wazuh Manager saat Rule 100004 aktif. Node ini mengekstrak variabel krusial seperti $exec.all_fields.data.srcip (IP Penyerang) dan $exec.all_fields.agent.id (ID Target).Wazuh_Auth (HTTP GET): * URL: https://<WAZUH_MANAGER_IP>:55000/security/user/authenticateHeader: Authorization: Basic [ENCRYPTED_BASE64_KEY]Fungsi: Melakukan handshake otorisasi dan menghasilkan token akses JWT pendek (masa aktif 15 menit) demi menjaga keamanan API.Trigger_Action (HTTP PUT):URL: https://<WAZUH_MANAGER_IP>:55000/active-response?agents_list=$exec.all_fields.agent.idHeader: Authorization: Bearer $Wazuh_Auth.body.data.tokenBody JSON:JSON{
  "command": "firewall-drop0",
  "alert": {
    "data": {
      "srcip": "$exec.all_fields.data.srcip"
    }
  }
}
🧪 Skenario Simulasi Serangan & Validasi Respons1. Peluncuran Serangan Eksploitasi (Red Team Operation)Serangan dilakukan dari terminal eksternal menggunakan perkakas pengujian beban ApacheBench untuk mensimulasikan trafik DDoS berbasis HTTP Flood:Bashab -n 50000 -c 50 [http://10.0.0.5/](http://10.0.0.5/)
2. Hasil Deteksi Log Real-Time (SIEM Alerting)Wazuh Manager berhasil mendeteksi anomali volume trafik berlebih dalam waktu singkat dan memicu alert tingkat tinggi, kemudian meneruskannya langsung ke platform SOAR.> Gambar 2: Struktur Data JSON Alert SIEM yang Berhasil Ditangkap oleh Node Trigger.3. Eksekusi Orchestrator Sukses (SOAR Execution)Shuffle mengolah data yang masuk, meminta token otentikasi baru, dan menembakkan instruksi Active Response kembali ke API Wazuh. Parameter affected_items mencatat angka 1, menandakan instruksi penahanan telah berhasil dikirimkan ke agen.> Gambar 3: Status Respon 200 OK dari RESTful API Wazuh Menandakan Perintah AR Diterima.4. Penahanan di Tingkat Firewall (Blue Team Enforcement)Pada sisi server agen target (hanip), skrip internal membaca instruksi dari Manager dan segera memanggil utilitas kernel Linux iptables untuk melakukan pembuangan paket data (DROP) terhadap IP asal serangan.> Gambar 4: Aturan Pertahanan Baru pada Tabel Netfilter Kernel Linux Berhasil Mengisolasi IP Penyerang.Hasilnya, trafik dari penyerang langsung terputus (Connection Timeout), sementara pengguna legal dari jaringan eksternal lain tetap mampu mengakses layanan web tanpa hambatan.🛠️ Riwayat Penanganan Masalah & Evaluasi (Troubleshooting Ledger)Selama proses deployment lab SOC ini, ditemukan beberapa kendala teknis krusial yang menjadi poin pembelajaran penting (lessons learned):Jenis Kendala (Error)Penyebab UtamaSolusi PenyelesaianConnectTimeout (Port 55000)Aturan Network Security Group (NSG) di Azure memblokir koneksi masuk dari luar jaringan internal.Menambahkan aturan Inbound Port Rule baru pada portal Azure untuk membuka Port 55000/TCP.401 Unauthorized (API)Kredensial akun admin web dashboard tidak tersinkronisasi secara otomatis dengan database otentikasi API internal port 55000.Mengganti user target ke pengguna internal mesin (wazuh-wui) dan menggunakan enkripsi manual format Header Basic Auth Base64.406 Not Acceptable / Error 6002Node HTTP Shuffle tidak mendeklarasikan jenis tipe data yang dikirimkan pada bagian tubuh (Body).Menyuntikkan parameter "Content-Type": "application/json" ke dalam struktur Headers Shuffle.1652: Command Not DefinedBlok komando aktif, namun rute fungsional <active-response> di dalam file konfigurasi ossec.conf masih terkurung komentar XML (``).Menghapus penanda komentar dan merestart ulang layanan wazuh-manager.Cannot read 'srcip' from dataSkrip bawaan firewall-drop menolak parameter arguments tunggal karena memerlukan struktur objek berkerangka Alert.Merombak struktur Body JSON pada Shuffle dengan membungkus parameter IP ke dalam objek nested: {"alert": {"data": {"srcip": "..."}}}.Self-Denial of Service (Self-DoS)Eksekusi Active Response memblokir IP publik pengelola/jaringan SOC itu sendiri sehingga memutuskan akses manajemen (SSH/Web).Mendaftarkan seluruh IP manajemen, IP internal, dan IP gateway ke dalam daftar pengecualian <white_list> pada konfigurasi global Wazuh Manager.📝 KesimpulanProyek praktikum Kelompok 6 ini berhasil membuktikan bahwa integrasi platform SIEM/XDR (Wazuh) dan SOAR (Shuffle) memberikan kapabilitas mitigasi ancaman otomatis yang sangat tangguh. Pengalihan proses respons dari manusia ke mesin (Automated Playbook) terbukti mampu meminimalkan dampak serangan DDoS pada infrastruktur web publik secara instan dan efisien.

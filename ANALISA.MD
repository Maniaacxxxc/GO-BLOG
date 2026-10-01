# Laporan Analisa Fitur Tersembunyi Modem ZTE G5TS (Unlocked Version)

---

## 1. Ringkasan Eksekutif & Identitas Perangkat

Hasil investigasi mendalam terhadap modem **ZTE G5TS** (berbasis arsitektur OpenWrt / Linux `S1` dengan komunikasi antarmuka **UBUS JSON-RPC**) menunjukkan bahwa perangkat ini memiliki berbagai fitur tersembunyi tingkat lanjut (*engineering/advanced features*). Fitur-fitur ini sebagian besar disembunyikan (*hidden/restricted*) dari antarmuka Web UI standar bawaan operator, namun backend API dan modul sistem modem mendukung penuh kontrol fungsional tersebut.

### Informasi Perangkat (Live Data):
| Parameter | Nilai / Keterangan |
|---|---|
| **Model Perangkat** | ZTE G5TS (CPE Modem) |
| **Hardware Version** | `G5TSHW_1.0.0` |
| **Firmware / Software** | `IOH_ID_G5TSV1.0.0B04` (`BD_IOHIDG5TSV1.0.0B04`) |
| **Inner Software Version**| `BD_IOHIDG5TSV1.0.0B03` |
| **Platform OS** | OpenWrt Linux (Kernel / UBUS RPC Framework) |
| **MAC Address WLAN** | `5C:7D:AE:ED:E2:8D` |
| **IMEI** | `869338073562674` |
| **Status Kunci SIM** | Unlocked (`lock_status: NA`, Bebas semua operator) |

---

## 2. Status Sinyal & Jaringan Saat Ini (Real-Time)

Berdasarkan pembacaan telemetry internal (`zte_nwinfo_api`):
- **Operator Terhubung:** Smartfren (MCC: `510`, MNC: `09`)
- **Tipe Jaringan Aktif:** **ENDC (5G NSA Dual Connectivity)**
- **Carrier Aggregation (CA):** Aktif (2x LTE CA + 5G NSA)
  - **Primary Band (LTE):** Band 3 (EARFCN: `1313`, PCI: `129`, Bandwidth: `20 MHz`)
  - **Secondary Band (LTE):** Band 1 (EARFCN: `325`, PCI: `398`, Bandwidth: `15 MHz`)
  - **5G NR Band:** **n41** (NR ARFCN: `504990`, PCI: `121`, Bandwidth: `50 MHz`)
- **Kualitas Sinyal LTE:** RSRP `-84 dBm` ~ `-87 dBm`, RSRQ `-9 dB` ~ `-12 dB`, SNR `10.0 - 15.0 dB` (Sangat Baik / 5 Bar)
- **Kualitas Sinyal 5G NR:** RSRP `-81 dBm` ~ `-83 dBm`, RSRQ `-11 dB`, SNR `23.5 - 25.5 dB` (Sangat Prima)

---

## 3. Daftar Fitur Tersembunyi yang Berhasil Ditemukan

### A. Fitur Kunci Frekuensi (Band Locking)
Modem ZTE G5TS mendukung penguncian pita frekuensi secara spesifik baik untuk 4G LTE maupun 5G (SA dan NSA).

1. **4G LTE Band Lock:**
   - **Supported Hardware Bands:** Band 1 (2100MHz), Band 3 (1800MHz), Band 8 (900MHz), Band 28 (700MHz).
   - **Endpoint API:** `zte_nwinfo_api` -> `nwinfo_set_lte_ext_band`
   - **Parameter:** `{"lte_ext_band_lock": "1,3,8,28"}` (bisa dikustomisasi, misalnya hanya `"3"` untuk lock Band 3, atau `"1,3"`).
   - **Legacy Bitmask Endpoint:** `SET_NETWORK_BAND_LOCK` dengan `lte_band_lock: "0x008000085"`.

2. **5G SA / NSA Band Lock:**
   - **Supported Hardware Bands:** n1, n28, n41, n78.
   - **Endpoint API:** `zte_nwinfo_api` -> `nwinfo_set_sa_bandlock`
   - **Parameter:** `{"nr5g_sa_band_lock": "1,28,41,78"}` atau `{"nr5g_nsa_band_lock": "41"}`
   - **Reset Kunci Band/Cell ke Default:** `zte_nwinfo_api` -> `nwinfo_reset_band_cell_setting` (`{}`).

---

### B. Fitur Kunci Pemancar / Menara BTS (Cell Locking / PCI Lock)
Fitur ini sangat berguna jika modem sering berpindah-pindah BTS (tower flapping) atau ingin mengunci ke BTS yang memiliki performa dan bandwidth tertinggi.

1. **LTE Cell Lock (PCI + EARFCN):**
   - **Endpoint:** goform `LTE_LOCK_CELL_SET`
   - **Parameter:**
     ```json
     {
       "lte_lockcell_pci": "129",
       "lte_lockcell_earfcn": "1313"
     }
     ```
2. **5G NR Cell Lock:**
   - **Endpoint:** `zte_nwinfo_api` -> `nwinfo_lock_nr_cell`
   - **Parameter:**
     ```json
     {
       "Z5g_lockcell_pci": "121",
       "Z5g_lockcell_arfcn": "504990",
       "Z5g_lockcell_band": "41",
       "Z5g_lockcell_scs": "30"
     }
     ```

---

### C. Pemilihan Mode Jaringan Khusus (Network Mode Selection)
Modem ini mendukung pemilihan mode jaringan yang fleksibel:
- **5G/4G Auto (ENDC/SA):** `4G_AND_5G`
- **5G NSA Only (LTE Anchor + 5G):** `LTE_AND_5G`
- **5G SA Only (Pure Standalone 5G):** `Only_5G`
- **4G Only (LTE Only):** `Only_LTE`
- **Endpoint API:** `zte_nwinfo_api` -> `nwinfo_set_netselect`

---

### D. Kontrol Antena Eksternal vs Internal (Antenna Switching)
Modem mendukung konfigurasi pemilihan antena secara manual atau otomatis:
- **Pilihan:** `internal`, `external`, `auto`
- **Endpoint API:** `zte_nwinfo_api` -> `nwinfo_set_external_ant`
- **Payload:** `{"ant_state": "external"}` atau `{"ant_state": "internal"}`

---

### E. Menu Developer / Engineering Options & Diagnostik Lanjutan
Terdapat level hak akses pengembang (**Developer Options**) pada URL rute `#developer_options` dan `#developer_options_login`.
- **Developer Login Endpoint:** `zwrt_web` -> `web_developer_option_login` (Telah diverifikasi aktif dan dapat di-unlock menggunakan password web: `Bismillah`).
- **Fitur di dalamnya:**
  1. **Packet Capture (TCPDump Web):** `router_webui_catch_tcpdump` (Capture traffic LAN/WAN langsung dari Web UI).
  2. **Modem Qualcomm Diagnostic Log:** `router_webui_catch_modemlog` (Dump QXDM/Modem log untuk analisa radio RF).
  3. **Tracing Tool & Syslog Level:** Pengaturan level log kernel/system hingga mode debug.
  4. **WLAN Throughput Testing:** `WLAN_TEST_TPUTS_SET` (Internal iperf/tputs test engine).
  5. **MBN Mode Selector:** `WAN_MBN_MODE_SET` (Pilihan file konfigurasi modem Qualcomm per operator).

---

### F. USSD & SMS Gateway
Backend modem memiliki modul lengkap untuk menjalankan perintah USSD dan manajemen SMS:
- **USSD Execution:** `zwrt_ussd` -> `libzte_ussd_web_process` (Dapat digunakan untuk cek pulsa, kuota, isi voucher via `*123#`, `*888#`, dll.).
- **SMS Manager:** `zwrt_wms` -> `zte_libwms_send_sms` dan `zte_libwms_get_sms_data` (Dapat membaca dan mengirim SMS langsung via API).

---

### G. Enterprise Networking & Tunneling
Mendukung protokol tunneling enterprise yang jarang aktif di modem rumahan biasa:
- **VPN Client:** PPTP, L2TP, IPsec, GRE Tunnel.
- **VXLAN Overlay:** Mendukung konfigurasi VXLAN VNI & VTEP interface.
- **Operation Modes:** `PPP` (Cellular Gateway), `LTE_BRIDGE` (Bridge mode murni langsung memberikan IP publik ke router mikrotik/PC), `AUTO_LTE_GATEWAY`, `MULTIWAN`.
- **DSDS / SIM Slot Threshold:** Pengaturan auto switch slot kartu berdasarkan threshold RSRP/SNR (`slot_switch_thr_rsrp1: -115 dBm`, dll.).
- **VoIP / VoLTE SIP Client:** Dukungan penuh SIP Account, SIP Server, dan codec audio G.711 / G.729 / G.722.

---

## 4. Cara Menggunakan Fitur Tersembunyi (Skrip & API)

Seluruh komunikasi modem ZTE G5TS menggunakan protokol **JSON-RPC over HTTPS/HTTP** ke endpoint `/ubus/?t=<timestamp>`.

### Algoritma Otentikasi Password:
1. Ambil salt acak dari modem: `zwrt_web` -> `web_login_info`
2. Hitung: `Hash = SHA256( SHA256(password).toUpperCase() + salt ).toUpperCase()`
3. Kirim ke `zwrt_web` -> `web_login` untuk mendapatkan `ubus_rpc_session` (token sesi).

---

### Skrip Python Siap Pakai: Tool Manajemen ZTE G5TS (Aman)

Simpan skrip di bawah ini (misalnya `zte_manager.py`) untuk mengecek sinyal detail, melakukan Band Lock, atau Cell Lock secara instan:

```python
#!/usr/bin/env python3
"""
ZTE G5TS Management & Band Locking Script
"""
import requests
import urllib3
import hashlib
import json
import time

urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

MODEM_IP = "192.168.0.1"
PASSWORD = "Bismillah"

class ZTEG5TS:
    def __init__(self, ip=MODEM_IP, password=PASSWORD):
        self.ip = ip
        self.password = password
        self.session = requests.Session()
        self.session.headers.update({
            'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64)',
            'Referer': f'https://{self.ip}/index.html',
            'Origin': f'https://{self.ip}',
            'Content-Type': 'application/json'
        })
        self.sid = "00000000000000000000000000000000"

    def _ubus(self, module, method, params={}):
        url = f"https://{self.ip}/ubus/?t={int(time.time()*1000)}"
        body = [{
            "jsonrpc": "2.0",
            "id": 1,
            "method": "call",
            "params": [self.sid, module, method, params]
        }]
        headers = {
            "Z-Mode": "1" if self.sid != "00000000000000000000000000000000" else "0",
            "Z-Tag": method
        }
        res = self.session.post(url, json=body, headers=headers, verify=False, timeout=10)
        return res.json()[0]

    def login(self):
        info = self._ubus("zwrt_web", "web_login_info")
        salt = info["result"][1]["zte_web_sault"]
        
        h1 = hashlib.sha256(self.password.encode('utf-8')).hexdigest().upper()
        h2 = hashlib.sha256((h1 + salt).encode('utf-8')).hexdigest().upper()
        
        res = self._ubus("zwrt_web", "web_login", {"password": h2})
        if "result" in res and res["result"][1].get("result") == 0:
            self.sid = res["result"][1]["ubus_rpc_session"]
            print(f"[+] Login Berhasil! Sesi Token: {self.sid}")
            return True
        else:
            print("[-] Login Gagal:", res)
            return False

    def get_signal_info(self):
        """Membaca informasi sinyal dan BTS secara detail"""
        res = self._ubus("zte_nwinfo_api", "nwinfo_get_netinfo")
        data = res["result"][1]
        print("\n" + "="*45)
        print(f" OPERATOR       : {data.get('network_provider_fullname')} ({data.get('network_provider')})")
        print(f" NETWORK TYPE   : {data.get('network_type')} (Mode: {data.get('net_select')})")
        print(f" LTE ACTIVE BAND: {data.get('wan_active_band')} (PCI: {data.get('lte_pci')}, EARFCN: {data.get('wan_active_channel')})")
        print(f" LTE SIGNAL     : RSRP: {data.get('lte_rsrp')} dBm | RSRQ: {data.get('lte_rsrq')} dB | SNR: {data.get('lte_snr')} dB")
        if data.get('nr5g_action_band'):
            print(f" 5G NR BAND     : {data.get('nr5g_action_band')} (PCI: {data.get('nr5g_pci')}, ARFCN: {data.get('nr5g_action_channel')}, BW: {data.get('nr5g_bandwidth')}MHz)")
            print(f" 5G NR SIGNAL   : RSRP: {data.get('nr5g_rsrp')} dBm | RSRQ: {data.get('nr5g_rsrq')} dB | SNR: {data.get('nr5g_snr')} dB")
        print(f" LTE CA DETAIL  : {data.get('lteca')}")
        print("="*45 + "\n")

    def lock_lte_band(self, bands="1,3,8,28"):
        """
        Kunci band LTE tertentu.
        Contoh:
          lock_lte_band("3") -> Kunci hanya Band 3 (1800MHz)
          lock_lte_band("1,3") -> Kunci Band 1 dan Band 3
          lock_lte_band("1,3,8,28") -> Buka semua Band LTE (Default)
        """
        print(f"[*] Mengatur Kunci Band LTE ke: {bands} ...")
        res = self._ubus("zte_nwinfo_api", "nwinfo_set_lte_ext_band", {
            "lte_ext_band_lock": bands
        })
        print("[+] Hasil:", res)

    def lock_5g_band(self, sa_bands="1,28,41,78"):
        """Kunci band 5G SA/NSA"""
        print(f"[*] Mengatur Kunci Band 5G SA ke: {sa_bands} ...")
        res = self._ubus("zte_nwinfo_api", "nwinfo_set_sa_bandlock", {
            "nr5g_sa_band_lock": sa_bands
        })
        print("[+] Hasil:", res)

    def reset_lock(self):
        """Mereset semua penguncian Band & Cell ke pengaturan otomatis"""
        print("[*] Mereset penguncian Band & Cell ...")
        res = self._ubus("zte_nwinfo_api", "nwinfo_reset_band_cell_setting", {})
        print("[+] Hasil:", res)

if __name__ == "__main__":
    modem = ZTEG5TS()
    if modem.login():
        modem.get_signal_info()
```

---

## 5. Kesimpulan & Rekomendasi Penggunaan Aman

1. **Status Perangkat:**
   Modem ZTE G5TS Anda berada dalam kondisi sehat, berstatus **Unlocked (bebas kartu/all operator)**, dan saat ini berjalan sangat stabil pada jaringan 5G NSA Smartfren (Carrier Aggregation B3 + B1 + n41) dengan kualitas sinyal optimal.
2. **Ketersediaan Fitur:**
   Fitur **Lock Band 4G (B1, B3, B8, B28)**, **Lock Band 5G (n1, n28, n41, n78)**, **Cell Lock (PCI/EARFCN)**, **External Antenna Switch**, **USSD**, dan **Developer Diagnostics** sepenuhnya didukung oleh firmware modem dan dapat dieksekusi secara aman melalui API UBUS tanpa risiko merusak firmware/hardware perangkat.
3. **Peringatan Keamanan (Safety Precautions):**
   - Tidak disarankan mengubah parameter NVRAM tingkat rendah selain yang disediakan oleh API `zte_nwinfo_api`.
   - Jika sinyal hilang setelah melakukan lock band/cell tertentu (karena operator tidak memancarkan band tersebut di lokasi Anda), cukup jalankan fungsi `reset_lock()` untuk mengembalikan ke pencarian otomatis (*auto-select*).

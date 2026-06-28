```
📎-Production Deployment

> **UAS Sistem Operasi + Jaringan Komputer**-Kelas Sentul, Sesi 16
>
> Static HTML site yang dideploy ke VPS menggunakan Docker Compose, diakses via subdomain kustom (melalui reverse proxy Nginx), dan dipantau secara real-time menggunakan Uptime Kuma.

---

📎 1. Architecture Diagram

```text
Developer laptop
       │
       ▼ git push (main)
GitHub Repo ───trigger───► GitHub Actions (CI/CD)
                                │
                                ▼ SSH deploy
                          ┌────────────────────────────────────────────────────────┐
                          │ VPS Server (IP: 103.168.146.195)                       │
                          │                                                        │
 User Browser             │  nginx (Port 80) ───proxy_pass───►  App Container      │
      │                   │  (Subdomain DNS)                     (Docker Port 8085)│
      ▼ resolve domain    │                                        │               │
 (Domain DNS A record) ───┼────────────────────────────────────────┘               │
                          │                                                        │
                          │   ▲                                                    │
                          │   │ health check                                       │
                          │                                                        │
                          │  Uptime Kuma (Monitoring Container)                    │
                          └────────────────────────────────────────────────────────┘

```

# Alur:

1. **Developer** melakukan `git push` ke branch `main` di GitHub Repo.
2. **GitHub Actions** otomatis berjalan: melakukan build (jika diperlukan) → SSH ke VPS → mengelola deployment container.
3. **User** mengakses subdomain kustom → DNS melakukan resolve ke IP VPS `103.168.146.195` → request diterima oleh Nginx port 80 → Nginx melakukan `proxy_pass` ke port internal Docker `8085`.
4. **Uptime Kuma** melakukan polling endpoint setiap menit untuk memantau status uptime website secara real-time.

📎 Tech Stack

| Layer | Tools |
| --- | --- |
| **App** | Static HTML / CSS |
| **Web Server / Reverse Proxy** | Nginx |
| **Container** | Docker + Docker Compose |
| **CI/CD** | GitHub Actions |
| **Monitoring** | Uptime Kuma |
| **VPS Provider** | Herza / Hostingan Partner |
| **Domain** | `bianujiweb.my.id` |

---

## 2. Struktur Repo

```text
UAS-WEB-DEPLOYMENT/
├── .github/
│   └── workflows/
│       └── deploy.yml          # CI/CD pipeline - trigger saat push ke main
├── Documents/
│   └── uas-tazkia/             # Folder project utama aplikasi
│       ├── docker-compose.yml  # Definisi App Container (Aplikasi + Monitoring)
│       └── html/
│           └── index.html      # Aplikasi static web (Happy Birthday)
└── README.md                   # Dokumentasi deployment

```



📎 3. Setup & Deployment

📎 3.1 Domain & DNS

• <Main Domain:> `bianujiweb.my.id`
• <Sub Domain:> `tazkia.bianujiweb.my.id`
• <DNS Record:> A Record dengan host `tazkia` mengarah langsung ke IP VPS `103.168.146.195` dengan TTL `14400`.

📎 3.2 Spesifikasi VPS

• <OS:> Ubuntu Server
• <User Akses:> `bian14fire` (Non-sudoers)

📎 3.3 Konfigurasi Reverse Proxy (Nginx)

Karena user utama deployment tidak memiliki hak akses `sudo`, konfigurasi routing port `80` ke port Docker `8085` dikonfigurasikan pada block server Nginx berikut:

```nginx
server {
    listen 80;
    server_name tazkia.bianujiweb.my.id;

    location / {
        proxy_pass [http://127.0.0.1:8085](http://127.0.0.1:8085);
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

```

📎 3.4 Container Configuration (Docker Compose)

Aplikasi diisolasi menggunakan Docker Compose dan dijalankan pada port `8085` agar tidak bentrok dengan layanan lain di VPS:

```yaml
version: '3.8'

services:
  web-uas:
    image: nginx:latest
    container_name: uas-web-app
    ports:
      - "8085:80"
    volumes:
      - ./html:/usr/share/nginx/html
    restart: always

  uptime-kuma:
    image: louislam/uptime-kuma:1
    container_name: uas-monitoring-kuma
    ports:
      - "3001:3001"
    volumes:
      - uptime-kuma-data:/app/data
    restart: always

volumes:
  uptime-kuma-data:

```

Untuk menjalankan container secara manual di dalam folder project, gunakan perintah:
docker compose up -d --build

📎 3.5 Monitoring

* **Dashboard Platform:** Uptime Kuma
* **Target Monitoring:** `http://103.168.146.195:8085`
* **Interval Check:** 60 detik



📎 4. Lessons Learned

• <Manajemen Kolaborasi & Resource VPS:> Sebagai penyewa utama VPS yang membagi space server untuk kebutuhan bersama, proyek ini memberikan pelajaran berharga mengenai pentingnya *resource allocation* dan isolasi aplikasi menggunakan Docker agar port antar project tidak saling bertabrakan.
• <Pemahaman Hak Akses (Privilege Limitation):> Menghadapi kendala non-sudoers (`bian14fire is not in the sudoers file`) memberikan pemahaman nyata tentang dunia kerja DevOps/Sysadmin. Kita dipaksa kreatif melakukan deployment aplikasi secara aman di dalam ruang lingkup *home directory* (`~/`) tanpa mengorbankan stabilitas keamanan folder root sistem sistem operasi.
• <Konsep Web Pasca-Deployment:> Memahami secara mendalam bahwa deployment tidak sekadar membuat container aktif, melainkan bagaimana mengatur manajemen DNS tingkat lanjut, pengelolaan subdomain kustom di registrar, serta menjembatani traffic publik menggunakan Nginx Reverse Proxy.



📎 5. Referensi

• https://software.endy.muhardin.com/devops/deployment-microservice-kere-hore-1/

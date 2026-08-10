# Sentinel WABOT — catatan isolasi

Projek ini **berdiri sendiri**. Asalnya dari `MreRes/AI-WhatsApp-CS-Sales-Multi-Account`
branch `WAHA-version` (commit `9570695`), tapi riwayat git upstream sudah dibuang dan
diganti satu commit awal yang bersih. Tidak ada remote upstream, tidak ada relasi fork.

Tujuannya: bisa jalan **berbarengan** dengan projek Hermes/Sentinel lama tanpa satu pun
titik rebutan — port, container, network, volume, maupun database.

## Peta port (semua di band 35xx)

| Servis | Port host | Lama (Hermes) |
|---|---|---|
| Web (Next.js) | **3500** | 3000 |
| API (NestJS + Socket.IO) | **3501** | 3001 |
| Postgres | **3502** → 5432 | 5433 |
| Redis | **3503** → 6379 | 6379 |
| Nginx (profil `nginx`) | **3580** → 80 | 80 |
| Prometheus | **3590** → 9090 | 9090 |
| Alertmanager | **3593** → 9093 | 9093 |
| Hermes sidecar (opsional) | **3575** | 8675 |

## Identitas yang dipisah

- Nama projek Compose: `sentinel-wabot` (ditulis eksplisit lewat `name:` di
  `docker-compose.yml`, jadi namespace-nya tetap terpisah walau dijalankan dari folder mana pun)
- Container: `sentinel-wabot-{postgres,redis,api,web,nginx,prometheus,alertmanager}`
- Network: `sentinel_wabot`
- Volume: `wabot_{pgdata,redisdata,sessions,media,prometheus_data}`
- Database default: user/pass/db `wabot` (dulu `hermes`)
- User di container API: `wabot` (dulu `hermes`)
- Seed owner: `owner@sentinel-wabot.local`
- Nama paket npm root: `sentinel-wabot`

## Yang SENGAJA tidak diubah

Ini bukan kelalaian — mengubahnya justru merusak:

- `HERMES_MODEL`, `Hermes-4-70B`, `Hermes-4-405B` → nama model Nous Research, bukan
  identitas projek.
- `HERMES_BIN`, `HERMES_SIDECAR_*`, `HERMES_NOTIFY_TARGET`, `hermes-agent`,
  `services/hermes-sidecar/` → CLI/sidecar eksternal milik Nous, dipakai apa adanya.
- Tabel & tipe Postgres `hermes_reviews`, `HermesDecision`, kolom `hermes_review_id`,
  nama metrik `hermes_*`, dan alert `Hermes*` di `monitoring/alerts.yml` → nama internal
  di dalam DB sendiri. Karena databasenya sudah terpisah, tidak ada tabrakan; menggantinya
  berarti membongkar rantai migrasi Prisma tanpa manfaat.
- `docs/superpowers/**` → dokumen desain historis, dibiarkan utuh sebagai arsip.

## Bukti pemisahan Docker — diadu langsung dengan `hermes_repo`

Dibaca langsung dari `hermes_repo/docker-compose*.yml` pada 2026-08-10 (**baca saja, nol
perubahan di sana**). Nol irisan di semua sumbu:

| Sumbu | `hermes_repo` | `sentinel_wabot` |
|---|---|---|
| Nama projek Compose | `hermes_repo` (implisit dari nama folder) | `sentinel-wabot` (**eksplisit** lewat `name:`) |
| Web | 3000 | **3500** |
| API | 3001 | **3501** |
| Postgres | 5433 | **3502** |
| Redis | 6379 → 6380 (override) | **3503** |
| Nginx | 80 | **3580** |
| Prometheus | 9090 | **3590** |
| Alertmanager | 9093 | **3593** |
| Container | `hermes-*` | `sentinel-wabot-*` |
| Network | `hermes` | `sentinel_wabot` |
| Volume | `hermes_pgdata/redisdata/sessions/media/prometheus_data` | `wabot_*` |
| DB user/db | `hermes` | `wabot` |

Karena nama projek Compose berbeda, `docker compose down -v` di salah satu projek **tidak
bisa** menyentuh volume projek yang lain. Keduanya boleh menyala berbarengan.

## Login

Kredensial owner sudah ditulis ke `.env` (di-gitignore, mode 600) — **tidak ikut ke GitHub**.
Akun terbentuk saat `npm run db:seed` dijalankan pertama kali di database yang masih kosong.

> **Jebakan yang perlu diingat**: `seed.js` memakai `upsert` dengan `update: {}`. Artinya kalau
> user dengan email itu **sudah ada**, seed ulang **tidak** mengganti passwordnya. Mau ganti
> password nanti? Lewat UI, atau hapus dulu barisnya di tabel `users`.

## Catatan operasional — jangan jalankan `git` dari sesi Cowork

Folder ini di-mount ke sesi Cowork lewat jembatan device, dan mount itu **tidak mengizinkan
unlink**. Akibatnya `git commit` di sana berhasil menulis commit tapi meninggalkan
`.git/HEAD.lock`, `.git/index.lock`, dan `tmp_obj_*` yang tak terhapus — operasi git
berikutnya langsung mati dengan *"Another git process seems to be running"*.

**Jalankan semua perintah git untuk repo ini dari Terminal macOS langsung.** Kalau terlanjur
macet, hapus manual: `rm -f .git/*.lock .git/objects/*/tmp_obj_* .git/objects/maintenance.lock`.

## Yang belum diverifikasi (jujur, jangan dianggap hijau)

- **Belum pernah di-build atau dijalankan.** Belum `npm install`, belum `tsc`, belum
  `docker compose up`. Perubahan di atas murni tekstual dan terukur, tapi bukti
  "benar-benar jalan" belum ada.
- **Cek port kosong tidak dilakukan di macOS.** Sesi Cowork ini cuma bisa menjalankan
  perintah di VM Linux, bukan di host Mac. Sebelum `up` pertama, jalankan sendiri:
  `lsof -nP -iTCP -sTCP:LISTEN | grep -E ':35(0[0-3]|75|80|90|93)'` — harus kosong.
- `docker-compose.prod.yml` masih memublikasikan **80/443** untuk nginx. Itu wajar untuk
  VPS berdomain, tapi jangan dipakai di laptop yang juga menjalankan Hermes lama.

## Langkah pertama

```bash
cd sentinel_wabot
# .env sudah disiapkan (JWT_SECRET acak + kredensial owner). Tinggal isi AI_API_KEY.
npm install
docker compose up -d postgres redis
npm run dev:api               # http://localhost:3501/api/v1
npm run dev:web               # http://localhost:3500
```

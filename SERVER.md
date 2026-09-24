# Server: romzakaz.uz

Sayt serveri haqida qisqa ma'lumot. Bu faylda **sirlar yo'q**: kalitlar, tokenlar va secret qiymatlari bu yerga yozilmaydi.

Holat sanasi: 2026-09-24

## Umumiy

| | |
|---|---|
| Provayder | DigitalOcean, Frankfurt (`fra1`) |
| IP | `159.65.114.9` |
| Hostname | `romzakaz-server` |
| Resurslar | 1 vCPU, 1 GB RAM, 24 GB disk |
| OS | Ubuntu 24.04.5 LTS, yadro 6.8.0-142 |
| Vaqt zonasi | UTC (Toshkent = UTC+5) |

## O'rnatilgan dasturlar

| Dastur | Versiya | Manba |
|---|---|---|
| Caddy | v2.11.4 | Caddy rasmiy repozitoriysi |
| Node.js | v24.21.0 (LTS), npm 11.19.0 | NodeSource |
| Docker | 29.8.1, Compose 5.5.1 | Docker rasmiy repozitoriysi |
| git | 2.43.0 | Ubuntu |
| ufw | 0.36.2 | Ubuntu |

Node.js va Docker hozircha ishlatilmayapti, ular keyingi darslar uchun o'rnatilgan.

Server `server/bootstrap.sh` skripti bilan sozlangan. Bu skript kurs papkasida saqlanadi, shu repoda emas.

## Domen va DNS (Cloudflare)

| Turi | Nomi | Qiymati | Proxy |
|---|---|---|---|
| A | `@` (romzakaz.uz) | `159.65.114.9` | Proxied (to'q sariq bulut) |
| A | `www` | `159.65.114.9` | Proxied (to'q sariq bulut) |

- Nameserver'lar: `kay.ns.cloudflare.com`, `rommy.ns.cloudflare.com`
- SSL/TLS rejimi: **Full (strict)** bo'lishi kerak. **Flexible** rejimini tanlamang, aks holda sayt cheksiz yo'naltirishga tushib qoladi. Joriy rejimni Cloudflare panelida tekshiring.

## Xizmatlar va portlar

| Xizmat | Port | Kimga ochiq |
|---|---|---|
| SSH (`ssh`) | 22/tcp | Internet |
| Caddy | 80/tcp, 443/tcp | Internet |
| Caddy boshqaruv API | 2019/tcp | Faqat server ichida (`127.0.0.1`) |
| systemd-resolved (DNS) | 53 | Faqat server ichida |
| Docker | — | Hozircha konteyner yo'q |

## Caddy va sayt

- Sayt papkasi: `/var/www/romzakaz`. Bu repoga `git clone` qilingan, egasi `deploy`.
- Konfiguratsiya: `/etc/caddy/Caddyfile`. Eski nusxalari shu papkada `Caddyfile.bak-<sana>` nomi bilan saqlangan.
- `romzakaz.uz` papkadagi fayllarni statik sayt sifatida ko'rsatadi.
- `www.romzakaz.uz` → `https://romzakaz.uz` (301 bilan doimiy yo'naltirish).
- HTTPS sertifikatlari: Let's Encrypt. Caddy ularni o'zi oladi va o'zi yangilaydi.
- Internetdan yashirilgan fayllar (404 qaytaradi): `.git`, `.github`, `.gitignore`, `CLAUDE.md`, `SERVER.md`, `run.bat`.
  **Repoga saytga tegishli bo'lmagan yangi fayl qo'shsangiz, uni ham shu ro'yxatga qo'shing.**
- Caddyfile'ni o'zgartirgandan keyin: `sudo caddy validate --config /etc/caddy/Caddyfile`, so'ng `sudo systemctl reload caddy`.

## Firewall (ufw)

- Kiruvchi ulanishlar: hammasi yopiq, faqat **22, 80 va 443** ochiq (IPv4 va IPv6).
- Chiquvchi ulanishlar: hammasi ochiq.
- Diqqat: Docker konteynerdan tashqariga ochgan portlar (`-p`) ufw qoidalarini chetlab o'tadi.

## SSH

- **root orqali kirish o'chirilgan** (`PermitRootLogin no`).
- Parol bilan kirish o'chirilgan, faqat SSH kalit bilan kiriladi.
- X11 forwarding o'chirilgan.
- Sozlama fayli: `/etc/ssh/sshd_config.d/01-hardening.conf`.
- Serverga faqat `deploy` foydalanuvchisi sifatida kiriladi, root huquqi kerak bo'lsa `sudo` ishlatiladi (parol so'ramaydi):
  ```
  ssh -i C:\Users\Dell\.ssh\id_ed25519_romzakaz deploy@159.65.114.9
  ```
- `deploy`'dagi kalitlar:
  1. Egasining shaxsiy kaliti.
  2. GitHub Actions kaliti. U **cheklangan**: faqat `git -C /var/www/romzakaz pull --ff-only` buyrug'ini bajara oladi.

## Swap

- `/swapfile`, 2 GB, `/etc/fstab` orqali qayta yuklangandan keyin ham yoqiladi.
- `vm.swappiness=10`: swap faqat RAM tugab qolganda ishlatiladi.

## Avtomatik yangilanishlar

- `unattended-upgrades` yoqilgan: xavfsizlik yangilanishlari har kuni o'zi o'rnatiladi.
- Yangilanish qayta yuklashni talab qilsa, server **23:00 UTC = 04:00 Toshkent vaqtida** o'zi qayta yuklanadi. Sayt 1–2 daqiqa ishlamay turadi.
- Sozlama fayllari: `/etc/apt/apt.conf.d/20auto-upgrades`, `/etc/apt/apt.conf.d/52unattended-reboot`.

## Avtodeploy (GitHub Actions)

1. `main` branch'ga `git push` qilinadi.
2. `.github/workflows/deploy.yml` ishga tushadi. Uni GitHub'dagi **Actions → Deploy → Run workflow** orqali qo'lda ham ishga tushirsa bo'ladi.
3. Workflow serverga `deploy` sifatida SSH orqali ulanadi, serverda `git pull --ff-only` bajariladi.
4. Caddy yangi fayllarni darhol ko'rsatadi, qayta yuklash kerak emas.

GitHub Secrets (qiymatlari faqat GitHub'da saqlanadi):

| Secret | Nima |
|---|---|
| `DEPLOY_SSH_KEY` | GitHub Actions uchun alohida yopiq kalit |
| `DEPLOY_KNOWN_HOSTS` | Serverning "barmoq izi". `StrictHostKeyChecking` yoqilgan |

Server qayta o'rnatilsa yoki uning SSH kaliti o'zgarsa, `DEPLOY_KNOWN_HOSTS` secret'ini yangilash kerak.

## Loglar

| Nima | Buyruq / joy |
|---|---|
| Caddy (sayt, sertifikatlar) | `sudo journalctl -u caddy -f` |
| SSH kirishlari | `sudo journalctl -u ssh --since today` |
| Avtomatik yangilanishlar | `/var/log/unattended-upgrades/unattended-upgrades.log` |
| apt tarixi | `/var/log/apt/history.log` |
| Firewall | `sudo ufw status verbose`, bloklanganlar: `sudo journalctl -k \| grep UFW` |
| Docker | `sudo journalctl -u docker` |
| Avtodeploy | GitHub → repo → **Actions** bo'limi |
| Server bootstrap'i | `sudo less /root/bootstrap.log` |
| Umumiy holat | `df -h /`, `free -h`, `systemctl --failed` |

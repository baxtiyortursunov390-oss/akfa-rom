# Server: romzakaz.uz

Sayt serveri haqida qisqa ma'lumot. Bu faylda **sirlar yo'q**: kalitlar, tokenlar va secret qiymatlari bu yerga yozilmaydi.

Holat sanasi: 2026-09-25 (Telegram bot qo'shildi)

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

Node.js Telegram botini ishlatadi ([pastda](#telegram-bot-romzakaz007bot)). Docker hozircha ishlatilmayapti.

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
| Telegram bot (`romzakaz-bot`) | 3000/tcp | Faqat server ichida (`127.0.0.1`), tashqaridan faqat Caddy orqali |
| systemd-resolved (DNS) | 53 | Faqat server ichida |
| Docker | — | Hozircha konteyner yo'q |

## Caddy va sayt

- Sayt papkasi: `/var/www/romzakaz`. Bu repoga `git clone` qilingan, egasi `deploy`.
- Konfiguratsiya: `/etc/caddy/Caddyfile`. Eski nusxalari shu papkada `Caddyfile.bak-<sana>` nomi bilan saqlangan.
- `romzakaz.uz` papkadagi fayllarni statik sayt sifatida ko'rsatadi.
- `www.romzakaz.uz` → `https://romzakaz.uz` (301 bilan doimiy yo'naltirish).
- `romzakaz.uz/tg/<maxfiy>` → Telegram bot (`127.0.0.1:3000`). Aniq yo'l sir, u faqat serverda saqlanadi: Caddyfile va botning `.env` fayli.
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
  2. Sayt uchun GitHub Actions kaliti. U **cheklangan**: faqat `git -C /var/www/romzakaz pull --ff-only` buyrug'ini bajara oladi.
  3. Bot uchun GitHub Actions kaliti. U **cheklangan**: faqat `/usr/local/bin/romzakaz-bot-deploy` skriptini ishga tushira oladi.

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

## Telegram bot (@romzakaz007bot)

| | |
|---|---|
| Repo | `baxtiyortursunov390-oss/romzakaz-bot` (**private**) |
| Papka | `/srv/romzakaz-bot`, egasi `deploy`. `/var/www`da emas, shuning uchun internetga chiqmaydi |
| Xizmat | `romzakaz-bot.service` (systemd): `deploy` nomidan ishlaydi, `Restart=always`, server qayta yuklanganda o'zi yonadi |
| Rejim | Webhook: Telegram → Cloudflare → Caddy (`/tg/<maxfiy>`) → `127.0.0.1:3000` |
| Himoya | Telegram har so'rovda `secret_token` yuboradi, noto'g'ri bo'lsa bot 401 qaytaradi |
| Sirlar | `/srv/romzakaz-bot/.env` (`chmod 600`): bot tokeni, AI kaliti, egasining chat ID'si, webhook siri. Repoda yo'q |
| AI | Gemini (`gemini-3.5-flash-lite`); xarakter — `xarakter.md`, bilim bazasi — `bilim/` |

**Muhim:** botni kompyuterda `npm start` (polling) bilan ishga tushirmang, bu serverdagi webhook'ni o'chirib qo'yadi. Kod buni o'zi tekshiradi va webhook o'rnatilgan bo'lsa, ishga tushmaydi. `FORCE_POLLING=1` bilan majburan ishga tushirsangiz, keyin serverda `sudo systemctl restart romzakaz-bot` qiling.

Boshqarish:
```
sudo systemctl status romzakaz-bot      # holati
sudo systemctl restart romzakaz-bot     # qayta ishga tushirish
sudo journalctl -u romzakaz-bot -f      # loglar
```

### Bot avtodeploy'i
1. Bot repo'sida `main`ga `git push` qilinadi.
2. `.github/workflows/deploy.yml` serverga cheklangan kalit bilan ulanadi.
3. Serverda `/usr/local/bin/romzakaz-bot-deploy` ishlaydi (egasi root, repodan tashqarida):
   - `git pull --ff-only`;
   - `npm ci --omit=dev`;
   - xizmat qayta ishga tushiriladi.

   Xizmat 5 soniyada `active` bo'lmasa, Actions qizil bo'ladi.
4. Server repodan kodni **faqat o'qish huquqli deploy key** bilan oladi: `~/.ssh/romzakaz_bot_repo`, SSH taxallusi `github-romzakaz-bot`.

Bot repo'sidagi secret'lar: `DEPLOY_SSH_KEY` va `DEPLOY_KNOWN_HOSTS` (sayt repo'sidagi bilan bir xil ma'noda, lekin kalit boshqa).

Eslatma: suhbat xotirasi va vaqt kutayotgan mijoz raqamlari faqat RAM'da saqlanadi. Xizmat qayta ishga tushganda ular tozalanadi.

## Loglar

| Nima | Buyruq / joy |
|---|---|
| Telegram bot | `sudo journalctl -u romzakaz-bot -f` |
| Caddy (sayt, sertifikatlar) | `sudo journalctl -u caddy -f` |
| SSH kirishlari | `sudo journalctl -u ssh --since today` |
| Avtomatik yangilanishlar | `/var/log/unattended-upgrades/unattended-upgrades.log` |
| apt tarixi | `/var/log/apt/history.log` |
| Firewall | `sudo ufw status verbose`, bloklanganlar: `sudo journalctl -k \| grep UFW` |
| Docker | `sudo journalctl -u docker` |
| Avtodeploy (sayt va bot) | GitHub → tegishli repo → **Actions** bo'limi |
| Server bootstrap'i | `sudo less /root/bootstrap.log` |
| Umumiy holat | `df -h /`, `free -h`, `systemctl --failed` |

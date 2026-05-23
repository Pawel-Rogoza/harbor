# Środowisko projektowe: sudormrf-server

Plik ten stanowi główny dokument onboardingowy dla modeli LLM. Zawiera profil administratora, konwencje pracy, aktualny stan infrastruktury oraz historię decyzji architektonicznych. Zapoznaj się z poniższymi danymi przed rozpoczęciem realizacji kolejnych kroków.

## 👤 Profil i cele projektu
- **Właściciel**: Paweł, Customer Support Technician w cyber_Folks (zespół cyberAdmin — hosting, web, mail, Linux).
- **Cel edukacyjny**: Samodzielna administracja serwerem hostingowym od podstaw (web stack, security, mail, monitoring, IaC) w celu przejścia do roli DevOps Engineering / Senior Tech.
- **Cel komercyjny**: Stabilne środowisko pod hosting małych stron WWW dla własnych klientów.
- **Repozytorium**: https://gitlab.com/Pawel-Rogoza/sudormrf-hosting
- **Środowisko lokalne**: Windows + WSL2 (Ubuntu) + VS Code z rozszerzeniem WSL. Autoryzacja przez klucze ED25519.

## 🛠️ Specyfikacja VPS (Hetzner Cloud)
- **Instancja**: CPX23 (Falkenstein)
- **Zasoby**: 3 vCPU AMD, 4 GB RAM, 40 GB NVMe (w pełni alokowane)
- **System operacyjny**: AlmaLinux 10.1 "Heliotrope Lion"
- **Tożsamość**: Hostname `s1.sudormrf.pl`, domena operatorska `sudormrf.pl`
- **Backup storage**: Hetzner Storage Box BX11 (planowane)

## 📋 Konwencje pracy i komunikacji
- **Język**: Pracujemy po polsku, zachowując oryginalne nazewnictwo techniczne (vhost, reverse proxy, hardening, handshake itp.).
- **Styl odpowiedzi**: Konkretny, techniczny, bez owijania w bawełnę i pochlebstw. Jeśli czegoś nie wiesz – mów wprost. Jeśli pomysł jest zły – zaoferuj merytoryczną kontrę.
- **Edukacja przede wszystkim**: Przy każdej komendzie, dyrektywie (nginx/Apache/Postfix) czy potoku w Bashu tłumacz mechanizm działania pod spodem (warstwa jądra, sieciowa, protokoły).
- **Strategia Git**: Małe, atomowe commity (`Etap N: opis` lub `docs:` / `fix:`). Praca na branchach per etap, merge do main na koniec. Brak haseł i sekretów w repozytorium.

---

## 🗂️ Status etapów

| Etap | Temat | Status | Data ukończenia |
|------|-------|--------|-----------------|
| 0a   | Setup GitLab + struktura repo | ✅ Ukończone | 02.05.2026 |
| 0b   | Hardening systemu | ✅ Ukończone | 23.05.2026 |
| 1    | Web stack (nginx + Apache + PHP-FPM + MariaDB) | ✅ Ukończone | 23.05.2026 |
| 2    | SSL / TLS Termination | ✅ Ukończone | 23.05.2026 |
| 3    | Separacja klientów | ⏳ Planowane | - |
| 4    | Bezpieczeństwo (mod_security, malware, CSF) | ⏳ Planowane | - |
| 5    | Monitoring (Netdata → Prometheus + Grafana) | ⏳ Planowane | - |
| 6    | Backupy (restic + Hetzner Storage Box) | ⏳ Planowane | - |
| 7    | Mail server (Postfix + Dovecot + Rspamd) | ⏳ Planowane | - |
| 8    | Ansible — przepisanie konfiguracji do IaC | ⏳ Planowane | - |
| 9    | (opcjonalnie) Panel HestiaCP / własny DNS / GitLab Pages | ⏳ Planowane | - |

Legenda: ⏳ planowane · 🔄 w trakcie · ✅ ukończone · ⚠️ wymaga rewizji

---

## 🖥️ Stan infrastruktury

### Serwer bazowy
- **OS/Kernel**: AlmaLinux 10.1 (Heliotrope Lion) / Kernel 6.12.0-124.21.1.el10_1.x86_64
- **Użytkownicy**: `root` (zablokowany w SSH), `captain` (roboczy, grupa wheel, pełne sudo bez hasła w `/etc/sudoers.d/captain`).
- **Zapora sieciowa (firewalld)**: Aktywny profil publiczny. Otwarte usługi i porty: `2137/tcp` (Custom SSH), `http` (port 80/tcp), `https` (port 443/tcp).

### Usługi systemowe (systemd)
- `sshd.service` (OpenSSH, nasłuch na porcie 2137, autoryzacja wyłącznie przez klucz, AllowUsers captain)
- `firewalld.service` (Zarządzanie filtrowaniem pakietów nftables)
- `chronyd.service` (Synchronizacja czasu NTP, strefa `Europe/Warsaw`)
- `dnf-automatic.timer` (Automatyczne pobieranie i wdrażanie aktualizacji bezpieczeństwa systemowych)
- `qemu-guest-agent.service` (Integracja z hiperwizorem KVM Hetznera)
- `mariadb.service` (Baza danych MariaDB 11.4, nasłuch lokalny / Unix socket)
- `php85-php-fpm.service` (Interpreter PHP 8.5 z repozytorium Remi SCL)
- `httpd.service` (Backend WWW Apache 2.4, Event MPM, nasłuch na 127.0.0.1:8080)
- `nginx.service` (Frontend WWW Nginx, nasłuch na 0.0.0.0:80 oraz 0.0.0.0:443)

### Repozytoria i pliki konfiguracyjne
- **Repozytoria**: BaseOS, AppStream, CRB, EPEL, Remi-Safe, Remi-Modular
- **Pliki krytyczne**:
  - `/etc/ssh/sshd_config.d/01-hardening.conf` (Konfiguracja SSH)
  - `/etc/dnf/automatic.conf` (Konfiguracja automatycznych aktualizacji)
  - `/etc/cloud/templates/hosts.redhat.tmpl` (Szablon generowania pliku hosts)
  - `/etc/nginx/conf.d/s1.sudormrf.pl.conf` (Vhost frontendu Nginx, terminacja SSL, nagłówki Real-IP, proxy do portu 8080)
  - `/etc/httpd/conf.d/php-fpm.conf` (Integracja Apache z socketem PHP-FPM za pomocą mod_proxy_fcgi)
  - `/var/opt/remi/php85/run/php-fpm/www.sock` (Główny uniksowy socket PHP-FPM, uprawnienia zarządzane przez POSIX ACL: listen.acl_users = apache)

---

## 🏗️ Specyfikacja Stosu Webowego i Szyfrowania

### 1. Architektura Reverse Proxy
Stos rozdziela warstwę sieciową i serwowanie statyki od logiki backendowej.
- Frontend (Nginx): Słucha świata zewnętrznego (0.0.0.0). Pliki statyczne (.jpg, .css, .js itp.) serwuje samodzielnie bezpośrednio z dysku NVMe. Ruch dynamiczny i pozostałe zapytania przekazuje lokalnie do Apache. Wstrzykuje nagłówki tożsamości klienta: Host, X-Real-IP, X-Forwarded-For, X-Forwarded-Proto.
- Backend (Apache): Słucha wyłącznie na pętli zwrotnej (127.0.0.1:8080). Przetwarza pliki .htaccess w katalogach domen. Skrypty .php deleguje do interpretera PHP-FPM przez protokół FastCGI przy użyciu lokalnego gniazda domeny uniksowej (Unix Domain Socket).

### 2. Szyfrowanie i Certyfikaty SSL/TLS
- Urząd Certyfikacji: Let's Encrypt, automatyzacja za pomocą Certbota (Protokół ACME, HTTP-01 challenge na porcie 80).
- Ścieżki certyfikatu domeny operatorskiej s1.sudormrf.pl:
  - Certyfikat i łańcuch: /etc/letsencrypt/live/s1.sudormrf.pl/fullchain.pem
  - Klucz prywatny: /etc/letsencrypt/live/s1.sudormrf.pl/privkey.pem
- Bezpieczeństwo: Terminacja połączeń SSL następuje całkowicie na poziomie Nginxa. Wymuszone profile protokołów TLSv1.2 oraz TLSv1.3 (plik options-ssl-nginx.conf). Aktywny mechanizm Perfect Forward Secrecy (PFS) poprzez dedykowany plik parametrów Diffiego-Hellmana (/etc/letsencrypt/ssl-dhparams.pem).

---

## 🚧 Otwarte wątki / decyzje na później

- Implementacja pełnej izolacji i uprawnień dla nowych użytkowników systemowych (Etap 3).

---

## 📚 Linki do kluczowych notatek

- [README projektu](../README.md)
- [Etap 0 — Hardening](./stages/00-hardening.md)
- [Etap 1 — Web Stack](./stages/01-web-stack.md)
- [Etap 2 — SSL Encryption](./stages/02-ssl-encryption.md)

---

## 📝 Historia ważnych decyzji

- **2026-05-02** — Hetzner Cloud (CPX23, Falkenstein). Powód: doskonały stosunek ceny do wydajności, 20 TB transferu, świetna integracja z Storage Box oraz Ansible.
- **2026-05-02** — AlmaLinux 10. Powód: zgodność ze środowiskiem produkcyjnym w pracy (cyber_Folks), głęboka nauka SELinux, stabilność ekosystemu RHEL.
- **2026-05-02** — Apache + nginx (reverse proxy). Powód: nginx jako kluczowe narzędzie DevOps do terminowania SSL i cachowania, Apache ze względu na kompatybilność wsteczną (.htaccess) dla przyszłych stron klientów.
- **2026-05-23** — Implementacja PHP 8.5 z repozytorium Remi jako pakiety Software Collections (SCL) ze względu na wygaszenie tradycyjnej modułowości DNF w EL10 oraz bezpieczne zarządzanie uprawnieniami socketu uniksowego przez POSIX ACL.

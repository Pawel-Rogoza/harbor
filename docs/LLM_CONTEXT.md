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
| 1    | Web stack (nginx + Apache + PHP-FPM + MariaDB) | ⏳ Planowane | - |
| 2    | SSL + WordPress + Redis | ⏳ Planowane | - |
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
- **Zapora sieciowa (firewalld)**: Aktywny profil publiczny, otwarty wyłącznie port `2137/tcp`. Usługa standardowa SSH (port 22) usunięta z zapory.

### Usługi systemowe (systemd)
- `sshd.service` (OpenSSH, nasłuch na porcie 2137, autoryzacja wyłącznie przez klucz, AllowUsers captain)
- `firewalld.service` (Zarządzanie filtrowaniem pakietów nftables)
- `chronyd.service` (Synchronizacja czasu NTP, strefa `Europe/Warsaw`)
- `dnf-automatic.timer` (Automatyczne pobieranie i wdrażanie aktualizacji bezpieczeństwa systemowych)
- `qemu-guest-agent.service` (Integracja z hiperwizorem KVM Hetznera)

### Repozytoria i pliki konfiguracyjne
- **Repozytoria**: BaseOS, AppStream, CRB, EPEL
- **Pliki krytyczne**:
  - `/etc/ssh/sshd_config.d/01-hardening.conf` (Konfiguracja SSH)
  - `/etc/dnf/automatic.conf` (Konfiguracja automatycznych aktualizacji)
  - `/etc/cloud/templates/hosts.redhat.tmpl` (Szablon generowania pliku hosts z zachowaniem publicznego IP pod zmiennymi fqdn/hostname)

---

## 🚧 Otwarte wątki / decyzje na później

- (puste)

---

## 📚 Linki do kluczowych notatek

- [README projektu](../README.md)
- [Etap 0 — Hardening](./stages/00-hardening.md)

---

## 📝 Historia ważnych decyzji

- **2026-05-02** — Hetzner Cloud (CPX23, Falkenstein). Powód: doskonały stosunek ceny do wydajności, 20 TB transferu, świetna integracja z Storage Box oraz Ansible.
- **2026-05-02** — AlmaLinux 10. Powód: zgodność ze środowiskiem produkcyjnym w pracy (cyber_Folks), głęboka nauka SELinux, stabilność ekosystemu RHEL.
- **2026-05-02** — Apache + nginx (reverse proxy). Powód: nginx jako kluczowe narzędzie DevOps do terminowania SSL i cachowania, Apache ze względu na kompatybilność wsteczną (.htaccess) dla przyszłych stron klientów.

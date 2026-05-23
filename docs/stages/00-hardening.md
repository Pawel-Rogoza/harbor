# Etap 0 — Hardening systemu i setup roboczy

**Data rozpoczęcia**: 02.05.2026
**Cel**: Bezpieczna podstawa serwera + setup narzędzi (git, dokumentacja).

## Kontekst

Świeżo postawiony VPS Hetzner CPX23 z AlmaLinux 10.1 w Falkenstein.
Dysk 40 GB NVMe w pełni alokowany przez domyślny obraz startowy dostawcy.
Domena: s1.sudormrf.pl, PTR ustawiony, DNS A record działa.

## Plan etapu

- [x] Setup GitLab + struktura repo
- [x] Pierwsze logowanie i orientacja w systemie
- [x] Aktualizacja systemu
- [x] Utworzenie użytkownika roboczego (sudo, klucz SSH)
- [x] Hardening SSH (wyłączenie roota, haseł, port)
- [x] firewalld - podstawowa konfiguracja
- [x] Automatyczne aktualizacje bezpieczeństwa (dnf-automatic)
- [x] Time sync (chrony)
- [x] Hostname, hosts, locale
- [x] Podstawowy monitoring (htop, glances, journalctl orientacja)
- [x] Commit do gita

## Notatki techniczne

### Usługi i pakiety
- Obraz minimalny Hetznera nie zawierał narzędzi SELinux oraz demona firewalld. Zainstalowano pakiety:
  - `policycoreutils-python-utils` (dla polecenia `semanage`)
  - `firewalld` (zarządzanie nftables)
  - `dnf-automatic` (automatyczne aktualizacje)
  - `htop` (z repozytorium EPEL)

### Automatyczne aktualizacje
Skonfigurowano `/etc/dnf/automatic.conf` do pobierania i automatycznego wdrażania wyłącznie poprawek bezpieczeństwa (`upgrade_type = security`), minimalizując ryzyko regresji usług na produkcji. Włączono timer systemd: `dnf-automatic.timer`.

### Kontrola dostępu i SSH (Port 2137)
- Utworzono użytkownika systemowego `captain` z powłoką `/bin/bash` i dodano go do grupy `wheel`.
- Skonfigurowano dedykowaną regułę sudoers w `/etc/sudoers.d/captain` z flagą `NOPASSWD:ALL` pod kątem przyszłej automatyzacji w Ansible.
- Wdrożono plik `/etc/ssh/sshd_config.d/01-hardening.conf`:
  - Przeniesiono SSH na port `2137`.
  - Wyłączono bezpośrednie logowanie na konto root (`PermitRootLogin no`).
  - Całkowicie wyłączono autoryzację hasłem (`PasswordAuthentication no`), wymuszając klucz ED25519.
  - Wprowadzono ścisłą białą listę użytkowników (`AllowUsers captain`).

### SELinux & firewalld
- Zarejestrowano nowy port SSH w polityce SELinux, aby jądro zezwoliło procesowi `sshd_t` na bindowanie gniazda TCP:
  `semanage port -a -t ssh_port_t -p tcp 2137`
- Skonfigurowano zaporę sieciową, otwierając nowy port i usuwając standardową usługę SSH (port 22):
  `firewall-cmd --permanent --zone=public --add-port=2137/tcp`
  `firewall-cmd --permanent --zone=public --remove-service=ssh`
  `firewall-cmd --reload`

### Tożsamość sieciowa (cloud-init i /etc/hosts)
- Zmieniono strefę czasową na `Europe/Warsaw`.
- Ustawiono statyczny hostname na `s1.sudormrf.pl`.
- Ze względu na aktywny moduł `manage_etc_hosts` w `cloud-init`, mapowanie lokalnego IP na FQDN wdrożono bezpośrednio w szablonie `/etc/cloud/templates/hosts.redhat.tmpl` (Wariant A), przypisując publiczny adres IP serwera do zmiennych `{{fqdn}}` i `{{hostname}}`. Zapobiega to nadpisaniu pliku `/etc/hosts` przy reboocie.

## Wnioski

System został skutecznie zabezpieczony na warstwie dostępowej (L3/L4 przez firewalld oraz L7 przez restrykcje OpenSSH). Jądro Linuksa poprzez SELinux aktywnie izoluje proces demona SSH. Serwer posiada pełną autonomię tożsamości sieciowej i poprawną synchronizację czasu (chrony), co stanowi stabilną i bezpieczną bazę pod instalację stosu webowego w kolejnym etapie.

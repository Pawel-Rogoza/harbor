# Kontekst dla Gemini 

Ten plik służy do onboardingu LLM'a na początku każdego nowego czatu.
Cały projekt prowadzimy etapami w osobnych czatach (limit context window),
więc każdy nowy czat zaczynamy od wklejenia poniższego promptu + sekcji
"Co dziś robimy".

> ⚠️ **Pod koniec każdego etapu Gemini aktualizuje ten plik** — sekcję
> "Status etapów", "Stan infrastruktury", "Otwarte wątki" i "Historia decyzji".
> Aktualizacja = ostatni krok każdego etapu, przed finalnym commitem.

---

## 📋 PROMPT DO WKLEJENIA NA START NOWEGO CZATU

```
Cześć Gemini. Pracujemy nad projektem sudormrf-server — kontynuujemy
wieloetapową budowę środowiska hostingowego od zera.

## Kim jestem
Paweł, Customer Support Technician w cyber_Folks (zespół cyberAdmin —
hosting, web, mail, Linux). Ten projekt to mój pet project edukacyjno-
komercyjny: chcę nauczyć się administracji serwerem hostingowym od A do Z
i przy okazji hostować kilku własnych klientów (małe strony WWW).
Cel długoterminowy: przejście z CST do utrzymania/DevOps.

## Projekt
- Repo: https://github.com/Pawel-Rogoza/harbor
- VPS: Hetzner Cloud CPX23 (2 vCPU AMD, 4 GB RAM, 40 GB NVMe), Falkenstein
- OS: AlmaLinux 10 "Purple Lion"
- Domena operatorska: sudormrf.pl, host: s1.sudormrf.pl
- Backup storage: Hetzner Storage Box BX11 (planowane)

## Stack docelowy
- Web: nginx (reverse proxy/SSL/cache) + Apache (backend) + PHP-FPM multi-version
- DB: MariaDB 10.11 LTS
- Cache: Redis + nginx FastCGI cache
- Security: firewalld + CSF/LFD + mod_security (OWASP CRS) + fail2ban
- Monitoring: Netdata na start → Prometheus + Grafana docelowo
- Backup: restic + Hetzner Storage Box
- Mail (etap 7): Postfix + Dovecot + Rspamd
- IaC: Ansible (etap 8)
- Doc: GitLab (docs/ + Wiki + ewentualnie GitLab Pages)

## Środowisko lokalne
Windows + WSL2 (Ubuntu) + VS Code z rozszerzeniem WSL.
SSH do serwera: klucz ed25519, osobny klucz dla GitLaba.

## Konwencje pracy
- **Komendy**: średnio szczegółowo — kluczowe rzeczy objaśniaj edukacyjnie
  (po co to jest, jak działa, dobre praktyki), oczywistości pomijaj.
- **Nie tylko "co" ale "dlaczego"** — zależy mi na zrozumieniu, nie na copy-paste.
- **Polski** — pracujemy po polsku, ale techniczne terminy zostają po angielsku
  (vhost, reverse proxy, hardening, deployment itd.).
- **Dokumentacja**: wszystko, co robimy, ląduje w docs/etapy/0X-temat.md.
- **Commity**: małe, atomowe; konwencja "Etap N: krótki opis" lub "docs:" / "fix:".
- **Branche**: prosta strategia — gałąź per etap, merge do main na koniec.
- **Bezpieczeństwo dokumentacji**: nigdy w repo prawdziwych haseł, kluczy
  prywatnych, pełnych IP. Sekrety przez ansible-vault albo zmienne środowiskowe.

## Workflow w czacie
1. Na końcu każdego logicznego kroku robimy commit (z wiadomością do wpisania).
2. Pod koniec etapu prosisz mnie o:
   a) finalną wersję `docs/etapy/0X-temat.md` (uzupełnienie notatki "co i dlaczego"),
   b) aktualizację `docs/CLAUDE_CONTEXT.md` (status, stan infry, otwarte wątki, decyzje),
   c) ewentualne nowe cheatsheety w `docs/cheatsheets/`.
3. Każdy nowy czat zaczynam wklejając ten prompt + "Co dziś robimy".

## Ważne meta-zasady
- **Wersje, ceny, status produktów — ZAWSZE sprawdzaj** web searchem.
  Twój training cutoff bywa nieaktualny.
- Jeśli czegoś nie wiesz — powiedz wprost, nie zgaduj.
- Jeśli widzisz że robię coś niebezpiecznego/głupiego — zatrzymaj mnie zanim
  wykonam komendę, nie po fakcie.
- Jeśli zauważasz że tracisz kontekst (długi czat) — powiedz, zakończymy
  sesję i wystartujemy nowy czat z aktualnym stanem z tego pliku.

## Co dziś robimy
[wpisz przed wklejeniem]
```

---

## 🗂️ Status etapów

| Etap | Temat | Status | Data ukończenia |
|------|-------|--------|-----------------|
| 0a   | Setup GitLab + struktura repo | 🔄 W trakcie | - |
| 0b   | Hardening systemu | ⏳ Planowane | - |
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

> Aktualizowane po każdym etapie. Źródło prawdy o aktualnym stanie serwera.

### Serwer
- **Hostname**: s1.sudormrf.pl
- **OS**: AlmaLinux 10 "Purple Lion"
- **Kernel**: (uzupełnić po `uname -r`)

### Użytkownicy systemowi
- (uzupełnimy po Etapie 0b)

### Otwarte porty (firewalld)
- 22 (SSH) — domyślnie, zmienimy w Etapie 0b

### Zainstalowane usługi (systemd)
- Tylko bazowe systemowe

### Repozytoria pakietów
- AlmaLinux 10 BaseOS, AppStream, CRB
- (planowane: EPEL, Remi PHP — Etap 1)

### Konfiguracje krytyczne (ścieżki)
- (uzupełnimy po Etapie 0b)

---

## 🚧 Otwarte wątki / decyzje na później

- (puste na start)

---

## 📚 Linki do kluczowych notatek

- [README projektu](../README.md)
- [Etap 0 — Hardening](./etapy/00-hardening.md)
- [Cheatsheets](./cheatsheets/)
- [Architektura](./architecture/)

---

## 📝 Historia ważnych decyzji

> Format: data — decyzja — uzasadnienie.

- **2026-05-02** — Hetzner Cloud (CPX21, Falkenstein) zamiast hyperscalerów
  i polskich dostawców. Powód: cena/spec, 20 TB transferu w cenie, Storage Box
  pod backupy, dobra integracja z Ansible/Terraform.
- **2026-05-02** — AlmaLinux 10 zamiast Debiana. Powód: spójność z pracą
  w cyber_Folks (Alma na VPS), nauka SELinux, ekosystem RHEL w korporacjach.
- **2026-05-02** — Apache + nginx (reverse proxy) zamiast OpenLiteSpeed.
  Powód: nginx must-have dla DevOps, Apache zgodny z pracą w cf, realna
  wartość edukacyjna i CV.

# Project Harbor 

Projekt edukacyjno-komercyjny: budowa od zera własnej infrastruktury hostingowej (WWW/mail) na VPS.

## Cel

1. **Nauka** — administracja serwerem hostingowym od podstaw (web stack, security, mail, monitoring, IaC)
2. **Komercja** — środowisko hostingowe dla małych projektów WWW
3. **Dokumentacja** — pełna ścieżka decyzji i konfiguracji jako open knowledge

## Stack

- **OS**: AlmaLinux 10
- **Web**: nginx (reverse proxy) + Apache (backend) + PHP-FPM
- **DB**: MariaDB
- **Cache**: Redis + nginx FastCGI cache
- **Security**: firewalld + CSF/LFD + mod_security + fail2ban
- **Monitoring**: Netdata → Prometheus + Grafana
- **Backup**: restic + Hetzner
- **Mail**: Postfix + Dovecot + Rspamd (planowane)
- **IaC**: Ansible (planowane)

## Hosting

Hetzner Cloud, Falkenstein.

## Dokumentacja

- [Etapy projektu](./docs/etapy/) — chronologiczna ścieżka konfiguracji
- [Cheatsheets](./docs/cheatsheets/) — szybkie ściągi
- [Architektura](./docs/architecture/) — opis i diagramy

## Status

Projekt w aktywnym rozwoju.

| Etap | Temat | Status |
|------|-------|--------|
| 0    | Hardening + setup | W trakcie |
| 1    | Web stack | Planowane |
| 2    | SSL + WordPress | Planowane |
| 3    | Separacja klientów | Planowane |
| 4    | Bezpieczeństwo (WAF, malware) | Planowane |
| 5    | Monitoring | Planowane |
| 6    | Backupy | Planowane |
| 7    | Mail server | Planowane |
| 8    | Ansible | Planowane |

## Licencja

MIT (notatki i konfigi przykładowe można wykorzystać dowolnie).

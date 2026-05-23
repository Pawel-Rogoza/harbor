# Stage 1: Web Stack Deployment (Nginx + Apache + PHP-FPM 8.5)

## 1. Topologia i Przepływ Ruchu (Network Architecture)
Wdrożono hybrydową strukturę serwera WWW typu Reverse Proxy. Rozdzielenie ról pomiędzy asynchroniczny frontend (Nginx) a elastyczny backend (Apache) pozwala na maksymalną optymalizację serwowania statyki z zachowaniem pełnej kompatybilności z mechanizmami hostingu współdzielonego (.htaccess).

Mechanizm przepływu żądania:
1. Nginx (Frontend): Nasłuchuje na masce bindowania 0.0.0.0 na porcie 80. Odpowiada za bezpośrednie serwowanie statyki (JPEG, PNG, CSS, JS) z dysku za pomocą wysoce wydajnego podsystemu wejścia/wyjścia. Pliki dynamiczne (.php) przekazuje dalej za pomocą dyrektywy proxy_pass do lokalnej pętli zwrotnej 127.0.0.1:8080.
2. Apache (Backend): Schowany na interfejsie loopback 127.0.0.1:8080. Odbiera czysty ruch HTTP od Nginxa. Przetwarza reguły przepisywania linków w plikach .htaccess (Event MPM).
3. PHP-FPM 8.5 (Backend Exec): Demon interpretera przetwarzający żądania przekazane przez Apache za pomocą protokołu FastCGI przez lokalne gniazdo uniksowe.

---

## 2. Głęboka Teoria: Gniazda Uniksowe vs Porty TCP/IP
Komunikacja IPC (Inter-Process Communication) pomiędzy serwerem HTTPd a interpreterem PHP-FPM została zrealizowana za pomocą gniazda uniksowego (Unix Domain Socket).

* Unix Domain Socket (.sock): Funkcjonuje jako wirtualny punkt końcowy osadzony w systemie plików i zarządzany w całości w przestrzeni jądra (Kernel Space). Transmisja danych zachodzi poprzez kopiowanie bloków pamięci RAM pomiędzy buforami procesów. Jądro systemu operacyjnego całkowicie pomija stos sieciowy (brak enkapsulacji w pakiety TCP, brak sum kontrolnych, brak numeracji sekwencyjnej i okien transmisji). Minimalizuje to cykle CPU i eliminuje opóźnienia.
* TCP Loopback Socket (127.0.0.1:9000): Wymaga pełnego narzutu sieciowego. Dane przechodzą przez wirtualny interfejs lo, angażując podsystem sieciowy do routowania pakietów wewnątrz systemu. Rozwiązanie to generuje narzut procesora i jest stosowane wyłącznie w skalowaniu poziomym (wieloserwerowym).

---

## 3. Implementacja PHP-FPM 8.5 (Remi) i Mechanizm POSIX ACL

### Zmiany w architekturze dystrybucji EL10
W systemie AlmaLinux 10 tradycyjny mechanizm wirtualnych strumieni modułowych DNF (AppStream Modules) został oznaczony jako przestarzały (deprecated) z uwagi na skomplikowanie zależności i wygaszanie w DNF5. Pakiety PHP 8.5 z repozytorium Remi są wdrażane jako jawnie wersjonowane pakiety Software Collections (SCL), co pozwala na bezkonfliktowe współistnienie wielu wersji PHP (Multi-PHP) na jednym serwerze.

### Kontrola Dostępu przez POSIX ACL (Access Control Lists)
W pakietach PHP 8.5 domyślnie porzucono mechanizm tradycyjnego definiowania właściciela pliku gniazda uniksowego za pomocą dyrektyw listen.owner i listen.group. Zamiast tego wdrożono zaawansowany podsystem POSIX ACL:

   listen.acl_users = apache

Mechanika działania: Standardowy model uprawnień Linuksa (UGO) pozwala przypisać do pliku tylko jednego właściciela i jedną grupę. POSIX ACL rozszerza ten system, umożliwiając jądru przypisanie precyzyjnych praw zapisu i odczytu (RW) do pliku socketu dla wielu niezależnych użytkowników systemowych. Usługa php85-php-fpm tworzy gniazdo w strukturze /var/opt/remi/php85/run/php-fpm/www.sock i bezpośrednio nadaje procesowi systemowemu apache pełne prawa dostępu, gwarantując separację i stabilność stosu bez rozluźniania uprawnień globalnych.

---

## 4. Przekazywanie Tożsamości: Transparentne Proxy i Real-IP
Ponieważ Apache odbiera połączenia wyłącznie z adresu 127.0.0.1, natywnie traci on informację o publicznym IP klienta. W celu zachowania pełnej przezroczystości dla mechanizmów bezpieczeństwa aplikacji (np. zabezpieczenia brute-force, firewalle aplikacyjne w PHP), Nginx wstrzykuje adres klienta do nagłówków HTTP:

   proxy_set_header Host $host;
   proxy_set_header X-Real-IP $remote_addr;
   proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
   proxy_set_header X-Forwarded-Proto $scheme;

Wyjaśnienie mechaniczne nagłówków:
* X-Real-IP: Przekazuje surowy, początkowy adres IP klienta ($remote_addr), który zainicjował potok TCP.
* X-Forwarded-For: Łańcuch adresów IP gromadzący dane o wszystkich pośrednikach (np. jeśli klient łączy się przez Cloudflare, jego prawdziwe IP jest doklejane do listy).
* X-Forwarded-Proto: Informuje backend, czy oryginalne zapytanie ze świata przyszło po protokole https czy http. Zapobiega to wpadaniu aplikacji w pętle przekierowań (redirection loops).

---

## 5. Procedury Weryfikacyjne i Komendy Diagnostyczne

### 1. Sprawdzenie stanów gniazd sieciowych (Złoty Standard):
   ss -tlnp | grep -E 'nginx|httpd'
Weryfikacja: Oczekiwany wynik to 0.0.0.0:80 powiązane z procesem nginx, oraz 127.0.0.1:8080 powiązane z procesem httpd.

### 2. Sprawdzenie uprawnień i obecności gniazda uniksowego:
   ls -la /var/opt/remi/php85/run/php-fpm/www.sock
   getfacl /var/opt/remi/php85/run/php-fpm/www.sock
Weryfikacja: getfacl musi jednoznacznie wskazać aktywne uprawnienie zapisu/odczytu (rw) dla użytkownika apache.

### 3. Weryfikacja parametrów interpretera PHP:
Wywołanie pliku z funkcją phpinfo() musi potwierdzić sekcje:
* Server API: FPM/FastCGI
* PHP Version: 8.5.x
* $_SERVER['SERVER_SOFTWARE']: Apache (potwierdzenie transparentności Nginxa jako proxy).

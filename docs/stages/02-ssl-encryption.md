# Stage 2: SSL/TLS Termination & Domain Setup

## 1. Topologia Szyfrowania (SSL Termination Architecture)
Bezpieczeństwo kryptograficzne wdrożono w modelu terminacji SSL na poziomie brzegowym (Nginx). 

Mechanizm działania:
1. Klient nawiązuje w pełni zaszyfrowane połączenie TLS z serwerem Nginx na porcie 443.
2. Nginx przetwarza uścisk dłoni (handshake), weryfikuje poprawność sesji i odszyfrowuje pakiety danych przy użyciu asymetrycznych kluczy serwera.
3. Odszyfrowany, bezpieczny ruch HTTP jest przekazywany wewnątrz systemu (localhost) do serwera Apache na port 8080.
4. Korzyść: Apache nie jest obciążany operacjami kryptograficznymi, co znacząco zmniejsza zużycie cykli procesora przez backend.

---

## 2. Protokół ACME i Weryfikacja Let's Encrypt
Automatyczne wystawianie i odnawianie certyfikatów zrealizowano za pomocą klienta Certbot, bazującego na protokole ACME (Automated Certificate Management Environment).

Mechanizm wyzwania HTTP-01 (Challenge):
1. Certbot wysyła żądanie wygenerowania kluczy dla domeny s1.sudormrf.pl do urzędu certyfikacji Let's Encrypt.
2. Let's Encrypt zwraca unikalny token i nakazuje jego publikację pod ścieżką http://s1.sudormrf.pl/.well-known/acme-challenge/
3. Wtyczka python3-certbot-nginx automatycznie modyfikuje konfigurację Nginxa w locie, wystawiając wymagany token.
4. Walidatory Let's Encrypt odpytują globalny DNS o rekord typu A domeny s1.sudormrf.pl, pobierają IP serwera, wykonują zapytanie na porcie 80 i weryfikują token. Po sukcesie certyfikaty zostają podpisane i przesłane na serwer.

---

## 3. Perfect Forward Secrecy (PFS) i ssl_dhparam
W konfiguracji wdrożono zaawansowane mechanizmy ochrony danych historycznych za pomocą parametrów Diffiego-Hellmana (ssl_dhparam).

* Bez PFS: Gdyby serwer używał standardowej wymiany kluczy opartej na czystym RSA, klucz prywatny serwera (privkey.pem) byłby bezpośrednio zaangażowany w szyfrowanie klucza sesji. W przypadku przechwycenia i zapisu zaszyfrowanego ruchu sieciowego przez atakującego, późniejsze zdobycie klucza prywatnego pozwoliłoby na wsteczne odszyfrowanie wszystkich historycznych sesji użytkowników.
* Z PFS (Perfect Forward Secrecy): Użycie protokołów ECDHE/DHE wymusza generowanie tymczasowych (efemerycznych) kluczy dla każdego uścisku dłoni osobno. Klucz prywatny serwera służy wyłącznie do podpisu i uwierzytelnienia tożsamości. Nawet jeśli klucz privkey.pem zostanie skompromitowany w przyszłości, historyczny ruch zapisany przez napastnika pozostaje niemożliwy do odszyfrowania, ponieważ klucze sesyjne zostały trwale zniszczone w pamięci RAM zaraz po zakończeniu połączenia.

---

## 4. Wdrożone Dyrektywy Kryptograficzne (Nginx Configuration)
Certbot zaimplementował do vhosta następujące parametry sterujące podsystemem TLS:

   listen 443 ssl;
   ssl_certificate /etc/letsencrypt/live/s1.sudormrf.pl/fullchain.pem;
   ssl_certificate_key /etc/letsencrypt/live/s1.sudormrf.pl/privkey.pem;
   include /etc/letsencrypt/options-ssl-nginx.conf;
   ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

Funkcja plików konfiguracyjnych:
* fullchain.pem: Zawiera publiczny certyfikat domeny oraz pełny łańcuch certyfikatów pośrednich urzędu certyfikacji.
* privkey.pem: Klucz prywatny serwera. Wymaga bezwzględnej ochrony uprawnień systemowych.
* options-ssl-nginx.conf: Globalny plik konfiguracyjny definiujący bezpieczne protokoły (wymuszenie TLSv1.2 oraz TLSv1.3) oraz nowoczesne pakiety szyfrów, eliminując podatne na ataki starsze standardy (SSLv3, TLSv1.0, TLSv1.1).

---

## 5. Procedury Weryfikacyjne
1. Sprawdzenie statusu nasłuchiwania na porcie zabezpieczonym:
   ss -tlnp | grep nginx
   Weryfikacja: Port *:443 musi być aktywny pod procesem nginx.
2. Sprawdzenie poprawności instalacji certyfikatu z poziomu zewnętrznego (OpenSSL CLI):
   openssl s_client -connect s1.sudormrf.pl:443 -tls1_3

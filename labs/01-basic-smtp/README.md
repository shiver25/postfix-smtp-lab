# Postfix Lab — lekcja 1: dwa serwery, kolejki i SMTP

Laboratorium do świadomej nauki Postfixa. **Bez Dovecota, TLS, SASL i publicznego SMTP** na tym etapie. Zaczynamy od dwóch serwerów oraz pytania: *kto przechowuje wiadomość, gdy odbiorca jest niedostępny?*

## Architektura

```text
 [mail-a] --SMTP TCP/25--> [mail-b] --> local(8) --> /var/mail/student
    |                         |
 queue-a                   queue-b

Docker network: mail-lab (internal: true)
Brak mapowania portów kontenerów na hosta.
```

- **mail-a**: nadawca. `relayhost = [mail-b]:25` kieruje każdą wiadomość do domen spoza `mydestination` do `mail-b`; nawiasy kwadratowe wyłączają wyszukiwanie MX dla next-hop.
- **mail-b**: odbiorca. `b.lab.test` jest w `mydestination`; doręczenie do lokalnego konta systemowego `student` realizuje transport `local(8)` — **jeszcze nie LMTP**.
- **Sieć**: Docker Compose `internal: true` ogranicza wyjście z sieci laboratorium; nie publikujemy portu 25. Laboratorium nadal ma dostęp do hosta przez narzędzia Docker, więc nie traktuj kontenerów jako granicy bezpieczeństwa dla nieufnego kodu.
- **Wolumeny**: kolejki obu Postfixów (`queue-a`, `queue-b`) oraz skrzynka (`mailbox-b`) zachowują dane pomiędzy typowymi restartami kontenerów.
- **Obrazy**: Debian 12 (`bookworm-slim`) i pakiet Postfix dystrybucji, nie gotowy „mail-server all-in-one”.

## Struktura

```text
postfix-smtp-lab/
├── README.md
└── labs/
    └── 01-basic-smtp/
        ├── compose.yaml
        ├── Dockerfile
        ├── README.md
        └── config/
            ├── mail-a/main.cf
            └── mail-b/main.cf
```

## Wymagania i uruchomienie

Docker Engine z pluginem `docker compose` oraz możliwość pobrania obrazu i pakietów **podczas budowania**. Sieć wewnętrzna dotyczy już działających serwisów, a nie pobierania zależności przy `docker compose build`.

```bash
docker compose config             # obejrzyj wyrenderowaną konfigurację
docker compose up -d --build
# po uruchomieniu:
docker compose ps
docker compose exec mail-a postconf myhostname mydestination relayhost
docker compose exec mail-b postconf myhostname mydestination default_transport
```

Jeżeli `docker compose up` zgłosi problem z `maillog_file`, sprawdź wersję Postfixa (`docker compose exec mail-a postconf mail_version`) i logi kontenera. Wykorzystujemy obsługę logowania do stdout przy `postfix start-fg`; nie jest ona dostępna we wszystkich starszych wydaniach.

## Eksperyment A: pojedyncza wiadomość

Nadaj wiadomość z kontenera `mail-a` za pomocą lokalnego interfejsu `sendmail` Postfixa:

```bash
printf 'From: tester@a.lab.test\nTo: student@b.lab.test\nSubject: Proba SMTP\n\nCzesc z mail-a.\n' \
  | docker compose exec -T mail-a sendmail -i -f tester@a.lab.test student@b.lab.test
```

Zajrzyj do kolejki i logów:

```bash
docker compose exec mail-a postqueue -p
docker compose exec mail-b postqueue -p
docker compose logs --tail=80 mail-a mail-b
```

Odczytaj skrzynkę lokalnego odbiorcy:

```bash
docker compose exec mail-b sh -c 'ls -l /var/mail/student; tail -n 40 /var/mail/student'
```

**Pytanie:** kiedy `mail-a` usuwa wiadomość z kolejki? Szukaj `status=sent` po potwierdzeniu przyjęcia przez `mail-b`. Dalsze dostarczenie przez `mail-b` jest już jego odpowiedzialnością.

## Eksperyment B: niedostępny serwer odbiorcy

Zatrzymaj `mail-b`, **potem** nadaj kolejną wiadomość z `mail-a`:

```bash
docker compose stop mail-b
printf 'From: tester@a.lab.test\nTo: student@b.lab.test\nSubject: Odbiorca offline\n\nTa wiadomosc ma poczekac.\n' \
  | docker compose exec -T mail-a sendmail -i -f tester@a.lab.test student@b.lab.test

docker compose exec mail-a postqueue -p
docker compose logs --tail=80 mail-a
```

**Oczekiwane:** `mail-a` nie otrzyma potwierdzenia SMTP od `mail-b`, więc przechowuje wiadomość w swojej kolejce (`deferred` po nieudanej próbie). Odczekaj chwilę, jeśli kolejka początkowo wygląda inaczej.

Włącz odbiorcę i przyspiesz ponowienie:

```bash
docker compose start mail-b
docker compose exec mail-a postqueue -f
docker compose logs --tail=80 mail-a mail-b
docker compose exec mail-b sh -c 'tail -n 40 /var/mail/student'
```

`postqueue -f` wymusza próbę doręczenia, nie zmienia konfiguracji. **Nie wykonuj takiego eksperymentu na produkcyjnym serwerze.**

## Eksperyment C: nieistniejący odbiorca

Wyślij na `nie-ma@b.lab.test`. `mail-b` powinien odpowiedzieć trwałą odmową dla nieznanego lokalnego odbiorcy, o ile domyślne mapowanie lokalnych kont pozostaje aktywne. Zobacz różnicę między odrzuceniem odbiorcy (`5xx`) a tymczasową niedostępnością (`4xx` / brak połączenia). Możesz obejrzeć ewentualną wiadomość zwrotną w kolejce/narzędziach lokalnych `mail-a`.

## Najważniejsze pojęcia

| Element | Rola |
| --- | --- |
| `smtpd` | Odbiera rozmowę SMTP na porcie 25. |
| `cleanup` | Przygotowuje przyjętą wiadomość do kolejki. |
| `qmgr` | Zarządza dostarczeniem wiadomości z kolejki. |
| `smtp(8)` | Klient Postfixa łączący się z drugim serwerem. |
| `local(8)` | Końcowe doręczenie do lokalnej skrzynki na `mail-b`. |
| `relayhost` | Jawnie skonfigurowany następny serwer SMTP. |
| `deferred` | Kolejka dla doręczeń odroczonych po nieudanych próbach. |

**Zasada odpowiedzialności:** po udanym przyjęciu `DATA` i odpowiedzi `250` dany serwer przejmuje odpowiedzialność za wiadomość. Jeśli nadawca nie otrzyma odpowiedzi, może ponowić próbę, co czasem prowadzi do duplikatów.

## Ograniczenia i dalsze lekcje

- W tej wersji **nie ma DNS/MX** — routing do `mail-b` jest statyczny i celowo uproszczony.
- Nie ma jeszcze **Dovecota ani LMTP**; dodamy je w kolejnej lekcji, gdy zrozumiemy ścieżkę Postfix → Postfix.
- Brak TLS oraz SMTP AUTH — laboratorium nie jest publicznym serwerem pocztowym.
- Nie włączaj na tym etapie publicznego przekierowania portu 25 ani `network_mode: host`.
- Następne kroki: `450` kontra `550`, retry/backoff, kontrolowana awaria doręczenia lokalnego, Dovecot/LMTP, DNS/MX, następnie TLS/SASL.

## Zatrzymanie i sprzątanie

```bash
docker compose down          # kontenery i sieć, bez usuwania wolumenów
docker compose down -v       # UWAGA: usuwa również kolejki i wiadomości!
```

Nie używaj `down -v`, gdy chcesz zachować wyniki eksperymentów.

## Eksperymenty z kolejką Postfixa

Podczas testów wywołaliśmy kilka różnych awarii dostarczania wiadomości. W każdym przypadku wiadomość pozostała w kolejce nadawcy, ale przyczyna niepowodzenia była inna.

| Scenariusz | DSN | Stan kolejki |
|---|---|---|
| Niedostępna nazwa DNS `mail-b` | `4.4.3` | `deferred` |
| Tymczasowy błąd odbiorcy `451` | `4.3.0` | `deferred` |
| Odmowa połączenia TCP/25 | `4.4.1` | `deferred` |
| Przywrócenie odbiorcy i skuteczne przekazanie | `2.0.0` | `sent` |

**Wniosek:** `deferred` oznacza odroczone doręczenie, ale nie wyjaśnia, dlaczego wiadomość nie została dostarczona.

Przyczyny szukamy w kodzie DSN i komunikacie diagnostycznym z logów.

Przykładowo:

- `4.4.3` — problem z systemem katalogowym lub DNS.
- `4.3.0` — tymczasowy problem systemu pocztowego.
- `4.4.1` — problem z nawiązaniem połączenia.
- `2.0.0` — operacja zakończona pomyślnie.

Po skutecznym przekazaniu wiadomości do następnego serwera Postfix usuwa ją ze swojej aktywnej kolejki. Odpowiedzialność za dalsze doręczenie przejmuje odbiorca SMTP.

**Uwaga:** wynik `sent` na `mail-a` oznacza skuteczne przekazanie wiadomości do `mail-b`, a niekoniecznie jej ostateczne dostarczenie do skrzynki użytkownika.

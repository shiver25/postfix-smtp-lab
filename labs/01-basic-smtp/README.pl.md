[English](README.md) | [Polski](README.pl.md)

# Postfix Lab — lekcja 1: dwa serwery, kolejki i SMTP

Praktyczne laboratorium pozwalające zrozumieć działanie Postfixa. Na tym etapie **nie korzystamy z Dovecota, TLS, SASL ani publicznego SMTP**.

Zaczynamy od dwóch serwerów pocztowych i jednego podstawowego pytania:

**Kto przechowuje wiadomość, gdy serwer odbiorcy jest niedostępny?**

## Architektura

```text
 [mail-a] --SMTP TCP/25--> [mail-b] --> local(8) --> /var/mail/student
    |                         |
 queue-a                   queue-b

Sieć Docker: mail-lab (internal: true)
Porty kontenerów nie są publikowane na hoście.
```

- **mail-a** — serwer nadawczy. `relayhost = [mail-b]:25` kieruje wiadomości dla domen spoza `mydestination` do `mail-b`. Nawiasy kwadratowe wyłączają wyszukiwanie rekordów MX dla następnego serwera.
- **mail-b** — serwer odbiorczy. Domena `b.lab.test` należy do `mydestination`. Wiadomości adresowane do lokalnego użytkownika systemowego `student` są dostarczane za pomocą transportu `local(8)` — **jeszcze nie LMTP**.
- **Sieć** — Docker Compose wykorzystuje `internal: true`, ograniczając łączność z sieciami zewnętrznymi. Port 25 nie jest publikowany na hoście. Dostęp do kontenerów przez mechanizmy Dockera nadal istnieje, dlatego nie należy traktować kontenerów jako granicy bezpieczeństwa dla niezaufanego kodu.
- **Wolumeny** — osobne wolumeny przechowują kolejki obu instancji Postfixa (`queue-a`, `queue-b`) oraz skrzynkę odbiorcy (`mailbox-b`). Dane są zachowywane pomiędzy zwykłymi restartami kontenerów.
- **Obrazy** — Debian 12 (`bookworm-slim`) z pakietem Postfix dostarczanym przez dystrybucję, zamiast gotowego obrazu serwera pocztowego typu all-in-one.

## Struktura projektu

```text
postfix-smtp-lab/
├── README.md
├── README.pl.md
├── LICENSE
└── labs/
    └── 01-basic-smtp/
        ├── README.md
        ├── README.pl.md
        ├── compose.yaml
        ├── Dockerfile
        └── config/
            ├── mail-a/main.cf
            └── mail-b/main.cf
```

## Wymagania i uruchomienie

Potrzebujesz Docker Engine z wtyczką Docker Compose oraz dostępu do Internetu, aby **podczas budowania obrazów** pobrać obraz bazowy i wymagane pakiety.

Wewnętrzna sieć Dockera dotyczy działających kontenerów, a nie procesu budowania obrazów.

Z katalogu głównego repozytorium:

```bash
cd labs/01-basic-smtp

docker compose config
docker compose up -d --build
docker compose ps
```

Sprawdź efektywną konfigurację Postfixa:

```bash
docker compose exec mail-a postconf myhostname mydestination relayhost
docker compose exec mail-b postconf myhostname mydestination default_transport
```

Jeżeli uruchomienie zakończy się błędem związanym z `maillog_file`, sprawdź wersję Postfixa i logi kontenerów:

```bash
docker compose exec mail-a postconf mail_version
docker compose logs mail-a mail-b
```

Laboratorium korzysta z logowania na standardowe wyjście przy uruchamianiu Postfixa przez `postfix start-fg`. Funkcjonalność ta może nie być dostępna w starszych wersjach Postfixa.

## Eksperyment A: poprawne dostarczenie wiadomości

Wyślij wiadomość z `mail-a` za pomocą lokalnego interfejsu `sendmail` Postfixa:

```bash
printf 'From: tester@a.lab.test\nTo: student@b.lab.test\nSubject: Proba SMTP\n\nCzesc z mail-a.\n' \
  | docker compose exec -T mail-a sendmail -i \
      -f tester@a.lab.test student@b.lab.test
```

Sprawdź kolejki obu serwerów i logi:

```bash
docker compose exec mail-a postqueue -p
docker compose exec mail-b postqueue -p
docker compose logs --tail=80 mail-a mail-b
```

Odczytaj lokalną skrzynkę odbiorcy:

```bash
docker compose exec mail-b \
  sh -c 'ls -l /var/mail/student; tail -n 40 /var/mail/student'
```

### Co powinno się wydarzyć?

Serwer `mail-a` przekazuje wiadomość do `mail-b` przez SMTP. Serwer odbiorczy przyjmuje ją, a następnie dostarcza do `/var/mail/student`.

Podczas naszych testów w logu `mail-a` pojawił się wpis:

```text
E25FAA2C41: to=<student@b.lab.test>,
relay=mail-b[172.12.3.3]:25,
dsn=2.0.0, status=sent
(250 2.0.0 Ok: queued as ECEC88ED85)
```

Następnie serwer nadawczy usunął wiadomość ze swojej kolejki:

```text
E25FAA2C41: removed
```

Na `mail-b`:

```text
ECEC88ED85: to=<student@b.lab.test>,
relay=local, dsn=2.0.0,
status=sent (delivered to mailbox)
```

Identyfikatory kolejki i adresy IP kontenerów będą różnić się pomiędzy uruchomieniami.

Zauważ, że każdy Postfix nadaje wiadomości własny **Queue ID**:

```text
mail-a: E25FAA2C41
mail-b: ECEC88ED85
```

Jednak oba serwery zachowują ten sam nagłówek wiadomości:

```text
Message-ID: <20261019135009.E25FAA2C41@mail-a.a.lab.test>
```

**Pytanie:** kiedy `mail-a` usuwa wiadomość ze swojej kolejki?

Poszukaj `status=sent` oraz pozytywnej odpowiedzi SMTP od `mail-b`.

Skuteczne przekazanie wiadomości do następnego serwera SMTP **nie musi oznaczać jej końcowego dostarczenia do skrzynki odbiorcy**.

W tym eksperymencie końcowe doręczenie potwierdza osobny wpis dotyczący transportu `local(8)` na `mail-b`.

## Eksperyment B: niedostępny serwer odbiorcy

Zatrzymaj serwer odbiorczy **przed** wysłaniem kolejnej wiadomości:

```bash
docker compose stop mail-b

printf 'From: tester@a.lab.test\nTo: student@b.lab.test\nSubject: Odbiorca offline\n\nTa wiadomosc ma poczekac.\n' \
  | docker compose exec -T mail-a sendmail -i \
      -f tester@a.lab.test student@b.lab.test
```

Sprawdź kolejkę i logi nadawcy:

```bash
docker compose exec mail-a postqueue -p
docker compose logs --tail=80 mail-a
```

### Oczekiwane zachowanie

Ponieważ `mail-b` jest niedostępny, `mail-a` nie może zakończyć dostarczania wiadomości przez SMTP.

Po nieudanej próbie Postfix pozostawia wiadomość w kolejce i odracza jej dostarczenie.

W zależności od stanu sieci Dockera awaria może zostać zgłoszona jako problem z rozwiązaniem nazwy DNS albo błąd połączenia.

Jeżeli początkowo kolejka pokazuje inny stan, odczekaj chwilę i ponownie sprawdź logi.

Przywróć działanie serwera odbiorczego:

```bash
docker compose start mail-b
```

Wymuś kolejną próbę dostarczenia:

```bash
docker compose exec mail-a postqueue -f
```

Sprawdź wynik:

```bash
docker compose logs --tail=80 mail-a mail-b

docker compose exec mail-b \
  sh -c 'tail -n 40 /var/mail/student'
```

`postqueue -f` zleca ponowną próbę dostarczenia wiadomości z kolejki. Nie zmienia konfiguracji Postfixa ani nie gwarantuje sukcesu, jeśli przyczyna awarii nadal występuje.

**Nie przeprowadzaj takich eksperymentów na produkcyjnym serwerze pocztowym.**

## Eksperyment C: nieistniejący odbiorca

Tym razem wyślij wiadomość na adres, który nie odpowiada żadnemu lokalnemu użytkownikowi na `mail-b`:

```bash
printf 'From: tester@a.lab.test\nTo: unknown@b.lab.test\nSubject: Unknown recipient\n\nTesting permanent rejection.\n' \
  | docker compose exec -T mail-a sendmail -i \
      -f tester@a.lab.test unknown@b.lab.test
```

Sprawdź logi i kolejkę:

```bash
docker compose logs --tail=80 mail-a mail-b
docker compose exec mail-a postqueue -p
```

Jeżeli domyślna weryfikacja lokalnych odbiorców jest aktywna, `mail-b` powinien odrzucić nieistniejącego odbiorcę trwałym błędem SMTP, zwykle `550 5.1.1`.

To pokazuje różnicę pomiędzy:

- **Błędem tymczasowym (`4xx`)** — dostarczenie może zostać ponowione.
- **Trwałą odmową (`5xx`)** — ponawianie tej samej próby bez usunięcia przyczyny zazwyczaj nie ma sensu.

Wiadomość jest przekazywana lokalnie do `mail-a`, dlatego początkowe polecenie `sendmail` może zakończyć się powodzeniem, mimo że późniejsze doręczenie przez SMTP zostanie odrzucone.

Postfix może wygenerować raport o niedostarczeniu wiadomości (*bounce*, DSN). Taki raport również może trafić do kolejki, dlatego sprawdź logi zamiast zakładać, że informacja dotarła już do nadawcy.

## Eksperyment D: jedna wiadomość, dwóch odbiorców i bounce

Dotychczas wysyłaliśmy wiadomości do jednego odbiorcy. Sprawdźmy teraz, co się wydarzy, gdy **jedna wiadomość będzie miała dwóch odbiorców**, a jeden z nich nie istnieje.

Przy okazji zobaczymy, jak Postfix generuje raport o niedostarczeniu (*bounce*) i co się dzieje, gdy również tego raportu nie można dostarczyć.

Pierwszą część eksperymentu można powtórzyć na aktualnej konfiguracji laboratorium. Dalsza część dokumentuje problem z NIS, który wystąpił podczas jego przygotowywania.

### Krok 1. Wyślij wiadomość do dwóch odbiorców

Upewnij się, że `mail-b` działa:

```bash
docker compose start mail-b
```

Wyślij jedną wiadomość do dwóch odbiorców:

```bash
printf 'From: tester@a.lab.test\nTo: student@b.lab.test, unknown@b.lab.test\nSubject: One message two recipients\n\nTesting multiple recipients.\n' \
  | docker compose exec -T mail-a sendmail -i \
      -f tester@a.lab.test \
      student@b.lab.test unknown@b.lab.test
```

Sprawdź logi:

```bash
docker compose logs --tail=80 mail-a mail-b
```

### Krok 2. Jeden Queue ID, dwa wyniki dostarczenia

W naszym eksperymencie `mail-a` przyjął wiadomość z dwoma odbiorcami:

```text
78A70A2C4A: from=<tester@a.lab.test>, size=354, nrcpt=2 (queue active)
```

Następnie otrzymał dwa różne wyniki:

```text
78A70A2C4A: to=<unknown@b.lab.test>,
dsn=5.1.1, status=bounced

78A70A2C4A: to=<student@b.lab.test>,
dsn=2.0.0, status=sent
```

Pierwszy odbiorca nie istnieje. `mail-b` odrzucił go odpowiedzią:

```text
550 5.1.1 Recipient address rejected:
User unknown in local recipient table
```

Odmowa nastąpiła podczas polecenia SMTP `RCPT TO`, przed przekazaniem treści wiadomości.

Drugi odbiorca został zaakceptowany, a `mail-b` nadał otrzymanej wiadomości własny Queue ID:

```text
824A28ED83: from=<tester@a.lab.test>,
size=535, nrcpt=1 (queue active)
```

Następnie dostarczył wiadomość lokalnie:

```text
824A28ED83: to=<student@b.lab.test>,
relay=local, dsn=2.0.0,
status=sent (delivered to mailbox)
```

**Wniosek:** Queue ID na `mail-a` identyfikuje wiadomość z dwoma odbiorcami, ale Postfix śledzi wynik doręczenia osobno dla każdego odbiorcy.

### Krok 3. Postfix generuje raport o niedostarczeniu

Ponieważ jeden z odbiorców został trwale odrzucony, `mail-a` wygenerował raport DSN:

```text
78A70A2C4A: sender non-delivery notification: 84D34A2C52
```

Powstała nowa wiadomość z własnym Queue ID:

```text
84D34A2C52: from=<>, size=2461, nrcpt=1 (queue active)
```

Zwróć uwagę na `from=<>`.

**Raport DSN ma pustego nadawcę kopertowego.** Dzięki temu nieudane dostarczenie raportu nie spowoduje generowania kolejnych raportów o błędach, które mogłyby utworzyć nieskończoną pętlę.

### Krok 4. Raport DSN również może trafić do `deferred`

Podczas przygotowywania laboratorium na `mail-a` mieliśmy domyślne ustawienie:

```ini
alias_maps = hash:/etc/aliases, nis:mail.aliases
```

Kontener nie korzystał jednak z NIS.

Przy próbie lokalnego doręczenia raportu DSN Postfix zgłosił:

```text
84D34A2C52: to=<tester@a.lab.test>,
relay=local, dsn=4.3.0,
status=deferred (alias database unavailable)
```

Wiadomość pierwotna została już obsłużona i usunięta z kolejki. W `deferred` pozostała **nowa wiadomość — raport DSN**.

To ważne rozróżnienie podczas analizowania `mailq`.

### Krok 5. Usunięcie błędu konfiguracji i ponowienie dostarczenia

W `config/mail-a/main.cf` ograniczyliśmy mapowanie aliasów do lokalnego pliku:

```ini
alias_maps = hash:/etc/aliases
```

Następnie przebudowaliśmy kontener:

```bash
docker compose up -d --build --no-deps mail-a
```

Sprawdziliśmy konfigurację:

```bash
docker compose exec mail-a postconf alias_maps
```

Wynik:

```text
alias_maps = hash:/etc/aliases
```

Kolejka została zachowana, ponieważ `/var/spool/postfix` znajduje się na nazwanym wolumenie Dockera.

Po uruchomieniu Postfixa i ponownej próbie dostarczenia w logach pojawiło się:

```bash
docker compose exec mail-a postqueue -f
docker compose logs --tail=40 mail-a
```

otrzymaliśmy:

```text
84D34A2C52: to=<tester@a.lab.test>,
relay=local, dsn=5.1.1,
status=bounced (unknown user: "tester")

84D34A2C52: removed
```

Tym razem problem z bazą aliasów został rozwiązany, ale ujawniła się inna przyczyna: na `mail-a` nie istnieje lokalny użytkownik `tester`.

Ponieważ niedostarczona wiadomość sama była raportem DSN z pustym nadawcą kopertowym, Postfix nie wygenerował kolejnego bounce'a.

### Wnioski

Z jednego eksperymentu wynikają cztery ważne obserwacje:

1. Jedna wiadomość w kolejce może mieć wielu odbiorców.
2. Każdy odbiorca może mieć inny wynik dostarczenia.
3. Raport o niedostarczeniu jest osobną wiadomością z własnym Queue ID.
4. Raport DSN również może trafić do `deferred`, a jego niedostarczenie nie powoduje generowania kolejnego raportu DSN.

**Ważne:** ten eksperyment pokazuje częściowy sukces i trwałe odrzucenie wiadomości. Nie pokazuje jeszcze sytuacji, w której jeden odbiorca otrzymał wiadomość, a drugi nadal oczekuje w `deferred`. To wymaga osobnego testu z kontrolowanym błędem tymczasowym.

W aktualnej wersji laboratorium `mail-a` ma już poprawione `alias_maps`, dlatego nowy użytkownik nie odtworzy historycznego błędu NIS bez świadomego przywrócenia starej konfiguracji. Nie jest to konieczne do wykonania podstawowego eksperymentu.

## Zaobserwowane błędy dostarczania

Podczas przygotowywania laboratorium celowo wywołaliśmy kilka rodzajów awarii.

| Scenariusz | Zaobserwowany DSN | Stan dostarczenia |
|---|---|---|
| Błąd rozwiązania nazwy DNS `mail-b` | `4.4.3` | `deferred` |
| Tymczasowy błąd odbiorcy (`451`) | `4.3.0` | `deferred` |
| Odmowa połączenia TCP/25 | `4.4.1` | `deferred` |
| Przywrócenie odbiorcy i przyjęcie wiadomości | `2.0.0` | `sent` |

Są to wyniki osobnych eksperymentów. Nie każdy z tych błędów można odtworzyć przez samo zatrzymanie kontenera odbiorczego.

### Jak interpretować te wyniki?

- `4.4.3` — problem z systemem katalogowym lub DNS.
- `4.3.0` — tymczasowy problem systemu pocztowego.
- `4.4.1` — problem z nawiązaniem połączenia sieciowego.
- `2.0.0` — pomyślne zakończenie operacji.

**Najważniejszy wniosek: `deferred` jest stanem dostarczenia, a nie diagnozą.**

Różne awarie mogą powodować ten sam stan `deferred`.

Aby ustalić przyczynę, trzeba przeanalizować komunikat diagnostyczny oraz kod DSN zapisany przez Postfixa.

Pamiętaj również, że kod DSN w logu Postfixa nie zawsze pochodzi bezpośrednio od zdalnego serwera SMTP.

Postfix może sam przypisać odpowiedni kod diagnostyczny, np. wtedy, gdy nie udało się w ogóle nawiązać połączenia TCP.

## Najważniejsze pojęcia

| Element | Rola |
|---|---|
| `smtpd` | Przyjmuje przychodzące połączenia SMTP. |
| `pickup` | Odbiera wiadomości przekazane lokalnie. |
| `cleanup` | Przetwarza wiadomości przed umieszczeniem ich w kolejce. |
| `qmgr` | Zarządza kolejką i planowaniem prób dostarczenia. |
| `smtp(8)` | Klient SMTP odpowiedzialny za przekazywanie wiadomości do kolejnego serwera. |
| `local(8)` | Dostarcza wiadomości do lokalnych skrzynek pocztowych. |
| `relayhost` | Określa jawnie skonfigurowany następny serwer SMTP. |
| `deferred` | Oznacza odroczone dostarczenie wiadomości. |
| Queue ID | Identyfikator wiadomości w kolejce danej instancji Postfixa. |
| Message-ID | Nagłówek służący identyfikacji wiadomości pomiędzy systemami; nie musi być unikalny ani wiarygodny. |

### Kiedy następuje przekazanie odpowiedzialności?

Po poprawnym przyjęciu wiadomości i wysłaniu końcowej odpowiedzi `250` po `DATA`, serwer odbiorczy przejmuje odpowiedzialność za jej dalszą obsługę.

Serwer nadawczy może wtedy usunąć własną kopię z kolejki.

Jeżeli serwer odbiorczy przyjmie wiadomość, ale połączenie zostanie zerwane, zanim nadawca otrzyma potwierdzenie, nadawca może ponowić próbę.

To jeden z powodów, dla których możliwe jest otrzymanie zduplikowanych wiadomości e-mail.

## Diagnostyka

Sprawdź kolejki:

```bash
docker compose exec mail-a postqueue -p
docker compose exec mail-b postqueue -p
```

Przejrzyj ostatnie logi:

```bash
docker compose logs --tail=100 mail-a mail-b
```

Sprawdź efektywną konfigurację:

```bash
docker compose exec mail-a postconf -n
docker compose exec mail-b postconf -n
```

Jeżeli znasz Queue ID wiadomości znajdującej się w kolejce, możesz sprawdzić jej zawartość:

```bash
docker compose exec mail-a postcat -q QUEUE_ID
```

Zastąp `QUEUE_ID` rzeczywistym identyfikatorem z kolejki danego serwera.

Podczas diagnozowania problemów z dostarczaniem warto odpowiedzieć na kilka pytań:

1. Czy Postfix przyjął wiadomość?
2. Czy wiadomość została umieszczona w kolejce?
3. Czy klient SMTP podjął próbę doręczenia?
4. Czy serwer odbiorczy odpowiedział, czy połączenie w ogóle się nie powiodło?
5. Czy dostarczenie zostało odroczone, trwale odrzucone, czy zakończyło się sukcesem?
6. Czy następny serwer tylko przyjął wiadomość, czy została ona dostarczona do końcowej skrzynki?

## Ograniczenia i kolejne lekcje

W tym laboratorium świadomie pomijamy kilka elementów typowego produkcyjnego serwera pocztowego:

- Brak routingu na podstawie zewnętrznych rekordów DNS/MX. Następny serwer jest jawnie skonfigurowany jako `[mail-b]:25`, chociaż wewnętrzny DNS Dockera rozwiązuje nazwę kontenera.
- Brak Dovecota i LMTP.
- Brak TLS oraz uwierzytelniania SMTP.
- Brak filtrowania antyspamowego i publicznie dostępnego SMTP.
- Brak trybu `network_mode: host` i publikowanych portów SMTP.

Nie wystawiaj tych kontenerów bezpośrednio do Internetu.

W kolejnych lekcjach można zbadać różnicę między błędami tymczasowymi i trwałymi, harmonogram ponawiania dostarczania (*retry/backoff*), błędy lokalnego doręczania, Dovecota i LMTP, DNS/MX oraz TLS/SASL.

To propozycje przyszłych eksperymentów, a nie wymagania do uruchomienia tego laboratorium.

## Dodatkowe materiały

- [Kolejka Postfixa, błędy SMTP i ponawianie dostarczania — Logos](https://logos.net.pl/pl/blog/postfix-smtp-kolejka-bledy-dostarczania/)
- [Kody błędów SMTP — jak czytać i diagnozować — Logos](https://logos.net.pl/pl/blog/kody-bledow-smtp-jak-czytac-i-diagnozowac/)
- [Dokumentacja Postfixa](https://www.postfix.org/documentation.html)
- [RFC 5321 — Simple Mail Transfer Protocol](https://www.rfc-editor.org/rfc/rfc5321.html)
- [RFC 3463 — Enhanced Mail System Status Codes](https://www.rfc-editor.org/rfc/rfc3463.html)

## Zatrzymanie i sprzątanie

Zatrzymaj kontenery i usuń sieć laboratorium:

```bash
docker compose down
```

Nazwane wolumeny zostaną zachowane.

Aby usunąć kontenery, sieci oraz nazwane wolumeny:

```bash
docker compose down -v
```

**Uwaga:** `down -v` usuwa również wiadomości i kolejki zapisane w wolumenach laboratorium.

Nie używaj tego polecenia, jeśli chcesz zachować wyniki eksperymentów.

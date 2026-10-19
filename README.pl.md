[English](README.md) | [Polski](README.pl.md)

# Postfix SMTP Lab

Zbiór praktycznych laboratoriów do nauki działania Postfixa i protokołu SMTP.

Celem projektu jest zrozumienie dostarczania poczty, odpowiedzi SMTP, kolejek wiadomości, ponawiania prób dostarczenia i obsługi błędów poprzez powtarzalne eksperymenty, a nie tylko analizę przykładowych konfiguracji.

Każde laboratorium działa w odizolowanym środowisku Docker Compose i koncentruje się na określonym zagadnieniu związanym z infrastrukturą pocztową.

## Dostępne laboratoria

### [01 — Podstawy SMTP](labs/01-basic-smtp/README.pl.md)

Dwa serwery Postfix komunikujące się przez SMTP.

W laboratorium nauczysz się:

- Wysyłać wiadomości pomiędzy dwoma serwerami Postfix.
- Rozumieć rolę procesów `smtpd`, `smtp`, `qmgr` i `local`.
- Sprawdzać kolejki pocztowe i analizować logi dostarczania wiadomości.
- Symulować niedostępność serwera odbiorczego.
- Obserwować odroczone dostarczanie wiadomości (`deferred`) i mechanizm ponawiania prób.
- Rozróżniać błędy tymczasowe i trwałe.
- Rozumieć, kiedy odpowiedzialność za wiadomość przechodzi pomiędzy serwerami SMTP.

## Szybki start

Wymagania:

- Docker Engine
- Wtyczka Docker Compose

Sklonuj repozytorium i uruchom pierwsze laboratorium:

```bash
git clone https://github.com/shiver25/postfix-smtp-lab
cd postfix-smtp-lab/labs/01-basic-smtp

docker compose up -d --build
docker compose ps
```

Następnie wykonaj eksperymenty opisane w [polskiej instrukcji laboratorium](labs/01-basic-smtp/README.pl.md).

Zatrzymaj laboratorium:

```bash
docker compose down
```

Domyślnie nazwane wolumeny pozostają zachowane. Polecenie `docker compose down -v` usuwa również kolejki pocztowe i zapisane wiadomości.

## Założenia projektu

- **Nauka przez eksperymenty.** Każde laboratorium pokazuje zachowania Postfixa i SMTP, które można samodzielnie zaobserwować.
- **Powtarzalność.** Laboratoria wykorzystują Docker Compose oraz jawne pliki konfiguracyjne.
- **Izolacja.** Porty SMTP nie są publikowane na hoście, a pierwsze laboratorium działa w wewnętrznej sieci Dockera.
- **Wyjaśnianie błędów.** Kody odpowiedzi SMTP, rozszerzone kody statusu, logi i stany kolejek są istotną częścią nauki.
- **Jedno zagadnienie na raz.** Kolejne elementy infrastruktury wprowadzamy dopiero wtedy, gdy są potrzebne.

## Bezpieczeństwo

Laboratoria są przeznaczone do lokalnej nauki i testowania.

Nie są gotowymi do wdrożenia produkcyjnymi serwerami pocztowymi. W szczególności pierwsze laboratorium nie obejmuje TLS, uwierzytelniania SMTP, filtrowania antyspamowego ani kompletnego systemu obsługi poczty.

Nie udostępniaj kontenerów laboratorium bezpośrednio w Internecie.

## Licencja

Projekt jest udostępniany na licencji MIT.

Szczegóły znajdują się w pliku [LICENSE](LICENSE).

## Dodatkowe materiały

- [Dlaczego e-mail nie dociera? Kolejka Postfixa, błędy SMTP i ponawianie dostarczania](https://logos.net.pl/pl/blog/postfix-smtp-kolejka-bledy-dostarczania/)
- [Kody błędów SMTP — jak je czytać i diagnozować?](https://logos.net.pl/pl/blog/kody-bledow-smtp-jak-czytac-i-diagnozowac/)

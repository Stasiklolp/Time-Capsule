# Witaj w środowisku dev `sak` — przewodnik dla Stanisława

Ten dokument tłumaczy, jak pracować na Twoim prywatnym, zdalnym środowisku deweloperskim CMSO Odoo. Jest też kontekstem dla Claude Code, którego u siebie uruchamiasz.

> **Dla Claude'a:** ten plik + `CLAUDE.md` w korzeniu repo są **wymaganym kontekstem** projektu. `CLAUDE.md` zawiera ogólne zasady (polskie diakrytyki, deploy, sync); ten plik — specyfikę Twojej instancji `sak`. Sekcja „Specjalna sekcja dla Claude Code" na dole zawiera twarde reguły działania.

---

## TL;DR (pierwsze 30 sekund)

| Co | Gdzie |
|---|---|
| Twoje URL | https://sak.odoo.cmso.bet |
| Baza Odoo | `odoo_sak` (świeża kopia produkcji) |
| Logowanie do Odoo | Twoim **kontem produkcyjnym** CMSO (hasła nie są resetowane) |
| Logowanie SSH | `ssh s.sak@odoo.cmso.bet` |
| Kod | `/opt/odoo/sak/cmso-odoo/` (Twój checkout, gałąź `sak`) |
| Komendy | `dev-restart`, `dev-upgrade`, `dev-logs`, `dev-db-refresh` |
| Repo | git@github.com:lukaszolek/cmso-odoo.git |

Po zalogowaniu po SSH automatycznie jesteś w `/opt/odoo/sak/cmso-odoo`. Edytuj kod, zapisuj, odśwież przeglądarkę.

---

## Pierwsze logowanie (jednorazowo)

```bash
ssh s.sak@odoo.cmso.bet

# 1. Skonfiguruj swoją tożsamość dla git (potrzebne do commitów)
cd /opt/odoo/sak/cmso-odoo
git config user.name "Stanisław Sak"
git config user.email "<twój_email_GitHub>"

# 2. Sprawdź, że klucz SSH do GitHuba działa
ssh -T git@github.com
# Oczekiwana odpowiedź: "Hi <twój_login>! You've successfully authenticated..."

# 3. Sprawdź gałąź i pobierz aktualną wersję
git branch --show-current      # powinno być: sak
git pull

# 4. Sprawdź, że Twoja instancja Odoo żyje
curl -I https://sak.odoo.cmso.bet/web/login    # powinno być: 200
```

Otwórz w przeglądarce: **https://sak.odoo.cmso.bet** — załaduje się Odoo z (kopią) danych produkcji. Zaloguj się **swoim normalnym kontem produkcyjnym CMSO**.

---

## Architektura — jak to działa pod spodem

Na jednym serwerze (`odoo.cmso.bet`) działają trzy instancje Odoo obok siebie, każda we własnym kontenerze Dockera, z osobną bazą w jednym wspólnym kontenerze PostgreSQL:

| Środowisko | Domena | Port | Kontener | Baza | Gałąź |
|---|---|---|---|---|---|
| Produkcja | odoo.cmso.bet | 8069 | `odoo_production` | `odoo` | `main` |
| Staging | staging.odoo.cmso.bet | 8070 | `odoo_staging` | `odoo_staging` | `main` |
| **Twoje (sak)** | **sak.odoo.cmso.bet** | **8071** | **`odoo_sak`** | **`odoo_sak`** | **`sak`** |

Twój kontener `odoo_sak` ma podpięty bind-mount Twojego katalogu z kodem (`/opt/odoo/sak/cmso-odoo/` → `/mnt/extra-addons/...` w kontenerze). Uruchamiany jest z `--dev=all` i `workers=0`, więc **zmiany w plikach Pythona są wykrywane automatycznie** i Odoo restartuje proces.

### Granice uprawnień

Twój login `s.sak` **nie ma grupy `docker`** ani sudo na nic poza 4 komendami `cmso-dev-* sak`. Tak jest celowo — produkcja siedzi na tym samym hoście, więc luźny dostęp do dockera = de facto root nad produkcją. Nie próbuj się tego obchodzić; gdyby było potrzebne więcej uprawnień, zgłoś do Łukasza.

### Struktura Twoich plików

```
/opt/odoo/sak/
├── cmso-odoo/          ← Twój checkout (gałąź sak). EDYTUJESZ TUTAJ.
│   ├── cmso_production/
│   ├── cmso_investor_portal/
│   ├── cmso_quality_mrp/
│   ├── cmso_iso/
│   ├── cmso/
│   ├── addons-community/ ← moduły OCA (gitignored, skopiowane z produkcji)
│   ├── oca_addons/
│   └── quality_control_oca/
├── data/               ← filestore Odoo (załączniki, sesje). Własność kontenera.
├── config/             ← root-owned, nie ruszasz
└── compose/            ← root-owned, nie ruszasz
```

---

## Codzienna praca

### Zmiany w Pythonie (logika)

**Po prostu zapisz plik.** `--dev=all` + `workers=0` powodują, że Odoo wykrywa zmianę i samo restartuje proces. Odśwież przeglądarkę.

Jeśli auto-reload nie zadziała (rzadko — niektóre `_register_hook`, `_setup_complete`, zmiany w `__init__.py` modułu):
```bash
dev-restart
```

### Zmiany w XML (widoki, akcje, menu) lub strukturze modelu (nowe pole, `@api.depends`, `@api.constrains`)

Wymagany jest upgrade modułu — Odoo musi przeładować definicje do bazy:
```bash
dev-upgrade
```

Trwa ~30–60 sekund. Pod spodem: zatrzymanie kontenera → jednorazowy `odoo -u cmso_production,cmso_investor_portal,cmso_quality_mrp,cmso_iso,cmso --stop-after-init` → restart głównego kontenera.

### Zmiany w CSS / JS / statyczne assety

Hard refresh w przeglądarce: **Cmd+Shift+R** (Mac) / **Ctrl+Shift+R** (Linux/Windows). Jeśli nadal stare assety — otwórz DevTools (F12), zakładka Network, zaznacz „Disable cache", F5.

### Skrót decyzyjny

| Co zmieniłeś | Co zrobić |
|---|---|
| Python — tylko ciało funkcji | nic (auto-reload), ewentualnie `dev-restart` |
| Python — nowe pole / `@api.depends` / `@api.constrains` | `dev-upgrade` |
| XML — view, akcja, menu, dane | `dev-upgrade` |
| `__manifest__.py` (zmiana `depends`, `data`) | `dev-upgrade` |
| CSS, JS, statyczne assety | hard refresh przeglądarki |
| `requirements.txt` (nowy pakiet pip) | poproś admina o rebuild obrazu `odoo_dev:latest` |

### Podgląd logów

```bash
dev-logs              # ostatnie 100 linii + live tail (Ctrl+C aby przerwać)
dev-logs 500          # ostatnie 500 linii + live tail
```

### Świeże dane z produkcji

Kiedy potrzebujesz aktualnej kopii produkcji (np. test na zamówieniach z dziś):
```bash
dev-db-refresh
```

Co się dzieje:
1. Zatrzymanie Twojego kontenera.
2. `pg_dump` produkcji (**read-only, produkcja nie cierpi — żadnego przestoju**).
3. Drop i odtworzenie `odoo_sak`, wgranie dumpa, kopia filestore.
4. Ustawienia dev: `web.base.url = https://sak.odoo.cmso.bet`, wyłączenie `ir_cron` i `ir_mail_server`, flaga `database.is_dev = True`.
5. Restart Twojego kontenera.

⚠️ **Destrukcyjne dla Twojej bazy** — wszystkie testowe rekordy i zmiany w `odoo_sak` przepadają. Backup poprzedniej wersji idzie do `/opt/odoo/backups/sak_backup_*.sql.gz`.

Cron i e-maile są wyłączane intencjonalnie — Twoja instancja dev nie wysyła maili do realnych klientów ani nie odpala zaplanowanych akcji.

**Hasła użytkowników NIE są resetowane** — logujesz się swoim normalnym kontem produkcyjnym.

---

## Git workflow

Twoja domyślna gałąź to `sak` (odbita od `main`). Stamtąd pracujesz.

### Codzienny cykl

```bash
git status                                  # co masz zmienione
git diff cmso_production/models/foo.py      # podgląd
git add cmso_production/models/foo.py
git commit -m "feat: krótki opis zmiany"
git push                                    # na origin/sak
```

### Synchronizacja z `main`

Co kilka dni (albo przed PR) zaciągnij zmiany z `main`, żeby uniknąć rozjazdu:
```bash
git fetch origin
git merge origin/main         # albo: git rebase origin/main
git push
```

### Wprowadzenie Twoich zmian do produkcji

Otwierasz Pull Request `sak → main` na GitHubie:
1. Upewnij się że `sak` jest zsynchronizowany z `main` (wyżej).
2. Idź na https://github.com/lukaszolek/cmso-odoo → „Compare & pull request" dla `sak`.
3. Opisz zmiany, link do issue (jeśli jest), poproś o review.
4. Po merge `main` — Łukasz deployuje na staging i produkcję (`./deploy.sh staging`, potem `./deploy.sh production`).

### Konwencje commitów (z CLAUDE.md)

- `feat:` — nowa funkcjonalność
- `fix:` — naprawa błędu
- `refactor:` — refaktoryzacja kodu
- `docs:` — dokumentacja
- `style:` — formatowanie, bez zmian logiki
- `test:` — testy

Opis po polsku, krótko i konkretnie — preferowane „co i dlaczego" zamiast samego „co".

---

## Testowanie zmian

Brak osobnego frameworka testów (jak na razie). Standardowy cykl:
1. Edytujesz kod.
2. Odpalasz scenariusz w przeglądarce na https://sak.odoo.cmso.bet.
3. Patrzysz w `dev-logs` jeśli coś sypie błędem / Traceback.
4. Po skończeniu — commit, push, PR.

Masz pod ręką pełen zestaw danych produkcyjnych (po `dev-db-refresh`), więc testujesz na realnych sytuacjach: prawdziwych zamówieniach, projektach, użytkownikach.

---

## Czego NIE robić

- **Nie próbuj `sudo` z innym argumentem niż `sak`** — `sudo cmso-dev-restart staging` itp. zostanie odrzucone (prośba o hasło). Twoje uprawnienia są twardo zawężone do Twojej instancji.
- **Nie edytuj `/opt/odoo/sak/config/` ani `/opt/odoo/sak/compose/`** — root-owned. Jeśli potrzebujesz zmiany konfiguracji (więcej `workers`, inny `log_level`, dodanie biblioteki pip) — napisz do Łukasza.
- **Nie próbuj się dostać do bazy produkcyjnej `odoo`** — to inna baza w tym samym Postgresie. Nie masz hasła (celowo). Twoja kopia w `odoo_sak` to to samo, ale bezpiecznie odizolowane.
- **Nie pushuj bezpośrednio do `main` ani `staging`** — zawsze przez PR z gałęzi `sak`. Te gałęzie deployowane są skryptem.
- **Nie commituj sekretów** — `.env`, klucze, hasła. Jeśli nie wiesz, czy coś commitować — zapytaj.
- **Nie zostawiaj `dev-logs` otwartego w terminalu na noc** — żre I/O. Ctrl+C kiedy skończysz.

---

## Co jeśli coś nie działa

| Symptom | Co zrobić |
|---|---|
| `https://sak.odoo.cmso.bet` daje 502/504 | `dev-logs` — kontener prawdopodobnie zsypał się; jeśli proces nie wraca, `dev-restart` |
| Po `dev-upgrade` Odoo wstaje z błędem | `dev-logs 200` → szukaj `ERROR` / `Traceback`; cofnij zmianę, popraw, ponów |
| `git push` woła o hasło / „Permission denied (publickey)" | klucz SSH GitHuba nie działa — zgłoś Łukaszowi |
| `dev-db-refresh` zawodzi w połowie | wyślij Łukaszowi cały output, prawdopodobnie problem z dostępem do filestore lub Postgresa |
| Dziwne rzeczy w przeglądarce po zmianach JS/CSS | hard refresh (Cmd+Shift+R), potem DevTools → Disable cache + F5 |
| Auto-reload Pythona nie zadziałał | `dev-restart` |
| Wszystko nie działa, nie rozumiem dlaczego | `dev-restart` + `dev-logs`; jeśli dalej źle — `dev-db-refresh` (czysta baza); jeśli dalej źle — Łukasz |

---

## Specjalna sekcja dla Claude Code

Jeśli czytasz to jako AI agent uruchomiony przez Stanisława w `/opt/odoo/sak/cmso-odoo`:

### Twój kontekst wykonawczy
- Działasz jako użytkownik `s.sak` (uid 1005).
- **Nie masz grupy `docker`.** Wszystko związane z kontenerem Odoo idzie przez `sudo cmso-dev-* sak` (lub aliasy `dev-*`).
- **Masz `sudo NOPASSWD` wyłącznie dla 4 wrapperów** z argumentem `sak`:
  - `sudo cmso-dev-restart sak` (alias: `dev-restart`)
  - `sudo cmso-dev-upgrade sak` (alias: `dev-upgrade`)
  - `sudo cmso-dev-logs sak [N]` (alias: `dev-logs`)
  - `sudo cmso-dev-db-refresh sak` (alias: `dev-db-refresh`)
- **Próby innych komend** (np. `sudo docker ps`, `sudo cmso-dev-restart staging`, `sudo cat /opt/odoo/sak/config/odoo.conf`) zostaną odrzucone z prośbą o hasło. **Nie próbuj** ich obchodzić — to nie jest błąd konfiguracji, tylko model bezpieczeństwa.

### Twoje twarde reguły działania

1. **Edytuj kod tylko w `/opt/odoo/sak/cmso-odoo/`.** Nigdzie indziej. Nie ruszaj `/opt/odoo/production/`, `/opt/odoo/staging/`, `/opt/odoo/sak/config/`, `/opt/odoo/sak/compose/`.
2. **Po edycji wykonaj odpowiednią akcję odświeżania automatycznie** (tabela „Skrót decyzyjny" wyżej). To jest **wymóg z CLAUDE.md** — nie zostawiaj zmian bez zastosowania.
3. **Polskie diakrytyki** w polskich tekstach UI są obowiązkowe: `ą`, `ć`, `ę`, `ł`, `ń`, `ó`, `ś`, `ź`, `ż` (i wielkie). Nigdy „PLYTY"→ poprawnie „PŁYTY", nigdy „Usun"→ „Usuń". To twarda zasada projektu (CLAUDE.md).
4. **Nigdy nie operuj nazwami `staging` ani `production`** w komendach docker/git/sudo, nawet jeśli pomysł brzmi sensownie. Twój sandbox to **wyłącznie** `sak`.
5. **Nie próbuj się dostać do innych baz Postgresa** — masz dostęp tylko do `odoo_sak` przez kontener Odoo (nie bezpośrednio).
6. **Nie commituj na `main` ani `staging`** — wyłącznie gałąź `sak`. Otwarcie PR do `main` to decyzja człowieka.
7. **Przed `dev-db-refresh` ZAPYTAJ użytkownika** — to nadpisuje całą bazę dev, zmiany testowe znikają. Nigdy nie odpalaj „dla porządku".
8. **Niezmienione obrazy Docker** — jeśli widzisz, że potrzeba nowej biblioteki pip (`ModuleNotFoundError`), nie kombinuj z `docker exec ... pip install` (i tak nie masz uprawnień). Zgłoś użytkownikowi — admin przebuduje obraz `odoo_dev:latest`.

### Konwencje projektu (wyciąg z CLAUDE.md — przeczytaj cały plik)

- Prefiks modeli: `cmso.*`
- Każdy model: `_description = '...'`
- Computed fields: `@api.depends(...)`
- Manifest — kolejność `data`: security → views → data; aktualizuj `version`
- Polskie napisy we wszystkich UI fieldach: `string=`, `help=`, `<field>`, komunikatach walidacji

### Główne moduły CMSO

- **`cmso`** — bazowy moduł CMSO
- **`cmso_production`** — główna logika produkcji (`mrp.production`, harmonogramowanie, projekty)
- **`cmso_quality_mrp`** — kontrola jakości w produkcji
- **`cmso_investor_portal`** — portal inwestorski
- **`cmso_iso`** — wymagania ISO
- **`quality_control_oca`**, **`addons-community/*`**, **`oca_addons/*`** — moduły zewnętrzne (gitignored, lokalnie skopiowane z produkcji)

### Główne pliki referencyjne

- `CLAUDE.md` — zasady projektu (deploy, sync, konwencje, środowiska)
- `docs/MODELS.md` — inwentarz modeli z opisami i relacjami (jeśli istnieje)
- `README.md` — ogólny opis projektu
- `__manifest__.py` w każdym module — co zawiera i od czego zależy

---

## Kontakt

- **Łukasz Olek** — `lukasz.olek@framky.com` / Slack
- **Repo + issues** — https://github.com/lukaszolek/cmso-odoo
- **Twoja instancja** — https://sak.odoo.cmso.bet
- **Dokumentacja projektu** — `CLAUDE.md`, `README.md`, `docs/`

Powodzenia! 🚀

# Ruff, uv i ty — Python Summit 2026

Materiały do wystąpienia „Ruff, uv i ty” (Python Summit, Warszawa, 20.10.2026) — komendy i konfiguracja ze slajdów, w formie do skopiowania.

- `slides/` — slajdy (PDF), dodane po wystąpieniu.
- Wersje, na których wszystko sprawdzałem (18–19.10.2026): **uv 0.13.1, Ruff 0.17.1, ty 0.0.89**, Python 3.15.0.

## Jutro rano: jedna komenda we własnym repo

```sh
uvx ruff check --statistics                 # domyślne reguły, bez instalowania czegokolwiek
uvx ruff check --select ALL --statistics    # wszystkie 954 reguły, dla ciekawych
```

`uvx` uruchamia narzędzie w osobnym, tymczasowym środowisku — nic nie dotyka Twojego projektu.

## Nowy projekt w trzech komendach

```sh
uv init demo && cd demo     # git, .python-version, pyproject.toml, układ src/
uv add httpx                # od razu robi lock i sync: pyproject.toml, uv.lock, .venv
uv run demo                 # brakujący Python pobiera się sam, środowiska nie aktywujesz
```

Skrypt w jednym pliku (PEP 723):

```sh
uv init --script skrypt.py
uv add --script skrypt.py httpx   # zależności w komentarzu na górze pliku
uv run skrypt.py                  # skrypt dostaje własne środowisko
```

Nowa osoba w zespole: `git clone … && uv sync` i może pracować.

## Demo z wystąpienia

```sh
uv init demo && cd demo && ls -a
uv python install 3.15 && uv python pin 3.15 && uv run python -V
uv run demo
uv add fastapi uvicorn            # na 3.15 bez fastapi[standard]: PyYAML nie ma jeszcze kół dla 3.15
uv run uvicorn main:app           # http://localhost:8000/docs
uvx pycowsay "Python Summit 2026" # drugi raz: start z cache w ok. 50 ms
```

```python
# main.py
from fastapi import FastAPI

app = FastAPI()


@app.get("/")
def hello():
    return {"hello": "Python Summit"}
```

Odnogi: `uv tree`, `uvx ty check`, `uvx --python 3.15 python`, `uv python install 3.15t` i `uv run --python 3.15t python -c "import sys; print(sys._is_gil_enabled())"`, `uvx --from rich python -m rich`. Bez sieci: `UV_OFFLINE=1`.

## pyproject.toml (slajd 14)

```toml
[project]
name = "demo"
version = "0.1.0"
requires-python = ">=3.12"
dependencies = ["httpx"]

[dependency-groups]
dev = ["ruff==0.17.1", "ty==0.0.89"]

[tool.uv]
# starszy uv odmówi pracy
required-version = ">=0.13"

[tool.ruff.lint]
# jawnie zamiast 407 domyślnych
select = ["E", "F", "I", "UP", "B"]

[tool.ty.rules]
# na start same ostrzeżenia
all = "warn"

[tool.ty.terminal]
# ostrzeżenia nie wywalają CI
error-on-warning = false
```

Przykład błędu, który łapie Ruff bez żadnego pluginu (reguła B015, flake8-bugbear, w domyślnym zestawie):

```python
def test_total():
    result = total([1, 2])
    result == 3  # B015: zapomniany assert — test zawsze przechodzi
```

## To samo sprawdzenie w trzech miejscach

**Edytor.** PyCharm 2025.3+ ma uv, Ruffa i ty wbudowane; VS Code: rozszerzenia Ruff i ty (oba działają jako language server).

**Commit.** Hook uruchamiany przez `uv run`, więc wersja Ruffa pochodzi z `uv.lock` (`.pre-commit-config.yaml`, instalacja: `uvx pre-commit install`):

```yaml
repos:
  - repo: local
    hooks:
      - id: ruff-check
        name: ruff check
        entry: uv run ruff check --fix
        language: system
        types_or: [python, pyi]
        require_serial: true
      - id: ruff-format
        name: ruff format
        entry: uv run ruff format
        language: system
        types_or: [python, pyi]
        require_serial: true
```

**CI** (GitHub Actions). `--locked` przerywa build, gdy `uv.lock` nie zgadza się z `pyproject.toml`. Akcję `setup-uv` przypinaj do pełnej wersji — od v8 nie ma ruchomych tagów `@v8`:

```yaml
name: ci
on: [push, pull_request]

jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - uses: astral-sh/setup-uv@v9.0.0
        with:
          enable-cache: true
      - run: uv sync --locked
      - run: uv run ruff check
      - run: uv run ruff format --check
      - run: uv run ty check
      - run: uv run pytest
```

Wszędzie to samo: `uv.lock` pilnuje wersji, `pyproject.toml` — reguł.

## Agenci AI (AGENTS.md / CLAUDE.md)

```md
Po każdej zmianie uruchom, każde osobno (nie przez &&):
uv run ruff check --fix
uv run ruff format
uv run ty check
uv run pytest
```

Osobno, bo `ruff check --fix` przy nienaprawialnych znaleziskach kończy się kodem 1 i w łańcuchu `&&` uciąłby resztę.

## Migracja w czterech krokach

Każdy krok daje zysk osobno i każdy da się cofnąć.

1. **CI bez zmian w kodzie:** `uv pip install -r requirements.txt` zamiast `pip install`.
2. **Lock:** zależności do `[project]`, `uv lock`, w CI `uv sync --locked`. Z Poetry, Pipenv, pip-tools: `uvx migrate-to-uv` (status Alpha — wynik przejrzyj).
3. **Ruff:** `select = ["E", "F"]`, reguły dokładaj co sprint; formatowanie całego repo jednym commitem (`uv run ruff format`, hash do `.git-blame-ignore-revs`). Linia bazowa dla starych znalezisk: `ruff check --add-noqa` lub `--add-ignore` — patrz pułapki.
4. **ty:** najpierw jako ostrzeżenia (`all = "warn"`) obok mypy; przełącz, gdy mypy daje więcej szumu niż pożytku.

## Ryzyko i plan wyjścia

```toml
[tool.uv]
exclude-newer = "7 days"   # tygodniowa karencja na nowe wydania
```

```sh
uv export --format pylock.toml      # PEP 751 — wyjście jedną komendą
uv export --format requirements.txt
uv audit                            # znane podatności w zależnościach z locka (preview)
```

uv mówi standardami: PEP 621 (metadane), 723 (skrypty), 735 (grupy zależności), 751 (pylock.toml). Nie musisz ufać — wystarczy mieć wyjście.

## Pułapki

- `uv sync` usuwa z `.venv` wszystko, czego nie ma w `uv.lock`. `uv pip …` nie czyta `uv.lock` — to osobny tryb zgodny z pip.
- Aktualizacja Ruffa do 0.16+ bez jawnego `select` włącza ok. 400 reguł zamiast 59 — czerwone CI. Przypnij wersję i wpisz `select`.
- `ruff check --fix --unsafe-fixes` potrafi zmienić działanie kodu. Bez tej flagi Ruff robi tylko bezpieczne poprawki.
- `ruff check --add-ignore .` (tak samo `--add-noqa .`) dopisuje komentarze do **każdego** pliku, który znajdzie, także poza repo (tam nie ma `.gitignore`). Tylko w repo, przy czystym `git status`, potem `git diff`.
- ty 0.0.x zmienia diagnostyki między wersjami — przypinaj dokładną wersję. Nie ma systemu pluginów (Django: dziś Pyrefly).
- Plik `uv.toml` obok `pyproject.toml` sprawia, że sekcja `[tool.uv]` jest ignorowana; `.python-version` potrafi kłócić się z `requires-python`.

## Bonus: monorepo

```toml
# pyproject.toml w korzeniu repo
[tool.uv.workspace]
members = ["packages/*", "services/*"]

[tool.uv.sources]
core = { workspace = true }
```

```sh
uv sync --all-packages
uv run --package api pytest
```

Jeden `uv.lock` na całe repo, pakiety korzystają z siebie bez publikowania; każdy pakiet może nadpisać reguły Ruffa we własnym `pyproject.toml`.

## Pomiary ze slajdu 7

Linux, 2 rdzenie, 18.10.2026. Instalacja 80 pakietów z rozgrzanym cache, bez kompilacji `.pyc`: pip 26.2.1 — 9,7 s, uv 0.13.1 — 0,33 s. Lint całego repo Django (529 tys. linii): flake8 7.4.1 — 33,5 s, Ruff 0.17.1 z 407 domyślnymi regułami — 0,9 s. Typy: benchmark Astrala ([BENCHMARKS.md](https://github.com/astral-sh/ty/blob/main/BENCHMARKS.md)), Home Assistant: ty 1,81 s, mypy 24,60 s.

```sh
time pip install --no-compile -r requirements.txt   # świeże środowisko za każdym razem
time uv pip install -r requirements.txt
time flake8 .
time ruff check --no-cache .
```

## Źródła

- [uv](https://docs.astral.sh/uv/) · [`exclude-newer`](https://docs.astral.sh/uv/reference/settings/#exclude-newer) · [workspaces](https://docs.astral.sh/uv/concepts/projects/workspaces/) · [uv audit](https://astral.sh/blog/uv-audit)
- [Ruff](https://docs.astral.sh/ruff/) · [Ruff 0.16](https://astral.sh/blog/ruff-v0.16.0) · [reguła B015](https://docs.astral.sh/ruff/rules/useless-comparison/) · [tutorial, `--add-noqa`](https://docs.astral.sh/ruff/tutorial/)
- [ty](https://github.com/astral-sh/ty) · [Pyrefly 1.0](https://pyrefly.org/blog/v1.0/)
- [Astral w OpenAI](https://astral.sh/blog/openai) · [ogłoszenie OpenAI](https://openai.com/index/openai-to-acquire-astral/) · [Talk Python #552](https://talkpython.fm/episodes/show/552/astral-joins-openai) · [Charlie Marsh w Test Set (luty 2026)](https://wesmckinney.com/transcripts/2026-02-24-test-set-charlie-marsh)
- [State of Django 2026](https://blog.jetbrains.com/pycharm/2026/08/the-state-of-django-2026-boring-is-so-back/) · [ceny runnerów GitHub Actions](https://www.tenki.cloud/blog/github-actions-runner-pricing-2026)

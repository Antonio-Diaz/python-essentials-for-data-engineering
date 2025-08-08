# Guía práctica: Tipado y buenas prácticas en Python

*typing + mypy · estructuras de control · comprensiones/generadores · manejo de excepciones · logging coherente*

---

## 1) ¿Por qué te importa?

* **Tipos**: documentan intención, evitan bugs tontos y facilitan el refactor.
* **mypy**: valida estáticamente tus anotaciones sin ejecutar el código.
* **Buenas prácticas**: comprensiones, generadores y `try/except` bien usados → código más claro y rápido de mantener.
* **Logging**: si no queda en logs, no existió. Un formato consistente acelera el debug.

---

## 2) Setup rápido (opcional con Poetry)

```bash
# En tu proyecto
poetry add --group dev mypy
```

En `pyproject.toml` (o `mypy.ini`) agrega una base sensata:

```toml
[tool.mypy]
python_version = "3.12"
disallow_untyped_defs = true
warn_return_any = true
warn_unused_ignores = true
no_implicit_optional = true
strict_optional = true
pretty = true
```

Ejecuta:

```bash
poetry run mypy .
```

---

## 3) Tipado estático con `typing` + `mypy`

### Lo esencial (Python 3.12+)

```python
from dataclasses import dataclass
from typing import Optional, Iterable

def percent(x: float, total: float) -> float:
    if total == 0:
        raise ValueError("total no puede ser 0")
    return (x / total) * 100

@dataclass(frozen=True)
class Item:
    sku: str
    price: float
    qty: int = 1

def total(items: Iterable[Item]) -> float:
    return sum(i.price * i.qty for i in items)

def find_sku(items: Iterable[Item], sku: str) -> Optional[Item]:
    return next((i for i in items if i.sku == sku), None)
```

**Pistas**

* Usa tipos de la stdlib: `list[str]`, `dict[str, float]`, `tuple[int, ...]`.
* `Optional[T]` ⇔ `T | None`.
* Prefiere `dataclasses` para objetos inmutables y legibles.
* Si una lib no tiene tipos, busca *stubs* `types-<paquete>` o tipa la interfaz mínima que uses.

---

## 4) Estructuras de control y comprensiones/generadores

```python
nums = [1, 2, 3, 4, 5, 6]

# Comprensión (lista nueva)
evens_sq: list[int] = [n*n for n in nums if n % 2 == 0]

# Generador (perezoso, sin crear lista en memoria)
evens_sq_gen = (n*n for n in nums if n % 2 == 0)

# Patrones útiles
from math import prod
has_big = any(n > 100 for n in nums)
all_positive = all(n > 0 for n in nums)
product = prod(n for n in nums if n % 2 == 1)
```

**Consejo**: usa **lista** si vas a reusar/almacenar; usa **generador** si vas a iterar una sola vez o los datos son grandes.

---

## 5) Manejo de excepciones limpio

```python
class InvalidRowError(Exception):
    """Fila inválida en el dataset."""

def parse_int(s: str) -> int:
    try:
        return int(s)
    except ValueError as e:
        raise InvalidRowError(f"No es entero: {s!r}") from e
```

* Evita `except:` “a pelo”. Captura la excepción específica.
* Usa `raise ... from e` para no perder el *traceback* original.
* `try/except/else/finally`: `else` es para la ruta feliz *si no hubo excepción*.

---

## 6) Logging coherente para depurar

```python
import logging
from pathlib import Path

LOG_DIR = Path("logs")
LOG_DIR.mkdir(exist_ok=True)

logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s | %(levelname)s | %(name)s | %(message)s",
    handlers=[logging.FileHandler(LOG_DIR / "app.log"), logging.StreamHandler()],
)

logger = logging.getLogger("app")

def load_file(path: Path) -> str:
    logger.info("Leyendo archivo: %s", path)
    try:
        return path.read_text(encoding="utf-8")
    except FileNotFoundError:
        logger.exception("Archivo no encontrado")  # log con traceback
        raise
```

**Reglas simples**

* Un **formato único** en todo el repo.
* Usa `logger.info` para eventos normales, `warning` para situaciones raras, `error/exception` para fallos.
* No silencies excepciones sin log.

---

# Casos de uso fáciles (paso a paso)

## Caso 1: “Limpia y tipa un CSV”

**Objetivo**: leer un CSV de ventas, validar filas, calcular totales y registrar errores.

1. Crea `ventas.csv`:

```csv
sku,price,qty
A-01,9.5,3
A-02,not_a_number,2
A-03,12.0,1
```

2. Código (`clean_csv.py`):

```python
from __future__ import annotations
from dataclasses import dataclass
from typing import Iterable
import csv, logging
from pathlib import Path

logging.basicConfig(level=logging.INFO, format="%(levelname)s | %(message)s")
log = logging.getLogger("clean_csv")

@dataclass(frozen=True)
class Sale:
    sku: str
    price: float
    qty: int

class BadRow(Exception): ...

def parse_row(row: dict[str, str]) -> Sale:
    try:
        return Sale(
            sku=row["sku"],
            price=float(row["price"]),
            qty=int(row["qty"]),
        )
    except Exception as e:
        raise BadRow(f"Row inválida: {row}") from e

def load_sales(path: Path) -> list[Sale]:
    good: list[Sale] = []
    with path.open() as f:
        for i, row in enumerate(csv.DictReader(f), start=2):
            try:
                good.append(parse_row(row))
            except BadRow:
                log.warning("Fila %d descartada", i, exc_info=True)
    return good

def total_amount(sales: Iterable[Sale]) -> float:
    return sum(s.price * s.qty for s in sales)

if __name__ == "__main__":
    sales = load_sales(Path("ventas.csv"))
    print(f"Ventas válidas: {len(sales)} | Total: ${total_amount(sales):.2f}")
```

3. Corre `mypy`:

```bash
poetry run mypy clean_csv.py
```

**Aprendes**: `dataclass`, tipos básicos, comprensiones/generadores, `try/except`, `logger.warning`.

---

## Caso 2: “Filtra y transforma con comprensiones”

**Objetivo**: dado un listado de usuarios, quedarse con los activos y normalizar sus emails.

```python
from typing import Iterable

def normalize_email(email: str) -> str:
    return email.strip().lower()

def active_emails(users: Iterable[dict[str, str]]) -> list[str]:
    # users: [{'email': 'X', 'active': '1'/'0'}, ...]
    return [
        normalize_email(u["email"])
        for u in users
        if u.get("active") == "1" and "@" in u.get("email", "")
    ]

if __name__ == "__main__":
    data = [
        {"email": "  ANA@EXAMPLE.COM ", "active": "1"},
        {"email": "bad-email", "active": "1"},
        {"email": "bob@example.com", "active": "0"},
    ]
    print(active_emails(data))  # ['ana@example.com']
```

**Aprendes**: comprensión con condición, funciones pequeñas tipadas y reusables.

---

## Caso 3: “Pipeline perezoso con generadores”

**Objetivo**: procesar un archivo grande línea a línea sin reventar memoria.

```python
from collections.abc import Iterator
from pathlib import Path

def read_lines(path: Path) -> Iterator[str]:
    with path.open(encoding="utf-8") as f:
        for line in f:
            yield line.rstrip("\n")

def only_errors(lines: Iterator[str]) -> Iterator[str]:
    for ln in lines:
        if "ERROR" in ln:
            yield ln

def count(lines: Iterator[str]) -> int:
    return sum(1 for _ in lines)

if __name__ == "__main__":
    n = count(only_errors(read_lines(Path("app.log"))))
    print(f"Errores: {n}")
```

**Aprendes**: *pipelines* con generadores, *typing* con `Iterator`, coste de memoria mínimo.

---

## Checklist de “hecho”

* [ ] mypy corre sin errores (o con mensajes entendibles).
* [ ] Todas las funciones públicas tienen anotaciones de tipo.
* [ ] No hay `except:` genéricos.
* [ ] Los logs tienen formato consistente y se usan `info/warning/exception` según corresponda.
* [ ] Comprensiones/generadores usados donde aportan claridad o performance.

# Guía práctica: Manejo de archivos en Python

**CSV, JSON, Parquet · `with open` y `encoding` · `json`, `csv`, `pathlib`**

---

## 1) Objetivos

Al terminar podrás:

* Leer y escribir **CSV** y **JSON** usando **stdlib** (`csv`, `json`, `pathlib`) con buen manejo de `encoding`.
* Entender cuándo y cómo usar `with open(...)` para **cerrar** correctamente archivos.
* Leer y escribir **Parquet** usando **paquetes externos** (p. ej., `pyarrow` o `pandas`) y conocer sus diferencias.
* Construir utilidades pequeñas y reutilizables para pipelines de datos.

> Requisitos: Python 3.12. Para Parquet: `pyarrow` o `pandas` (opcional).

---

## 2) `with open` y `encoding` (lo esencial)

* Usa `with open(...)` **siempre**: garantiza cierre del archivo, incluso ante excepciones.
* Texto: modos `"rt"` / `"wt"`. Binario: `"rb"` / `"wb"`.
* **Encoding recomendado**: `"utf-8"` (universal).
* CSV (texto) en Windows: `newline=""` evita líneas en blanco extra.

```python
from pathlib import Path

path = Path("data/ejemplo.txt")
path.parent.mkdir(exist_ok=True)

# Escritura segura
with path.open("wt", encoding="utf-8") as f:
    f.write("Hola mundo\n")

# Lectura segura
with path.open("rt", encoding="utf-8") as f:
    contenido = f.read()
```

> Si recibes archivos con BOM: usa `"utf-8-sig"` al leer. Si hay caracteres inválidos, considera `errors="replace"` (mejor loguear y corregir en origen).

---

## 3) CSV con `csv` (stdlib)

### 3.1 Leer (filas como diccionarios)

```python
import csv
from pathlib import Path
from typing import Iterator

def leer_csv_dict(path: Path, encoding: str = "utf-8") -> Iterator[dict[str, str]]:
    with path.open("rt", encoding=encoding, newline="") as f:
        reader = csv.DictReader(f)
        for row in reader:
            yield row

# Uso
for r in leer_csv_dict(Path("data/ventas.csv")):
    print(r["sku"], float(r["price"]) * int(r["qty"]))
```

### 3.2 Escribir (cabeceras + quoting)

```python
import csv
from pathlib import Path

rows = [
    {"sku": "A-01", "price": 9.5, "qty": 3},
    {"sku": "A,02", "price": 12.0, "qty": 1},  # coma en dato → requiere quoting
]

out = Path("data/ventas_out.csv")
with out.open("wt", encoding="utf-8", newline="") as f:
    writer = csv.DictWriter(f, fieldnames=["sku", "price", "qty"], quoting=csv.QUOTE_MINIMAL)
    writer.writeheader()
    writer.writerows(rows)
```

### 3.3 Detectar separador con `csv.Sniffer`

```python
import csv
from pathlib import Path

def detectar_dialect(path: Path, encoding="utf-8") -> csv.Dialect:
    with path.open("rt", encoding=encoding, newline="") as f:
        sample = f.read(4096)
        return csv.Sniffer().sniff(sample)

dialect = detectar_dialect(Path("data/ventas.csv"))
with Path("data/ventas.csv").open("rt", encoding="utf-8", newline="") as f:
    reader = csv.reader(f, dialect)
    for row in reader:
        print(row)
```

**Buenas prácticas CSV**

* Convierte tipos **después** de leer: `float(row["price"])`, `int(row["qty"])`.
* Loguea filas descartadas (no silencies errores).
* Si el archivo es grande, procesa **fila a fila** (iterador), no cargues todo a memoria.

---

## 4) JSON con `json` (stdlib)

### 4.1 Leer un objeto/array JSON

```python
import json
from pathlib import Path
from typing import Any

def leer_json(path: Path, encoding="utf-8") -> Any:
    with path.open("rt", encoding=encoding) as f:
        return json.load(f)  # ¡carga todo en memoria!

data = leer_json(Path("data/config.json"))
```

### 4.2 Escribir JSON “humano”

```python
import json
from pathlib import Path

obj = {"name": "Ana", "tags": ["data", "python"], "active": True}
with Path("data/config_out.json").open("wt", encoding="utf-8") as f:
    json.dump(obj, f, ensure_ascii=False, indent=2, sort_keys=True)
```

### 4.3 JSON por líneas (NDJSON)

Para datos grandes, usa **JSON Lines** (una línea = un objeto):

```python
from pathlib import Path
import json

def stream_ndjson(path: Path):
    with path.open("rt", encoding="utf-8") as f:
        for line in f:
            if line.strip():
                yield json.loads(line)

def write_ndjson(path: Path, items: list[dict]):
    with path.open("wt", encoding="utf-8") as f:
        for it in items:
            f.write(json.dumps(it, ensure_ascii=False) + "\n")
```

> El `json` de la stdlib no **stremea** arrays enormes; para eso considera formatos línea-a-línea (NDJSON) o librerías especializadas.

---

## 5) Parquet (requiere librerías externas)

**No** está en la stdlib. Dos caminos comunes:

### 5.1 `pyarrow` puro (rápido y sin pandas)

```bash
poetry add pyarrow
```

```python
import pyarrow as pa
import pyarrow.parquet as pq
from pathlib import Path

records = [
    {"sku": "A-01", "price": 9.5, "qty": 3},
    {"sku": "A-02", "price": 12.0, "qty": 1},
]
table = pa.Table.from_pylist(records)
pq.write_table(table, "data/ventas.parquet")

table2 = pq.read_table("data/ventas.parquet")
print(table2.to_pydict())
```

### 5.2 Con `pandas` (cómodo si ya trabajas dataframes)

```bash
poetry add pandas pyarrow
```

```python
import pandas as pd

df = pd.DataFrame(records)
df.to_parquet("data/ventas_pd.parquet", index=False)  # usa pyarrow por debajo
df2 = pd.read_parquet("data/ventas_pd.parquet")
print(df2.dtypes)
```

**Tips Parquet**

* Define tipos esperados: evita mezclar números/strings en una misma columna.
* Parquet **es columnar**: perfecto para analítica; compresión eficiente.

---

## 6) Mini utilidades reutilizables (con tipos + logs)

```python
from __future__ import annotations
import csv, json, logging
from pathlib import Path
from typing import Iterable

logging.basicConfig(level=logging.INFO, format="%(levelname)s | %(message)s")
log = logging.getLogger("io_utils")

def csv_to_ndjson(csv_path: Path, jsonl_path: Path, *, encoding="utf-8") -> int:
    """Convierte CSV→NDJSON. Devuelve filas válidas escritas."""
    count = 0
    with csv_path.open("rt", encoding=encoding, newline="") as fin, \
         jsonl_path.open("wt", encoding="utf-8") as fout:
        reader = csv.DictReader(fin)
        for i, row in enumerate(reader, start=2):
            try:
                row["price"] = float(row["price"])
                row["qty"] = int(row["qty"])
                fout.write(json.dumps(row, ensure_ascii=False) + "\n")
                count += 1
            except Exception:
                log.warning("Fila %d inválida: %r", i, row, exc_info=True)
    log.info("Escritas %d filas válidas", count)
    return count
```

---

## 7) CLI de ejemplo (CSV→Parquet con `pyarrow`)

```python
import argparse
import csv
from pathlib import Path
import pyarrow as pa
import pyarrow.parquet as pq

def csv_to_parquet(csv_path: Path, parquet_path: Path, encoding="utf-8") -> None:
    with csv_path.open("rt", encoding=encoding, newline="") as f:
        reader = csv.DictReader(f)
        rows = [ {k: (float(v) if k in {"price"} else int(v) if k=="qty" else v) for k,v in r.items()} 
                 for r in reader ]
    table = pa.Table.from_pylist(rows)
    pq.write_table(table, parquet_path)

if __name__ == "__main__":
    ap = argparse.ArgumentParser()
    ap.add_argument("csv_in", type=Path)
    ap.add_argument("parquet_out", type=Path)
    ap.add_argument("--encoding", default="utf-8")
    args = ap.parse_args()
    csv_to_parquet(args.csv_in, args.parquet_out, args.encoding)
```

---

## 8) Casos de uso (fáciles)

### Caso A: “CSV → JSON ordenado”

* **Entrada**: `ventas.csv` con columnas `sku,price,qty`.
* **Tarea**: leer CSV (con `DictReader`), convertir `price` y `qty` a numérico y escribir `ventas.json` **ordenado** por `sku`, con `indent=2`, `ensure_ascii=False`.
* **Bonus**: detectar separador con `Sniffer`.

### Caso B: “NDJSON → CSV”

* **Entrada**: `eventos.jsonl` (cada línea, un objeto con `ts`, `user_id`, `event`).
* **Tarea**: escribir `eventos.csv` con esas tres columnas (en ese orden), cuidando `newline=""`.
* **Bonus**: contar líneas y loguear cada 100k.

### Caso C: “CSV → Parquet”

* **Entrada**: `ventas.csv` (puede tener `;` como separador).
* **Tarea**: normalizar tipos y escribir `ventas.parquet` con `pyarrow`.
* **Bonus**: validar que `price >= 0` y `qty > 0`; descartar filas inválidas con log `warning`.

---


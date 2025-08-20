# Introducción a **Pandas** y **Polars**

---

## ¿Qué es cada cosa?

* **Pandas**: la librería “clásica” de análisis y manipulación tabular en Python; ofrece estructuras flexibles (Series/DataFrame) y un ecosistema enorme de tutoriales y extensiones. ([pandas.pydata.org][1])
* **Polars**: DataFrames modernos con foco en rendimiento. Su motor está escrito en **Rust**, usa **procesamiento columnar y vectorizado**, y es **multihilo** por diseño. Provee modo **eager** y **lazy**. ([docs.pola.rs][2], [pola.rs][3])

> Rendimiento: en benchmarks de su propio proyecto, Polars reporta **> 30×** frente a pandas en consultas tipo TPC-H (según el escenario/hardware). Toma estos números como **referencia** y no como promesa universal; hay comparativas independientes (H2O.ai) que también muestran ventajas claras. ([pola.rs][3], [h2oai.github.io][4])

---

## Filosofía de ejecución: **eager vs. lazy**

* **Pandas**: *eager* — cada operación se ejecuta en el acto. Sencillo de razonar y depurar paso a paso. ([pandas.pydata.org][1])
* **Polars**: soporta *eager* (similar a pandas) y *lazy* con **plan de consulta** optimizable; en *lazy* las operaciones se aplazan y se ejecutan cuando haces `.collect()`, permitiendo reordenar, empujar filtros y paralelizar. ([docs.pola.rs][5])

---

## Misma tarea, dos estilos

### Cargar, filtrar, agrupar, ordenar

**Pandas (eager)**

```python
import pandas as pd

df = pd.read_csv("ventas.csv")
out = (df[df["total"] > 100]
       .groupby("cliente", as_index=False)["total"].sum()
       .sort_values("total", ascending=False))
```

**Polars (eager)**

```python
import polars as pl

df = pl.read_csv("ventas.csv")
out = (df.filter(pl.col("total") > 100)
       .group_by("cliente")
       .agg(pl.col("total").sum())
       .sort("total", descending=True))
```

**Polars (lazy)**

```python
import polars as pl

lf = pl.scan_csv("ventas.csv")  # lazy: no lee aún
out = (lf.filter(pl.col("total") > 100)
       .group_by("cliente")
       .agg(pl.col("total").sum())
       .sort("total", descending=True)
       .collect())               # ejecuta el plan aquí
```

*(En `lazy`, Polars construye y optimiza un plan antes de ejecutar.)* ([docs.pola.rs][5])

---

## Tabla de equivalencias (rápida)

| Acción             | Pandas                       | Polars (eager)                            |
| ------------------ | ---------------------------- | ----------------------------------------- |
| Leer CSV           | `pd.read_csv(...)`           | `pl.read_csv(...)`                        |
| Selección columnas | `df[["a","b"]]`              | `df.select(["a","b"])`                    |
| Filtro filas       | `df[df.a>0]`                 | `df.filter(pl.col("a")>0)`                |
| Agregación         | `df.groupby("k")["x"].sum()` | `df.group_by("k").agg(pl.col("x").sum())` |
| Ordenar            | `df.sort_values("x")`        | `df.sort("x")`                            |
| Escribir Parquet   | `df.to_parquet(...)`         | `df.write_parquet(...)`                   |

---

## ¿Cuándo elegir cada uno?

**Elige Pandas si…**

* Quieres **aprender primero** con la API más difundida y abundancia de recursos.
* Tu dataset cabe en memoria y priorizas rapidez de prototipado. ([pandas.pydata.org][1])

**Elige Polars si…**

* Buscas **rendimiento** out-of-the-box (multihilo, SIMD) y/o **pipelines** optimizables con *lazy*.
* Trabajas con datos grandes (lectura selectiva, streaming híbrido) o quieres paralelismo sin Dask. ([docs.pola.rs][2], [pola.rs][3])

> Nota: si ya estás casado con Pandas pero necesitas acelerar ciertas cargas, también existen rutas como **cuDF** (GPU) que aceleran pandas sin cambiar código, aunque eso ya es otro stack. ([NVIDIA Developer][6])

---

## Instalación “starter”

```bash
# Pandas
poetry add pandas

# Polars con CPU
poetry add polars

# (Opcional) Parquet/Arrow en pandas
poetry add pyarrow
```

---

## Mini-retos para practicar (5–10 min c/u)

1. **Top-N por grupo** (ventas por cliente) en Pandas y en Polars (*eager*).
2. Repite #1 en **Polars Lazy** leyendo de CSV con `scan_csv` y compara tiempos. ([docs.pola.rs][5])
3. Crea una **columna derivada** (margen = precio\*qty – costo) y filtra outliers (z-score) en ambos.

---

## Resumen

* Pandas = **flexibilidad y ergonomía**; Polars = **motor en Rust**, **multihilo** y **optimización de consultas** con *lazy*. ([pandas.pydata.org][1], [docs.pola.rs][2])
* En ciertos benchmarks, Polars muestra **> 30×** frente a pandas; tu **milla real** dependerá de datos y transformaciones. Mide siempre en tu caso. ([pola.rs][3])

[1]: https://pandas.pydata.org/docs/getting_started/overview.html?utm_source=chatgpt.com "Package overview — pandas 2.3.1 documentation - PyData |"
[2]: https://docs.pola.rs/?utm_source=chatgpt.com "Polars user guide: Index"
[3]: https://pola.rs/?utm_source=chatgpt.com "Polars — DataFrames for the new era"
[4]: https://h2oai.github.io/db-benchmark/?utm_source=chatgpt.com "Database-like ops benchmark - GitHub Pages"
[5]: https://docs.pola.rs/user-guide/concepts/lazy-api/?utm_source=chatgpt.com "Lazy API"
[6]: https://developer.nvidia.com/blog/rapids-cudf-accelerates-pandas-nearly-150x-with-zero-code-changes/?utm_source=chatgpt.com "RAPIDS cuDF Accelerates pandas Nearly 150x with Zero ..."

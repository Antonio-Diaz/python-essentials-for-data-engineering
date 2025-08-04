## Guía práctica: Entornos y configuración en Python 3.12

*Instalación, venv vs virtualenv, introducción a Poetry, uso de la CLI y logging básico*

---

### 1. Contexto y objetivos

Al terminar esta sección podrás:

1. **Instalar** Python 3.12 de forma segura en cualquier SO (Windows, macOS, Linux).
2. **Crear y administrar entornos virtuales** con `venv` y `virtualenv`, entendiendo cuándo usar cada uno.
3. **Gestionar dependencias** con Poetry y publicar tu primer paquete local.
4. **Trabajar en la línea de comandos** con buenas prácticas de shell scripting.
5. **Configurar un sistema de logging** sencillo pero profesional para tus scripts y notebooks.

> **Duración recomendada**: 1 h 30 min de teoría guiada + 1 h de práctica autodirigida.

---

### 2. Instalación de Python 3.12

| Sistema                   | Pasos clave                                                                                                                                                |
| ------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Windows**               | 1. Descarga el instalador oficial desde *python.org*. <br>2. Marca “Add Python to PATH”. <br>3. Activa “Install for all users” para evitar rutas extrañas. |
| **macOS**                 | 1. Instala Homebrew: `/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"`. <br>2. `brew install python@3.12`. |
| **Linux (Debian/Ubuntu)** | 1. `sudo add-apt-repository ppa:deadsnakes/ppa` <br>2. `sudo apt update && sudo apt install python3.12 python3.12-venv python3.12-dev`.                    |

> **Tip**: comprueba la versión con `python3.12 --version` y nunca desinstales la versión del sistema.

---

### 3. venv vs virtualenv

| Característica                     | `venv` (stdlib)                                  | `virtualenv` (externo)                   |
| ---------------------------------- | ------------------------------------------------ | ---------------------------------------- |
| Instalación                        | Incluido desde Python 3.3                        | `pip install virtualenv`                 |
| Velocidad de creación              | Rápida (usa binarios ya instalados)              | Más rápida (usa symlinks agresivos)      |
| Compatibilidad múltiples versiones | Limitada (depende del ejecutable que lo invoque) | Excelente (`virtualenv -p python3.12 …`) |
| Extras                             | Sin *activation scripts* avanzados               | Soporta *prompt* custom y *seed pip*     |

**Uso básico**

```bash
# Crear con venv
python3.12 -m venv .venv
source .venv/bin/activate  # Unix
.\.venv\Scripts\activate   # Windows

# Crear con virtualenv para 3.12 explícito
virtualenv -p python3.12 venv312
source venv312/bin/activate
```

**Cuándo usar cada uno**

* *Proyectos sencillos / CI ligero*: `venv`.
* *Múltiples intérpretes, builds reproducibles, máquinas compartidas*: `virtualenv`.

---

### 4. Introducción a Poetry (gestión de dependencias + empaquetado)

1. **Instalación**

   ```bash
   curl -sSL https://install.python-poetry.org | python3 -
   ```

   Añade `$HOME/.local/bin` a tu `PATH`.

2. **Crear proyecto**

   ```bash
   mkdir awesome_project && cd awesome_project
   poetry init --name awesome_project --python ^3.12
   ```

3. **Instalar dependencias**

   ```bash
   poetry add pandas==2.2.0 rich
   poetry add --group dev pytest black
   ```

4. **Entrar al entorno**

   ```bash
   poetry shell
   ```

5. **Publicar paquete local**

   ```bash
   poetry build          # genera wheel + sdist
   pip install dist/*.whl  # prueba instalación local
   ```

> **Ventaja clave**: bloquea versiones en `poetry.lock`, facilitando reproducibilidad 100 %.

---

### 5. Uso de la CLI y buenas prácticas

| Buen hábito                  | Ejemplo                                              |
| ---------------------------- | ---------------------------------------------------- |
| **Scripts auto-ejecutables** | Añade shebang: `#!/usr/bin/env python3`              |
| **Variables de entorno**     | `export PYTHONWARNINGS="ignore::DeprecationWarning"` |
| **Ayuda transparente**       | `python -m pip --help` vs `pip --help`               |
| **Aliases productivos**      | `alias act="source .venv/bin/activate"`              |

> **Mini-reto**: crea un script `project_clean.sh` que elimine carpetas `__pycache__` y archivos `*.pyc` del proyecto.

---

### 6. Configuración básica de logging

```python
import logging
from pathlib import Path

LOG_PATH = Path("logs/app.log")
LOG_PATH.parent.mkdir(exist_ok=True)

logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s | %(levelname)-8s | %(name)s | %(message)s",
    handlers=[
        logging.FileHandler(LOG_PATH),
        logging.StreamHandler()
    ],
)

logger = logging.getLogger("awesome_project")

def main():
    logger.info("Script iniciado")
    # …

if __name__ == "__main__":
    main()
```

**Puntos clave**

* Usa `__name__` para jerarquía automática.
* Escala con `logging.config.dictConfig` cuando tengas múltiples módulos.

---

## Casos de uso

### A. Nivel fácil

| Caso                                    | Descripción                                                                              | Pasos                                                                       |
| --------------------------------------- | ---------------------------------------------------------------------------------------- | --------------------------------------------------------------------------- |
| **1. Script “Hello World” autenticado** | Escribe un script que imprima “Hola Data Engineer” y registre la ejecución en `app.log`. | 1. Crea `.venv` con `venv`. <br>2. Añade logging básico.                    |
| **2. CLI para convertir CSV → Parquet** | Convierte un archivo `data.csv` a Parquet usando Pandas.                                 | 1. `poetry add pandas pyarrow`. <br>2. Usa `argparse` y maneja excepciones. |

### B. Nivel intermedio

| Caso                                  | Descripción                                                                                                                          | Retos técnicos                                                                                                                 |
| ------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------ |
| **3. Micro-ETL con Poetry + logging** | Descarga un dataset público (ej. Open AQ), lo transforma y almacena en DuckDB. Todo dentro de un paquete Poetry.                     | - Diseña estructura `src/<package>/`. <br>- Usa `logging` con niveles INFO/ERROR. <br>- Crea *task* `make load` en `Makefile`. |
| **4. Switch dinámico de entornos**    | Crea un *wrapper* Bash que detecte si existe `.venv` o `poetry.lock`; activa el entorno adecuado antes de ejecutar `python main.py`. | - Detectar archivos con `[ -f … ]`. <br>- Usar `exec` para reemplazar el shell.                                                |

---

### Actividades de refuerzo

1. **Kata 5 × 5**: genera cinco entornos virtuales en carpeta `playground/` y activa cada uno en menos de cinco minutos. Documenta comandos usados.
2. **Debugging Challenge**: un compañero “rompe” tu `pyproject.toml` duplicando dependencias. Encuentra y soluciona el conflicto con `poetry lock --no-update`.
3. **Logging Drill**: extiende el script del caso fácil #2 para que rote logs cada 1 MB usando `logging.handlers.RotatingFileHandler`.

---

### Checklist de finalización

* [ ] Python 3.12 instalado y verificado
* [ ] Entorno virtual creado (`venv` o `virtualenv`)
* [ ] Proyecto inicial Poetry con dependencias y lockfile
* [ ] Primer script con logging a consola y archivo
* [ ] Casos de uso fáciles completados
* [ ] Al menos un caso intermedio terminado y subido a GitHub

> **Siguiente paso**: profundizar en *typing* y *mypy* (Sesión 2), aplicando estos conceptos sobre los mismos scripts.

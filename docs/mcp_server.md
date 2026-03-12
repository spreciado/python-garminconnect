# Garmin Connect como servidor MCP

Esta guía muestra cómo usar este repositorio como base para montar un servidor MCP que exponga consultas de Garmin Connect (actividades, métricas diarias y estado de entrenamiento) para usarlo desde clientes MCP como Claude Desktop, Cursor o VS Code.

## 1) Idea general

`garminconnect` ya te da:
- Autenticación con tokens persistidos en `~/.garminconnect`.
- Métodos de consulta para actividad y salud (por ejemplo `get_activities`, `get_stats`, `get_heart_rates`, `get_training_status`).

Tu servidor MCP solo tiene que:
1. Inicializar un cliente `Garmin` autenticado.
2. Exponer herramientas MCP que llamen esos métodos.
3. Normalizar los parámetros (fecha, límite, etc.) y devolver JSON.

## 2) Dependencias

En un entorno virtual:

```bash
python -m venv .venv
source .venv/bin/activate
pip install garminconnect mcp
```

> Nota: este repo ya incluye ejemplos de login y reutilización de tokens en `example.py`.

## 3) Servidor MCP mínimo

Crea `mcp_server.py` (fuera o dentro del repo):

```python
from __future__ import annotations

import os
from datetime import date
from pathlib import Path
from typing import Any

from garminconnect import Garmin
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("garmin-connect")


def _get_client() -> Garmin:
    """Inicializa Garmin usando token store existente."""
    tokenstore = Path(os.getenv("GARMINTOKENS", "~/.garminconnect")).expanduser()

    # Usa tokens ya creados con example.py/demo.py.
    garmin = Garmin()
    garmin.login(str(tokenstore))
    return garmin


@mcp.tool()
def resumen_diario(fecha: str | None = None) -> dict[str, Any]:
    """Devuelve resumen diario (pasos, calorías, distancia, etc.)."""
    cdate = fecha or date.today().isoformat()
    api = _get_client()
    return api.get_stats(cdate)


@mcp.tool()
def actividades_recientes(limite: int = 10, inicio: int = 0) -> list[dict[str, Any]]:
    """Lista actividades recientes."""
    api = _get_client()
    return api.get_activities(start=inicio, limit=limite)


@mcp.tool()
def actividad_detalle(activity_id: str) -> dict[str, Any]:
    """Detalle de una actividad por id."""
    api = _get_client()
    return api.get_activity(activity_id)


@mcp.tool()
def estado_entrenamiento(fecha: str | None = None) -> dict[str, Any]:
    """Training status para una fecha (YYYY-MM-DD)."""
    cdate = fecha or date.today().isoformat()
    api = _get_client()
    return api.get_training_status(cdate)


if __name__ == "__main__":
    mcp.run()
```

## 4) Autenticación (una sola vez)

Antes de ejecutar el servidor MCP, genera tokens:

```bash
python example.py
```

Eso crea tokens en `~/.garminconnect` (o en `GARMINTOKENS` si la defines).

## 5) Ejecutar servidor MCP

```bash
python mcp_server.py
```

El servidor queda en modo stdio (lo que esperan la mayoría de clientes MCP).

## 6) Conectar cliente MCP

Configura tu cliente para lanzar:

- **command**: `python`
- **args**: `[/ruta/a/mcp_server.py]`
- **env** (opcional): `GARMINTOKENS=/ruta/a/tokens`

Ejemplo conceptual (JSON):

```json
{
  "mcpServers": {
    "garmin-connect": {
      "command": "python",
      "args": ["/workspace/python-garminconnect/mcp_server.py"],
      "env": {
        "GARMINTOKENS": "/home/tu_usuario/.garminconnect"
      }
    }
  }
}
```

## 7) Consultas en tiempo real: recomendaciones prácticas

Garmin Connect no es streaming en tiempo real; es polling sobre endpoints HTTP. Para que se sienta “tiempo real”:

- Consulta ventanas cortas (última actividad, stats del día).
- Usa caché de 30-120s para evitar rate-limit.
- Implementa reintento exponencial ante 429/5xx.
- Evita pedir detalles pesados en cada consulta; pide resumen + detalle bajo demanda.

## 8) Endpoints útiles de este repo para MCP

- `get_activities(start, limit)` para timeline reciente.
- `get_activity(activity_id)` para inspección puntual.
- `get_stats(fecha)` para resumen diario.
- `get_heart_rates(fecha)` para ritmo cardíaco diario.
- `get_hrv_data(fecha)` para HRV.
- `get_training_status(fecha)` y `get_training_readiness(fecha)` para preparación/estado.

## 9) Seguridad

- No hardcodees email/password en el servidor MCP.
- Usa solo token store local y permisos restrictivos (`chmod 700 ~/.garminconnect`).
- Si vas a compartir el servidor, hazlo detrás de un runner controlado y con cuenta dedicada.

## 10) Siguiente paso recomendado

Si quieres, puedes extender este servidor con herramientas MCP por dominio:

- `hoy_resumen_salud`
- `ultimo_entrenamiento`
- `comparar_7_dias`
- `recomendacion_carga` (lógica tuya encima de datos Garmin)

Así separas la capa de datos (este repo) de la capa de inteligencia (el LLM cliente).

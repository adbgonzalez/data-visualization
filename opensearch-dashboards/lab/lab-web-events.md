# Laboratorio guiado: `web_events`

## 1. Obxectivo

Neste laboratorio imos construír un pipeline completo de datos:

```text
produtor Python -> HTTP -> Data Prepper -> OpenSearch -> OpenSearch Dashboards
```

O escenario é `web_events`, pensado como práctica guiada para aprender o fluxo completo e, en particular, o papel de Data Prepper na transformación de eventos textuais en documentos JSON.

---

## 2. Resultado esperado

Ao rematar o laboratorio deberías ter:

- un produtor en Python que xere eventos simulados e os envíe periodicamente por HTTP a Data Prepper
- un pipeline de Data Prepper que reciba eses eventos, extraia os campos e os envíe a OpenSearch
- un índice `web_events` con documentos xa estruturados
- un dashboard básico con varias visualizacións sobre ese índice

---

## 3. Campos do escenario

Os eventos do laboratorio deben conter estes campos:

- `timestamp`
- `servizo`
- `client_ip`
- `method`
- `endpoint`
- `status`
- `latency_ms`
- `bytes_out`
- `user_agent`
- `mensaxe`

Neste laboratorio imos tratar:

- `status` como `integer`
- `latency_ms` como valor decimal
- `bytes_out` como `integer`

---

## 4. Formato do evento de entrada

O produtor xera unha liña de texto desestruturada ou semi-estruturada e envíaa por HTTP a Data Prepper dentro dun obxecto JSON.

O contido textual do evento terá un formato como este:

```text
2026-03-27T10:15:23Z servizo=web client_ip=10.0.0.10 method=GET endpoint=/login status=200 latency_ms=124.52 bytes_out=2048 user_agent="Mozilla/5.0" mensaxe="GET /login -> 200 (124.52 ms)"
```

O produtor enviará esa liña dentro dun `payload` JSON coma este:

```json
[
  {
    "message": "2026-03-27T10:15:23Z servizo=web client_ip=10.0.0.10 method=GET endpoint=/login status=200 latency_ms=124.52 bytes_out=2048 user_agent=\"Mozilla/5.0\" mensaxe=\"GET /login -> 200 (124.52 ms)\""
  }
]
```

Este formato é útil porque:

- segue sendo lexible a simple vista
- permite manter o evento “en cru” nun campo `message`
- é relativamente doado de parsear con `grok` en Data Prepper

---

## 5. Produtor de eventos

Neste exemplo úsase o porto `2024` para evitar colisións con outros pipelines ou servizos HTTP do contorno. Antes de lanzar o laboratorio, comproba que ese porto non estea xa en uso. Se no teu contorno está ocupado, podes escoller outro, pero lembra que o cambio debe facerse tanto no produtor como no pipeline.

O seguinte script Python xera unha liña nova en cada iteración e envíaa por HTTP a Data Prepper. O `timestamp` é sempre o momento actual da xeración do evento:

```python
#!/usr/bin/env python3
import time
import random
import requests
from datetime import datetime, timezone

DATA_PREPPER_URL = "http://data-prepper:2024/web-events-pipeline/logs"

POLL_SECONDS = 2

SERVICES = ["web", "api", "auth"]
METHODS = ["GET", "POST", "PUT", "DELETE"]
ENDPOINTS = [
    "/",
    "/login",
    "/logout",
    "/api/v1/items",
    "/api/v1/orders",
    "/health",
]
STATUS_CODES = [200, 201, 204, 400, 401, 403, 404, 500, 502, 503]
USER_AGENTS = [
    "Mozilla/5.0",
    "curl/8.0",
    "PostmanRuntime/7.36",
    "Python-requests/2.31",
]
IPS = ["10.0.0.10", "10.0.0.11", "10.0.0.12", "192.168.1.20", "172.16.0.5"]


def now_utc_iso():
    return datetime.now(timezone.utc).isoformat().replace("+00:00", "Z")


print(f"[web_events] enviando a {DATA_PREPPER_URL}")

while True:
    servizo = random.choice(SERVICES)
    method = random.choice(METHODS)
    endpoint = random.choice(ENDPOINTS)
    status = random.choice(STATUS_CODES)
    latency_ms = round(random.uniform(5, 1200), 2)
    bytes_out = random.randint(200, 20000)
    user_agent = random.choice(USER_AGENTS)
    client_ip = random.choice(IPS)
    mensaxe = f"{method} {endpoint} -> {status} ({latency_ms} ms)"

    line = (
        f'{now_utc_iso()} '
        f'servizo={servizo} '
        f'client_ip={client_ip} '
        f'method={method} '
        f'endpoint={endpoint} '
        f'status={status} '
        f'latency_ms={latency_ms} '
        f'bytes_out={bytes_out} '
        f'user_agent="{user_agent}" '
        f'mensaxe="{mensaxe}"'
    )

    payload = [{"message": line}]
    response = requests.post(DATA_PREPPER_URL, json=payload, timeout=5)
    response.raise_for_status()

    print("[web_events] enviado:", line)
    time.sleep(POLL_SECONDS)
```

Exemplo de execución:

```powershell
python .\producer_web_events.py
```

Importante:

- o produtor envía cada evento cun `POST` HTTP a Data Prepper
- o `payload` vai en formato JSON e o campo que contén o texto bruto é `message`
- a URL configurada en `DATA_PREPPER_URL` debe coincidir co porto e coa ruta configurados no pipeline
- se Data Prepper non está escoitando nesa URL, o produtor fallará ao enviar o evento

---

## 6. Pipeline de Data Prepper

O pipeline de Data Prepper pode ser este:

```yaml
web-events-pipeline:
  source:
    http:
      port: 2024
      path: "/web-events-pipeline/logs"
      health_check_service: true
      unauthenticated_health_check: true
  processor:
    - grok:
        match:
          message:
            - '^%{TIMESTAMP_ISO8601:timestamp} servizo=%{DATA:servizo} client_ip=%{IP:client_ip} method=%{WORD:method} endpoint=%{DATA:endpoint} status=%{NUMBER:status} latency_ms=%{NUMBER:latency_ms} bytes_out=%{NUMBER:bytes_out} user_agent="%{DATA:user_agent}" mensaxe="%{GREEDYDATA:mensaxe}"$'
        break_on_match: true
        named_captures_only: true
    - convert_type:
        keys:
          - "status"
          - "bytes_out"
        type: "integer"
    - convert_type:
        key: "latency_ms"
        type: "double"
  sink:
    - opensearch:
        hosts: ["https://opensearch:9200"]
        username: "admin"
        password: "Opensearch#2025"
        insecure: true
        index_type: "custom"
        index: "web_events"
```

O pipeline debe facer estes pasos:

1. Recibir os eventos por HTTP na ruta `/web-events-pipeline/logs`.
2. Extraer os campos do texto almacenado en `message` cun patrón `grok`.
3. Converter os tipos numéricos ao formato axeitado.
4. Inserir os documentos resultantes no índice `web_events` de OpenSearch.

Importante:

- Data Prepper debe estar escoitando no porto configurado, neste exemplo `2024`
- a ruta configurada en `path` debe coincidir exactamente coa URL usada polo produtor
- se cambias o porto ou a ruta HTTP, terás que actualizar tamén `DATA_PREPPER_URL` no produtor
- antes de lanzar o laboratorio, comproba que o porto elixido non estea xa ocupado por outro servizo
- no exemplo actual, o produtor envía os eventos a `http://data-prepper:2024/web-events-pipeline/logs`

---

## 7. Creación do índice

Antes de cargar os datos, crea o índice `web_events` cunha petición como esta:

```http
PUT web_events
{
  "mappings": {
    "properties": {
      "timestamp": { "type": "date" },
      "servizo": { "type": "keyword" },
      "client_ip": { "type": "ip" },
      "method": { "type": "keyword" },
      "endpoint": { "type": "keyword" },
      "status": { "type": "integer" },
      "latency_ms": { "type": "float" },
      "bytes_out": { "type": "integer" },
      "user_agent": { "type": "keyword" },
      "mensaxe": { "type": "text" }
    }
  }
}
```

---

## 8. Comprobación mínima en OpenSearch

Antes de pasar a Dashboards, comproba:

- que o índice `web_events` existe
- que se indexaron documentos
- que `status` quedou como `integer`
- que `latency_ms` quedou como `float`
- que `bytes_out` quedou como `integer`
- que `timestamp` se recoñece como campo temporal

---

## 9. Visualizacións a construír no dashboard

Constrúe un dashboard básico con estas pezas:

### KPIs

- total de eventos
- total de erros do servidor, por exemplo filtrando `status >= 500`
- latencia media
- volume total de resposta, por exemplo suma de `bytes_out`

### Gráficas

- gráfica de liñas coa evolución temporal do número de eventos
- gráfica de barras cos eventos por `servizo`
- gráfico circular coa distribución por `method`
- gráfica de barras cos códigos `status` máis frecuentes
- histograma da distribución de `latency_ms`

### Táboa

- táboa de datos con resumo por `servizo`
  - `Count`
  - `Average(latency_ms)`
  - `Max(latency_ms)`
  - `Sum(bytes_out)`

### Controis

- filtro por `servizo`
- filtro por `method`
- filtro por `status`
- control de rango para `latency_ms`

---

# Práctica: `ecommerce_events`

## 1. Obxectivo

Nesta práctica imos construír un pipeline completo de datos:

```text
produtor Python -> HTTP -> Data Prepper -> OpenSearch -> OpenSearch Dashboards
```

O escenario é `ecommerce_events`, pensado para aplicar o aprendido no laboratorio guiado nun contexto máis analítico e próximo a operacións de comercio electrónico.

---

## 2. Resultado esperado

Ao rematar a práctica deberías ter:

- un produtor en Python que xere eventos simulados de comercio electrónico e os envíe periodicamente por HTTP a Data Prepper
- un pipeline de Data Prepper que reciba eses eventos, extraia os campos e os envíe a OpenSearch
- un índice `ecommerce_events` con documentos xa estruturados
- un dashboard con visualizacións orientadas á análise do proceso de compra

---

## 3. Campos do escenario

Os eventos desta práctica deben conter estes campos:

- `timestamp`
- `service`
- `level`
- `latency_ms`
- `status`
- `user`
- `event_type`
- `product_id`
- `category`
- `price`

Neste escenario imos tratar:

- `status` como `integer`
- `latency_ms` como valor decimal
- `price` como valor decimal

---

## 4. Formato do evento de entrada

O produtor xera unha liña de texto desestruturada ou semi-estruturada e envíaa por HTTP a Data Prepper dentro dun obxecto JSON.

O contido textual do evento terá un formato como este:

```text
2026-03-27T11:10:25Z service=checkout level=INFO latency_ms=184.42 status=201 user=user_12 event_type=purchase product_id=SKU-1042 category=electronics price=249.90
```

O produtor enviará esa liña dentro dun `payload` JSON coma este:

```json
[
  {
    "message": "2026-03-27T11:10:25Z service=checkout level=INFO latency_ms=184.42 status=201 user=user_12 event_type=purchase product_id=SKU-1042 category=electronics price=249.90"
  }
]
```

Este formato permite:

- ver o evento “en cru” con facilidade
- transformar despois os datos con `grok` en Data Prepper
- xerar un índice máis rico para análise funcional e técnica

---

## 5. Produtor de eventos

Neste exemplo úsase o porto `2025` para evitar colisións co laboratorio guiado ou con outros servizos HTTP do contorno. Antes de lanzar a práctica, comproba que ese porto non estea xa en uso. Se no teu contorno está ocupado, podes escoller outro, pero lembra que o cambio debe facerse tanto no produtor como no pipeline.

O seguinte script Python xera unha liña nova en cada iteración e envíaa por HTTP a Data Prepper. O `timestamp` é sempre o momento actual da xeración do evento:

```python
#!/usr/bin/env python3
import time
import random
import requests
from datetime import datetime, timezone

DATA_PREPPER_URL = "http://data-prepper:2025/ecommerce-events-pipeline/logs"
POLL_SECONDS = 2

SERVICES = ["frontend", "catalog", "cart", "checkout", "payments"]
LEVELS = ["INFO", "WARN", "ERROR"]
EVENT_TYPES = [
    "view_product",
    "search",
    "add_to_cart",
    "remove_from_cart",
    "begin_checkout",
    "purchase",
    "payment_failed",
]
CATEGORIES = ["electronics", "books", "fashion", "home", "sports"]
USERS = [f"user_{n:02d}" for n in range(1, 31)]


def now_utc_iso():
    return datetime.now(timezone.utc).isoformat().replace("+00:00", "Z")


def choose_status(event_type, level):
    if event_type == "payment_failed":
        return random.choice([400, 402, 409, 500, 502])
    if level == "ERROR":
        return random.choice([500, 502, 503])
    if level == "WARN":
        return random.choice([200, 201, 400, 404, 409, 429])
    return random.choice([200, 201, 204])


def choose_price(event_type):
    if event_type in ["purchase", "begin_checkout", "add_to_cart"]:
        return round(random.uniform(10, 800), 2)
    return round(random.uniform(5, 300), 2)


print(f"[ecommerce_events] enviando a {DATA_PREPPER_URL}")

while True:
    event_type = random.choice(EVENT_TYPES)
    level = random.choices(
        population=LEVELS,
        weights=[0.78, 0.15, 0.07],
        k=1,
    )[0]

    if event_type == "payment_failed":
        service = "payments"
        level = random.choice(["WARN", "ERROR"])
    elif event_type in ["purchase", "begin_checkout"]:
        service = random.choice(["checkout", "payments"])
    elif event_type in ["add_to_cart", "remove_from_cart"]:
        service = "cart"
    else:
        service = random.choice(["frontend", "catalog"])

    status = choose_status(event_type, level)
    latency_ms = round(random.uniform(20, 1800), 2)
    price = choose_price(event_type)
    category = random.choice(CATEGORIES)
    product_id = f"SKU-{random.randint(1000, 9999)}"
    user = random.choice(USERS)

    line = (
        f"{now_utc_iso()} "
        f"service={service} "
        f"level={level} "
        f"latency_ms={latency_ms} "
        f"status={status} "
        f"user={user} "
        f"event_type={event_type} "
        f"product_id={product_id} "
        f"category={category} "
        f"price={price}"
    )

    payload = [{"message": line}]
    response = requests.post(DATA_PREPPER_URL, json=payload, timeout=5)
    response.raise_for_status()

    print("[ecommerce_events] enviado:", line)
    time.sleep(POLL_SECONDS)
```

Exemplo de execución:

```powershell
python .\producer_ecommerce_events.py
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
ecommerce-events-pipeline:
  source:
    http:
      port: 2025
      path: "/ecommerce-events-pipeline/logs"
      health_check_service: true
      unauthenticated_health_check: true
  processor:
    - grok:
        match:
          message:
            - '^%{TIMESTAMP_ISO8601:timestamp} service=%{DATA:service} level=%{LOGLEVEL:level} latency_ms=%{NUMBER:latency_ms} status=%{NUMBER:status} user=%{USERNAME:user} event_type=%{DATA:event_type} product_id=%{DATA:product_id} category=%{DATA:category} price=%{NUMBER:price}$'
        break_on_match: true
        named_captures_only: true
    - convert_type:
        keys:
          - "status"
        type: "integer"
    - convert_type:
        keys:
          - "latency_ms"
          - "price"
        type: "double"
  sink:
    - opensearch:
        hosts: ["https://opensearch:9200"]
        username: "admin"
        password: "Opensearch#2025"
        insecure: true
        index_type: "custom"
        index: "ecommerce_events"
```

O pipeline debe facer estes pasos:

1. Recibir os eventos por HTTP na ruta `/ecommerce-events-pipeline/logs`.
2. Extraer os campos do texto almacenado en `message` cun patrón `grok`.
3. Converter os tipos numéricos ao formato axeitado.
4. Inserir os documentos resultantes no índice `ecommerce_events` de OpenSearch.

Importante:

- Data Prepper debe estar escoitando no porto configurado, neste exemplo `2025`
- a ruta configurada en `path` debe coincidir exactamente coa URL usada polo produtor
- se cambias o porto ou a ruta HTTP, terás que actualizar tamén `DATA_PREPPER_URL` no produtor
- antes de lanzar a práctica, comproba que o porto elixido non estea xa ocupado por outro servizo
- no exemplo actual, o produtor envía os eventos a `http://data-prepper:2025/ecommerce-events-pipeline/logs`

---

## 7. Creación do índice

Antes de cargar os datos, crea o índice `ecommerce_events` cunha petición como esta:

```http
PUT ecommerce_events
{
  "mappings": {
    "properties": {
      "timestamp": { "type": "date" },
      "service": { "type": "keyword" },
      "level": { "type": "keyword" },
      "latency_ms": { "type": "float" },
      "status": { "type": "integer" },
      "user": { "type": "keyword" },
      "event_type": { "type": "keyword" },
      "product_id": { "type": "keyword" },
      "category": { "type": "keyword" },
      "price": { "type": "float" }
    }
  }
}
```

---

## 8. Comprobación mínima en OpenSearch

Antes de pasar a Dashboards, comproba:

- que o índice `ecommerce_events` existe
- que se indexaron documentos
- que `status` quedou como `integer`
- que `latency_ms` quedou como valor decimal
- que `price` quedou como valor decimal
- que `timestamp` se recoñece como campo temporal

---

## 9. Visualizacións a construír no dashboard

Constrúe un dashboard orientado á análise do proceso de compra con estas pezas:

### KPIs

- total de eventos
- compras completadas, por exemplo filtrando `event_type:purchase`
- pagos fallidos, por exemplo filtrando `event_type:payment_failed`
- importe total, por exemplo suma de `price`

### Gráficas

- gráfica de liñas coa evolución temporal dos eventos
- gráfica de liñas coa evolución temporal das compras
- gráfica de barras cos eventos por `category`
- gráfica de barras cos eventos por `event_type`
- gráfica de barras cos servizos con máis eventos
- histograma da distribución de `price`

### Táboa

- táboa de datos con resumo por `category`
  - `Count`
  - `Sum(price)`
  - `Average(price)`

### Controis

- filtro por `category`
- filtro por `event_type`
- filtro por `service`
- control de rango para `price`

---

## 10. Rúbrica de corrección

A práctica avaliarase sobre 10 puntos.

### 10.1. Reparto xeral da puntuación

| Bloque | Puntuación |
| --- | ---: |
| Produtor + pipeline + chegada a OpenSearch | 1 punto |
| Deseño xeral do dashboard | 1 punto |
| Visualizacións e filtros | 8 puntos |
| Total | 10 puntos |

### 10.2. Produtor, pipeline e chegada a OpenSearch

| Nivel | Puntuación | Criterio |
| --- | ---: | --- |
| Completo | 1 punto | O produtor xera eventos, o pipeline funciona correctamente e os documentos chegan ao índice `ecommerce_events` cos campos principais ben extraídos. |
| Parcial | 0,5 puntos | O fluxo está parcialmente operativo, pero hai problemas nalgún paso, por exemplo eventos mal parseados, campos incompletos ou inxestión irregular. |
| Insuficiente | 0 puntos | O pipeline non funciona ou os eventos non chegan a OpenSearch. |

### 10.3. Deseño xeral do dashboard

| Nivel | Puntuación | Criterio |
| --- | ---: | --- |
| Completo | 1 punto | O dashboard está ben organizado, resulta lexible e presenta unha estrutura lóxica para analizar os datos. |
| Parcial | 0,5 puntos | O dashboard funciona, pero ten unha organización mellorable ou unha distribución pouco clara. |
| Insuficiente | 0 puntos | O dashboard está desordenado, é difícil de interpretar ou non hai un deseño recoñecible. |

### 10.4. Visualizacións e filtros

Estes 8 puntos repartiranse equitativamente entre 8 elementos.

| Elemento | Puntuación máxima |
| --- | ---: |
| KPI de total de eventos | 1 punto |
| KPI de compras completadas | 1 punto |
| KPI de pagos fallidos ou importe total | 1 punto |
| Gráfica temporal | 1 punto |
| Gráfica por categoría | 1 punto |
| Gráfica por tipo de evento | 1 punto |
| `Data Table` | 1 punto |
| Filtros do dashboard | 1 punto |

### 10.5. Guía de corrección para cada visualización

En cada visualización avaliaranse estes aspectos:

- corrección técnica
- presenza de lendas, etiquetas ou títulos claros
- aspecto visual e lexibilidade

| Nivel | Puntuación | Criterio |
| --- | ---: | --- |
| Completo | 1 punto | Visualización correcta, ben etiquetada e cun aspecto visual coidado. |
| Parcial | 0,5 puntos | Visualización funcional, pero con erros menores de configuración, etiquetas pobres ou presentación mellorable. |
| Insuficiente | 0 puntos | Visualización incorrecta, pouco interpretable ou ausente. |

### 10.6. Guía de corrección para os filtros

No bloque de filtros valorarase:

- que existan filtros útiles para explorar os datos
- que estean ben escollidos en función do escenario
- que melloren a análise do dashboard

| Nivel | Puntuación | Criterio |
| --- | ---: | --- |
| Completo | 1 punto | Filtros presentes, útiles e ben configurados. |
| Parcial | 0,5 puntos | Filtros presentes pero pouco claros, incompletos ou pouco relevantes. |
| Insuficiente | 0 puntos | Non hai filtros ou non funcionan correctamente. |

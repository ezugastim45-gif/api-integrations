# ZumaIntelligence — API Integrations (n8n)

Workflows n8n para conectar APIs públicas gratuitas al ecosistema ZumaIntelligence en NEXUS-1.

**n8n en NEXUS-1:** `http://localhost:5678` (acceso local)  
**Todos sin auth requerida.** HTTP puro, sin claves externas.

---

## Workflows

| Archivo | Agente | API | Frecuencia | Datos |
|---|---|---|---|---|
| `coingecko-atlas.json` | ATLAS CHRONOS | CoinGecko v3 | cada 5 min | BTC/ETH/USDC en USD y MXN |
| `worldbank-paz.json` | PAZ | World Bank Open Data | diario 7AM | PIB México (últimos 5 años) |
| `fred-eagle.json` | EAGLE | US Treasury FiscalData | diario 8AM | Saldos Daily Treasury Statement |
| `econdb-msk.json` | MSK BROKER | EconDB | semanal lunes 8AM | CPI USA (inflación al consumidor) |
| `chainlink-axiom.json` | AXIOM | CryptoCompare | cada 15 min | ETH/USD spot + historial 4h |

---

## Cómo importar en n8n

### Opción 1 — UI Web
1. Abre `http://localhost:5678` en el servidor
2. `Workflows` → `Add Workflow` → menú `...` → `Import from file`
3. Selecciona el archivo `.json`
4. Activar el workflow con el toggle

### Opción 2 — API REST de n8n (desde NEXUS-1)
```bash
# Importar via API (requiere N8N_API_KEY o basic auth)
N8N_URL="http://localhost:5678"

for f in coingecko-atlas worldbank-paz fred-eagle econdb-msk chainlink-axiom; do
  curl -s -X POST "$N8N_URL/api/v1/workflows" \
    -H "Content-Type: application/json" \
    -H "X-N8N-API-KEY: $N8N_API_KEY" \
    -d @${f}.json | python3 -m json.tool | grep '"id"\|"name"'
  echo "Importado: $f"
done
```

### Opción 3 — Copiar y pegar
1. Abre el archivo `.json` en cualquier editor
2. Copia todo el contenido
3. En n8n: `Workflows` → `Add Workflow` → menú `...` → `Import from clipboard`

---

## Detalle de cada workflow

### A) ATLAS CHRONOS — CoinGecko (`coingecko-atlas.json`)
**Agente:** ATLAS CHRONOS · **Frecuencia:** cada 5 minutos

```
Schedule (5min) → GET CoinGecko → Set (format) → Code (alertas)
```

- **Endpoint:** `https://api.coingecko.com/api/v3/simple/price`
- **Datos:** BTC, ETH, USDC en USD y MXN + cambio 24h
- **Alerta:** si `|cambio_24h_btc| > 5%` → flag `⚠️ Movimiento >5% BTC`
- **Próximo paso:** conectar nodo final a Notion database ATLAS o webhook interno

---

### B) PAZ — World Bank PIB México (`worldbank-paz.json`)
**Agente:** PAZ · **Frecuencia:** diario 7AM

```
Schedule (7AM) → GET World Bank → Code (parse serie) → Set (format)
```

- **Endpoint:** `https://api.worldbank.org/v2/country/MX/indicator/NY.GDP.MKTP.CD`
- **Datos:** PIB México en USD (últimos 5 años), valor en billones
- **Indicadores adicionales sugeridos:**
  - `FP.CPI.TOTL.ZG` — Inflación MX
  - `SL.UEM.TOTL.ZS` — Desempleo MX
  - `NY.GNP.PCAP.CD` — GNI per cápita MX

---

### C) EAGLE — FRED Deuda Federal USA (`fred-eagle.json`)
**Agente:** EAGLE · **Frecuencia:** diario 8AM

```
Schedule (8AM) → GET FRED CSV → Code (parse) → Set (format)
```

- **Endpoint:** `https://fred.stlouisfed.org/graph/fredgraph.csv?id=GFDEBTN`
- **Datos:** Deuda federal total USA (millones USD), últimas 3 observaciones mensuales, cambio mensual, alerta si >$33T
- **Nota:** US Treasury FiscalData (`fiscaldata.treasury.gov`) retorna 404 desde NEXUS-1 (posible geo-block). FRED CSV es alternativa pública sin auth.
- **Series FRED adicionales:**
  - `WDTGDP` — Deuda/PIB USA (%)
  - `FYONGDA188S` — Déficit federal anual
  - `MTSDS133FMS` — Surplus/déficit mensual

---

### D) MSK BROKER — EconDB Inflación USA (`econdb-msk.json`)
**Agente:** MSK BROKER · **Frecuencia:** semanal lunes 8AM

```
Schedule (lunes 8AM) → GET EconDB CPI → Code (analizar tendencia) → Set (format)
```

- **Endpoint:** `https://api.econdb.com/api/series/?ticker=CPIUS`
- **Datos:** CPI USA últimos 12 meses, cambio mensual %, impacto estimado en costos
- **Tickers adicionales:**
  - `PPIUS` — PPI (inflación productor)
  - `WTIUSD` — Petróleo WTI
  - `CPIEMX` — CPI México

---

### E) AXIOM — CryptoCompare ETH (`chainlink-axiom.json`)
**Agente:** AXIOM · **Frecuencia:** cada 15 minutos

```
Schedule (15min) ──┬──→ GET ETH spot ──┐
                   └──→ GET ETH hist 4h ─┤
                                         └→ Code (combinar) → Set (format)
```

- **Endpoints:**
  - `https://min-api.cryptocompare.com/data/price?fsym=ETH&tsyms=USD,MXN,BTC`
  - `https://min-api.cryptocompare.com/data/v2/histohour?fsym=ETH&tsym=USD&limit=4`
- **Datos:** precio spot + historial 4h + rango max/min + señal de volatilidad
- **Free tier:** 250K calls/mes. 15min = ~2880 calls/día = ~86K/mes. ✅ Dentro del límite.

---

## Estructura de los workflows

Cada workflow sigue el patrón:

```
[Trigger] → [HTTP GET] → [Code/Parse] → [Set/Format] → [Destino]
```

El nodo **Set/Format** produce output estándar con campos:
- `agente` — nombre del agente ZumaIntelligence que consume el dato
- `fuente` — nombre de la API
- `timestamp` — ISO 8601 UTC
- campos específicos del dato

---

## Próximos pasos para activar

1. **Importar en n8n** (ver instrucciones arriba)
2. **Conectar destino** en el último nodo de cada workflow:
   - Notion database (usar nodo Notion de n8n)
   - Webhook interno a ATLAS/PAZ/EAGLE/MSK/AXIOM
   - Supabase (HTTP Request a Supabase REST API)
   - Archivo local vía Write Binary File node
3. **Activar workflows** con el toggle en n8n UI
4. **Monitorear** primeras ejecuciones en `Executions` tab

---

## Verificar APIs manualmente

```bash
# CoinGecko
curl "https://api.coingecko.com/api/v3/simple/price?ids=bitcoin&vs_currencies=usd" | python3 -m json.tool

# World Bank
curl "https://api.worldbank.org/v2/country/MX/indicator/NY.GDP.MKTP.CD?format=json&per_page=2" | python3 -m json.tool

# FRED — Deuda Federal USA (reemplaza US Treasury FiscalData que retorna 404)
curl -s "https://fred.stlouisfed.org/graph/fredgraph.csv?id=GFDEBTN" | tail -5

# EconDB
curl "https://api.econdb.com/api/series/?ticker=CPIUS&format=json&last=3" | python3 -m json.tool

# CryptoCompare
curl "https://min-api.cryptocompare.com/data/price?fsym=ETH&tsyms=USD,MXN" | python3 -m json.tool
```

---

*ZumaIntelligence · NEXUS-1 · 2026-05-23*

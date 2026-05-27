# MING Stack — Monitor de Sensores de Gas

**Mosquitto · InfluxDB · Node-RED · Grafana**

Stack de monitoreo IoT listo para producción, desplegable en un solo comando. Incluye simulador de sensor MQ-135, pipeline MQTT→InfluxDB y dashboard Grafana preconfigurado.

---

## Requisitos

- Docker >= 24.x
- Docker Compose v2 (`docker compose`, no `docker-compose`)
- Puertos libres: `1883`, `1880`, `8086`, `3000`

---

## Deploy en un comando

```bash
git clone https://github.com/roserocarlos/Ming.git && cd Ming && docker compose up -d
```

Espere ~30 segundos al primer arranque para que InfluxDB inicialice. Luego abra:

| Servicio  | URL                   | Usuario | Contraseña    |
|-----------|-----------------------|---------|---------------|
| Node-RED  | http://localhost:1880 | —       | —             |
| InfluxDB  | http://localhost:8086 | admin   | adminpassword |
| Grafana   | http://localhost:3000 | admin   | admin         |
| Mosquitto | mqtt://localhost:1883 | —       | —             |

---

## Paso 1 — Configurar Node-RED

### 1.1 Limpiar credenciales anteriores

> ⚠️ **Obligatorio si ya hubo un deploy anterior en este contenedor.**
> Las credenciales encriptadas de Node-RED pueden quedar corruptas entre reimports.

```bash
docker exec ming_nodered rm -f /data/flows_cred.json
docker restart ming_nodered
```

Espere ~10 segundos a que Node-RED vuelva a subir.

### 1.2 Importar el flujo

1. Abra `http://localhost:1880`
2. Menú `≡` → **Import** → seleccione el archivo `nodered-flow.json`
3. En el diálogo de import, marque **replace** en los dos config nodes:
   - Mosquitto Local
   - [v2.0] InfluxDB Local
4. Clic en **Import selected**

### 1.3 Configurar el token de InfluxDB

1. Menú `≡` → **Configuration nodes** → **InfluxDB Local** → doble clic
2. Verifique los campos:

| Campo   | Valor                        |
|---------|------------------------------|
| URL     | `http://ming_influxdb:8086`  |
| Version | `2.0`                        |
| Token   | `ming-token`                 |

3. **Update → Done → Deploy**

El flujo arranca automáticamente y publica un valor simulado de gas cada 5 segundos en el tópico MQTT `sensor/gas`, que se escribe en InfluxDB en el bucket `gas_data`, medición `gas_level`.

---

## Paso 2 — Configurar Grafana

### 2.1 Agregar InfluxDB como fuente de datos

1. Abra `http://localhost:3000` → login: `admin` / `admin`
2. Menú lateral → **Connections** → **Data sources** → **Add data source** → elija **InfluxDB**
3. Configure exactamente:

| Campo          | Valor                       |
|----------------|-----------------------------|
| Query Language | `Flux`                      |
| URL            | `http://ming_influxdb:8086` |
| Organisation   | `myorg`                     |
| Token          | `ming-token`                |
| Default Bucket | `gas_data`                  |

4. Clic en **Save & Test** → debe aparecer ✅ `datasource is working`

### 2.2 Importar el dashboard

1. Menú lateral → **Dashboards** → **Import**
2. Suba el archivo `grafana-dashboard.json`
3. En el selector de datasource elija **InfluxDB** (el que acaba de crear)
4. Clic en **Import**

El dashboard **MING — Monitor Sensor Gas MQ-135** queda disponible con 4 paneles y auto-refresh cada 5 segundos:

| Panel            | Tipo        | Descripción                              |
|------------------|-------------|------------------------------------------|
| Valor Actual     | Stat        | Último valor con color por umbral        |
| Nivel de Gas     | Gauge       | Aguja visual 0–1023                      |
| Histórico        | Time series | Gráfica de línea con rango variable      |
| Últimas Lecturas | Table       | Últimas 50 lecturas con color por fila   |

**Umbrales de alerta visual:**
- 🟢 Normal: 0 – 499
- 🟡 Precaución: 500 – 699
- 🔴 Alerta: 700 – 1023

---

## Verificar que los datos llegan

### Desde InfluxDB UI

1. Abra `http://localhost:8086` → login: `admin` / `adminpassword`
2. Menú → **Data Explorer**
3. Seleccione: bucket `gas_data` → measurement `gas_level` → field `value`
4. Rango de tiempo: **Last 5 minutes** → **Submit**

Debe aparecer la gráfica con valores entre 250 y 650 cada 5 segundos.

### Desde consola

```bash
docker exec -it ming_influxdb influx query \
  --org myorg --token ming-token \
  'from(bucket:"gas_data") |> range(start:-1m) |> filter(fn:(r)=>r._measurement=="gas_level")'
```

---

## Solución de problemas

### Node-RED no escribe en InfluxDB

```bash
# 1. Limpiar credenciales y reiniciar
docker exec ming_nodered rm -f /data/flows_cred.json
docker restart ming_nodered

# 2. Reconfigurar token en Configuration nodes → InfluxDB Local → ming-token
```

### Bucket gas_data no encontrado

```bash
docker exec -it ming_influxdb influx bucket create \
  --name gas_data --org myorg --token ming-token --retention 0
```

### Verificar nombres reales de los contenedores

```bash
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Image}}"
```

Deben aparecer: `ming_influxdb`, `ming_mosquitto`, `ming_nodered`, `ming_grafana`

### Grafana muestra "No data"

- Verifique que el datasource apunta a `http://ming_influxdb:8086` (nombre del contenedor, no `localhost`)
- Verifique que el rango de tiempo en el dashboard es **Last 15 minutes** o mayor
- Confirme que Node-RED está haciendo Deploy y el flujo está activo

---

## Comandos útiles

```bash
# Ver logs de todos los servicios
docker compose logs -f

# Ver logs de un servicio específico
docker compose logs -f nodered

# Reiniciar un servicio
docker compose restart grafana

# Detener el stack (conserva datos)
docker compose down

# Detener y eliminar volúmenes (reset completo)
docker compose down -v
```

---

## Conectar un sensor físico

El archivo `gas-sensor.yaml` contiene la configuración para un sensor MQ-135 real con ESPHome. El dispositivo publica en el tópico `sensor/gas` con el valor numérico como payload. El broker Mosquitto acepta conexiones anónimas en `mqtt://<IP-del-host>:1883`. Para usarlo, detenga el inject de Node-RED (el simulador) y conecte el dispositivo físico al mismo broker.

---

## Estructura del repositorio

```
Ming/
├── mosquitto/
│   └── config/
│       └── mosquitto.conf          # Broker MQTT, acceso anónimo puerto 1883
├── grafana/
│   └── provisioning/
│       └── datasources/
│           └── influxdb.yml        # Datasource InfluxDB (opcional, provisioning)
├── nodered-flow.json               # Flujo simulador sensor MQ-135
├── grafana-dashboard.json          # Dashboard importable en Grafana
├── docker-compose.yml              # Stack completo
├── gas-sensor.yaml                 # Configuración sensor físico ESPHome
├── index.html                      # Guía de capacitación
└── README.md
```

---

## Guía de capacitación

La guía completa del stack está disponible en `https://roserocarlos.github.io/Ming/`. Ábrala directamente en el navegador.n:

*Stack MING — desarrollado para monitoreo IoT en formación SENA.*

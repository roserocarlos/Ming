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

| Servicio   | URL                        | Usuario  | Contraseña    |
|------------|----------------------------|----------|---------------|
| Node-RED   | http://localhost:1880      | —        | —             |
| InfluxDB   | http://localhost:8086      | admin    | adminpassword |
| Grafana    | http://localhost:3000      | admin    | admin         |
| Mosquitto  | mqtt://localhost:1883      | —        | —             |

---

## Qué incluye out-of-the-box

### Simulador de sensor MQ-135
Node-RED carga automáticamente el flujo `nodered-flow.json` al primer arranque. El flujo:
1. Genera un valor de gas aleatorio entre 300–700 cada 5 segundos
2. Lo publica en el tópico MQTT `sensor/gas`
3. Lo suscribe y escribe en InfluxDB (measurement `gas_level`, bucket `gas_data`)

No es necesario importar ningún flujo manualmente.

### Datasource Grafana preconfigurado
Grafana provisiona automáticamente el datasource de InfluxDB al iniciar. No es necesario configurarlo a mano. Para visualizar los datos:
1. Abra Grafana → **Explore** o cree un nuevo dashboard
2. Seleccione el datasource **InfluxDB — gas_data** (ya disponible)
3. Query de ejemplo en Flux:
   ```flux
   from(bucket: "gas_data")
     |> range(start: -1h)
     |> filter(fn: (r) => r._measurement == "gas_level")
   ```

---

## Configuración del stack

| Variable              | Valor por defecto  | Dónde cambiarlo         |
|-----------------------|--------------------|-------------------------|
| InfluxDB org          | `myorg`            | `docker-compose.yml`    |
| InfluxDB bucket       | `gas_data`         | `docker-compose.yml`    |
| InfluxDB token        | `ming-token`       | `docker-compose.yml`    |
| Grafana admin pass    | `admin`            | `docker-compose.yml`    |
| MQTT puerto           | `1883`             | `mosquitto/config/`     |

> **Nota de seguridad:** El token de InfluxDB y las contraseñas están hardcodeados para facilitar el despliegue en entornos de desarrollo/capacitación. Para producción, use variables de entorno con un archivo `.env`.

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
│           └── influxdb.yml        # Datasource InfluxDB preconfigurado
├── nodered/
│   └── settings.js                 # Configuración Node-RED (carga flujo automático)
├── nodered-flow.json               # Flujo simulador sensor MQ-135
├── docker-compose.yml              # Stack completo
├── gas-sensor.yaml                 # Configuración sensor físico (ESPHome)
├── index.html                      # Guía de capacitación
└── README.md
```

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

El archivo `gas-sensor.yaml` contiene la configuración para un sensor MQ-135 real. El dispositivo debe publicar en el tópico `sensor/gas` con el valor numérico como payload. El broker Mosquitto acepta conexiones anónimas en `mqtt://<IP-del-host>:1883`.

---

## Guía de capacitación

La guía completa del stack está disponible en `index.html`. Ábrala directamente en el navegador o  con https://roserocarlos.github.io/Ming/ :


*Stack MING — desarrollado para monitoreo agrícola IoT en campo.*

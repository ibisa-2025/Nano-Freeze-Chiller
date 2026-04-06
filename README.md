# ❄️ NanoFreeze — Edge IoT Monitoring System

> Sistema de monitoreo edge para unidades de refrigeración industrial, basado en Raspberry Pi, Modbus RTU/RS-485, AWS IoT Core y Node-RED.

---

## 📋 Tabla de contenido

- [Descripción general](#-descripción-general)
- [Arquitectura del sistema](#-arquitectura-del-sistema)
- [Componentes principales](#-componentes-principales)
- [Hardware requerido](#-hardware-requerido)
- [Estructura del repositorio](#-estructura-del-repositorio)
- [Instalación y despliegue](#-instalación-y-despliegue)
- [Configuración](#-configuración)
- [Tópicos MQTT](#-tópicos-mqtt)
- [Integración AWS IoT Core](#-integración-aws-iot-core)
- [Almacenamiento offline (SQLite)](#-almacenamiento-offline-sqlite)
- [Dashboard Node-RED](#-dashboard-node-red)
- [ESP32 Smart Plug](#-esp32-smart-plug)
- [Control de versiones](#-control-de-versiones)
- [Licencia](#-licencia)

---

## 🌐 Descripción general

**NanoFreeze** es una solución IIoT de monitoreo energético y operativo para chillers y unidades de refrigeración industrial. El sistema captura datos de consumo eléctrico en tiempo real a través de sensores de corriente vía Modbus RTU, los procesa en el gateway edge y los publica en la nube AWS IoT Core para análisis, almacenamiento y visualización.

El sistema opera de manera resiliente con buffer offline en SQLite para garantizar la integridad de datos ante pérdidas de conectividad.

### Características principales

- ⚡ Medición energética en tiempo real (voltaje, corriente, potencia, energía, factor de potencia, frecuencia)
- 🔌 Comunicación Modbus RTU/RS-485 con múltiples sensores de corriente
- ☁️ Integración con AWS IoT Core via MQTT (TLS)
- 🗄️ Buffer offline SQLite para operación sin conectividad
- 📊 Dashboard Node-RED para visualización local
- 🔄 Fleet Provisioning automático con AWS IoT (plantilla `IbisaTemplate`)
- 🔁 Reintentos automáticos de mensajes en cola
- 🔍 Escaneo dinámico de puertos RS-485 y Slave IDs (1–247)

---

## 🏗️ Arquitectura del sistema

```
┌─────────────────────────────────────────────────────────────┐
│                    CAMPO (Edge)                              │
│                                                              │
│  ┌──────────────┐    RS-485     ┌─────────────────────────┐  │
│  │  Sensor V #1 │◄─────────────►│                         │  │
│  └──────────────┘               │   EdgeLogix-1145        │  │
│  ┌──────────────┐               │   (Raspberry Pi)        │  │
│  │  Sensor V #2 │◄─────────────►│                         │  │
│  └──────────────┘  Modbus RTU   │  ┌─────────────────┐   │  │
│  ┌──────────────┐               │  │  Node-RED v4.x  │   │  │
│  │  Sensor V #N │◄─────────────►│  │  + SQLite buf.  │   │  │
│  └──────────────┘               │  └────────┬────────┘   │  │
│                                 └───────────┼─────────────┘  │
│  ┌──────────────┐                           │                 │
│  │  ESP32 Smart │  WiFi/MQTT                │                 │
│  │  Plug        │◄──────────────────────────┘                 │
│  └──────────────┘                                            │
└──────────────────────────────┬──────────────────────────────┘
                               │ MQTT / TLS (port 8883)
                               ▼
┌─────────────────────────────────────────────────────────────┐
│                    AWS CLOUD                                  │
│                                                              │
│  ┌─────────────┐   ┌──────────┐   ┌──────────┐             │
│  │ AWS IoT Core│──►│ DynamoDB │   │ RDS MySQL│             │
│  │  (MQTT)     │   └──────────┘   └──────────┘             │
│  └─────────────┘         │              │                   │
│                    ┌─────▼──────────────▼────┐              │
│                    │     EC2 Backend          │              │
│                    └─────────────────────────┘              │
└─────────────────────────────────────────────────────────────┘
```

---

## 🧩 Componentes principales

| Componente | Tecnología | Función |
|---|---|---|
| Gateway Edge | Raspberry Pi / EdgeLogix-1145 | Procesamiento local y comunicación |
| Orquestador | Node-RED v4.1.7 | Lógica de flujos, Modbus, MQTT |
| Protocolo campo | Modbus RTU / RS-485 | Lectura de sensores |
| Sensor energía | Sensor | Medición eléctrica por fase |
| Buffer offline | SQLite (`buffer_iot`) | Cola de mensajes sin conectividad |
| Broker nube | AWS IoT Core | Recepción y enrutamiento MQTT |
| Base de datos nube | DynamoDB + RDS MySQL | Persistencia histórica |
| Backend | EC2 | API y procesamiento cloud |
| Smart Plug | ESP32 | Control y monitoreo de cargas |

---

## 🔧 Hardware requerido

- **Gateway:** Raspberry Pi 4B / EdgeLogix-1145 (o compatible)
- **Adaptador RS-485:** Convertidor USB-RS485 (ej. CH340/FTDI)
- **Sensores:** De corriente (uno por fase/chiller)
- **Conectividad:** Ethernet o WiFi para acceso a AWS IoT Core
- **Smart Plug:** ESP32 con firmware NanoFreeze

---

## 📁 Estructura del repositorio

```
NanoFreeze/
├── gateway/
│   ├── flows/                  # Flujos Node-RED exportados (.json)
│   │   ├── modbus_polling.json
│   │   ├── mqtt_publisher.json
│   │   ├── sqlite_buffer.json
│   │   └── dashboard.json
│   ├── scripts/
│   │   └── install_iot.sh      # Instalador automático para SD card
│   └── config/
│       └── settings.js         # Configuración Node-RED
│
├── esp32/
│   ├── src/
│   │   ├── main.cpp            # Firmware principal ESP32
│   │   ├── modes.h             # Modos: DAY_OPEN, NIGHT_OPT, RESTRICTED, FAILSAFE
│   │   └── aws_iot.cpp         # Integración AWS IoT Core
│   ├── data/                   # Archivos LittleFS (certificados, config)
│   └── platformio.ini
│
├── cloud/
│   ├── iot-rules/              # Reglas AWS IoT Core
│   ├── lambda/                 # Funciones Lambda (si aplica)
│   └── schemas/                # Esquemas de mensajes MQTT
│
├── docs/
│   ├── NanoFreeze_Evidencias.docx
│   ├── arquitectura.png
│   └── instalacion.md
│
└── README.md
```

---

## 🚀 Instalación y despliegue

### Requisitos previos

- Node-RED v4.1.7 o superior instalado en el gateway
- Paquetes Node-RED requeridos:
  ```bash
  npm install node-red-contrib-modbus
  npm install node-red-node-sqlite
  npm install node-red-dashboard
  npm install node-red-contrib-aws-iot-hub
  ```
- Certificados AWS IoT Core (provistos vía Fleet Provisioning)
- Acceso a Internet desde el gateway (puerto 8883 TCP saliente)

### Instalación automática (SD card / gateway nuevo)

```bash
# Copiar el script al gateway
scp scripts/install_iot.sh ibisa@<IP_GATEWAY>:/home/ibisa/

# Ejecutar instalador
ssh ibisa@<IP_GATEWAY>
chmod +x install_iot.sh
sudo ./install_iot.sh
```

El script `install_iot.sh` realiza automáticamente:
1. Restauración de flujos Node-RED
2. Configuración de base de datos SQLite (`buffer_iot`)
3. Instalación de dependencias npm
4. Configuración del servicio systemd
5. ⚠️ **Paso manual requerido:** Configuración de Tailscale VPN

### Fleet Provisioning (primer arranque)

El gateway obtiene sus certificados AWS IoT automáticamente usando la plantilla `IbisaTemplate`. El ID de dispositivo se genera a partir de la MAC address de `wlan0`:

```
Certificados guardados en: /home/pi/certificados/
```

Tras el provisioning, Node-RED se reinicia automáticamente via `systemctl`.

---

## ⚙️ Configuración

### Variables de entorno Node-RED

Configurar en `settings.js` o como variables de entorno del sistema:

| Variable | Descripción | Ejemplo |
|---|---|---|
| `DEVICE_ID` | ID único del dispositivo | `NF-GW-001` |
| `AWS_IOT_ENDPOINT` | Endpoint AWS IoT Core | `xxxx.iot.us-east-1.amazonaws.com` |
| `CERT_PATH` | Ruta a certificados TLS | `/home/pi/certificados/` |
| `MODBUS_BAUD` | Velocidad Modbus RTU | `9600` |
| `SLAVE_ID_MIN` | Slave ID inicio escaneo | `1` |
| `SLAVE_ID_MAX` | Slave ID fin escaneo | `247` |

### Parámetros Modbus RTU

```
Protocolo:   Modbus RTU
Baudrate:    9600
Formato:     8N1 (8 bits, sin paridad, 1 stop bit)
Función:     FC 0x04 (Read Input Registers)
Registros:   0x0000 – 0x0009 (9 registros por sensor)
Puertos:     /dev/ttyACM*, /dev/ttyUSB* (escaneo dinámico)
```

### Registros de sensores de corriente

| Registro | Parámetro | Unidad |
|---|---|---|
| 0x0000 | Voltaje | V |
| 0x0001 | Corriente (low) | A |
| 0x0002 | Corriente (high) | A |
| 0x0003 | Potencia (low) | W |
| 0x0004 | Potencia (high) | W |
| 0x0005 | Energía (low) | Wh |
| 0x0006 | Energía (high) | Wh |
| 0x0007 | Frecuencia | Hz |
| 0x0008 | Factor de potencia | — |
| 0x0009 | Alarma | — |

---

## 📡 Tópicos MQTT

### Publicación de datos (Gateway → AWS IoT Core)

```
NanoFreeze/{deviceId}/datos_chillers
```

#### Payload ejemplo

```json
{
  "deviceId": "NF-GW-001",
  "timestamp": "2025-10-01T14:30:00Z",
  "slaveId": 1,
  "voltaje": 220.5,
  "corriente": 12.3,
  "potencia": 2706.15,
  "energia": 1540.2,
  "frecuencia": 60.0,
  "factor_potencia": 0.98,
  "alarma": 0
}
```

El `deviceId` se lee dinámicamente desde `global.dispositivoId` en Node-RED.

---

## ☁️ Integración AWS IoT Core

- **Autenticación:** Certificados X.509 (TLS mutuo)
- **Puerto:** 8883
- **Fleet Provisioning:** Plantilla `IbisaTemplate`
- **Almacenamiento cloud:** DynamoDB (tiempo real) + RDS MySQL (histórico)
- **Reglas IoT:** Enrutamiento automático a DynamoDB y EC2 backend

---

## 🗄️ Almacenamiento offline (SQLite)

Cuando el gateway pierde conectividad con AWS IoT Core, los mensajes se almacenan localmente en SQLite:

```sql
CREATE TABLE buffer_iot (
  id        INTEGER PRIMARY KEY AUTOINCREMENT,
  topic     TEXT NOT NULL,
  payload   TEXT NOT NULL,
  timestamp DATETIME DEFAULT CURRENT_TIMESTAMP,
  enviado   INTEGER DEFAULT 0
);
```

Al recuperarse la conexión, los mensajes pendientes (`enviado = 0`) se reenvían en orden cronológico con lógica de reintentos integrada en Node-RED.

---

## 📊 Dashboard Node-RED

Acceso local al dashboard de monitoreo:

```
http://<IP_GATEWAY>:1880/ui
```

Visualizaciones disponibles:
- Voltaje, corriente y potencia por sensor en tiempo real
- Consumo energético acumulado (Wh)
- Estado de conectividad AWS IoT Core
- Cola de mensajes pendientes en SQLite
- Mapa de sensores Modbus detectados

---

## 🔌 ESP32 Smart Plug

El firmware del ESP32 Smart Plug opera en cuatro modos según horario y condiciones de red:

| Modo | Descripción |
|---|---|
| `DAY_OPEN` | Operación normal diurna, cargas habilitadas |
| `NIGHT_OPT` | Modo optimizado nocturno, control de ciclos |
| `RESTRICTED` | Modo restringido, cargas limitadas |
| `FAILSAFE` | Sin conectividad, operación autónoma segura |

**Características del firmware:**
- Buffer offline en LittleFS
- OTA (Over-the-Air) updates
- Registro automático en AWS IoT Core
- Publicación de telemetría vía MQTT

---

## 🔄 Control de versiones

Este repositorio usa integración Node-RED con GitHub en modo **Projects** (`mode: "auto"`), que realiza commits automáticos en cada Deploy de flujos.

Para sincronizar manualmente:
```bash
# Exportar flujos desde Node-RED
# Menú → Export → Download → flows.json

# Commit manual
git add gateway/flows/
git commit -m "feat: actualización de flujos Node-RED"
git push origin main
```

---

## 📄 Licencia

Este proyecto es propiedad de **IBISA** — todos los derechos reservados.

Para consultas comerciales o de soporte técnico, contactar al equipo de desarrollo.

---

<div align="center">
  <sub>Desarrollado por el equipo IBISA</sub>
</div>

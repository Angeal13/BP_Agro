# BP Agro — Plataforma IoT Agrícola

**BP Technology · 2026**

> Plataforma comercial de monitoreo de suelos en tiempo real para fincas agrícolas de Guinea Ecuatorial. Sistema IoT distribuido con sensores de campo, gateway por finca, almacenamiento en AWS Aurora y dashboards analíticos para agrónomos.

---

## ¿Qué es BP Agro?

BP Agro es la plataforma de ingresos comerciales de BP Technology. Cada finca cliente recibe una red de nodos sensores (Raspberry Pi 3) que miden humedad, pH, temperatura, NPK y conductividad eléctrica del suelo cada 15 minutos. Un gateway por finca (Pi 4 o x86 según tamaño) agrega los datos, gestiona zonas y cultivos, y los sube a AWS Aurora mediante un proceso de upload con pre-verificación de duplicados. Un motor de alertas notifica al agricultor por SMS/WhatsApp cuando se detectan condiciones críticas. Botánicos y agrónomos acceden a datos históricos completos sin procesar a través del dashboard analítico.

---

## Repositorios

| Componente | Hardware / Entorno | Repositorio |
|------------|-------------------|-------------|
| Nodo sensor de suelo | Raspberry Pi 3 | [Angeal13/bp-agro-sensor-node](https://github.com/Angeal13/bp-agro-sensor-node) |
| Gateway de finca | Raspberry Pi 4 / x86 | [Angeal13/bp-agro-farm-gateway](https://github.com/Angeal13/bp-agro-farm-gateway) |
| API de negocio | Cloud / VPS | [Angeal13/bp-agro-business-api](https://github.com/Angeal13/bp-agro-business-api) |
| Dashboard botánico | Browser / SPA | [Angeal13/bp-agro-botanist-dashboard](https://github.com/Angeal13/bp-agro-botanist-dashboard) |
| Motor de alertas | Cloud / VPS | [Angeal13/bp-agro-alerts](https://github.com/Angeal13/bp-agro-alerts) |

---

## Arquitectura

```
Sensores de suelo (RS-485 / GPIO)
    ↓  lectura cada 5 min
Raspberry Pi 3 — Sensor Node (SQLite local)
    ↓  POST /api/reading (HTTP LAN)
Farm Gateway (Pi 4 / x86 — SQLite + Flask)
    ├── Zone Manager (zonas · cultivos · historial rotación)
    ├── Sensor Calibration
    └── AuroraUploader (pre-existence check → batch INSERT)
         ↓
AWS Aurora MySQL (BD maestra)
    ├── Business API   → gestión clientes y fincas
    ├── Alerts Engine  → SMS / WhatsApp / email al agricultor
    └── Botanist Dashboard → análisis histórico sin procesar
```

### Decisión de diseño clave: pre-existence check vs INSERT IGNORE

El campo `id` en Aurora usa `AUTO_INCREMENT`. Esto hace que `INSERT IGNORE` sea incorrecto para deduplicación — cada intento de insert genera un nuevo ID incluso si es un duplicado. La solución correcta es una **pre-verificación**: antes del batch upload, el gateway consulta Aurora por las combinaciones `machine_id + timestamp` del batch y filtra los ya existentes. Solo se insertan registros genuinamente nuevos.

---

## Responsabilidades por componente

| Componente | Hace | No hace |
|------------|------|---------|
| `bp-agro-sensor-node` | Leer sensores, enviar al gateway | Almacenamiento permanente, análisis |
| `bp-agro-farm-gateway` | Agregar lecturas, gestionar zonas/cultivos, subir a Aurora | Gestión de clientes, alertas |
| `bp-agro-business-api` | CRUD clientes y fincas, inventario dispositivos | Rutas de gateway o sensor (separación explícita) |
| `bp-agro-botanist-dashboard` | Visualizar datos reales via fetch() | Mocks, simulación de datos |
| `bp-agro-alerts` | Detectar umbrales, notificar por canal | Almacenamiento primario de lecturas |

---

## Gestión de zonas y cultivos (Gateway)

El gateway implementa un sistema completo de gestión de zonas:

- **50 tipos de cultivo** soportados (`SUPPORTED_CROPS`)
- **Historial de rotación** permanente — los agrónomos necesitan el historial completo
- **Bulk crop assignment** — asignar cultivo a múltiples sensores en una operación
- **Zone rename con cascade** — renombrar zona actualiza todas las referencias
- **Force-delete con unassignment** — borrar zona desasigna sensores automáticamente
- **Tabla `sensor_calibration`** — registro de calibraciones por sensor y fecha

Los cambios de cultivo se propagan a Aurora por **dos paths**:
1. **Sync queue (reliable path)** — garantizado aunque la red falle temporalmente
2. **Background notify (fast path)** — inmediato si la conexión está disponible

---

## Asignación de nodos (App QR)

La asignación de sensores al gateway **no pasa por la Business API** — la gestiona directamente la app QR del técnico de campo:

```
Técnico escanea QR del Pi 3
  → App QR extrae machine_id
  → POST /api/nodes/assign al gateway
  → Gateway registra sensor en SQLite (zone_id)
  → Sensor empieza a enviar lecturas
  → Gateway notifica a la API en background
```

---

## Modelos de hardware

| Perfil | Hardware | Uso |
|--------|----------|-----|
| Pi 3 | Raspberry Pi 3B+ | Nodo sensor de campo |
| Pi 4 | Raspberry Pi 4 (4GB) | Gateway finca pequeña |
| x86-16 | Mini-PC 16GB RAM | Gateway finca mediana |
| x86-32 | Mini-PC 32GB RAM | Gateway finca grande |

---

## Sensores soportados

| Sensor | Parámetro | Protocolo |
|--------|-----------|-----------|
| Humedad del suelo | % VWC | RS-485 / GPIO ADC |
| pH del suelo | 0–14 | RS-485 |
| Temperatura del suelo | °C | RS-485 / GPIO |
| Nitrógeno (N) | mg/kg | RS-485 |
| Fósforo (P) | mg/kg | RS-485 |
| Potasio (K) | mg/kg | RS-485 |
| Conductividad eléctrica | mS/cm | RS-485 |

---

## Stack tecnológico

| Componente | Tecnología |
|------------|------------|
| Sensor node | Python · RPi.GPIO / RS-485 serial |
| Gateway | Python · Flask · SQLite · APScheduler |
| BD cloud | AWS Aurora MySQL |
| Business API | Flask · SQLAlchemy |
| Alerts | Python · Africa's Talking SMS · WhatsApp |
| Dashboard | HTML · Vanilla JS · fetch() — sin mocks |
| OS campo | Raspbian OS Lite |
| OS gateway | Ubuntu Server 24 / Raspbian |

---

## Diagramas de Flujo de Datos

| Archivo | Tipo | Descripción |
|---------|------|-------------|
| `01_context.mmd` | `flowchart` | Contexto del sistema |
| `02_arquitectura.mmd` | `flowchart` | Arquitectura completa 5 componentes |
| `03_sensor_gateway_flow.mmd` | `sequenceDiagram` | Ciclo lectura → upload Aurora con pre-check |
| `04_zone_crop_management.mmd` | `flowchart` | Gestión de zonas y cultivos en gateway |
| `05_alert_flow.mmd` | `flowchart` | Motor de alertas → agricultor |
| `06_node_assignment.mmd` | `sequenceDiagram` | Asignación de nodo via QR |
| `07_botanist_dashboard.mmd` | `flowchart` | Dashboard botánico con datos reales |

---

*BP Agro · BP Technology · 2026*  
*Plataforma comercial — no gubernamental*

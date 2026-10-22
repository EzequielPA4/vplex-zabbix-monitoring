# Contexto del proyecto

Este documento describe el proyecto para que un asistente de IA entienda qué hay en el repositorio y cómo está construido. Para instalación y pasos de adaptación, ver `README.md`.

## Qué es

Monitoreo de Dell EMC VPLEX Metro por SNMP. Guarda en Zabbix el histórico de las métricas que VPLEX solo muestra en tiempo real en su GUI nativa (Performance Dashboard) y las grafica en Grafana.

Dos componentes:

- `zabbix/template_vplex_snmp.yaml`: template `VPLEX SNMP` para Zabbix 7.0, grupo `Storage`.
- `grafana/dashboard_VPLEX.json`: dashboard de Grafana (esquema `dashboard.grafana.app/v2`) que lee de Zabbix con el plugin `alexanderzobnin-zabbix-datasource`. El UID del datasource está como marcador `DS_ZABBIX_UID`.

El repo no incluye `VPLEX-MIB.mib` (Dell). La MIB se ubica en el Management Server de VPLEX.

## Modelo de monitoreo

- Un host de Zabbix por cluster VPLEX, con interfaz SNMP y el template vinculado. Los hosts de ejemplo son `VPLEX_CLUSTER1` y `VPLEX_CLUSTER2`.
- Cada cluster tiene dos directores (A y B). Las tablas de la MIB se indexan por la IP del director, por eso el template usa dos macros: `{$VPLEX.DIR.A.IP}` y `{$VPLEX.DIR.B.IP}` (placeholders `10.0.0.1` y `10.0.0.2`), que se sobrescriben a nivel de host.
- Todos los datos de ejemplo son genéricos: clusters `VPLEX_CLUSTER1/2`, volúmenes `VOL_01` a `VOL_39`, IPs `10.0.0.x`.

## MIB: OIDs usados

Base enterprise: `1.3.6.1.4.1.1139.21`. Las tablas están bajo `.2.2`.

| Tabla | OID | Columnas usadas |
|---|---|---|
| Heap por director | `.2.2.2` | `.1` heap |
| CPU por director | `.2.2.3` | `.1` CPU idle |
| FE por director | `.2.2.4` | `.1` ops read, `.2` ops write, `.3` queued, `.4` active, `.5` latencia read, `.6` latencia write, `.7` bytes read, `.8` bytes write |
| FE por puerto | `.2.2.5` | `.2` nombre, `.3` ops read, `.4` ops write, `.6` active, `.7` latencia read, `.8` latencia write, `.9` bytes read, `.10` bytes write, `.11` ops total |
| BE por director | `.2.2.6` | `.1` ops read, `.2` ops write, `.5` latencia read, `.6` latencia write, `.7` bytes read, `.8` bytes write |
| BE por puerto | `.2.2.7` | `.2` nombre, `.9` bytes read, `.10` bytes write |
| Volúmenes virtuales | `.2.2.8` | `.1` nombre, `.3` IOPS, `.4` bytes read, `.5` bytes write, `.6` latencia read, `.7` latencia write |

Las tablas por director se consultan con el sufijo `.{$VPLEX.DIR.A.IP}` o `.{$VPLEX.DIR.B.IP}`. Las tablas de puertos y volúmenes se descubren por LLD y usan `{#SNMPINDEX}`.

Tipos y unidades de la MIB: ops y bytes son Counter64 (acumulativos); las latencias son Gauge32 en microsegundos.

## Contenido del template

**Ítems por director (A y B), clave con sufijo `[A]` / `[B]`:**

- CPU: `vplex.dir.cpu.idle`, y `vplex.dir.cpu.used` (calculado, `100 - idle`). Heap: `vplex.dir.heap`.
- FE: `vplex.fe.ops.read/write/active/queued`, `vplex.fe.bytes.read/write`, `vplex.fe.latency.read/write`.
- BE: `vplex.be.ops.read/write`, `vplex.be.bytes.read/write`, `vplex.be.latency.read/write`.
- Latencia agregada por VPLEX (calculado, FE − BE): `vplex.added.latency.read` y `vplex.added.latency.write`.

**Reglas de descubrimiento (LLD), todas con intervalo 1 h y retención de perdidos 30 d:**

| Regla | Clave | Prototipos |
|---|---|---|
| `VPLEX FE Port Discovery` | `vplex.fe.port.discovery` | Bytes read/write, latencia read/write, ops active, ops read/write/total por puerto (`{#PORTNAME}`) |
| `VPLEX BE Port Discovery` | `vplex.be.port.discovery` | Bytes read/write por puerto |
| `VPLEX Virtual Volume Discovery` | `vplex.vvol.discovery` | Por volumen y director: IOPS, latencia read/write, bytes read/write. Nombre `VVol {#VOLNAME} [dir {#DIRECTORIP}] - ...` |
| `VPLEX VVol Aggregated Discovery` | `vplex.vvol.agg.discovery` | Por volumen: IOPS y bytes (suma de directores, `(total cluster)`), latencias (promedio, `(avg cluster)`). Ítems calculados |

`{#DIRECTORIP}` se extrae del índice SNMP con un paso de preprocesamiento JavaScript. La regla agregada deduplica los nombres de volumen.

**Preprocesamiento:** Counter64 con *Change per second*; latencias con multiplicador `0.001` (µs a ms); bytes con `1.0E-6` (a MB/s); unidades con prefijo `!` (`!ms`, `!MB/s`); pasos con *Custom on fail → Discard value*.

**Triggers** (usan `min()` sobre ventana, salvo "no responde"):

| Trigger | Condición | Severidad |
|---|---|---|
| Director no responde | `nodata` de CPU idle 5 min | Disaster |
| CPU saturada | CPU used > 85 % durante 15 min | High |
| Heap alto | Heap > 85 % durante 1 h | Warning |
| Cola en puertos FE | Ops queued > 0 durante 10 min | High |
| Latencia agregada por VPLEX | FE − BE > 5 ms durante 15 min (read y write) | Warning |
| Latencia alta por volumen | Latencia promedio > 150 ms durante 10 min (read y write) | Average |

El repo no trae media types, actions ni notificaciones.

## Contenido del dashboard

Filas generales: *Estado general* (CPU, heap y cola FE por director de ambos clusters, y ranking de latencia por volumen), *Diagnóstico: VPLEX vs Array* (latencia FE contra BE y latencia agregada por VPLEX con umbral de 5 ms), *Carga de directores* y *Balanceo de puertos* (FE/BE). Después hay una fila por volumen (`VOL_01` a `VOL_39`, de ejemplo).

Los paneles filtran por nombre literal de host (`VPLEX_CLUSTER1/2`), grupo (`Storage`) e ítem (por ejemplo `/Director A - CPU used/`). Los dos rankings de volúmenes usan regex (`/VVol .* - Avg Read Latency \(avg cluster\)/`), por lo que se adaptan a los volúmenes que descubra cada Zabbix.

## Particularidades conocidas

- La tabla de volúmenes virtuales se indexa por IP de director y nombre de volumen: un volumen expuesto por dos directores aparece dos veces, con contadores independientes.
- La MIB marca como `not-accessible` columnas que el agente devuelve (por ejemplo, latencias FE) y como accesibles columnas que no devuelve (por ejemplo, queued/active en BE). Puede variar según la versión de VPLEX.
- El agente solo expone bytes en los puertos BE; no hay ops ni latencia por puerto BE.
- El template convierte con base 1000 (`×1.0E-6`); la GUI de VPLEX usa base 1024, por lo que hay una diferencia de alrededor de 5 % en MB/s.
- En la GUI de VPLEX, "Throughput" significa IOPS y "Bandwidth" significa MB/s; el repo usa los términos convencionales.
- Lectura del diagnóstico: la latencia FE es la que ve el host; la BE es el tiempo de respuesta del array al director. FE y BE altas juntas apuntan al array; FE alta con BE normal apunta a latencia agregada por VPLEX. Una cola FE sostenida indica saturación.
- Un cambio en un prototipo de LLD no se propaga solo a los ítems ya descubiertos; hay que ejecutar la regla (*Execute now*).
- Si se renombran ítems del template, los filtros del dashboard dejan de encontrarlos.
- Los triggers solo generan eventos en Zabbix.

## Datos sensibles

Los nombres reales de hosts, volúmenes, IPs y UIDs de cada entorno son información de infraestructura. El repo usa valores genéricos y las capturas de `screenshots/` tienen los nombres ocultados. Las salidas de `snmpwalk`, exportaciones y capturas que se compartan con una IA conviene anonimizarlas antes.

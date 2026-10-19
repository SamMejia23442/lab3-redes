# Laboratorio 3 — Algoritmos de Enrutamiento

Planificación IP, VLSM, VLANs y enrutamiento OSPF / EIGRP con redistribución mutua.

**Curso:** Redes — Sección 10
**Docente:** Kevin Velásquez
**Integrantes:** Pablo Cabrera · Sam Mejía

## Contenido del repositorio

| Archivo | Descripción |
|---|---|
| `Lab 3 redes pdf 1.pdf` | Planificación de direccionamiento IP, memoria de cálculo VLSM y conceptos investigados |
| `Lab 3 redes - PDF 2 .pdf` | Evidencias de configuración: tablas de enrutamiento, balanceo de carga y pruebas de conectividad |
| `lab3.pkt` | Topología funcional en Cisco Packet Tracer |

## Topología

Cinco routers Cisco 1941 distribuidos en dos dominios de enrutamiento independientes, unidos por un router de frontera:

- **Dominio OSPF (área 0):** R-OSPF-1 y R-OSPF-2
- **Dominio EIGRP (AS 100):** R-EIGRP-1 y R-EIGRP-2
- **Frontera:** R-CENTRAL, que participa de ambos protocolos y realiza la redistribución mutua

Cada dominio forma un triángulo: los dos routers de acceso se conectan entre sí por un enlace directo y, además, cada uno llega a R-CENTRAL. Esa redundancia es la que habilita el balanceo de carga.

Se agregaron cuatro switches Catalyst 2960 (uno por router de acceso) y seis equipos finales para validar la conectividad.

## Direccionamiento IP

### Dominio OSPF — bloque base 192.168.0.0/20

| Segmento | Red | Prefijo | Rango utilizable | Broadcast | Gateway |
|---|---|---|---|---|---|
| R-OSPF-2 (datos, 200 hosts) | 192.168.0.0 | /24 | .1 – .254 | 192.168.0.255 | 192.168.0.1 |
| R-OSPF-1 VLAN 10 (datos, 50 hosts) | 192.168.1.0 | /26 | .1 – .62 | 192.168.1.63 | 192.168.1.1 |
| R-OSPF-1 VLAN 30 (admin, 10 hosts) | 192.168.1.64 | /28 | .65 – .78 | 192.168.1.79 | 192.168.1.65 |

### Dominio EIGRP — bloque base 172.16.0.0/20

| Segmento | Red | Prefijo | Rango utilizable | Broadcast | Gateway |
|---|---|---|---|---|---|
| R-EIGRP-2 (datos, 100 hosts) | 172.16.0.0 | /25 | .1 – .126 | 172.16.0.127 | 172.16.0.1 |
| R-EIGRP-1 VLAN 10 (datos, 60 hosts) | 172.16.0.128 | /26 | .129 – .190 | 172.16.0.191 | 172.16.0.129 |
| R-EIGRP-1 VLAN 20 (voz, 30 hosts) | 172.16.0.192 | /27 | .193 – .222 | 172.16.0.223 | 172.16.0.193 |

### Enlaces punto a punto — bloque base 10.0.0.0/24

Cada enlace serial usa un prefijo /30 (dos direcciones utilizables).

| Enlace | Red | Extremo A | Extremo B |
|---|---|---|---|
| R-OSPF-1 ↔ R-OSPF-2 | 10.0.0.0/30 | 10.0.0.1 | 10.0.0.2 |
| R-OSPF-1 ↔ R-CENTRAL | 10.0.0.4/30 | 10.0.0.5 | 10.0.0.6 |
| R-OSPF-2 ↔ R-CENTRAL | 10.0.0.8/30 | 10.0.0.9 | 10.0.0.10 |
| R-EIGRP-1 ↔ R-EIGRP-2 | 10.0.0.12/30 | 10.0.0.13 | 10.0.0.14 |
| R-EIGRP-1 ↔ R-CENTRAL | 10.0.0.16/30 | 10.0.0.17 | 10.0.0.18 |
| R-EIGRP-2 ↔ R-CENTRAL | 10.0.0.20/30 | 10.0.0.21 | 10.0.0.22 |

## Asignación de interfaces

| Router | Interfaz | Dirección | Conecta a |
|---|---|---|---|
| R-OSPF-1 | Gi0/0.10 / Gi0/0.30 | 192.168.1.1 / 192.168.1.65 | SW-OSPF-1 (trunk) |
| | Se0/1/0 | 10.0.0.1 | R-OSPF-2 |
| | Se0/1/1 | 10.0.0.5 | R-CENTRAL |
| R-OSPF-2 | Gi0/0 | 192.168.0.1 | switch de sitio |
| | Se0/1/1 | 10.0.0.2 | R-OSPF-1 |
| | Se0/1/0 | 10.0.0.9 | R-CENTRAL |
| R-CENTRAL | Se0/0/0 | 10.0.0.6 | R-OSPF-1 |
| | Se0/0/1 | 10.0.0.10 | R-OSPF-2 |
| | Se0/1/0 | 10.0.0.18 | R-EIGRP-1 |
| | Se0/1/1 | 10.0.0.22 | R-EIGRP-2 |
| R-EIGRP-1 | Gi0/0.10 / Gi0/0.20 | 172.16.0.129 / 172.16.0.193 | SW-EIGRP-1 (trunk) |
| | Se0/1/0 | 10.0.0.13 | R-EIGRP-2 |
| | Se0/1/1 | 10.0.0.17 | R-CENTRAL |
| R-EIGRP-2 | Gi0/0 | 172.16.0.1 | switch de sitio |
| | Se0/1/1 | 10.0.0.14 | R-EIGRP-1 |
| | Se0/1/0 | 10.0.0.21 | R-CENTRAL |

## Enrutamiento implementado

**OSPF** — proceso 1, área única 0. Se anuncian las subinterfaces de VLAN, la LAN de R-OSPF-2 y los tres enlaces seriales del lado OSPF.

**EIGRP** — sistema autónomo 100 con `no auto-summary`, de modo que las subredes se propagan con su máscara real y no se resumen automáticamente en el límite classful.

**Enrutamiento entre VLANs** — se implementó router-on-a-stick: la interfaz física del router se configura sin dirección y cada VLAN recibe una subinterfaz con encapsulación `dot1Q`. El enlace hacia el switch opera como trunk 802.1Q.

**Redistribución mutua en R-CENTRAL**

De EIGRP hacia OSPF las rutas se inyectan como externas de **tipo 1 (E1)**, para que el costo refleje tanto la métrica externa como el costo interno de llegar al ASBR. De OSPF hacia EIGRP se declara explícitamente la métrica semilla, ya que EIGRP no puede inferir los parámetros físicos de una red externa y, sin ella, asignaría métrica infinita:

```
router eigrp 100
 redistribute ospf 1 metric 100000 10 255 1 1500
```

**Balanceo de carga en OSPF (ECMP)** — se ajustaron los costos de interfaz (20 en el enlace directo entre los routers OSPF, 10 en cada tramo hacia R-CENTRAL) para que ambos caminos resulten equivalentes. La tabla de enrutamiento muestra dos rutas con métrica 40 y `traffic share count` 1 en cada una.

**Balanceo de carga en EIGRP (variance)** — mediante el ajuste del retardo del enlace directo, la ruta por R-CENTRAL cumple la condición de factibilidad y califica como *feasible successor*. Con `variance 2` ambas rutas ingresan a la tabla de enrutamiento.

**Sumarización** — R-CENTRAL anuncia hacia el dominio EIGRP una única ruta de resumen que cubre todas las subredes del lado OSPF, en lugar de inyectarlas por separado:

```
interface Serial0/1/0
 ip summary-address eigrp 100 192.168.0.0 255.255.254.0
```

## Validación

La conectividad extremo a extremo se verificó con `ping` y `tracert` desde un equipo de la VLAN 10 de R-OSPF-1 hacia un equipo de la LAN de R-EIGRP-2, atravesando ambos dominios de enrutamiento a través de R-CENTRAL. Las salidas correspondientes se documentan en el segundo PDF.

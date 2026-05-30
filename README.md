# 🛡️ Wazuh SIEM — Home Security Operations Lab

> Laboratorio de seguridad real implementado en infraestructura física para práctica y documentación de operaciones SOC. Sin máquinas virtuales — hardware real, eventos reales.

---

## 📋 Tabla de contenidos

1. [Descripción del laboratorio](#descripción-del-laboratorio)
2. [Arquitectura](#arquitectura)
3. [Equipos y tecnologías](#equipos-y-tecnologías)
4. [Instalación Wazuh Server (Ubuntu)](#instalación-wazuh-server-ubuntu)
5. [Configuración IP estática en Ubuntu Desktop](#configuración-ip-estática-en-ubuntu-desktop)
6. [Instalación Agente en Kali Linux](#instalación-agente-en-kali-linux)
7. [Instalación Agente en macOS](#instalación-agente-en-macos)
8. [Integración DVR Hikvision vía Syslog](#integración-dvr-hikvision-vía-syslog)
9. [Casos de uso SOC](#casos-de-uso-soc)
   - [Detección SSH Brute Force](#caso-1--detección-de-ataque-ssh-brute-force)
   - [Filtrado y análisis de alertas](#caso-2--filtrado-y-análisis-de-alertas)
   - [Monitoreo de cámaras IP](#caso-3--monitoreo-de-cámaras-ip-hikvision)
   - [Eventos AppArmor](#caso-4--detección-de-eventos-apparmor)
10. [Mapeo MITRE ATT&CK](#mapeo-mitre-attck)
11. [Lecciones aprendidas](#lecciones-aprendidas)
12. [Próximos pasos](#próximos-pasos)

---

## Descripción del laboratorio

Laboratorio SOC casero implementado sobre hardware físico real. El objetivo es practicar detección, análisis y respuesta a incidentes replicando condiciones de un entorno empresarial pequeño.

**Wazuh** actúa como plataforma SIEM centralizada recibiendo eventos de:
- Endpoints Linux y macOS mediante agentes
- DVR de cámaras de seguridad Hikvision mediante Syslog
- Firewall UFW del servidor Ubuntu

---

## Arquitectura

```
                    ┌─────────────────────────────────┐
                    │         RED LOCAL (LAN)           │
                    │         192.168.1.0/24            │
                    └─────────────────────────────────┘
                                     │
        ┌────────────────────────────┼──────────────────────────┐
        │                            │                          │
 ┌──────▼──────┐             ┌───────▼──────┐          ┌───────▼──────┐
 │  UBUNTU PC   │             │  KALI LINUX  │          │    macOS     │
 │ 192.168.1.18 │◄───────────►│ 192.168.1.40 │          │ 192.168.1.12 │
 │ Wazuh Server │             │   Agent 001  │          │  Agent 002   │
 │  + Manager   │             │  (Notebook)  │          │  (MacBook)   │
 │  + Indexer   │             └──────────────┘          └──────────────┘
 │  + Dashboard │
 └──────▲───────┘
        │
        │  rsyslog UDP 514
        │
 ┌──────┴───────┐
 │  DVR HIKVISION│
 │  192.168.1.2  │
 │  CCTV / Cámaras│
 └──────────────┘
```

---

## Equipos y tecnologías

| Rol | IP | Sistema Operativo | Función |
|-----|----|------------------|---------|
| Wazuh Server | 192.168.1.18 | Ubuntu (PC físico) | Manager, Indexer, Dashboard |
| Agente 001 | 192.168.1.40 | Kali Linux (Notebook físico) | Agente monitoreado / testing |
| Agente 002 | 192.168.1.12 | macOS 15.7.3 (MacBook Air) | Agente monitoreado |
| Fuente Syslog | 192.168.1.2 | DVR Hikvision | Cámaras IP vía rsyslog |

**Stack tecnológico:**
- Wazuh 4.14.5 (Manager + Indexer + Dashboard)
- OpenSearch (indexación de eventos)
- Wazuh Agent v4.14.5 (Linux / macOS)
- rsyslog (recepción de Syslog del DVR)
- UFW (firewall Ubuntu)

---

## Instalación Wazuh Server (Ubuntu)

### Prerequisitos

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install curl -y
```

### Instalación All-in-One

```bash
curl -sO https://packages.wazuh.com/4.7/wazuh-install.sh
chmod +x wazuh-install.sh
sudo bash wazuh-install.sh -a
```

> ⚠️ El proceso tarda entre 10–20 minutos. Al finalizar guarda las credenciales del dashboard.

### Verificar servicios

```bash
sudo systemctl status wazuh-manager
sudo systemctl status wazuh-indexer
sudo systemctl status wazuh-dashboard
```

### Acceso al Dashboard

```
URL:      https://192.168.1.18:443
Usuario:  admin
Password: <generada durante instalación>
```

---

## Configuración IP estática en Ubuntu Desktop

Ubuntu Desktop usa **NetworkManager** como gestor de red. La forma correcta de fijar la IP es con `nmcli`, no editando archivos Netplan directamente.

### Verificar interfaz y conexión activa

```bash
# Ver interfaces disponibles
ip a

# Ver conexión WiFi activa
nmcli connection show --active

# Si el driver WiFi no responde, recargarlo
sudo modprobe -r iwlwifi && sleep 2 && sudo modprobe iwlwifi
```

### Fijar IP estática con nmcli

```bash
# Modificar la conexión activa (reemplaza "NOMBRE_CONEXION" con la tuya)
nmcli connection modify "NOMBRE_CONEXION" \
  ipv4.method manual \
  ipv4.addresses 192.168.1.18/24 \
  ipv4.gateway 192.168.1.1 \
  ipv4.dns "8.8.8.8"

nmcli connection up "NOMBRE_CONEXION"
ip a
```

> 📌 **Lección aprendida:** En Ubuntu Desktop, editar archivos Netplan puede entrar en conflicto con NetworkManager. Usar siempre `nmcli` para gestionar redes en entornos con interfaz gráfica.

---

## Instalación Agente en Kali Linux

### En el servidor — obtener IP

```bash
ip a | grep inet
```

### En Kali — instalar el agente

```bash
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | sudo gpg \
  --no-default-keyring \
  --keyring gnupg-ring:/usr/share/keyrings/wazuh.gpg --import

echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] \
  https://packages.wazuh.com/4.x/apt/ stable main" | \
  sudo tee /etc/apt/sources.list.d/wazuh.list

sudo apt update && sudo apt install wazuh-agent -y
```

### Configurar conexión al servidor

```bash
sudo nano /var/ossec/etc/ossec.conf
```

```xml
<client>
  <server>
    <address>192.168.1.18</address>
    <port>1514</port>
    <protocol>tcp</protocol>
  </server>
</client>
```

### Activar el agente

```bash
sudo systemctl daemon-reload
sudo systemctl enable wazuh-agent
sudo systemctl start wazuh-agent
sudo systemctl status wazuh-agent
```

---

## Instalación Agente en macOS

### Descargar e instalar

```bash
# Apple Silicon
curl -so wazuh-agent.pkg \
  https://packages.wazuh.com/4.x/macos/wazuh-agent-4.14.5-1.arm64.pkg

# Intel
# curl -so wazuh-agent.pkg \
#   https://packages.wazuh.com/4.x/macos/wazuh-agent-4.14.5-1.intel64.pkg

sudo WAZUH_MANAGER="192.168.1.18" \
     WAZUH_AGENT_NAME="MacBook-Air" \
     installer -pkg wazuh-agent.pkg -target /
```

### Iniciar el agente

```bash
sudo /Library/Ossec/bin/wazuh-control start
sudo /Library/Ossec/bin/wazuh-control status
```

> 📌 macOS puede solicitar permisos en **Preferencias del Sistema → Seguridad y Privacidad → Privacidad** para que el agente acceda a los logs del sistema.

> ⚠️ Mantener el agente actualizado a la misma versión que el servidor para evitar desconexiones.

---

## Integración DVR Hikvision vía Syslog

Las cámaras Hikvision no admiten agente Wazuh, pero el DVR puede enviar logs mediante **Syslog UDP/514**. La arquitectura es:

```
DVR Hikvision → rsyslog (514) → /var/log/hikvision-events.log → Wazuh logcollector
```

### Paso 1 — Configurar rsyslog para recibir del DVR

```bash
sudo nano /etc/rsyslog.d/hikvision.conf
```

```
# Habilitar recepción UDP 514
module(load="imudp")
input(type="imudp" port="514")

# Escribir logs del DVR Hikvision al archivo dedicado
if $fromhost-ip == '192.168.1.2' then /var/log/hikvision-events.log
& stop
```

```bash
# Crear archivo de logs
sudo touch /var/log/hikvision-events.log
sudo chmod 644 /var/log/hikvision-events.log

sudo systemctl restart rsyslog
sudo ss -ulnp | grep 514   # verificar que rsyslog escucha
```

### Paso 2 — Configurar UFW para permitir tráfico del DVR

```bash
sudo ufw allow from 192.168.1.2 to any port 514 proto udp
sudo ufw status | grep 514
```

### Paso 3 — Agregar localfile en Wazuh ossec.conf

```bash
sudo nano /var/ossec/etc/ossec.conf
```

Agregar dentro de `<ossec_config>`:

```xml
<localfile>
  <log_format>syslog</log_format>
  <location>/var/log/hikvision-events.log</location>
  <only-future-events>no</only-future-events>
</localfile>
```

```bash
sudo systemctl restart wazuh-manager

# Verificar que Wazuh lee el archivo
sudo tail -f /var/ossec/logs/ossec.log | grep hikvision
```

**Salida esperada:**
```
wazuh-logcollector: INFO: (1950): Analyzing file: '/var/log/hikvision-events.log'
```

### Paso 4 — Configurar el DVR Hikvision

En la interfaz web del DVR (`http://192.168.1.2`):
```
Configuración → Red → Ajustes avanzados → Configuración del servidor de registros
```

- **Habilitado:** ✅
- **Dirección del servidor:** `192.168.1.18`
- **Puerto:** `514`

> 📌 El botón "Prueba" del DVR verifica la conexión a rsyslog, no directamente a Wazuh.

---

## Casos de uso SOC

---

### Caso 1 — Detección de ataque SSH Brute Force

**Herramienta:** Hydra desde Kali Linux → Target: Ubuntu Server

```bash
hydra -l root -P /usr/share/wordlists/rockyou.txt \
      ssh://192.168.1.18 -t 4 -V
```

**Alertas generadas:**

| Rule ID | Descripción | Nivel |
|---------|-------------|-------|
| 5710 | Attempted login using non-existent user | 5 |
| 5712 | SSHD brute force trying to get access | 10 |
| 5763 | Multiple SSHD authentication failures | 10 |

**Filtro en Threat Hunting:**
```
rule.id: 5712 AND agent.name: "kali"
```

---

### Caso 2 — Filtrado y análisis de alertas

**Queries útiles en Threat Hunting:**

```kql
# Alertas críticas últimas 24h
rule.level >= 10

# Eventos por agente específico
agent.name: "kali"

# Eventos por IP de origen
data.srcip: "192.168.1.40"

# Técnica MITRE
rule.mitre.technique: "Brute Force"

# Logs del DVR Hikvision
location: "/var/log/hikvision-events.log"

# Sesiones sudo ejecutadas
rule.id: 5402
```

**Eventos observados en el lab (últimas 24h):**

| Rule ID | Descripción | Agente | Significado |
|---------|-------------|--------|-------------|
| 533 | Puerto abierto/cerrado | Todos | Cambios de conectividad |
| 510 | Host-based anomaly (rootcheck) | kali | Verificación de integridad |
| 503 | Wazuh agent started | kali | Reinicio del agente |
| 5402 | Sudo to ROOT executed | servidor | Comandos administrativos |
| 5501/5502 | PAM session opened/closed | servidor | Sesiones de usuario |
| 89602/89603 | Screen locked/unlocked | MacBook | Actividad de sesión macOS |
| 52002 | AppArmor DENIED | servidor | Restricciones de seguridad |

---

### Caso 3 — Monitoreo de cámaras IP Hikvision

**Tipo de eventos capturados del DVR:**

```xml
<EventNotificationAlert version="1.0" xmlns="http://www.hikvision.com/ver20/XMLSchema">
  <dateTime>2026-05-14T01:22:38</dateTime>
  <eventType>videoloss</eventType>
  <eventDescription>videoloss alarm</eventDescription>
</EventNotificationAlert>
```

**Tipos de alerta Hikvision monitoreados:**

| Evento | Descripción | Relevancia SOC |
|--------|-------------|----------------|
| `videoloss` | Pérdida de señal de cámara | Posible manipulación física |
| `motiondetection` | Detección de movimiento | Actividad en zona monitoreada |
| `login` / `logout` | Acceso a interfaz DVR | Control de acceso administrativo |
| `configurationChange` | Cambio de configuración | Posible alteración no autorizada |

**Análisis SOC — videoloss a las 00:52:**
Pérdida repetida de señal de cámara en horario nocturno → investigar si es falla técnica o manipulación física del dispositivo.

**Verificar en tiempo real:**
```bash
sudo tail -f /var/log/hikvision-events.log
```

---

### Caso 4 — Detección de eventos AppArmor

**Observación:** 4,210 eventos `AppArmor DENIED` (rule 52002) detectados en el servidor.

**Análisis:**
```bash
sudo grep "apparmor" /var/log/syslog | tail -10
```

Los eventos corresponden a `snap.firmware-updater` intentando acceder a `/proc/sys/vm/max_map_count` — comportamiento normal de actualizaciones de sistema. No representa una amenaza, pero ejemplifica cómo Wazuh captura intentos de acceso denegados por el sistema de control de aplicaciones de Ubuntu.

---

## Mapeo MITRE ATT&CK

| Técnica | ID MITRE | Caso en este lab |
|---------|----------|-----------------|
| Brute Force | T1110 | Ataque SSH desde Kali |
| Valid Accounts | T1078 | Monitoreo de logins PAM |
| Network Service Discovery | T1046 | Escaneos de puertos |
| Impair Defenses | T1562 | Eventos AppArmor |
| Video Capture | T1125 | Monitoreo DVR/cámaras |
| System Log Files | T1005 | Análisis logs Hikvision |

---

## Lecciones aprendidas

- En **Ubuntu Desktop**, NetworkManager gestiona la red — usar `nmcli` en lugar de editar Netplan directamente evita conflictos.
- Los **DVR Hikvision** envían Syslog al puerto 514 fijo — la arquitectura correcta es `DVR → rsyslog → archivo → Wazuh`.
- El puerto 514 en Ubuntu lo ocupa `rsyslogd` por defecto — Wazuh recibe los logs a través del archivo intermediario.
- Las **cámaras IP** son vectores de ataque subestimados; integrarlas al SIEM agrega visibilidad valiosa sobre accesos físicos.
- Los eventos **AppArmor** generan muchas alertas de nivel bajo — importante crear filtros para separar ruido de señal real.
- Trabajar con **hardware físico real** expone problemas reales: drivers WiFi, conflictos de servicios, permisos de archivos.

---

## Próximos pasos

- [ ] Crear reglas personalizadas para eventos Hikvision (videoloss, motion detection)
- [ ] Configurar alertas por email ante eventos críticos del DVR
- [ ] Implementar Suricata como IDS de red complementario
- [ ] Documentar análisis de PCAPs con Wireshark
- [ ] Crear playbook formal de respuesta a incidentes SSH Brute Force
- [ ] Actualizar agente macOS a v4.14.5
- [ ] Mapear todos los eventos del lab a MITRE ATT&CK Navigator
- [ ] Integrar VirusTotal para análisis de hashes

---

## Estructura del repositorio

```
wazuh-siem-lab/
├── README.md                        ← Este archivo
├── docs/
│   └── screenshots/                 ← Capturas del dashboard
├── configs/
│   ├── ossec.conf                   ← Configuración Wazuh Manager
│   └── hikvision-rsyslog.conf       ← Configuración rsyslog para DVR
└── playbooks/
    └── ssh-bruteforce-response.md   ← (próximamente)
```

---

## Recursos

- [Documentación oficial de Wazuh](https://documentation.wazuh.com)
- [MITRE ATT&CK Framework](https://attack.mitre.org)
- [Wazuh Rules Repository](https://github.com/wazuh/wazuh-ruleset)
- [Hikvision SDK & Integration Docs](https://www.hikvisionamericas.com/support)

---

*Laboratorio desarrollado con fines educativos y de práctica profesional en ciberseguridad.*
*Autor: Roxana Garat

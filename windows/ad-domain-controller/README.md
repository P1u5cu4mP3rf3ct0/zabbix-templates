# 🖥️ Windows AD Event Log — Domain Controller

Plantilla de Zabbix para la monitorización del log de eventos de seguridad de Windows en Controladores de Dominio de Active Directory.

![Zabbix](https://img.shields.io/badge/Zabbix%20Server-7.x-red?style=flat-square)
![Windows Server](https://img.shields.io/badge/Windows%20Server-2016%20%7C%202019%20%7C%202022-0078D4?style=flat-square&logo=data:image/svg+xml;base64,PHN2ZyB4bWxucz0iaHR0cDovL3d3dy53My5vcmcvMjAwMC9zdmciIHZpZXdCb3g9IjAgMCA4OCA4OCI+PHBhdGggZmlsbD0id2hpdGUiIGQ9Ik0wIDEyLjQwMmwzNS42ODctNC44Ni4wMTYgMzQuNDIzLTM1LjY3LjIwM3ptMzUuNjcgMzMuNTI5bC4wMjggMzQuNDUzTC4wMjggNzUuNDguMDI2IDQ1Ljd6bTQuMzI2LTM5LjAyNUw4Ny4zMTQgMHY0MS41MjdsLTQ3LjMxOC4zNzZ6bTQ3LjMyOSAzOS4zNDlsLS4wMTEgNDEuMzQtNDcuMzE4LTYuNjc4LS4wNjYtMzQuNzM5eiIvPjwvc3ZnPg==)
![License](https://img.shields.io/badge/License-MIT-green?style=flat-square)

---

## 📋 Requisitos

| Componente | Requisito | Notas |
|---|---|---|
| Zabbix | 7.x o superior | Versiones anteriores no soportan la sintaxis de triggers utilizada |
| Modo del agente | Activo obligatorio | El agente pasivo no puede recopilar logs de eventos |
| SO compatible | Windows Server 2016 / 2019 / 2022 | En versiones anteriores algunos event IDs pueden diferir |
| Permisos del agente | Lectura de logs de eventos | El servicio del agente debe correr como cuenta con acceso a los logs |
| Auditoría avanzada | Habilitada en GPO | Sin ella, muchos event IDs no se generan en el log de seguridad |

**Logs necesarios según rol del servidor:**

| Log | Rol | Necesario para | Habilitado por defecto |
|---|:---:|---|:---:|
| `Security` | Todos | Usuarios, grupos, autenticación, privilegios, servicios, tareas | ✅ |
| `System` | Todos | Fallos de procesamiento de GPO | ✅ |
| `DhcpAdminEvents` | DHCP Server | Monitorización del servicio DHCP y detección de servidores no autorizados | ✅ *(si el rol está instalado)* |
| `Microsoft-Windows-DNSServer/Audit` | DNS Server | Cambios en zonas y registros DNS | ✅ *(desde WS 2016)* |

> ⚠️ Para los eventos de GPO (5136/5137/5141) es necesario habilitar **Audit Directory Service Changes** en la Política de Auditoría Avanzada del dominio (`auditpol /set /subcategory:"Directory Service Changes" /success:enable`).

---

## 📦 Instalación

1. En Zabbix, ir a **Configuración → Plantillas → Importar**
2. Seleccionar el archivo `template_windows_ad_event_log.yaml`
3. Asignar la plantilla al host del Controlador de Dominio
4. Verificar que el Agente Zabbix está configurado en **modo activo**

---

## 🔍 Eventos monitorizados

### 👤 Cuentas de usuario

| Event ID | Descripción | Severidad | Recovery | Acción recomendada |
|:---:|---|:---:|:---:|---|
| 4720 | Cuenta de usuario creada | `INFO` | — | Verificar que la creación es legítima y autorizada |
| 4722 | Cuenta de usuario habilitada | — | — | Correlacionar con 4725 si se usa trigger combinado |
| 4723 | Intento de cambio de contraseña | `INFO` | — | Investigar si hay intentos repetidos fallidos |
| 4724 | Restablecimiento de contraseña por administrador | `WARNING` | — | Confirmar que el administrador estaba autorizado |
| 4725 | Cuenta de usuario deshabilitada | `INFO` | 4722 | Se resuelve automáticamente cuando la cuenta es habilitada |
| 4726 | Cuenta de usuario eliminada | `WARNING` | — | Acción irreversible — verificar inmediatamente |
| 4738 | Atributos de cuenta de usuario modificados | `INFO` | — | Revisar el atributo modificado (nombre, flags, expiración...) |
| 4740 | Cuenta de usuario bloqueada | `WARNING` | 4767 | Identificar el origen del bloqueo (equipo, aplicación) |
| 4767 | Cuenta de usuario desbloqueada | — | *(recovery)* | Confirmar que el desbloqueo fue realizado por un administrador |

### 👥 Grupos de seguridad

| Event ID | Descripción | Tipo de grupo | Severidad | Recovery | Riesgo |
|:---:|---|:---:|:---:|:---:|---|
| 4728 | Miembro añadido | Global | `INFO` | 4729 | Medio — revisar si el grupo tiene permisos elevados |
| 4729 | Miembro eliminado | Global | — | *(recovery)* | — |
| 4732 | Miembro añadido | Local | `WARNING` | 4733 | Alto — especialmente si el grupo es *Administrators* o *Remote Desktop Users* |
| 4733 | Miembro eliminado | Local | — | *(recovery)* | — |
| 4756 | Miembro añadido | Universal | `INFO` | 4757 | Medio — los grupos universales afectan a todo el bosque AD |
| 4757 | Miembro eliminado | Universal | — | *(recovery)* | — |

### 🔐 Autenticación

| Event ID | Descripción | Severidad | Posible causa | Acción recomendada |
|:---:|---|:---:|---|---|
| 4625 | Fallo de inicio de sesión | `INFO` | Contraseña incorrecta, cuenta bloqueada, hora desincronizada | Investigar si se repite para el mismo usuario o desde la misma máquina |
| 4648 | Inicio de sesión con credenciales explícitas | `WARNING` | Uso de `runas`, `psexec`, movimiento lateral | Verificar si corresponde a actividad administrativa legítima |
| 4771 | Fallo de preautenticación Kerberos | `WARNING` | Contraseña incorrecta, ataque AS-REP Roasting, Kerberoasting | Correlacionar con la IP de origen y el volumen de intentos |

### 📜 Políticas y dominio

| Event ID | Descripción | Severidad | Impacto | Acción recomendada |
|:---:|---|:---:|---|---|
| 4713 | Política Kerberos modificada | `HIGH` | Afecta a toda la autenticación del dominio — cambios en TTL de tickets, algoritmos de cifrado | Verificar el cambio en la GPO de *Default Domain Policy* |
| 4739 | Política de dominio modificada | `AVERAGE` | Puede afectar a política de contraseñas, bloqueo de cuentas o Kerberos | Revisar qué parámetro fue modificado y por quién |

### 🏢 GPO y objetos de Active Directory

> Requiere habilitar **Audit Directory Service Changes** en la Política de Auditoría Avanzada.

| Event ID | Descripción | Severidad | Objetos afectados | Acción recomendada |
|:---:|---|:---:|---|---|
| 4706 | Nueva confianza de dominio creada | `HIGH` | Amplía el perímetro de seguridad del dominio | Verificar inmediatamente con el equipo de identidad |
| 4707 | Confianza de dominio eliminada | `AVERAGE` | Puede romper la autenticación entre dominios | Confirmar que era una eliminación planificada |
| 5136 | Objeto de directorio modificado | `INFO` | GPOs, OUs, cuentas, grupos, políticas de contraseña... | Revisar el atributo modificado y el autor del cambio |
| 5137 | Objeto de directorio creado | `INFO` | GPOs, OUs, cuentas, grupos... | Verificar que la creación es legítima y autorizada |
| 5141 | Objeto de directorio eliminado | `WARNING` | GPOs, OUs — puede ser irreversible sin backup | Restaurar desde backup AD si la eliminación no es intencionada |
| 1085 *(System)* | Fallo en el procesamiento de GPO | `AVERAGE` | Las políticas no se aplican correctamente en el equipo | Revisar conectividad con el DC, replicación AD y DNS |
| 1055 *(System)* | GPO no pudo contactar con el DC | `AVERAGE` | Ninguna GPO se aplica hasta restaurar la conexión | Verificar conectividad de red y resolución DNS |

### 💻 Cuentas de equipo

| Event ID | Descripción | Severidad | Acción recomendada |
|:---:|---|:---:|---|
| 4741 | Cuenta de equipo creada en el dominio | `INFO` | Verificar que la unión al dominio fue autorizada por el equipo de sistemas |
| 4743 | Cuenta de equipo eliminada del dominio | `WARNING` | Confirmar si era una baja planificada — el equipo perderá acceso a recursos del dominio |

### 🔑 Privilegios y servicios

| Event ID | Descripción | Severidad | Riesgo | Acción recomendada |
|:---:|---|:---:|---|---|
| 4672 | Privilegios especiales asignados a un inicio de sesión | `INFO` | Medio — se genera con cada login de administrador | Investigar si aparece para cuentas que no deberían tener privilegios |
| 4697 | Nuevo servicio instalado en el sistema | `HIGH` | Alto — técnica común de persistencia de malware | Verificar el nombre, ruta del ejecutable y la cuenta que lo instaló |
| 4698 | Tarea programada creada | `WARNING` | Alto — técnica común de persistencia | El trigger permanece abierto mientras la tarea siga existiendo en el sistema |
| 4699 | Tarea programada eliminada | — | — | Actúa como recovery del trigger de creación (4698) |
| 4702 | Tarea programada modificada | `INFO` | Medio — puede ser una modificación de persistencia existente | Verificar el nombre de la tarea y el autor del cambio |

### 🛡️ Integridad del sistema

| Event ID | Descripción | Severidad | Implicaciones | Acción recomendada |
|:---:|---|:---:|---|---|
| 1102 | Log de seguridad borrado | `HIGH` | Posible acción de encubrimiento tras un incidente | Investigar inmediatamente — comprobar backups del log y actividad previa |
| 4616 | Hora del sistema modificada | `HIGH` | Puede invalidar tickets Kerberos y romper la replicación AD | Sincronizar con el servidor NTP del dominio y verificar la causa |

### 🌐 Servidor DHCP *(log: DhcpAdminEvents)*

| Event ID | Nivel | Descripción | Severidad | Impacto | Acción recomendada |
|:---:|:---:|---|:---:|---|---|
| 1008 | Error | Servicio DHCP detenido por error crítico | `HIGH` | Los clientes no pueden obtener ni renovar IPs | Revisar el log del sistema y reiniciar el servicio DHCP |
| 1020 | Warning | Pool de direcciones casi agotado | `AVERAGE` | Próxima indisponibilidad de IPs | Ampliar el rango del scope o reducir el tiempo de arrendamiento |
| 1053 | Error | Servidor DHCP no autorizado detectado en la red | `HIGH` | El servicio DHCP puede detenerse por seguridad | Localizar y desconectar el servidor DHCP no autorizado |
| 1054 | Error | Servicio DHCP detenido por servidor no autorizado | `HIGH` | Ningún cliente puede obtener IP hasta resolver el conflicto | Eliminar el servidor no autorizado y reiniciar el servicio DHCP |
| 1063 | Error | Pool de direcciones completamente agotado | `HIGH` | Los nuevos clientes no pueden obtener dirección IP | Ampliar urgentemente el scope o forzar la liberación de arrendamientos expirados |

### 🌍 Servidor DNS *(log: Microsoft-Windows-DNSServer/Audit)*

| Event ID | Descripción | Ámbito | Severidad | Acción recomendada |
|:---:|---|:---:|:---:|---|
| 512 | Zona DNS creada | Zona completa | `WARNING` | Verificar que la nueva zona es legítima y está correctamente configurada |
| 513 | Zona DNS eliminada | Zona completa | `HIGH` | Acción potencialmente crítica — verificar si existe backup y si fue planificada |
| 514 | Configuración de zona DNS modificada | Zona completa | `INFO` | Revisar los cambios (transferencias de zona, actualizaciones dinámicas, TTL...) |
| 515 | Registro DNS creado | Registro individual | `INFO` | Puede ser muy frecuente con actualizaciones dinámicas activas — ver notas |
| 516 | Registro DNS eliminado | Registro individual | `WARNING` | Verificar que la eliminación era intencionada — puede causar fallos de resolución |

---

## 📊 Métricas calculadas

| Clave | Descripción | Intervalo de cálculo | Umbral AVERAGE | Umbral HIGH |
|---|---|:---:|:---:|:---:|
| `ad.count.lockouts.1h` | Total de bloqueos de cuentas | cada 5 min · ventana 1h | ≥ 5 | ≥ 15 |
| `ad.count.authfail.1h` | Total de fallos de autenticación (4625) | cada 5 min · ventana 1h | ≥ 20 | ≥ 100 |
| `ad.count.kerberosfail.1h` | Total de fallos de preautenticación Kerberos (4771) | cada 5 min · ventana 1h | ≥ 10 | ≥ 50 |
| `ad.count.userchanges.1h` | Cambios en cuentas de usuario (4720/4722/4725/4726/4738) | cada 5 min · ventana 1h | — | — |
| `ad.count.groupchanges.1h` | Cambios en membresía de grupos (4728/4729/4732/4733/4756/4757) | cada 5 min · ventana 1h | — | — |
| `ad.count.passwordevents.1h` | Eventos de contraseña (4723/4724) | cada 5 min · ventana 1h | — | — |
| `ad.count.critical.1h` | Eventos críticos de seguridad (1102/4697/4698/4648) | cada 5 min · ventana 1h | — | ≥ 3 |
| `ad.count.gpochanges.1h` | Cambios en objetos AD y GPOs (5136/5137/5141) | cada 5 min · ventana 1h | ≥ 10 | — |
| `ad.count.dnschanges.1h` | Cambios en zonas y registros DNS (512–516) | cada 5 min · ventana 1h | — | — |

---

## 🔗 Dependencias entre triggers

Los triggers de menor severidad se suprimen automáticamente cuando un trigger de mayor severidad del mismo tipo está activo, evitando duplicidad de alertas.

| Trigger suprimido | Severidad | Se activa a partir de | Depende de | Severidad | Se activa a partir de |
|---|:---:|:---:|---|:---:|:---:|
| Alta tasa de bloqueos | `AVERAGE` | ≥ 5 en 1h | Ola masiva de bloqueos | `HIGH` | ≥ 15 en 1h |
| Alta tasa de fallos de autenticación | `AVERAGE` | ≥ 20 en 1h | Ataque de fuerza bruta | `HIGH` | ≥ 100 en 1h |
| Fallos Kerberos elevados | `AVERAGE` | ≥ 10 en 1h | Posible Kerberoasting | `HIGH` | ≥ 50 en 1h |
| Pool DHCP casi lleno | `AVERAGE` | warning (1020) | Pool DHCP agotado | `HIGH` | error (1063) |

---

## ⚠️ Notas

- **Registros DNS (515/516):** puede generar mucho ruido si las actualizaciones dinámicas de DNS están activas. Se recomienda deshabilitar el trigger de creación en entornos con alta actividad dinámica.
- **Privilegios especiales (4672):** puede ser muy frecuente en DCs activos ya que se genera con cada inicio de sesión de administrador. Considera ajustar o deshabilitar su trigger si genera demasiado ruido.
- **Eventos GPO (5136/5137/5141):** requieren habilitar explícitamente **Audit Directory Service Changes** en la configuración de auditoría avanzada del dominio.
- **DHCP y DNS:** los módulos correspondientes solo son relevantes si el servidor tiene esos roles instalados. Si no es así, los items simplemente no recopilarán datos.

---

## 📄 Licencia

[MIT](https://github.com/P1u5cu4mP3rf3ct0/zabbix-templates/blob/main/LICENSE) — Libre para usar, modificar y distribuir. Se agradece dar crédito al autor.

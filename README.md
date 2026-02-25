# 📊 zabbix-templates

Colección de plantillas de Zabbix listas para usar, construidas desde la experiencia real.

![Zabbix Server](https://img.shields.io/badge/Zabbix%20Server-7.x-red?style=flat-square)
![License](https://img.shields.io/badge/License-MIT-green?style=flat-square)
![Contributions Welcome](https://img.shields.io/badge/Contributions-Welcome-brightgreen?style=flat-square)

Libres para usar, modificar y distribuir — se agradece dar crédito al autor.

---

## 📁 Estructura

```
zabbix-templates/
├── LICENSE
├── README.md
└── windows/
    └── ad-domain-controller-2022/
        ├── README.md
        └── template_windows_ad_event_log_2022.yaml

```

---

## 📦 Plantillas disponibles

### 🪟 Windows

| Plantilla | Descripción | Zabbix | SO |
|---|---|:---:|---|
| [AD Event Log - Domain Controller](./windows/ad-domain-controller/) | Monitorización del log de eventos de seguridad de Windows para Controladores de Dominio. Cubre cuentas de usuario, grupos, autenticación, GPO, DHCP y DNS. | 7.x | Windows Server 2016 / 2019 / 2022 |

---

## 🚀 Instalación

1. En Zabbix, ir a **Configuración → Plantillas → Importar**
2. Seleccionar el archivo `.yaml` de la plantilla deseada
3. Asignar la plantilla al host correspondiente
4. Asegurarse de que el Agente Zabbix está en **modo activo**

Cada plantilla incluye su propio `README.md` con los requisitos y notas específicas.

---

## 📄 Licencia

[MIT](./LICENSE) — Libre para usar, modificar y distribuir. Se agradece dar crédito al autor.

---

## 👤 Autor

Creado y mantenido por [P1u5cu4mP3rf3ct0](https://github.com/P1u5cu4mP3rf3ct0).

Si te ha sido útil, considera dejar una ⭐ en el repositorio.

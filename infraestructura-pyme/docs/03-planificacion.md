# 03 – Planificación del Proyecto

## Índice

- [1. Alcance del proyecto](#1-alcance-del-proyecto)
- [2. Fases y tareas](#2-fases-y-tareas)
- [3. Diagrama de Gantt](#3-diagrama-de-gantt)
- [4. Gestión de riesgos](#4-gestión-de-riesgos)
- [5. Equipo y responsabilidades](#5-equipo-y-responsabilidades)
- [6. Criterios de aceptación](#6-criterios-de-aceptación)

---

## 1. Alcance del proyecto

### Dentro del alcance ✅

- Instalación y configuración de pila LAMP (Linux + Apache + MySQL + PHP) en Ubuntu Server 22.04 LTS.
- Balanceador de carga HAProxy delante del servidor web (añadido en sesión 4).
- Acceso remoto seguro mediante SSH con autenticación por clave pública.
- Cortafuegos UFW con política de mínima exposición.
- Monitorización del sistema con Netdata.
- Copias de seguridad automáticas con `mysqldump` + `rsync` y rotación de 7 días.
- Certificado TLS gratuito con Certbot (Let's Encrypt).
- Documentación técnica completa en Markdown.
- Plan de recuperación ante desastres.

### Fuera del alcance ❌

- Desarrollo de la aplicación web del cliente.
- Configuración de CDN o WAF externo.
- Alta disponibilidad de la base de datos (replicación MySQL).
- Despliegue en cloud (AWS, Azure, GCP).

---

## 2. Fases y tareas

### Fase 1 – Análisis y diseño (Sesión 1)

| ID | Tarea | Responsable | Estado |
|---|---|---|---|
| T01 | Levantamiento de requisitos del cliente | Plataforma | ✅ Completado |
| T02 | Diseño de arquitectura de red | Plataforma | ✅ Completado |
| T03 | Selección de versiones de software | Plataforma | ✅ Completado |
| T04 | Definición de política de backups | Operaciones | ✅ Completado |
| T05 | Diseño del sistema de monitorización | Operaciones | ✅ Completado |

### Fase 2 – Documentación de instalación (Sesión 2)

| ID | Tarea | Responsable | Estado |
|---|---|---|---|
| T06 | Documentar instalación Apache + PHP | Plataforma | ✅ Completado |
| T07 | Documentar instalación MySQL + creación de BD | Plataforma | ✅ Completado |
| T08 | Documentar configuración SSH y UFW | Plataforma | ✅ Completado |
| T09 | Documentar instalación y configuración Netdata | Operaciones | ✅ Completado |
| T10 | Documentar scripts de backup automático | Operaciones | ✅ Completado |

### Fase 3 – Documentación de operación y recuperación (Sesión 3)

| ID | Tarea | Responsable | Estado |
|---|---|---|---|
| T11 | Guía de mantenimiento diario/semanal/mensual | Operaciones | ✅ Completado |
| T12 | Plan de recuperación ante desastres | Operaciones | ✅ Completado |
| T13 | Revisión cruzada de todos los documentos | Ambos | ✅ Completado |
| T14 | Resolución de conflictos Git (rebase) | Ambos | ✅ Completado |

### Fase 4 – Integración del balanceador y entrega (Sesión 4) 🆕

| ID | Tarea | Responsable | Estado |
|---|---|---|---|
| T15 | Documentar instalación y configuración HAProxy | Plataforma | ✅ Completado |
| T16 | Actualizar diagrama de red con HAProxy | Plataforma | ✅ Completado |
| T17 | Actualizar guía de operación con tareas del balanceador | Operaciones | ✅ Completado |
| T18 | Verificación de enlaces internos y coherencia del repo | Ambos | ✅ Completado |
| T19 | Redacción de REVISION.md | Ambos | ✅ Completado |
| T20 | Creación de release v1.0 en GitHub | Ambos | ✅ Completado |

---

## 3. Diagrama de Gantt

El proyecto se divide en 4 sesiones de trabajo (aproximadamente 2 horas cada una).

```
TAREA                              S1          S2          S3          S4
                                [========]  [========]  [========]  [========]

T01 Requisitos cliente            [===]
T02 Diseño arquitectura           [======]
T03 Versiones software            [===]
T04 Política de backups           [===]
T05 Diseño monitorización         [======]
T06 Doc. Apache + PHP                         [======]
T07 Doc. MySQL + BD                           [======]
T08 Doc. SSH + UFW                            [======]
T09 Doc. Netdata                              [======]
T10 Scripts backup                            [======]
T11 Guía mantenimiento                                    [======]
T12 Plan recuperación                                     [======]
T13 Revisión cruzada                                      [======]
T14 Conflicto Git (rebase)                                [======]
T15 Doc. HAProxy                                                      [====]
T16 Actualizar diagrama                                               [====]
T17 Operación balanceador                                             [====]
T18 Coherencia repositorio                                            [====]
T19 REVISION.md                                                       [====]
T20 Release v1.0                                                          []
```

> **Leyenda:** `[===]` = trabajo activo en esa sesión.

---

## 4. Gestión de riesgos

| ID | Riesgo | Probabilidad | Impacto | Mitigación |
|---|---|---|---|---|
| R01 | Conflictos Git no resueltos correctamente | Media | Alto | Comunicación frecuente entre miembros; `git pull` antes de trabajar. |
| R02 | Documentación técnica incorrecta (comandos erróneos) | Media | Medio | Revisión cruzada obligatoria mediante PR antes de merge. |
| R03 | Pérdida de datos de la BD del cliente | Baja | Muy alto | Backups automáticos diarios con retención 7 días + copia offsite mensual. |
| R04 | Acceso no autorizado por SSH | Baja | Alto | Autenticación solo por clave pública; SSH restringido a red local. |
| R05 | Caída del servidor web en producción | Media | Alto | Monitorización con Netdata + alertas; procedimiento de reinicio documentado. |
| R06 | Cambios de alcance tardíos del cliente | Alta | Medio | Proceso de gestión de cambios con Issue en GitHub y estimación antes de implementar. |

---

## 5. Equipo y responsabilidades

### Roles del proyecto (rotativos a partir de la sesión 3)

#### Documentalista de Plataforma

**Sesiones 1–2:**
- Documentación de infraestructura base: análisis, diseño, servidor web, base de datos, SSH/firewall.
- Apertura y gestión de PR de la rama `feature/plataforma-base`.

**Sesiones 3–4 (rol intercambiado):**
- Documentación de operación, recuperación y CHANGELOG.
- Integración HAProxy en diseño y planificación.

#### Documentalista de Operaciones

**Sesiones 1–2:**
- Documentación de monitorización, backups.
- Apertura y gestión de PR de la rama `feature/operaciones`.

**Sesiones 3–4 (rol intercambiado):**
- Documentación servidor web, base de datos, SSH/firewall.
- Integración HAProxy en guía de operación.

### Convenciones de nomenclatura de ramas

| Tipo | Patrón | Ejemplo |
|---|---|---|
| Nueva funcionalidad | `feature/<descripcion>` | `feature/balanceador-config` |
| Corrección de errores | `bugfix/<descripcion>` | `bugfix/tabla-versiones` |
| Hotfix urgente | `hotfix/<descripcion>` | `hotfix/enlace-roto-readme` |

---

## 6. Criterios de aceptación

El proyecto se considera **completado** cuando se cumplan todos los criterios siguientes:

### Repositorio Git

- [ ] Rama `main` protegida: solo merge mediante PR aprobado.
- [ ] Mínimo 2 conflictos resueltos documentados en el historial (1 merge + 1 rebase).
- [ ] Todos los PR con al menos 1 revisión del compañero.
- [ ] Mensajes de commit descriptivos en todos los commits.
- [ ] Release `v1.0` creada y publicada en GitHub.

### Documentación

- [ ] Todos los archivos Markdown del esqueleto inicial redactados.
- [ ] README.md actualizado con estado final del proyecto.
- [ ] CHANGELOG.md con entradas para cada sesión.
- [ ] REVISION.md con reflexión del equipo.
- [ ] Todos los enlaces internos entre documentos funcionan.
- [ ] Sin archivos huérfanos ni referencias inconsistentes.

### Contenido técnico

- [ ] Comandos de instalación verificados y correctos.
- [ ] Diagrama de red incluye HAProxy.
- [ ] Política de backups con retención y automatización definidas.
- [ ] Plan de recuperación ante desastres operativo.
- [ ] Guía de mantenimiento con tareas periódicas definidas.

---

*Documento mantenido por el equipo de sistemas. Última actualización: sesión 4.*  
*Ver también: [02-diseno.md](02-diseno.md) | [01-analisis.md](01-analisis.md) | [05-operacion.md](../05-operacion.md)*
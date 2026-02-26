# KoboToolbox API — Documentación Técnica Completa de Integración

**Versión:** 2.0 | **API:** v2 (KPI) | **Fecha:** Febrero 2026  
**Audiencia:** Desarrolladores backend, ingenieros de integración, arquitectos de software  
**Nivel:** Senior / Profesional

---

## Tabla de Contenidos

1. [Visión General Técnica](#1-visión-general-técnica)
2. [Arquitectura de la API](#2-arquitectura-de-la-api)
3. [Autenticación Profunda](#3-autenticación-profunda)
4. [Endpoints Principales — Análisis Profundo](#4-endpoints-principales)
5. [Modelado Interno de Datos](#5-modelado-interno-de-datos)
6. [Integraciones Reales](#6-integraciones-reales)
7. [Uso Avanzado](#7-uso-avanzado)
8. [Problemas Reales y Limitaciones](#8-problemas-reales-y-limitaciones)
9. [Comparativa Técnica](#9-comparativa-técnica)
10. [Buenas Prácticas de Ingeniería](#10-buenas-prácticas-de-ingeniería)

---

## 1. Visión General Técnica

### 1.1 ¿Qué es KoboToolbox?

KoboToolbox es una plataforma de código abierto para la recolección de datos en campo. Fue desarrollada originalmente por la Harvard Humanitarian Initiative (HHI) y se consolidó como organización sin ánimo de lucro independiente en 2019. Su uso principal se concentra en contextos humanitarios, investigación en desarrollo, y proyectos de monitoreo y evaluación (M&E).

Desde el punto de vista técnico, KoboToolbox es un ecosistema de servicios Django interconectados que expone una API REST JSON para la gestión completa del ciclo de vida de formularios y datos: diseño, despliegue, recolección, validación, exportación e integración.

### 1.2 Arquitectura del Ecosistema Kobo

El ecosistema consta de los siguientes componentes principales, todos de código abierto:

```
┌─────────────────────────────────────────────────────────────────────┐
│                     ECOSISTEMA KOBOTOOLBOX                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                     KPI (Principal)                           │   │
│  │  Django REST Framework Application                           │   │
│  │  ─────────────────────────────────────────────               │   │
│  │  • API v2 (/api/v2/*)                                        │   │
│  │  • Gestión de formularios (assets)                           │   │
│  │  • Biblioteca de preguntas                                   │   │
│  │  • Permisos y compartición                                   │   │
│  │  • Exportaciones (async + sync)                              │   │
│  │  • Reportes                                                  │   │
│  │  • Interfaz web (React SPA)                                  │   │
│  │                                                               │   │
│  │  Desde v2.024.25: incluye openrosa (antes KoboCAT)          │   │
│  └──────────────┬───────────────────────────────────────────────┘   │
│                 │                                                    │
│                 │ Internal API                                       │
│                 ▼                                                    │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │              openrosa (ex-KoboCAT)                           │   │
│  │  Integrado en KPI como aplicación Django                     │   │
│  │  ─────────────────────────────────────────                   │   │
│  │  • Protocolo OpenRosa (recepción de envíos)                  │   │
│  │  • Almacenamiento de submissions                             │   │
│  │  • Archivos multimedia adjuntos                              │   │
│  │  • API v1 (DEPRECADA - decommission Enero 2026)              │   │
│  └──────────────┬───────────────────────────────────────────────┘   │
│                 │                                                    │
│                 ▼                                                    │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                  Enketo Express                              │   │
│  │  Motor de formularios web HTML5                              │   │
│  │  ─────────────────────────────────────────                   │   │
│  │  • Renderizado de formularios XForm en navegador             │   │
│  │  • Soporte offline (service workers)                         │   │
│  │  • Vista previa de formularios                               │   │
│  │  • Edición de envíos existentes                              │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  ┌──────────────────────────┐  ┌─────────────────────────────────┐ │
│  │    Bases de Datos        │  │    Librerías de Soporte         │ │
│  │  ──────────────────      │  │  ─────────────────────          │ │
│  │  • PostgreSQL (KPI)      │  │  • formpack (parseo exports)    │ │
│  │  • PostgreSQL (KoboCAT)  │  │  • pyxform (XLSForm→XForm)     │ │
│  │  • MongoDB (Enketo)      │  │  • XLSForm standard            │ │
│  │  • Redis (caché/colas)   │  │  • OpenRosa XForms spec        │ │
│  └──────────────────────────┘  └─────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────┘
```

**Nota arquitectónica crítica:** Desde la versión 2.024.25, KoboCAT fue consolidado dentro de KPI como una aplicación Django llamada `openrosa`. Antes eran dos proyectos Django separados con bases de datos PostgreSQL independientes (separación que se realizó en v2.020.18). Esto tiene implicaciones directas para las integraciones: toda la API v2 se sirve desde un único punto (KPI), mientras que la API v1 (que apuntaba a KoboCAT/`kc.`) está deprecada y programada para eliminación.

### 1.3 Flujo Completo de Datos

```
 DISEÑO                  DESPLIEGUE              RECOLECCIÓN
┌──────────┐            ┌──────────┐            ┌──────────────────┐
│ XLSForm  │            │ KPI      │            │ KoboCollect (ODK)│
│ o Form-  │──deploy──▶ │ deploys  │──────────▶ │ Enketo Web       │
│ builder  │            │ to       │            │ API directa      │
│ (UI/API) │            │ openrosa │            └────────┬─────────┘
└──────────┘            └──────────┘                     │
                                                         │ OpenRosa protocol
                                                         │ o API POST
                                                         ▼
 ALMACENAMIENTO          PROCESAMIENTO           CONSUMO
┌──────────────┐        ┌──────────────┐        ┌───────────────────┐
│ PostgreSQL   │        │ formpack     │        │ API v2 JSON       │
│ (submissions)│──────▶ │ (transform)  │──────▶ │ Exports CSV/XLSX  │
│ Media files  │        │              │        │ REST Services     │
│ (filesystem) │        └──────────────┘        │ (webhooks)        │
└──────────────┘                                │ Sync exports      │
                                                │ Power BI / Excel  │
                                                └───────────────────┘
```

### 1.4 Servidores Públicos

| Servidor | URL Base KPI | Región | Uso |
|----------|-------------|--------|-----|
| Global | `https://kf.kobotoolbox.org` | América/Global | Servidor principal gratuito |
| Unión Europea | `https://eu.kobotoolbox.org` | Europa (UE) | Cumplimiento GDPR |
| OCHA (legacy) | `https://kobo.humanitarianresponse.info` | Global | Migrado a kf.kobotoolbox.org |
| Self-hosted | `https://[tu-dominio]` | Variable | Instalaciones privadas |

**Importante:** Los antiguos dominios `kc.kobotoolbox.org` y `kc.humanitarianresponse.info` corresponden a KoboCAT y sus endpoints v1 están deprecados. Toda integración nueva debe usar exclusivamente los dominios `kf.*` o `eu.*` con la API v2.

---

## 2. Arquitectura de la API

### 2.1 Tipo y Diseño

KoboToolbox expone una **API REST JSON** construida sobre **Django REST Framework (DRF)**. La documentación interactiva OpenAPI/Swagger está disponible en:

- Schema OpenAPI (YAML): `https://[server]/api/v2/schema/`
- Schema OpenAPI (JSON): `https://[server]/api/v2/schema/?format=json`
- Interfaz interactiva: `https://[server]/api/v2/docs/`

Características del diseño REST:

- **Content-Type:** `application/json` (principal), soporta `application/xml` en algunos endpoints
- **Formato de respuesta:** JSON por defecto; se puede forzar con `?format=json` o `?format=api` (interfaz browsable DRF)
- **HATEOAS parcial:** Los recursos incluyen URLs absolutas a recursos relacionados (hipermedia)
- **Paginación:** Limit/offset estándar con envolvente `count`/`next`/`previous`
- **Filtrado:** Query parameter `q` con sintaxis propia tipo búsqueda booleana
- **Versionado:** Vía path (`/api/v2/`), la v1 está deprecada

### 2.2 Base URLs y Versionado

```
API v2 (ACTIVA - usar siempre):
  https://kf.kobotoolbox.org/api/v2/
  https://eu.kobotoolbox.org/api/v2/

API v1 (DEPRECADA - decommission Enero 2026):
  https://kf.kobotoolbox.org/api/v1/     ← NO USAR
  https://kc.kobotoolbox.org/api/v1/     ← NO USAR

Documentación interactiva:
  https://kf.kobotoolbox.org/api/v2/docs/

Schema OpenAPI:
  https://kf.kobotoolbox.org/api/v2/schema/?format=json
```

### 2.3 Recursos Principales

```
/api/v2/
├── assets/                              # Formularios, preguntas, bloques, templates
│   ├── {uid}/                           # Asset individual
│   │   ├── data/                        # Submissions (envíos de datos)
│   │   │   └── {submission_id}/         # Submission individual
│   │   ├── exports/                     # Exportaciones asíncronas
│   │   │   └── {export_uid}/            # Exportación individual
│   │   ├── export-settings/             # Config de exports síncronos
│   │   │   └── {setting_uid}/
│   │   │       ├── data.csv             # Descarga síncrona CSV
│   │   │       └── data.xlsx            # Descarga síncrona XLSX
│   │   ├── permission-assignments/      # Permisos del asset
│   │   ├── files/                       # Media files del formulario
│   │   ├── paired-data/                 # Dynamic data attachments
│   │   ├── hooks/                       # REST Services (webhooks)
│   │   │   └── {hook_uid}/
│   │   │       └── logs/                # Logs de envíos webhook
│   │   ├── deployment/                  # Estado de despliegue
│   │   └── versions/                    # Historial de versiones
│   └── ?q=...                           # Filtrado con query parser
├── asset_usage/                         # Uso de almacenamiento
├── permissions/                         # Catálogo de permisos disponibles
├── users/                               # Información de usuarios
│   ├── {username}/
│   └── me/                              # Usuario autenticado actual
└── service_usage/                       # Métricas de uso del servicio
```

### 2.4 Flujo Típico Cliente → Servidor

```
┌────────────┐                              ┌──────────────────┐
│   Cliente   │                              │  KoboToolbox     │
│  (Backend)  │                              │  API v2 (KPI)    │
└──────┬─────┘                              └────────┬─────────┘
       │                                              │
       │  1. GET /api/v2/assets/                      │
       │  Authorization: Token xxxxx                  │
       │─────────────────────────────────────────────▶│
       │                                              │
       │  200 OK {count, next, results: [...]}        │
       │◀─────────────────────────────────────────────│
       │                                              │
       │  2. GET /api/v2/assets/{uid}/                │
       │─────────────────────────────────────────────▶│
       │                                              │
       │  200 OK {uid, name, content, ...}            │
       │◀─────────────────────────────────────────────│
       │                                              │
       │  3. GET /api/v2/assets/{uid}/data/           │
       │     ?limit=100&start=0                       │
       │─────────────────────────────────────────────▶│
       │                                              │
       │  200 OK {count, next, results: [...]}        │
       │◀─────────────────────────────────────────────│
       │                                              │
       │  4. POST /api/v2/assets/{uid}/exports/       │
       │     {source, type, fields_from_all_versions} │
       │─────────────────────────────────────────────▶│
       │                                              │
       │  202 Accepted {uid, status: "created"}       │
       │◀─────────────────────────────────────────────│
       │                                              │
       │  5. GET /api/v2/assets/{uid}/exports/{eid}/  │
       │─────────────────────────────────────────────▶│
       │                                              │
       │  200 OK {status: "complete", result: url}    │
       │◀─────────────────────────────────────────────│
       │                                              │
```

---

## 3. Autenticación Profunda

### 3.1 Método: Token Authentication

KoboToolbox utiliza un sistema de autenticación basado en **Token estático personal** (similar a un API Key). No implementa OAuth 2.0 ni flujos con refresh tokens. Cada usuario tiene un único token persistente asociado a su cuenta.

**Características del token:**

- Es un string alfanumérico de 40 caracteres (hex)
- Permanente hasta que se regenere manualmente
- Un token por usuario (no hay tokens por aplicación o por scope)
- Otorga acceso completo a todos los recursos del usuario (no hay scopes granulares)
- Se envía en el header `Authorization` de cada request

### 3.2 Obtención del Token

**Método 1 — Vía interfaz web:**

1. Iniciar sesión en `https://kf.kobotoolbox.org`
2. Clic en el icono de perfil → ACCOUNT SETTINGS
3. Pestaña Security
4. Clic en "Display" para revelar el API Key

**Método 2 — Vía endpoint directo (navegador autenticado):**

```
GET https://kf.kobotoolbox.org/token/?format=json
```

Respuesta:
```json
{
    "token": "abcdef1234567890abcdef1234567890abcdef12"
}
```

**Método 3 — Vía curl con credenciales:**

```bash
curl -u "mi_usuario:mi_password" \
  "https://kf.kobotoolbox.org/token/?format=json"
```

### 3.3 Uso del Token en Requests

```bash
# Header Authorization (MÉTODO RECOMENDADO)
curl -X GET "https://kf.kobotoolbox.org/api/v2/assets.json" \
  -H "Authorization: Token abcdef1234567890abcdef1234567890abcdef12"
```

**Alternativa — Autenticación básica HTTP:**

```bash
# Basic Auth (funciona pero NO es recomendado para producción)
curl -u "usuario:password" \
  "https://kf.kobotoolbox.org/api/v2/assets.json"
```

### 3.4 Flujo Real de Autenticación

```
┌─────────┐                                    ┌──────────┐
│ Cliente  │                                    │ KPI      │
└────┬────┘                                    └────┬─────┘
     │                                               │
     │  GET /api/v2/assets/                          │
     │  (sin header Authorization)                   │
     │──────────────────────────────────────────────▶│
     │                                               │
     │  401 Unauthorized                             │
     │  {"detail":"Authentication credentials        │
     │   were not provided."}                        │
     │◀──────────────────────────────────────────────│
     │                                               │
     │  GET /api/v2/assets/                          │
     │  Authorization: Token abc123...               │
     │──────────────────────────────────────────────▶│
     │                                               │
     │  ┌─────────────────────────┐                  │
     │  │ Django Auth Backend:    │                  │
     │  │ 1. Extrae token del    │                  │
     │  │    header               │                  │
     │  │ 2. Busca en tabla      │                  │
     │  │    authtoken_token      │                  │
     │  │ 3. Asocia con User     │                  │
     │  │ 4. Verifica is_active  │                  │
     │  └─────────────────────────┘                  │
     │                                               │
     │  200 OK                                       │
     │  {count: 5, results: [...]}                   │
     │◀──────────────────────────────────────────────│
```

### 3.5 Modelo de Permisos

KoboToolbox no tiene scopes de API. Los permisos se gestionan a nivel de **asset** (formulario/proyecto). El catálogo completo de permisos disponible en `/api/v2/permissions/` es:

| Codename | Descripción | Permisos que implica |
|----------|-------------|---------------------|
| `view_asset` | Ver formulario | — |
| `change_asset` | Editar formulario | `view_asset` |
| `manage_asset` | Gestión total | Todos los demás |
| `add_submissions` | Enviar datos | — |
| `view_submissions` | Ver envíos | `view_asset` |
| `change_submissions` | Editar envíos | `view_asset` |
| `delete_submissions` | Eliminar envíos | — |
| `validate_submissions` | Validar envíos | `view_asset` |
| `discover_asset` | Descubrir en listados | — |
| `partial_submissions` | Acceso parcial a datos (row-level) | — |

**Nota importante:** `partial_submissions` es **contradictorio** con `change_submissions` y `view_submissions`. No se pueden asignar simultáneamente. `partial_submissions` permite acceso solo a submissions que cumplan condiciones específicas (filtros row-level).

El usuario `AnonymousUser` es un usuario especial que controla el acceso público. Para hacer un formulario público para envío de datos, se asignan permisos a `AnonymousUser`.

### 3.6 Errores Frecuentes de Autenticación

| Código | Respuesta | Causa | Solución |
|--------|----------|-------|----------|
| 401 | `Authentication credentials were not provided` | Sin header Authorization | Añadir `Authorization: Token xxx` |
| 401 | `Invalid token` | Token incorrecto o regenerado | Verificar token actual en Account Settings |
| 403 | `You do not have permission` | Token válido pero sin permisos sobre el recurso | Verificar permisos del asset con el owner |
| 403 | `Authentication credentials were not provided` | Header malformado | Verificar formato exacto: `Token ` (con espacio) |
| 404 | `Not found` | Asset existe pero no tienes ni `view_asset` | Solicitar permiso al owner |

**Errores sutiles para desarrolladores:**

1. **Confundir `Token` con `Bearer`**: KoboToolbox usa `Authorization: Token xxx`, NO `Authorization: Bearer xxx`.
2. **Trailing slash**: Muchos endpoints requieren trailing slash. `GET /api/v2/assets` puede devolver redirect 301 a `/api/v2/assets/`.
3. **Formato**: Si no añades `?format=json` o `.json`, DRF puede devolver la interfaz HTML browsable en lugar de JSON puro, dependiendo del header `Accept`.

---

## 4. Endpoints Principales — Análisis Profundo

### 4.1 Assets (Formularios/Proyectos)

Los **assets** son el recurso central de KoboToolbox. Un asset puede ser de diferentes tipos: `survey` (formulario), `question`, `block`, `template`, o `collection`.

#### 4.1.1 Listar todos los assets

```
GET /api/v2/assets/
```

**Headers requeridos:**
```
Authorization: Token {tu_token}
Accept: application/json
```

**Parámetros de query:**

| Parámetro | Tipo | Descripción |
|-----------|------|-------------|
| `q` | string | Query de búsqueda booleana (ver filtrado) |
| `limit` | int | Resultados por página (default: ~30) |
| `offset` | int | Desplazamiento para paginación |
| `ordering` | string | Campo de ordenamiento (ej: `-date_modified`) |
| `format` | string | `json` o `api` |

**Request real (curl):**

```bash
curl -X GET "https://kf.kobotoolbox.org/api/v2/assets.json?q=asset_type:survey&limit=10" \
  -H "Authorization: Token abcdef1234567890abcdef1234567890abcdef12"
```

**Response real (estructura):**

```json
{
    "count": 42,
    "next": "https://kf.kobotoolbox.org/api/v2/assets/?limit=10&offset=10",
    "previous": null,
    "results": [
        {
            "url": "https://kf.kobotoolbox.org/api/v2/assets/a4etXeWtqcoodSxLV8a6Uq/",
            "date_created": "2024-01-15T10:30:00Z",
            "date_modified": "2025-02-20T14:22:00Z",
            "date_deployed": "2024-01-16T08:00:00Z",
            "owner": "https://kf.kobotoolbox.org/api/v2/users/project_owner/",
            "owner__username": "project_owner",
            "summary": {
                "geo": false,
                "labels": ["Spanish", "English"],
                "columns": ["type", "name", "label"],
                "lock_all": false,
                "lock_any": false,
                "languages": ["Spanish (es)", "English (en)"],
                "row_count": 35,
                "name_quality": {},
                "naming_conflicts": [],
                "default_translation": "Spanish (es)"
            },
            "has_deployment": true,
            "deployed_version_id": "vKj9eM2nRtCb5aW",
            "deployment__active": true,
            "deployment__submission_count": 1200,
            "deployment__last_submission_time": "2025-02-20T14:22:00Z",
            "permissions": [
                {
                    "url": "https://kf.kobotoolbox.org/api/v2/assets/a4etXeWtqcoodSxLV8a6Uq/permission-assignments/pQngg.../",
                    "user": "https://kf.kobotoolbox.org/api/v2/users/project_owner/",
                    "permission": "https://kf.kobotoolbox.org/api/v2/permissions/add_submissions/",
                    "label": "Add submissions"
                }
            ],
            "tag_string": "salud,encuesta_campo",
            "uid": "a4etXeWtqcoodSxLV8a6Uq",
            "kind": "asset",
            "name": "Encuesta de Salud Comunitaria",
            "asset_type": "survey",
            "version_id": "vKj9eM2nRtCb5aW",
            "version__content_hash": "05be1113c6ae6666...",
            "version_count": 12,
            "settings": {
                "description": "Formulario para recolección de datos de salud",
                "sector": { "value": "Health", "label": "Health" },
                "country": [{ "value": "COL", "label": "Colombia" }]
            },
            "data": "https://kf.kobotoolbox.org/api/v2/assets/a4etXeWtqcoodSxLV8a6Uq/data/"
        }
    ]
}
```

**Sintaxis de filtrado (`q` parameter):**

```
# Formularios del usuario "meg" cuyo nombre contiene "quixotic" (case insensitive)
q=owner__username:meg AND name__icontains:quixotic

# Solo surveys desplegados
q=asset_type:survey AND deployment__active:true

# Assets modificados después de cierta fecha
q=date_modified__gte:2025-01-01

# Operadores soportados:
#   field:value              (exacto)
#   field__icontains:value   (contiene, case insensitive)
#   field__gte:value         (mayor o igual)
#   field__lte:value         (menor o igual)
#   AND / OR                 (operadores lógicos)
```

#### 4.1.2 Obtener un asset individual (detalle completo)

```
GET /api/v2/assets/{uid}/
```

**Request:**
```bash
curl -X GET "https://kf.kobotoolbox.org/api/v2/assets/a4etXeWtqcoodSxLV8a6Uq/" \
  -H "Authorization: Token abc123..."
```

**Response (incluye `content` — la definición completa del formulario):**

```json
{
    "url": "https://kf.kobotoolbox.org/api/v2/assets/a4etXeWtqcoodSxLV8a6Uq/",
    "uid": "a4etXeWtqcoodSxLV8a6Uq",
    "name": "Encuesta de Salud Comunitaria",
    "asset_type": "survey",
    "content": {
        "survey": [
            {
                "type": "start",
                "name": "start",
                "$kuid": "abc123",
                "$autoname": "start"
            },
            {
                "type": "end",
                "name": "end",
                "$kuid": "def456",
                "$autoname": "end"
            },
            {
                "type": "text",
                "name": "nombre_paciente",
                "label": ["Nombre del paciente", "Patient name"],
                "required": true,
                "$kuid": "ghi789",
                "$autoname": "nombre_paciente"
            },
            {
                "type": "integer",
                "name": "edad",
                "label": ["Edad", "Age"],
                "required": true,
                "$kuid": "jkl012",
                "$autoname": "edad"
            },
            {
                "type": "select_one",
                "name": "genero",
                "label": ["Género", "Gender"],
                "select_from_list_name": "lista_genero",
                "$kuid": "mno345",
                "$autoname": "genero"
            },
            {
                "type": "begin_group",
                "name": "sintomas",
                "label": ["Síntomas", "Symptoms"],
                "$kuid": "pqr678"
            },
            {
                "type": "select_multiple",
                "name": "sintomas_principales",
                "label": ["Síntomas principales", "Main symptoms"],
                "select_from_list_name": "lista_sintomas",
                "$kuid": "stu901"
            },
            {
                "type": "end_group",
                "$kuid": "vwx234"
            },
            {
                "type": "geopoint",
                "name": "ubicacion",
                "label": ["Ubicación GPS", "GPS Location"],
                "$kuid": "yza567"
            }
        ],
        "choices": [
            {
                "name": "masculino",
                "label": ["Masculino", "Male"],
                "list_name": "lista_genero",
                "$kuid": "ch001"
            },
            {
                "name": "femenino",
                "label": ["Femenino", "Female"],
                "list_name": "lista_genero",
                "$kuid": "ch002"
            },
            {
                "name": "fiebre",
                "label": ["Fiebre", "Fever"],
                "list_name": "lista_sintomas",
                "$kuid": "ch003"
            },
            {
                "name": "tos",
                "label": ["Tos", "Cough"],
                "list_name": "lista_sintomas",
                "$kuid": "ch004"
            }
        ],
        "settings": {
            "form_title": "Encuesta de Salud Comunitaria",
            "default_language": "Spanish (es)"
        },
        "translated": ["label", "hint"],
        "translations": ["Spanish (es)", "English (en)"]
    },
    "deployment__active": true,
    "deployment__submission_count": 1200
}
```

#### 4.1.3 Crear un nuevo asset

```
POST /api/v2/assets/
```

**Headers:**
```
Authorization: Token {tu_token}
Content-Type: application/json
```

**Body:**
```json
{
    "name": "Mi Nuevo Formulario",
    "asset_type": "survey",
    "content": {
        "survey": [
            {"type": "text", "name": "pregunta_1", "label": ["¿Cuál es tu nombre?"]}
        ],
        "settings": {
            "form_title": "Mi Nuevo Formulario"
        }
    }
}
```

**Response: 201 Created** con el asset completo incluyendo `uid` generado.

#### 4.1.4 Desplegar un formulario

```
POST /api/v2/assets/{uid}/deployment/
```

**Body:**
```json
{
    "active": true
}
```

Para actualizar un despliegue existente:
```
PATCH /api/v2/assets/{uid}/deployment/
```

**Edge cases importantes:**

- No puedes desplegar un asset de tipo `question`, `block` o `template` — solo `survey`
- Redesplegar un formulario con cambios de estructura puede afectar la exportación de datos existentes
- Un formulario puede estar "no activo" (`active: false`) manteniendo los datos recolectados

#### 4.1.5 Clonar un formulario

```
POST /api/v2/assets/{uid}/clone/
```

**Body (opcional):**
```json
{
    "name": "Copia de mi formulario"
}
```

---

### 4.2 Submissions (Envíos de Datos)

Los submissions son los datos recolectados por los formularios desplegados.

#### 4.2.1 Listar submissions de un formulario

```
GET /api/v2/assets/{uid}/data/
```

**Parámetros de query:**

| Parámetro | Tipo | Descripción |
|-----------|------|-------------|
| `limit` | int | Máximo de resultados por página. **Máx: 1,000** (desde Enero 2026). Default: **100** |
| `start` | int | Offset para paginación |
| `sort` | JSON | Ordenamiento estilo MongoDB: `{"_submission_time": -1}` |
| `query` | JSON | Filtro estilo MongoDB: `{"_submission_time": {"$gt": "2025-01-01"}}` |
| `fields` | JSON | Campos específicos a retornar: `["nombre", "edad"]` |
| `format` | string | `json` (default) o `xml` |

**⚠️ CAMBIO CRÍTICO (Enero 2026):** Los límites de paginación del endpoint `/data/` fueron modificados:
- **Máximo por página:** 1,000 registros (antes era 30,000)
- **Default por página:** 100 registros (antes era 30,000)
- Este cambio **NO** afecta los exports síncronos (`/export-settings/{uid}/data.csv|xlsx`)

**Request real:**

```bash
curl -X GET "https://kf.kobotoolbox.org/api/v2/assets/a4etXeWtqcoodSxLV8a6Uq/data.json?limit=100&start=0" \
  -H "Authorization: Token abc123..."
```

**Response real:**

```json
{
    "count": 1200,
    "next": "https://kf.kobotoolbox.org/api/v2/assets/a4etXeWtqcoodSxLV8a6Uq/data/?limit=100&start=100",
    "previous": null,
    "results": [
        {
            "_id": 12345678,
            "nombre_paciente": "María García",
            "edad": "34",
            "genero": "femenino",
            "sintomas/sintomas_principales": "fiebre tos",
            "ubicacion": "4.7110 -74.0721 2640 10",
            "_uuid": "f1e2d3c4-b5a6-9788-90ab-cdef01234567",
            "_submission_time": "2025-02-15T08:30:22",
            "_submitted_by": "enumerator_01",
            "_validation_status": {
                "uid": "validation_status_approved",
                "color": "#00ff00",
                "label": "Approved"
            },
            "_attachments": [
                {
                    "download_url": "https://kf.kobotoolbox.org/api/v2/assets/a4et.../data/12345678/attachments/foto_001.jpg/",
                    "download_small_url": "...",
                    "download_medium_url": "...",
                    "download_large_url": "...",
                    "mimetype": "image/jpeg",
                    "filename": "enumerator_01/attachments/foto_001.jpg",
                    "instance": 12345678,
                    "xform": 474,
                    "id": 98765
                }
            ],
            "_geolocation": [4.7110, -74.0721],
            "_tags": [],
            "_notes": [],
            "_status": "submitted_via_web",
            "_xform_id_string": "a4etXeWtqcoodSxLV8a6Uq",
            "_version_": "vKj9eM2nRtCb5aW",
            "meta/instanceID": "uuid:f1e2d3c4-b5a6-9788-90ab-cdef01234567",
            "formhub/uuid": "f739945244514a6bb304dc35d6049880"
        }
    ]
}
```

**Observaciones críticas sobre la estructura de submissions:**

1. **Todos los campos son strings**: Incluso campos numéricos (`"edad": "34"`), coordenadas GPS, y fechas. Tu código debe parsear los tipos.
2. **Los grupos se representan con `/`**: El campo `sintomas_principales` dentro del grupo `sintomas` aparece como `"sintomas/sintomas_principales"`.
3. **Los select_multiple usan espacios**: Las respuestas múltiples se separan por espacio: `"fiebre tos"`.
4. **Los geopoints son strings**: `"4.7110 -74.0721 2640 10"` = latitud longitud altitud precisión.
5. **Campos de sistema prefijados con `_`**: `_id`, `_uuid`, `_submission_time`, `_submitted_by`, etc.

#### 4.2.2 Obtener un submission individual

```
GET /api/v2/assets/{uid}/data/{submission_id}/
```

#### 4.2.3 Filtrado estilo MongoDB

El parámetro `query` acepta sintaxis JSON similar a MongoDB:

```bash
# Submissions después de una fecha
curl -G "https://kf.kobotoolbox.org/api/v2/assets/{uid}/data/" \
  --data-urlencode 'query={"_submission_time": {"$gt": "2025-01-01T00:00:00"}}' \
  -H "Authorization: Token abc123..."

# Submissions de un enumerador específico
curl -G "https://kf.kobotoolbox.org/api/v2/assets/{uid}/data/" \
  --data-urlencode 'query={"_submitted_by": "enumerator_01"}' \
  -H "Authorization: Token abc123..."

# Combinación con AND implícito
curl -G "https://kf.kobotoolbox.org/api/v2/assets/{uid}/data/" \
  --data-urlencode 'query={"_submission_time": {"$gt": "2025-01-01"}, "edad": {"$gte": "18"}}' \
  -H "Authorization: Token abc123..."

# Operadores disponibles: $gt, $gte, $lt, $lte, $ne, $in, $nin, $exists, $regex
```

#### 4.2.4 Editar un submission existente

```
PATCH /api/v2/assets/{uid}/data/{submission_id}/
```

**Body:**
```json
{
    "nombre_paciente": "María García López"
}
```

#### 4.2.5 Eliminar un submission

```
DELETE /api/v2/assets/{uid}/data/{submission_id}/
```

**Response:** `204 No Content`

#### 4.2.6 Actualizar estado de validación

```
PATCH /api/v2/assets/{uid}/data/{submission_id}/validation_status/
```

**Body:**
```json
{
    "uid": "validation_status_approved"
}
```

Valores posibles:
- `validation_status_not_approved`
- `validation_status_approved`
- `validation_status_on_hold`

#### 4.2.7 Enviar datos vía API (POST submissions)

**Nota importante:** En la API v2, el envío de datos programáticos **no es un endpoint REST estándar POST**. Los datos se envían mediante el protocolo OpenRosa o vía la API de submissions. Esta es una de las áreas menos documentadas de la API.

Para envío programático de datos, la ruta más común en producción es usar el endpoint de submission del protocolo OpenRosa que está integrado en el servidor, o utilizar la interfaz Enketo.

---

### 4.3 Exportaciones

KoboToolbox ofrece dos mecanismos de exportación fundamentalmente diferentes:

#### 4.3.1 Exportaciones Asíncronas

Son trabajos en segundo plano que generan archivos descargables.

**Crear exportación:**

```
POST /api/v2/assets/{uid}/exports/
```

**Headers:**
```
Authorization: Token {tu_token}
Content-Type: application/json
```

**Body:**
```json
{
    "source": "https://kf.kobotoolbox.org/api/v2/assets/a4etXeWtqcoodSxLV8a6Uq/",
    "type": "csv",
    "fields_from_all_versions": true,
    "lang": "Spanish (es)",
    "hierarchy_in_labels": true,
    "group_sep": "/",
    "multiple_select": "both"
}
```

**Opciones disponibles de `type`:** `csv`, `xlsx`, `geojson`, `spss_labels`

**Response: 202 Accepted**
```json
{
    "url": "https://kf.kobotoolbox.org/api/v2/assets/a4et.../exports/eXpOrTuId123/",
    "uid": "eXpOrTuId123",
    "status": "created",
    "date_created": "2025-02-20T14:30:00Z",
    "source": "https://kf.kobotoolbox.org/api/v2/assets/a4et.../",
    "type": "csv",
    "result": null
}
```

**Consultar estado (polling):**

```
GET /api/v2/assets/{uid}/exports/{export_uid}/
```

**Response cuando completa:**
```json
{
    "uid": "eXpOrTuId123",
    "status": "complete",
    "result": "https://kf.kobotoolbox.org/api/v2/assets/a4et.../exports/eXpOrTuId123/data.csv",
    "date_created": "2025-02-20T14:30:00Z"
}
```

**Posibles valores de `status`:** `created`, `processing`, `complete`, `error`

**Descargar el archivo:**
```bash
curl -L "https://kf.kobotoolbox.org/api/v2/assets/{uid}/exports/{eid}/data.csv" \
  -H "Authorization: Token abc123..." \
  -o export_data.csv
```

#### 4.3.2 Exportaciones Síncronas

Proporcionan URLs que siempre devuelven la versión más reciente de los datos. Ideal para dashboards (Power BI, Excel).

**Listar export-settings:**
```
GET /api/v2/assets/{uid}/export-settings/
```

**Response:**
```json
{
    "count": 1,
    "results": [
        {
            "uid": "eSeTtInG1234",
            "url": "https://kf.kobotoolbox.org/api/v2/assets/{uid}/export-settings/eSeTtInG1234/",
            "name": "Reporte mensual",
            "date_modified": "2025-02-01T10:00:00Z",
            "export_settings": {
                "lang": "Spanish (es)",
                "type": "csv",
                "fields": [],
                "group_sep": "/",
                "multiple_select": "both",
                "hierarchy_in_labels": true,
                "fields_from_all_versions": true
            },
            "data_url_csv": "https://kf.kobotoolbox.org/api/v2/assets/{uid}/export-settings/eSeTtInG1234/data.csv",
            "data_url_xlsx": "https://kf.kobotoolbox.org/api/v2/assets/{uid}/export-settings/eSeTtInG1234/data.xlsx"
        }
    ]
}
```

**Descargar datos síncronos:**
```bash
# CSV
curl -L "https://kf.kobotoolbox.org/api/v2/assets/{uid}/export-settings/{setting_uid}/data.csv" \
  -H "Authorization: Token abc123..." \
  -o datos.csv

# XLSX
curl -L "https://kf.kobotoolbox.org/api/v2/assets/{uid}/export-settings/{setting_uid}/data.xlsx" \
  -H "Authorization: Token abc123..." \
  -o datos.xlsx
```

**Limitaciones de exports síncronos:**
- Los datos se actualizan cada **5 minutos** (caché del servidor)
- El export debe completarse en **120 segundos** o fallará
- Proyectos muy grandes pueden exceder el timeout
- Los repeat groups solo se exportan como hojas separadas en XLSX (no en CSV)

---

### 4.4 REST Services (Webhooks)

Los REST Services son el mecanismo de webhook de KoboToolbox. Permiten enviar automáticamente datos a un endpoint externo cuando se recibe una nueva submission.

#### 4.4.1 Listar hooks de un asset

```
GET /api/v2/assets/{uid}/hooks/
```

#### 4.4.2 Crear un hook

```
POST /api/v2/assets/{uid}/hooks/
```

**Body:**
```json
{
    "name": "Webhook a mi servidor",
    "endpoint": "https://mi-servidor.com/api/kobo-webhook",
    "active": true,
    "subset_fields": [],
    "email_notification": true,
    "export_type": "json",
    "auth_level": "basic_auth",
    "settings": {
        "custom_headers": {
            "X-Custom-Header": "mi-valor"
        },
        "username": "mi_usuario",
        "password": "mi_password"
    }
}
```

**Opciones de `export_type`:** `json`, `xml`
**Opciones de `auth_level`:** `no_auth`, `basic_auth`

**Comportamiento del webhook:**

- Se dispara **solo al crear** un nuevo submission (NO en edición ni eliminación)
- En caso de fallo, reintenta 3 veces: a los 60s, 600s, y 6,000s
- Se pueden habilitar notificaciones email por fallos
- Se puede enviar solo un subconjunto de campos (`subset_fields`)

**Limitación crítica:** Los webhooks solo se activan para submissions enviadas desde dispositivos móviles (KoboCollect/Enketo). Los datos editados/limpiados vía la interfaz web de KoboToolbox **NO** disparan el webhook.

#### 4.4.3 Ver logs de un hook

```
GET /api/v2/assets/{uid}/hooks/{hook_uid}/logs/
```

**Response:**
```json
{
    "count": 150,
    "results": [
        {
            "uid": "hLoG123",
            "url": "...",
            "date_modified": "2025-02-20T14:30:00Z",
            "status": 200,
            "status_code": 2,
            "message": "Success"
        }
    ]
}
```

**Valores de `status_code`:** `0` (pending), `1` (failed), `2` (success)

#### 4.4.4 Reintentar envíos fallidos

```
PATCH /api/v2/assets/{uid}/hooks/{hook_uid}/logs/{log_uid}/retry/
```

---

### 4.5 Usuarios y Permisos

#### 4.5.1 Obtener usuario actual

```
GET /api/v2/users/me/
```

**Response:**
```json
{
    "url": "https://kf.kobotoolbox.org/api/v2/users/mi_usuario/",
    "username": "mi_usuario",
    "date_joined": "2023-06-15T10:00:00Z",
    "is_superuser": false,
    "is_active": true,
    "extra_details": {
        "name": "Juan Pérez",
        "organization": "ONG Salud Global",
        "organization_type": "non-profit",
        "sector": "Health",
        "country": "Colombia"
    },
    "git_rev": "2.025.05a"
}
```

#### 4.5.2 Listar permisos de un asset

```
GET /api/v2/assets/{uid}/permission-assignments/
```

**Response:**
```json
{
    "count": 3,
    "results": [
        {
            "url": "https://kf.kobotoolbox.org/api/v2/assets/{uid}/permission-assignments/pAsSiGn1/",
            "user": "https://kf.kobotoolbox.org/api/v2/users/enumerator_01/",
            "permission": "https://kf.kobotoolbox.org/api/v2/permissions/add_submissions/",
            "label": "Add submissions"
        },
        {
            "url": "https://kf.kobotoolbox.org/api/v2/assets/{uid}/permission-assignments/pAsSiGn2/",
            "user": "https://kf.kobotoolbox.org/api/v2/users/supervisor/",
            "permission": "https://kf.kobotoolbox.org/api/v2/permissions/view_submissions/",
            "label": "View submissions"
        }
    ]
}
```

#### 4.5.3 Asignar permisos

```
POST /api/v2/assets/{uid}/permission-assignments/
```

**Body:**
```json
{
    "user": "https://kf.kobotoolbox.org/api/v2/users/nuevo_usuario/",
    "permission": "https://kf.kobotoolbox.org/api/v2/permissions/view_submissions/"
}
```

**Nota:** Los permisos se referencian como URLs absolutas (HATEOAS).

#### 4.5.4 Para hacer público un formulario (acceso anónimo)

```json
{
    "user": "https://kf.kobotoolbox.org/api/v2/users/AnonymousUser/",
    "permission": "https://kf.kobotoolbox.org/api/v2/permissions/add_submissions/"
}
```

### 4.6 Media Files

#### 4.6.1 Listar archivos media de un formulario

```
GET /api/v2/assets/{uid}/files/
```

**Response:**
```json
{
    "count": 2,
    "results": [
        {
            "uid": "aFiLe123",
            "asset": "https://kf.kobotoolbox.org/api/v2/assets/{uid}/",
            "file_type": "form_media",
            "content": "https://kf.kobotoolbox.org/api/v2/assets/{uid}/files/aFiLe123/content/",
            "metadata": {
                "hash": "md5:93fb96bced1e3b392abfc22934afe51a",
                "filename": "logo.jpg",
                "mimetype": "image/jpeg"
            }
        }
    ]
}
```

**Nota:** En v2, solo se soporta `form_media` como tipo de archivo. Otros tipos de metadata de v1 (`doc`, `mapbox_layer`, `source`) no fueron portados.

### 4.7 Uso de Almacenamiento

```
GET /api/v2/asset_usage/
```

Devuelve métricas de uso por asset, incluyendo `storage_bytes`.

---

## 5. Modelado Interno de Datos

### 5.1 Estructura JSON de un Asset (Formulario)

La representación interna de un formulario en KoboToolbox sigue una estructura que refleja el estándar XLSForm pero en formato JSON:

```json
{
    "content": {
        "survey": [
            // Cada fila del worksheet "survey" de un XLSForm
            {
                "type": "begin_group",          // Tipo de pregunta/elemento
                "name": "info_personal",        // Nombre XML del campo
                "label": ["Info Personal"],     // Array por idioma
                "$kuid": "uniqueKoboId",        // ID interno Kobo
                "$autoname": "info_personal",   // Nombre auto-generado
                "appearance": "field-list"       // Atributos de presentación
            },
            {
                "type": "select_one",
                "name": "genero",
                "label": ["Género", "Gender"],   // [idioma1, idioma2]
                "hint": ["Seleccione uno", "Select one"],
                "required": true,
                "select_from_list_name": "genero_opciones",
                "relevant": "${edad} >= 18",     // Lógica condicional
                "constraint": "",
                "constraint_message": "",
                "$kuid": "anotherUniqueId"
            }
        ],
        "choices": [
            // Cada fila del worksheet "choices"
            {
                "list_name": "genero_opciones",
                "name": "masculino",
                "label": ["Masculino", "Male"],
                "$kuid": "choiceKuid1"
            }
        ],
        "settings": {
            "form_title": "Mi Encuesta",
            "version": "v2025.02",
            "default_language": "Spanish (es)",
            "style": "pages"
        },
        "translated": ["label", "hint", "constraint_message"],
        "translations": ["Spanish (es)", "English (en)"]
    }
}
```

### 5.2 Cómo se Almacenan las Respuestas (Submissions)

Las submissions se almacenan en PostgreSQL (a través del componente openrosa/KoboCAT) y se exponen con la siguiente estructura plana:

```
┌─────────────────────────────────────────────────────────────────┐
│                     SUBMISSION JSON                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Campos de sistema (prefijo _):                                 │
│  ├── _id: 12345678                    (PK integer)              │
│  ├── _uuid: "uuid:f1e2..."           (UUID v4 del envío)       │
│  ├── _submission_time: "2025-..."     (ISO 8601 UTC)            │
│  ├── _submitted_by: "user01"          (username)                │
│  ├── _xform_id_string: "a4etXe..."   (asset UID)               │
│  ├── _version_: "vKj9eM..."          (versión del form)        │
│  ├── _status: "submitted_via_web"     (método de envío)         │
│  ├── _geolocation: [lat, lon]         (array si hay GPS)        │
│  ├── _validation_status: {...}        (objeto de validación)    │
│  ├── _attachments: [...]              (archivos adjuntos)       │
│  ├── _tags: []                        (etiquetas)               │
│  └── _notes: []                       (notas)                   │
│                                                                  │
│  Campos del formulario:                                         │
│  ├── nombre_paciente: "María"         (text → string)           │
│  ├── edad: "34"                       (integer → STRING!)       │
│  ├── genero: "femenino"               (select_one → string)     │
│  ├── sintomas/sintomas_principales:   (grupo/campo → "a b c")  │
│  │   "fiebre tos"                                               │
│  ├── ubicacion:                       (geopoint → string)       │
│  │   "4.7110 -74.0721 2640 10"       (lat lon alt precision)   │
│  └── foto: "user/attachments/f.jpg"   (imagen → ruta)          │
│                                                                  │
│  Campos OpenRosa internos:                                      │
│  ├── meta/instanceID: "uuid:..."                                │
│  └── formhub/uuid: "f73994..."                                  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 5.3 Repeat Groups (Grupos Repetitivos)

Los repeat groups se almacenan como arrays anidados dentro de la submission:

```json
{
    "_id": 12345,
    "nombre_hogar": "García Pérez",
    "miembros_familia": [
        {
            "miembros_familia/nombre": "Juan",
            "miembros_familia/edad": "45",
            "miembros_familia/relacion": "jefe_hogar"
        },
        {
            "miembros_familia/nombre": "María",
            "miembros_familia/edad": "42",
            "miembros_familia/relacion": "conyuge"
        }
    ]
}
```

**Implicaciones para exportación:**
- En CSV: los repeat groups NO se incluyen (solo el formulario principal)
- En XLSX: cada repeat group se exporta como una hoja separada
- En JSON API: se incluyen como arrays anidados

### 5.4 Relaciones entre Recursos

```
┌───────────────┐       ┌───────────────────┐
│    User       │──1:N──│     Asset         │
│  (owner)      │       │  (survey/question) │
└───────────────┘       └───────┬───────────┘
                                │
                    ┌───────────┼───────────┐
                    │ 1:N       │ 1:N       │ 1:N
                    ▼           ▼           ▼
            ┌───────────┐ ┌─────────┐ ┌──────────────────┐
            │Submissions│ │ Files   │ │Permission        │
            │  (data)   │ │ (media) │ │Assignments       │
            └─────┬─────┘ └─────────┘ │(user+permission) │
                  │                    └──────────────────┘
                  │ 1:N
                  ▼
            ┌───────────┐
            │Attachments│
            │(multimedia│
            │ files)    │
            └───────────┘

Asset 1:N → Exports (async)
Asset 1:N → Export Settings (sync)
Asset 1:N → Hooks (REST Services)
Asset 1:N → Versions (historial)
Hook  1:N → Hook Logs (resultados de envío)
```

---

## 6. Integraciones Reales

### 6.1 Python — Implementación Completa

```python
"""
KoboToolbox API Client - Implementación de producción
Requiere: pip install requests
"""

import requests
import time
import json
import csv
import io
from typing import Optional, Generator
from datetime import datetime


class KoboAPI:
    """
    Cliente completo para la API v2 de KoboToolbox.
    
    Uso:
        api = KoboAPI(
            base_url="https://kf.kobotoolbox.org",
            token="tu_token_aqui"
        )
        
        # Listar formularios
        forms = api.list_assets(asset_type="survey")
        
        # Obtener submissions con paginación automática
        for submission in api.iter_submissions("a4etXeWtqcoodSxLV8a6Uq"):
            print(submission["_id"])
    """
    
    def __init__(self, base_url: str, token: str, timeout: int = 30):
        self.base_url = base_url.rstrip("/")
        self.api_url = f"{self.base_url}/api/v2"
        self.session = requests.Session()
        self.session.headers.update({
            "Authorization": f"Token {token}",
            "Accept": "application/json",
        })
        self.timeout = timeout
    
    # ──────────────────────────────────────────────────────────────
    # ASSETS (FORMULARIOS)
    # ──────────────────────────────────────────────────────────────
    
    def list_assets(
        self,
        asset_type: str = "survey",
        q: Optional[str] = None,
        limit: int = 100,
        offset: int = 0,
        ordering: str = "-date_modified"
    ) -> dict:
        """
        Lista assets con filtrado y paginación.
        
        Args:
            asset_type: "survey", "question", "block", "template", "collection"
            q: Query de búsqueda (ej: "name__icontains:salud")
            limit: Resultados por página
            offset: Desplazamiento
            ordering: Campo de orden (prefijo - para descendente)
        
        Returns:
            Dict con keys: count, next, previous, results
        """
        params = {
            "limit": limit,
            "offset": offset,
            "ordering": ordering,
            "format": "json",
        }
        
        # Construir query
        query_parts = [f"asset_type:{asset_type}"]
        if q:
            query_parts.append(q)
        params["q"] = " AND ".join(query_parts)
        
        response = self.session.get(
            f"{self.api_url}/assets/",
            params=params,
            timeout=self.timeout
        )
        response.raise_for_status()
        return response.json()
    
    def get_asset(self, uid: str) -> dict:
        """Obtiene detalle completo de un asset incluyendo content."""
        response = self.session.get(
            f"{self.api_url}/assets/{uid}/",
            params={"format": "json"},
            timeout=self.timeout
        )
        response.raise_for_status()
        return response.json()
    
    def get_form_fields(self, uid: str) -> list:
        """
        Extrae la lista de campos del formulario con sus tipos y labels.
        Útil para mapeo de datos.
        """
        asset = self.get_asset(uid)
        content = asset.get("content", {})
        survey = content.get("survey", [])
        translations = content.get("translations", [])
        
        fields = []
        group_stack = []
        
        for row in survey:
            row_type = row.get("type", "")
            
            if row_type == "begin_group" or row_type == "begin_repeat":
                group_stack.append(row.get("name", ""))
                continue
            
            if row_type == "end_group" or row_type == "end_repeat":
                if group_stack:
                    group_stack.pop()
                continue
            
            # Ignorar campos de sistema
            if row_type in ("start", "end", "today", "deviceid",
                           "phonenumber", "username", "audit",
                           "calculate", "note"):
                continue
            
            name = row.get("name", "")
            full_path = "/".join(group_stack + [name]) if group_stack else name
            
            labels = row.get("label", [])
            if isinstance(labels, str):
                labels = [labels]
            
            fields.append({
                "name": name,
                "path": full_path,
                "type": row_type,
                "labels": dict(zip(translations, labels)) if translations else {"default": labels[0] if labels else name},
                "required": row.get("required", False),
                "group": "/".join(group_stack) if group_stack else None,
            })
        
        return fields
    
    def deploy_asset(self, uid: str, active: bool = True) -> dict:
        """Despliega o actualiza el despliegue de un formulario."""
        response = self.session.post(
            f"{self.api_url}/assets/{uid}/deployment/",
            json={"active": active},
            timeout=self.timeout
        )
        if response.status_code == 405:
            # Ya está desplegado, usar PATCH
            response = self.session.patch(
                f"{self.api_url}/assets/{uid}/deployment/",
                json={"active": active},
                timeout=self.timeout
            )
        response.raise_for_status()
        return response.json()
    
    # ──────────────────────────────────────────────────────────────
    # SUBMISSIONS (DATOS)
    # ──────────────────────────────────────────────────────────────
    
    def get_submissions(
        self,
        uid: str,
        limit: int = 100,
        start: int = 0,
        query: Optional[dict] = None,
        fields: Optional[list] = None,
        sort: Optional[dict] = None,
    ) -> dict:
        """
        Obtiene una página de submissions.
        
        Args:
            uid: Asset UID
            limit: Max resultados (máx 1000 desde Ene 2026)
            start: Offset
            query: Filtro MongoDB-style, ej: {"edad": {"$gte": "18"}}
            fields: Lista de campos a retornar
            sort: Ordenamiento, ej: {"_submission_time": -1}
        
        Returns:
            Dict con count, next, previous, results
        """
        params = {
            "limit": min(limit, 1000),  # Respetar límite máximo
            "start": start,
            "format": "json",
        }
        if query:
            params["query"] = json.dumps(query)
        if fields:
            params["fields"] = json.dumps(fields)
        if sort:
            params["sort"] = json.dumps(sort)
        
        response = self.session.get(
            f"{self.api_url}/assets/{uid}/data/",
            params=params,
            timeout=self.timeout
        )
        response.raise_for_status()
        return response.json()
    
    def iter_submissions(
        self,
        uid: str,
        query: Optional[dict] = None,
        fields: Optional[list] = None,
        sort: Optional[dict] = None,
        page_size: int = 1000,
    ) -> Generator[dict, None, None]:
        """
        Iterador que recorre TODAS las submissions con paginación automática.
        
        Uso:
            for sub in api.iter_submissions("uid123"):
                process(sub)
        """
        start = 0
        total = None
        
        while True:
            data = self.get_submissions(
                uid=uid,
                limit=page_size,
                start=start,
                query=query,
                fields=fields,
                sort=sort,
            )
            
            if total is None:
                total = data.get("count", 0)
                if total == 0:
                    return
            
            results = data.get("results", [])
            if not results:
                return
            
            for submission in results:
                yield submission
            
            start += len(results)
            
            if start >= total or data.get("next") is None:
                return
    
    def count_submissions(self, uid: str, query: Optional[dict] = None) -> int:
        """Obtiene solo el conteo sin descargar datos."""
        data = self.get_submissions(uid, limit=1, query=query)
        return data.get("count", 0)
    
    def get_submissions_since(
        self,
        uid: str,
        since: datetime,
        **kwargs
    ) -> Generator[dict, None, None]:
        """
        Obtiene submissions desde una fecha específica.
        Útil para sincronización incremental.
        """
        query = {"_submission_time": {"$gt": since.isoformat()}}
        if "query" in kwargs:
            query.update(kwargs.pop("query"))
        
        return self.iter_submissions(uid, query=query, **kwargs)
    
    # ──────────────────────────────────────────────────────────────
    # EXPORTACIONES
    # ──────────────────────────────────────────────────────────────
    
    def create_export(
        self,
        uid: str,
        export_type: str = "csv",
        lang: Optional[str] = None,
        fields_from_all_versions: bool = True,
        hierarchy_in_labels: bool = True,
        group_sep: str = "/",
    ) -> dict:
        """Crea una exportación asíncrona."""
        payload = {
            "source": f"{self.api_url}/assets/{uid}/",
            "type": export_type,
            "fields_from_all_versions": fields_from_all_versions,
            "hierarchy_in_labels": hierarchy_in_labels,
            "group_sep": group_sep,
        }
        if lang:
            payload["lang"] = lang
        
        response = self.session.post(
            f"{self.api_url}/assets/{uid}/exports/",
            json=payload,
            timeout=self.timeout
        )
        response.raise_for_status()
        return response.json()
    
    def wait_for_export(
        self,
        uid: str,
        export_uid: str,
        poll_interval: int = 3,
        max_wait: int = 300,
    ) -> dict:
        """
        Espera a que una exportación asíncrona se complete.
        
        Returns:
            Export dict con status "complete" y result URL.
        
        Raises:
            TimeoutError: Si excede max_wait segundos.
            RuntimeError: Si la exportación falla.
        """
        elapsed = 0
        while elapsed < max_wait:
            response = self.session.get(
                f"{self.api_url}/assets/{uid}/exports/{export_uid}/",
                params={"format": "json"},
                timeout=self.timeout
            )
            response.raise_for_status()
            export = response.json()
            
            status = export.get("status")
            if status == "complete":
                return export
            elif status == "error":
                raise RuntimeError(
                    f"Export falló: {export.get('messages', 'Sin detalles')}"
                )
            
            time.sleep(poll_interval)
            elapsed += poll_interval
        
        raise TimeoutError(
            f"Export no completó en {max_wait}s. UID: {export_uid}"
        )
    
    def export_and_download(
        self,
        uid: str,
        output_path: str,
        export_type: str = "csv",
        **kwargs
    ) -> str:
        """
        Crea export, espera completar, y descarga el archivo.
        
        Returns:
            Ruta del archivo descargado.
        """
        export = self.create_export(uid, export_type=export_type, **kwargs)
        completed = self.wait_for_export(uid, export["uid"])
        
        result_url = completed["result"]
        response = self.session.get(result_url, stream=True, timeout=120)
        response.raise_for_status()
        
        with open(output_path, "wb") as f:
            for chunk in response.iter_content(chunk_size=8192):
                f.write(chunk)
        
        return output_path
    
    # ──────────────────────────────────────────────────────────────
    # PERMISOS
    # ──────────────────────────────────────────────────────────────
    
    def get_permissions(self, uid: str) -> list:
        """Lista permisos asignados a un asset."""
        response = self.session.get(
            f"{self.api_url}/assets/{uid}/permission-assignments/",
            params={"format": "json"},
            timeout=self.timeout
        )
        response.raise_for_status()
        return response.json().get("results", [])
    
    def assign_permission(
        self,
        uid: str,
        username: str,
        permission_codename: str
    ) -> dict:
        """
        Asigna un permiso a un usuario sobre un asset.
        
        Args:
            uid: Asset UID
            username: Nombre de usuario (o "AnonymousUser" para público)
            permission_codename: ej "view_submissions", "add_submissions"
        """
        payload = {
            "user": f"{self.api_url}/users/{username}/",
            "permission": f"{self.api_url}/permissions/{permission_codename}/",
        }
        response = self.session.post(
            f"{self.api_url}/assets/{uid}/permission-assignments/",
            json=payload,
            timeout=self.timeout
        )
        response.raise_for_status()
        return response.json()
    
    # ──────────────────────────────────────────────────────────────
    # WEBHOOKS (REST SERVICES)
    # ──────────────────────────────────────────────────────────────
    
    def create_webhook(
        self,
        uid: str,
        endpoint_url: str,
        name: str = "Webhook",
        auth_user: Optional[str] = None,
        auth_pass: Optional[str] = None,
        custom_headers: Optional[dict] = None,
    ) -> dict:
        """Crea un REST Service (webhook) para un asset."""
        payload = {
            "name": name,
            "endpoint": endpoint_url,
            "active": True,
            "export_type": "json",
            "auth_level": "basic_auth" if auth_user else "no_auth",
            "settings": {}
        }
        
        if auth_user:
            payload["settings"]["username"] = auth_user
            payload["settings"]["password"] = auth_pass or ""
        
        if custom_headers:
            payload["settings"]["custom_headers"] = custom_headers
        
        response = self.session.post(
            f"{self.api_url}/assets/{uid}/hooks/",
            json=payload,
            timeout=self.timeout
        )
        response.raise_for_status()
        return response.json()


# ══════════════════════════════════════════════════════════════════
# EJEMPLO DE USO COMPLETO
# ══════════════════════════════════════════════════════════════════

if __name__ == "__main__":
    # Configuración
    api = KoboAPI(
        base_url="https://kf.kobotoolbox.org",
        token="TU_TOKEN_AQUI"
    )
    
    # 1. Listar formularios activos
    assets = api.list_assets(asset_type="survey")
    print(f"Total de formularios: {assets['count']}")
    
    for asset in assets["results"]:
        print(f"  [{asset['uid']}] {asset['name']}"
              f" - {asset.get('deployment__submission_count', 0)} envíos")
    
    # 2. Obtener campos del primer formulario
    if assets["results"]:
        first_uid = assets["results"][0]["uid"]
        fields = api.get_form_fields(first_uid)
        print(f"\nCampos de '{assets['results'][0]['name']}':")
        for f in fields:
            print(f"  {f['path']} ({f['type']})")
    
    # 3. Descargar submissions incrementales (desde hace 7 días)
        from datetime import timedelta
        since = datetime.utcnow() - timedelta(days=7)
        
        count = 0
        for sub in api.get_submissions_since(first_uid, since=since):
            count += 1
        print(f"\nSubmissions últimos 7 días: {count}")
    
    # 4. Exportar a CSV
        output = api.export_and_download(
            first_uid,
            output_path="datos_export.csv",
            export_type="csv",
            lang="Spanish (es)"
        )
        print(f"\nArchivo exportado: {output}")
```

### 6.2 Node.js — Implementación Backend Completa

```javascript
/**
 * KoboToolbox API Client - Implementación Node.js para producción
 * Requiere: npm install node-fetch@2 (o usar Node 18+ fetch nativo)
 */

const fetch = require('node-fetch'); // Para Node < 18

class KoboClient {
    /**
     * @param {string} baseUrl - URL base (ej: "https://kf.kobotoolbox.org")
     * @param {string} token - API Token
     */
    constructor(baseUrl, token) {
        this.baseUrl = baseUrl.replace(/\/$/, '');
        this.apiUrl = `${this.baseUrl}/api/v2`;
        this.headers = {
            'Authorization': `Token ${token}`,
            'Accept': 'application/json',
            'Content-Type': 'application/json',
        };
    }

    /**
     * Request genérico con manejo de errores y reintentos.
     */
    async request(endpoint, options = {}, retries = 3) {
        const url = endpoint.startsWith('http') 
            ? endpoint 
            : `${this.apiUrl}${endpoint}`;
        
        const config = {
            headers: this.headers,
            timeout: 30000,
            ...options,
        };

        for (let attempt = 1; attempt <= retries; attempt++) {
            try {
                const response = await fetch(url, config);
                
                if (response.status === 429) {
                    // Rate limit: esperar y reintentar
                    const waitTime = Math.pow(2, attempt) * 1000;
                    console.warn(
                        `Rate limit (429). Reintentando en ${waitTime}ms...`
                    );
                    await this.sleep(waitTime);
                    continue;
                }
                
                if (!response.ok) {
                    const errorBody = await response.text();
                    throw new Error(
                        `HTTP ${response.status}: ${errorBody}`
                    );
                }
                
                // Si la respuesta es un archivo binario
                if (config.responseType === 'buffer') {
                    return await response.buffer();
                }
                
                return await response.json();
            } catch (err) {
                if (attempt === retries) throw err;
                await this.sleep(1000 * attempt);
            }
        }
    }

    sleep(ms) {
        return new Promise(resolve => setTimeout(resolve, ms));
    }

    // ── ASSETS ──────────────────────────────────────────────────

    async listAssets({ assetType = 'survey', limit = 100, offset = 0 } = {}) {
        return this.request(
            `/assets/?q=asset_type:${assetType}&limit=${limit}`
            + `&offset=${offset}&format=json`
        );
    }

    async getAsset(uid) {
        return this.request(`/assets/${uid}/?format=json`);
    }

    // ── SUBMISSIONS ─────────────────────────────────────────────

    async getSubmissions(uid, { limit = 1000, start = 0, query, sort } = {}) {
        let url = `/assets/${uid}/data/?limit=${Math.min(limit, 1000)}`
            + `&start=${start}&format=json`;
        
        if (query) url += `&query=${encodeURIComponent(JSON.stringify(query))}`;
        if (sort)  url += `&sort=${encodeURIComponent(JSON.stringify(sort))}`;
        
        return this.request(url);
    }

    /**
     * Generador asíncrono que itera sobre TODAS las submissions.
     * 
     * Uso:
     *   for await (const sub of client.iterSubmissions(uid)) {
     *       console.log(sub._id);
     *   }
     */
    async *iterSubmissions(uid, { query, sort, pageSize = 1000 } = {}) {
        let start = 0;
        let total = null;

        while (true) {
            const data = await this.getSubmissions(uid, {
                limit: pageSize,
                start,
                query,
                sort,
            });

            if (total === null) {
                total = data.count || 0;
                if (total === 0) return;
            }

            const results = data.results || [];
            if (results.length === 0) return;

            for (const sub of results) {
                yield sub;
            }

            start += results.length;
            if (start >= total || !data.next) return;
        }
    }

    // ── EXPORTACIONES ───────────────────────────────────────────

    async createExport(uid, { type = 'csv', lang, allVersions = true } = {}) {
        return this.request(`/assets/${uid}/exports/`, {
            method: 'POST',
            body: JSON.stringify({
                source: `${this.apiUrl}/assets/${uid}/`,
                type,
                fields_from_all_versions: allVersions,
                hierarchy_in_labels: true,
                group_sep: '/',
                ...(lang && { lang }),
            }),
        });
    }

    async waitForExport(uid, exportUid, { pollInterval = 3000, maxWait = 300000 } = {}) {
        const startTime = Date.now();
        
        while (Date.now() - startTime < maxWait) {
            const exp = await this.request(
                `/assets/${uid}/exports/${exportUid}/?format=json`
            );

            if (exp.status === 'complete') return exp;
            if (exp.status === 'error') {
                throw new Error(`Export falló: ${JSON.stringify(exp)}`);
            }

            await this.sleep(pollInterval);
        }

        throw new Error(`Export timeout después de ${maxWait}ms`);
    }

    async exportAndDownload(uid, options = {}) {
        const exp = await this.createExport(uid, options);
        const completed = await this.waitForExport(uid, exp.uid);
        
        return this.request(completed.result, { responseType: 'buffer' });
    }

    // ── SINCRONIZACIÓN PERIÓDICA ────────────────────────────────

    /**
     * Obtiene submissions nuevas desde la última sincronización.
     * Almacena el timestamp de la última sync para uso incremental.
     * 
     * @param {string} uid - Asset UID
     * @param {string} lastSyncTime - ISO 8601 timestamp
     * @returns {Object} { submissions: [], newSyncTime: string }
     */
    async syncNewSubmissions(uid, lastSyncTime) {
        const query = lastSyncTime 
            ? { _submission_time: { $gt: lastSyncTime } }
            : {};
        
        const submissions = [];
        let latestTime = lastSyncTime;
        
        for await (const sub of this.iterSubmissions(uid, {
            query,
            sort: { _submission_time: 1 },
        })) {
            submissions.push(sub);
            
            if (!latestTime || sub._submission_time > latestTime) {
                latestTime = sub._submission_time;
            }
        }
        
        return {
            submissions,
            count: submissions.length,
            newSyncTime: latestTime,
        };
    }
}

// ═══════════════════════════════════════════════════════════════
// EJEMPLO: Sincronización periódica con base de datos
// ═══════════════════════════════════════════════════════════════

async function runPeriodicSync() {
    const client = new KoboClient(
        'https://kf.kobotoolbox.org',
        'TU_TOKEN_AQUI'
    );
    
    const FORM_UID = 'a4etXeWtqcoodSxLV8a6Uq';
    
    // En producción, leer lastSync de tu base de datos
    let lastSync = null; // Primera vez: obtener todo
    
    // Sincronizar
    const result = await client.syncNewSubmissions(FORM_UID, lastSync);
    
    console.log(`Nuevos registros: ${result.count}`);
    
    for (const sub of result.submissions) {
        // Aquí insertas en tu base de datos
        console.log(`  [${sub._id}] ${sub._submission_time}`);
    }
    
    // Guardar nuevo timestamp para próxima sync
    lastSync = result.newSyncTime;
    console.log(`Próxima sync desde: ${lastSync}`);
}

module.exports = { KoboClient };
```

### 6.3 CURL — Ejemplos Mínimos Reproducibles

```bash
# ═══════════════════════════════════════════════════════════
# Variables de entorno (configurar una vez)
# ═══════════════════════════════════════════════════════════
export KOBO_URL="https://kf.kobotoolbox.org"
export KOBO_TOKEN="abcdef1234567890abcdef1234567890abcdef12"
export KOBO_AUTH="Authorization: Token ${KOBO_TOKEN}"

# ═══════════════════════════════════════════════════════════
# 1. OBTENER TOKEN (con credenciales)
# ═══════════════════════════════════════════════════════════
curl -u "usuario:password" "${KOBO_URL}/token/?format=json"
# {"token":"abcdef1234567890abcdef1234567890abcdef12"}

# ═══════════════════════════════════════════════════════════
# 2. LISTAR FORMULARIOS (solo surveys activos)
# ═══════════════════════════════════════════════════════════
curl -s "${KOBO_URL}/api/v2/assets.json?q=asset_type:survey" \
  -H "${KOBO_AUTH}" | python3 -m json.tool

# ═══════════════════════════════════════════════════════════
# 3. DETALLE DE UN FORMULARIO
# ═══════════════════════════════════════════════════════════
curl -s "${KOBO_URL}/api/v2/assets/a4etXeWtqcoodSxLV8a6Uq/" \
  -H "${KOBO_AUTH}" | python3 -m json.tool

# ═══════════════════════════════════════════════════════════
# 4. OBTENER SUBMISSIONS (primeras 100)
# ═══════════════════════════════════════════════════════════
curl -s "${KOBO_URL}/api/v2/assets/a4etXeWtqcoodSxLV8a6Uq/data.json?limit=100" \
  -H "${KOBO_AUTH}" | python3 -m json.tool

# ═══════════════════════════════════════════════════════════
# 5. SUBMISSIONS FILTRADAS (desde una fecha)
# ═══════════════════════════════════════════════════════════
curl -s -G "${KOBO_URL}/api/v2/assets/a4etXeWtqcoodSxLV8a6Uq/data.json" \
  --data-urlencode 'query={"_submission_time":{"$gt":"2025-02-01T00:00:00"}}' \
  --data-urlencode 'limit=100' \
  -H "${KOBO_AUTH}" | python3 -m json.tool

# ═══════════════════════════════════════════════════════════
# 6. CREAR EXPORTACIÓN ASÍNCRONA (CSV)
# ═══════════════════════════════════════════════════════════
curl -s -X POST "${KOBO_URL}/api/v2/assets/a4etXeWtqcoodSxLV8a6Uq/exports/" \
  -H "${KOBO_AUTH}" \
  -H "Content-Type: application/json" \
  -d '{
    "source": "https://kf.kobotoolbox.org/api/v2/assets/a4etXeWtqcoodSxLV8a6Uq/",
    "type": "csv",
    "fields_from_all_versions": true,
    "hierarchy_in_labels": true,
    "group_sep": "/"
  }'

# ═══════════════════════════════════════════════════════════
# 7. VERIFICAR ESTADO DE EXPORTACIÓN
# ═══════════════════════════════════════════════════════════
curl -s "${KOBO_URL}/api/v2/assets/a4etXeWtqcoodSxLV8a6Uq/exports/eXpOrTuId123/" \
  -H "${KOBO_AUTH}"

# ═══════════════════════════════════════════════════════════
# 8. DESCARGAR EXPORTACIÓN SÍNCRONA (CSV directo)
# ═══════════════════════════════════════════════════════════
curl -L "${KOBO_URL}/api/v2/assets/a4etXeWtqcoodSxLV8a6Uq/export-settings/eSeTtInG1234/data.csv" \
  -H "${KOBO_AUTH}" \
  -o datos.csv

# ═══════════════════════════════════════════════════════════
# 9. ASIGNAR PERMISOS
# ═══════════════════════════════════════════════════════════
curl -s -X POST \
  "${KOBO_URL}/api/v2/assets/a4etXeWtqcoodSxLV8a6Uq/permission-assignments/" \
  -H "${KOBO_AUTH}" \
  -H "Content-Type: application/json" \
  -d '{
    "user": "https://kf.kobotoolbox.org/api/v2/users/enumerator_01/",
    "permission": "https://kf.kobotoolbox.org/api/v2/permissions/add_submissions/"
  }'

# ═══════════════════════════════════════════════════════════
# 10. CREAR WEBHOOK (REST Service)
# ═══════════════════════════════════════════════════════════
curl -s -X POST \
  "${KOBO_URL}/api/v2/assets/a4etXeWtqcoodSxLV8a6Uq/hooks/" \
  -H "${KOBO_AUTH}" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Webhook producción",
    "endpoint": "https://mi-servidor.com/api/kobo-webhook",
    "active": true,
    "export_type": "json",
    "auth_level": "no_auth"
  }'
```

---

## 7. Uso Avanzado

### 7.1 Paginación

**Sistema actual (desde Enero 2026):**

- El endpoint `/api/v2/assets/{uid}/data/` usa paginación `limit`/`start`
- **Default:** 100 resultados por página
- **Máximo:** 1,000 resultados por página
- La respuesta incluye `count` (total), `next` (URL siguiente página), `previous`

**Estrategia de paginación eficiente:**

```python
# PATRÓN RECOMENDADO: Iterar usando "next" URL
def paginate_all(session, initial_url, headers):
    """Sigue la cadena de URLs 'next' hasta agotar resultados."""
    url = initial_url
    all_results = []
    
    while url:
        response = session.get(url, headers=headers)
        data = response.json()
        all_results.extend(data.get("results", []))
        url = data.get("next")  # None cuando no hay más páginas
    
    return all_results
```

**Para el endpoint de assets:**

```
GET /api/v2/assets/?limit=100&offset=200  # offset-based
```

### 7.2 Filtrado Avanzado

**Query parser de assets (`q` parameter):**

```
# Sintaxis: campo__operador:valor
# Operadores: (exact), __icontains, __gte, __lte, __gt, __lt
# Lógicos: AND, OR (en mayúsculas)

# Formularios desplegados del usuario "juan"
q=owner__username:juan AND deployment__active:true

# Formularios con "salud" en el nombre (case insensitive)
q=name__icontains:salud

# Formularios modificados en 2025
q=date_modified__gte:2025-01-01 AND date_modified__lt:2026-01-01
```

**Query de submissions (estilo MongoDB):**

```json
// Operadores soportados
{"campo": "valor"}                    // igualdad exacta
{"campo": {"$gt": "valor"}}           // mayor que
{"campo": {"$gte": "valor"}}          // mayor o igual
{"campo": {"$lt": "valor"}}           // menor que
{"campo": {"$lte": "valor"}}          // menor o igual
{"campo": {"$ne": "valor"}}           // distinto de
{"campo": {"$in": ["a", "b"]}}        // está en lista
{"campo": {"$nin": ["a", "b"]}}       // no está en lista
{"campo": {"$exists": true}}          // campo existe
{"campo": {"$regex": "patrón"}}       // regex (limitado)

// Combinaciones (AND implícito)
{
    "_submission_time": {"$gte": "2025-01-01"},
    "_submitted_by": "enumerator_01",
    "edad": {"$gte": "18"}
}
```

### 7.3 Webhooks y Alternativas

**REST Services (Webhooks nativos):**

Como se detalló en la sección 4.4, los REST Services son el mecanismo webhook de KoboToolbox. Las limitaciones principales:

1. Solo se disparan en **creación** de submissions (no en edición/eliminación)
2. Solo desde envíos móviles (no ediciones web)
3. No soportan firma de payload (HMAC)
4. La autenticación se limita a Basic Auth
5. Reintentos limitados a 3 veces

**Alternativa recomendada: Polling con sincronización incremental**

Para superar las limitaciones de los webhooks, el patrón más robusto en producción es un polling periódico con tracking incremental:

```python
import json
from datetime import datetime
from pathlib import Path

class IncrementalSync:
    """
    Sincronización incremental usando timestamp.
    Más confiable que webhooks para datos limpios/editados.
    """
    
    def __init__(self, api: KoboAPI, state_file: str = "sync_state.json"):
        self.api = api
        self.state_file = Path(state_file)
        self.state = self._load_state()
    
    def _load_state(self) -> dict:
        if self.state_file.exists():
            return json.loads(self.state_file.read_text())
        return {}
    
    def _save_state(self):
        self.state_file.write_text(json.dumps(self.state, indent=2))
    
    def sync_form(self, uid: str, callback) -> int:
        """
        Sincroniza un formulario. Ejecuta callback(submission) para cada nuevo.
        
        Args:
            uid: Asset UID
            callback: Función que recibe cada submission nueva
        
        Returns:
            Número de submissions procesadas
        """
        last_sync = self.state.get(uid, {}).get("last_submission_time")
        
        query = {}
        if last_sync:
            query["_submission_time"] = {"$gt": last_sync}
        
        count = 0
        latest_time = last_sync
        
        for sub in self.api.iter_submissions(
            uid,
            query=query if query else None,
            sort={"_submission_time": 1}
        ):
            callback(sub)
            count += 1
            
            sub_time = sub.get("_submission_time", "")
            if not latest_time or sub_time > latest_time:
                latest_time = sub_time
        
        if latest_time and latest_time != last_sync:
            self.state[uid] = {
                "last_submission_time": latest_time,
                "last_sync": datetime.utcnow().isoformat(),
                "records_synced": count,
            }
            self._save_state()
        
        return count
```

### 7.4 Exportación Automática (Pipeline)

```python
"""Pipeline automatizado de exportación y carga a base de datos."""

import sqlite3
import json

def pipeline_kobo_to_sqlite(
    api: KoboAPI,
    form_uid: str,
    db_path: str = "kobo_data.db"
):
    """
    Pipeline completo: KoboToolbox → SQLite
    
    1. Obtiene la estructura del formulario
    2. Crea la tabla si no existe
    3. Descarga submissions incrementalmente
    4. Inserta en SQLite
    """
    
    # 1. Obtener estructura
    fields = api.get_form_fields(form_uid)
    
    # 2. Conectar y crear tabla
    conn = sqlite3.connect(db_path)
    cursor = conn.cursor()
    
    # Mapeo de tipos Kobo → SQLite
    type_map = {
        "text": "TEXT",
        "integer": "INTEGER",
        "decimal": "REAL",
        "select_one": "TEXT",
        "select_multiple": "TEXT",
        "date": "TEXT",
        "datetime": "TEXT",
        "geopoint": "TEXT",
        "image": "TEXT",
        "audio": "TEXT",
        "video": "TEXT",
        "barcode": "TEXT",
        "note": "TEXT",
    }
    
    columns = [
        "_id INTEGER PRIMARY KEY",
        "_uuid TEXT UNIQUE",
        "_submission_time TEXT",
        "_submitted_by TEXT",
        "_validation_status TEXT",
    ]
    
    safe_names = {}
    for f in fields:
        safe_name = f["path"].replace("/", "__").replace("-", "_")
        safe_names[f["path"]] = safe_name
        sql_type = type_map.get(f["type"], "TEXT")
        columns.append(f'"{safe_name}" {sql_type}')
    
    table_name = f"form_{form_uid[:16]}"
    
    create_sql = f'CREATE TABLE IF NOT EXISTS "{table_name}" ({", ".join(columns)})'
    cursor.execute(create_sql)
    conn.commit()
    
    # 3. Obtener último _id sincronizado
    cursor.execute(
        f'SELECT MAX(_submission_time) FROM "{table_name}"'
    )
    last_time = cursor.fetchone()[0]
    
    query = {}
    if last_time:
        query["_submission_time"] = {"$gt": last_time}
    
    # 4. Insertar submissions
    inserted = 0
    for sub in api.iter_submissions(
        form_uid,
        query=query if query else None,
        sort={"_submission_time": 1}
    ):
        values = {
            "_id": sub.get("_id"),
            "_uuid": sub.get("_uuid"),
            "_submission_time": sub.get("_submission_time"),
            "_submitted_by": sub.get("_submitted_by"),
            "_validation_status": json.dumps(
                sub.get("_validation_status", {})
            ),
        }
        
        for f in fields:
            safe = safe_names[f["path"]]
            values[safe] = sub.get(f["path"])
        
        placeholders = ", ".join(["?"] * len(values))
        cols = ", ".join([f'"{k}"' for k in values.keys()])
        
        cursor.execute(
            f'INSERT OR REPLACE INTO "{table_name}" ({cols}) '
            f'VALUES ({placeholders})',
            list(values.values())
        )
        inserted += 1
        
        if inserted % 100 == 0:
            conn.commit()
    
    conn.commit()
    conn.close()
    
    return inserted
```

### 7.5 Integración con MongoDB

```python
"""Integración directa KoboToolbox → MongoDB."""

from pymongo import MongoClient, UpdateOne

def sync_to_mongodb(
    api: KoboAPI,
    form_uid: str,
    mongo_uri: str = "mongodb://localhost:27017",
    db_name: str = "kobo_data"
):
    """
    Sincroniza submissions de Kobo a MongoDB.
    Usa upsert basado en _id para idempotencia.
    """
    client = MongoClient(mongo_uri)
    db = client[db_name]
    collection = db[f"form_{form_uid[:16]}"]
    
    # Crear índices
    collection.create_index("_id", unique=True)
    collection.create_index("_submission_time")
    collection.create_index("_submitted_by")
    
    # Obtener último timestamp
    last_doc = collection.find_one(
        sort=[("_submission_time", -1)]
    )
    last_time = last_doc["_submission_time"] if last_doc else None
    
    query = {}
    if last_time:
        query["_submission_time"] = {"$gt": last_time}
    
    # Bulk upsert
    operations = []
    for sub in api.iter_submissions(
        form_uid,
        query=query if query else None,
        sort={"_submission_time": 1}
    ):
        operations.append(
            UpdateOne(
                {"_id": sub["_id"]},
                {"$set": sub},
                upsert=True
            )
        )
        
        if len(operations) >= 500:
            collection.bulk_write(operations)
            operations = []
    
    if operations:
        result = collection.bulk_write(operations)
        return result.upserted_count + result.modified_count
    
    return 0
```

---

## 8. Problemas Reales y Limitaciones

### 8.1 Rate Limits

**Estado actual de rate limiting:**

KoboToolbox **no documenta oficialmente rate limits específicos** para los servidores públicos. Sin embargo, en la práctica:

- Los servidores públicos (`kf.kobotoolbox.org`, `eu.kobotoolbox.org`) aplican throttling bajo carga alta
- Las respuestas pueden incluir HTTP 429 (Too Many Requests) sin header `Retry-After` consistente
- Las exportaciones síncronas tienen un caché de 5 minutos que actúa como rate limit implícito
- Las exportaciones asíncronas tienen un timeout de 120 segundos

**Recomendaciones prácticas:**
- Implementar backoff exponencial para reintentos (ver código Node.js en sección 6.2)
- Limitar requests a ~60 por minuto como regla conservadora
- Para bulk data, preferir exports sobre paginación del endpoint `/data/`
- Cachear resultados localmente

### 8.2 Inconsistencias de Documentación

| Problema | Detalle | Solución |
|----------|---------|----------|
| Documentación interactiva vs comportamiento real | La UI de `/api/v2/docs/` puede mostrar endpoints que no funcionan como se describe | Siempre verificar con requests reales |
| Endpoints v1 aún referenciados | Muchos recursos de la comunidad y tutoriales usan v1 | Usar solo v2; consultar guía de migración oficial |
| Formato de envío de datos (POST submissions) | La documentación v2 no explica claramente cómo enviar submissions programáticamente | En la práctica, las submissions se envían vía OpenRosa o Enketo, no directamente al endpoint REST |
| Trailing slashes | Algunos endpoints requieren `/` final, otros no | Siempre incluir trailing slash para evitar redirects 301 |
| Filtro `query` vs `q` | `/assets/` usa `q` (parser propio); `/data/` usa `query` (MongoDB-style) | Son sistemas completamente diferentes |

### 8.3 Errores Habituales de Desarrolladores

**1. Asumir tipos de datos correctos en submissions:**
```json
// INCORRECTO: esperar que "edad" sea integer
{ "edad": 34 }

// CORRECTO: Kobo devuelve TODO como string
{ "edad": "34" }
```

**2. No manejar el cambio de paginación de 2026:**
```python
# INCORRECTO: pedir 30,000 registros como antes
requests.get(f"{url}/data/?limit=30000")  # Devuelve máx 1,000

# CORRECTO: paginar en bloques de 1,000
for page in paginate(url, page_size=1000):
    process(page)
```

**3. Usar `Bearer` en lugar de `Token`:**
```bash
# INCORRECTO
Authorization: Bearer abc123...

# CORRECTO
Authorization: Token abc123...
```

**4. Confundir asset UID con submission ID:**
```
Asset UID:     a4etXeWtqcoodSxLV8a6Uq   (alfanumérico, ~22 chars)
Submission ID: 12345678                   (integer)
```

**5. No seguir redirects en descargas de exports:**
```bash
# INCORRECTO (sin -L, la descarga falla con redirect)
curl "https://kf.kobotoolbox.org/.../data.csv" -o datos.csv

# CORRECTO (seguir redirects)
curl -L "https://kf.kobotoolbox.org/.../data.csv" -o datos.csv
```

**6. Esperar que webhooks capturen ediciones:**

Los REST Services (webhooks) solo se disparan en la creación de nuevas submissions desde móvil/Enketo. Las ediciones realizadas en la interfaz web no disparan el webhook. Para capturar cambios, usar polling con sincronización incremental.

### 8.4 Limitaciones Técnicas Conocidas

| Limitación | Impacto | Alternativa |
|------------|---------|-------------|
| Sin OAuth 2.0 | No hay tokens con scopes, refresh, ni multi-app | Crear cuentas de servicio dedicadas |
| Sin webhooks para ediciones/eliminaciones | Pérdida de sincronía en datos editados | Polling incremental por `_submission_time` |
| Token único por usuario | No se puede revocar acceso de una app sin afectar otras | Cuentas de servicio separadas |
| Submissions como strings | Todo campo es string en JSON | Parsear tipos en el cliente |
| Repeat groups no exportan en CSV | Datos anidados perdidos en CSV | Usar XLSX o JSON API directamente |
| Caché de 5 min en exports síncronos | Datos pueden estar hasta 5 min desactualizados | Aceptar latencia o usar JSON API |
| Export timeout 120s | Formularios grandes pueden fallar | Usar filtros en export settings o JSON API paginada |

---

## 9. Comparativa Técnica

### 9.1 KoboToolbox vs ODK Central vs Google Forms API

| Característica | KoboToolbox API v2 | ODK Central API | Google Forms API |
|---|---|---|---|
| **Tipo de API** | REST JSON (DRF) | REST JSON (OData + custom) | REST JSON (Google API) |
| **Autenticación** | Token estático | Session + App Users + API tokens | OAuth 2.0 + API Keys |
| **Scopes/Granularidad** | Sin scopes (token = acceso total) | Roles por proyecto + App Users para submit-only | OAuth scopes granulares |
| **Formato de formularios** | XLSForm / JSON interno | XLSForm / XForm XML | JSON propietario |
| **Submissions vía API** | Limitado (vía OpenRosa) | REST POST + OpenRosa | Respuestas solo lectura |
| **Paginación** | limit/start (máx 1,000) | OData $skip/$top | pageToken |
| **Filtrado de datos** | MongoDB-style query | OData $filter | Limitado |
| **Webhooks** | REST Services (solo creación) | No nativo (vía OData integrations) | Watches (Push notifications) |
| **Exports** | Async + Sync (CSV/XLSX/GeoJSON) | OData feed + CSV/JSON | Responses como JSON |
| **Repeat groups** | Soporte nativo | Soporte nativo | No soporta |
| **GPS/Geo** | geopoint, geotrace, geoshape | geopoint, geotrace, geoshape | No nativo |
| **Open Source** | Sí (AGPL) | Sí (Apache 2.0) | No |
| **Self-hosting** | Sí (Docker) | Sí (Docker) | No |
| **Rate limits** | No documentado oficialmente | No documentado oficialmente | Cuotas bien definidas |
| **Offline data collection** | KoboCollect + Enketo offline | ODK Collect | Google Forms offline limitado |
| **Entities/Case management** | Dynamic data attachments (limitado) | Entities (robusto) | No |
| **Documentación API** | OpenAPI/Swagger autogenerado | Documentación manual detallada | Excelente (Google standard) |

### 9.2 Cuándo Elegir Cada Opción

**Elegir KoboToolbox cuando:**
- El proyecto es humanitario o de desarrollo con presupuesto limitado
- Se necesita servidor gratuito gestionado
- El equipo prefiere interfaz de diseño de formularios visual
- Se requieren formularios multilenguaje complejos
- El hosting EU es un requisito (GDPR)

**Elegir ODK Central cuando:**
- Se necesita case management longitudinal (Entities)
- Se requiere mayor control sobre el servidor
- La organización tiene capacidad técnica para mantener infraestructura
- Se necesita un modelo de permisos más granular (App Users)

**Elegir Google Forms API cuando:**
- La integración es con ecosistema Google (Sheets, Drive, BigQuery)
- Los formularios son simples (sin GPS, repeat groups, offline)
- Se necesita OAuth 2.0 estándar
- No hay requerimiento de recolección offline

---

## 10. Buenas Prácticas de Ingeniería

### 10.1 Arquitectura Recomendada

```
┌─────────────────────────────────────────────────────────────┐
│                    ARQUITECTURA DE INTEGRACIÓN               │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌─────────────┐     ┌──────────────────────────────────┐   │
│  │ KoboToolbox │     │       Tu Backend                  │   │
│  │ Server      │     │                                    │   │
│  │             │     │  ┌────────────────────────────┐   │   │
│  │  /api/v2/   │◀────│──│  Kobo Sync Service        │   │   │
│  │             │     │  │  (cron: cada 5-15 min)     │   │   │
│  │  webhooks ──│────▶│  │  • Polling incremental     │   │   │
│  │  (push)     │     │  │  • Webhook receiver        │   │   │
│  │             │     │  │  • Deduplicación (_uuid)    │   │   │
│  └─────────────┘     │  └────────────┬───────────────┘   │   │
│                      │               │                    │   │
│                      │               ▼                    │   │
│                      │  ┌────────────────────────────┐   │   │
│                      │  │  Data Transformation       │   │   │
│                      │  │  Layer                      │   │   │
│                      │  │  • Parseo de tipos          │   │   │
│                      │  │  • Normalización grupos     │   │   │
│                      │  │  • Validación de negocio    │   │   │
│                      │  │  • Manejo repeat groups     │   │   │
│                      │  └────────────┬───────────────┘   │   │
│                      │               │                    │   │
│                      │               ▼                    │   │
│                      │  ┌────────────────────────────┐   │   │
│                      │  │  Base de Datos              │   │   │
│                      │  │  (PostgreSQL/MongoDB)       │   │   │
│                      │  │  • Tablas normalizadas      │   │   │
│                      │  │  • Índices por timestamp    │   │   │
│                      │  │  • UUID como clave única    │   │   │
│                      │  └────────────┬───────────────┘   │   │
│                      │               │                    │   │
│                      │               ▼                    │   │
│                      │  ┌────────────────────────────┐   │   │
│                      │  │  API / Dashboard            │   │   │
│                      │  │  • Reportes                 │   │   │
│                      │  │  • Visualizaciones          │   │   │
│                      │  │  • Alertas                  │   │   │
│                      │  └────────────────────────────┘   │   │
│                      │                                    │   │
│                      └──────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### 10.2 Patrones de Diseño Backend

**Patrón 1: Dual ingestion (webhook + polling)**

```python
"""
Combina webhook (baja latencia) con polling (completitud).
El webhook procesa datos en tiempo casi real.
El polling periódico captura ediciones y submissions perdidas.
"""

# Webhook receiver (Flask/FastAPI endpoint)
@app.post("/webhook/kobo")
async def receive_webhook(request: Request):
    data = await request.json()
    submission_uuid = data.get("_uuid")
    
    # Insertar con flag "source=webhook"
    await db.upsert_submission(data, source="webhook")
    
    return {"status": "ok"}

# Polling job (ejecutar cada 15 minutos via cron/scheduler)
async def polling_sync():
    for form_uid in TRACKED_FORMS:
        last_sync = await db.get_last_sync_time(form_uid)
        
        async for sub in api.get_submissions_since(form_uid, last_sync):
            # Upsert captura tanto nuevos como editados
            await db.upsert_submission(sub, source="polling")
        
        await db.update_sync_time(form_uid)
```

**Patrón 2: Idempotencia con `_uuid`**

```python
"""
Usa _uuid como clave de idempotencia.
Permite re-procesar sin duplicados.
"""

def upsert_submission(conn, table, submission):
    """INSERT ... ON CONFLICT UPDATE (idempotente)."""
    uuid = submission["_uuid"]
    
    conn.execute("""
        INSERT INTO submissions (uuid, data, updated_at)
        VALUES (?, ?, NOW())
        ON CONFLICT (uuid) DO UPDATE SET
            data = excluded.data,
            updated_at = NOW()
    """, [uuid, json.dumps(submission)])
```

**Patrón 3: Transformación de tipos**

```python
"""
Transformador de tipos para submissions de Kobo.
Convierte strings a tipos nativos basándose en la definición del formulario.
"""

def transform_submission(submission: dict, field_types: dict) -> dict:
    """
    Args:
        submission: Raw submission de la API
        field_types: Dict de {campo: tipo_kobo}
    """
    result = {}
    
    for key, value in submission.items():
        if key.startswith("_"):
            result[key] = value
            continue
        
        if value is None or value == "":
            result[key] = None
            continue
        
        kobo_type = field_types.get(key, "text")
        
        if kobo_type == "integer":
            result[key] = int(value)
        elif kobo_type == "decimal":
            result[key] = float(value)
        elif kobo_type == "date":
            result[key] = datetime.strptime(value, "%Y-%m-%d").date()
        elif kobo_type == "datetime":
            result[key] = datetime.fromisoformat(value)
        elif kobo_type == "select_multiple":
            result[key] = value.split(" ") if value else []
        elif kobo_type == "geopoint":
            parts = value.split(" ")
            result[key] = {
                "lat": float(parts[0]),
                "lon": float(parts[1]),
                "alt": float(parts[2]) if len(parts) > 2 else None,
                "precision": float(parts[3]) if len(parts) > 3 else None,
            }
        else:
            result[key] = value
    
    return result
```

### 10.3 Lista de Verificación para Producción

```
PRE-DEPLOYMENT CHECKLIST — Integración KoboToolbox
═══════════════════════════════════════════════════

□ Autenticación
  □ Token almacenado de forma segura (env vars o secrets manager)
  □ Cuenta de servicio dedicada (no usar cuenta personal)
  □ Token funciona contra el servidor correcto (kf vs eu)

□ Resiliencia
  □ Reintentos con backoff exponencial implementados
  □ Timeout configurado (30s para API, 120s para exports)
  □ Manejo de HTTP 429 (rate limit)
  □ Manejo de HTTP 502/503/504 (server errors temporales)
  □ Circuit breaker para fallos persistentes

□ Datos
  □ Paginación con límite máximo de 1,000 por página
  □ Parseo de tipos implementado (todos los campos son strings)
  □ Manejo de repeat groups (arrays anidados)
  □ Deduplicación por _uuid
  □ Manejo de submissions sin GPS (_geolocation puede ser [null, null])

□ Sincronización
  □ Estado de sync persistido (último timestamp)
  □ Mecanismo de recovery para syncs fallidas
  □ Dual ingestion si se usan webhooks (webhook + polling)
  □ Validación de que el webhook solo dispara en creación

□ Exports
  □ Seguir redirects (curl -L, requests allow_redirects=True)
  □ Caché de 5 minutos en exports síncronos entendido
  □ Timeout de 120s en exports síncronos manejado

□ Monitoreo
  □ Logging de errores de API
  □ Alertas para sync failures
  □ Métricas de latencia y volumen de datos
  □ Health check del servicio de sincronización
```

---

## Apéndice A: Referencia Rápida de Endpoints

| Acción | Método | Endpoint |
|--------|--------|----------|
| Listar assets | GET | `/api/v2/assets/` |
| Detalle asset | GET | `/api/v2/assets/{uid}/` |
| Crear asset | POST | `/api/v2/assets/` |
| Editar asset | PATCH | `/api/v2/assets/{uid}/` |
| Eliminar asset | DELETE | `/api/v2/assets/{uid}/` |
| Clonar asset | POST | `/api/v2/assets/{uid}/clone/` |
| Desplegar | POST | `/api/v2/assets/{uid}/deployment/` |
| Listar submissions | GET | `/api/v2/assets/{uid}/data/` |
| Detalle submission | GET | `/api/v2/assets/{uid}/data/{id}/` |
| Editar submission | PATCH | `/api/v2/assets/{uid}/data/{id}/` |
| Eliminar submission | DELETE | `/api/v2/assets/{uid}/data/{id}/` |
| Validar submission | PATCH | `/api/v2/assets/{uid}/data/{id}/validation_status/` |
| Crear export (async) | POST | `/api/v2/assets/{uid}/exports/` |
| Estado export | GET | `/api/v2/assets/{uid}/exports/{eid}/` |
| Export settings | GET | `/api/v2/assets/{uid}/export-settings/` |
| Export síncrono CSV | GET | `/api/v2/assets/{uid}/export-settings/{sid}/data.csv` |
| Export síncrono XLSX | GET | `/api/v2/assets/{uid}/export-settings/{sid}/data.xlsx` |
| Listar permisos | GET | `/api/v2/assets/{uid}/permission-assignments/` |
| Asignar permiso | POST | `/api/v2/assets/{uid}/permission-assignments/` |
| Revocar permiso | DELETE | `/api/v2/assets/{uid}/permission-assignments/{pid}/` |
| Listar hooks | GET | `/api/v2/assets/{uid}/hooks/` |
| Crear hook | POST | `/api/v2/assets/{uid}/hooks/` |
| Logs de hook | GET | `/api/v2/assets/{uid}/hooks/{hid}/logs/` |
| Reintentar hook | PATCH | `/api/v2/assets/{uid}/hooks/{hid}/logs/{lid}/retry/` |
| Listar archivos | GET | `/api/v2/assets/{uid}/files/` |
| Versiones | GET | `/api/v2/assets/{uid}/versions/` |
| Catálogo permisos | GET | `/api/v2/permissions/` |
| Usuario actual | GET | `/api/v2/users/me/` |
| Uso de storage | GET | `/api/v2/asset_usage/` |
| Token | GET | `/token/?format=json` |

## Apéndice B: Códigos de Estado HTTP

| Código | Significado en contexto Kobo |
|--------|------------------------------|
| 200 | OK — Recurso obtenido correctamente |
| 201 | Created — Asset/permiso/hook creado |
| 202 | Accepted — Exportación creada (procesando en background) |
| 204 | No Content — Recurso eliminado correctamente |
| 301 | Moved Permanently — Falta trailing slash en URL |
| 400 | Bad Request — JSON malformado o parámetros inválidos |
| 401 | Unauthorized — Token ausente o inválido |
| 403 | Forbidden — Sin permisos sobre el recurso |
| 404 | Not Found — Recurso no existe o sin permiso de descubrimiento |
| 405 | Method Not Allowed — Método HTTP no soportado en endpoint |
| 429 | Too Many Requests — Rate limit alcanzado |
| 500 | Internal Server Error — Error del servidor |
| 502/503/504 | Server errors temporales — Reintentar con backoff |

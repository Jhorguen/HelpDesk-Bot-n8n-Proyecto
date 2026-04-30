# 🤖 HelpDeskBot By Jhorguen V.

> Bot conversacional para la gestión interna de solicitudes de soporte, construido con Telegram, n8n y Google Sheets.

[![Status](https://img.shields.io/badge/status-en%20producción-success)]()
[![Platform](https://img.shields.io/badge/platform-Telegram-blue)]()
[![Engine](https://img.shields.io/badge/engine-n8n%20Community-orange)]()
[![Storage](https://img.shields.io/badge/storage-Google%20Sheets-green)]()
[![Hosting](https://img.shields.io/badge/hosting-Sliplane-purple)]()

## 🌐 Workflow en producción

🔗 **Acceso al workflow desplegado:**
[https://n8nio-n8n-w6p7kg.sliplane.app/workflow/iSH2ZZHWKzl4Si6r](https://n8nio-n8n-w6p7kg.sliplane.app/workflow/iSH2ZZHWKzl4Si6r)

> ⚠️ El acceso al editor de n8n requiere credenciales del propietario.
> El bot público está disponible en Telegram bajo el username `@CRHelpDesk_bot`.

---

## 📋 Tabla de contenidos

- [Descripción](#-descripción)
- [Características](#-características)
- [Arquitectura](#-arquitectura)
- [Stack tecnológico](#-stack-tecnológico)
- [Modelo de datos](#-modelo-de-datos)
- [Flujo conversacional](#-flujo-conversacional)
- [Validaciones](#-validaciones)
- [Instalación y despliegue](#-instalación-y-despliegue)
- [Uso](#-uso)
- [Estructura del workflow](#-estructura-del-workflow)
- [Decisiones de diseño](#-decisiones-de-diseño)
- [Limitaciones y mejoras futuras](#-limitaciones-y-mejoras-futuras)
- [Autor](#-autor)

---

## 📝 Descripción

**HelpDeskBot** es un bot conversacional diseñado para gestionar solicitudes internas de soporte dentro de una organización. Centraliza la atención de incidentes técnicos, solicitudes administrativas y consultas generales mediante un canal único, accesible y amigable a través de Telegram.

El sistema opera bajo un enfoque **conversacional controlado**: no toma decisiones autónomas ni aplica aprendizaje automático. Su función es organizar, registrar y presentar información, guiando al usuario mediante opciones numéricas claras y mensajes humanizados.

---

## ✨ Características

- 🎫 **Creación guiada de tickets** mediante asistente conversacional (wizard)
- 🔍 **Consulta de estado** de solicitudes por ID
- 📋 **Listado personalizado** de solicitudes propias
- 📊 **Reportes agregados** por estado, tipo y prioridad
- 👥 **Control de acceso** por usuario activo/inactivo
- ✅ **Validaciones obligatorias** en cada paso del flujo
- 📜 **Trazabilidad completa** mediante logs de auditoría
- ⚙️ **Operación 24/7** sin necesidad de infraestructura local
- 🔒 **Determinista**: sin IA, todas las decisiones son predecibles

---

## 🏗️ Arquitectura

El sistema sigue una arquitectura de tres capas con flujo de datos unidireccional:

```
┌────────────────┐    Webhook     ┌────────────────┐
│                │ ─────────────► │                │
│   Telegram     │                │      n8n       │
│   (Cliente)    │ ◄───────────── │  (Workflows)   │
│                │   sendMessage  │                │
└────────────────┘                └───────┬────────┘
                                          │
                                          │  API
                                          ▼
                                  ┌────────────────┐
                                  │ Google Sheets  │
                                  │  HelpDeskBot   │
                                  │      _DB       │
                                  └────────────────┘
```

### Capas

| Capa | Componente | Rol |
|------|------------|-----|
| **Presentación** | Telegram | Canal de interacción con el usuario final |
| **Orquestación** | n8n Community | Motor de lógica, validaciones y enrutamiento |
| **Persistencia** | Google Sheets | Almacenamiento de tickets, usuarios, sesiones y logs |

---

## 🛠️ Stack tecnológico

| Componente | Versión | Rol |
|------------|---------|-----|
| **Telegram Bot API** | v6+ | Interfaz de usuario |
| **n8n Community Edition** | latest | Motor de orquestación |
| **Google Sheets API** | v4 | Persistencia de datos |
| **Google Cloud Service Account** | - | Autenticación con Google Sheets |
| **Sliplane** | - | Hosting del contenedor n8n |
| **Docker** | 24+ | Contenedorización |

---

## 💾 Modelo de datos

Documento único en Google Sheets: **`HelpDeskBot_DB`**

### Hoja `SOLICITUDES`

Almacena todos los tickets generados por los usuarios.

| Campo | Tipo | Validación | Descripción |
|-------|------|------------|-------------|
| `id_ticket` | string | Único, auto-generado | Formato: `HD-YYYYMMDD-NNNN` |
| `tipo` | enum | Obligatorio | `Soporte técnico` \| `Solicitud administrativa` \| `Consulta general` |
| `prioridad` | enum | Obligatorio | `Alta` \| `Media` \| `Baja` |
| `descripcion` | string | 10-500 caracteres | Texto libre del problema |
| `estado` | enum | Obligatorio | `Abierto` \| `En proceso` \| `Cerrado` |
| `creado_por` | string | FK a USUARIOS | Telegram user ID del solicitante |
| `fecha_creacion` | datetime | ISO-8601 | Timestamp automático |

### Hoja `USUARIOS`

Define quiénes están autorizados a usar el bot.

| Campo | Tipo | Descripción |
|-------|------|-------------|
| `telegram_user` | string | Identificador único de Telegram (chat_id) |
| `nombre` | string | Nombre completo del colaborador |
| `rol` | enum | `usuario` \| `agente` \| `admin` |
| `activo` | boolean | `TRUE` \| `FALSE` — controla el acceso |

### Hoja `SESIONES`

Almacena el estado conversacional de cada usuario activo.

| Campo | Tipo | Descripción |
|-------|------|-------------|
| `telegram_user` | string | Clave primaria — un usuario, una fila |
| `pantalla_actual` | string | Estado del wizard donde está el usuario |
| `datos_temporales` | JSON | Datos parciales acumulados durante el wizard |
| `ultima_actualizacion` | datetime | Timestamp del último cambio |

### Hoja `LOGS`

Bitácora de auditoría con cada interacción del usuario.

| Campo | Tipo | Descripción |
|-------|------|-------------|
| `timestamp` | datetime | Momento exacto del evento |
| `telegram_user` | string | Usuario que originó el evento |
| `pantalla` | string | Estado del wizard donde ocurrió |
| `opcion` | string | Opción seleccionada o texto enviado |
| `resultado` | enum | `OK` \| `ERROR_VALIDACION` \| `ERROR_SISTEMA` \| `CANCELADO` |

---

## 💬 Flujo conversacional

### Mensaje de bienvenida

```
Hola, soy HelpDeskBot 👋

Estoy aquí para ayudarte con solicitudes de soporte
de forma rápida y ordenada.

Por favor escribe el número de la opción que quieras usar.

Menú principal:
0. Ayuda
1. Crear solicitud
2. Consultar estado de solicitud
3. Mis solicitudes
4. Reportes
5. Configuración
```

### Wizard de creación de solicitud

```
1. Selección de tipo
   ↓
2. Selección de prioridad
   ↓
3. Descripción (10-500 caracteres)
   ↓
4. Confirmación explícita
   ↓
5. Registro en SOLICITUDES
   ↓
6. Mensaje de éxito con ID
```

### Estados del wizard (máquina de estados)

| Estado | Transiciones permitidas |
|--------|------------------------|
| `MENU_PRINCIPAL` | → `CREAR_TIPO` (1) \| `CONSULTAR_TICKET` (2) \| `LISTADO` (3) \| `REPORTES` (4) \| `CONFIG` (5) |
| `CREAR_TIPO` | → `CREAR_PRIORIDAD` (1, 2, 3) \| `MENU_PRINCIPAL` (9) |
| `CREAR_PRIORIDAD` | → `CREAR_DESCRIPCION` (1, 2, 3) \| `MENU_PRINCIPAL` (9) |
| `CREAR_DESCRIPCION` | → `CREAR_CONFIRMACION` (texto válido) \| `MENU_PRINCIPAL` (cancelar) |
| `CREAR_CONFIRMACION` | → `MENU_PRINCIPAL` (1: guarda; 9: descarta) |
| `CONSULTAR_TICKET` | → `MENU_PRINCIPAL` (tras mostrar resultado) |

---

## 🛡️ Validaciones

| Validación | Regla | Acción ante falla |
|------------|-------|-------------------|
| **Usuario activo** | `USUARIOS.activo = TRUE` | Mensaje informativo, sin acceso al menú |
| **Opción de menú** | Coincide con valor del menú actual | Reenvía menú con mensaje de error |
| **Tipo válido** | Valor en `{1, 2, 3, 9}` | Reenvía pantalla `CREAR_TIPO` |
| **Prioridad válida** | Valor en `{1, 2, 3, 9}` | Reenvía pantalla `CREAR_PRIORIDAD` |
| **Descripción válida** | `10 ≤ longitud ≤ 500` | Reenvía pantalla `CREAR_DESCRIPCION` |
| **Confirmación** | Usuario respondió `1` explícitamente | Aborta el guardado |

---

## 🚀 Instalación y despliegue

### Prerrequisitos

- Cuenta de Telegram
- Cuenta de Google con acceso a Google Cloud Console
- Cuenta de Sliplane (o cualquier hosting con Docker)

### Paso 1: Crear el bot en Telegram

1. Abrir chat con `@BotFather` en Telegram
2. Enviar `/newbot` y seguir las instrucciones
3. Guardar el **token** generado de forma segura

### Paso 2: Configurar Google Cloud

1. Crear proyecto en [Google Cloud Console](https://console.cloud.google.com)
2. Habilitar **Google Sheets API** y **Google Drive API**
3. Crear una **Service Account**
4. Generar y descargar la clave JSON
5. Compartir el documento `HelpDeskBot_DB` con el email de la Service Account (rol Editor)

### Paso 3: Preparar Google Sheets

Crear documento `HelpDeskBot_DB` con cuatro hojas:

- `SOLICITUDES` con columnas: `id_ticket | tipo | prioridad | descripcion | estado | creado_por | fecha_creacion`
- `USUARIOS` con columnas: `telegram_user | nombre | rol | activo`
- `SESIONES` con columnas: `telegram_user | pantalla_actual | datos_temporales | ultima_actualizacion`
- `LOGS` con columnas: `timestamp | telegram_user | pantalla | opcion | resultado`

Agregar al menos un usuario en `USUARIOS` con `activo = TRUE`.

### Paso 4: Desplegar n8n en Sliplane

1. Crear servidor en Sliplane
2. Deploy service desde imagen `docker.io/n8nio/n8n:latest`
3. Configurar variables de entorno:

```env
HOST=0.0.0.0
WEBHOOK_URL=https://$SLIPLANE_DOMAIN
N8N_HOST=$SLIPLANE_DOMAIN
N8N_PROTOCOL=https
N8N_SECURE_COOKIE=false
N8N_ENCRYPTION_KEY=<clave-auto-generada>
GENERIC_TIMEZONE=America/Bogota
```

4. Montar volumen persistente en `/home/node/.n8n`
5. Esperar el deploy (2-5 minutos)

### Paso 5: Configurar n8n

1. Acceder a la URL pública del servicio
2. Crear cuenta de propietario
3. Importar el workflow desde `HelpDeskBot_workflow.json`
4. Configurar credenciales:
   - **Telegram API**: pegar el token del bot
   - **Google Service Account**: pegar `client_email` y `private_key` del JSON
5. Asignar credenciales a los nodos correspondientes
6. Verificar que cada nodo de Google Sheets apunte a la hoja correcta
7. **Publish** el workflow

### Paso 6: Configurar webhook de Telegram

n8n registra el webhook automáticamente al publicar. Verificar con:

```
https://api.telegram.org/bot<TOKEN>/getWebhookInfo
```

Debe retornar `ok: true` con la URL de Sliplane.

---

## 📱 Uso

### Para usuarios finales

1. Buscar el bot en Telegram por su username
2. Enviar `/start` para iniciar
3. Seguir las opciones numéricas del menú
4. En cualquier momento, enviar `9` para cancelar el flujo actual

### Comandos disponibles

| Comando | Función |
|---------|---------|
| `/start` | Mostrar menú principal |
| `/menu` | Alias de `/start` |
| `/inicio` | Alias de `/start` |
| `/ayuda` | Mostrar ayuda contextual |

### Para administradores

- Registrar nuevos usuarios directamente en la hoja `USUARIOS` con `activo = TRUE`
- Revocar acceso cambiando `activo = FALSE`
- Revisar la hoja `LOGS` para auditoría
- Cambiar el estado de tickets manualmente en la hoja `SOLICITUDES`

---

## 🧩 Estructura del workflow

El workflow consta de **23 nodos** organizados en grupos funcionales:

### Entrada y normalización
- **Telegram Trigger** — Recibe mensajes del webhook
- **Normalizar Input** — Extrae campos clave del payload
- **Buscar Usuario** — Consulta hoja USUARIOS

### Validación de acceso
- **Validar Usuario Activo** — Determina estado del usuario
- **Switch Acceso** — Enruta según permisos (Activo / Inactivo / No Registrado)

### Mensajes de rechazo
- **Rechazar Inactivo** — Mensaje para cuentas desactivadas
- **Rechazar No Registrado** — Mensaje para usuarios desconocidos

### Estado conversacional
- **Leer Sesión** — Recupera la pantalla actual del usuario
- **Motor Conversacional** ⭐ — Cerebro del bot: máquina de estados

### Enrutamiento de acciones
- **Switch Acción** — Enruta según la acción determinada por el motor

### Rama: Crear Ticket
- **Leer Solicitudes (para ID)** — Lee tickets existentes
- **Generar ID Ticket** — Calcula ID consecutivo del día
- **Guardar Ticket** — Persiste nuevo ticket
- **Preparar Respuesta Éxito** — Construye mensaje de confirmación

### Rama: Consultar Ticket
- **Buscar Ticket** — Busca por ID
- **Formatear Consulta** — Genera mensaje con detalles

### Rama: Mis Solicitudes
- **Leer Mis Solicitudes** — Lee todas las solicitudes
- **Formatear Mis Solicitudes** — Filtra por usuario y formatea listado

### Rama: Reportes
- **Leer Para Reporte** — Lee todas las solicitudes
- **Formatear Reporte** — Agrega por estado, tipo y prioridad

### Salida final
- **Enviar Telegram** — Envía respuesta al usuario
- **Actualizar Sesión** — Guarda nueva pantalla en SESIONES
- **Registrar Log** — Persiste evento en LOGS

---

## 💡 Decisiones de diseño

### ¿Por qué una hoja `SESIONES` adicional?

Telegram es **stateless**: cada mensaje llega aislado sin memoria de los anteriores. Para implementar un wizard de varios pasos, era necesario persistir el estado conversacional. La hoja `SESIONES` actúa como una **máquina de estados finita** que recuerda en qué pantalla está cada usuario.

### ¿Por qué el ID tiene formato `HD-YYYYMMDD-NNNN`?

- **Legible para humanos**: el prefijo `HD` identifica el sistema, la fecha facilita búsquedas
- **Ordenable cronológicamente**: clasificación natural por fecha
- **Sin colisiones diarias**: el consecutivo evita duplicados

### ¿Por qué centralizar la lógica en un único nodo Code?

El **Motor Conversacional** concentra toda la lógica de transiciones de estado, validaciones y generación de mensajes. Esto:
- Facilita mantenimiento (un solo lugar para cambiar comportamiento)
- Mejora legibilidad (la lógica completa se ve junta)
- Simplifica testing (un solo punto de entrada)

### ¿Por qué reordenar el flujo `Telegram → Sesión → Log`?

Originalmente el orden era `Sesión → Telegram → Log`, pero el nodo de Sheets sobrescribe el output, perdiendo `mensaje` y `chat_id`. Reordenar permite que `Enviar Telegram` reciba directamente los datos de los nodos de formateo, manteniendo la integridad del payload.

### ¿Por qué no usar IA?

Por **diseño explícito**. El enunciado pedía un sistema determinista. La IA introduciría variabilidad innecesaria, alucinaciones y latencia. Las opciones numéricas garantizan claridad y previsibilidad para el usuario.

---

## ⚠️ Limitaciones y mejoras futuras

### Limitaciones actuales

- ❌ Sin lógica diferenciada por rol (todos los usuarios activos pueden hacer todo)
- ❌ No hay edición de tickets desde el bot (solo desde Sheets)
- ❌ Reportes básicos (sin filtros temporales)
- ❌ Sin notificaciones a un canal de agentes
- ❌ Concurrencia básica (last-write-wins en Sheets)
- ❌ Sin paginación para usuarios con muchos tickets

### Mejoras futuras propuestas

- 🔄 Migrar persistencia a **PostgreSQL** para mejor rendimiento
- 🚀 Agregar **Redis** para cache de sesiones (latencia <10ms)
- 👥 Implementar **roles diferenciados** (agente puede tomar tickets, admin ve todo)
- 📨 **Notificaciones a canal de agentes** cuando se crea un ticket
- 📊 **Reportes avanzados** con filtros por fecha, prioridad, etc.
- 🌐 **Dashboard web** para gestión visual
- 🔐 **2FA** para acciones administrativas
- 📈 **Métricas y observabilidad** (Prometheus + Grafana)
- 🤖 **Capa opcional de IA** para clasificación automática del tipo

---

## 👤 Autor

**Jhorguen V.**

---

<p align="center">
  Hecho con ❤️ usando Telegram + n8n + Google Sheets 
</p>

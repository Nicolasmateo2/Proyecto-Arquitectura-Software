# SERVI-RED — App Móvil Ciudadana
## Documentación Técnica del Módulo Frontend Móvil

> Sistema de recolección participativa de datos sobre servicios públicos domiciliarios en Colombia.
> **SSPD · CRA · Ciudadanos · Operadores**

---

## Estructura del Proyecto

```
Proyecto-Arquitectura-Software/
├── app/                          ← Frontend móvil (HTML/CSS/JS)
│   ├── index.html               ← App principal (6 pantallas)
│   ├── styles.css               ← Sistema de diseño completo
│   ├── supabase-client.js       ← Cliente API + Cola Offline (ADR-01)
│   ├── app.js                   ← Router + controlador principal
│   └── modules/
│       ├── reporte.js           ← RF-01, RF-02, RF-03 (Persona 2)
│       └── extras.js            ← RF-04, RF-05, RF-06, Módulo G (Persona 3)
│
└── backend/                     ← Capa Supabase
    ├── schema.sql               ← PostgreSQL + PostGIS + RLS + RPC
    ├── SERVI-RED.postman_collection.json
    └── edge-functions/
        └── process-notifications/
            └── index.ts         ← TypeScript (Deno) — ADR-03
```

---

## Decisiones de Arquitectura Implementadas

| ADR | Descripción | Implementación |
|-----|-------------|----------------|
| **ADR-01** | Supabase + PostgreSQL + PostGIS | `supabase-client.js`, `schema.sql` |
| **ADR-02** | RLS sin autenticación obligatoria | Políticas en `schema.sql` + formularios sin login |
| **ADR-03** | Notificaciones asíncronas (Outbox) | `edge-functions/process-notifications/index.ts` |
| **ADR-05** | OpenStreetMap + Leaflet | Mapas en pantallas Reporte, Comunidad, Denuncia |
| **ADR-06** | Web Mobile-First (MVP) → Kotlin | App HTML responsive como prototipo funcional |
| **RNF-22** | Idempotencia con `client_request_id` | UUID en cada petición POST |
| **Rest-4** | Cola offline con reintentos automáticos | `OfflineQueue` en `supabase-client.js` |

---

## Pantallas y Módulos Implementados

### 🏠 Home
- Hero banner con llamada a la acción
- Acciones rápidas a los 4 módulos principales
- Lista de reportes recientes con estado en tiempo real
- Módulo de reporte por voz (Web Speech API)

### 📢 Módulo B — Reporte de Cortes (RF-01) · Persona 2
- Selector visual de tipo de servicio (chips interactivos)
- Formulario con descripción, hora de inicio, magnitud del área
- Captura de foto (adjunta a Supabase Storage)
- Mapa Leaflet + OpenStreetMap con captura GPS automática
- Idempotencia garantizada con `client_request_id` (UUID)
- Cola offline: si no hay red, guarda localmente y reintenta
- Modal de confirmación con ID de trazabilidad único

### 🗺️ Módulo C — Validación Comunitaria (RF-02, RF-03) · Persona 2
- Mapa con marcadores de reportes activos en la zona
- Botón "Confirmar" para validación comunitaria
- Botón "Ya volvió" para notificar restablecimiento
- Cálculo automático de duración al superar umbral de confirmaciones

### 🚨 Módulo D — Denuncias Anónimas (RF-04) · Persona 3
- Formulario sin ningún dato personal (Zero PII)
- Tipo de fraude, servicio, municipio, descripción
- Mapa con ubicación aproximada (redondeada ±1km)
- Cola de moderación antes de transmitir al operador
- Confirmación con referencia interna (sin datos identificables)

### 🔍 Módulo E — Seguimiento por ID (RF-05) · Persona 3
- Búsqueda por ID de trazabilidad sin login
- Timeline visual: Recibido → En Atención → Resuelto
- IDs de demostración para testing rápido
- Suscripción automática a Supabase Realtime (WebSocket)

### 🔔 Módulo F — Notificaciones (RF-06, ADR-03) · Persona 3
- Edge Function TypeScript que procesa la tabla `outbox_eventos`
- Envío push via Firebase Cloud Messaging (FCM)
- In-app via Supabase Realtime
- Resiliencia: fallos de push no revierten reportes
- Máximo 5 reintentos por evento con log de errores

### 🏅 Módulo G — Gamificación + Voz · Persona 3
- Insignias cívicas (Primer Reporte, Guardián Comunitario, etc.)
- Estadísticas de participación del ciudadano
- Reporte guiado por voz (Web Speech API, idioma: es-CO)
- Detección automática de tipo de servicio por palabras clave

---

## Cómo Conectar el Backend Real (Supabase)

1. Crear proyecto en [supabase.com](https://supabase.com)
2. Ejecutar `backend/schema.sql` en el SQL Editor de Supabase
3. Habilitar PostGIS: `CREATE EXTENSION IF NOT EXISTS postgis;`
4. Editar `app/supabase-client.js`:
   ```js
   const SUPABASE_URL = 'https://TU_PROJECT_ID.supabase.co';
   const SUPABASE_ANON_KEY = 'TU_ANON_KEY';
   ```
5. Importar `backend/SERVI-RED.postman_collection.json` en Postman para validar los endpoints
6. Deploy de la Edge Function:
   ```bash
   supabase functions deploy process-notifications
   supabase functions schedule process-notifications "*/2 * * * *"
   ```
7. Configurar variables de entorno en Supabase Dashboard:
   - `SUPABASE_SERVICE_ROLE_KEY`
   - `FCM_SERVER_KEY` (de Firebase Console)

---

## Cómo Ejecutar la App Localmente

Simplemente abrir `app/index.html` en cualquier navegador moderno.  
No requiere servidor ni compilación — funciona en modo demostración completo.

Para testing con servidor local:
```bash
npx serve app/
# o
python -m http.server 8080 --directory app/
```

---

## Colección Postman — Endpoints incluidos

| # | Endpoint | Método | Módulo |
|---|----------|--------|--------|
| 1 | `POST /rest/v1/reportes` | POST | RF-01 |
| 2 | `GET /rest/v1/reportes?id=eq.{id}` | GET | RF-05 |
| 3 | `POST /rpc/get_reportes_cercanos` | POST | RF-03 (PostGIS) |
| 4 | `POST /rpc/confirmar_reporte` | POST | RF-03 |
| 5 | `POST /rpc/notificar_restablecimiento` | POST | RF-02 |
| 6 | `POST /rest/v1/denuncias_anonimas` | POST | RF-04 |
| 7 | `GET /rest/v1/outbox_eventos` | GET | RF-06 Monitor |
| 8 | `POST /functions/v1/process-notifications` | POST | RF-06 Edge |
| 9 | Prueba de idempotencia con mismo UUID | POST | RNF-22 |

---

## Tecnologías Utilizadas

- **Frontend:** HTML5, CSS3 (Variables CSS, Glassmorphism), Vanilla JavaScript ES2022
- **Mapas:** Leaflet.js 1.9.4 + OpenStreetMap (ADR-05)
- **Backend:** Supabase (PostgreSQL 15 + PostGIS) — ADR-01
- **Notificaciones:** Firebase Cloud Messaging + Supabase Realtime — RF-06
- **Serverless:** Deno / TypeScript Edge Functions — ADR-03
- **Fonts:** Inter + Space Grotesk (Google Fonts)
- **Voz:** Web Speech API (SpeechRecognition, idioma es-CO)

---

*SERVI-RED v1.0 — MVP Módulo Ciudadano · Arquitectura de Software 2024*

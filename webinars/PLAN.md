# Plan: Plataforma de Webinars sobre FOSSStudio

## 1. Estado actual del proyecto

FOSSStudio es un estudio de video-podcast auto-hospedado con:

- **mediasoup (SFU WebRTC)** — hasta 10 participantes simultáneos
- **Streaming RTMP** — composición server-side con ffmpeg hacia YouTube
- **Grabación** — multi-track y composited
- **Soundboard + Intro videos** — inserción fullscreen en el stream
- **Overlays** — banners con nombre, lower-thirds, logo, wallpaper
- **Noise suppression** — RNNoise WASM
- **Roles** — admin / subadmin con autenticación y TOTP
- **Email** — SMTP para invitaciones
- **Almacenamiento** — archivos JSON planos (`storage.js`)
- **Salas en memoria** — estado volátil en `rooms.js` (Map)

### Lo que ya funciona para webinars

| Capacidad | Estado |
|---|---|
| Múltiples invitados via WebRTC | ✅ Hasta 10 |
| Streaming a YouTube (RTMP) | ✅ Composición server-side |
| Layouts (grid / spotlight) | ✅ |
| Banners / overlays | ✅ |
| Intro videos fullscreen | ✅ |
| Grabación | ✅ |
| Autenticación con roles | ✅ |
| Invitaciones por email | ✅ |

### Lo que falta

| Capacidad | Estado |
|---|---|
| Base de datos relacional | ❌ |
| Escenas preconfiguradas | ❌ |
| Video de espera (holding screen) | ❌ |
| Screen sharing | ❌ |
| Backstage / green room | ❌ |
| Programación de webinars | ❌ |
| Página de registro público | ❌ |
| Chat / Q&A para audiencia | ❌ |
| Polls / encuestas en vivo | ❌ |

---

## 2. Base de datos relacional

### Por qué migrar de JSON plano a BD relacional

- Las escenas, webinars programados, registros de asistentes y polls necesitan relaciones (1:N, N:N)
- Consultas como "próximos webinars", "registros por evento", "resultados de encuesta" son costosas en JSON plano
- Concurrencia segura: múltiples procesos leyendo/escribiendo al mismo archivo JSON es frágil
- Migraciones versionadas para evolucionar el esquema sin perder datos

### Motor elegido: **SQLite + better-sqlite3**

**Justificación:**
- Cero dependencias externas (no necesita PostgreSQL/MySQL server)
- Consistente con la filosofía auto-hospedada del proyecto (un solo binario/servicio)
- better-sqlite3 es síncrono y muy rápido para volúmenes bajos-medios
- WAL mode para lecturas concurrentes
- Si en el futuro se necesita PostgreSQL, el esquema SQL es portable

### Esquema de base de datos

```sql
-- ============================================================
-- Usuarios (migración desde users.json)
-- ============================================================
CREATE TABLE users (
  id            TEXT PRIMARY KEY,  -- UUID
  username      TEXT NOT NULL UNIQUE,
  email         TEXT,
  role          TEXT NOT NULL DEFAULT 'subadmin',  -- 'admin' | 'subadmin'
  password_hash TEXT,
  totp_enabled  INTEGER NOT NULL DEFAULT 0,
  totp_secret   TEXT,
  invite_token  TEXT,
  invite_expires INTEGER,  -- unix ms
  allow_server_recording INTEGER NOT NULL DEFAULT 0,
  created_at    INTEGER NOT NULL DEFAULT (unixepoch() * 1000)
);

-- ============================================================
-- Configuración por usuario (ex user.settings)
-- ============================================================
CREATE TABLE user_settings (
  user_id     TEXT PRIMARY KEY REFERENCES users(id) ON DELETE CASCADE,
  wallpaper   TEXT,
  bg          TEXT,
  logo        TEXT,
  stream_url  TEXT NOT NULL DEFAULT 'rtmp://a.rtmp.youtube.com/live2',
  stream_key  TEXT NOT NULL DEFAULT '',
  updated_at  INTEGER NOT NULL DEFAULT (unixepoch() * 1000)
);

-- ============================================================
-- Soundboard clips y intro videos (1:N desde user)
-- ============================================================
CREATE TABLE media_assets (
  id        TEXT PRIMARY KEY,
  user_id   TEXT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  type      TEXT NOT NULL,  -- 'sound' | 'intro' | 'waiting_video' | 'scene_media'
  name      TEXT NOT NULL,
  ext       TEXT NOT NULL,
  file_path TEXT,
  created_at INTEGER NOT NULL DEFAULT (unixepoch() * 1000)
);

-- ============================================================
-- Sesiones / Shows (migración desde sessions.json)
-- ============================================================
CREATE TABLE sessions (
  id        TEXT PRIMARY KEY,
  owner_id  TEXT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  name      TEXT NOT NULL,
  slug      TEXT NOT NULL UNIQUE,
  created_at INTEGER NOT NULL DEFAULT (unixepoch() * 1000)
);

-- ============================================================
-- Escenas preconfiguradas
-- ============================================================
CREATE TABLE scenes (
  id          TEXT PRIMARY KEY,
  session_id  TEXT NOT NULL REFERENCES sessions(id) ON DELETE CASCADE,
  name        TEXT NOT NULL,          -- "Solo presentador", "Panel", etc.
  layout_type TEXT NOT NULL,          -- 'single' | 'grid' | 'spotlight' | 'screen_share' | 'picture_in_picture' | 'title_card' | 'custom'
  sort_order  INTEGER NOT NULL DEFAULT 0,
  config      TEXT NOT NULL DEFAULT '{}',  -- JSON con parámetros específicos del layout
  is_default  INTEGER NOT NULL DEFAULT 0,
  created_at  INTEGER NOT NULL DEFAULT (unixepoch() * 1000)
);

-- Qué participantes aparecen en cada escena (y en qué posición)
CREATE TABLE scene_slots (
  id        TEXT PRIMARY KEY,
  scene_id  TEXT NOT NULL REFERENCES scenes(id) ON DELETE CASCADE,
  slot_index INTEGER NOT NULL,        -- posición en el layout (0, 1, 2...)
  role      TEXT NOT NULL DEFAULT 'any',  -- 'host' | 'guest' | 'screen' | 'any'
  peer_id   TEXT,                     -- NULL = se asigna dinámicamente
  label     TEXT,                     -- etiqueta del slot
  config    TEXT NOT NULL DEFAULT '{}',
  UNIQUE(scene_id, slot_index)
);

-- ============================================================
-- Webinars programados
-- ============================================================
CREATE TABLE webinars (
  id              TEXT PRIMARY KEY,
  session_id      TEXT NOT NULL REFERENCES sessions(id) ON DELETE CASCADE,
  title           TEXT NOT NULL,
  description     TEXT,
  slug            TEXT NOT NULL UNIQUE,  -- URL pública
  scheduled_at    INTEGER NOT NULL,      -- unix ms
  duration_minutes INTEGER DEFAULT 60,
  timezone        TEXT DEFAULT 'America/Mexico_City',
  status          TEXT NOT NULL DEFAULT 'draft',  -- 'draft' | 'scheduled' | 'live' | 'ended' | 'cancelled'
  waiting_video_id TEXT REFERENCES media_assets(id),
  landing_image   TEXT,                  -- ruta a imagen de portada
  rtmp_url        TEXT,
  stream_key      TEXT,
  max_attendees   INTEGER DEFAULT 0,     -- 0 = sin límite
  require_registration INTEGER NOT NULL DEFAULT 1,
  created_at      INTEGER NOT NULL DEFAULT (unixepoch() * 1000),
  updated_at      INTEGER NOT NULL DEFAULT (unixepoch() * 1000)
);

-- Escenas asignadas a un webinar (orden de uso)
CREATE TABLE webinar_scenes (
  webinar_id TEXT NOT NULL REFERENCES webinars(id) ON DELETE CASCADE,
  scene_id   TEXT NOT NULL REFERENCES scenes(id) ON DELETE CASCADE,
  sort_order INTEGER NOT NULL DEFAULT 0,
  PRIMARY KEY (webinar_id, scene_id)
);

-- ============================================================
-- Registro de asistentes
-- ============================================================
CREATE TABLE registrations (
  id          TEXT PRIMARY KEY,
  webinar_id  TEXT NOT NULL REFERENCES webinars(id) ON DELETE CASCADE,
  name        TEXT NOT NULL,
  email       TEXT NOT NULL,
  company     TEXT,
  registered_at INTEGER NOT NULL DEFAULT (unixepoch() * 1000),
  reminded    INTEGER NOT NULL DEFAULT 0,  -- ya se le envió recordatorio
  attended    INTEGER NOT NULL DEFAULT 0,
  UNIQUE(webinar_id, email)
);

-- ============================================================
-- Panelistas / invitados del webinar
-- ============================================================
CREATE TABLE panelists (
  id          TEXT PRIMARY KEY,
  webinar_id  TEXT NOT NULL REFERENCES webinars(id) ON DELETE CASCADE,
  name        TEXT NOT NULL,
  email       TEXT NOT NULL,
  role        TEXT NOT NULL DEFAULT 'guest',  -- 'host' | 'co-host' | 'guest'
  invite_token TEXT,
  accepted    INTEGER NOT NULL DEFAULT 0,
  join_url    TEXT,
  UNIQUE(webinar_id, email)
);

-- ============================================================
-- Polls / encuestas en vivo
-- ============================================================
CREATE TABLE polls (
  id          TEXT PRIMARY KEY,
  webinar_id  TEXT NOT NULL REFERENCES webinars(id) ON DELETE CASCADE,
  question    TEXT NOT NULL,
  type        TEXT NOT NULL DEFAULT 'single',  -- 'single' | 'multiple'
  status      TEXT NOT NULL DEFAULT 'draft',   -- 'draft' | 'open' | 'closed'
  show_results INTEGER NOT NULL DEFAULT 0,     -- mostrar en overlay
  created_at  INTEGER NOT NULL DEFAULT (unixepoch() * 1000)
);

CREATE TABLE poll_options (
  id       TEXT PRIMARY KEY,
  poll_id  TEXT NOT NULL REFERENCES polls(id) ON DELETE CASCADE,
  label    TEXT NOT NULL,
  sort_order INTEGER NOT NULL DEFAULT 0
);

CREATE TABLE poll_votes (
  id          TEXT PRIMARY KEY,
  option_id   TEXT NOT NULL REFERENCES poll_options(id) ON DELETE CASCADE,
  voter_id    TEXT,  -- registration_id o anónimo
  voted_at    INTEGER NOT NULL DEFAULT (unixepoch() * 1000)
);

-- ============================================================
-- Chat / Q&A
-- ============================================================
CREATE TABLE chat_messages (
  id          TEXT PRIMARY KEY,
  webinar_id  TEXT NOT NULL REFERENCES webinars(id) ON DELETE CASCADE,
  sender_name TEXT NOT NULL,
  sender_type TEXT NOT NULL DEFAULT 'attendee',  -- 'host' | 'panelist' | 'attendee'
  message     TEXT NOT NULL,
  is_question INTEGER NOT NULL DEFAULT 0,  -- marcado como Q&A
  is_answered INTEGER NOT NULL DEFAULT 0,
  pinned      INTEGER NOT NULL DEFAULT 0,
  created_at  INTEGER NOT NULL DEFAULT (unixepoch() * 1000)
);

-- ============================================================
-- Índices para consultas frecuentes
-- ============================================================
CREATE INDEX idx_webinars_status ON webinars(status);
CREATE INDEX idx_webinars_scheduled ON webinars(scheduled_at);
CREATE INDEX idx_registrations_webinar ON registrations(webinar_id);
CREATE INDEX idx_registrations_email ON registrations(email);
CREATE INDEX idx_chat_webinar ON chat_messages(webinar_id);
CREATE INDEX idx_polls_webinar ON polls(webinar_id);
CREATE INDEX idx_scenes_session ON scenes(session_id);
CREATE INDEX idx_media_user ON media_assets(user_id, type);
```

### Diagrama de relaciones

```
users 1──N sessions
users 1──N user_settings (1:1)
users 1──N media_assets

sessions 1──N scenes
sessions 1──N webinars

scenes 1──N scene_slots
scenes N──N webinars (via webinar_scenes)

webinars 1──N registrations
webinars 1──N panelists
webinars 1──N polls
webinars 1──N chat_messages
webinars 1──1 media_assets (waiting_video)

polls 1──N poll_options
poll_options 1──N poll_votes
```

---

## 3. Arquitectura de la solución

### Componentes nuevos

```
server/
├── src/
│   ├── db.js              ← Inicialización SQLite + migraciones
│   ├── migrations/
│   │   ├── 001_initial.sql
│   │   ├── 002_scenes.sql
│   │   ├── 003_webinars.sql
│   │   └── ...
│   ├── models/
│   │   ├── user.js        ← Reemplaza users.js (queries SQL)
│   │   ├── session.js     ← Reemplaza parte de sessions en storage.js
│   │   ├── scene.js
│   │   ├── webinar.js
│   │   ├── registration.js
│   │   ├── panelist.js
│   │   ├── poll.js
│   │   └── chat.js
│   ├── routes/
│   │   ├── scenes.js      ← CRUD de escenas
│   │   ├── webinars.js    ← CRUD + programación
│   │   ├── registration.js ← Registro público
│   │   ├── polls.js       ← Crear/abrir/cerrar polls
│   │   └── chat.js        ← WebSocket chat
│   ├── scenes-engine.js   ← Motor de escenas (genera filter_complex para ffmpeg)
│   ├── waiting.js         ← Video de espera en loop
│   ├── backstage.js       ← Lógica de green room
│   ├── screen-share.js    ← getDisplayMedia + producer mediasoup
│   └── scheduler.js       ← Cron para recordatorios y auto-start
web/
├── host/
│   ├── scenes.html        ← Editor de escenas
│   ├── webinar-setup.html ← Configuración de webinar
│   └── webinar-live.html  ← Panel de control en vivo
├── public/
│   ├── register.html      ← Página de registro
│   └── webinar.html       ← Landing page del webinar
├── js/
│   ├── scenes.js
│   ├── webinar.js
│   ├── screen-share.js
│   └── chat-widget.js
```

### Flujo de un webinar

```
1. Host crea sesión + escenas preconfiguradas
2. Host programa webinar (fecha, título, video de espera)
3. Sistema genera landing page pública con formulario de registro
4. Asistentes se registran → se les envía confirmación por email
5. 24h y 1h antes → scheduler envía recordatorios
6. Host abre backstage → panelistas se conectan via WebRTC
7. Stream inicia con video de espera en loop → se envía a RTMP
8. Host cambia a escena "En Vivo" → transición al contenido
9. Host cambia escenas, abre polls, modera Q&A durante el webinar
10. Host finaliza → escena de cierre → stream termina
11. Grabación disponible para descarga
```

---

## 4. Dependencia nueva: better-sqlite3

```json
{
  "dependencies": {
    "better-sqlite3": "^11.7.0"
  }
}
```

**Justificación:** Driver SQLite síncrono para Node.js, sin servidor externo, WAL mode para rendimiento concurrente. Es la opción estándar en el ecosistema Node para SQLite embebido.

---

## 5. Migración de datos existentes

La migración de JSON plano a SQLite es parte de la Fase 1:

1. Al iniciar, `db.js` verifica si la BD existe
2. Si no existe, ejecuta las migraciones SQL en orden
3. Si existen `users.json` y `sessions.json`, los importa a las tablas correspondientes
4. Los archivos JSON originales se renombran a `.json.bak` como respaldo
5. `storage.js` se mantiene temporalmente como fallback durante la transición
6. Una vez verificada la migración, se elimina el código de JSON plano

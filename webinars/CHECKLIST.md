# Checklist por etapas: Plataforma de Webinars

---

## Fase 1 — Base de datos y migración

- [ ] Instalar `better-sqlite3` como dependencia
- [ ] Crear `server/src/db.js` con inicialización SQLite, WAL mode y sistema de migraciones
- [ ] Crear `server/src/migrations/001_initial.sql` con tablas: `users`, `user_settings`, `media_assets`, `sessions`
- [ ] Crear modelo `server/src/models/user.js` que reemplace `users.js` con queries SQL
- [ ] Crear modelo `server/src/models/session.js` que reemplace lógica de sesiones en `storage.js`
- [ ] Escribir migrador automático de `users.json` → tabla `users`
- [ ] Escribir migrador automático de `sessions.json` → tabla `sessions`
- [ ] Integrar `db.js` en `index.js` (inicializar BD al arrancar)
- [ ] Actualizar `api.js` para usar modelos SQL en lugar de `storage.js` / `users.js`
- [ ] Actualizar `auth.js` para usar el modelo SQL de usuarios
- [ ] Actualizar `settings.js` para usar `user_settings` en BD
- [ ] Verificar que login, TOTP, invitaciones y CRUD de usuarios funcionen con BD
- [ ] Respaldar archivos JSON como `.json.bak` después de migración exitosa
- [ ] Escribir tests básicos para los modelos SQL

---

## Fase 2 — Sistema de escenas

- [ ] Crear migración `002_scenes.sql` con tablas: `scenes`, `scene_slots`
- [ ] Crear modelo `server/src/models/scene.js` (CRUD de escenas y slots)
- [ ] Crear ruta `server/src/routes/scenes.js` con endpoints REST:
  - [ ] `GET /api/scenes/:sessionId` — listar escenas de una sesión
  - [ ] `POST /api/scenes` — crear escena
  - [ ] `PUT /api/scenes/:id` — editar escena
  - [ ] `DELETE /api/scenes/:id` — eliminar escena
  - [ ] `PUT /api/scenes/:id/slots` — configurar slots de la escena
  - [ ] `PUT /api/scenes/reorder` — reordenar escenas
- [ ] Definir tipos de layout y su configuración JSON:
  - [ ] `single` — solo un participante (pantalla completa)
  - [ ] `grid` — grilla automática (2x2, 3x3, etc.)
  - [ ] `spotlight` — un principal grande + miniaturas
  - [ ] `screen_share` — pantalla compartida + miniaturas laterales
  - [ ] `picture_in_picture` — pantalla + cámara pequeña
  - [ ] `title_card` — imagen/texto estático (para intros/cierres)
  - [ ] `custom` — posiciones libres definidas por el usuario
- [ ] Crear `server/src/scenes-engine.js`:
  - [ ] Función que traduce una escena a filtro `filter_complex` de ffmpeg
  - [ ] Soporte de transiciones (crossfade de 0.5s) entre escenas
  - [ ] Gestión de qué feeds están activos según la escena
- [ ] Modificar `streaming.js` para aceptar escena activa en lugar de layout fijo
- [ ] Modificar `recording/` para respetar la escena activa
- [ ] Agregar mensajes WebSocket para cambio de escena en tiempo real
- [ ] Crear UI `web/host/scenes.html`:
  - [ ] Editor visual de escenas (drag & drop de slots)
  - [ ] Preview de cada layout
  - [ ] Botones de acceso rápido durante sesión en vivo
- [ ] Crear `web/js/scenes.js` con lógica del editor
- [ ] Integrar selector de escenas en el dashboard del host existente

---

## Fase 3 — Video de espera (Holding Screen)

- [ ] Extender tabla `media_assets` para tipo `waiting_video`
- [ ] Crear `server/src/waiting.js`:
  - [ ] Reproducir video en loop como input de ffmpeg
  - [ ] Alternativamente: imagen estática + audio de fondo en loop
  - [ ] Transición suave al contenido en vivo cuando el host lo indica
- [ ] Agregar concepto de estado del stream: `waiting` → `live` → `ended`
- [ ] Modificar `streaming.js`:
  - [ ] En estado `waiting`: enviar video de espera al RTMP
  - [ ] Al cambiar a `live`: crossfade al contenido de la escena activa
  - [ ] Al `ended`: volver a video de espera o cortar stream
- [ ] UI en dashboard del host:
  - [ ] Subir/seleccionar video de espera
  - [ ] Botón "Iniciar stream con pantalla de espera"
  - [ ] Botón "Ir en vivo" (transición a escena activa)
  - [ ] Botón "Finalizar" (volver a espera o cortar)
- [ ] Indicador visual del estado actual (waiting/live/ended) para el host

---

## Fase 4 — Compartir pantalla (Screen Sharing)

- [ ] Crear `server/src/screen-share.js`:
  - [ ] Registrar producer adicional de tipo `screen` en mediasoup
  - [ ] Límite: solo 1 screen share activo a la vez
  - [ ] El host decide si permitir screen share a invitados
- [ ] Crear `web/js/screen-share.js`:
  - [ ] Botón "Compartir pantalla" usando `getDisplayMedia()`
  - [ ] Crear producer mediasoup con el stream de pantalla
  - [ ] Indicador visual de quién está compartiendo
  - [ ] Botón "Dejar de compartir"
- [ ] Integrar screen share en el motor de escenas:
  - [ ] Layout `screen_share`: pantalla grande + cámaras en miniatura
  - [ ] Layout `picture_in_picture`: pantalla + 1 cámara pequeña
  - [ ] Cambio automático de escena al iniciar screen share (configurable)
- [ ] Modificar `streaming.js` para incluir el feed de pantalla en el composite
- [ ] Modificar `signaling.js` para los nuevos mensajes de screen share

---

## Fase 5 — Backstage / Green Room

- [ ] Crear `server/src/backstage.js`:
  - [ ] Estado por peer: `backstage` | `live` | `removed`
  - [ ] Los peers en `backstage` se conectan via WebRTC pero NO aparecen en el stream
  - [ ] El host puede mover peers entre `backstage` y `live`
- [ ] Modificar `rooms.js`:
  - [ ] Agregar campo `stage` al peer: `'backstage'` o `'live'`
  - [ ] Nuevos peers entran en `backstage` por defecto
- [ ] Modificar `streaming.js`:
  - [ ] Solo incluir en el composite los peers con `stage === 'live'`
  - [ ] Relanzar ffmpeg cuando un peer cambia de estado
- [ ] Modificar `signaling.js`:
  - [ ] Peers en backstage reciben audio/video de otros peers en backstage
  - [ ] Peers en backstage NO reciben el feed del stream
  - [ ] Mensaje `stage-change` para notificar transiciones
- [ ] UI del host:
  - [ ] Lista de peers con indicador backstage/live
  - [ ] Botón "Traer al aire" / "Enviar a backstage" por peer
  - [ ] Preview del backstage separado del preview del stream
- [ ] UI del invitado:
  - [ ] Indicador claro de si está "al aire" o en backstage
  - [ ] Preview de audio/video mientras espera

---

## Fase 6 — Programación de webinars y registro

- [ ] Crear migración `003_webinars.sql` con tablas: `webinars`, `webinar_scenes`, `registrations`, `panelists`
- [ ] Crear modelo `server/src/models/webinar.js` (CRUD)
- [ ] Crear modelo `server/src/models/registration.js`
- [ ] Crear modelo `server/src/models/panelist.js`
- [ ] Crear rutas `server/src/routes/webinars.js`:
  - [ ] `POST /api/webinars` — crear webinar
  - [ ] `GET /api/webinars` — listar webinars del host
  - [ ] `GET /api/webinars/:id` — detalle
  - [ ] `PUT /api/webinars/:id` — editar
  - [ ] `DELETE /api/webinars/:id` — cancelar
  - [ ] `POST /api/webinars/:id/panelists` — invitar panelista
  - [ ] `PUT /api/webinars/:id/status` — cambiar estado (draft→scheduled→live→ended)
- [ ] Crear rutas públicas `server/src/routes/registration.js`:
  - [ ] `GET /webinar/:slug` — landing page pública
  - [ ] `POST /api/register/:slug` — registrar asistente
  - [ ] Confirmación por email al registrarse
- [ ] Crear `server/src/scheduler.js`:
  - [ ] Cron job (setInterval) que revisa webinars próximos
  - [ ] Enviar recordatorio 24h antes a registrados (marcar `reminded=1`)
  - [ ] Enviar recordatorio 1h antes
  - [ ] Enviar invitación a panelistas con link de acceso
- [ ] Crear `web/public/register.html` — landing page de registro
- [ ] Crear `web/host/webinar-setup.html` — formulario de creación/edición
- [ ] Crear `web/host/webinar-live.html` — panel de control en vivo
- [ ] Integrar emails usando la infraestructura SMTP existente (`email.js`)

---

## Fase 7 — Chat y Q&A

- [ ] Crear migración `004_chat.sql` con tabla `chat_messages`
- [ ] Crear modelo `server/src/models/chat.js`
- [ ] Crear ruta WebSocket `server/src/routes/chat.js`:
  - [ ] Canal de chat por webinar
  - [ ] Tipos de mensaje: chat normal, pregunta (Q&A), respuesta del host
  - [ ] El host puede fijar (pin) mensajes
  - [ ] Rate limiting: máx. 1 mensaje por segundo por usuario
- [ ] Crear `web/js/chat-widget.js`:
  - [ ] Widget embebible en la landing page del webinar
  - [ ] Tabs: Chat / Q&A
  - [ ] Indicador de mensajes fijados
  - [ ] Autoscroll con detección de scroll manual
- [ ] Integrar chat en `web/public/webinar.html`
- [ ] Integrar panel de moderación en `web/host/webinar-live.html`

---

## Fase 8 — Polls / Encuestas en vivo

- [ ] Crear migración `005_polls.sql` con tablas: `polls`, `poll_options`, `poll_votes`
- [ ] Crear modelo `server/src/models/poll.js`
- [ ] Crear rutas `server/src/routes/polls.js`:
  - [ ] `POST /api/polls` — crear poll
  - [ ] `PUT /api/polls/:id/open` — abrir votación
  - [ ] `PUT /api/polls/:id/close` — cerrar votación
  - [ ] `PUT /api/polls/:id/show` — mostrar resultados en overlay
  - [ ] `POST /api/polls/:id/vote` — votar (público)
  - [ ] `GET /api/polls/:id/results` — resultados en tiempo real
- [ ] Overlay de resultados en el stream:
  - [ ] Modificar `streaming.js` para superponer resultados de polls como texto/barra
  - [ ] Generar imagen PNG temporal con resultados y usarla como overlay en ffmpeg
- [ ] UI del host:
  - [ ] Crear poll con pregunta y opciones
  - [ ] Botones abrir/cerrar/mostrar resultados
  - [ ] Vista previa de resultados en tiempo real
- [ ] UI del asistente:
  - [ ] Widget de votación en la landing page
  - [ ] Vista de resultados cuando el host los muestra

---

## Fase 9 — Integración y pulido

- [ ] Unificar todas las rutas en `api.js`
- [ ] Revisión de seguridad:
  - [ ] Validar que endpoints públicos no filtren datos privados
  - [ ] Rate limiting en registro y chat
  - [ ] Sanitizar inputs de chat y polls contra XSS
  - [ ] Verificar permisos (solo host puede cambiar escenas/estado)
- [ ] UI/UX:
  - [ ] Navegación coherente entre dashboard existente y nuevas páginas
  - [ ] Responsive para panel del host en tablet
  - [ ] Indicadores de estado claros (backstage/live/waiting)
- [ ] Documentación:
  - [ ] Actualizar README con las nuevas funcionalidades
  - [ ] Documentar variables de entorno nuevas (si las hay)
  - [ ] Guía rápida: "Cómo crear tu primer webinar"
- [ ] Tests:
  - [ ] Tests E2E con Playwright para flujo completo de webinar
  - [ ] Tests unitarios para modelos SQL
  - [ ] Test de migración de JSON a SQLite
- [ ] Performance:
  - [ ] Verificar que el cambio de escenas en ffmpeg sea < 2s
  - [ ] Verificar latencia del chat WebSocket
  - [ ] Probar con 10 panelistas + 500 registrados simultáneos

---

## Notas importantes

### Orden de ejecución recomendado

Las fases están diseñadas para construirse incrementalmente. Cada fase agrega valor por sí sola:

1. **Fase 1** es prerequisito de todas las demás
2. **Fases 2-5** pueden trabajarse en paralelo (son independientes entre sí)
3. **Fase 6** depende de que al menos Fase 2 esté completa
4. **Fases 7-8** dependen de Fase 6
5. **Fase 9** es la integración final

### Compatibilidad hacia atrás

- La migración de JSON a SQLite debe ser transparente: si el usuario ya tenía datos, se conservan
- Los endpoints existentes de la API mantienen su contrato
- La funcionalidad actual de podcast sigue funcionando sin configurar webinars

### Stack confirmado

- **Backend:** Node.js + Express + WebSocket + mediasoup + ffmpeg
- **Base de datos:** SQLite via better-sqlite3
- **Frontend:** HTML/CSS/JS vanilla (consistente con el proyecto actual, sin frameworks)
- **Email:** SMTP existente
- **Streaming:** RTMP a YouTube (existente)

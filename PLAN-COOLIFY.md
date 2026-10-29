# Plan: OpenReply en Coolify para sustituir a ManyChat

Objetivo: responder automáticamente (DM privado + respuesta pública opcional) a los comentarios con palabra clave en los posts/reels de tu cuenta de Instagram, alojado en tu VPS con Coolify.

## 0. Qué es OpenReply (lo que importa para desplegarlo)

| Pieza | Qué hace | En Coolify |
| --- | --- | --- |
| **web** (`npm run start`) | Dashboard, login, OAuth de Instagram, **recibe los webhooks** de Meta | Servicio con dominio HTTPS público |
| **worker** (`npm run worker`) | **Envía los DM** (cola BullMQ), polling de comentarios perdidos cada 5 min | Servicio sin dominio, siempre encendido |
| **cron** (`scripts/cron.sh`) | Refresca el token de Instagram a diario (si no corre, **todo deja de funcionar en silencio** a los ~60 días) | Servicio sin dominio |
| **PostgreSQL** | Campañas, logs, cuentas, sesiones | Recurso Database de Coolify |
| **Redis** | Cola de envíos + rate limiter | Recurso Database de Coolify |

Reglas duras del repo:
- web, worker y cron comparten **la misma imagen** (Dockerfile en la raíz) y **el mismo** `DATABASE_URL`, `REDIS_URL` y `ENCRYPTION_KEY`. Si `ENCRYPTION_KEY` difiere o cambia, ningún DM se puede descifrar y hay que reconectar Instagram.
- Las migraciones (`prisma migrate deploy`) se ejecutan **al arrancar** la web, no en la build.
- Login solo por *magic link* por email → necesitas Resend o tu propio SMTP.

## 1. Decisiones previas

1. **Conexión con Instagram** — recomiendo **tu propia app de Meta** (gratis):
   - Para tu propia cuenta **no necesitas App Review** ni verificación de negocio: basta con añadir tu cuenta como *Instagram tester* y publicar la app.
   - Alternativa: **Zernio** (de pago, patrocinador del proyecto): te ahorra crear la app de Meta. Limitaciones: el selector de posts solo muestra los 25 últimos y la analítica requiere un add-on. Solo compensa si no quieres pelearte con el panel de Meta.
2. **Dominio**: un subdominio dedicado, p. ej. `reply.tutopress.co`. Debe ser el dominio **principal y sin redirecciones** (Meta no sigue bien los 307 en los webhooks).
3. **Email de login**: Resend (dominio verificado) o `EMAIL_SERVER` con tu SMTP actual.

## 2. Preparación (en tu máquina)

1. Haz **fork** de `diwenne/openreply` en tu GitHub (así controlas cuándo actualizas).
2. Añade al fork el archivo [`docker-compose.coolify.yml`](docker-compose.coolify.yml) que he dejado en esta carpeta.
3. (Opcional, recomendado) Cambia en el `Dockerfile` `node:20-slim` → `node:22-slim` en las dos etapas: Node 20 ya está fuera de soporte. Prisma 7 / Next 16 aceptan Node 22.12+.
4. Genera los secretos (guárdalos en un gestor de contraseñas, no en el repo):
   ```bash
   openssl rand -base64 32   # NEXTAUTH_SECRET
   openssl rand -base64 32   # CRON_SECRET
   openssl rand -hex 32      # ENCRYPTION_KEY (64 caracteres hex, obligatorio)
   openssl rand -hex 16      # WEBHOOK_VERIFY_TOKEN
   ```
5. DNS: registro `A` de `reply.tutopress.co` → IP de la VPS.

## 3. Coolify: bases de datos

1. **+ New → Database → PostgreSQL 16**. Nombre `openreply-db`. Copia la **Postgres URL (internal)**.
   - Activa **Scheduled Backups** (diario, idealmente a un S3/R2).
2. **+ New → Database → Redis 7**. Copia la **Redis URL (internal)**.
   - En *Custom Redis Configuration* pon `maxmemory-policy noeviction` (BullMQ lo exige; con política de expulsión se pierden trabajos).
   - Persistencia AOF activada.
3. No expongas ninguno de los dos a Internet (sin "Make it publicly available").

## 4. Coolify: la aplicación

1. **+ New → Application → (GitHub App o Public Repo) → `juansebas0712/openreply`**, rama **`coolify`** (`main` queda limpia para sincronizar con upstream; después de cada *Sync fork* haz `git rebase main` en `coolify`).
2. Build Pack: **Docker Compose**, ruta del compose: `/docker-compose.coolify.yml`.
3. En la configuración del recurso, activa **Connect To Predefined Network** (para que alcance a Postgres/Redis por hostname interno).
4. Dominio: asigna `https://reply.tutopress.co` **solo al servicio `web`** (puerto 3000). worker y cron sin dominio.
5. Variables de entorno:

   | Variable | Valor |
   | --- | --- |
   | `NEXTAUTH_URL` | `https://reply.tutopress.co` (sin barra final) |
   | `NEXTAUTH_SECRET`, `CRON_SECRET`, `ENCRYPTION_KEY`, `WEBHOOK_VERIFY_TOKEN` | los del paso 2.4 |
   | `DATABASE_URL` | URL interna de Postgres |
   | `REDIS_URL` | URL interna de Redis |
   | `EMAIL_FROM` | `OpenReply <login@tutopress.co>` (dominio verificado) |
   | `RESEND_API_KEY` **o** `EMAIL_SERVER` | uno de los dos |
   | `ALLOWED_EMAILS` | tu(s) email(s). **Imprescindible**: sin esto cualquiera que llegue a la URL puede crearse cuenta |
   | `META_GRAPH_API_VERSION` | `v25.0` |
   | `INSTAGRAM_APP_ID`, `INSTAGRAM_APP_SECRET`, `FACEBOOK_APP_SECRET` | del paso 5 (puedes desplegar primero sin ellas y añadirlas luego) |

6. **Deploy**. Comprueba en logs: `web` aplica migraciones y arranca, `worker` imprime `[DM Worker] Started`, `cron` imprime `[cron] scheduler started`.
7. Verifica `https://reply.tutopress.co/api/health` → `status: ok` y `worker.healthy: true`.
8. Entra con tu email (magic link) → se crea tu workspace.

## 5. App de Meta (camino directo)

Sigue `docs/setup.md`, pasos 4–9. Resumen con tu dominio:

1. developers.facebook.com → Crear app → tipo **Business** → caso de uso **"Manage messaging and content on Instagram"** (no Facebook Login, no Marketing API).
2. Copia `INSTAGRAM_APP_ID` + `INSTAGRAM_APP_SECRET` (producto Instagram → *API setup with Instagram login*) y `FACEBOOK_APP_SECRET` (App settings → Basic). Ojo: el Instagram App ID **no** es el Facebook App ID. Mételos en Coolify y redespliega.
3. **Roles → Instagram testers**: invita a tu cuenta y **acepta la invitación desde Instagram** (Ajustes → Apps y sitios web → Invitaciones de tester). Paso que todo el mundo se salta.
4. Redirect OAuth: `https://reply.tutopress.co/api/instagram/callback`
5. Webhook: callback `https://reply.tutopress.co/api/webhook`, verify token = `WEBHOOK_VERIFY_TOKEN`. Suscribe **`comments`** y **`messages`**. Prueba con *Test → Send to My Server*.
6. URLs legales (las sirve OpenReply): `/privacy`, `/terms`, `/data-deletion`.
7. **Publica la app (Live)**. En modo Development los comentarios reales **no** llegan.
8. En OpenReply → Settings → **Connect Instagram**.

## 6. Migración desde ManyChat (corte)

No pueden convivir: si ambos están conectados, los dos responden, y ManyChat puede seguir siendo el "dueño del hilo" y hacer fallar los botones del DM (`not the thread owner`).

1. **Inventario**: exporta/apunta cada automatización de ManyChat (post, palabras clave, texto del DM, enlaces, respuesta pública, si exige follow).
2. **Recréalas en OpenReply** como campañas, en pausa/borrador (hay plantillas e importación CSV en *Campaigns → Import*). Usa `{username}`, enlaces con tracking y *follow gate* si lo usabas.
3. **Ventana de corte** (hora de poco tráfico):
   1. ManyChat → Settings → Instagram → **Disconnect channel**.
   2. Meta Business Suite → Configuración → Integraciones → **Conversation routing** → tu cuenta → *Partner apps*: elimina ManyChat; en tu app de OpenReply activa **Access all conversations** y **Take control of conversations**.
   3. Activa las campañas en OpenReply.
4. **Prueba** desde otra cuenta (tus propios comentarios se ignoran a propósito): comenta la palabra clave → llega el DM → pulsa el botón → llega el enlace. Revisa *DM Logs* (fila `SENT`).
5. Mantén la cuenta de ManyChat sin cancelar 1–2 semanas como plan B; después cancela.

## 7. Operación

- **Monitor**: Uptime Kuma (o similar) contra `/api/health`; alerta si deja de dar 200 — detecta worker caído o cola atascada.
- **Token**: comprueba a los pocos días en los logs de `cron` la línea `refresh-tokens ok`.
- **Backups**: Postgres diario (paso 3). Guarda `ENCRYPTION_KEY` fuera de la VPS.
- **Actualizaciones**: *Sync fork* en GitHub → redeploy en Coolify (las migraciones se aplican solas al arrancar). Lee el changelog antes; haz backup antes de actualizar.
- **Diagnóstico**: tablas `WebhookEvent` (¿llegó el comentario?), `DmLog` (¿se envió?), `OperationalEvent` (errores del worker).

## 8. Límites a tener en cuenta (aplican igual que con ManyChat)

- Solo cuentas **Business o Creator**.
- Un DM privado por comentario; los mensajes siguientes dependen de la ventana de 24 h de Instagram.
- Rate limit por cuenta (~750 respuestas privadas/hora): el exceso se encola, no se pierde.
- Funciones de ManyChat que OpenReply **no** tiene: flujos conversacionales complejos, secuencias, CRM/etiquetas de contactos, integraciones tipo Zapier. Si dependes de ellas, revísalo antes del corte.

## Checklist

- [ ] Fork + `docker-compose.coolify.yml` (+ Node 22 opcional)
- [ ] Secretos generados y guardados
- [ ] DNS apuntando a la VPS
- [ ] Postgres + Redis (noeviction, backups) en Coolify
- [ ] App desplegada, `/api/health` OK, login con email
- [ ] App de Meta: tester aceptado, redirect, webhook (`comments` + `messages`), Live
- [ ] Instagram conectado en OpenReply
- [ ] Campañas recreadas
- [ ] Corte: ManyChat desconectado + Conversation routing
- [ ] Prueba end-to-end desde otra cuenta
- [ ] Monitor de `/api/health` y revisión de `refresh-tokens`

# Reflexión · CI/CD con Firebase Hosting y hardening de headers

**Alumna:** María Melina Duarte · Programación Avanzada 2026 · FCyT Concepción del Uruguay

## Entregables

| # | Entregable | Link |
|---|---|---|
| 1 | Repo | https://github.com/xxMeDuzaa/practica-firebase-cicd |
| 2 | Producción | https://practica-ci-cd.web.app |
| 3 | PR mergeado con comentario del bot | https://github.com/xxMeDuzaa/practica-firebase-cicd/pull/2 |

## Mediciones (entregable 4)

### B1 · Antes (producción sin headers)

Calificación securityheaders.com: **D**

Observación: sin configurar nada, Firebase Hosting ya envía `Strict-Transport-Security` por defecto y un `Cache-Control: max-age=3600` igual para todo (HTML y assets). Ese caché de 1 hora en el HTML implica que un deploy o un rollback podía tardar hasta una hora en verse.

```text
▶ HTML: https://practica-ci-cd.web.app/
  status 200
  ✘ content-security-policy (falta)
  ✔ strict-transport-security: max-age=31556926; includeSubDomains; preload
  ✘ x-content-type-options (falta)
  ✘ x-frame-options (falta)
  ✘ referrer-policy (falta)
  ✘ permissions-policy (falta)
  · cache-control: max-age=3600

▶ Asset estático: /_next/static/chunks/1x6obpmlu1_c2.js
  ✘ cache-control: max-age=3600

❌ 6 control(es) pendiente(s).
```

### B5 · Después (producción con headers)

Calificación securityheaders.com: ** A+**

La CSP servida en producción ya no tiene `'unsafe-inline'`: `script-src` lista los `sha256` calculados por `scripts/csp-hashes.mjs` en el pipeline. El HTML pasó de `max-age=3600` a `no-cache` y los assets con hash a un año `immutable`.

```text
▶ HTML: https://practica-ci-cd.web.app/
  status 200
  ✔ content-security-policy: default-src 'self'; script-src 'self' 'sha256-OBTN3RiyCV4Bq7dFqZ5a2pAXjnCcCYeTJMO2I/LYKeo=' 'sha256-bcUxw3Lxd1…
  ✔ strict-transport-security: max-age=31556926; includeSubDomains; preload
  ✔ x-content-type-options: nosniff
  ✔ x-frame-options: DENY
  ✔ referrer-policy: strict-origin-when-cross-origin
  ✔ permissions-policy: camera=(), microphone=(), geolocation=(), payment=(), usb=(), bluetooth=(), serial=(), hid=(), midi=(), magnet…
  · cache-control: no-cache

▶ Asset estático: /_next/static/chunks/1x6obpmlu1_c2.js
  ✔ cache-control: public, max-age=31536000, immutable

✅ Todos los controles pasaron.
```

## Decisiones de configuración

**Pipeline**
- Preview: trigger `pull_request`, `expires: 3d`, `concurrency` por número de PR con `cancel-in-progress: true` (un push nuevo deja obsoleto al build anterior).
- Producción: trigger `push` a `main`, `channelId: live`, `concurrency` grupo `production` con `cancel-in-progress: false` (un deploy cortado a la mitad podría dejar una versión inconsistente; mejor hacer fila), environment `production` para poder exigir aprobación manual.
- Trazabilidad: en el PR uso `github.event.pull_request.head.sha` y `github.head_ref`, porque `github.sha` en un `pull_request` es el *merge commit* sintético que GitHub arma para probar, no el último commit de la rama. En `main` uso `github.sha` y `github.ref_name`.
- Gate: después de desplegar el preview, el workflow corre `check-headers.mjs` contra la URL del preview; si falta un header, el check del PR queda en rojo.

**Caché**
- Regla general `**` → `Cache-Control: no-cache` (el navegador puede guardar el HTML pero tiene que revalidar con ETag en cada visita).
- Regla específica `/_next/static/**` → `public, max-age=31536000, immutable`. Va después de la general porque, si dos reglas definen el mismo header, gana la última.

**CSP final**

```text
default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self';
img-src 'self' https://picsum.photos https://fastly.picsum.photos; font-src 'self';
connect-src 'self'; object-src 'none'; base-uri 'self'; form-action 'self';
frame-ancestors 'none'; upgrade-insecure-requests
```

Esa es la versión commiteada. Nota: en esta máquina el emulador local no aplicó los headers de `firebase.json`, así que la validación la hice en un preview channel real (`npm run deploy:preview`), que pasó todos los controles. En el pipeline, `scripts/csp-hashes.mjs` corre después del build y reemplaza `'unsafe-inline'` por los `sha256` de los scripts inline de ese build, así que **lo que llega a preview y a producción no tiene `'unsafe-inline'`**.

## Preguntas

### 1. ¿Por qué el preview tiene vencimiento?
Un preview es una URL pública con código que todavía no se revisó. Si nunca vence se acumulan versiones viejas accesibles por cualquiera que tenga el link (que queda en el PR, en chats, en el historial): pueden tener bugs o vulnerabilidades ya corregidas en producción, datos de prueba o features no anunciadas, y un buscador podría indexarlas. Además, Firebase limita la cantidad de canales por sitio, así que sin vencimiento se llenan. Con `expires: 3d` el entorno es efímero: existe mientras se revisa y después desaparece solo.

### 2. Desplegar vs. liberar
**Desplegar** es poner una versión en la infraestructura; **liberar** es hacer que los usuarios reales la reciban. En este flujo, el workflow de PR *despliega* a un preview channel pero no libera nada: producción no cambia. Se *libera* cuando la versión pasa al canal `live`, ya sea por el merge a `main`, por `hosting:clone` de un preview a `live` (libera algo que ya estaba desplegado, sin recompilar) o por un rollback, que vuelve a liberar una versión anterior que seguía guardada en Hosting. Como cada release de Firebase es una versión inmutable, liberar o volver atrás es solo mover un puntero, por eso tarda segundos.

### 3. Clave de la service account
Un repo (y más uno público) es copiado, clonado y queda en el historial de git para siempre: una clave commiteada se considera filtrada aunque después se borre. En un secreto de GitHub se guarda cifrada, solo se expone a los workflows, se enmascara en los logs y no se entrega a PRs que vienen de forks. Quien obtenga la clave puede desplegar cualquier contenido en `<project>.web.app` (defacement, phishing o malware bajo un dominio confiable), borrar canales o hacer rollbacks, y usar cualquier otro permiso que tenga esa cuenta en el proyecto. Para limitar el daño: darle solo el rol **Firebase Hosting Admin** (mínimo privilegio, que es lo que crea `firebase init hosting:github`), proteger `main` y el environment `production` con revisión obligatoria, rotar la clave periódicamente y ante cualquier sospecha, revisar los audit logs de GCP, y, mejor todavía, reemplazar la clave JSON por **Workload Identity Federation** (OIDC de GitHub Actions), que da credenciales de corta duración y no deja ninguna clave que filtrar.

### 4. ¿Por qué la CSP rompió la app?
Con `script-src 'self'` la página cargaba pero el filtro de `/cursos` no respondía. En `out/cursos.html` se ve que Next.js incrusta scripts inline (`<script>self.__next_f.push([1,"…"])</script>`) con el payload RSC que React necesita para **hidratar** los componentes de cliente, y además un `<script id="_R_">` de Turbopack que arranca la carga de los chunks (en total, 5 scripts inline distintos en este build). La CSP bloquea todo script inline que no esté autorizado, así que el HTML se ve pero React nunca toma el control de los eventos. La imagen de portada también desaparecía por `img-src 'self'`: `picsum.photos` responde con un redirect a `fastly.picsum.photos`, y la CSP se evalúa en cada salto del redirect, así que hay que permitir los dos orígenes exactos (no `https:` ni `*`).

Opciones evaluadas:

| Opción | Costo de seguridad |
|---|---|
| `'unsafe-inline'` en `script-src` | Alto: si alguien logra inyectar HTML (XSS) puede ejecutar cualquier script inline; la CSP deja de proteger contra el ataque principal para el que existe. |
| Nonce por request | Ideal, pero necesita un servidor que genere un nonce distinto en cada respuesta. Con hosting estático no hay. |
| **Hashes `sha256`** (elegida para lo desplegado) | Bajo: solo se ejecutan los scripts inline exactos que generó el build; uno inyectado tiene otro hash y se bloquea. El costo es operativo: los hashes cambian en cada build (el SHA del footer viaja en el payload), así que hay que calcularlos en el pipeline, entre `next build` y el deploy. Por eso está el paso `csp-hashes.mjs`. |
| `'strict-dynamic'` + hash | Útil si los scripts cargan otros dinámicamente; acá los chunks ya son `'self'`, no hace falta. |

### 5. ¿Qué ataque mitiga cada header de B2?
- **Strict-Transport-Security** (`max-age=63072000; includeSubDomains`, 2 años): impide *SSL stripping* y downgrade a HTTP en redes hostiles (un Wi-Fi público que intercepta la primera petición). El navegador recuerda que el sitio solo va por HTTPS. Detalle: `.app` ya está en la lista de precarga HSTS de los navegadores, así que `web.app` está protegido incluso en la primera visita; el header igual hace falta para dominios propios. No agregué `preload` porque no controlo el dominio. En la medición se ve que en `*.web.app` Firebase sobrescribe el valor configurado con el suyo (`max-age=31556926; includeSubDomains; preload`); el de `firebase.json` aplicaría en un dominio propio.
- **X-Content-Type-Options: nosniff**: evita el *MIME sniffing*, por ejemplo que un archivo subido como `.txt` o imagen con contenido JS/HTML sea interpretado y ejecutado como script (XSS por confusión de tipos).
- **X-Frame-Options: DENY**: evita el *clickjacking*: otro sitio no puede meter el nuestro en un iframe invisible para que el usuario haga clic sin saberlo. Se complementa con `frame-ancestors 'none'` en la CSP, que es la versión moderna.
- **Referrer-Policy: strict-origin-when-cross-origin**: evita filtrar la URL completa (rutas, parámetros con tokens o datos de búsqueda) a sitios de terceros. Hacia afuera solo se manda el origen, y nada si se baja de HTTPS a HTTP.
- **Permissions-Policy**: desactiva cámara, micrófono, geolocalización, pago, USB, sensores, etc. Si se inyecta un script o un iframe de terceros, no puede pedir esos permisos (reduce la superficie para espionaje o fingerprinting). El catálogo no usa ninguna, así que se niegan todas.
- Extras: **Cross-Origin-Opener-Policy: same-origin** aísla la ventana de popups de otros orígenes (mitiga ataques tipo XS-Leaks/Spectre y *tabnabbing*); **X-Permitted-Cross-Domain-Policies: none** bloquea políticas de clientes Flash/PDF heredados.

### 6. ¿Por qué `/_next/static/` se cachea un año y el HTML no?
Los archivos de `/_next/static/` tienen un hash del contenido en el nombre (`page-3f9a1c.js`): si el código cambia, cambia el nombre, así que un archivo con un nombre dado **nunca** va a tener otro contenido y se puede marcar `immutable`. El HTML, en cambio, siempre se sirve en la misma URL (`/cursos`) y es el que apunta a los assets de la versión actual. Si el HTML quedara cacheado un año, un deploy nuevo o, peor, un **rollback** no le llegaría al usuario: seguiría viendo la versión vieja (o la que se quiso retirar) hasta que expire su caché. Con `no-cache` el navegador revalida en cada visita (si no cambió responde 304, barato) y ve el cambio en cuanto se libera. Este esquema hace que deploys y rollbacks sean instantáneos para el usuario sin sacrificar rendimiento: lo pesado (JS y CSS) se baja una sola vez por versión.

### 7. Si la app necesitara SSR
- **Arquitectura**: no alcanza con archivos estáticos. Habría que usar **Firebase App Hosting** (o Cloud Run), donde Next corre como un servidor Node en contenedores; el pipeline construye una imagen y el rollback pasa a ser entre revisiones del servicio. Aparecen cold starts, logs del servidor, variables de entorno y secretos en runtime.
- **CSP**: mejora. El servidor puede generar un **nonce** distinto por request (por ejemplo en el middleware/proxy de Next) y usar `script-src 'nonce-…' 'strict-dynamic'`, sin `'unsafe-inline'` ni hashes recalculados en CI. Los headers se podrían definir en código además de en la config. Como contrapartida, ese HTML ya no se puede cachear en el CDN porque el nonce cambia en cada respuesta.
- **Costo**: App Hosting exige el plan **Blaze** (con tarjeta, pago por uso de Cloud Run, Cloud Build y Artifact Registry). Deja de ser costo cero garantizado y hay que poner alertas de presupuesto.

### A6 · Ciclo completo y rollback (registro)

| Paso | Canal | Versión en el footer | Release de Hosting |
|---|---|---|---|
| PR #2 abierto (`feat/titulo`) | preview `pr2-feat-titulo` | `aac91c9` · rama `feat/titulo` · título nuevo | — |
| Mientras tanto, producción sin cambios | live | `d5c3527` · `main` · título viejo | `bd4a4d` |
| Merge del PR #2 | live | `30c9d46` · `main` · título nuevo | `b39f71` |
| Rollback desde la consola (versión `bd4a4d`) | live | `d5c3527` · `main` · título viejo | `bd4a4d` |
| Bonus: `hosting:clone practica-ci-cd:pr2-feat-titulo practica-ci-cd:live` | live | `aac91c9` · `feat/titulo` · título nuevo | copia del preview |

El preview muestra `aac91c9` (último commit de la rama) y producción `30c9d46` (el merge commit que GitHub crea en `main`), por eso los SHA difieren entre canales. En el bonus, producción pasó a mostrar exactamente `aac91c9` / `feat/titulo`: `hosting:clone` copia los archivos ya compilados del preview, sin build nuevo, así que se libera bit a bit lo que se revisó. El rollback tardó segundos porque Hosting solo vuelve a apuntar `live` a una versión ya subida, sin recompilar. Elegí `bd4a4d` y no `c446a2` porque esta última es el deploy manual de la medición B1, sin headers: hacer rollback a ella habría reabierto el problema de seguridad.

**Incidente en el pipeline:** el primer run del PR #2 falló en el gate con `status 404`: el script consultó el canal recién creado en el mismo segundo del deploy, antes de que el CDN lo sirviera. Lo corregí haciendo que el paso espere hasta recibir HTTP 200 (con reintentos cada 5 s, hasta unos 2 minutos) antes de verificar los headers. Un gate de verificación tiene que tolerar la consistencia eventual del CDN, porque si no da falsos negativos.

### Rollback: ¿qué pasa con `main`?
Después del rollback en la consola, `live` sirve la versión anterior pero `main` sigue con el código nuevo: el repo y producción quedan desincronizados. En el próximo merge el pipeline compila `main` completo y **vuelve a publicar el cambio que se había retirado**, junto con lo nuevo. En un equipo real, el rollback en consola se usa como mitigación inmediata y enseguida se corrige el repo: un `git revert` del commit problemático por PR (así `main` refleja lo que está en producción y queda registro), o un *roll forward* con el arreglo. Otra opción es usar feature flags para desactivar el cambio sin redeploy.

### Bonus · promoción sin recompilar
`npx firebase hosting:clone practica-ci-cd:CANAL practica-ci-cd:live` copia los archivos exactos que se revisaron en el preview al canal `live`. Lo que se aprobó es bit a bit lo que se libera (no hay un segundo build que pueda dar otro resultado). La contra es que el footer va a mostrar la rama del PR y su SHA, no `main`.

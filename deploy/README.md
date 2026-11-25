# pspp-app — Despliegue (compilar + nginx)

Este documento cubre la parte de **frontend** del despliegue: compilar
los estáticos y configurar el nginx que los sirve y hace de proxy hacia
el backend. El backend (repo `pspp-api`) se despliega por separado —
ver su propio `deploy/README.md` para systemd, Docker o el modelo
SEDATU del lado del backend.

## 1. Compilar

Requiere Node.js 22 en el servidor (o en el equipo/pipeline que genere
`dist/`). El frontend necesita **todas** las dependencias para
compilar, dev incluidas (`vite`, el bundler, es una devDependency; sin
ella `npm run build` no existe):

```bash
npm ci
npm run build
```

Esto genera `dist/`: los archivos estáticos que nginx va a servir. Una
vez que el build terminó, **las dependencias de desarrollo ya no hacen
falta**: nginx solo lee `dist/`, nunca vuelve a tocar `node_modules` ni
a correr Vite. Si se quiere liberar espacio: `rm -rf node_modules`.

### Variables de compilación (no de ejecución)

Se leen **solo durante `npm run build`**, nunca después de que `dist/`
ya existe — cambiarlas exige recompilar:

- **`VITE_API_URL`**: a dónde apunta el frontend compilado para llamar
  a la API. Por omisión, en el código, es la ruta relativa `/api/v1`
  (`src/api/client.js` y otros). Con ese valor por omisión, el mismo
  `dist/` compilado funciona en **cualquier dominio** sin recompilar:
  como la ruta es relativa, el navegador la resuelve contra el dominio
  desde el que se sirvió la página, y quien decide a dónde va realmente
  `/api/v1` es nginx (`location /api/`, ver sección 2), no el frontend.
  No hay que definir esta variable a propósito, salvo que el backend
  viva en un dominio distinto al del frontend, o bajo un subpath (ver
  "Modelo SEDATU" más abajo).
- **`VITE_CARTO_API_KEY`**: la llave del proveedor de mapas base
  (CARTO). Si esta llave cambia, **hay que recompilar** para que el
  nuevo valor quede incluido en `dist/`; cambiarla en el entorno
  después de compilar no tiene ningún efecto, porque el build es
  estático.
- **`VITE_BASE_PATH`**: ruta bajo la que nginx publica el frontend. Por
  omisión `/` (raíz del dominio). Ver "Modelo SEDATU" para cuando se
  sirve bajo un subpath (`/pspp/`).

Para fijar estas variables al compilar:
`VITE_API_URL=... VITE_CARTO_API_KEY=... VITE_BASE_PATH=... npm run build`
(o exportarlas antes).

## 2. nginx (despliegue nativo, un solo servidor)

Configuración de referencia: [`deploy/nginx/pspp.conf`](nginx/pspp.conf)
— lee los comentarios de ese archivo antes de copiarlo, hay dos rutas
que casi siempre hay que ajustar (`root` y `proxy_pass`).

Antes de copiarla, revisar si ya hay algo escuchando el puerto que se
va a usar (8090 en la plantilla) o compitiendo por el `server_name`:

```bash
ls /etc/nginx/conf.d/
grep -rl "listen" /etc/nginx/conf.d/
```

Si no hay conflicto:

```bash
cp deploy/nginx/pspp.conf /etc/nginx/conf.d/pspp.conf
nginx -t
systemctl reload nginx
```

`nginx -t` valida la sintaxis antes de aplicar nada; si marca error,
nginx sigue corriendo con la configuración anterior, no se cae.

### Content-Security-Policy (CSP, paso 9 DGTIC — NOF-2)

Va a nivel `server`, no en el backend: quien sirve el documento HTML es
nginx — el backend solo contesta JSON bajo `/api/`.

```
default-src 'self';
script-src 'self';
style-src 'self' 'unsafe-inline' https://fonts.googleapis.com;
font-src 'self' https://fonts.gstatic.com;
img-src 'self' data: blob: https://*.basemaps.cartocdn.com https://*.tile.openstreetmap.org;
media-src 'self' blob:;
connect-src 'self';
frame-src 'self' blob: https:;
object-src 'none';
base-uri 'self';
form-action 'self';
frame-ancestors 'self';
```

**Por qué cada directiva que no es simplemente `'self'`:**

- **`style-src 'unsafe-inline'`** — probado con el navegador real contra
  esta misma política, no asumido: `/login` (toda su UI usa
  `style={{...}}` de React — fondos, colores de marca) pasa con **cero**
  violaciones de `style-src` aunque se quite `unsafe-inline`. La razón
  es que React fija cada propiedad CSS por separado sobre el objeto
  `style` del DOM (la API CSSOM), que CSP no vigila — solo vigila
  escribir el atributo `style=""` completo como texto (`setAttribute`,
  `cssText`, o HTML con `dangerouslySetInnerHTML`, que este código no
  usa en ningún lado). **Leaflet sí lo necesita**: con la misma prueba
  pero cargando el mapa, el navegador reporta exactamente una violación
  ("Applying inline style violates... style-src") al posicionar sus
  teselas internamente, y sin `unsafe-inline` el mapa no se puede
  mostrar. Es la única directiva que de verdad lo requiere, y es por
  una librería externa, no por el código propio.
- **`img-src`** — permite los dos proveedores de teselas reales del
  código: `basemaps.cartocdn.com` (mapa base de Leaflet,
  `src/utils/cartoBasemap.js`, Territorio y Mapa de Seguimiento) y
  `tile.openstreetmap.org` (vista previa de capas geográficas
  KML/shapefile, `FilePreviewModal.jsx`) — revisados en el código, no
  adivinados. `blob:` es necesario para la vista previa de imágenes y
  PDF (`URL.createObjectURL`, ver `api/archivos.js`); `data:` para el
  ícono inline de `index.html`.
- **`font-src`/`style-src` con `fonts.googleapis.com`/`fonts.gstatic.com`**
  — la tipografía Noto Sans se carga de Google Fonts (`index.html`): la
  hoja de estilo viene de `fonts.googleapis.com` y el archivo de fuente
  en sí de `fonts.gstatic.com`, dos hosts distintos.
- **`media-src 'self' blob:'`** — vista previa de audio
  (`<audio src={blob:...}>`, mismo mecanismo de blob local que PDF/imagen).
- **`frame-src 'self' blob: https:`** — `blob:` para la vista previa de
  PDF (`<iframe src={blob:...}>`); `https:` porque una evidencia de tipo
  "enlace" se incrusta en un `<iframe>` apuntando a un sitio externo que
  el usuario capturó (`FilePreviewModal.jsx`) — una URL arbitraria que
  no se puede listar por dominio de antemano. Es, a propósito, la
  directiva más permisiva de la política: el costo de mantener esa
  función de incrustar enlaces externos.
- **`connect-src 'self'`** — todas las llamadas de `axios` van a la
  propia API (`/api/v1/...`, mismo origen vía nginx) y las notificaciones
  en tiempo real (SSE) también; ninguna petición `fetch`/`XHR` sale a
  otro origen.
- **`object-src 'none'`**, **`base-uri 'self'`**, **`form-action 'self'`**,
  **`frame-ancestors 'self'`** — endurecimiento estándar sin costo: el
  código no usa `<object>`/`<embed>`, no hay `<base>`, los formularios se
  envían por JS (nunca a otro origen), y no hay motivo para que otro
  sitio incruste esta plataforma en un `<iframe>` (protección adicional
  de *clickjacking*, redundante con `X-Frame-Options` pero cubriendo
  navegadores que solo miran CSP).

**Cómo se probó** (no solo "debería funcionar"): un build real de
`dist/` servido por nginx con esta cabecera exacta, abierto con
Chromium vía Playwright, viendo la consola del navegador. `/login` sin
ninguna violación de CSP (fuentes, estilos inline de React, scripts).
Un arnés aparte que carga Leaflet real con las dos URLs de teselas
reales del código (apuntadas a un servidor HTTPS local de prueba, mismos
nombres de host, para no depender de salida a internet) confirmó 24/24
peticiones de tesela en 200, Leaflet marcando cada una como cargada
(`tileload`), y cero violaciones de CSP en consola.

**Si se cambia algo del mapa o de las vistas previas**, repetir esta
prueba antes de dar por buena la política: abrir la consola del
navegador (F12 → Console) en una sesión real contra el nginx con la
cabecera aplicada, navegar el mapa y abrir una vista previa de cada
tipo (PDF, imagen, audio, capa geográfica, enlace externo), y confirmar
que no aparece ningún "Refused to..." ni "violates the following
Content Security Policy directive...".

### SELinux

Si el servidor tiene SELinux en modo `Enforcing` (verificar con
`getenforce`), nginx no puede servir `dist/` ni hablar con el backend
por el socket de red hasta que se le dé permiso explícito. El síntoma,
si se omite este paso, es **403 o 502 sin ninguna causa aparente en los
logs** — el archivo existe, los permisos de Unix están bien, y aun así
nginx no puede leerlo ni conectarse, porque SELinux lo bloquea en una
capa que no aparece en `ls -l`.

```bash
getenforce
# Si dice "Enforcing" (ajustar la ruta de dist si es distinta):
semanage fcontext -a -t httpd_sys_content_t "/opt/pspp/pspp-app/dist(/.*)?"
restorecon -Rv /opt/pspp/pspp-app/dist
setsebool -P httpd_can_network_connect 1
```

El primer comando etiqueta el contexto SELinux correcto para que nginx
pueda leer esos archivos; el segundo lo aplica a lo que ya existe en
disco; el tercero permite que nginx (el dominio `httpd_t`) inicie
conexiones de red salientes, que es lo que necesita para hacer
`proxy_pass` hacia el backend.

## 3. Modelo SEDATU (DGTIC) — nginx institucional

En este modelo el frontend se compila y se sirve nativamente, pero el
nginx que lo sirve es el de la institución, el backend está en **otro
servidor** (un solo contenedor, un puerto que asigna la DGTIC — ver
`pspp-api`, `deploy/docker-compose.sedatu.yml`), y normalmente todo va
bajo un subpath (p. ej. `/pspp/`), no en la raíz del dominio.

```bash
npm ci
VITE_BASE_PATH=/pspp/ VITE_API_URL=/pspp/api/v1 VITE_CARTO_API_KEY=... npm run build
# el resultado es dist/; copiarlo a la raíz que sirve el nginx institucional
```

- `VITE_BASE_PATH`: ruta bajo la que nginx publica el frontend
  (`/pspp/`, con ambas diagonales). Si la DGTIC asigna un dominio o
  subdominio propio, `/`.
- `VITE_API_URL`: ruta pública de la API, la que nginx manda al
  backend (`/pspp/api/v1`). Si el frontend va en la raíz, se omite
  (`/api/v1` por omisión).

**Pendiente conocido para servir bajo `/pspp/`:** algunas imágenes y
redirecciones del frontend usan rutas absolutas a la raíz
(`/sedatu-logo.png`, `/logo_pspp2.png`, la redirección a `/login` al
vencer la sesión, `/proyectos/nuevo`, `/docs/api-indicadores.md`) y no
respetan `VITE_BASE_PATH`. Bajo un subdirectorio esos logos no cargan y
esas redirecciones salen del subdirectorio. Con el frontend en la raíz
de un (sub)dominio no ocurre. Se atiende aparte de este cambio.

### Qué espera encontrar el nginx institucional

El backend no sirve archivos estáticos ni termina TLS; todo eso es del
nginx de la institución. Lo que necesita:

1. **Frontend**: servir `dist/` bajo la ruta base, con *fallback* a
   `index.html` (la aplicación usa rutas del lado del navegador; sin el
   fallback, recargar cualquier pantalla da 404).
2. **API**: `/pspp/api/` → `http://<dirección-del-backend>/api/`. La
   diagonal final de `proxy_pass` quita el prefijo `/pspp`: el backend
   solo conoce `/api/v1/...`.
3. **Notificaciones en tiempo real** (`/pspp/api/v1/notificaciones/stream`,
   Server-Sent Events): `proxy_buffering off` y `proxy_read_timeout`
   largo. Sin esto la campana de notificaciones deja de actualizarse
   sola, sin ningún error visible.
4. **Subidas**: `client_max_body_size` un poco mayor que
   `UPLOAD_MAX_SIZE_MB` del backend (210M para 200 MB: nginx mide el
   cuerpo completo, el backend solo el archivo) y
   `proxy_request_buffering off`, para que nginx no escriba primero su
   propia copia del archivo en disco. Valores mínimos y qué falla sin
   ellos: sección 4.
5. **Descargas y vistas previas**: `proxy_max_temp_file_size 0`, para
   que nginx no guarde en disco una copia de los archivos grandes que
   transmite el backend. nginx deja pasar `Range` sin configurar nada:
   el visor de PDF del navegador lo necesita y el backend responde 206.
6. **Cabeceras**: `X-Forwarded-For` / `X-Forwarded-Proto` / `Host`, como
   en cualquier proxy.
7. **Nada del almacén**: ninguna `location` apunta al almacén de
   objetos. El navegador nunca habla con él directamente.
8. `/health` no necesita publicarse: lo usa el healthcheck del
   contenedor del backend, desde dentro.
9. **`X-Forwarded-For` agregando, nunca reemplazando** (paso 9 DGTIC):
   en la infraestructura de la DGTIC la cadena es F5 → nginx
   institucional → backend, **dos saltos**. El backend necesita saber
   cuántos para identificar correctamente a cada persona en el límite
   de peticiones (`PROXY_SALTOS_CONFIABLES=2` en el entorno del
   backend — ver `pspp-api`, `deploy/README.md` sección 12) — si el F5
   o el nginx institucional **reemplazan** la cabecera en vez de
   agregarle su propio salto, o si el valor configurado no coincide con
   los saltos reales, el backend identifica mal a cada persona: en el
   peor caso, todo el tráfico cae en una sola IP y la primera ráfaga
   agota la cuota para todo el mundo.
10. **`Content-Security-Policy` sin modificar**: el backend no la pone
    (la pone este nginx, sección 2), pero si el F5 o un WAF
    institucional reescribe o elimina cabeceras de respuesta por
    política propia, la política deja de llegar al navegador — sin
    aviso ni error, el mapa y las vistas previas simplemente dejan de
    estar protegidas por CSP (aunque sigan funcionando igual, porque
    CSP es una capa de defensa adicional, no la única). Confirmar con
    las herramientas de desarrollador del navegador (pestaña Network →
    respuesta de `/pspp/` → cabecera `Content-Security-Policy`) después
    de cualquier cambio en el F5 o el WAF.

Configuración de referencia (validar con `nginx -t`; ajustar la
dirección del backend y la ruta de `dist`):

```nginx
# dentro del server {} institucional (TLS ya configurado ahí)

location = /pspp { return 301 /pspp/; }

location /pspp/ {
    alias /opt/pspp/pspp-app/dist/;
    try_files $uri $uri/ /pspp/index.html;
}

location /pspp/api/v1/notificaciones/stream {
    proxy_pass http://<dirección-del-backend>/api/v1/notificaciones/stream;
    proxy_http_version 1.1;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_buffering off;
    proxy_cache off;
    proxy_read_timeout 24h;
}

location /pspp/api/ {
    proxy_pass http://<dirección-del-backend>/api/;
    proxy_http_version 1.1;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    client_max_body_size 210M;      # > UPLOAD_MAX_SIZE_MB (cuerpo completo, no solo el archivo)
    client_body_timeout 60s;        # >= 60 s sin recibir nada antes de cortar
    proxy_request_buffering off;    # subidas: sin copia en disco de nginx
    proxy_max_temp_file_size 0;     # descargas: sin copia en disco de nginx
    proxy_read_timeout 300s;
    proxy_send_timeout 300s;
}
```

Con SELinux en `Enforcing`, aplican los mismos ajustes de la sección 2
(`httpd_sys_content_t` sobre `dist`, `httpd_can_network_connect`).

## 4. Requisitos del servidor web intermedio

Una subida pasa por todas las capas que hay entre el navegador y el
backend: en SEDATU, el F5, el nginx institucional y el contenedor. **La
más restrictiva decide.** Si una capa intermedia tiene un límite menor
que los de la plataforma, la subida falla en esa capa, aunque todo lo
demás esté bien, y casi siempre con un error que no dice cuál capa fue.

Esta sección es el contrato con quien opere esas capas. La plataforma
no las controla.

### 4.1 Los límites de la cadena

| Capa | Límite | Valor | Dónde se configura |
|---|---|---|---|
| Navegador | Petición normal | 30 s | `VITE_TIEMPO_ESPERA_MS` (`src/api/limites.js`) |
| Navegador | Operación de proceso (exportar, duplicar, importar) | 5 min | `VITE_TIEMPO_ESPERA_PROCESO_MS` |
| Navegador | Subida o descarga grande: se cancela si **deja de avanzar** | 60 s sin avance | `VITE_TRANSFERENCIA_SIN_AVANCE_MS` |
| Navegador | Subida o descarga grande: tope total | 60 min | `VITE_TIEMPO_MAXIMO_TRANSFERENCIA_MS` |
| Navegador | Tamaño de archivo (rechazo antes de subir) | 200 MB | `VITE_UPLOAD_MAX_SIZE_MB` |
| nginx | Tamaño del cuerpo completo | 210M | `client_max_body_size` |
| nginx | Espera entre dos lecturas del cuerpo | 60 s | `client_body_timeout` |
| nginx | Espera de respuesta del backend | 300 s | `proxy_read_timeout` |
| nginx | Espera entre dos envíos al backend | 300 s | `proxy_send_timeout` |
| nginx | Copia del cuerpo en disco | desactivada | `proxy_request_buffering off` |
| nginx | Copia de la respuesta en disco | desactivada | `proxy_max_temp_file_size 0` |
| Backend (`pspp-api`) | Tamaño de archivo (evidencias, adjuntos, portada) | 200 MB | `UPLOAD_MAX_SIZE_MB` (multer, solo el archivo) |
| Backend (`pspp-api`) | Archivo del importador de Excel/CSV | 10 MB | fijo en `src/routes/importar.routes.js` |
| Backend (`pspp-api`) | Cuerpo JSON | 10 MB | fijo en `src/server.js` (`express.json`) |
| Backend (`pspp-api`) | Cuerpo de formulario urlencoded | 100 KB | valor por omisión de Express; la plataforma no lo usa para archivos |
| Backend (`pspp-api`) | Recepción completa de una petición | 60 min | `TIEMPO_MAXIMO_PETICION_SEGUNDOS` (Node trae 300 s) |
| Backend (`pspp-api`) | Recepción de cabeceras | 60 s | valor por omisión de Node (`headersTimeout`) |

Por qué la cadena es coherente:

- **Tamaño**: el navegador rechaza más de 200 MB antes de subir nada. El
  backend acepta hasta 200 MB de archivo. nginx deja pasar 210M de cuerpo,
  que es el archivo más el sobre multipart.
- **Tiempo de una subida**: no hay un límite total corto en ninguna capa.
  Lo que corta es la falta de avance: 60 s en el navegador y 60 s en
  `client_body_timeout`. Un archivo grande en un enlace lento termina
  mientras siga avanzando, hasta el tope de 60 min, que es igual en el
  navegador y en el backend.
- **Después de subir**: el backend guarda el archivo en el almacén y
  contesta. nginx espera hasta 300 s (`proxy_read_timeout`).
- **Proceso largo**: el navegador espera 5 min, lo mismo que
  `proxy_read_timeout`. Esperar más en el navegador no sirve, porque nginx
  ya habría contestado 504.

> El nginx del VPS (repo `pspp-api`, `nginx/default.conf`, el de
> `docker-compose.prod.yml`) usa 200M y deja la copia en disco activada
> (el comportamiento por omisión). Funciona, porque nginx junta el
> archivo completo antes de pasarlo al backend. Lo único que no cabe es
> un archivo de entre ~199.9 y 200 MB.

> **Incidente real — por qué reiniciar nginx tras recrear el backend no
> es opcional (en el VPS Docker).** `nginx/default.conf` (repo
> `pspp-api`) usa `proxy_pass http://backend:3000;` con el nombre del
> servicio Docker directo, sin variable ni `resolver`. nginx resuelve
> ese nombre a una IP **una sola vez, al arrancar**, y la cachea para
> toda la vida del proceso. Al recrear el contenedor `backend` (nueva
> imagen, o solo reiniciarlo) Docker le asigna una IP interna nueva en
> su red — pero nginx sigue intentando hablar con la IP vieja, así que
> **toda la API responde 502** aunque el backend esté perfectamente
> sano. Pasó de verdad: al reconstruir el backend una segunda vez sin
> reiniciar nginx después, la plataforma quedó devolviendo 502 en todas
> las rutas durante varios minutos. No hay ningún error en `docker logs
> pspp-backend` (la petición nunca llega ahí) ni nada visible a simple
> vista: el frontend es estático, así que la pantalla de login sigue
> cargando con normalidad — la plataforma "parece arriba" aunque nada de
> la API funcione. Por eso el checklist de despliegue con Docker (ver
> `pspp-api`, `README.md`) trae `restart nginx` como paso
> **obligatorio** después de recrear `backend` o `frontend`, nunca
> opcional ni "solo si algo se ve raro". Esto es específico del VPS
> Docker: ni el despliegue nativo de este repo (nginx habla con
> `127.0.0.1:3100`, un puerto fijo, no una IP de contenedor) ni el
> modelo SEDATU (el nginx institucional tampoco resuelve un nombre de
> servicio Docker) tienen este riesgo. El principio general, para
> cualquier capa intermedia que sí lo tenga, está generalizado como
> requisito 13 más abajo.

### 4.2 Requisitos mínimos para cualquier servidor intermedio

Aplican a cada capa entre el navegador y el backend: nginx institucional,
F5, WAF, balanceador. Son mínimos: un valor mayor está bien.

| # | Requisito | Valor mínimo | Qué falla si no se cumple |
|---|---|---|---|
| 1 | Tamaño máximo del cuerpo de la petición en `/api/` (`client_max_body_size` en nginx; límite de petición o de carga en el F5 o WAF) | **210 MB** | Las subidas grandes se rechazan con **413** antes de llegar al backend. El usuario ve "El archivo excede el tamaño máximo que acepta el servidor" aunque el archivo esté por debajo de 200 MB. |
| 2 | Espera sin recibir datos del cliente (`client_body_timeout`; *idle timeout* del perfil TCP/HTTP en el F5) | **60 s** | Con enlaces lentos o con pausas de red, la subida se corta a media transferencia (408 o conexión reiniciada). |
| 3 | Espera de la respuesta del backend (`proxy_read_timeout`; *idle timeout* del lado servidor en el F5) | **300 s** | Exportar a Excel, duplicar un proyecto con archivos o confirmar una importación grande responden **504**. Las descargas grandes se cortan si el navegador del usuario lee despacio. |
| 4 | Espera al enviar datos al backend (`proxy_send_timeout`) | **300 s** | Las subidas largas se cortan cuando el backend tarda en aceptar datos. |
| 5 | **Duración total** de una petición, si la capa impone una | **60 min** | Cualquier subida que dure más que ese límite falla, por mucho que avance. Mejor no imponer duración total en `/api/`. |
| 6 | No guardar en disco una copia del cuerpo de la petición (`proxy_request_buffering off`) | desactivado | Con poco disco, nginx llena `client_body_temp` con cada subida (hasta 200 MB por subida simultánea) y empieza a responder 500 a todo. Además, la subida se mide dos veces: primero del usuario a nginx y luego de nginx al backend. |
| 7 | No guardar en disco una copia de la respuesta (`proxy_max_temp_file_size 0`) | desactivado | Mismo problema con las descargas y vistas previas de archivos grandes, en `proxy_temp`. |
| 8 | Ruta de notificaciones `/api/v1/notificaciones/stream` sin buffer de respuesta (`proxy_buffering off`) y con espera larga (`proxy_read_timeout 24h`) | buffer desactivado, 24 h | La campana de notificaciones deja de actualizarse sola. No hay error visible. |
| 9 | Dejar pasar sin modificar la cabecera `Authorization`, la cabecera `Range`, la *query string* (`?boleto=`) y las respuestas `206` | sin cambios | Sin `Authorization`, toda la API responde 401. Sin `?boleto=`, las descargas dan 401. Sin `Range` o sin 206, la vista previa de archivos de texto (que pide solo los primeros 2 MB) baja el archivo completo o falla. |
| 10 | No almacenar en caché respuestas de `/api/` | sin caché | Un usuario podría recibir datos o archivos de otro. Las respuestas de boleto ya llevan `Cache-Control: no-store`. |
| 11 | No inspeccionar ni bloquear por contenido los `multipart/form-data` de `/api/` (reglas de WAF sobre cargas de archivos) | sin bloqueo | El WAF rechaza evidencias legítimas (audio, ZIP de capas geográficas, PDF) con un error genérico de la institución, no de la plataforma. Si es obligatorio inspeccionarlos, el límite de tamaño de esa inspección también debe ser ≥ 210 MB. |
| 12 | Dejar pasar sin modificar (ni recortar, ni reemplazar) la cabecera `X-Forwarded-For`, agregando la IP real de quien se conecta en vez de sustituirla — cada capa que la toque debe **agregar**, nunca reescribir el valor completo | sin modificar, solo agregar | El backend usa esta cabecera para identificar a cada persona en el límite de peticiones. Si una capa la reescribe en vez de agregarle su propia IP, dos personas distintas pueden terminar compartiendo la misma IP "vista" por el backend — o, peor, la IP de la propia capa intermedia — y una ráfaga de una persona bloquearía a las demás. |
| 13 | Si esta capa resuelve la dirección del backend por nombre (DNS interno, descubrimiento de servicio) en vez de una IP o puerto fijo, debe volver a resolverla cuando el backend se recree, o reiniciarse junto con él | re-resolver o reiniciarse tras cada recreación del backend | Toda la API responde **502** aunque el backend esté sano y atendiendo — la capa intermedia se quedó con la dirección vieja. No hay ningún error en los logs del backend porque la petición nunca llega a él, y el frontend (estático, servido aparte) sigue respondiendo, así que la plataforma "parece arriba". Ver la nota de 4.1 sobre `nginx/default.conf` para un caso real de esto. |

Si se cambia `UPLOAD_MAX_SIZE_MB` (en `pspp-api`), mover junto con él
`VITE_UPLOAD_MAX_SIZE_MB` (recompilando este frontend) y el requisito 1
(`UPLOAD_MAX_SIZE_MB` + 10 MB). Si se cambia
`TIEMPO_MAXIMO_PETICION_SEGUNDOS`, mover junto con él
`VITE_TIEMPO_MAXIMO_TRANSFERENCIA_MS` y el requisito 5.

### 4.3 Cómo se ve cada falla desde la plataforma

El navegador traduce cada falla a un mensaje
(`src/api/transferencias.js`), que se muestra en el propio formulario o
en un aviso:

| Mensaje que ve el usuario | Causa probable |
|---|---|
| "El archivo pesa más de 200 MB, el máximo permitido." | El archivo de verdad excede el límite. No se subió nada. |
| "El archivo excede el tamaño máximo que acepta el servidor (200 MB)." | 413: alguna capa tiene menos que el requisito 1, o el archivo excede el límite del backend. |
| "La transferencia dejó de avanzar durante más de 60 segundos…" | Red del usuario caída o muy degradada, o una capa intermedia retiene la petición sin pasarla. |
| "El servidor cortó la petición porque tardó demasiado en llegar." | 408: requisitos 2 o 5, o `TIEMPO_MAXIMO_PETICION_SEGUNDOS`. |
| "El servidor tardó demasiado en responder…" | 504: requisito 3. |
| "Se perdió la conexión con el servidor…" | Conexión reiniciada a media petición: requisitos 2, 4 o 5 en un F5 o WAF que corta sin responder. |

Para probar la cadena completa, sube desde el navegador una evidencia de
unos 190 MB con la red limitada a ~2 Mbps (herramientas de desarrollo →
Red → limitación personalizada). Tarda unos 13 minutos y debe terminar
con la barra al 100 % y "Documento guardado".

## 5. Verificación

Con el backend (`pspp-api`) ya arriba y respondiendo en su puerto
directo:

```bash
# nginx sirve el frontend
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:8090/

# nginx hace el proxy correctamente al backend
curl -s http://localhost:8090/api/v1/auth/login -X POST \
  -H "Content-Type: application/json" \
  -d '{"correo":"no-existe@ejemplo.org","password":"x"}'
# Se espera un 401 con un JSON de error (credenciales inválidas), no un
# 502 (502 significa que nginx no pudo hablar con el backend) ni un 404
# (significa que la ruta /api/ no está llegando al backend).
```

Si da 502: revisar que `proxy_pass` apunte a donde de verdad escucha el
backend, y que ese backend esté activo. Si da 403 al servir el
frontend: revisar SELinux (sección 2).

Para el modelo SEDATU, el mismo criterio contra la URL pública real:

```bash
curl -s -o /dev/null -w "%{http_code}\n" https://sistemas.sedatu.gob.mx/pspp/
curl -s https://sistemas.sedatu.gob.mx/pspp/api/v1/auth/login -X POST \
  -H "Content-Type: application/json" \
  -d '{"correo":"no-existe@ejemplo.org","password":"x"}'
```

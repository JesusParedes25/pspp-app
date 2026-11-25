# pspp-app

Frontend de **PSPP v2.0** (Plataforma de Seguimiento de Proyectos
Prioritarios), SEDATU — secretaría federal mexicana de ordenamiento
territorial. Uso interno.

React 18 + Vite 5 + Tailwind, React Router v6, `@xyflow/react` (lienzo
de Diagrama), Leaflet (mapas). Este repositorio **solo compila
estáticos** — no tiene servidor propio ni lógica de negocio: todo eso
vive en el backend.

**Repositorio hermano:** la API vive en
[`pspp-api`](https://github.com/JesusParedes25/pspp-api), un repositorio
aparte, que también concentra el despliegue (Docker, nginx, systemd) y
la documentación formal — incluido el modelo de permisos
(`docs/modelo-permisos.md`, entregable DGTIC) y el manual de usuario.
Este repositorio solo documenta lo que es propio de servir el frontend:
compilar y la configuración de nginx que lo sirve (`deploy/`).

## Qué necesita para correr

- **Node.js 22**.
- Un backend (`pspp-api`) accesible por HTTP — en desarrollo, normalmente
  en `http://localhost:3000`.

## Desarrollo

```bash
npm install
npm run dev
```

Por omisión, el frontend llama a la API en la ruta relativa `/api/v1`
— en `npm run dev`, Vite no hace de proxy hacia el backend por sí solo;
para hablar con un backend en otra URL durante el desarrollo, fijar
`VITE_API_URL` (ver `.env` o exportar antes de `npm run dev`).

Con Docker, usando el `docker-compose.yml` de `pspp-api` (que levanta
también Postgres+PostGIS, MinIO y el backend): clonar este repositorio
**junto a** `pspp-api`, como `../pspp-app` relativo a `pspp-api` — ver
el README de `pspp-api`.

## Pruebas, lint y build

```bash
npm test          # Vitest
npm run test:e2e  # Playwright (e2e/) — necesita la app corriendo
npm run lint       # ESLint
npm run build      # genera dist/
```

## Estructura

```
src/
  pages/         una por pantalla
  components/    organizados por área (proyectos/, seguimiento/, admin/, ...)
  api/           funciones que llaman a la API (axios)
  hooks/         hooks compartidos (permisos, sesión, descargas...)
  context/       Auth, UI
  router/        definición de rutas
  utils/         formateo, cálculo de permisos del lado del cliente, generación de PDF/Excel
e2e/             specs de Playwright (test:e2e, no Vitest)
public/          estáticos servidos tal cual (logos, favicon)
deploy/          nginx (estáticos + proxy de /api/, CSP) y requisitos del
                 servidor web intermedio — ver deploy/README.md
```

## Despliegue

Ver [`deploy/README.md`](deploy/README.md): cómo compilar (`npm run
build` y las variables `VITE_*`), la configuración de nginx de
referencia (estáticos + proxy hacia el backend, notificaciones en
tiempo real, límites de subida, CSP, SELinux), el modelo SEDATU
(nginx institucional, backend en otro servidor, bajo un subpath), y los
requisitos mínimos para cualquier servidor web intermedio (nginx, F5,
WAF) entre el navegador y el backend.

El backend (`pspp-api`) se despliega por separado — ver su propio
`deploy/README.md`.

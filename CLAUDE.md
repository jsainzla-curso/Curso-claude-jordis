# CLAUDE.md

Resttek: plataforma de gestión de restaurantes. Monorepo con **npm workspaces** (`packages/*`):

| Paquete | Qué es | Puerto |
| --- | --- | --- |
| `api` | Backend Express 5 + TypeScript (ESM) + SQLite | 3000 |
| `web-admin` | Angular 21 — panel de administración (admin, manager) | 4200 |
| `web-empleados` | Angular 21 — cocina, barra y salón (cocinero, camarero, manager) | 4201 |
| `web-clientes` | Angular 21 — carta, carrito y pedidos (cliente) | 4202 |
| `web-shared` | Librería Angular común a los tres frontends (no se ejecuta sola) | — |

Documentación detallada en `docs/` (`arquitectura/`, `dominio/`, `revisiones/`). Consúltala antes de cambios de arquitectura; si cambias algo que contradiga lo documentado, actualiza también el doc.

## Comandos (desde la raíz)

```bash
npm install            # instala y enlaza todos los workspaces
npm run seed           # datos de prueba (idempotente, INSERT OR IGNORE)
npm run dev:api        # API con tsx watch → http://localhost:3000/health
npm run dev:admin      # / dev:empleados / dev:clientes
npm test               # tests de la API (Vitest); los frontends no tienen tests
```

Credenciales de prueba: la contraseña de cada usuario es su propio email (p. ej. `admin@resttek.com`). Lista completa en `README.md`.

## Lo común a los tres frontends

### Stack y estilo Angular

- Angular 21: **standalone components** (sin NgModules), **signals** para el estado, **zoneless** (ningún paquete usa `zone.js`).
- Control de flujo nuevo en plantillas: `@if` / `@for`, no `*ngIf` / `*ngFor`.
- Inyección con `inject()`, no por constructor. Servicios y stores con `@Injectable({ providedIn: 'root' })`.
- Guards e interceptors **funcionales** (`CanActivateFn`, `HttpInterceptorFn`).
- RxJS solo para HTTP. Iconos con `lucide-angular`: cada app registra los suyos con `LucideAngularModule.pick({...})` en `app.config.ts`; si usas un icono nuevo, añádelo ahí.
- Formato: 2 espacios, comillas simples. El uso de `;` no es uniforme entre apps: respeta el estilo del fichero que edites.
- Idioma: código en inglés; textos de interfaz y mensajes de error al usuario en castellano. Los valores de dominio se mantienen en castellano tal como los guarda la API (ver "Dominio").

### Arranque de cada app (`app.config.ts`)

Las tres apps configuran lo mismo:

```ts
{ provide: API_URL, useValue: environment.apiUrl },   // '/api/v1'
provideRouter(routes),
provideHttpClient(withInterceptors([authInterceptor, errorInterceptor])),
importProvidersFrom(LucideAngularModule.pick({ ... }))
```

### Rutas (`app.routes.ts`)

- `/login` → `LoginComponent` de `web-shared`.
- `''` → `ShellComponent` propio de la app (layout + navegación) protegido con `authGuard`; las páginas cuelgan como `children`.
- Lazy loading con `loadComponent` / `loadChildren`.

### Comunicación con la API

- Todas las peticiones van a rutas relativas `/api/v1/...`; en desarrollo las resuelve `proxy.conf.json` (idéntico en las tres apps) hacia `http://localhost:3000`. No escribas URLs absolutas al puerto 3000.
- Para la URL base, **inyecta `API_URL`** de `@resttek/web-shared` en los servicios nuevos. Los servicios antiguos de `web-admin` y `web-empleados` importan `environment.apiUrl`; funciona igual, pero no se puede sustituir en tests.
- El token JWT lo añade `authInterceptor`, y los 401 los gestiona `errorInterceptor`, que cierra la sesión y lleva a `/login`. No lo hagas a mano en los servicios.

### Librería compartida `@resttek/web-shared`

Se consume **desde el código fuente** (alias en el `tsconfig.json` de cada app → `../web-shared/src/index.ts`, sin paso de build). Exporta:

- `API_URL`: InjectionToken con la URL base.
- `AuthStore`: signals `token`, `user`, `isAuthenticated`, `userRole`; persiste en `localStorage` (`auth_token`, `auth_user`).
- `AuthService`: `login()`, `register()`, `logout()`.
- `authGuard`, `authInterceptor`, `errorInterceptor`.
- `LoginComponent`, `RegisterComponent` (este último solo lo usa `web-clientes`).

Reglas:

- Lo que necesiten dos o más apps va en `web-shared`, no copiado en cada app. Las apps no dependen entre sí ni tienen carpeta `shared/` propia.
- Todo lo nuevo de `web-shared` se exporta en `src/index.ts`.
- Tras cambiar `web-shared` hay que **reiniciar** el `ng serve` del frontend.
- `web-shared/src/lib/assets/` se copia a `/assets` de cada app (configurado en los `angular.json`); ahí está el logo.

### Estilos (design system)

Tema oscuro con acento verde, fuente Inter, variables CSS (`--bg-*`, `--text-*`, `--green-*`, `--radius`, …) y clases base (`.btn`, `.card`, `.form-control`, `.badge`, `.table`…). Usa esas variables y clases en lugar de colores fijos.

**Ojo:** `web-shared/src/lib/styles/base.css` no lo importa nadie. Cada app tiene su propia copia en `src/styles.css`, que es la que se carga (`web-empleados` añade algunas líneas propias). Si cambias un estilo común, replícalo en los tres `styles.css`.

### Estado

- `web-admin` y `web-empleados` usan el **patrón Store** por feature: signals privadas expuestas con `asReadonly()`, el trío `loading` / `error` / datos, el service HTTP convertido con `firstValueFrom` y los componentes hablando solo con el store.
- `web-clientes` solo tiene `CartStore` (local); sus componentes llaman a los servicios con `.subscribe()`. Es una decisión aceptada: no la "corrijas" sin pedirlo.
- No hay websockets: el tiempo real se hace con **polling** (30 s en `OrderStore` de empleados; 10 s y 5 s en los pedidos de clientes). Para el polling, limpia siempre el intervalo en `ngOnDestroy`.

## Dominio (común a todas las apps)

- **Roles** (valores exactos): `admin`, `manager` (en la interfaz se muestra como "gerente"; nunca uses `gerente` como valor), `cocinero`, `camarero`, `cliente`.
- **Los clientes son `Employee`** con `role = 'cliente'` y `restaurantId = null`. Por eso login y registro devuelven `{ token, employee }` en todas las apps.
- **Estados de ítem de pedido:** `pendiente` → `preparando` → `listo` → `entregado`. Cada unidad pedida es un `OrderItem` con `quantity: 1`.
- **Categorías de plato:** `entrante`, `principal`, `postre`, `bebida`. **Unidades de ingrediente:** `kg`, `g`, `l`, `ml`, `unidad`.
- El filtrado por rol en el frontend solo afecta a la navegación. Quien autoriza de verdad es la API (`authenticate` / `authorize()`); cualquier restricción real de permisos va en el backend.

Glosario y modelo de datos completos en `docs/dominio/`.

## API (resumen)

- ESM: los imports llevan extensión `.js` y usan path aliases (`@config/*`, `@services/*`, `@employee/*`…).
- Dos estilos conviven: hexagonal + DDD en `src/contexts/employee/`, y por capas (`models/`, `repositories/`, `services/`, `controllers/`, `routes/`) para el resto. Sigue el estilo del módulo que toques.
- Los errores de dominio heredan de `AppError` (`src/errors/DomainErrors.ts`); el `errorHandler` asigna el código HTTP según el nombre de la clase.
- Tests unitarios con Vitest junto al código (`*.test.ts`), con dobles en carpetas `mocks/`. Con `NODE_ENV=test` la base de datos es SQLite en memoria.

Detalle en `docs/arquitectura/arquitectura-api.md`.

# Platziflix

Plataforma de cursos online. **Monorepo poliglota con 4 artefactos desplegables por separado**, sin código compartido entre ellos:

| Artefacto | Stack | Puerto / Base URL |
|---|---|---|
| `Backend/` | FastAPI + PostgreSQL 15 | `:8000` — **única fuente de verdad de datos** |
| `Frontend/` | Next.js 15 App Router | `:3000` → `http://localhost:8000` |
| `Mobile/PlatziFlixAndroid/` | Kotlin + Jetpack Compose | → `http://10.0.2.2:8000` (emulador) |
| `Mobile/PlatziFlixiOS/` | Swift + SwiftUI | → `http://localhost:8000` |

La API REST es la **única** integración: no hay BFF, ni SDK generado, ni caché compartida. Los tres clientes son consumidores puros y stateless.

---

## Regla de oro: el contrato está replicado a mano en 4 lugares

No existe generación de código desde OpenAPI. **Cualquier cambio en la forma de un payload obliga a tocar los cuatro archivos**, o los clientes se rompen en silencio:

1. `Backend/app/schemas/` + los dicts que arma `Backend/app/services/course_service.py`
2. `Frontend/src/types/index.ts` y `Frontend/src/types/rating.ts`
3. `Mobile/PlatziFlixAndroid/.../data/entities/CourseDTO.kt`
4. `Mobile/PlatziFlixiOS/PlatziFlixiOS/Data/Entities/CourseDTO.swift`

`Backend/specs/00_contracts.md` es el contrato *original del diseño* y **ya no describe el sistema real** — no lo trates como fuente de verdad. Si lo consultas, verifica contra `main.py`.

---

## Trampas del dominio (leer antes de tocar modelos o endpoints)

### `Lesson` en la DB es `class` en la API

Hay **dos vocabularios para la misma entidad** y la traducción es implícita:

- DB / ORM: tabla `lessons`, modelo `Lesson`
- API y los 3 clientes: la llaman `class` / `classes`
- En `GET /classes/{class_id}` además se renombra `name` → `title`

No "corrijas" uno de los dos lados sin migrar los cuatro artefactos a la vez.

### `Backend/app/models/class_.py` es código muerto y roto

Define un modelo `Class` que **no debe importarse**: no está en `models/__init__.py`, declara `back_populates="classes"` contra una relación que `Course` no tiene, y apunta a una tabla `classes` que ninguna migración crea. Importarlo rompe el mapeo de SQLAlchemy al arrancar. Es candidato a borrarse.

### Soft delete es una invariante global

`BaseModel` (`Backend/app/models/base.py`) da `id`, `created_at`, `updated_at` y `deleted_at` a **todas** las entidades. Consecuencias obligatorias:

- **Nunca** hacer `DELETE` físico. Borrar = `deleted_at = datetime.utcnow()`.
- **Toda** query de lectura debe filtrar `.filter(X.deleted_at.is_(None))`. Omitirlo devuelve registros borrados.

---

## Comandos

### Backend — Docker es obligatorio

La API y la DB solo corren en Docker Compose. **Nunca ejecutes `alembic`, `pytest`, `python` o `uv` en el host**: la app espera el host `db` y `DATABASE_URL` del compose. Todo pasa por los targets del `Makefile`, que hacen `docker-compose exec api`.

Antes de ejecutar cualquier comando de backend: verifica que el contenedor `api` esté arriba y usa el target del `Makefile` (`make help` los lista). No inventes comandos nuevos — si falta uno, agrégalo al `Makefile`.

```bash
cd Backend
make start              # levantar db + api
make migrate            # aplicar migraciones (dentro del contenedor)
make create-migration   # crear migración (pide el mensaje interactivamente)
make seed-fresh         # limpiar y repoblar datos de prueba
make logs
```

### Frontend — yarn, no npm

Hay `yarn.lock` y `.npmrc`. Usar `yarn` siempre.

---

## Convenciones propias (difieren de los defaults)

**Backend**
- Las rutas de `main.py` **no tocan el ORM**: delegan en `CourseService` vía `Depends(get_course_service)`. Excepción conocida a corregir: `GET /classes/{class_id}` consulta `Lesson` directamente desde la ruta.
- `CourseService` ya es un god-object de ~400 líneas (cursos + todo el subsistema de ratings). Nueva lógica de ratings debería ir a un `RatingService` separado, no engordarlo más.
- Dependencias con `uv`, no pip.

**Frontend**
- Las llamadas nuevas a la API van en `src/services/*.ts` siguiendo el patrón de `ratingsApi.ts` (timeout con `AbortController`, `ApiError` tipado, `NEXT_PUBLIC_API_URL`). **No** agregar más `fetch` inline en Server Components — las páginas existentes lo hacen y es el patrón que estamos dejando atrás.
- Estilos: SCSS + CSS Modules (`Componente.module.scss` junto al componente). No hay Tailwind ni CSS-in-JS.
- TypeScript strict.

**Mobile — ambas apps usan la misma Clean Architecture de 3 capas**
```
Presentation (ViewModel + UiState + Views)
     ↓ depende de
Domain (modelos puros + interfaz/protocolo de repositorio)
     ↑ implementa
Data (DTO + Mapper + RemoteRepository + red)
```
- `Domain` **nunca** importa de `Data` ni de `Presentation`. Los DTO no cruzan a `Presentation`: siempre pasan por un Mapper.
- Android: MVI (`handleEvent(UiEvent)` + `StateFlow`), DI manual en `di/AppModule.kt`. El flag `USE_MOCK_DATA` permite correr sin backend.
- iOS: MVVM, DI por parámetro default (`= NetworkManager.shared`).

**Naming por lenguaje**: `snake_case` (Python, JSON de la API), `camelCase` (TS/Kotlin/Swift), `PascalCase` (tipos).

**Testing requerido** para funcionalidad nueva: `Backend/app/tests/` (pytest dentro del contenedor), Vitest + React Testing Library en Frontend, JUnit / XCTest en móvil.

---

## Deuda conocida — no re-descubrir, ya está diagnosticada

Estado a 2026-08-08. Si vas a trabajar en una de estas áreas, arregla el punto en el camino; si no, déjalo.

**Bloqueantes para producción**
- **No hay CORS** configurado en `Backend/app/main.py`. `ratingsApi.ts` corre en el navegador → toda escritura de rating desde el cliente será bloqueada.
- **No hay autenticación en ninguna capa.** `user_id` llega como entero arbitrario en body/path, sin tabla `users` ni FK. Cualquiera puede calificar como cualquiera.
- **URLs de desarrollo hardcodeadas en los 4 artefactos.** Solo `ratingsApi.ts` usa variable de entorno.

**Desalineaciones cliente ↔ servidor**
- `ratingsApi.getUserRating` llama `GET /courses/{id}/ratings/{userId}`; esa ruta solo acepta `PUT`/`DELETE`. El GET real es `/courses/{id}/ratings/user/{user_id}`.
- El backend señala "sin rating" con `HTTPException(204)` — un 204 no puede llevar body, y el cliente espera 404. `handleApiResponse` lo rechaza por falta de `content-type` en vez de devolver `null`.
- `generateMetadata` en `Frontend/src/app/course/[slug]/page.tsx` usa `courseData.title`; la API entrega `name` → título `undefined`.

**Rendimiento**
- `get_all_courses` llama `get_course_rating_stats` por curso, y cada llamada hace 3 queries → **1 + 3N consultas** en el listado principal. Se resuelve con un solo `GROUP BY course_id`.

---

## Paridad de features por plataforma

| Feature | Backend | Frontend | Android | iOS |
|---|---|---|---|---|
| Listado de cursos | ✅ | ✅ | ✅ | ✅ |
| Detalle por slug | ✅ | ✅ | ❌ | ✅ |
| Reproductor de video | ✅ | ✅ | ❌ | ❌ |
| Ratings (CRUD + stats) | ✅ | ✅ | ❌ | ❌ |
| Progreso / Quiz / Favoritos | ❌ | tipos declarados sin uso | ❌ | ❌ |

Al agregar una feature, decide y declara explícitamente si es solo-web o multiplataforma. Si es multiplataforma, los DTO de Android e iOS necesitan los campos nuevos.

---

## Reglas de trabajo

1. No hagas commit ni push sin que se te pida.
2. Cambios de schema → migración Alembic vía `make create-migration`. Nunca editar una migración ya aplicada.
3. Si un cambio toca el contrato REST, enumera explícitamente qué artefactos de los 4 quedan desincronizados.

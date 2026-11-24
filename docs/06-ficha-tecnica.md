# 📋 Ficha técnica: Jacobot 2.0

[⬅️ Volver al inicio](../README.md)

| | |
|---|---|
| **Producto** | Jacobot 2.0, agente empresarial de IA para Microsoft 365 |
| **Componente** | Tool Gateway (API HTTP) · versión 0.2.0 |
| **Estado** | Preproducción. Fases 0–3 construidas; **nunca ejecutado contra un Microsoft 365 real** |
| **Modelo de despliegue** | Una instalación por cliente (single-tenant) |
| **Fecha de la ficha** | 23/11/2026 · valores extraídos del código |

Leyenda: ✅ construido y probado en local (Microsoft 365 siempre simulado) · 🟡 escrito, sin ejecutar · ⏳ planificado

---

## 1. 🏗️ Plataforma

| Elemento | Especificación | Estado |
|---|---|---|
| Lenguaje | Python 3.12 | ✅ |
| Framework API | FastAPI 0.115.14 · Uvicorn 0.32.1 | ✅ |
| Validación | Pydantic 2.14.2, contratos con campos extra prohibidos | ✅ |
| ORM / migraciones | SQLAlchemy 2.0.55 · Alembic 1.20.1 | ✅ |
| Base de datos (producción) | PostgreSQL 17 · psycopg 3.2.13 | ✅ suite completa en 17.12 |
| Base de datos (local y pruebas) | SQLite; rechazada fuera de `local` y `test` | ✅ |
| Identidad | Microsoft Entra ID · PyJWT 2.15.1 · MSAL 1.39.0 | ✅ simulado |
| Microsoft 365 | Microsoft Graph v1.0 · httpx 0.28.1 | ✅ simulado |
| Contenedor | `python:3.12-slim`, usuario sin privilegios (uid 10001) | ✅ construida y arrancada en local |
| CI | GitHub Actions: ruff, mypy, pytest (SQLite y PostgreSQL), migraciones, pip-audit | 🟡 nunca ejecutada |
| Interfaz | Teams + Copilot Studio | ⏳ Fase 4 |
| Infraestructura | Azure + Bicep + Key Vault | ⏳ Fase 7 |

## 2. 🔌 API

Autenticación: `Authorization: Bearer` con token de usuario de Entra ID, salvo `/health`.

| Método | Ruta | Función | Rol |
|---|---|---|---|
| POST | `/v1/tools/create_project` | Preparar alta de proyecto | `Jacobot.ProjectCreator` |
| POST | `/v1/tools/archive_document` | Preparar archivado o movimiento | `Jacobot.User` |
| POST | `/v1/tools/rename_document` | Preparar cambio de nombre | `Jacobot.User` |
| GET | `/v1/projects` | Listar proyectos | `Jacobot.User` |
| GET | `/v1/projects/{id}/documents` | Documentos de un proyecto | `Jacobot.User` |
| GET | `/v1/documents/search` | Buscar por nombre | `Jacobot.User` |
| GET | `/v1/documents/{id}` | Detalle y versiones | `Jacobot.User` |
| GET | `/v1/intake` | Bandeja de entrada | `Jacobot.User` |
| POST | `/v1/operations/{id}/confirm` | Confirmar | Solicitante |
| POST | `/v1/operations/{id}/decision` | Aprobar o rechazar | `Jacobot.Approver`, distinto del solicitante |
| POST | `/v1/operations/{id}/execute` | Ejecutar | Solicitante |
| POST | `/v1/operations/{id}/reconcile` | Resolver resultado incierto | Solicitante |
| POST | `/v1/operations/{id}/cancel` | Cancelar | Solicitante |
| GET | `/v1/operations/{id}` | Consultar estado | Solicitante, o aprobador si está pendiente |
| GET | `/health` | Disponibilidad | Sin autenticación |

- Las rutas `tools` exigen `Idempotency-Key` (8–64 caracteres).
- La especificación OpenAPI solo se publica en `local`.

## 3. 📜 Acciones y riesgo

| Acción | Riesgo base | ¿Aprobación? |
|---|---|---|
| Crear proyecto | Bajo | Solo si el cliente lo configura |
| Archivar / mover | Medio | Sí, si la etiqueta es restringida **o ilegible** |
| Renombrar | Medio | Sí, si la etiqueta es restringida **o ilegible** |
| Lecturas | Bajo | No |

> ⚠️ Con Graph real y la configuración por defecto, **todas** las acciones de archivar y renombrar
> pedirán aprobación: leer etiquetas exige un permiso sobre toda la empresa que Jacobot no pide.

## 4. 🔐 Seguridad

| Control | Especificación |
|---|---|
| Firma del token | Solo RS256; se valida emisor, audiencia, tenant, `oid` y expiración (tolerancia 30 s) |
| Alcance exigido | `Jacobot.Invoke`, delegado; se rechazan tokens de aplicación |
| Permiso en Microsoft 365 | `Sites.Selected` delegado, concesión `write` por sitio |
| Intercambio de token | On-Behalf-Of; certificado obligatorio en producción |
| Integridad de operaciones | HMAC-SHA256 sobre operación, actor, parámetros y riesgo; se comprueba antes de mostrar, aprobar, ejecutar o reconciliar |
| Integridad de aprobaciones | HMAC-SHA256, un solo uso, con vencimiento, sin autoaprobación |
| Auditoría | Cadena HMAC-SHA256, escrita en la misma transacción que el estado |
| Ámbito de archivos | Bandeja de entrada y `Proyectos/{proyecto}/{categoría}`, por identificadores |
| Cambios concurrentes | `If-Match` (eTag) y verificación del destino antes de mover; bloqueos de base de datos en orden fijo |
| Arranque | Falla si no se declara el entorno; los errores de configuración no muestran claves |

## 5. 📏 Límites y tiempos

| Parámetro | Valor |
|---|---|
| Vigencia de una confirmación | 15 min |
| Vigencia de una aprobación | 24 h |
| Ejecución considerada atascada | 5 min |
| Caché del token de Graph | máx. 5 min, 512 entradas |
| Plazo por petición (token, llamadas y esperas) | 60 s; una respuesta a goteo puede excederlo |
| Tiempo de espera hacia Microsoft | máx. 30 s por fase de cada llamada (5 s para conectar); token: 10 s |
| Margen mínimo para iniciar una escritura | más de 10 s restantes |
| Intentos por llamada a Graph | máx. 4 |
| Espera total entre reintentos, por petición | máx. 30 s, sea cual sea la causa |
| Reintento de una operación incierta | hasta 3 días desde su creación |
| Llamadas a Graph por búsqueda | máx. 50, páginas incluidas |
| Resultados por búsqueda / listado de proyectos | 25 / 100 |
| Longitud de un nombre | 1–128 caracteres |

## 6. 🚦 Estados de una operación (12)

`DRAFT` · `VALIDATED` · `AWAITING_CONFIRMATION` · `AWAITING_APPROVAL` · `APPROVED` · `EXECUTING` ·
`SUCCEEDED` · `FAILED` · `UNKNOWN` · `RECONCILED` · `CANCELLED` · `EXPIRED`

## 7. 🗄️ Datos

| Tabla | Contenido |
|---|---|
| `operations` | Operaciones, estado, parámetros confirmados, evidencia |
| `approvals` | Decisiones firmadas |
| `audit_events` | Eventos de auditoría encadenados |
| `audit_head` | Último eslabón de la cadena, por tenant |

Jacobot no guarda el contenido de los documentos. Sí guarda nombres, identificadores, enlaces,
etiquetas y qué usuario hizo cada operación. Las lecturas se auditan por cantidad, sin nombres.

## 8. 🧪 Calidad (medido el 23/11/2026)

| Métrica | Valor |
|---|---|
| Pruebas distintas que pasan | **205** |
| Ejecuciones por base de datos | 305 en SQLite y 305 en PostgreSQL 17.12 (100 son las mismas pruebas repetidas sobre el adaptador HTTP) |
| Controles añadidos en esta ronda, quitados uno a uno | 38 de 38 hacen fallar alguna prueba |
| Imagen Docker | Construye, migra PostgreSQL y responde; sin token devuelve 401 |
| Pruebas contra Microsoft 365 real | 5 escritas, **0 ejecutadas** |
| ruff · mypy estricto | Sin errores (17 archivos) |
| Revisión de seguridad | 4 rondas con un **modelo de IA** auditor; la última: 0 altos, 2 medios y 6 bajos, todos corregidos |

Las revisiones de seguridad **no son un pentest externo** ni una certificación. La cuarta ronda
volvió a auditar las correcciones anteriores y encontró varias incompletas. Ya están completadas y
con prueba, pero **las correcciones de esa cuarta ronda no se han vuelto a auditar**.

## 9. ⚠️ Limitaciones conocidas

| | Limitación |
|---|---|
| ☁️ | Nada se ha probado en un Microsoft 365 real: permisos efectivos, revocación y tiempos están sin verificar |
| 🐘 | PostgreSQL y la imagen Docker solo se han probado en un equipo local; la CI no se ha ejecutado |
| 📒 | Quien pueda escribir en la base de datos puede **recortar el final** de la auditoría sin que se detecte; la solución está diseñada, no construida: requiere un ancla externa (Fase 7) |
| 🗄️ | Quien pueda escribir en la base de datos también puede alterar el estado, el resultado, el enlace y las fechas que muestra una operación |
| 🔑 | Las claves de firma viven en la memoria de la aplicación; con ellas y acceso a la base se podrían falsificar registros |
| 🔍 | La búsqueda es por nombre, no por contenido |
| 🗑️ | Sin borrado, subida de archivos ni restauración de versiones |
| 📝 | Si falla la escritura del Registro documental, se audita pero no se reintenta sola |
| 🚦 | Sin límite de peticiones por usuario |
| ⏳ | El plazo de 60 s no se ha comparado con el límite de espera de Copilot Studio |
| 🕵️ | Un 403 de Microsoft se devuelve como 403; podría revelar que existe un archivo ajeno. Sin verificar hasta tener tenant |

## 10. 📦 Requisitos del cliente

- Microsoft 365 con SharePoint y Entra ID.
- Un administrador que registre la app, dé el consentimiento y autorice el sitio.
- Roles `Jacobot.User`, `Jacobot.ProjectCreator` y `Jacobot.Approver` asignados.
- Fase 4: licencia de Copilot Studio. Fase 7: suscripción de Azure. **Costos sin calcular todavía.**

[⬅️ Anterior: 📚 Glosario](05-glosario.md) · [Volver al inicio 🏠](../README.md)

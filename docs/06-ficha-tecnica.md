# 📋 Ficha técnica: Jacobot 2.0

[⬅️ Volver al inicio](../README.md)

| | |
|---|---|
| **Producto** | Jacobot 2.0, agente empresarial de IA para Microsoft 365 |
| **Componente** | Tool Gateway (API HTTP) · versión 0.2.0 |
| **Estado** | Preproducción. Fases 0–3 construidas; **nunca ejecutado contra un Microsoft 365 real** |
| **Modelo de despliegue** | Una instalación por cliente (single-tenant) |
| **Fecha de la ficha** | 23/11/2026 · valores extraídos del código |

Leyenda: ✅ construido y probado con simulación · 🟡 escrito, sin ejecutar · ⏳ planificado

---

## 1. 🏗️ Plataforma

| Elemento | Especificación | Estado |
|---|---|---|
| Lenguaje | Python 3.12 | ✅ |
| Framework API | FastAPI 0.115.14 · Uvicorn 0.32.1 | ✅ |
| Validación | Pydantic 2.14.2, contratos con campos extra prohibidos | ✅ |
| ORM / migraciones | SQLAlchemy 2.0.55 · Alembic 1.20.1 | ✅ |
| Base de datos (producción) | PostgreSQL 17 · psycopg 3.2.13 | 🟡 sin probar |
| Base de datos (local y pruebas) | SQLite; rechazada fuera de `local` y `test` | ✅ |
| Identidad | Microsoft Entra ID · PyJWT 2.15.1 · MSAL 1.39.0 | ✅ simulado |
| Microsoft 365 | Microsoft Graph v1.0 · httpx 0.28.1 | ✅ simulado |
| Contenedor | `python:3.12-slim`, usuario sin privilegios | 🟡 imagen sin construir |
| CI | GitHub Actions: ruff, mypy, pytest, migraciones, pip-audit | 🟡 nunca ejecutada |
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
| Integridad de operaciones | HMAC-SHA256 sobre operación, actor, parámetros y riesgo |
| Integridad de aprobaciones | HMAC-SHA256, un solo uso, con vencimiento, sin autoaprobación |
| Auditoría | Cadena HMAC-SHA256, escrita en la misma transacción que el estado |
| Ámbito de archivos | Bandeja de entrada y `Proyectos/{proyecto}/{categoría}`, por identificadores |
| Cambios concurrentes | `If-Match` (eTag) y verificación del destino antes de mover |

## 5. 📏 Límites y tiempos

| Parámetro | Valor |
|---|---|
| Vigencia de una confirmación | 15 min |
| Vigencia de una aprobación | 24 h |
| Ejecución considerada atascada | 5 min |
| Caché del token de Graph | máx. 5 min, 512 entradas |
| Tiempo de espera hacia Microsoft | 30 s por llamada |
| Intentos por llamada a Graph | máx. 4 |
| Espera total por reintentos 429/503, por petición | máx. 30 s |
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
| Pruebas distintas que pasan | **169** |
| Ejecuciones totales | 267 (98 son las mismas pruebas repetidas sobre el adaptador HTTP) |
| Pruebas contra Microsoft 365 real | 5 escritas, **0 ejecutadas** |
| ruff · mypy estricto | Sin errores (17 archivos) |
| Revisión de seguridad | 3 rondas con un **modelo de IA** auditor |

Las revisiones de seguridad **no son un pentest externo** ni una certificación. Sus hallazgos se
corrigieron y sus pruebas pasan, pero las correcciones no se volvieron a auditar.

## 9. ⚠️ Limitaciones conocidas

| | Limitación |
|---|---|
| ☁️ | Nada se ha probado en un Microsoft 365 real: permisos efectivos, revocación y tiempos están sin verificar |
| 🐘 | PostgreSQL, la imagen Docker y la CI no se han ejecutado |
| 📒 | Quien pueda escribir en la base de datos puede **recortar el final** de la auditoría sin que se detecte; requiere un ancla externa (Fase 7) |
| 🔑 | Las claves de firma viven en la memoria de la aplicación; con ellas y acceso a la base se podrían falsificar registros |
| 🔍 | La búsqueda es por nombre, no por contenido |
| 🗑️ | Sin borrado, subida de archivos ni restauración de versiones |
| 📝 | Si falla la escritura del Registro documental, se audita pero no se reintenta sola |
| 🚦 | Sin límite de peticiones por usuario |
| ⏳ | El tope de 30 s cubre esperas por 429/503; los reintentos por cortes de conexión o timeouts no cuentan en él |

## 10. 📦 Requisitos del cliente

- Microsoft 365 con SharePoint y Entra ID.
- Un administrador que registre la app, dé el consentimiento y autorice el sitio.
- Roles `Jacobot.User`, `Jacobot.ProjectCreator` y `Jacobot.Approver` asignados.
- Fase 4: licencia de Copilot Studio. Fase 7: suscripción de Azure. **Costos sin calcular todavía.**

[⬅️ Anterior: 📚 Glosario](05-glosario.md) · [Volver al inicio 🏠](../README.md)

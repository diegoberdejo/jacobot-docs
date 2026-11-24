# ⚙️ Cómo funciona Jacobot por dentro

[⬅️ Volver al inicio](../README.md)

---

## 🧱 Las piezas

```mermaid
flowchart TB
    subgraph MS["☁️ Microsoft 365 del cliente"]
        TEAMS["💬 Teams"]
        CS["🧠 Copilot Studio"]
        ENTRA["🔐 Entra ID<br/><i>identidad</i>"]
        SP["📂 SharePoint"]
        LIST["📒 Lista<br/>Registro documental"]
    end

    subgraph AZ["🔷 Azure del cliente"]
        GW["🛡️ Jacobot Gateway<br/><i>FastAPI</i>"]
        DB[("🗄️ PostgreSQL<br/>operaciones · aprobaciones · auditoría")]
        KV["🔑 Key Vault<br/><i>secretos y claves</i>"]
    end

    TEAMS --> CS
    CS -- "acción del catálogo<br/>+ token del usuario" --> GW
    GW -- "valida token" --> ENTRA
    GW -- "OBO: token del usuario<br/>con Sites.Selected" --> ENTRA
    GW -- "Microsoft Graph" --> SP
    GW -- "fila operativa" --> LIST
    GW <--> DB
    KV -.-> GW
```

| Pieza | Qué hace | Analogía 🧑‍🍳 |
|---|---|---|
| 💬 **Teams** | Donde el empleado escribe | El mostrador |
| 🧠 **Copilot Studio** | Entiende el pedido y elige la herramienta | El mesero que toma la orden |
| 🛡️ **Gateway** | Verifica, pide confirmación, ejecuta y audita | El jefe de cocina que revisa cada orden |
| 🔐 **Entra ID** | Confirma quién es cada persona | El guardia de la puerta |
| 📂 **SharePoint** | Donde viven los documentos | La despensa |
| 🗄️ **PostgreSQL** | Memoria oficial de operaciones y aprobaciones | El libro de órdenes |
| 📒 **Registro documental** | Lista que la gente puede leer en SharePoint | La pizarra a la vista |

---

## 🔄 La vida de una solicitud

```mermaid
sequenceDiagram
    autonumber
    actor U as 👩 Ana
    participant T as 💬 Teams + 🧠 Copilot
    participant G as 🛡️ Gateway
    participant DB as 🗄️ Base de datos
    participant M as ☁️ SharePoint

    U->>T: "Archiva esta cotización en Migración SAP"
    T->>G: archive_document(documento, proyecto) + token
    G->>G: 🔐 valida token · 📜 revisa rol
    G->>M: 🔎 ¿dónde está? ¿existe el destino?
    G->>G: 🏷️ sugiere categoría "Cotizaciones"
    G->>DB: guarda operación + huella (digest)
    G-->>T: 👀 vista previa + huella
    T-->>U: "Archivaré … en Proyectos/Migración SAP/Cotizaciones. ¿Confirmas?"
    U->>T: ✅ Confirmar
    T->>G: confirm(huella)
    G->>G: 🔁 revalida rol, riesgo, etiqueta y destino
    G->>M: ⚙️ mueve el archivo (solo si no cambió: If-Match)
    G->>M: 🔎 lo vuelve a leer para comprobar
    G->>DB: ✅ SUCCEEDED + 📒 auditoría
    G->>M: 📝 fila en "Registro documental"
    G-->>T: resultado verificado + enlace
    T-->>U: "✅ Archivado en Proyectos/Migración SAP/Cotizaciones"
```

### 🔑 La "huella" (digest)

Cuando Jacobot prepara una operación, calcula una **huella criptográfica** de *exactamente*
lo que se va a hacer: qué archivo, qué destino, quién lo pide y con qué nivel de riesgo. La
confirmación y la aprobación quedan atadas a esa huella. Si alguien cambia algo después, aunque
sea directamente en la base de datos, la huella deja de coincidir y Jacobot se niega a ejecutar. 🚫

---

## 🚦 La máquina de estados

Cada operación pasa por estados explícitos. No hay atajos:

```mermaid
stateDiagram-v2
    [*] --> DRAFT
    DRAFT --> VALIDATED: ✅ contrato y permisos ok
    VALIDATED --> AWAITING_CONFIRMATION: 👀 vista previa
    AWAITING_CONFIRMATION --> APPROVED: ✅ el usuario confirma
    AWAITING_CONFIRMATION --> AWAITING_APPROVAL: 🧑‍⚖️ requiere aprobación
    AWAITING_APPROVAL --> APPROVED: 👍 otra persona aprueba
    AWAITING_APPROVAL --> CANCELLED: 👎 rechazada
    APPROVED --> EXECUTING: ⚙️ ejecutar
    EXECUTING --> SUCCEEDED: ✅ comprobado
    EXECUTING --> FAILED: ❌ no se aplicó
    EXECUTING --> UNKNOWN: ❓ no se sabe si se aplicó
    UNKNOWN --> SUCCEEDED: 🔎 reconciliado, sí se hizo
    UNKNOWN --> RECONCILED: 🔎 reconciliado, es seguro reintentar
    UNKNOWN --> FAILED: 🔎 otra persona cambió algo, o pasaron más de 3 días
    RECONCILED --> EXECUTING: 🔁 reintento seguro
    AWAITING_CONFIRMATION --> EXPIRED: ⏰
    AWAITING_APPROVAL --> EXPIRED: ⏰
    APPROVED --> EXPIRED: ⏰
    RECONCILED --> EXPIRED: ⏰
    AWAITING_CONFIRMATION --> CANCELLED: ❌
    SUCCEEDED --> [*]
    FAILED --> [*]
    CANCELLED --> [*]
    EXPIRED --> [*]
```

### ❓ ¿Por qué existe "UNKNOWN"?

Internet falla. A veces Microsoft **sí** movió el archivo, pero la respuesta se perdió por el
camino. Un sistema ingenuo reintentaría y crearía duplicados, o diría "falló" cuando en realidad
funcionó. Jacobot marca esos casos como **UNKNOWN** y, antes de reintentar, **reconcilia**: mira
en SharePoint qué pasó de verdad.

```mermaid
flowchart TB
    A["⚙️ Mover archivo"] --> B["📡 ...respuesta perdida..."] --> C["❓ UNKNOWN"]
    C --> D{"🔎 Reconciliar:<br/>¿dónde está el archivo?"}
    D -->|"✅ ya está en el destino"| E["SUCCEEDED"]
    D -->|"📍 sigue en el origen<br/>(hasta 3 días)"| F["RECONCILED<br/>reintento seguro"]
    D -->|"⚠️ alguien lo movió"| G["FAILED"]
```

---

## 🧑‍⚖️ Aprobaciones

```mermaid
sequenceDiagram
    actor A as 👩 Ana (solicita)
    participant G as 🛡️ Gateway
    actor B as 👨 Bruno (aprobador)

    A->>G: archivar "Contrato confidencial.pdf"
    G->>G: 🏷️ etiqueta restringida → riesgo ALTO
    G-->>A: 👀 vista previa · ⚠️ requiere aprobación
    A->>G: ✅ confirmo
    G->>G: ✍️ crea aprobación firmada
    G-->>B: 🔔 pendiente de aprobar
    B->>G: 👍 apruebo (misma huella)
    G->>G: ✔️ Bruno ≠ Ana · ✔️ rol aprobador · ✔️ no vencida · ✔️ firma intacta
    A->>G: ⚙️ ejecutar (con SU propio token)
    G->>G: ✔️ se usa una sola vez · 🔁 revalida todo
    G-->>A: ✅ hecho
```

Las reglas:
- 🚫 **Nadie se aprueba a sí mismo.**
- ✍️ **Cada aprobación va firmada**: si alguien la edita en la base de datos, se detecta.
- ⏰ **Vence** pasado un tiempo.
- 1️⃣ **Se usa una sola vez.**
- 🔑 **La ejecución usa el token de quien pidió**, nunca el del aprobador.

---

## 🗂️ La estructura de documentos

```
📂 Biblioteca de documentos (sitio autorizado)
├── 📥 Bandeja de entrada          ← archivos por archivar
└── 📁 Proyectos
    ├── 📁 Migración SAP
    │   ├── 📁 Cotizaciones        ← "Cotizacion_ABC_2026.pdf" va aquí
    │   ├── 📁 Specs técnicas
    │   ├── 📁 Prototipos
    │   └── 📁 Otros
    └── 📁 Implementación ERP
        └── ...
```

🔒 Jacobot **solo** puede tocar la 📥 Bandeja de entrada y las carpetas
`Proyectos/{proyecto}/{categoría}`. Lo comprueba subiendo por los **identificadores** de las
carpetas, no comparando rutas de texto: una ruta cambia si alguien renombra una carpeta, y el
identificador no. Fuera de esa zona, todo responde lo mismo: "no encontrado".

### 🏷️ Categoría sugerida

| Archivo | Sugerencia |
|---|---|
| `Cotizacion_ABC_2026.pdf` | 📁 Cotizaciones |
| `Presupuesto final.xlsx` | 📁 Cotizaciones |
| `Especificaciones técnicas.pdf` | 📁 Specs técnicas |
| `prototipo final.fig` | 📁 Prototipos |
| `scan_0042.pdf` | ❓ Jacobot **pregunta** la categoría |

La sugerencia **siempre** se confirma. Un documento confidencial nunca se mueve solo por una
predicción.

---

## 📒 Dos registros, dos propósitos

| | 📒 Registro documental | 🔏 Auditoría oficial |
|---|---|---|
| Dónde | Lista de SharePoint | Base de datos del Gateway |
| Para quién | Personas (consulta diaria) | Auditores, soporte, seguridad |
| ¿Editable? | Sí, por usuarios con permiso | No: cadena de firmas; editar o insertar se detecta (recortar el final, todavía no) |
| ¿Se usa para decidir? | ❌ Nunca | ✅ Es la fuente de verdad |

[⬅️ Volver al inicio](../README.md) · [Siguiente: 🛡️ Seguridad ➡️](02-seguridad.md)

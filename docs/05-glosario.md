# 📚 Glosario

[⬅️ Volver al inicio](../README.md)

Términos técnicos explicados sin jerga.

| | Término | Qué significa |
|---|---|---|
| 🏢 | **Tenant** | El "espacio" de una empresa dentro de Microsoft 365: sus usuarios, correos, SharePoint, Teams |
| 🔐 | **Entra ID** | El sistema de identidad de Microsoft (antes *Azure AD*). Sabe quién es cada persona |
| 🎟️ | **Token** | Un "pase" digital firmado por Microsoft que dice quién eres y qué puedes hacer. Caduca en minutos u horas |
| 🔄 | **On-Behalf-Of (OBO)** | Jacobot cambia tu pase por otro que le permite actuar **como tú** en SharePoint, con tus mismos permisos y nada más |
| 🔑 | **Sites.Selected** | Permiso de Microsoft que da acceso **solo** a los sitios que un administrador elige, no a todo SharePoint |
| ⚡ | **CAE** (*Continuous Access Evaluation*) | Permite que Microsoft corte el acceso antes de que caduque el pase si a alguien le revocan la cuenta |
| 🌐 | **Microsoft Graph** | La "puerta de entrada" para que un programa trabaje con SharePoint, Outlook, Teams, etc. |
| 🧠 | **Copilot Studio** | Herramienta de Microsoft para crear asistentes de IA que viven en Teams |
| 🛡️ | **Gateway** | El programa de Jacobot que se interpone entre la IA y Microsoft 365, revisando cada acción |
| 📜 | **Catálogo** | La lista cerrada de acciones que Jacobot sabe hacer. Lo que no está ahí, no se puede pedir |
| 👀 | **Vista previa** | Lo que Jacobot te muestra antes de hacer algo, para que confirmes |
| 🔏 | **Huella / digest** | Un código único calculado sobre lo que se va a hacer. Si algo cambia, la huella deja de coincidir |
| ✍️ | **Firma HMAC** | Un sello criptográfico que delata si alguien edita una aprobación o un evento de auditoría sin tener la clave |
| 🧑‍⚖️ | **Segregación de funciones** | Quien pide no puede ser quien aprueba |
| ❓ | **UNKNOWN** | Estado de una operación cuando no se sabe si Microsoft la ejecutó (por ejemplo, se cortó la conexión) |
| 🔎 | **Reconciliar** | Revisar en SharePoint qué pasó realmente antes de reintentar |
| 🔁 | **Idempotencia** | Mandar la misma petición dos veces no hace la acción dos veces |
| 🏷️ | **Etiqueta de confidencialidad** | Marca de Microsoft Purview que indica si un documento es público, interno o confidencial |
| 🎯 | **Prompt injection** | Intento de engañar a la IA con instrucciones escondidas en un documento o correo |
| 🧱 | **Zero Trust** | Principio de seguridad: no confiar en nada por defecto y verificarlo todo, siempre |
| 📤 | **DLP** (*Data Loss Prevention*) | Reglas que impiden que la información sensible salga por donde no debe |
| 🚪 | **Offboarding** | Retirar a un cliente de forma limpia: quitar permisos y borrar sus recursos |
| 🧪 | **Prueba live** | Prueba que corre contra un Microsoft 365 real, no contra una simulación |
| 🐳 | **Docker** | Forma de empaquetar la aplicación para que funcione igual en cualquier servidor |
| 🔷 | **Azure** | La nube de Microsoft donde se instalará el Gateway de cada cliente |

[⬅️ Anterior: 🗺️ Roadmap](04-roadmap.md) · [Siguiente: 📋 Ficha técnica ➡️](06-ficha-tecnica.md)

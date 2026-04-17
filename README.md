# Automatización Minutas Diarias — Nubceo

Digest diario de **reuniones con clientes** enviado a `bautista.lanusse@nubceo.com` a las **18:00 ART**, todos los días.  
Si no hubo reuniones con clientes ese día, no se envía nada.

---

## Credenciales Make.com

| Variable | Valor |
|---|---|
| Team ID | `1201957` |
| Gmail connection ID | `8153044` — `bautista.lanusse@nubceo.com` |
| Google Docs connection ID | `7470589` — `bautista.lanusse@nubceo.com` |
| Escenario ID | `4666085` |

---

## Flujo de Módulos

```
Módulo 1 (Gmail Search)
    ↓
Módulo 6 (Router)
    ├── Route A: cliente + tiene doc link → Módulo 2 (Google Docs) → Módulo 3 (Gemini con 2.text)
    └── Route B: cliente + sin doc link  → Módulo 7 (Gemini con 1.htmlBody)
    ↓
Módulo 4 (Aggregator — feeder: 1)
    ↓
Módulo 5 (Send Email — solo si aggregator no vacío)
```

### Módulo 1 — Gmail Search (`google-email:executeEmailSearchQuery v4`)

```json
{
  "connectionId": 8153044,
  "q": "from:gemini-notes@google.com newer_than:2d",
  "format": "full",
  "limit": 20,
  "filterType": "gmailSearch"
}
```

> `newer_than:2d` para capturar reuniones post-18hs que generan el email después del run diario.

### Módulo 6 — Router (`builtin:BasicRouter v1`)

Separa el flujo en dos rutas basadas en el tipo de email.

### Route A — Con Google Doc

**Filtro en Módulo 2 (AND):**

| # | Campo | Operador | Valor |
|---|---|---|---|
| 1 | `{{1.htmlBody}}` | contains | `formas parte de un grupo que se ha invitado` |
| 2 | `{{1.htmlBody}}` | contains | `docs.google.com/document/d/` |

**Módulo 2 — Google Docs (`google-docs:getADocument v1`)**

```json
{
  "connectionId": 7470589,
  "documentId": "first(split(last(split(1.htmlBody; \"docs.google.com/document/d/\")); \"/\"))",
  "filter": "image",
  "select": "map",
  "includeTabsContent": false
}
```

**Módulo 3 — Gemini AI (`gemini-ai:simpleTextPrompt v1`)** — usa `{{substring(2.text; 0; 28000)}}`

### Route B — Sin Google Doc (email body directo)

**Filtro en Módulo 7 (AND):**

| # | Campo | Operador | Valor |
|---|---|---|---|
| 1 | `{{1.htmlBody}}` | contains | `formas parte de un grupo que se ha invitado` |
| 2 | `{{1.htmlBody}}` | not contains | `docs.google.com/document/d/` |

**Módulo 7 — Gemini AI (`gemini-ai:simpleTextPrompt v1`)** — usa `{{substring(1.htmlBody; 0; 28000)}}`

### Módulo 4 — Aggregator (`builtin:BasicAggregator v1`)

```json
{
  "feeder": 1,
  "value": "{{replace(replace(if(3.result; 3.result; 7.result); \"```html\"; \"\"); \"```\"; \"\")}}"
}
```

> `if(3.result; 3.result; 7.result)` — toma el resultado de Route A o B según cuál corrió.

### Módulo 5 — Send Email (`google-email:sendAnEmail v4`)

**Filtro antes:** `join(map(4.array; "value"); "") != ""`

```json
{
  "connectionId": 8153044,
  "bodyType": "rawHtml",
  "subject": "Reuniones · {{formatDate(now; \"DD/MM/YYYY\")}}",
  "to": [{"name": "Bautista Lanusse", "address": "bautista.lanusse@nubceo.com"}]
}
```

---

## Cómo distinguir reuniones de clientes vs internas

Gemini Notes envía dos tipos de emails según el origen de la reunión:

| Tipo | Texto en htmlBody | Ejemplos |
|---|---|---|
| **Cliente** (se incluye) | `Has recibido este correo porque formas parte de un grupo que se ha invitado a la reunión.` | Dagma, Uca, Grupo Alas, Magazzino |
| **Interna** (se excluye) | `Estas notas se han enviado a los invitados de tu organización.` | Conciliador, Apollo, Desarrollo de cuentas |

El filtro `formas parte de un grupo que se ha invitado` en ambas rutas garantiza que solo pasen reuniones con clientes.

---

## Wrapper del Email

```html
<!DOCTYPE html><html lang="es"><head><meta charset="UTF-8"></head>
<body style="margin:0;padding:0;background:#f1f5f9;font-family:-apple-system,BlinkMacSystemFont,'Segoe UI',Arial,sans-serif;">
<div style="max-width:660px;margin:0 auto;padding:20px 12px;">
  <div style="background:linear-gradient(135deg,#0f172a,#1e3a5f);border-radius:12px 12px 0 0;padding:22px 26px;">
    <h1 style="margin:0 0 4px;font-size:18px;font-weight:800;color:#fff;">
      <span style="color:#38bdf8;">Nubceo</span> · Reuniones
    </h1>
    <p style="margin:0;font-size:12px;color:#94a3b8;">{{formatDate(now; "dddd D [de] MMMM [de] YYYY")}}</p>
  </div>
  <div style="padding:10px 0;">{{join(map(4.array; "value"); "")}}</div>
  <div style="background:#0f172a;border-radius:0 0 12px 12px;padding:12px 22px;text-align:center;">
    <p style="margin:0;font-size:10px;color:#475569;">Generado automáticamente · Google Meet + Gemini Notes</p>
  </div>
</div>
</body></html>
```

---

## Prompt Gemini (Módulos 3 y 7)

- Devolver ÚNICAMENTE HTML empezando con `<div` y terminando con `</div>`
- Sin markdown, sin backticks, estilos inline en todo
- Módulo 3: contenido desde `{{substring(2.text; 0; 28000)}}` (Google Doc)
- Módulo 7: contenido desde `{{substring(1.htmlBody; 0; 28000)}}` (email body directo)
- Fecha del email: `{{formatDate(1.date; "DD/MM/YYYY")}}`
- Fecha de hoy: `{{formatDate(now; "DD/MM/YYYY")}}`

**Secciones del card:**
1. **CABECERA:** dark gradient + nombre de reunión (`1.subject`). Si la fecha del email es distinta a hoy, badge: `📅 Reunión del [DD/MM/YYYY] — día anterior`
2. **RESUMEN:** máx. 2 oraciones
3. **TIEMPO DE HABLA:** timestamps por speaker. Si no hay: "Sin transcripción disponible"
4. **PREGUNTAS CLAVE:** las 3 más importantes que Nubceo le hizo al cliente
5. **DOLORES:** los 3 principales
6. **PRÓXIMOS PASOS:** tabla `Acción / Quien / Fecha`

---

## Scheduling

- **Todos los días a las 18:00 ART** (= 21:00 UTC)
- `newer_than:2d` asegura que reuniones post-18hs del día anterior también aparezcan

---

## Comportamiento — Reuniones post-18hs

El escenario corre a las 18:00 ART. Las reuniones que terminan después generan el email de Gemini Notes cuando el escenario ya corrió → aparecen en el **digest del día siguiente** con badge amarillo:

```
📅 Reunión del [DD/MM/YYYY] — día anterior
```

Caso que originó este cambio: reunión con Dagma (16/04/2026).

---

## Gotchas Críticos

1. **`bodyType: "rawHtml"` ES OBLIGATORIO** en `sendAnEmail v4` — sin esto el mail llega vacío.
2. **`join(map(4.array; "value"); "")`** — NO usar `join(4.array; "")`.
3. **Filtro cliente obligatorio:** `formas parte de un grupo que se ha invitado` — distingue reuniones con clientes de reuniones internas de Nubceo.
4. **Filtro doc obligatorio en Route A:** `docs.google.com/document/d/` — previene `[404] File not found: export.`
5. **Route B para emails sin doc link** (como Dagma): usa `1.htmlBody` directo, condición `text:notcontain` en `docs.google.com/document/d/`
6. **Aggregator value:** `if(3.result; 3.result; 7.result)` — toma resultado de whichever route corrió.
7. **Doc ID:** `first(split(last(split(1.htmlBody; "docs.google.com/document/d/")); "/"))`.
8. **Si Gemini devuelve \`\`\`html...\`\`\`** → `replace(replace(x; "```html"; ""); "```"; "")` en el aggregator.
9. **Si el escenario queda `isinvalid:true`** → crear uno nuevo, no se puede recuperar.
10. **El texto del doc está en `2.text`** (no `2.body` ni `2.content`).
11. **El run endpoint del MCP da 502** cuando el escenario tarda mucho. Usar **Run once** desde Make.com directamente.

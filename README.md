# Automatización Minutas Diarias — Nubceo

Digest diario de **todas las reuniones con Gemini Notes** enviado a `bautista.lanusse@nubceo.com` a las **18:00 ART**, todos los días.  
Si no hubo reuniones en el día, no se envía nada.

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

### Módulo 1 — Gmail Search (`google-email:executeEmailSearchQuery v4`)

```json
{
  "connectionId": 8153044,
  "q": "from:gemini-notes@google.com newer_than:1d",
  "format": "full",
  "limit": 20,
  "filterType": "gmailSearch"
}
```

### Módulo 2 — Google Docs (`google-docs:getADocument v1`) — con filtro antes

**Filtro:** `Solo reuniones con agenda.virtual` (AND de dos condiciones)

| # | Campo | Operador | Valor |
|---|---|---|---|
| 1 | `{{1.htmlBody}}` | contains | `agenda.virtual@nubceo.com` |
| 2 | `{{1.htmlBody}}` | contains | `docs.google.com/document/d/` |

> Condición 1: filtra solo emails de reuniones con clientes (el CC de Gemini Notes incluye `agenda.virtual@nubceo.com`).  
> Condición 2: previene el error `[404] File not found: export.` cuando algún email de Gemini no incluye link directo al doc.

```json
{
  "connectionId": 7470589,
  "documentId": "first(split(last(split(1.htmlBody; \"docs.google.com/document/d/\")); \"/\"))",
  "filter": "image",
  "select": "map",
  "includeTabsContent": false
}
```

> El texto limpio del doc está en `2.text`

### Módulo 3 — Gemini AI (`gemini-ai:simpleTextPrompt v1`)

- **model:** `gemini-2.5-flash`
- **Prompt:** ver sección [Prompt Gemini](#prompt-gemini) abajo

### Módulo 4 — Aggregator (`builtin:BasicAggregator v1`)

```json
{
  "feeder": 1,
  "value": "{{replace(replace(3.result; \"```html\"; \"\"); \"```\"; \"\")}}"
}
```

> Para leer el array: `join(map(4.array; "value"); "")` — NO `join(4.array; "")`.

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

## Prompt Gemini (Módulo 3)

- Devolver ÚNICAMENTE HTML empezando con `<div` y terminando con `</div>`
- Sin markdown, sin backticks, estilos inline en todo
- Contenido: `{{substring(2.text; 0; 28000)}}`
- Fecha del email: `{{formatDate(1.date; "DD/MM/YYYY")}}`
- Fecha de hoy: `{{formatDate(now; "DD/MM/YYYY")}}`

**Secciones del card:**
1. **CABECERA:** dark gradient + nombre de reunión (`1.subject`). Si la fecha del email es distinta a la fecha de hoy, agregar debajo del nombre un badge: `<span style="background:#f59e0b;color:#000;font-size:10px;font-weight:700;padding:2px 8px;border-radius:4px;letter-spacing:0.5px;">📅 Reunión del {{formatDate(1.date; "DD/MM/YYYY")}} — día anterior</span>`
2. **RESUMEN:** máx. 2 oraciones, qué se trató y cómo quedó el deal
3. **TIEMPO DE HABLA:** timestamps `0:01:23 Nombre:`, % por speaker (cap 90s). Si no hay: "Sin transcripción disponible"
4. **PREGUNTAS CLAVE:** las 3 más importantes que Nubceo le hizo al cliente
5. **DOLORES:** los 3 principales, una línea cada uno
6. **PRÓXIMOS PASOS:** tabla `Acción / Quien / Fecha`. Fecha solo si se mencionó, sino "A definir"

---

## Scheduling

- **Todos los días a las 18:00 ART** (= 21:00 UTC)
- Si no hay emails de Gemini Notes ese día → aggregator vacío → no se envía nada

---

## Comportamiento — Reuniones post-18hs

El escenario corre a las 18:00 ART. Las reuniones que **terminan después de esa hora** generan el email de Gemini Notes cuando el escenario ya corrió, por lo que aparecen en el **digest del día siguiente**.

Para identificarlas claramente, el Módulo 3 recibe:
- `FECHA DEL EMAIL: {{formatDate(1.date; "DD/MM/YYYY")}}`
- `FECHA DE HOY: {{formatDate(now; "DD/MM/YYYY")}}`

Si las fechas difieren, Gemini agrega en la cabecera del card un badge amarillo:

```
📅 Reunión del [DD/MM/YYYY] — día anterior
```

Así queda registro de todas las notas aunque la reunión haya cortado tarde. Caso que originó este cambio: reunión con Dagma (16/04/2026).

---

## Gotchas Críticos

1. **`bodyType: "rawHtml"` ES OBLIGATORIO** en `sendAnEmail v4` — sin esto el mail llega vacío.
2. **`join(map(4.array; "value"); "")`** — NO usar `join(4.array; "")`.
3. **Filtro en módulo 2 obligatorio (AND):** `agenda.virtual@nubceo.com` (filtra solo reuniones con clientes) + `docs.google.com/document/d/` (previene `[404] File not found: export.`)
4. **Doc ID:** `first(split(last(split(1.htmlBody; "docs.google.com/document/d/")); "/"))`.
5. **Si Gemini devuelve \`\`\`html...\`\`\`** → `replace(replace(x; "```html"; ""); "```"; "")` en el aggregator.
6. **Si el escenario queda `isinvalid:true`** → crear uno nuevo, no se puede recuperar.
7. **El texto del doc está en `2.text`** (no `2.body` ni `2.content`).
8. **El run endpoint del MCP da 502** cuando el escenario tarda mucho (6+ reuniones con Gemini). Usar **Run once** desde Make.com directamente.

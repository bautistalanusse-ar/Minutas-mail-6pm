# Automatización Minutas Diarias — Nubceo

Digest diario de reuniones **con clientes** enviado a `bautista.lanusse@nubceo.com` a las **18:00 ART**.  
Solo se envía si hubo al menos una reunión con cliente en el día.

## Contexto

- **Empresa vendedora:** Nubceo (dominio `@nubceo.com`)
- Google Meet genera automáticamente emails desde `gemini-notes@google.com` con un link a un Google Doc con el resumen + transcripción
- Se filtra **únicamente** reuniones donde `agenda.virtual@nubceo.com` está copiada (TO o CC) → son las reuniones con clientes
- Se envía **un solo email por día** con **todas las reuniones del día** concatenadas
- **Si no hay reuniones con clientes en el día, no se envía ningún email**

---

## Credenciales Make.com

| Variable | Valor |
|---|---|
| Team ID | `1201957` |
| Gmail connection ID | `8153044` (google-email) |
| Google Docs connection ID | `7470589` (google) |
| Escenario ID | `4666085` |

---

## Flujo de Módulos

### Módulo 1 — Gmail Search (`google-email:executeEmailSearchQuery v4`)

```json
{
  "connectionId": 8153044,
  "q": "from:gemini-notes@google.com newer_than:1d (to:agenda.virtual@nubceo.com OR cc:agenda.virtual@nubceo.com)",
  "format": "full",
  "limit": 20,
  "filterType": "gmailSearch"
}
```

> Filtra solo reuniones donde `agenda.virtual@nubceo.com` está presente → reuniones con clientes.  
> Si no hay resultados, el aggregator queda vacío y el email no se envía.

### Módulo 2 — Google Docs (`google-docs:getADocument v1`)

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

> Para leer el array después: `join(map(4.array; "value"); "")` — NO `join(4.array; "")`.

### Módulo 5 — Send Email (`google-email:sendAnEmail v4`)

```json
{
  "connectionId": 8153044,
  "bodyType": "rawHtml",
  "subject": "Reuniones · {{formatDate(now; \"DD/MM/YYYY\")}}",
  "to": [{"name": "Bautista Lanusse", "address": "bautista.lanusse@nubceo.com"}]
}
```

**Filtro antes de módulo 5:** `join(map(4.array; "value"); "") != ""`

> Si el módulo 1 no encontró emails (no hubo reuniones con clientes), el aggregator produce un array vacío → el filtro bloquea el envío. No se manda ningún email.

---

## Wrapper del Email (body del Módulo 5)

```html
<!DOCTYPE html><html lang="es"><head><meta charset="UTF-8"></head>
<body style="margin:0;padding:0;background:#f1f5f9;font-family:-apple-system,BlinkMacSystemFont,'Segoe UI',Arial,sans-serif;">
<div style="max-width:660px;margin:0 auto;padding:20px 12px;">
  <!-- HEADER -->
  <div style="background:linear-gradient(135deg,#0f172a,#1e3a5f);border-radius:12px 12px 0 0;padding:22px 26px;">
    <h1 style="margin:0 0 4px;font-size:18px;font-weight:800;color:#fff;">
      <span style="color:#38bdf8;">Nubceo</span> · Reuniones
    </h1>
    <p style="margin:0;font-size:12px;color:#94a3b8;">{{formatDate(now; "dddd D [de] MMMM [de] YYYY")}}</p>
  </div>
  <!-- CARDS DE REUNIONES -->
  <div style="padding:10px 0;">{{join(map(4.array; "value"); "")}}</div>
  <!-- FOOTER -->
  <div style="background:#0f172a;border-radius:0 0 12px 12px;padding:12px 22px;text-align:center;">
    <p style="margin:0;font-size:10px;color:#475569;">Generado automáticamente · Google Meet + Gemini Notes</p>
  </div>
</div>
</body></html>
```

---

## Prompt Gemini (Módulo 3)

**Instrucciones para Gemini:**
- Devolver ÚNICAMENTE HTML válido empezando con `<div` y terminando con `</div>`
- Sin markdown, sin backticks, sin texto antes ni después
- Estilos inline en todo
- Contenido del doc: `{{substring(2.text; 0; 28000)}}`

**Secciones del card:**

1. **CABECERA:** gradient dark header con el nombre de la reunión (`1.subject`)
2. **RESUMEN:** máximo 2 oraciones. Qué se trató y cómo quedó el deal. Sin introducciones.
3. **TIEMPO DE HABLA:** leer timestamps del doc (formato `0:01:23 Nombre:`), calcular % por speaker (cap 90s por intervención). Nota `* estimado`. Si no hay timestamps: `"Sin transcripción disponible"`.
4. **PREGUNTAS CLAVE:** las 3 más importantes que Nubceo le hizo al cliente. Sin timestamps.
5. **DOLORES:** los 3 principales — qué quiere el cliente que NO puede hacer sin Nubceo. Una línea cada uno. Sin timestamps.
6. **PRÓXIMOS PASOS:** tabla con columnas `Acción / Quien / Fecha`. Fecha solo si se mencionó explícitamente en la reunión, sino `"A definir"`. Sin timestamps.

---

## Scheduling

- **type:** `daily`
- **time:** `18:00`

> Make.com interpreta `"18:00"` como ART (UTC-3) → ejecuta a las 21:00 UTC

---

## Lógica de activación

```
¿Hay emails de gemini-notes con agenda.virtual@nubceo.com en TO/CC hoy?
  ├── SÍ → procesar cada reunión con Gemini → enviar digest a las 18:00
  └── NO → no ejecutar módulos 2-4, aggregator vacío, filtro bloquea envío → silencio
```

---

## Gotchas Críticos

1. **`bodyType: "rawHtml"` ES OBLIGATORIO** en `sendAnEmail v4` — sin esto el mail llega vacío.
2. **`join(map(4.array; "value"); "")`** — NO usar `join(4.array; "")` que devuelve `{object}{object}`.
3. **Doc ID extraction:** usar `first(split(...; "/"))` NO `regex replace("/.*"; "")` — el punto en Make.com no matchea `\n`.
4. **Si Gemini devuelve \`\`\`html...\`\`\`** usar `replace(replace(x; "```html"; ""); "```"; "")` en el aggregator.
5. **Si el escenario queda `isinvalid:true`** después de un error en runtime, hay que crear uno nuevo (no se puede recuperar).
6. **El campo de texto del Google Doc está en `2.text`** (no `2.body` ni `2.content`).
7. **Filtro de reuniones con clientes:** la query Gmail usa `(to:agenda.virtual@nubceo.com OR cc:agenda.virtual@nubceo.com)` — sin esto aparecen reuniones internas.

---

## Reconstrucción desde cero

Con Claude Code + MCP de Make.com, usar este README como prompt completo para reconstruir toda la automatización sin reproducir los bugs listados arriba.

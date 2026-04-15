# n8n Workflows

## `gemba-walk-automation.json`

Automatisiert den Gemba-Walk-Prozess:

```
Airtable Automation (on record create) ──HTTP POST──▶ n8n Webhook
  ▶ IF: Fotos vorhanden?
      ├── ja  → jedes Foto laden → base64 → Image-Block → Array
      └── nein → leeres Image-Array (Text-only-Analyse)
  ▶ Claude (Messages API, claude-opus-4-6) mit allen Bildern + Kontext
  ▶ Update Waste Observation (AI Analysis, Solution Approach, Status)
  ▶ Pro Action: User-Lookup (Role → User-Record) → Action Item anlegen
  ▶ E-Mail an die/den Verantwortliche/n (Fallback: EHS-Team)
```

### Import in n8n

1. *Workflows → Import from File* → `gemba-walk-automation.json` auswählen.
2. Credentials anlegen:
   - **Airtable Personal Access Token** (Scopes: `data.records:read`, `data.records:write`, `schema.bases:read`)
   - **HTTP Header Auth** für Anthropic: Header-Name `x-api-key`, Wert = API-Key
   - **SMTP Account** für Mail-Versand
3. *Settings → Variables*: `AIRTABLE_BASE_ID` setzen.
4. Workflow aktivieren und die **Production-Webhook-URL** aus dem `Webhook`-Node kopieren – sie wird im nächsten Schritt in der Airtable-Automation eingetragen.

### Airtable-Setup

#### Tabellen

- **Waste Observation** – wie gehabt; muss die Felder `AI Analysis` (Long text), `Solution Approach` (Long text) und `Status` (Single-Select, Option `Analyzed` hinzufügen) enthalten.
- **Action Items** – `Owner` muss ein **Linked Record** zur Tabelle `Users` sein (nicht mehr Freitext), alle anderen Felder wie im ursprünglichen CSV.
- **Users** *(neu)* – mindestens folgende Felder:
  | Feld    | Typ              | Zweck                                                        |
  |---------|------------------|--------------------------------------------------------------|
  | Name    | Single line text | Anzeigename                                                   |
  | Role    | Single select    | Werte exakt wie im Prompt: `Shopfloor-Leiter`, `EHS-Manager`, `Instandhaltung`, `Lagerleitung`, `Qualitätsmanagement` |
  | Email   | Email / Text     | Benachrichtigungs-Adresse                                     |
  | Active  | Checkbox *(opt.)*| Wenn gesetzt, können ehemalige Mitarbeiter:innen deaktiviert werden |

#### Automation: neues Record → Webhook

In Airtable *Automations → Create new automation*:

1. Trigger: **When record created** in `Waste Observation`.
2. Action: **Run script** mit folgendem Inhalt:

```javascript
let config = input.config();
let record = await base.getTable('Waste Observation').selectRecordAsync(config.recordId);

// Attachments → reines JSON-Array mit url/filename/type
const photos = (record.getCellValue('Photos') || []).map(p => ({
    url: p.url,
    filename: p.filename,
    type: p.type,
}));

const payload = {
    recordId: record.id,
    fields: {
        'Waste Observation Name': record.getCellValueAsString('Waste Observation Name'),
        'Waste Type':             record.getCellValueAsString('Waste Type'),
        'Location':               record.getCellValueAsString('Location'),
        'Impact':                 record.getCellValueAsString('Impact'),
        'Priority':               record.getCellValueAsString('Priority'),
        'EHS Concern':            !!record.getCellValue('EHS Concern'),
        'EHS Focus':              record.getCellValueAsString('EHS Focus'),
        'Description / Notes':    record.getCellValueAsString('Description / Notes'),
        'Observer':               record.getCellValueAsString('Observer'),
        'Date Observed':          record.getCellValueAsString('Date Observed'),
        'Photos':                 photos,
    },
};

await fetch('https://<dein-n8n-host>/webhook/gemba-walk', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(payload),
});
```

Unter *Input variables* `recordId` auf `Airtable record ID` aus dem Trigger mappen.

### Knotenübersicht

| # | Node | Zweck |
|---|------|-------|
| 1 | Webhook | Empfängt den POST aus der Airtable-Automation |
| 2 | IF: Fotos vorhanden? | Weiche: mit oder ohne Bilder |
| 3–6 | Split Photos → HTTP → Code (Image-Block) → Aggregate | Baut das `images`-Array für Claude |
| 7 | Code: Kein Foto | FALSE-Pfad: `images = []` – Text-only-Analyse |
| 8 | Code: Claude-Request bauen | Fasst Bilder + Kontext zum Anthropic-Payload zusammen |
| 9 | Claude: Messages API | Ein einziger Call, Modell `claude-opus-4-6` |
| 10 | Code: JSON parsen | Robustes Parsing der Claude-Antwort |
| 11 | Airtable Update | Schreibt AI Analysis + Solution Approach ins Waste-Record, setzt Status `Analyzed` |
| 12 | Split: Actions | Pro Claude-Action ein Item |
| 13 | Airtable Search (Users) | Lookup `owner_role` gegen `Users.Role` (`alwaysOutputData = true`, damit unbekannte Rollen nicht den Flow abbrechen) |
| 14 | Airtable Create (Action Items) | Setzt `Owner` = Linked-Record zum gefundenen User (leer, wenn kein Match) |
| 15 | Email | Benachrichtigt die/den Verantwortliche/n; Fallback an `ehs-team@example.com` |

### Text-only-Fall

Wenn ein Record ohne Foto reinkommt, läuft der FALSE-Pfad: `images = []`. Der Prompt signalisiert Claude explizit „es liegen keine Fotos vor – analysiere ausschließlich den Text". Ergebnis, Update und Action-Items werden identisch erzeugt, nur ohne Bildkontext.

### Weitere mögliche Ausbaustufen

- **Webhook-Auth**: In Produktion Basic Auth / Header-Token am Webhook-Node aktivieren und im Airtable-Script mitschicken.
- **Response-URL zurück an Airtable**: Parallel zu Node 11 einen Kommentar/Slack-Post in einer Kanban-View erzeugen.
- **Feedback-Schleife**: Status `Analyzed` → manuelle Freigabe → Status `Approved` triggert einen Folge-Workflow, der Aktionen ins Ticket-System (Jira, Linear) pusht.

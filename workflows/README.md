# n8n Workflows

## `gemba-walk-automation.json`

Automatisiert den Gemba-Walk-Prozess:

```
Formular (Airtable) → neuer Record → Claude Vision-Analyse →
  → AI Analysis + Solution Approach zurück ins Record
  → Action Items (eigene Tabelle) inkl. Owner & Priorität
  → E-Mail an Verantwortliche
```

### Import in n8n

1. In n8n: *Workflows → Import from File* → `gemba-walk-automation.json` auswählen.
2. Credentials anlegen (IDs im Workflow sind Platzhalter):
   - **Airtable Personal Access Token** (Scopes: `data.records:read`, `data.records:write`, `schema.bases:read`)
   - **HTTP Header Auth** für Anthropic – Header-Name `x-api-key`, Wert = API-Key
   - **SMTP Account** für Mail-Versand
3. Unter *Settings → Variables*: `AIRTABLE_BASE_ID` mit Deiner Base-ID befüllen.
4. Tabellen in Airtable:
   - `Waste Observation` mit Feldern u.a. `Photos` (Attachment), `AI Analysis` (Long text), `Solution Approach` (Long text), `Status` (Single select inkl. Option `Analyzed`).
   - `Action Items` mit `Action Description`, `Action Notes`, `Waste Observation` (Linked record), `Owner`, `Priority`, `Status`, `Deliverable Date`.

### Knotenübersicht

| # | Node | Zweck |
|---|------|-------|
| 1 | Airtable Trigger | Pollt `Waste Observation` auf neue Records |
| 2 | IF | Prüft, ob ein Foto vorhanden ist |
| 3 | HTTP Request | Lädt das erste Foto als Binary |
| 4 | Extract From File | Wandelt Binary → Base64 |
| 5 | HTTP Request (Claude) | Sendet Bild + Kontext an `claude-opus-4-6`, erzwingt JSON-Antwort |
| 6 | Code | Parst die JSON-Antwort |
| 7 | Airtable Update | Schreibt `AI Analysis` + `Solution Approach` ins Ursprungs-Record |
| 8 | Split Out | Teilt das `actions`-Array in einzelne Items |
| 9 | Airtable Create | Legt pro Action einen `Action Items`-Record inkl. Linked-Record zur Observation an |
| 10 | Send Email | Benachrichtigt die/den Verantwortliche/n |

### Anpassungen, die Du wahrscheinlich brauchst

- **Owner-Zuordnung**: Aktuell liefert Claude eine Rolle (`Shopfloor-Leiter`, `EHS-Manager`). Für echte User-Links solltest Du in Airtable eine `Users`-Tabelle pflegen und vor Node 9 einen Lookup einbauen, der die Rolle auf einen konkreten Record mappt.
- **Mehrere Fotos**: Der Workflow nutzt nur `Photos[0]`. Für Multi-Foto-Analyse den HTTP-Request über alle Attachments loopen und als Array an die Claude-API schicken.
- **Webhook statt Polling**: Für Echtzeit-Reaktion in Airtable eine Automation `When record created → Send webhook` anlegen und Node 1 durch einen `Webhook`-Trigger ersetzen.

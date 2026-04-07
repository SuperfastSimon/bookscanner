# Book Scanner Pro — Blueprint

## Doel
Een single-page webapplicatie (één HTML-bestand) die boekfoto's uit Google Drive of lokale bestanden verwerkt met AI Vision, boekmetadata extraheert en prijzen schat, en alles exporteert naar Excel. Gehost op GitHub Pages (geen server, geen backend).

**Live URL:** https://superfastsimon.github.io/bookscanner  
**Repo:** https://github.com/SuperfastSimon/bookscanner  
**Enige bestand:** `index.html` (~780 regels)

---

## Architectuur

```
index.html
├── <head>
│   ├── CSP meta tag (security)
│   ├── xlsx.full.min.js  (Cloudflare CDN, SRI hash)
│   └── heic2any.min.js   (jsDelivr CDN, geen SRI)
│
├── <style>  — alle CSS inline, dark theme, CSS variables
│
├── HTML (4 tabs)
│   ├── Tab 1: API Keys
│   ├── Tab 2: Drive laden
│   ├── Tab 3: Verwerken
│   └── Tab 4: Resultaten
│
└── <script>
    ├── Google GSI client  (accounts.google.com/gsi/client, synchroon geladen)
    └── App JS (inline)
```

**Geen bundler, geen framework, geen server.** Alles draait client-side in de browser.

---

## Tab 1 — API Keys

### Invoervelden
| ID | Type | Doel |
|----|------|------|
| `k-openai` | password | OpenAI key (sk-...) |
| `k-mistral` | password | Mistral key |
| `k-groq` | password | Groq key (gsk_...) |
| `k-google` | text | Google OAuth Client ID |

### Gedrag
- **localStorage persistentie:** alle keys worden opgeslagen bij elke `input` event (prefix `bsp_`), hersteld bij `DOMContentLoaded`
- **Test-knop:** roept de `/models` endpoint aan van elke API en toont geldig/ongeldig
- **Google login:** OAuth 2.0 via Google Identity Services (GSI), scope `drive.readonly`, token opgeslagen in `gAccessToken` (globaal). Token verloopt na 1 uur — Client ID blijft opgeslagen zodat herlogin 1 klik is.
- **Key-status** (topbar): toont hoeveel keys/logins actief zijn

### Relevante functies
```
saveKey(name)        — schrijft input naar localStorage
loadSavedKeys()      — laadt bij DOMContentLoaded
testKey(name)        — test API bereikbaarheid
googleLogin()        — initialiseert OAuth token client, vraagt access token
```

---

## Tab 2 — Drive laden

### Twee invoerbronnen

**A. Google Drive (primair)**
1. Plak Drive-map URL → `parseDriveUrl()` extraheert folder ID(s) via regex `/\/folders\/([a-zA-Z0-9_-]{10,})/g`
2. Klik "Foto's laden" → `loadDrive()` → `loadImagesFromFolders()` → `collectImagesRecursive()`
3. Recursie doorzoekt alle submappen op alle niveaus
4. Bestanden worden herkend op MIME type (`image/*`) OF bestandsextensie (zie IMAGE_EXTS)

**B. Lokaal bestand**
- `<input type="file" multiple>` zonder `accept` attribuut (anders opent Android alleen Google Foto's)
- Gefilterd in JS op extensie via IMAGE_EXTS

### Kritieke gotcha: HEIC MIME type
Google Drive slaat iPhone HEIC-bestanden op als `application/octet-stream`, niet als `image/heic`. Detectie dus **altijd op extensie**, nooit alleen op MIME type.

```javascript
var IMAGE_EXTS = /\.(jpe?g|png|gif|webp|bmp|heic|heif|tiff?|avif)$/i;

// In collectImagesRecursive:
var images = items.filter(function(f){
  return (f.mimeType && f.mimeType.startsWith('image/')) || IMAGE_EXTS.test(f.name || '');
});
```

### Drive API
```
GET https://www.googleapis.com/drive/v3/files
  ?q='<folderId>' in parents and trashed=false
  &fields=nextPageToken,files(id,name,mimeType)
  &pageSize=200
  &pageToken=<pt>
Authorization: Bearer <gAccessToken>
```
Gepagineerd (pageSize 200), loopt door totdat `nextPageToken` leeg is.

### Queue-item structuur
```javascript
// Drive bestand:
{ name: 'IMG_001.HEIC', type: 'drive', id: '<driveFileId>' }

// Lokaal bestand:
{ name: 'foto.jpg', type: 'local', file: <File object> }
```

### Relevante functies
```
driveList(folderId, mimeFilter)         — pagineerde Drive API call
collectImagesRecursive(folderId, name)  — recursief alle submappen doorzoeken
loadImagesFromFolders(folderIds)        — reset teller, roept recursie aan
loadDrive()                             — validatie + aanroep
addFiles(files)                         — lokale bestanden toevoegen
showQueue()                             — wachtrij UI updaten
clearQueue()                            — wachtrij leegmaken
```

---

## Tab 3 — Verwerken

### Instellingen
| Element | Default | Doel |
|---------|---------|------|
| `opt-prices` | "all" | Elk boek / Alleen ISBN / Niet opzoeken |
| `opt-delay` | 1500ms | Vertraging tussen boeken (rate limit bescherming) |
| `opt-autosave` | 25 | Auto-export elke N boeken |

### Verwerkingslus (`startProc`)
```
voor elke item in queue:
  1. getImageBase64(item)     — download + HEIC-conversie
  2. analyzeBookImage(base64) — AI vision analyse
  3. fetchPrices(result)      — prijsschatting (optioneel)
  → addResultRow(result)
  → updateStats()
  → sleep(delay)
  → auto-save elke N boeken
```

### HEIC-conversie (`blobToJpegBase64`)
```
Poging 1: Canvas API
  → createObjectURL(blob) → <img>.onload → canvas.toDataURL('image/jpeg')
  → Werkt native op Safari/iOS
  → Werkt voor JPEG/PNG/WebP overal

Poging 2 (als canvas mislukt): heic2any library
  → heic2any({blob, toType:'image/jpeg', quality:0.85})
  → Werkt voor HEIC op Chrome/Firefox
  → Duurt 5-15s per foto op mobiel
  → Logt voortgang: "HEIC → JPEG omzetten..."
```

### AI Vision (`analyzeBookImage`)

**Primair: OpenAI gpt-4o-mini**
```
POST https://api.openai.com/v1/chat/completions
model: gpt-4o-mini
max_tokens: 300
content: [
  { type: 'text', text: VISION_PROMPT },
  { type: 'image_url', image_url: { url: 'data:image/jpeg;base64,...', detail: 'low' } }
]
```
Kosten: ~$0.000085 per foto (detail:low)

**Fallback: Mistral pixtral-12b-2409**
```
POST https://api.mistral.ai/v1/chat/completions
model: pixtral-12b-2409
content: [
  { type: 'text', text: VISION_PROMPT },
  { type: 'image_url', image_url: 'data:image/jpeg;base64,...' }
]
```

### Vision prompt
```
Analyseer deze boekcover. Geef ALLEEN JSON terug (geen uitleg), exact dit formaat:
{"title":"","author":"","isbn":"","year":"","publisher":"","lang":"nl/en/de/fr/other",
 "type":"roman/non-fictie/kinderboek/studieboek/other","condition":"goed/redelijk/slecht","confidence":0.9}
- isbn: alleen als duidelijk zichtbaar op cover/rug, anders leeg string
- year: alleen als zichtbaar, anders leeg string
- confidence: 0.0-1.0
```
Antwoord wordt geparsed via regex `/\{[\s\S]*\}/` zodat extra tekst rond de JSON geen problemen geeft.

### Rate limit afhandeling (`fetchWithRetry`)
- Leest `Retry-After` header
- Exponential backoff: 2s → 4s → 8s → 16s → 30s max
- Max 4 pogingen per aanroep
- Logt elke wachtstap zichtbaar in logbox

### Prijsschatting (`fetchPrices`)
- Optie "none": slaat over
- Optie "isbn": slaat over als geen ISBN
- Gebruikt OpenAI gpt-4o-mini of Groq llama-3.1-8b-instant
- Prompt: vraagt om prijsrange zoals "€3-8" voor Marktplaats/bol.com NL

### Resultaat-object
```javascript
{
  name: 'IMG_001.HEIC',
  title: 'De Ontdekking van de Hemel',
  author: 'Harry Mulisch',
  isbn: '9789023401920',
  year: '1992',
  publisher: 'De Bezige Bij',
  lang: 'nl',
  type: 'roman',
  condition: 'goed',
  priceInfo: '€4-10',
  status: 'ok',   // of 'err'
  err: ''
}
```

### UI elementen (tab 3)
| ID | Inhoud |
|----|--------|
| `st-tot` | Totaal in wachtrij |
| `st-done` | Verwerkt |
| `st-err` | Fouten |
| `st-isbn` | Boeken met ISBN |
| `st-pct` | Voortgang % |
| `pfill` | Progressiebalk (width%) |
| `prog-lbl` | "X / Y verwerkt" |
| `prog-eta` | Geschatte resterende tijd |
| `curbook` | Huidig bestand |
| `logbox` | Scrollend logvenster (monospace) |
| `cost-lbl` | Kosten in dollar |
| `save-lbl` | Auto-save status |

Log CSS klassen: `.lg` (groen=OK), `.le` (rood=fout), `.li` (blauw=info)

---

## Tab 4 — Resultaten

- Tabel met alle resultaat-velden
- **Download Excel** → `doExport(false)` → XLSX.js → `boeken_YYYY-MM-DD.xlsx`
- Auto-export bij voltooiing en elke N boeken (ook `doExport(true)`, zonder toast)
- Kolombreedte ingesteld per kolom
- **Wissen** → `clearRes()` leegt resultaten array + tabel

---

## Security

### Content Security Policy (meta tag)
```
default-src 'self'
script-src 'self' 'unsafe-inline'
  https://cdnjs.cloudflare.com
  https://accounts.google.com
  https://cdn.jsdelivr.net
connect-src
  https://api.openai.com
  https://api.mistral.ai
  https://api.groq.com
  https://www.googleapis.com
  https://accounts.google.com
  https://openlibrary.org
style-src 'self' 'unsafe-inline'
img-src 'self' data: blob:
frame-ancestors 'none'
```

### SRI hash
xlsx CDN heeft SRI: `integrity="sha384-OLBgp1GsljhM2TJ+sbHjaiH9txEUvgdDTAzHv2P24donTt6/529l+9Ua0vFImLlb"`  
heic2any heeft geen SRI (dynamisch geserveerd door jsDelivr)

### API keys
Nooit in de repo. Alleen in `localStorage` (prefix `bsp_`). Publieke repo is veilig.

---

## Globale staat (JS variabelen)

```javascript
var gAccessToken = null;      // Google OAuth access token (verloopt 1u)
var gTokenClient = null;      // Google OAuth token client object
var queue = [];               // Array van te verwerken items
var results = [];             // Array van verwerkte resultaten
var running = false;          // Verwerkingslus actief
var paused = false;           // Gepauzeerd
var totalCost = 0;            // Cumulatieve API kosten ($)
var t0 = null;                // Starttijdstip verwerking
var selectedFolderIds = [];   // Geselecteerde Drive folder IDs
var _collectCount = 0;        // Teller tijdens Drive scan
```

---

## Bekende beperkingen / aandachtspunten

1. **HEIC op Chrome mobiel:** heic2any duurt 5-15s per foto. Geen workaround — is de snelste JS-oplossing.
2. **Mistral rate limit:** gratis tier heeft lage RPM. `fetchWithRetry` handelt dit af maar bij grote batches is OpenAI sneller.
3. **Google OAuth token verloopt:** na 1 uur moet de gebruiker opnieuw inloggen (1 klik).
4. **Geen server:** alle API-aanroepen gaan direct vanuit de browser. CORS is toegestaan door alle gebruikte APIs.
5. **Tab 3 logbox zichtbaar tijdens Drive laden (tab 2):** logbox zit op tab 3, Drive-scan logt ook daarheen. Gebruiker ziet de logs pas als ze naar tab 3 gaan.

---

## Externe dependencies

| Library | Versie | CDN | Doel |
|---------|--------|-----|------|
| xlsx.full.min.js | 0.18.5 | Cloudflare | Excel export |
| heic2any.min.js | 0.0.4 | jsDelivr | HEIC→JPEG conversie |
| Google GSI client | latest | accounts.google.com | OAuth 2.0 |

---

## Gebruikersflow (happy path)

```
1. Tab 1: Plak OpenAI key → Test → Plak Google Client ID → Inloggen met Google
2. Tab 2: Plak Drive-map URL → Analyseer URL → Foto's laden uit Drive
          (wacht tot "N foto's toegevoegd")
3. Tab 2: Klik "Ga naar verwerken"
4. Tab 3: Kies instellingen → Start verwerking
          (wacht, logbox toont voortgang)
5. Tab 4: Bekijk resultaten → Download Excel
```

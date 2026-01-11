# Herznet EWR Invoice to Paperless-NGX

Automatischer Download von Rechnungen aus dem Herznet-Kundenportal und Upload zu Paperless-ngx.

## Übersicht

Dieser n8n-Workflow:
1. Loggt sich automatisch im Herznet-Kundenportal ein
2. Navigiert zur Dokumenten-Seite
3. Extrahiert alle verfügbaren PDF-Rechnungen
4. Lädt jede Rechnung herunter
5. Uploaded sie zu Paperless-ngx mit Tags, Korrespondent und Dokumenttyp

## Workflow-Ablauf

```
┌─────────────────────┐
│ Monatlich (10. Tag) │ ← Trigger
└──────────┬──────────┘
           ▼
┌─────────────────────┐
│ ⚙️ KONFIGURATION    │ ← Alle Einstellungen hier
└──────────┬──────────┘
           ▼
┌─────────────────────┐
│ 1. Login-Seite      │ ← Holt Cookies & CSRF-Token
└──────────┬──────────┘
           ▼
┌─────────────────────┐
│ 2. CSRF extrahieren │
└──────────┬──────────┘
           ▼
┌─────────────────────┐
│ 3. Login            │ ← Intrexx-Authentifizierung
└──────────┬──────────┘
           ▼
┌─────────────────────┐
│ 4. Cookies sammeln  │
└──────────┬──────────┘
           ▼
┌─────────────────────┐
│ 5. Dokumente laden  │ ← Dokumenten-Übersicht
└──────────┬──────────┘
           ▼
┌─────────────────────┐
│ 6. PDF-Links finden │ ← Extrahiert Download-URLs
└──────────┬──────────┘
           ▼
┌─────────────────────┐
│ 7. PDF herunterladen│ ← Loop für jedes PDF
└──────────┬──────────┘
           ▼
┌─────────────────────┐
│ 8. Upload Paperless │ ← Mit Tags & Metadaten
└──────────┬──────────┘
           ▼
┌─────────────────────┐
│ 9. Sammeln          │
└──────────┬──────────┘
           ▼
┌─────────────────────┐
│ 10. Zusammenfassung │ ← Bericht
└─────────────────────┘
```

## Konfiguration

Alle Einstellungen befinden sich im Node **"⚙️ KONFIGURATION"**:

### Herznet-Zugangsdaten

| Variable | Beschreibung | Beispiel |
|----------|--------------|----------|
| `HERZNET_USERNAME` | Kundennummer/Benutzername | `12345678-1234567` |
| `HERZNET_PASSWORD` | Passwort | `MeinPasswort123` |
| `baseUrl` | Portal-URL | `https://mein.herznet.de` |

### Paperless-ngx Einstellungen

| Variable | Beschreibung | Beispiel |
|----------|--------------|----------|
| `PAPERLESS_URL` | URL deiner Paperless-Instanz | `https://paperless.beispiel.de` |
| `PAPERLESS_TOKEN` | API-Token (siehe unten) | `ab12cd34ef56...` |
| `PAPERLESS_CORRESPONDENT` | ID des Korrespondenten | `20` |
| `PAPERLESS_DOCTYPE` | ID des Dokumenttyps | `6` |
| `PAPERLESS_TITLE_PREFIX` | Präfix für Dokumenttitel | `Rechnung_Herznet_` |
| `PAPERLESS_TAG_1` | Erste Tag-ID | `16` |
| `PAPERLESS_TAG_2` | Zweite Tag-ID | `8` |

## Paperless-ngx Token erstellen

### Option 1: Über die Web-Oberfläche

1. Öffne Paperless-ngx in deinem Browser
2. Klicke auf **Einstellungen** (Zahnrad-Symbol)
3. Gehe zu **Django Admin** (ganz unten)
4. Navigiere zu **Auth Token** → **Tokens**
5. Klicke auf **Token hinzufügen**
6. Wähle deinen Benutzer aus
7. Klicke **Speichern**
8. Kopiere den angezeigten Token

### Option 2: Über die Kommandozeile (Docker)

```bash
docker exec -it paperless-webserver python manage.py createsuperuser_token <username>
```

Oder in der Paperless-Shell:

```bash
docker exec -it paperless-webserver python manage.py shell

# In der Python-Shell:
from django.contrib.auth.models import User
from rest_framework.authtoken.models import Token

user = User.objects.get(username='dein_username')
token, created = Token.objects.get_or_create(user=user)
print(token.key)
```

## IDs in Paperless finden

### Tags, Korrespondenten, Dokumenttypen

Die IDs findest du über die API oder in der URL:

1. **Über die URL**: Öffne einen Tag/Korrespondent/Dokumenttyp in Paperless. Die ID steht in der URL:
   ```
   https://paperless.beispiel.de/tags/16/   ← ID = 16
   ```

2. **Über die API**:
   ```
   https://paperless.beispiel.de/api/tags/
   https://paperless.beispiel.de/api/correspondents/
   https://paperless.beispiel.de/api/document_types/
   ```

## Zeitplan anpassen

Der Trigger ist auf den **10. Tag jedes Monats** eingestellt.

Um das zu ändern:
1. Öffne den Node **"Monatlich (5. des Monats)"**
2. Ändere `triggerAtDayOfMonth` auf den gewünschten Tag

## Fehlerbehebung

### "Keine PDF-Links gefunden"
- Prüfe, ob die Herznet-Zugangsdaten korrekt sind
- Das Portal könnte seine Struktur geändert haben

### "401 Unauthorized" bei Paperless
- Token ist falsch oder abgelaufen
- Neuen Token erstellen (siehe oben)

### "400 Bad Request - tags incorrect type"
- Tags müssen als separate Parameter mit je einer ID übergeben werden
- Nicht als kommagetrennte Liste

### Dokumente erscheinen nicht in Paperless
- Prüfe den Paperless-Consumer-Log
- Dokument könnte als Duplikat erkannt worden sein

## Sicherheitshinweise

- Speichere Passwörter und Tokens niemals im Klartext in geteilten Workflows
- Nutze n8n-Credentials oder Umgebungsvariablen für sensible Daten
- Der Paperless-Token hat vollen API-Zugriff - behandle ihn wie ein Passwort

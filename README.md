# Tesla Trips

Privates, responsives Web-Dashboard für die Fahrten- und Ladehistorie eines Tesla Model Y. Die Oberfläche kombiniert Fahrten und Ladevorgänge chronologisch und lädt die Daten über eine Node-RED-/InfluxDB-API.

## Funktionen

- Zeitraumfilter für 1, 7, 14, 31, 90 und 365 Tage sowie alle Daten
- Gesamtzahl der Fahrten, gefahrene Kilometer und Durchschnittsverbrauch
- Kilometer seit der letzten Ladung, unabhängig vom gewählten Zeitraum
- Verbrauchskurve pro Tag für die letzten maximal 14 Tage im ausgewählten Zeitraum
- Fahrtdetails mit Dauer, Verbrauch, Geschwindigkeit, Energie und Batterieverlauf
- Gemeinsame chronologische Darstellung von Fahrten und AC-/DC-Ladevorgängen
- Aufklappbare Ladekurven
- Aufklappbare Leaflet-/OpenStreetMap-Karten mit Start- und Zielmarker
- Responsive Darstellung für Desktop und Mobilgeräte

## Dateien

| Datei | Beschreibung |
| --- | --- |
| `trips.html` | Seitenstruktur, Datenabruf, Berechnungen, Diagramme und Kartenlogik |
| `trips.css` | Grundlayout und Gestaltung des Dashboards |

Die Seite benötigt keinen Build-Schritt und kann direkt von einem Webserver ausgeliefert werden.

## Benötigte API-Endpunkte

Alle Endpunkte werden relativ zur aktuellen Domain aufgerufen.

### Fahrten

```http
GET /api/tesla/trips?days=31
```

Akzeptierte Antworten:

```json
[
  {
    "start_time": "2026-09-04T08:00:00+02:00",
    "end_time": "2026-09-04T08:30:00+02:00",
    "duration_seconds": 1800,
    "distance_km": 24.5,
    "consumed_kwh": 4.2,
    "avg_kwh_100km": 17.1,
    "avg_speed_kmh": 49,
    "start_battery_pct": 80,
    "end_battery_pct": 72,
    "start_lat": 48.2082,
    "start_lon": 16.3738,
    "end_lat": 48.2500,
    "end_lon": 16.4100
  }
]
```

Alternativ wird ein Objekt mit einem `trips`-Array akzeptiert:

```json
{ "trips": [] }
```

Gültige Werte für `days` sind `1`, `7`, `14`, `31`, `90`, `365` und `all`.

### Ladevorgänge

```http
GET /api/tesla/charging?days=31
```

Die Antwort kann direkt ein Array oder ein Objekt mit `charging_sessions` sein.

### Ladekurve

```http
GET /api/tesla/charging/curve?session_id=<ID>
```

Erwartet wird ein Objekt mit einem `points`-Array. Die Kurve wird erst beim Aufklappen des Ladevorgangs geladen.

## Kilometer seit der letzten Ladung

Zusätzlich zum ausgewählten Zeitraum lädt die Seite Fahrten und Ladevorgänge einmal mit `days=all`. Aus allen Fahrten nach dem Ende des jüngsten Ladevorgangs wird die gefahrene Distanz berechnet.

Falls die vollständige Abfrage fehlschlägt, verwendet die Seite als Fallback die Daten des aktuell ausgewählten Zeitraums.

## Karten

Die Kartenansicht verwendet Leaflet 1.9.4 und Kartenkacheln von OpenStreetMap. Eine Karte wird erst geladen, wenn bei einer Fahrt auf **Karte anzeigen** geklickt wird.

Unterstützte Koordinatenfelder:

- `start_lat` und `start_lon`
- `end_lat` und `end_lon`
- alternativ ausgeschriebene Felder wie `start_latitude` und `start_longitude`

Beim Öffnen einer Karte werden Anfragen für den betreffenden Kartenausschnitt an `tile.openstreetmap.org` gesendet. Leaflet wird über `unpkg.com` geladen.

## Bereitstellung mit Apache

Die Dateien können direkt im DocumentRoot liegen oder einzeln aus dem Git-Repository verlinkt werden:

```bash
ln -s /pfad/zum/teslatrips/trips.html /var/www/html/trips.html
ln -s /pfad/zum/teslatrips/trips.css /var/www/html/trips.css
```

Apache benötigt Leserechte auf den Dateien und Ausführungsrechte auf allen übergeordneten Verzeichnissen. Symbolische Links müssen durch die Apache-Konfiguration erlaubt sein, beispielsweise mit `Options FollowSymLinks`.

Die API-Endpunkte sollten unter derselben Domain erreichbar sein, da die Seite relative URLs verwendet.

## Datenfluss

Der derzeitige Backend-Datenfluss basiert auf Tesla-Telemetrie, Node-RED und InfluxDB:

```text
Tesla-Telemetrie
    → Node-RED Trip-Erkennung
    → Reverse-Geocoding von Start und Ziel
    → InfluxDB
    → HTTP-API
    → trips.html
```

Die Fahrtzusammenfassungen werden im Measurement `tesla_trip_v3` gespeichert.

## Routenaufzeichnung

Die Node-RED-Trip-Funktion wurde für eine Aufzeichnung der Position im Abstand von 30 Sekunden vorbereitet. Die Punkte werden für jede Fahrt über `trip_id` gruppiert und sollen im separaten Measurement `tesla_trip_route_v1` gespeichert werden.

Aktueller Stand:

- Sammeln der Punkte während einer Fahrt ist vorbereitet
- Schreiben nach `tesla_trip_route_v1` ist vorbereitet
- Praxistest bei einer abgeschlossenen Fahrt steht noch aus
- API-Endpunkt zum Lesen einer Route fehlt noch
- Die Webseite zeigt momentan Start und Ziel, aber noch nicht die tatsächlich gefahrene Linie

Der geplante Endpunkt lautet beispielsweise:

```http
GET /api/tesla/trips/route?trip_id=<TRIP_ID>
```

Sobald dieser Endpunkt vorhanden ist, kann `trips.html` die Routenpunkte beim Öffnen der Karte laden und als Leaflet-Polyline darstellen.

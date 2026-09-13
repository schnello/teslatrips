# Tesla Trips

Privates, responsives Web-Dashboard für die Fahrten- und Ladehistorie eines Tesla Model Y. Die Oberfläche kombiniert Fahrten und Ladevorgänge chronologisch und lädt die Daten über eine Node-RED-/InfluxDB-API.

## Funktionen

- Streckenfilter für die letzten 50, 300 oder 600 km sowie eine unbegrenzte Ansicht (Standard: 300 km)
- Gesamtzahl der Fahrten, gefahrene Kilometer und Durchschnittsverbrauch
- Kilometer seit der letzten Ladung, unabhängig von der gewählten Strecke
- Verbrauchskurve pro Tag für die letzten maximal 14 Tage innerhalb der gewählten Strecke
- Fahrtdetails mit Dauer, Verbrauch, Geschwindigkeit, Energie und Batterieverlauf
- Gemeinsame chronologische Darstellung von Fahrten und AC-/DC-Ladevorgängen
- Aufklappbare Ladekurven
- Aufklappbare Leaflet-/OpenStreetMap-Karten mit Start-, Zielmarker und tatsächlich aufgezeichneter Route
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

### Fahrtroute

```http
GET /api/tesla/trips/route?trip_id=<TRIP_ID>
```

Erwartet wird ein Objekt mit `trip_id`, `count` und einem nach `point_index` geordneten `points`-Array. Die Route wird erst beim Aufklappen der betreffenden Fahrt geladen.

## Kilometer seit der letzten Ladung

Die Seite lädt Fahrten und Ladevorgänge einmal mit `days=all` und begrenzt die Anzeige anschließend lokal auf die ausgewählte Fahrstrecke. Aus allen Fahrten nach dem Ende des jüngsten Ladevorgangs wird die gefahrene Distanz berechnet.

Falls die vollständige Abfrage fehlschlägt, zeigt die Seite eine Fehlermeldung an.

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

Die Node-RED-Trip-Funktion zeichnet die Position während einer Fahrt im Abstand von 30 Sekunden auf. Die Punkte werden über `trip_id` gruppiert und im separaten Measurement `tesla_trip_route_v1` gespeichert.

Aktueller Stand:

- Sammeln der Punkte während einer Fahrt funktioniert
- Schreiben nach `tesla_trip_route_v1` funktioniert
- Der Route-Endpunkt liefert die Punkte einer Fahrt anhand ihrer `trip_id`
- Die Webseite lädt die Route beim Öffnen der Karte
- Die tatsächlich aufgezeichnete Strecke wird als blaue Leaflet-Linie dargestellt

Der Endpunkt lautet:

```http
GET /api/tesla/trips/route?trip_id=<TRIP_ID>
```

Fahrten ohne gespeicherte Routenpunkte zeigen weiterhin nur Start- und Zielmarker.

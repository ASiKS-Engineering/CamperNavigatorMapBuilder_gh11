# CamperNavigator Map Builder

Der **CamperNavigator Map Builder** erstellt vollständige Offline-Navigationspakete für Wohnmobile und Camper.

Der Build-Prozess erzeugt:

* Offline-Karten im MBTiles-Format
* einen GraphHopper Routing-Graph
* ein fahrzeugspezifisches Routing-Profil
* ein portables Offline-Paket mit Manifest-Datei

Das erzeugte Paket kann anschließend von einer Offline-Navigationsanwendung verwendet werden.

---

## Funktionen

* Zusammenführen mehrerer OpenStreetMap PBF-Dateien
* Automatische Ermittlung der Bounding Box
* Erzeugung von Offline-Karten mit Planetiler
* Erstellung eines GraphHopper Routing-Graphs
* Fahrzeugbeschränkungen für Camper:

  * Höhe
  * Breite
  * Länge
  * Gewicht
  * Geschwindigkeit
* Custom Routing Model für:

  * Höhenbeschränkungen
  * Breitenbeschränkungen
  * Längenbeschränkungen
  * Gewichtsbeschränkungen
  * private Straßen
  * schlechte Oberflächen
* Erzeugung eines versionierten Offline-Pakets

---

# Voraussetzungen

## Software

Benötigt werden:

* Windows
* Java Runtime Environment
* Osmium Tool
* Planetiler
* GraphHopper
* 7-Zip

Empfohlene Struktur:

```
CamperNavigatorBuilder/

├── tools/
│   ├── osmium.exe
│   ├── planetiler.jar
│   ├── graphhopper-web-9.1.jar
│   └── 7z.exe
│
├── input/
│   └── *.osm.pbf
│
├── build/
│
├── output/
│
├── profiles/
│   ├── vehicle.json
│   └── custom_model.json
│
├── graphhopper-config.yml
│
└── build_maps.bat
```

---

# Build-Prozess

Der komplette Ablauf:

```
OSM PBF Dateien
        |
        v
    Osmium Merge
        |
        v
 merged.osm.pbf
        |
        +----------------+
        |                |
        v                v
   Planetiler       GraphHopper
        |                |
        v                v
  map.mbtiles     graphhopper/
        |
        v
 Offline Package
```

---

# Build starten

## Standard

```
build_maps.bat
```

Erzeugt ein Standardpaket:

```
eu-de-camper
```

---

## Parameter

Der Builder unterstützt Übergabewerte:

```
build_maps.bat continent region profile version
```

Beispiel:

```
build_maps.bat eu de camper 2026.07
```

Erzeugt:

```json
{
  "id": "eu-de-camper",
  "version": "2026.07",
  "profile": "camper"
}
```

---

## Parameterübersicht

| Parameter | Beispiel | Beschreibung  |
| --------- | -------- | ------------- |
| continent | eu       | Kontinent     |
| region    | de       | Region/Land   |
| profile   | camper   | Routingprofil |
| version   | 2026.07  | Paketversion  |

Beispiele:

```
build_maps.bat eu at camper 2026.07
```

ergibt:

```
eu-at-camper
```

---

```
build_maps.bat eu de truck 2026.08
```

ergibt:

```
eu-de-truck
```

---

# Fahrzeugprofil

Das Fahrzeugprofil liegt unter:

```
profiles/vehicle.json
```

Beispiel:

```json
{
  "vehicle": {
    "height": 3.20,
    "width": 2.35,
    "length": 7.40,
    "weight": 4.25,
    "trailer": {
      "enabled": false
    }
  },
  "routing": {
    "maxSpeed": 90
  }
}
```

Diese Werte werden automatisch in das GraphHopper Custom Model übernommen.

---

# Routing-Regeln

Der Builder erzeugt automatisch ein Custom Model.

Aktuelle Regeln:

## Fahrzeugbeschränkungen

Straßen werden ausgeschlossen, wenn:

* maximale Höhe kleiner als Fahrzeughöhe
* maximale Breite kleiner als Fahrzeugbreite
* maximale Länge kleiner als Fahrzeuglänge
* maximales Gewicht kleiner als Fahrzeuggewicht

---

## Straßenqualität

Beispiel:

```
GRAVEL
```

wird mit niedriger Priorität bewertet.

---

## Private Straßen

Private Straßen werden ausgeschlossen.

---

# Ergebnis

Nach erfolgreichem Build:

```
output/

├── graphhopper/
│
├── map.mbtiles
│
└── package/
    └── offline_navigation.zip
```

---

# Offline Paket

Das erzeugte ZIP besitzt folgende Struktur:

```
offline_navigation/

├── manifest.json
│
├── graph/
│   └── GraphHopper Daten
│
├── maps/
│   └── map.mbtiles
│
└── config/
    ├── bbox.json
    └── custom_model.json
```

---

# Routing-Modell

`config/custom_model.json` (Quelle: `config/custom_model.json` im Builder) ist die **einzige Definition der Routing-Gewichtung**.

- GraphHopper berechnet damit beim Import die Landmarks (`graph/landmarks_camper`).
- Die App liest dasselbe Modell aus dem Paket und wertet es selbst aus (GraphHoppers Janino-Compiler läuft unter Android nicht). Es gilt der Teil von GraphHoppers Custom-Model-Syntax mit `if` / `else_if` / `else`, `multiply_by`, `limit_to`, `distance_influence`, Vergleichen und `&&`, `||`, `!` auf Encoded Values. Nicht unterstützte Syntax lässt den Import-/Ladevorgang mit einer Fehlermeldung scheitern.
- Fahrzeugdaten (Maße, Gewicht, Höchstgeschwindigkeit) stehen **nicht** im Kartenpaket, sondern im Fahrzeugpaket (`.vpk`, `models/custom_model.json`). Die App ergänzt sie zur Laufzeit.
- Damit die Landmarks gültig bleiben, dürfen Fahrzeugmodell und Routing-Modus Kanten nur **teurer** machen: `multiply_by` ≤ 1, niedrigeres `limit_to`, `distance_influence` ≥ Wert im Kartenpaket. Das Basis-Modell darf deshalb keine Fahrzeugbeschränkungen enthalten und gibt Geschwindigkeit nach oben frei (130 km/h).
- Wird das Modell geändert, muss das Kartenpaket neu gebaut werden (Landmarks).

---

# Manifest

Jedes Paket enthält eine `manifest.json`.

Beispiel:

```json
{
  "formatVersion": 2,

  "builder": {
    "name": "CamperNavigator Map Builder",
    "version": "1.0.0",
    "created": "2026-07-30T08:37:15Z"
  },

  "package": {
    "id": "eu-de-camper",
    "name": "Germany",
    "version": "2026.07",
    "profile": "camper"
  },

  "routing": {
    "profile": "camper",
    "customModel": "config/custom_model.json",
    "customModelSha256": "<sha256 der Datei>",
    "landmarks": true
  },

  "content": {
    "graph": {
      "path": "graph"
    },
    "map": {
      "path": "maps/map.mbtiles",
      "format": "mbtiles"
    },
    "config": {
      "bbox": "config/bbox.json",
      "customModel": "config/custom_model.json"
    }
  }
}
```

---

# Versionierung

Das Paket verwendet mehrere Ebenen:

## Paket-ID

Beispiel:

```
eu-de-camper
```

Aufbau:

```
continent-region-profile
```

---

## Paket-Version

Beispiel:

```
2026.07
```

Beschreibt den Kartenstand.

---

## Build-Nummer

Ein Build kann zusätzlich eine eindeutige Nummer besitzen:

```
202607301106
```

Damit können mehrere Builds derselben Version unterschieden werden.

---

# Datenquellen

## OpenStreetMap

Die Kartendaten basieren auf OpenStreetMap-Daten.

Beachte die Lizenzbedingungen:

https://www.openstreetmap.org/copyright

---

# Erweiterungsmöglichkeiten

Die Paketstruktur ist für zukünftige Erweiterungen vorbereitet:

```
offline_navigation/

├── graph/
├── maps/
├── config/
├── elevation/
├── poi/
├── voices/
└── metadata/
```

Mögliche Erweiterungen:

* Höhenmodelle
* POI-Datenbanken
* Sprachpakete
* zusätzliche Routingprofile
* mehrere Fahrzeugtypen

---

# Lizenz

Dieses Projekt verwendet externe Komponenten mit eigenen Lizenzen:

* OpenStreetMap Daten
* Osmium
* Planetiler
* GraphHopper
* weitere verwendete Bibliotheken

Bitte die jeweiligen Lizenzbedingungen beachten.

---

# Projektziel

CamperNavigator soll eine vollständig offlinefähige Navigation ermöglichen, die speziell auf die Anforderungen von Wohnmobilen und Campern zugeschnitten ist.

Der Fokus liegt auf:

* zuverlässiger Offline-Navigation
* fahrzeugspezifischem Routing
* reproduzierbaren Kartenpaketen
* einfacher Aktualisierung
* modularer Erweiterbarkeit

# Datenvisualisierung für die Historische Forschung

Exploration verschiedener Strategien zur visuellen Analyse und Präsentation komplexer historischer Daten, u. a. Erstellung dynamischer Darstellungen wie Zeitstrahlen und Karten. Zudem wird die Möglichkeit der räumlichen Visualisierung thematischer Klassifikationen diskutiert (Pyplot, Wikibase Query Service).

## Übungen

Ziel: Erste rasche Visualisierungen (Matplotlib / Pyplot) aus den aufbereiteten & angereicherten Daten (Tasks 1 & 2) sowie SPARQL-Abfragen gegen Wikidata zur Kontextualisierung. Nutzung von Property P10626 ("deckenmalerei.eu ID") zur Verknüpfung mit den lokalen UUIDs.

Voraussetzungen:

- Dateien aus Task 1/2 (z. B. `buildings_core`, `building_dates`, `persons_enriched`, `building_person_roles_enriched`).
- Spalte mit lokaler UUID der Personen/Gebäude, die auf Wikidata via `P10626` referenziert (UUID = Wert der Property).

### 1. Daten-Lade & Basis-Struktur (Notebook oder Skript)

1. Einlesen relevanter Parquet/CSV Dateien (Strings beibehalten, Datumsspalten nicht automatisch parsen).
2. Merge Personenrollen + Personen (→ DataFrame `roles_persons`).
3. Erzeuge Hilfstabelle `persons_timeline` mit `birth_year`, `death_year`, `life_span_edtf`.

### 2. Simple Pyplot Visualisierungen

1. Histogramm der Bau-/Datierungsjahre (Verwendung `building_dates` → Mittelwert pro Gebäude aus Range).
2. Balkendiagramm: Top 10 Berufsbezeichnungen (aus `occupations`).
3. Horizontaler Zeitstrahl ausgewählter Personen (z. B. Architekten) mit Lebensspanne (Linien von `birth_year` zu `death_year`).
4. Scatterplot: `chronology_min` vs. `chronology_max` (Dauer / Spannweite der Datierungen) – Farbcode nach Rolle (z. B. Architekt vs. Maler, falls klassifizierbar).
5. Karten-Skizze (optional Quick & Dirty): Streudiagramm lat/lng (Projection ignoriert) mit Beschriftung der `appellation` (nur 20 Punkte sample).

Pyplot Snippet (Beispiel Histogramm):

```python
import matplotlib.pyplot as plt
import pandas as pd
bd = building_dates.dropna(subset=['year_start','year_end']).copy()
bd['year_mean'] = (bd.year_start.astype(int) + bd.year_end.astype(int)) / 2
plt.figure(figsize=(8,4))
plt.hist(bd['year_mean'], bins=30, color='#446699')
plt.title('Verteilung Datierungsjahre (Mittel aus Ranges)')
plt.xlabel('Jahr')
plt.ylabel('Anzahl Segmente')
plt.tight_layout()
plt.show()
```

### 3. Nutzung von Property P10626 (deckenmalerei.eu ID)

Property-Link: <https://www.wikidata.org/wiki/Property:P10626>

Aufgaben:

1. Prüfe für jede vorhandene `wikidata_id`, ob die lokale UUID als P10626 Wert vorhanden ist (SPARQL Query → Validierung).
2. Liste Fälle, wo `wikidata_id` existiert aber kein P10626 gesetzt ist → Kandidaten für mögliche Datenanreicherung zurück in Wikidata.
3. Erzeuge Tabelle `wd_link_status` mit Spalten: `wikidata_id`, `local_uuid`, `has_p10626` (bool).
4. Visualisiere Anteil gemappter Personen (Tortendiagramm oder Balkendiagramm).

### 4. SPARQL Queries (Wikidata Query Service)

1. Alle Personen mit gesetzter P10626 und deren Geburts-/Sterbedaten (Filter auf IDs aus lokalem Set):

```sparql
PREFIX wdt: <http://www.wikidata.org/prop/direct/>
PREFIX wd: <http://www.wikidata.org/entity/>
PREFIX p: <http://www.wikidata.org/prop/>
PREFIX ps: <http://www.wikidata.org/prop/statement/>
PREFIX pq: <http://www.wikidata.org/prop/qualifier/>
SELECT ?person ?personLabel ?uuid ?birth ?death WHERE {
	VALUES ?uuid { "<UUID_1>" "<UUID_2>" }  # Ersetze durch lokale UUID Werte
	?person wdt:P10626 ?uuid .
	OPTIONAL { ?person wdt:P569 ?birth }
	OPTIONAL { ?person wdt:P570 ?death }
	SERVICE wikibase:label { bd:serviceParam wikibase:language "de,en". }
}
```
2. Zähle wie viele deiner Personen (lokales Set) bereits in Wikidata sind:

```sparql
SELECT (COUNT(DISTINCT ?person) AS ?count) WHERE {
	VALUES ?uuid { "<UUID_1>" "<UUID_2>" }
	?person wdt:P10626 ?uuid .
}
```
3. Archivierungspotential: Personen mit P10626, die KEIN Sterbedatum haben (Lücken identifizieren):

```sparql
SELECT ?person ?personLabel ?uuid ?birth WHERE {
	VALUES ?uuid { "<UUID_1>" "<UUID_2>" }
	?person wdt:P10626 ?uuid .
	OPTIONAL { ?person wdt:P569 ?birth }
	FILTER NOT EXISTS { ?person wdt:P570 ?death }
	SERVICE wikibase:label { bd:serviceParam wikibase:language "de,en". }
}
```
4. Kontext: Andere Werke/Statements verknüpfter Personen (z. B. Beruf P106) – liefert zusätzliche Normalisierungslabels:

```sparql
SELECT ?person ?personLabel ?occupationLabel WHERE {
	VALUES ?uuid { "<UUID_1>" "<UUID_2>" }
	?person wdt:P10626 ?uuid ;
					wdt:P106 ?occupation .
	SERVICE wikibase:label { bd:serviceParam wikibase:language "de,en". }
}
```

### 5. Abgleich SPARQL vs. Lokal

1. Exportiere SPARQL Ergebnisse (CSV) → lade in Notebook.
2. Vergleiche `birth_year`/`death_year` lokal vs. `P569`/`P570` remote.
3. Markiere Differenzen (z. B. Jahr vs. genaueres Datum) → Reporting DataFrame.
4. Visualisiere Anzahl Abweichungen pro Feld.

### 6. Erweiterte Visualisierungsideen (optional)

- Timeline mit Overplot-Vermeidung (Alpha/Kernel Density) zur Darstellung der Lebensjahre.
- Sankey/Chord (externes Tool) für Person-Rolle-Beziehungen.
- Clustering Personen nach Berufsprofil (später mit Embeddings, jetzt nur Skizze).

### 7. Mini-Reporting

Erstelle Markdown-Abschnitt mit:

- Anzahl visualisierter Gebäude.
- Abdeckung Prozentsatz Wikidata-Verknüpfungen.
- Häufigste fehlende Datenfelder.
- Liste (Top 5) vorgeschlagene Ergänzungen für Wikidata (fehlendes Sterbedatum, fehlende P10626 etc.).

---
© Workshop "Historical Data Hacking" – Task 3 Visualisierung.

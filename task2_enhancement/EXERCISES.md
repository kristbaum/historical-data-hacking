# Task 2 – Datenanreicherung & Semantische Verknüpfung (Wikidata / GND)

Ziel: Personen- und Objekt-Daten aus Task 1 (insb. `persons` und `building_person_roles`) mit externen Normdaten abgleichen (Wikidata, GND) und strukturierte Zusatzinformationen (Lebensdaten, alternative Namen, Berufe) übernehmen.

> Voraussetzungen: Aufbereitete Long-Format-Tabellen aus Task 1 (oder OpenRefine Exporte). Optional: bereits bereinigte Personenlabels.

## 0. Vorbereitung

1. Exportiere aus Task 1 eine `persons.csv` mit Spalten: `person_id` (lokal/UUID oder leer), `person_label`.
2. Erstelle in OpenRefine ein neues Projekt `persons_reconcile`.
3. Entferne offensichtliche Klammerzusätze (z. B. Jahresangaben, wenn bereits vorhanden) mit GREL-RegEx – nur falls sie Matching behindern.

Optional Python-Vorprofilierung (Distinct Labels, Häufigkeit) zur Priorisierung der wichtigsten Einträge.

## 1. Wikidata Reconciliation (OpenRefine)

1. Wähle Menü: Reconcile → Add Standard Service → URL: `https://wikidata.reconci.link/en/api` (oder passender Sprach-Endpunkt).
2. Reconcile Spalte `person_label` gegen Wikidata (Type Suggestion "human" / Q5).
3. Aktiviere "Auto-match candidates with high confidence".
4. Identifiziere Einträge mit mehreren Kandidaten → manuell wählen.
5. Markiere unsichere Fälle (Tag in einer neuen Spalte `wd_status`: `matched`, `ambiguous`, `none`).
6. Füge neue Spalte `wikidata_id` hinzu: `cell.recon.match.id` (OpenRefine Funktion: "Add column based on this column" → Sprache GREL: `cell.recon.match.id`).
7. Füge Spalte `wikidata_qlabel` hinzu: `cell.recon.match.name`.

Qualitäts-Checks:

- Prozentuale Match-Rate.
- Anzahl Ambiguitäten.
- Top 10 Labels ohne Treffer.

## 2. Wikidata Enrichment (Lebensdaten)

1. Wähle Reconcile-Spalte → "Add columns from reconciled values".
2. Wähle Properties:
   - P569 (Geburtsdatum)
   - P570 (Sterbedatum)
   - P19 (Geburtsort; Label)
   - P20 (Sterbeort; Label)
   - P106 (Beruf; Label)
3. Spalten konsolidieren: ISO-Datum (YYYY-MM-DD) extrahieren → neue Felder `birth_year`, `death_year` (Regex: `^(\\d{4})`).
4. Ableite `life_span`: `birth_year + '-' + death_year` (oder Platzhalter `?`).
5. Prüfe auf inkonsistente Reihenfolge (Sterbejahr < Geburtsjahr) → Flag-Spalte.

Python (optional) für Post-Processing:

```python
import pandas as pd
persons = pd.read_csv('persons_reconciled_wikidata.csv', dtype=str)
for col in ['P569','P570']:
   persons[col] = persons[col].str.extract(r'^(\d{4})', expand=False)
persons['life_span'] = persons[['P569','P570']].apply(lambda r: f"{r.P569 or '?'}-{r.P570 or '?'}", axis=1)
```

## 3. GND Reconciliation (Entity-Fishing / DNB SRU / VIAF)

Ziel: Ergänzung GND-IDs (P227) für bereits gematchte Wikidata-Einträge oder direkte GND-Suche.

Varianten:

- a) Wikidata Already Contains GND: Hole P227 (GND-ID) über zusätzlichen Property-Import.
- b) Direkter GND-Abgleich (OpenRefine Custom Service) – falls Service verfügbar.

Schritt a (empfohlen, schnell):

1. In OpenRefine erneut "Add columns from reconciled values" → wähle P227.
2. Prüfe Format (Normform: Ziffern + 'X' möglich). Normalisiere: entferne Leerzeichen.

Schritt b (optional / falls Zeit):

1. Erstelle Liste verbleibender Personen ohne Wikidata Match.
2. Baue Skript zur GND SRU Suche (Z39.50 / SRU API) – optional, nicht Teil Grundworkflow.

## 4. Zusammenführung & Normalisierung

1. Erstelle finalen Personen-DataFrame mit Feldern:
   - `person_local_id`
   - `person_label`
   - `wikidata_id`
   - `gnd_id`
   - `birth_year`, `death_year`, `life_span`
   - `birth_place_label`, `death_place_label`
   - `occupations` (kommagetrennte Labels)
2. Explodiere Mehrfachberufe (falls nötig) in separate Tabelle `person_occupations`.
3. Vereinheitliche Orte (Mapping auf kontrollierte Gazetteer-IDs optional für Task 3).

## 5. Ableitung Historischer Zeit-Features (EDTF / Unsicherheiten)

1. Konvertiere `life_span` in EDTF:
   - Nur Geburtsjahr bekannt: `YYYY/..`
   - Nur Sterbejahr bekannt: `../YYYY`
   - Beide vorhanden: `YYYY/YYYY`
2. Spalte `life_span_edtf` hinzufügen.
3. Optional: Unbestimmte Jahre (`ca.`) aus Labels erkennen → Intervall weiten (z. B. ±5 Jahre) und als `~` Qualifier in EDTF (Approximation) notieren: `1750~`.

## 6. Daten-Qualitätsmetriken

Erzeuge Report (Markdown / DataFrame):

- Match-Rate Wikidata (% matched / total).
- Anteil mit vollständigen Lebensdaten (Geburts- & Sterbejahr).
- Anzahl distinct Berufe.
- Häufigste 10 Berufe.
- Anzahl Personen nur mit Geburts- oder nur mit Sterbejahr.
- GND-Abdeckung (% mit P227).

## 7. Aktualisierung Rollen-Tabelle

1. Mergen `building_person_roles` mit angereicherter Personenliste auf `person_id` (oder Label fallback wenn ID fehlt – vorher disambiguieren!).
2. Neue Felder ergänzen (`wikidata_id`, `life_span`, `occupations`).
3. Export als `building_person_roles_enriched.parquet` / `.csv`.
4. Optional: Filter Personen ohne Identifikatoren → Liste für spätere manuelle Recherche.

## 8. SPARQL Validierung (Wikidata Query Service)

Führe eine Beispiel-Abfrage aus (manuell im WDQS), um z. B. alle gefundenen `wikidata_id` zu verifizieren:

```sparql
SELECT ?person ?personLabel ?birth ?death WHERE {
   VALUES ?person { wd:Q42 wd:Q5582 }  # Beispiel: Ersetze mit echten IDs
   OPTIONAL { ?person wdt:P569 ?birth }
   OPTIONAL { ?person wdt:P570 ?death }
   SERVICE wikibase:label { bd:serviceParam wikibase:language "de,en". }
}
LIMIT 50
```

Dokumentiere Unterschiede zwischen SPARQL-Rückgaben und importierten OpenRefine-Attributen.

## 9. Persistenz & Versionierung

1. Lege Unterordner `enhanced/` an → speichere:
   - `persons_enriched.parquet`
   - `person_occupations.parquet`
   - `building_person_roles_enriched.parquet`
2. JSON Export der OpenRefine-Schritte: `openrefine_persons_history.json`.
3. Kurzes `README_enriched.md` mit Feldbeschreibung.

## 10. Reflexion / Offene Fragen

- Welche Matching-Fälle blieben ungeklärt? (Beispiele aufführen)
- Welche False Positives sind aufgetreten und wie erkennst du sie systematisch?
- Welche Felder wären noch sinnvoll (z. B. VIAF, LOC)?
- Wo entstehen potentielle Lizenz-/Nachnutzungsfragen?

---
© Workshop "Historical Data Hacking" – Task 2 Exercises.

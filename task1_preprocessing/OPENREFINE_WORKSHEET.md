# Task 1 – Kurz-Arbeitsblatt OpenRefine

Ziel: Schnelle, reproduzierbare Bereinigung & Normalisierung zentraler Felder aus `buildings.csv` mittels OpenRefine.

> Fokus: Sicht auf typische OpenRefine-Operationen (Facets, Clustering, Transformations, Splits, Rekonsilierungsvorbereitung). Dauer: ca. 30–40 Minuten.

## 1. Import

- Lade `buildings.csv` in OpenRefine.
- Alle Spalten zunächst als Text lassen (kein Auto-Parsing von Zahlen/Datumsangaben aktivieren).
- Projektname: `buildings_raw`.

## 2. Grundlegende Sichtbarkeit

- Entferne (nur in der Ansicht, nicht dauerhaft) extrem leere Spalten via Facet → „Facet by blank“ und spätere Auswahl für Export.
- Erzeuge eine Text-Facet auf `appellation` → erkenne Varianten / Dubletten.

## 3. Bereinigung `verbaleDating`

Ziel: Segmentierung & Vor-Normalisierung für spätere Python-Verarbeitung.

Schritte:

1. Expression (GREL) Trim: `value.trim()` (Spalte bearbeiten → Zellen transformieren).
2. Ersetze mehrere Leerzeichen: `value.replace(/\s+/,' ')`.
3. Split bei Komma (Spaltenmenü → „Edit cells → Split multi-valued cells“ → Separator `,`).
4. Neue Spalte aus dieser (JSON-Serialisierung einzelner Werte für Kontrolle): `value` (Beibehalten). Optional: Werte mit Regex-Facet `^[0-9]{3,4}(-[0-9]{3,4})?$` filtern (parsbar vs. unparsed).
5. Erzeuge Booleanspalte `is_range`: GREL: `value.match(/^[0-9]{3,4}-[0-9]{3,4}$/) != null`.
6. Erzeuge Spalten `year_start` und `year_end`:
   - Falls Range: `value.match(/^(\d{3,4})-(\d{3,4})$/)[0]` & `[1]`
   - Falls Einzeljahr: `value` in beide kopieren.
7. Filtere Ausreißer: Facet `year_start` → Numeric Facet → Werte außerhalb 800–(aktuelles Jahr) markieren.

Hinweis: Exportiere Zwischenergebnis als `building_dates_openrefine.csv` für Abgleich mit Pandas-Parsing.

## 4. Personen-/Rollenfelder (Beispiel: ARCHITECTS)

Ziel: Long-Format Grundlage.

Schritte:
 
1. Wähle Spalten `ARCHITECTS_1` … `ARCHITECTS_10`.
2. „Edit columns → Join columns“ (Separator `||`), erzeuge Sammelspalte `ARCHITECTS_JOIN`.
3. Split multi-valued cells an `||` → leere entfernen.
4. Entferne Duplikate (Facet by text, Auswahl blank → ausschließen).
5. Extrahiere UUID & Label:
   - Neue Spalte `person_id`: GREL: `if(value.contains('|'), value.split('|')[0], null)`
   - Neue Spalte `person_label_raw`: GREL: `if(value.contains('|'), value.split('|')[1], value)`
6. Clean Label: Neue Spalte `person_label`: `person_label_raw.trim().replace(/\s*,\s*/, ', ')`.
7. Cluster (Edit cells → Cluster & edit) auf `person_label` (Method: key collision + metaphone3). Prüfe Zusammenführungen.
8. Export als `architects_roles.csv` (Spalten: `building_row_index` / `person_id` / `person_label`). Optional: füge Quellspalte `ARCHITECTS` hinzu.

## 5. Vorbereitung für Wikidata-Reconciliation

Ziel: Spalte (z. B. `appellation` oder aufbereitete Personennamen) für spätere Abgleichung.

Schritte:

1. Entferne offensichtliche Zusätze (Regex Replace): Beispiel: `value.replace(/,\s*Deutschland$/,'')` (nur wenn sinnvoll!).
2. Normalisiere Großschreibung: `value.toTitlecase()` (sparsam einsetzen, um historische Schreibweisen nicht zu verfälschen).
3. Exportiere Liste eindeutiger Werte (`Facet → Export → tabular`) als `appellations_unique.csv`.

## 6. Audit & Undo/Redo

- Nutze das Undo/Redo Panel, dokumentiere die angewandten Schritte (Screenshot / JSON). Export: „Extract…“ → Speichere Transformations-JSON als `openrefine-history.json` für Reproduzierbarkeit.

## 7. Kurzer Qualitätsbericht (Markdown außerhalb OpenRefine)

- Anzahl ursprünglicher vs. bereinigter `verbaleDating` Tokens.
- Anzahl zusammengeführter Personenlabels durch Clustering.
- Wichtige offene Problemfälle (Stichpunkte).
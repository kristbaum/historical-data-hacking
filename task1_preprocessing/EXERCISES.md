# Task 1 – Daten-Preprocessing: Übungen (Pandas)

Ziel: Rohdaten aus `data/` (insb. `buildings.csv`) explorieren, bereinigen, normalisieren und für weitere Schritte (Anreicherung / Visualisierung) vorbereiten.

> Tipp: Lege für Experimente ein Notebook `notebooks/task1_preprocessing.ipynb` oder ein Skript `scripts/preprocessing_task1.py` an.

## 0. Setup

```python
import pandas as pd
from pathlib import Path
DATA = Path('../data')  # ggf. an Pfad anpassen
buildings = pd.read_csv(DATA / 'buildings.csv', dtype=str)  # alles zunächst als String laden
print(buildings.shape)
buildings.head()
```

Begründung: Viele Spalten enthalten gemischte / freie Werte → zunächst Strings vermeiden fehlerhafte automatische Typkonvertierungen.

## 1. Erste Profilierung

1. Zähle Null/Leer-Werte pro Spalte (`''`, `' '`, `NaN`).
2. Ermittle Anteil gefüllter Werte für Kernfelder: `ID`, `appellation`, `verbaleDating`, `locationLat`, `locationLng`.
3. Erzeuge eine Kurztabelle (DataFrame) mit (Spaltenname, Non-Null %, Anzahl unterschiedlicher Werte, Beispielwerte).

Snippets:

```python
import numpy as np
raw = buildings.copy()
null_like = raw.replace({'': pd.NA, ' ': pd.NA})
profile = []
for col in null_like.columns:
    s = null_like[col]
    profile.append({
        'column': col,
        'non_null_pct': s.notna().mean()*100,
        'n_unique': s.nunique(dropna=True),
        'sample_values': ', '.join(s.dropna().unique()[:3])
    })
profile_df = pd.DataFrame(profile).sort_values('non_null_pct')
profile_df.head(15)
```

Fragen:

- Welche Spalten sind fast leer und sollten ggf. ausgelagert / ignoriert werden?
- Gibt es offensichtliche Duplikate bei `ID`?

## 2. Bereinigung `verbaleDating`

Spalte enthält verbale Zeitangaben / Bereiche wie: `"1000-2020, 1773-1776"`.

Aufgaben:

1. Zerlege `verbaleDating` in einzelne Segmente (Trennzeichen: Komma) → normalisiere Leerzeichen.
2. Identifiziere Bereiche (Pattern `YYYY-YYYY`) vs. Einzeljahre (`YYYY`).
3. Baue eine normalisierte Tabelle `building_dates`: (`building_id`, `date_raw`, `year_start`, `year_end`, `is_range`, `precision`).
4. Berechne je Gebäude: minimaler Start, maximaler Endwert → `chronology_min`, `chronology_max` und füge sie wieder dem Haupt-DataFrame hinzu.
5. Detektiere Ausreißer (z. B. Jahr < 800 oder > aktuelles Jahr). Markiere sie für manuelle Prüfung.

Hinweise:

```python
import re
rows = []
for _, r in buildings[['ID','verbaleDating']].fillna('').iterrows():
    parts = [p.strip() for p in r.verbaleDating.split(',') if p.strip()]
    for p in parts:
        m_range = re.fullmatch(r'(\d{3,4})-(\d{3,4})', p)
        m_year  = re.fullmatch(r'(\d{3,4})', p)
        if m_range:
            y1, y2 = map(int, m_range.groups())
            rows.append((r.ID, p, y1, y2, True, 'year-range'))
        elif m_year:
            y = int(m_year.group(1))
            rows.append((r.ID, p, y, y, False, 'year'))
        else:
            # Nicht-parsbare Fälle separat behalten
            rows.append((r.ID, p, None, None, None, 'unparsed'))

building_dates = pd.DataFrame(rows, columns=['building_id','date_raw','year_start','year_end','is_range','precision'])
```

Optionale Normalisierung unparsed Tokens: RegEx erweitern (z. B. `ca. 1750`, `17. Jh.` → Mapping Tabellen). Dokumentiere Annahmen!

Metriken:

- Anteil parsebarer Segmente.
- Häufigste unparsed Muster (Top 10).

## 3. Geodaten-Validierung

1. Konvertiere `locationLat` / `locationLng` zu Float; markiere Zeilen, bei denen das misslingt.
2. Prüfe Wertebereiche (Lat ∈ [-90, 90], Lng ∈ [-180, 180]).
3. Erstelle Spalten `coord_valid` (bool) und `coord_quality` (Enum: `valid`, `out_of_range`, `non_numeric`).
4. Optional: Entferne identische Koordinaten-Duplikate, falls mehrere Gebäude dieselben Koordinaten teilen sollen – dokumentiere Entscheidungslogik.

Snippet:

```python
def to_float(s):
    try:
        return float(s)
    except (TypeError, ValueError):
        return pd.NA
coords = buildings[['locationLat','locationLng']].applymap(to_float)
valid_lat = coords.locationLat.between(-90,90)
valid_lng = coords.locationLng.between(-180,180)
buildings['coord_valid'] = valid_lat & valid_lng
```

## 4. Normalisierung mehrfacher Rollen-/Personenfelder

Viele Spalten enden auf Sequenzen `_1`…`_10` (z. B. `ARCHITECTS_1`, `ARCHITECTS_2`, ...). Werteform: `uuid|Nachname, Vorname`.

Ziel: Long-Format-Relation `building_person_roles`:
(`building_id`, `role` (z. B. `ARCHITECTS`), `sequence` (Nummer), `person_id` (UUID), `person_label`).

Aufgaben:

1. Identifiziere alle Basis-Rollen (Teil vor letztem `_` + Ziffern).
2. Iteriere über alle diese Spalten, extrahiere Werte ≠ leer.
3. Teile an erster Pipe `|` → `person_id`, zweiter Teil `person_label` (Fallback: kompletter String falls kein `|`).
4. Entferne potenzielle Dubletten (gleiche Kombination building_id + role + person_id).
5. Erzeuge optionale Personentabelle `persons` (distinct `person_id`, `person_label`, `label_clean`).
6. Bereinige `person_label`: Whitespace trimmen, vereinheitliche Kommaspacing.

Snippet-Fragmente:

```python
import itertools
role_cols = [c for c in buildings.columns if re.search(r'_\d+$', c)]
base_roles = sorted(set(re.sub(r'_\d+$','', c) for c in role_cols))
rows = []
for role in base_roles:
    for i in range(1, 11):
        col = f'{role}_{i}'
        if col not in buildings.columns: 
            continue
        for _, r in buildings[['ID', col]].iterrows():
            val = r[col]
            if pd.isna(val) or not str(val).strip():
                continue
            if '|' in val:
                pid, label = val.split('|',1)
            else:
                pid, label = None, val
            rows.append((r.ID, role, i, pid, label.strip()))

building_person_roles = pd.DataFrame(rows, columns=['building_id','role','sequence','person_id','person_label'])
persons = (building_person_roles
           .dropna(subset=['person_label'])
           .groupby(['person_id','person_label'], dropna=False)
           .size().reset_index(name='count'))
```

Validierung:

- Wie viele Rollen wurden extrahiert?
- Top 10 Personen je Rolle.

## 5. Konsolidierung & Export

1. Schlanken Haupt-DataFrame erstellen (`buildings_core`): wesentliche Attribute (ID, appellation, address*, chronology_min/max, coord_valid, locationLat/Lng als Float).
2. Daten in Unterordner `processed/` exportieren:
   - `buildings_core.parquet`
   - `building_dates.parquet`
   - `building_person_roles.parquet`
   - `persons.parquet`
3. Zusätzlich CSV-Export für Interoperabilität (UTF-8, `index=False`).
4. README-Ergänzung (Kurzbeschreibung der erzeugten Dateien) – (Kann in Task 4 weiter genutzt werden).

Snippet Export:

```python
out = Path('../processed'); out.mkdir(exist_ok=True)
buildings_core.to_parquet(out / 'buildings_core.parquet')
building_dates.to_parquet(out / 'building_dates.parquet')
# ...
```

## 6. Qualitätsmetriken & Checks

Erzeuge kleine Metriken (als DataFrame oder Markdown):

- Parsebarkeit Datum (%).
- Anzahl unparsed Datumseinträge.
- Anzahl extrahierter Personenbeziehungen.
- #Distinct Personen.
- Anteil gültiger Koordinaten.

# NRW-Wohngebäudebestand — Clustering-Analyse

## Überblick

Dieses Repositorium enthält die Clustering-Analyse von ~4,1 Mio.
Wohngebäuden in NRW (Nordrhein-Westfalen). Ziel: den Gebäudebestand auf
repräsentative Archetypen (Cluster) reduzieren, die in das
MOSAIQ-Energieoptimierungsmodell eingespeist werden können.

Aufbau und Arbeitsweise folgen dem Muster des niederländischen Pendants
(`netherlands-building-clustering-main`): nummerierte Notebooks, thematische
Datenordner, eine gemeinsame Arbeitsdatei und der **Log-Raum-KMeans mit
Coverage-Kriterium** (aus dessen `05_nl_clustering.ipynb`). Die Datengrundlage
und die Herleitung der Werte (DEA-Datenbasis, CO2online-Strom, TABULA-Wärme,
LOD2-Dachklassifikation) sind die NRW-eigenen.

---

## Wichtig vor dem Start — benötigte Datendateien

Die Notebooks brauchen große Datendateien, die wegen ihrer Größe **nicht im
Repositorium** liegen. Diese sind separat zu besorgen und wie folgt abzulegen:

| Datei                                                                       | Größe | Benötigt von                                                         |
| --------------------------------------------------------------------------- | ------- | --------------------------------------------------------------------- |
| `DEA_combined.parquet`                                                    | ~700 MB | `01_nrw_pipeline.ipynb` (nur zum Neuaufbau)                         |
| `nrw_buildings.parquet`                                                   | ~1 GB   | alle Notebooks — gemeinsame Arbeitsdatei (wird von 01 erzeugt)       |
| `lod2_nrw_roofs.geoparquet`                                               | ~170 MB | `02_nrw_clustering.ipynb`                                           |
| `katasterdaten/wohnflaechen_nrw.parquet`                                  | ~600 MB | `02_nrw_clustering.ipynb`, `05_nrw_rasenflaeche_geothermie.ipynb` |
| `final_tabula.xlsx`                                                       | ~13 MB  | `01_nrw_pipeline.ipynb`                                             |
| `geothermie/geothermie_waermeleitfaehigkeit/ews100_nrw_reduced.parquet`   | —      | `07_nrw_finaler_datensatz.ipynb`                                    |
| `geothermie/wasserschutzgebiete/Wasserschutzgebiete-NRW.shp` (+ Sidecars) | —      | `07_nrw_finaler_datensatz.ipynb`                                    |
| `zensus/zensus_daten_2021_polygone.parquet`                               | —      | `07_nrw_finaler_datensatz.ipynb`                                    |
| `pilotprojects/ochtrup_polygon.json`                                      | —      | `07_nrw_finaler_datensatz.ipynb` (Testgebiet, im Repo enthalten)    |

Alle generierten Ausgaben (Excel-Zusammenfassungen, Boxplots, Parquets,
`final_dataset.parquet`) landen im Ordner **`nrw_results/`**, der von den
Notebooks automatisch angelegt wird.

> **Hinweis zu `01_nrw_pipeline.ipynb`:** Nur nötig, um die Arbeitsdatei von
> Grund auf neu aufzubauen (`DEA_combined.parquet` → `nrw_buildings.parquet`).
> Wer `nrw_buildings.parquet` bereits hat, kann 01 überspringen.

---

## Öffnen und Ausführen

### Anforderungen

```
pip install pandas geopandas numpy matplotlib openpyxl pyarrow demandlib shapely requests
```

### Wichtig — VS Code aus dem Projektordner heraus öffnen

Alle Notebooks nutzen relative Pfade (`BASE_DIR = "."`). Das funktioniert nur,
wenn VS Code **aus dem Projektordner** geöffnet wird.

**Richtig:** Rechtsklick auf den Ordner → **Open with Code**
**Falsch:** eine einzelne `.ipynb`-Datei in VS Code ziehen

Bei direkt geöffneter Notebook-Datei lösen sich die Pfade nicht auf und es
gibt einen `FileNotFoundError`.

---

## Die gemeinsame Arbeitsdatei

Wie im NL-Repositorium arbeitet die Pipeline auf **einer** Arbeitsdatei:
`nrw_buildings.parquet` im Projektstamm (das Pendant zu
`nl_buildings.parquet`). Jedes Notebook liest sie, reichert sie um seine
Spalten an und überschreibt sie:

```
01  erzeugt   nrw_buildings.parquet        (Strom, Wärme, PV-Zonen)
02  reichert  Cluster- + Dachspalten an
03  reichert  Spitzenleistung an
05  reichert  Rasenfläche + Geothermie-Flag 1 an
07  liest     alles und schreibt nrw_results/final_dataset.parquet
```

---

## Notebooks

| Notebook                                 | Beschreibung                                                                                                                                                                                                                 |
| ---------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `01_nrw_pipeline.ipynb`                | **Optional** — baut `nrw_buildings.parquet` aus `DEA_combined.parquet` auf (CO2online-Strom, TABULA-Wärme, PV-Ertrag, TRY-Zonen)                                                                                 |
| `02_nrw_clustering.ipynb`              | Parzellen-Filter (bestätigt wohnbaulich) + Bedarfs-/Dach-Clustering per Log-Raum-KMeans + Zuweisung der Cluster-Repräsentanten an alle Gebäude. Erzeugt`Clustering_summary.xlsx` und `Demand_/Roof_*_assignments.csv` |
| `03_nrw_spitzenwaermeleistung.ipynb`   | BDEW-Lastprofile: Cluster-Wärmebedarf → Spitzenleistung (kW) je Sanierungsszenario                                                                                                                                         |
| `04_nrw_geothermie_kennlinie.ipynb`    | Standalone: Erdsonden-Kennlinie (Spitzenleistung → Sondenzahl → Fläche), nPro-Formel aus VDI 4640. Schreibt`geothermie_kennlinie.csv`                                                                                   |
| `05_nrw_rasenflaeche_geothermie.ipynb` | Rasenfläche per Parzellen-Left-Join + erstes Geothermie-Flag (Rasenfläche ≥ Sondenfläche)                                                                                                                                |
| `06_nrw_luftwaermepumpe.ipynb`         | **Leerer Platzhalter** — hier soll die Luft-Wärmepumpen-Logik aus 07 ausgelagert werden                                                                                                                              |
| `07_nrw_finaler_datensatz.ipynb`       | Geothermie-Flags 2+3 (Wärmeleitfähigkeit, Wasserschutzgebiete), Gasverfügbarkeit (Zensus), Luft-WP-Flag, finale Spalten →`nrw_results/final_dataset.parquet` + Archetyp-Kombinationen für das Testgebiet Ochtrup      |
| `schwellenwertvergleich_p95_p99.ipynb` | Optional: P95- vs P99.5-Ausreißer-Entfernung, unnummeriert wie`clustering_alternative.ipynb` im NL-Repo                                                                                                                   |

---

## Empfohlene Laufreihenfolge

```
1. 01_nrw_pipeline.ipynb           — optional, nur zum Neuaufbau der Arbeitsdatei
2. 02_nrw_clustering.ipynb            — Parzellen-Filter + Log-KMeans + Zuweisung
3. 03_nrw_spitzenwaermeleistung.ipynb  — braucht heat_cluster_1/2/3 aus 02
4. 04_nrw_geothermie_kennlinie.ipynb   — standalone, auch vor 02 möglich
5. 05_nrw_rasenflaeche_geothermie.ipynb — braucht Spitzenleistung (03) + Kennlinie (04)
6. 06_nrw_luftwaermepumpe.ipynb        — Platzhalter, überspringen
7. 07_nrw_finaler_datensatz.ipynb      — braucht die Arbeitsdatei nach 05
8. schwellenwertvergleich_p95_p99.ipynb — optional, zur Referenz
```

Keine Zirkularität, kein Notebook muss zweimal laufen.

---

## Der Clustering-Algorithmus (Log-Raum-KMeans)

Portiert aus `05_nl_clustering.ipynb` des NL-Repositoriums — dieselbe Methode,
auf NRW-Daten angewendet:

- **Eigene 1D-Lloyd-Implementierung im Log-Raum** — eine absolute Distanz im
  Log-Raum entspricht einer relativen Distanz im Originalraum. Cluster-Zentren
  sind deshalb **geometrische Mittel** (`exp(mean(log(werte)))`), und die
  Metrik passt direkt zum Coverage-Kriterium.
- **Coverage-Kriterium:** k steigt von 1 aufwärts, bis
  ≥ 80 % der Gebäude höchstens ±20 % relative Abweichung von ihrem Zentrum
  haben (Obergrenze `K_MAX = 12`).
- **Deterministisch:** Quantil-Initialisierung, kein Sampling, kein Seed —
  gleiche Eingabe ergibt identisches Ergebnis, Cluster-Nummern sind über Runs
  stabil.

NRW-spezifische Festlegungen (Datengrundlage/Herleitung):

- der **Parzellen-Filter** (Wohnbauflächen-Inner-Join) als Trainingsmenge,
- der **P99.5-Ausreißer-Cap** je Gebäudetyp vor dem Bedarfs-Clustering,
- der **Sonderfall SFH/TH-Strom**: nur 2 mögliche Werte (2.700 / 3.500 kWh aus
  der CO2online-Tabelle) — fest verdrahtet als k=2, weil das Coverage-Kriterium
  beide Werte auf k=1 kollabieren ließe (Abweichung ~12–14 % < ±20 %) und damit
  die Personenklassen-Unterscheidung verwerfen würde,
- die **LOD2-Dach-Klassifikation**: Flachdach < 10° Neigung, Ost+West
  kombiniert, Nord ausgeschlossen, Sattel-/Flachdach getrennt geclustert.

Methodisch (Teil des Algorithmus, nicht der Wert-Herleitung): Nullwerte werden
vor dem Fit ausgeschlossen und erhalten fest 0.0 — dadurch entstehen keine
künstlichen „Nullen-Cluster".

Die Zielabdeckung gilt global für alle Kombinationen: **80 %**. Das
Ausnahmen-Dict `COVERAGE_MIN_AUSNAHMEN` ist derzeit leer und bleibt als
Mechanismus für die gezielte Feinsteuerung einzelner (Gebäudetyp,
Merkmal)-Kombinationen erhalten — Kandidaten erkennt man im `Uebersicht`-Sheet
der Excel-Zusammenfassung an `coverage < coverage_ziel`.

---

## CO2online-Lookup-Tabelle (Strombedarf)

Verwendete Werte: **ohne elektrische Warmwasserbereitung** (das SESMG optimiert
Warmwasser separat). Quelle: CO2online Stromspiegel via Statista, Mai 2025.

| Personen | SFH/TH (kWh/Jahr) | MFH/AB (kWh/Jahr) |
| -------- | :---------------: | :---------------: |
| 1        |       1.800       |       1.200       |
| 2        |       2.700       |       1.900       |
| 3        |       3.500       |       2.400       |
| 4        |       4.000       |       2.900       |
| 5        |       4.500       |       3.100       |

---

## Ausreißer-Entfernung — P99.5

Falsch klassifizierte Industrie-, Hotel- und Hofgebäude werden vorher durch
den Parzellen-Filter ausgeschlossen. Das Cap hier ist ein separater, späterer
Schritt: die oberen 0,5 % der Gebäude nach `Al_m2` werden je Gebäudetyp vor dem
Clustering gekappt, weil ohne sie die größten 1-2 Cluster aufblähen und die
deutlich häufigeren kleinen Bedarfe unterrepräsentieren.

Das aggressivere P95 (obere 5 %) wurde getestet und verworfen — es entfernt zu
viele echte Gebäude. Details: `schwellenwertvergleich_p95_p99.ipynb`.

| Schwelle                        | Entfernte Gebäude | SFH-Schwelle | AB-Schwelle |
| ------------------------------- | :----------------: | :----------: | :---------: |
| P99.5 (aktuell)                 |   20.669 (0,5 %)   |   441 m²   |  2.199 m²  |
| P95 (verworfen — zu aggressiv) |  206.668 (5,0 %)  |   217 m²   |   940 m²   |

---

## Zahlenstand — nach dem nächsten Lauf bestätigen

Zahlenstand: **letzter vollständiger Lauf der Pipeline.** Nach dem nächsten
Lauf sind sie hier zu bestätigen und zu aktualisieren:

### Bedarfs-Clustering je Gebäudetyp (letzter Lauf, P99.5)

| Gebäudetyp |       k Strom       | k Wärme | Kombinationen |
| ----------- | :-----------------: | :------: | :-----------: |
| SFH         | 2 (fest verdrahtet) |    4    |       8       |
| TH          | 2 (fest verdrahtet) |    4    |       8       |
| MFH         |          4          |    5    |      20      |
| AB          |          4          |    4    |      16      |

### Dach-Clustering (LOD2, letzter Lauf)

| Gebäudetyp | Satteldach | Flachdach | Gesamt |
| ----------- | :--------: | :-------: | :----: |
| SFH         |  4×4=16  |     4     |   20   |
| TH          |  5×4=20  |     4     |   24   |
| MFH         |  4×4=16  |     4     |   20   |
| AB          |  5×4=20  |     5     |   25   |

### Geothermie-Machbarkeit (letzter Lauf)

| Gebäudetyp |  Machbar  | Zugeordnet |   %   |
| ----------- | :-------: | :--------: | :----: |
| SFH         | 2.461.069 | 2.485.862 | 99,0 % |
| TH          |  471.733  |  492.569  | 95,8 % |
| MFH         |  677.025  |  724.106  | 93,5 % |
| AB          |   4.058   |   4.300   | 94,4 % |

Parzellen zugeordnete Gebäude: **3.705.834 (89,7 %)** — ausgeschlossen
(Hotels, Garagen, Industrie, Mischnutzung): 427.489 (10,3 %).

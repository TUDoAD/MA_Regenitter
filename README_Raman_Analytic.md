# Raman Analytic – Standalone PLS-GUI (`Raman_Analytic.py`)

Eigenständige Desktop-Anwendung (Tkinter) zur Auswertung von **Raman-Spektren mit einem eingebetteten PLS1-Modell** und zur Berechnung von **NADH-Konzentrationen bzw. NAD⁺-Umsatz aus UV-Vis-Daten** mit eingebetteter Kalibrierung.

> **Besonderheit dieser Version:** Das trainierte Modell und die UV-Vis-Kalibrierung stehen **als Base64-Text in der Python-Datei selbst** (Z. 62–64). Es wird weder ein `Modelle/`-Ordner noch eine Excel-Datei benötigt – nur die Messdaten (CSV) und installierte Python-Pakete.
>
> Alle Zeilenangaben (`Z. 123`) beziehen sich auf die vorliegende Datei. Nach Änderungen können sie abweichen; die Funktionsnamen bleiben als Suchbegriff gültig.

---

## Inhaltsverzeichnis

1. [Was macht das Programm?](#1-was-macht-das-programm)
2. [Schnellstart](#2-schnellstart)
3. [Dateiaufbau im Überblick](#3-dateiaufbau-im-überblick)
4. [Schritt 0 – Eingebettete Ressourcen und Konfiguration](#4-schritt-0--eingebettete-ressourcen-und-konfiguration)
5. [Schritt 1 – Start: Pakete prüfen, Imports](#5-schritt-1--start-pakete-prüfen-imports)
6. [Schritt 2 – Modell laden und prüfen](#6-schritt-2--modell-laden-und-prüfen)
7. [Schritt 3 – Raman-Auswertung (Schritt für Schritt)](#7-schritt-3--raman-auswertung-schritt-für-schritt)
8. [Schritt 4 – UV-Vis / NADH-Auswertung](#8-schritt-4--uv-vis--nadh-auswertung)
9. [Schritt 5 – Darstellung, Tabelle, Plots](#9-schritt-5--darstellung-tabelle-plots)
10. [Schritt 6 – Kombinierter Plot und Plateau-Analyse](#10-schritt-6--kombinierter-plot-und-plateau-analyse)
11. [Schritt 7 – Export der Ergebnisse](#11-schritt-7--export-der-ergebnisse)
12. [Bedienung der GUI](#12-bedienung-der-gui)
13. [Das eingebettete Modell im Detail](#13-das-eingebettete-modell-im-detail)
14. [Anpassen und Erweitern](#14-anpassen-und-erweitern)
15. [Fehlersuche](#15-fehlersuche)
16. [Bekannte Besonderheiten und Hinweise](#16-bekannte-besonderheiten-und-hinweise)

---

## 1. Was macht das Programm?

```text
Raman-CSV-Ordner ──► Einlesen ──► Interpolation auf Modellachse ──► Despiking
        ──► modellspezifische Vorverarbeitung ──► PLS1-Vorhersage je Analyt
        ──► Bereichsprüfung ──► (Mittelwert/SD bei Wiederholungen)
        ──► Tabelle + Plots ──► automatischer CSV/JSON-Export

UV-Vis-CSV ──► Absorbanz ──► lineare Kalibrierung ──► NADH [mmol/L] ──► NAD⁺-Umsatz [%]
```

Ausgewertete Analyten (aus dem eingebetteten Modell): **1-Phenylethanol, Cyclohexanon (in der GUI „Cyclohexanone“), Cyclohexanol**.

---

## 2. Schnellstart

1. Datei `Raman_Analytic.py` in **VS Code** öffnen.
2. Oben rechts **„Run Python File“** klicken (oder `python Raman_Analytic.py`).
3. Beim ersten Start installiert das Programm fehlende Pakete automatisch (siehe [Schritt 1](#5-schritt-1--start-pakete-prüfen-imports)).
4. Es öffnen sich **zwei Fenster**: *Raman PLS – Concentration Analysis* und *UV-Vis – NADH Analysis*.
5. Im Raman-Fenster **zuerst den Messtyp wählen**, dann einen Ordner mit CSV-Dateien in die Drop-Zone ziehen oder **Select folder** klicken.
6. Ergebnisse erscheinen in Tabelle und Plots; CSV/JSON werden automatisch im Unterordner `Raman_PLS_Results/` gespeichert.

**Voraussetzungen:** Python 3 mit Tkinter (unter macOS ggf. Python von python.org installieren). Benötigte Pakete: `numpy`, `pandas`, `openpyxl`, `scipy`, `scikit-learn==1.8.0`, `joblib`, `matplotlib`, `tkinterdnd2`.

---

## 3. Dateiaufbau im Überblick

| Abschnitt | Zeilen | Inhalt |
|---|---|---|
| Docstring | 4–43 | Kurzbeschreibung, Ablauf, Startanleitung |
| 0. Dependency bootstrap | 45–169 | Eingebettete Daten, Konstanten, `legend_position_settings` (106), `ensure_packages` (133) |
| 1. Runtime imports | 171–199 | numpy, pandas, scipy, matplotlib (TkAgg), Tkinter, optional Drag & Drop |
| 2. Embedded model loading | 201–261 | `load_pls_model` (205), `save_figure_high_quality` (229) |
| 3. UV-Vis | 263–471 | `load_uv_vis_calibration` (267), `read_uv_vis_csv` (357), `calculate_uv_vis_concentrations` (453) |
| 4. Raman CSV input | 473–511 | `read_spectrum` (477) |
| 5. Preprocessing | 513–699 | `despike` (517), `preprocess` (537), `preprocess_with_config` (594) |
| 6. Gruppierung | 701–784 | `sample_key` (705), `natural_sort_key` (731), `group_csv_files` (745), `list_csv_files` (771) |
| 7. Laden/Mitteln | 786–815 | `load_mean_spectrum` (790) |
| 8. PLS1-Vorhersage | 817–940 | `predict_sample` (821), `analyte_display_name` (879), `set_errorbar_visibility` (893), `summarize_replicate_predictions` (909) |
| 9. GUI | 942–5246 | `plateau_statistics` (946), `PlateauAnalysisPanel` (972), `RamanPLSApp` (1204) |
| 10. Programmstart | 5248–5263 | `if __name__ == "__main__"` (5250) |

### Die Klasse `RamanPLSApp` nach Themen

| Thema | Methoden (Zeilen) |
|---|---|
| Initialisierung & Fenster | `__init__` (1206), `show_analysis_window` (1276), `close_analysis_window` (1282) |
| GUI-Aufbau | `build_gui` (1293), `set_initial_plot_pane_size` (2131), `build_uv_vis_block` (2150) |
| UV-Vis | `get_uv_vis_calibration` (2472), `scale_nadh_axes` (2554), `choose_uv_vis_file` (2694), `collect_uv_vis_csv_paths` (2721), `on_uv_vis_drop` (2771), `analyze_uv_vis_files` (2809), `update_uv_vis_table` (2951), `draw_nadh_concentration_plot` (2990), `save_nadh_plot_png` (3190), `save_uv_vis_csv` (3236) |
| Darstellungsoptionen | `refresh_spectra_legend` (3334), `update_analyte_toggles` (3384), `toggle_analyte_visibility` (3500), `rolling_average_window` (3574), `toggle_standard_deviation_visibility` (3625), `refresh_concentration_legend` (3653) |
| Messtyp & Upload | `on_measurement_mode_selected` (3790), `get_upload_settings` (3822), `on_measurement_interval_changed` (3884), `choose_folder` (3909), `on_drop` (3933) |
| **Raman-Analyse** | **`analyze_folder` (3971)** |
| Tabelle | `sort_table_by_column` (4232), `update_table` (4305) |
| Raman-Plots | `update_plot` (4348) |
| Export | `write_table_csv` (4718), `write_all_result_files` (4737), `save_results` (4795), `save_table_csv` (4819), `save_all_to_folder` (4857), `save_plot_files` (4908), `save_plot_png` (5178) |
| Zusatzfenster | `refresh_combined_plot_availability` (4952), `open_plateau_plot` (4966), `open_combined_plot` (4970) |
| Hauptschleife | `run` (5242) |

---

## 4. Schritt 0 – Eingebettete Ressourcen und Konfiguration

### 4.1 Eingebettete Daten (Z. 61–64)

```python
EMBEDDED_MODEL_BASE64 = "gAWVvQIAAAAAAAB9..."          # Z. 62 (ca. 1 MB Text)
EMBEDDED_MODEL_SHA256 = "c820398a62fecf6e1c7a7701e6253dd9d439b80c13a03c19c14c4f5078058789"  # Z. 63
EMBEDDED_UV_CALIBRATION_BASE64 = "UEsDBBQABgAIAAAAIQBi7p1o..."  # Z. 64 (Excel-Datei)
```

**Erklärung:**

- `EMBEDDED_MODEL_BASE64` enthält die komplette trainierte Joblib-Modelldatei (rund 760 kB binär) als Textzeile. Diese Zeile ist über eine Million Zeichen lang – **nicht in Editoren mit Zeilenumbruch öffnen oder manuell bearbeiten**.
- `EMBEDDED_MODEL_SHA256` ist die Prüfsumme der entschlüsselten Bytes. Sie stellt sicher, dass das Modell unverändert ist (Prüfung in `load_pls_model`, Z. 209).
- `EMBEDDED_UV_CALIBRATION_BASE64` ist eine Excel-Datei (`.xlsx` beginnt mit `PK` → Base64 `UEsD…`) mit den UV-Vis-Kalibrierdaten.
- Der Kommentar in Z. 61 hält fest: Die Daten werden **im Speicher** dekodiert; es wird keine temporäre Datei geschrieben.

### 4.2 Konstanten (Z. 65–104)

| Konstante | Zeile | Bedeutung |
|---|---|---|
| `MODEL_SKLEARN_VERSION = "1.8.0"` | 65 | scikit-learn-Version, mit der das Modell gespeichert wurde |
| `PNG_EXPORT_DPI = 2400` | 66 | Auflösung beim PNG-Export (sehr hoch → große Dateien) |
| `NAD_PLUS_LABEL`, `NAD_PLUS_PLOT_LABEL` | 67–68 | Achsentitel „NAD⁺ conversion / %“ (Text bzw. Matplotlib-Mathtext) |
| `NADH_CONVERSION_PERCENT_COLUMN` | 69 | Spaltenname `NADH_conversion_percent` |
| `NADH_Y_TICK_INTERVAL_OPTIONS` | 70 | Wählbare Y-Achsen-Intervalle des NADH-Plots (5, 10, 20, 25, 50) |
| `LEGEND_POSITION_OPTIONS` | 77 | Legendenoptionen (Below, Upper right, …, Hidden) |
| `LEGEND_POSITION_LOCATIONS` | 86 | Zuordnung GUI-Name → Matplotlib-`loc` |
| `REQUIRED_PACKAGES` | 94 | Import-Name → pip-Name (mit gepinntem scikit-learn) |

### 4.3 `legend_position_settings` (Z. 106)

Übersetzt die GUI-Auswahl in drei Rückgabewerte: *Legende sichtbar?*, *außerhalb der Achsen?*, *Matplotlib-Argumente*. Bei **„Below“** wird `bbox_to_anchor=(1.0, -0.20)` verwendet, damit die Legende unter dem Diagramm steht.

---

## 5. Schritt 1 – Start: Pakete prüfen, Imports

### 5.1 `ensure_packages()` (Z. 133–167, Aufruf in Z. 169)

```python
for module_name, pip_name in REQUIRED_PACKAGES.items():
    try:
        module = importlib.import_module(module_name)
        if module_name == "sklearn" and module.__version__ != MODEL_SKLEARN_VERSION:
            install_requests.append(pip_name)
    except ImportError:
        install_requests.append(pip_name)
```

**Erklärung:**

- Jedes Paket wird testweise importiert. Fehlt es, kommt es auf die Installationsliste.
- **scikit-learn** wird zusätzlich auf die **exakte Version `1.8.0`** geprüft. Grund: Das Modell wurde mit dieser Version „gepickelt“; andere Versionen können es falsch oder gar nicht laden (der Kommentar in Z. 48–50 weist darauf hin).
- Danach wird einmalig `pip install` über `sys.executable -m pip` aufgerufen – also in **derselben Python-Umgebung**, in der das Skript läuft.
- Der Aufruf steht **vor** den eigentlichen Imports (Z. 169 vor Z. 175), damit nach der Installation alles direkt importiert werden kann.

> ⚠️ Eine abweichende scikit-learn-Version wird in der aktiven Umgebung überschrieben. Am besten ein eigenes virtuelles Environment verwenden.

### 5.2 Runtime-Imports (Z. 175–198)

- `matplotlib.use("TkAgg")` (Z. 181): Diagramme werden in Tkinter-Fenstern gezeichnet.
- `savefig.dpi`, `savefig.facecolor`, `savefig.bbox` (Z. 182–184): globale Export-Standards (2400 dpi, weißer Hintergrund, enger Zuschnitt).
- Drag & Drop (Z. 190–198): Der Import von `tkinterdnd2` steht in einem `try`-Block. Gelingt er nicht, wird `DND_AVAILABLE = False` gesetzt – das Programm läuft dann ohne Ziehen-und-Ablegen (nur Button **Select folder**).

---

## 6. Schritt 2 – Modell laden und prüfen

### `load_pls_model()` (Z. 205–226)

```python
model_bytes = base64.b64decode(EMBEDDED_MODEL_BASE64)
if hashlib.sha256(model_bytes).hexdigest() != EMBEDDED_MODEL_SHA256:
    raise ValueError("The embedded model data are damaged.")
package = joblib.load(io.BytesIO(model_bytes))
```

**Erklärung (Zeile für Zeile):**

1. `base64.b64decode(...)` wandelt den Text zurück in Binärdaten.
2. Der SHA-256-Hash dieser Daten wird mit der erwarteten Prüfsumme verglichen. **Stimmt er nicht**, bricht das Programm ab (Modell beschädigt oder verändert).
3. `joblib.load(io.BytesIO(...))` lädt das Modell direkt **aus dem Arbeitsspeicher** – ohne Datei.
4. Anschließend wird geprüft, ob die Pflichtschlüssel `x_axis_cm-1`, `pls1_models` und `model_status` vorhanden sind; sonst `ValueError` mit Liste der fehlenden Einträge.

Das geladene Paket wird beim Start in `RamanPLSApp.__init__` (Z. 1206) gespeichert:

```python
self.package = load_pls_model()
self.x_model = np.asarray(self.package["x_axis_cm-1"], dtype=float)
```

`self.x_model` ist die **Raman-Achse des Trainings** (3528 Punkte, ca. 99,5 bis 3500,4 cm⁻¹). Alle Messspektren werden später auf genau diese Achse gebracht.

---

## 7. Schritt 3 – Raman-Auswertung (Schritt für Schritt)

Zentrale Methode: **`analyze_folder`** (Z. 3971–4197). Im Code sind die Schritte als `# Step 1 … Step 11` kommentiert.

### 7.0 Vorarbeit: Messtyp wählen (Pflicht)

Vor dem Upload muss im Raman-Fenster ein Messtyp gewählt werden (`measurement_mode_var`, Z. 1403; Prüfung in `get_upload_settings`, Z. 3822):

| Messtyp | Interner Wert | Bedeutung |
|---|---|---|
| **Single measurement run** | `"single"` | Jede CSV = ein eigener Zeitpunkt. **Messintervall in Minuten (> 0) ist Pflicht.** |
| **Repeated measurement run** | `"multiple"` | CSVs mit gleichem Probennamen = technische Wiederholungen. Intervall optional. |

Ohne Auswahl erscheint die Warnung *„Missing measurement type“* und der Upload wird abgebrochen.

### 7.1 Schritt 1–2: Modus lesen, Ordner auflösen (Z. 3978–3996)

```python
if measurement_mode == "multiple":
    interval_min = None
folder = Path(folder).expanduser().resolve()
```

Im Modus *multiple* wird das Intervall auf `None` gesetzt (Zeit ist dort nicht definiert). Der Ordnerpfad wird zu einem absoluten Pfad aufgelöst.

### 7.2 Schritt 3: Dateien sammeln und gruppieren (Z. 3997–4017)

**Single:**

```python
grouped = {path.stem: [path] for path in csvs}
```

Jede CSV bildet eine eigene Probe (Schlüssel = Dateiname ohne Endung).

**Repeated:** `group_csv_files` (Z. 745) gruppiert über `sample_key` (Z. 705):

```python
patterns = [
    r"^(.*?)[_-](\d+)$",        # V1_1, V1-2   -> V1
    r"^(.*?)\s*\((\d+)\)$",     # Probe (4)    -> Probe
]
```

| Dateiname | Probenschlüssel |
|---|---|
| `V1_1.csv`, `V1_2.csv` | `V1` |
| `Probe-3.csv` | `Probe` |
| `Probe (4).csv` | `Probe` |
| `Messung.csv` | `Messung` (keine Nummer erkannt) |

**Sonderfall (Z. 763):**

```python
if len(grouped) == len(csvs) and len(csvs) > 1:
    return {folder.name: csvs}
```

Hat jede Datei einen eigenen Schlüssel (keine Wiederholungsnummern erkannt), wird der **gesamte Ordner als eine Probe** behandelt (Name = Ordnername).

Die Dateien werden **natürlich sortiert** (`natural_sort_key`, Z. 731: `Probe2` vor `Probe10`). Versteckte Dateien und Excel-Sperrdateien werden von `list_csv_files` (Z. 771) ignoriert.

### 7.3 Schritt 5: Spektren einlesen und interpolieren

**`read_spectrum`** (Z. 477–510): Erwartet zwei Spalten (Raman-Verschiebung, Intensität) **ohne feste Kopfzeile**. Vier Formate werden nacheinander probiert:

| Trennzeichen | Dezimalzeichen |
|---|---|
| `;` | `,` |
| `,` | `.` |
| Tab | `,` |
| Tab | `.` |

```python
df = df.apply(pd.to_numeric, errors="coerce").dropna()
if len(df) > 100:
    return df.sort_values("shift").drop_duplicates("shift").reset_index(drop=True)
```

- Kopfzeilen und ungültige Zellen werden zu `NaN` und entfernt.
- Die Datei gilt nur als gültig, wenn **mehr als 100 Zeilen** übrig bleiben.
- Sortierung nach Verschiebung, doppelte x-Werte werden entfernt.

**`load_mean_spectrum`** (Z. 790–814):

```python
raw.append(np.interp(x_model, x, y))     # Z. 804
matrix = np.vstack(raw)
return matrix.mean(axis=0), matrix
```

- `np.interp` legt jedes Spektrum **exakt auf die Modellachse** (`x_model`). So stimmen die Variablen mit dem Training überein, egal welche Schrittweite das Messgerät verwendet.
- Zurückgegeben werden das **Mittelspektrum** (für die Anzeige) und die **Matrix aller Einzelspektren** (für Wiederholungsvorhersagen).

### 7.4 Schritt 6: Vorhersage (`predict_sample`, Z. 821–876)

Für jeden Analyten aus `package["pls1_models"]` laufen fünf Teilschritte:

**(a) Vorverarbeitung** – `preprocess_with_config` (Z. 594), siehe [7.5](#75-vorverarbeitung-im-detail).

**(b) Spektralbereich wählen** (Z. 846):

```python
mask = np.asarray(config["mask"], dtype=bool)
```

Die Maske enthält die beim Training ausgewählten Wellenzahlbereiche (Fingerprint-/CH-Bereich o. ä.). Nur diese Punkte gehen ins Modell.

**(c) Konzentration berechnen** (Z. 854):

```python
raw_prediction = float(config["model"].predict(transformed[mask].reshape(1, -1)).reshape(-1)[0])
```

`reshape(1, -1)` macht aus dem Vektor eine Zeile (eine Probe), `.predict` liefert die PLS-Vorhersage in **mmol/L**.

**(d) Bereichsprüfung** (Z. 859–862):

```python
low, high = config["calibration_range_mM"]
valid = bool(np.isfinite(raw_prediction) and low <= raw_prediction <= high)
output_prediction = raw_prediction if valid else np.nan
```

Liegt die Vorhersage **außerhalb des Kalibrierbereichs** (oder ist nicht endlich), wird der Ausgabewert zu `NaN`. Der **Rohwert bleibt** in `raw_prediction_mM` erhalten. Da die untere Grenze 0 ist, werden auch **negative Vorhersagen verworfen**.

**(e) Ergebnis-Dictionary** je Analyt mit `prediction_mM`, `raw_prediction_mM`, `range_low`, `range_high`, `outside_range`, `valid_count`, `excluded_count`, `status`.

### 7.5 Vorverarbeitung im Detail

`preprocess_with_config` (Z. 594–698) nutzt die **je Analyt im Modell gespeicherten Parameter** (`preprocess_params`). Fehlen sie (ältere Modelle), wird `preprocess` (Z. 537) mit festen Parametern verwendet.

**Immer zuerst: Despiking** (`despike`, Z. 517):

```python
median = medfilt(y, kernel_size=kernel)                         # Z. 521
residual = y - median
mad = np.median(np.abs(residual - np.median(residual))) + 1e-12
bad = np.abs(residual) > threshold * 1.4826 * mad               # Z. 528
result = y.copy()
result[bad] = median[bad]
```

- Ein Medianfilter (Kernel 5) schätzt das Signal ohne schmale Spikes (z. B. kosmische Strahlung).
- Die Streuung wird robust über die **MAD** (Median Absolute Deviation) bestimmt; `1.4826` rechnet sie in eine Standardabweichung um.
- Nur Punkte, die mehr als **8 robuste Standardabweichungen** abweichen, werden durch den Medianwert ersetzt – alle anderen bleiben unverändert.

**Danach je nach Modus:**

| Modus | Ablauf |
|---|---|
| `bc` | Savitzky-Golay-Glättung → Basislinie (breites SG-Fenster) abziehen |
| `snv` | wie `bc`, danach **Standard Normal Variate**: `(x − Mittelwert) / (Std + 1e-12)` |
| `d1` | 1. Ableitung per Savitzky-Golay (`delta` = Median der Achsenschritte) |
| `d1snv` | wie `d1`, danach SNV |

Kernzeilen für `bc`/`snv`:

```python
smooth   = savgol_filter(cleaned, window_length=smooth_window, polyorder=smooth_poly)
baseline = savgol_filter(smooth,  window_length=baseline_window, polyorder=baseline_poly)
corrected = smooth - baseline
```

Die Basislinie wird als **sehr stark geglättete Version** des Spektrums geschätzt (Fensterlänge z. B. 301 Punkte) und abgezogen. Bei SNV wird mit `ddof=1` (Stichproben-Std) normiert; die Konstante `1e-12` verhindert Division durch 0.

Ein unbekannter Modus löst einen `ValueError` aus.

### 7.6 Schritt 6a/6b/7: Single vs. Repeated

| | Single (`"single"`) | Repeated (`"multiple"`) |
|---|---|---|
| Eingang der Vorhersage | Mittelspektrum (= die eine CSV) | **jedes Replikat einzeln** |
| Aggregation | keine | `summarize_replicate_predictions` (Z. 909) |
| `Concentration_SD_mmol_L` | `NaN` | Stichproben-Standardabweichung |

**`summarize_replicate_predictions`** (Z. 909–939):

```python
valid = np.isfinite(predictions) & (predictions >= low) & (predictions <= high)
accepted = predictions[valid]
...
"prediction_mM": float(accepted.mean()) if accepted.size else np.nan,
"standard_deviation_mM": float(np.std(accepted, ddof=1)) if accepted.size > 1 else np.nan,
```

- Nur Replikate **innerhalb** des Kalibrierbereichs gehen in Mittelwert und SD ein.
- SD mit `ddof=1`; bei weniger als zwei gültigen Replikaten → `NaN`.
- `Valid_count` und `Excluded_count` dokumentieren, wie viele Replikate verwendet bzw. verworfen wurden.
- `raw_prediction_mM` ist der Mittelwert **aller endlichen** Rohvorhersagen (ohne Bereichsfilter).

### 7.7 Schritt 8–9: Ergebniszeilen und Zeitachse (Z. 4106–4170)

Pro **Probe × Analyt** entsteht eine Zeile. Die Zeit wird so berechnet (Z. 4143–4146):

```python
row["Time_min"] = sample_index * interval_min if interval_min is not None else np.nan
```

Zeit = **Position der Probe × Messintervall**. Ohne Intervall bleibt sie `NaN` – dann sind Zeitplots und der kombinierte Plot nicht verfügbar.

**Spalten der Ergebnistabelle:**

| Spalte | Bedeutung |
|---|---|
| `Sample`, `Time_min`, `Sample_order` | Probe, Zeit, Reihenfolge |
| `Analyte` | Analytname |
| `Concentration_mmol_L` | Ausgabewert (`NaN` außerhalb des Kalibrierbereichs) |
| `Concentration_SD_mmol_L` | SD über Replikate (nur „repeated“) |
| `Raw_concentration_mmol_L` | Rohvorhersage ohne Bereichsfilter |
| `Valid_count`, `Excluded_count` | gültige / verworfene Vorhersagen |
| `Raw_replicate_predictions_mmol_L` | Liste aller Einzelvorhersagen |
| `Calibration_range_mmol_L` | z. B. `0-120` |
| `Outside_calibration_range` | Bool |
| `Spectra_count` | Anzahl CSVs der Probe |

### 7.8 Schritt 10–11: Anzeige und Autosave

```python
self.update_table(results)
self.update_plot(means, results, measurement_mode)
self.save_results(auto=True)
```

Tabelle (`update_table`, Z. 4305) und Plots (`update_plot`, Z. 4348) werden aktualisiert, dann werden **CSV und JSON automatisch gespeichert** (Plots nicht – siehe [Export](#11-schritt-7--export-der-ergebnisse)). Die Statusleiste meldet Anzahl Proben, CSVs und verworfene Vorhersagen.

**Fehlerbehandlung:** Jede Ausnahme in `analyze_folder` landet im `except`-Block: Statusleiste „Analysis failed.“ plus Dialog *„Analysis error“* mit der Meldung.

---

## 8. Schritt 4 – UV-Vis / NADH-Auswertung

Bedienung im zweiten Fenster (`build_uv_vis_block`, Z. 2150): **Select NADH CSV files** (`choose_uv_vis_file`, Z. 2694) oder Drag & Drop (`on_uv_vis_drop`, Z. 2771). Mehrere Dateien und ganze Ordner sind möglich (`collect_uv_vis_csv_paths`, Z. 2721). Die Auswertung läuft in `analyze_uv_vis_files` (Z. 2809).

### 8.1 Kalibrierung aus eingebetteter Excel-Datei (`load_uv_vis_calibration`, Z. 267–354)

```python
calibration_sheet = pd.read_excel(io.BytesIO(base64.b64decode(EMBEDDED_UV_CALIBRATION_BASE64)),
                                  sheet_name=0, header=None)
concentration_row = pd.to_numeric(calibration_sheet.iloc[4], errors="coerce")
...
absorbance_measurements = calibration_sheet.iloc[5:14, calibration_columns]
mean_absorbance = absorbance_measurements.mean(axis=0).to_numpy(dtype=float)
```

**Layout des ersten Blatts:**

- **Zeile 5** (Index 4): Kalibrierkonzentrationen (nur numerische Zellen zählen).
- **Zeilen 6–14** (Index 5–13): Absorbanz-Messwerte je Konzentration → **Spaltenmittel**.

**Lineare Regression** (Z. 315):

```python
slope, intercept = np.polyfit(mean_absorbance[valid], concentrations[valid], 1)
```

Gerade `c [mmol/L] = Steigung · Absorbanz + Achsenabschnitt`. Mindestens **2 gültige Paare** sind nötig. Zusätzlich wird **R²** berechnet:

```python
r_squared = 1.0 - ss_residual / ss_total if ss_total > 0 else np.nan
```

Das Ergebnis (Steigung, Achsenabschnitt, R², Stützpunkte) wird beim ersten Bedarf berechnet und zwischengespeichert (`get_uv_vis_calibration`, Z. 2472); die Gleichung zeigt die GUI an (`set_uv_vis_equation`, Z. 2482).

### 8.2 UV-Vis-CSV einlesen (`read_uv_vis_csv`, Z. 357–450)

- Probiert mehrere Kodierungen und Trenner/Dezimal-Kombinationen (`,`/`.`, `;`/`,`, Tab/`.` …).
- Sucht eine Kopfzeile mit „Time“ und „Abs“; sonst werden die ersten beiden Spalten genommen.
- Ergebnis: `Time_min` und `Absorbance`.

### 8.3 Absorbanz → NADH (`calculate_uv_vis_concentrations`, Z. 453–470)

```python
concentrations = calibration["slope"] * uv_vis_data["Absorbance"] + calibration["intercept"]
low, high = np.min(calibration['concentrations']), np.max(calibration['concentrations'])
valid = np.isfinite(concentrations) & concentrations.between(low, high)     # Z. 465
uv_vis_data['NADH_raw_mmol_L'] = concentrations
uv_vis_data['Outside_calibration_range'] = ~valid
uv_vis_data["NADH_concentration_mmol_L"] = concentrations.where(valid)
```

- Werte außerhalb von `[min, max]` der **Kalibrierkonzentrationen** werden zu `NaN`, der Rohwert bleibt in `NADH_raw_mmol_L`.

### 8.4 NAD⁺-Umsatz in Prozent (Z. 3017–3020)

```python
plot_results[NADH_CONVERSION_PERCENT_COLUMN] = 100.0 * plot_results["NADH_concentration_mmol_L"]
```

> ⚠️ **Wichtige Annahme:** Der Umsatz in % ist einfach `100 × c(NADH) [mmol/L]`. Das entspricht einer **Ausgangskonzentration von 1 mmol/L** – ein anderer Wert ist im Code **nicht einstellbar**. Bei anderer Startkonzentration muss der Faktor `100.0` angepasst werden (z. B. `100.0 / c0`).

### 8.5 Darstellung

- Mehrere Dateien werden zusammengeführt (`Source_file`, `Source_path`, `File_index`).
- Optionen: Plottitel, „Show every n-th point“, Y-Achsen-Intervall (`NADH_Y_TICK_INTERVAL_OPTIONS`; Skalierung in `scale_nadh_axes`, Z. 2554), gleitender Mittelwert (Z. 3077), Legendenposition, Kurvennamen (`uv_vis_curve_label`, Z. 2607).
- Der gleitende Mittelwert ist **reine Darstellung**:

```python
raw.rolling(window_size, min_periods=1).mean().where(raw.notna())
```

  `min_periods=1` erlaubt Mittelung auch am Rand; `.where(raw.notna())` stellt sicher, dass verworfene Zeitpunkte **nicht** durch Nachbarwerte aufgefüllt werden.

---

## 9. Schritt 5 – Darstellung, Tabelle, Plots

Im Raman-Fenster (`build_gui`, Z. 1293–2130):

- **Zwei Plots:** Spektren und Konzentrationsverlauf (`update_plot`, Z. 4348).
- **Schalter** (`update_analyte_toggles`, Z. 3384): Analyten ein-/ausblenden (Z. 3500), Messpunkte, **gleitender Mittelwert** (Fenster ≥ 2, `rolling_average_window`, Z. 3574), **±1 SD** (Z. 3625).
- **„Every n-th Raman point“:** dünnt die Darstellung aus (nur Anzeige).
- **Legende:** getrennte Positionswahl für Spektren- und Konzentrationsplot (Z. 3327 / 3377).
- **Plottitel:** frei wählbar oder ausblendbar (Z. 3303).
- **Tabelle:** sortierbar per Spaltenklick (`sort_table_by_column`, Z. 4232).

**Intervall nachträglich ändern:** Eine Änderung des Messintervalls (`on_measurement_interval_changed`, Z. 3884; entprellt über `schedule_interval_update`, Z. 3877) berechnet **nur die Zeitachse neu – nicht die PLS-Vorhersage**.

**Moving Average (Raman, ab Z. 4608):** Der Kommentar im Code hält fest, dass er ein reines Anzeigemittel ist und Vorhersagen nicht verändert; verworfene Zeitpunkte werden auch hier nicht mit Nachbarwerten gefüllt (Z. 4617).

---

## 10. Schritt 6 – Kombinierter Plot und Plateau-Analyse

### 10.1 Kombinierter Plot (`open_combined_plot`, Z. 4970–5177)

Voraussetzungen (Z. 4971–4975): Raman-Ergebnisse **und** UV-Vis-Ergebnisse geladen, und `Time_min` enthält gültige Werte; sonst erscheint der Hinweis *„Measurement interval required“*. Die Buttons werden über `refresh_combined_plot_availability` (Z. 4952) aktiviert.

Funktionen:

- Raman und NADH auf **unabhängigen Y-Achsen**.
- **UV-Vis-Zeitversatz** in Minuten (Z. 4983–4989, Standard 0): Der Wert wird bei der UV-Vis-Zeit addiert, um beide Methoden zeitlich anzugleichen.
- Gleitender Mittelwert und „Every n-th point“ **getrennt** für Raman und UV-Vis (Z. 5095 / 5106).
- Buttons **Update** und **Save** (Z. 5159).

Mit `raman_only=True` (Button **Raman plateau analysis**, `open_plateau_plot`, Z. 4966) entfällt die UV-Vis-Seite (leeres UV-DataFrame, Z. 5043–5046).

### 10.2 Plateau-/Extraktionsintervalle (`PlateauAnalysisPanel`, Z. 972)

Zwei Reiter am unteren Rand (Z. 5051–5052):

| Reiter | `kind` | Funktion |
|---|---|---|
| **Plateaus / extractions** | `"plateau"` | Beliebig viele Intervalle (`E1`, `E2`, …, Umbenennung Z. 5040) mit Start/Ende in min |
| **Days** | `"day"` | Ein mittleres Intervall legt Tagesgrenzen fest (gestrichelte Linien, Beschriftung im Plot) |

Funktionen im Panel: `add` (1048), `apply` (1069), `delete` (1087), `draw` (1100), Ziehen der Grenzen mit der Maus (`press` 1149, `motion` 1161, `release` 1174), `update_statistics` (1179), `save` (1194, speichert `Plateau_statistics.csv`).

### 10.3 Statistik (`plateau_statistics`, Z. 946–969)

```python
mask = (np.isfinite(times) & np.isfinite(values)
        & (times >= region["start"]) & (times <= region["end"]))
selected = values[mask]
...
"N": len(selected),
"Mean_mmol_L": float(np.mean(selected)) if len(selected) else np.nan,
"SD_mmol_L": float(np.std(selected, ddof=1)) if len(selected) > 1 else np.nan
```

- Es werden die **ursprünglichen, endlichen Konzentrationen** ausgewertet – **Glättung und Punktausdünnung fließen nicht ein** (Docstring Z. 947).
- Ausgabe je Intervall × Methode (Raman / UV-Vis) × Analyt: `N`, Mittelwert, SD (`ddof=1`).
- Für UV-Vis wird der Zeitversatz addiert: `data["Time_min"] + uv_offset`.

---

## 11. Schritt 7 – Export der Ergebnisse

### Automatisch (nach jeder Raman-Auswertung)

`save_results(auto=True)` (Z. 4795) schreibt **nur Zahlenwerte, keine Grafiken** (`include_plots=not auto`):

```text
<Raman-Ordner>/Raman_PLS_Results/
├── Concentration_analysis.csv       (Trenner ";", Dezimalkomma; 1 Zeile je Probe)
├── Concentration_analysis.json      (alle Spalten, Records-Format)
└── UV_Vis_NADH_concentrations.csv   (nur wenn UV-Vis-Daten geladen sind)
```

- Zielordner: `default_result_dir` (Z. 4709) = `<Raman-Ordner>/Raman_PLS_Results`.
- `write_table_csv` (Z. 4718): Spalten `Time_min` und je Analyt `<Name> [mmol/L]`, sortiert nach `Sample_order`.

```python
concentrations.reset_index().to_csv(csv_path, index=False, sep=";", decimal=",")
```

→ Deutsches Excel-Format (Semikolon, Dezimalkomma).

### Manuell (nur nach Klick)

| Button | Ergebnis | Code |
|---|---|---|
| **Save results** | CSV/JSON **plus** alle Plots | `save_results(auto=False)` Z. 4795 |
| **Save all to folder** | Alles in einen frei gewählten Ordner | `save_all_to_folder` Z. 4857 |
| **Save table CSV** | Nur die Raman-Tabelle | `save_table_csv` Z. 4819 |
| **Save PNG/SVG** | Einzelner Plot | `save_plot_png` Z. 5178 |
| **Save NADH CSV** | UV-Vis-Tabelle | `save_uv_vis_csv` Z. 3236 |
| NADH-Plot speichern | PNG/SVG | `save_nadh_plot_png` Z. 3190 |

Dateinamen der Raman-Plots werden von `plot_filename` (Z. 4885) bzw. `plot_svg_filename` (Z. 4900) erzeugt, gespeichert von `save_plot_files` (Z. 4908). NADH-Plots heißen `NADH_concentration.png/.svg`.

### Bildqualität (`save_figure_high_quality`, Z. 229)

- PNG mit **2400 dpi** (`PNG_EXPORT_DPI`) und `compress_level=1` (schnell, dafür große Dateien); SVG ist auflösungsunabhängig.
- Legenden außerhalb der Achsen werden über `bbox_extra_artists` mitexportiert, damit sie nicht abgeschnitten werden.
- Bei Speicher-/Zeitproblemen `PNG_EXPORT_DPI` (Z. 66) senken.

---

## 12. Bedienung der GUI

### Raman-Fenster

1. **Messtyp** wählen (Single / Repeated); bei *Single* **Intervall in Minuten** eintragen.
2. **Ordner** in die Drop-Zone ziehen (`on_drop`, Z. 3933) oder **Select folder** (`choose_folder`, Z. 3909).
3. Ergebnis in Tabelle und Plots prüfen; Darstellung über die Schalter anpassen.
4. Bei Bedarf Messtyp/Intervall ändern: Der Wechsel des Messtyps bei geladenem Ordner löst eine **Neuauswertung** aus (Prüfung in Z. 3886), eine reine Intervalländerung nur die Zeitachsen-Aktualisierung.
5. **Save results / Save all to folder** für Plots.

### UV-Vis-Fenster

1. **Select NADH CSV files** oder Dateien/Ordner hineinziehen.
2. Plot, Tabelle und Kalibriergleichung ansehen; optional Titel, Kurvennamen, Y-Intervall, Mittelwertfenster anpassen.
3. **Save NADH CSV** bzw. PNG/SVG.

### Fensterverwaltung

Beide Fenster sind über das Menü **Windows** wieder einblendbar (Z. 1234–1243). Schließen blendet ein Fenster nur aus (`close_analysis_window`, Z. 1282); erst wenn **beide** geschlossen sind, endet das Programm.

### Zusatzfenster

| Button | Funktion | Code |
|---|---|---|
| **Raman + UV-Vis plot** | beide Methoden, getrennte Y-Achsen | `open_combined_plot` (Z. 4970) |
| **Raman plateau analysis** | Plateau-Analyse nur Raman | `open_plateau_plot` (Z. 4966) |

---

## 13. Das eingebettete Modell im Detail

Aus dem eingebetteten Modellpaket (Version `4.0_K37-K86`, Status `updated_internal_nested_group_cv_only`) ergibt sich:

| Analyt | Vorverarbeitung | Glättung (Fenster/Grad) | Basislinie (Fenster/Grad) | Spektralbereich (`window`) | PLS-Komponenten | Kalibrierbereich |
|---|---|---|---|---|---|---|
| 1-Phenylethanol | `snv` | 31 / 3 | 301 / 2 | `FP_CH` | 5 | 0 – 120 mmol/L |
| Cyclohexanon | `bc` | 11 / 2 | 501 / 2 | `FP_CH` | 6 | 0 – 200 mmol/L |
| Cyclohexanol | `bc` | 11 / 2 | 201 / 2 | `SEL` | 6 | 0 – 200 mmol/L |

- Raman-Achse: **3528 Punkte** von ca. 99,5 bis 3500,4 cm⁻¹.
- Herkunft laut Paket: 72 unabhängige Kalibrieransätze nach Störpunkt-Ausschluss, je Ansatz 5 Raman-CSV-Replikate vor dem Training gemittelt.
- Bewertung: verschachtelte, gruppierte Kreuzvalidierung.
- Das Modell erwartet laut Hinweis im Paket **den Mittelwert mehrerer Rohspektren derselben Probe, vorzugsweise 5 Wiederholungen**.

> Die Tabelle wurde durch Auslesen des eingebetteten Modells erstellt (nicht durch Lesen des Quellcodes). Bei einem Austausch des Modells ändern sich diese Werte.

---

## 14. Anpassen und Erweitern

**Anderes Modell einbetten**
Neue Joblib-Datei zu Base64 kodieren, in `EMBEDDED_MODEL_BASE64` (Z. 62) einfügen und die SHA-256-Prüfsumme der **binären** Datei in `EMBEDDED_MODEL_SHA256` (Z. 63) eintragen. Erwartete Schlüssel im Paket: `x_axis_cm-1`, `pls1_models`, `model_status`, außerdem `analytes` (wird in `analyze_folder` benötigt). Je Analyt in `pls1_models`: `model`, `mask`, `preprocessing`, `preprocess_params`, `calibration_range_mM`.

Beispiel zum Erzeugen der Werte:

```python
import base64, hashlib
data = open("modell.joblib", "rb").read()
print(base64.b64encode(data).decode())      # -> EMBEDDED_MODEL_BASE64
print(hashlib.sha256(data).hexdigest())     # -> EMBEDDED_MODEL_SHA256
```

**Andere UV-Vis-Kalibrierung**
Excel-Datei genauso Base64-kodieren und in `EMBEDDED_UV_CALIBRATION_BASE64` (Z. 64) einsetzen. Das Layout (Konzentrationen in Zeile 5, Absorbanzen in Zeilen 6–14) muss erhalten bleiben oder `iloc[4]` (Z. 270 ff.) und `iloc[5:14, …]` in `load_uv_vis_calibration` angepasst werden.

**Ausgangskonzentration für NAD⁺-Umsatz ändern**
Faktor `100.0` in Z. 3018 ersetzen, z. B. `100.0 / c0` (c0 in mmol/L).

**Anzeigenamen eines Analyten ändern**
Mapping in `analyte_display_name` (Z. 879) erweitern (derzeit `Cyclohexanon` → `Cyclohexanone`).

**Despike-Empfindlichkeit**
Parameter `kernel` und `threshold` in `despike` (Z. 517). Das Training muss die gleichen Einstellungen verwendet haben, sonst weichen Vorhersagen ab.

**Weiteren Vorverarbeitungsmodus ergänzen**
Zweig in `preprocess_with_config` (Z. 594) hinzufügen; Parameter stehen im Modell unter `preprocess_params`.

**PNG-Auflösung**
`PNG_EXPORT_DPI` (Z. 66).

---

## 15. Fehlersuche

| Symptom | Ursache / Lösung |
|---|---|
| `The embedded model data are damaged.` | Base64-Zeile in Z. 62 wurde verändert/abgeschnitten oder die Prüfsumme (Z. 63) passt nicht. Original wiederherstellen. |
| `The selected PLS model is incomplete…` | Im Modellpaket fehlen Pflichtschlüssel (Z. 213–221). |
| Warnungen/Fehler beim Laden des Modells | Falsche scikit-learn-Version; `1.8.0` muss installiert sein (Z. 65). |
| Datei lässt sich schlecht bearbeiten/öffnen | Die Zeilen 62 und 64 sind extrem lang. Zeilenumbruch im Editor ausschalten oder Zeilen einklappen. |
| `CSV file could not be read` | Weniger als 100 gültige numerische Zeilen oder unbekanntes Format (`read_spectrum`, Z. 477). |
| `No CSV files were found in the selected folder.` | Ordner enthält keine `.csv`-Dateien. |
| „Missing measurement type“ | Vor dem Upload Messtyp wählen (Z. 3822). |
| Bei *Single* wird nichts ausgewertet | Messintervall (> 0) fehlt (Prüfung in `get_upload_settings`). |
| Alle Konzentrationen leer (NaN) | Vorhersagen liegen außerhalb des Kalibrierbereichs (auch negative Werte); `Raw_concentration_mmol_L` prüfen. |
| Zeitplots / kombinierter Plot nicht verfügbar | Kein Messintervall eingegeben (`Time_min` = NaN) oder UV-Vis nicht geladen (Z. 4952, 4971–4975). |
| Drag & Drop funktioniert nicht | `tkinterdnd2` nicht ladbar (`DND_AVAILABLE = False`, Z. 198). Stattdessen **Select folder** nutzen. |
| UV-Vis-CSV nicht lesbar | Erwartet werden Spalten *Time (min)* und *Abs* (`read_uv_vis_csv`, Z. 357). |
| PNG-Export langsam/sehr groß | `PNG_EXPORT_DPI` senken oder SVG verwenden. |
| Tkinter fehlt (macOS) | Python von python.org installieren. |
| Startfehler | Das Programm gibt `Startup error:` aus und wirft die Ausnahme erneut (Z. 5254–5263). |

---

## 16. Bekannte Besonderheiten und Hinweise

- **Bereichsfilter statt Clipping:** Im Modellpaket steht der Hinweis, negative Ausgaben würden „in der Anwendung auf 0 mmol/L gekappt“. Im Code geschieht das **nicht**: Außerhalb von `[low, high]` – also auch bei negativen Werten – wird der Ausgabewert `NaN`. Der Rohwert bleibt in `Raw_concentration_mmol_L`. Wer Clipping auf 0 erwartet, muss `predict_sample` (Z. 859–863) anpassen.
- **Zwei verschiedene Kalibrierbereiche:** Raman-Analyten haben je einen eigenen Bereich aus dem Modell (z. B. 0–120 bzw. 0–200 mmol/L); NADH nutzt `[min, max]` der Kalibrierkonzentrationen der Excel-Kalibrierung.
- **NAD⁺-Umsatz** setzt implizit 1 mmol/L Ausgangskonzentration voraus (siehe [8.4](#84-nad-umsatz-in-prozent-z-30173020)).
- **Gleitende Mittelwerte sind Darstellung:** Sie verändern weder Vorhersagen noch Exporttabellen noch die Plateau-Statistik.
- **CSV-Exporte** der Raman-Tabelle und der UV-Vis-Daten nutzen Semikolon und Dezimalkomma; die Plateau-Statistik wird über den Speichern-Dialog des Panels geschrieben (`Plateau_statistics.csv`).
- **Plots werden nie automatisch gespeichert** (`include_plots=not auto`, Z. 4795 ff.).
- **Automatische Paketinstallation:** `ensure_packages` führt `pip install` in der aktiven Umgebung aus und kann scikit-learn auf `1.8.0` ändern.
- **Eine Datei, viel GUI-Code:** Der Großteil der Datei (Z. 1204–5246) ist die Klasse `RamanPLSApp`; `build_gui` allein umfasst rund 840 Zeilen (Z. 1293–2130).

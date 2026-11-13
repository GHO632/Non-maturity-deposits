# Non-Maturity-Deposits: Prognose von Einlagenvolumen

Dieses Projekt untersucht, wie sich das Volumen deutscher **Non-Maturity Deposits (NMD)** – täglich fälliger, nicht vertraglich gebundener Einlagen – mit unterschiedlichen Machine-Learning-Verfahren prognostizieren lässt. Im Mittelpunkt stehen Out-of-Sample-Prognosen, der Vergleich ökonomischer Kovariaten und die Frage, wie stabil die Modellgüte über verschiedene Zeiträume und Zinsregime ist.

> **Hinweis:** Die Ergebnisse sind modell- und datensatzabhängig und stellen weder eine Anlageempfehlung noch eine bankaufsichtliche oder regulatorische Bewertung dar.

## Inhalt

- [Projektziele](#projektziele)
- [Methodik](#methodik)
- [Repository-Aufbau](#repository-aufbau)
- [Voraussetzungen und Installation](#voraussetzungen-und-installation)
- [Schnellstart](#schnellstart)
- [Benchmark-Optionen](#benchmark-optionen)
- [Prognoseplot erstellen](#prognoseplot-erstellen)
- [Daten](#daten)
- [Ergebnisse und Interpretation](#ergebnisse-und-interpretation)
- [Reproduzierbarkeit und bekannte Einschränkungen](#reproduzierbarkeit-und-bekannte-einschränkungen)
- [Lizenz und Quellen](#lizenz-und-quellen)

## Projektziele

Das Repository bietet eine Python-Pipeline, um:

1. Einlagenvolumen aus historischen Daten über einen festen Prognosehorizont vorherzusagen.
2. klassische und neuronale Modelle mit naiven Referenzprognosen zu vergleichen.
3. den Einfluss verschiedener Kombinationen makroökonomischer und finanzieller Kovariaten zu untersuchen.
4. Ergebnisse mit Einzel- und Rolling-Origin-Evaluation sowie Grafiken und CSV-Dateien auszuwerten.

## Methodik

### Modelle

| Modell | Kurzbeschreibung |
| --- | --- |
| `naive_rw` | Random-Walk-Referenz: schreibt die letzte beobachtete Volumenänderung fort. |
| `naive_hold` | Naive-Hold-Referenz: hält das zuletzt beobachtete Volumenniveau konstant. |
| `linreg` | Lineares Sequenzmodell; beim Tuning wird Ridge-Regression verwendet. |
| `ffn` | Feedforward Neural Network auf historischen Sequenzfenstern. |
| `xgboost` | Gradient-Boosted Trees auf Sequenzmerkmalen. |
| `tft` | Temporal Fusion Transformer für Zeitreihenprognosen. |

### Kovariaten

Die gemeinsame Pipeline (`common.py`) verwendet standardmäßig folgende Variablen:

- `€STR` und `Einlagezinssatz`
- `GPRC_DEU` (geopolitischer Risikoindex)
- `MoM Inflation` (Inflation)
- `DAX`
- `10Y Bond` (zehnjährige Anleihe)

Für Sequenzmodelle lassen sich zusätzlich verzögerte Kovariatenwerte für 1, 5 und 20 Handelstage aktivieren.

### Prognose und Evaluation

Die zentrale Pipeline modelliert die erste Differenz des Einlagenvolumens (`Diff_Volume`) und rekonstruiert daraus das Volumenniveau. Das Standard-Trainingsende ist in `common.py` auf den **30. November 2023**, der Standard-Prognosehorizont auf **100 Datenbeobachtungen** und das Sequenzgedächtnis auf **100 Beobachtungen** gesetzt.

Bewertet werden MAE, RMSE und R² sowohl auf den Differenzen als auch auf dem Volumen. Die Rolling-Origin-Evaluation wiederholt den Test über mehrere Trainingsendpunkte. Optional können Ergebnisse nach Zinsregimen annotiert und Modellvergleiche mit einem Diebold-Mariano-Test ergänzt werden.

## Repository-Aufbau

```text
.
├── common.py                 # Gemeinsame Daten-, Modell- und Evaluationspipeline
├── covariate_benchmark.py    # CLI für Modell-/Kovariaten-Benchmarks
├── benchmark_viz.py          # Benchmark- und Rolling-Origin-Grafiken
├── forecast_plot.py          # CLI zum Erstellen eines Modell-Prognoseplots
├── treesearch_plot.py        # Gemeinsame Plot-Funktionen
├── linreg.py                 # Linear-Regression-Plot-Einstiegspunkt
├── ffn.py                    # FFN-Plot-Einstiegspunkt
├── xgboost1.py               # XGBoost-Plot-Einstiegspunkt
├── TFT.py                    # Separates TFT-Beispielskript
├── Data all variables.xlsx   # Eingabedatensatz der gemeinsamen Pipeline
├── Preprocessing/            # Datenaufbereitung und zugehörige Dateien
└── Results/                  # Gespeicherte Resultate und Visualisierungen
```

## Voraussetzungen und Installation

Benötigt wird Python 3.10 oder neuer. Die Pipeline verwendet unter anderem pandas, NumPy, scikit-learn, PyTorch/Lightning, `pytorch-forecasting`, XGBoost und Matplotlib. Für Excel-Dateien wird `openpyxl` benötigt. Einige Vorverarbeitungsskripte brauchen zusätzliche Pakete wie SciPy oder yfinance.

Erstelle eine virtuelle Umgebung und installiere die Pakete:

```bash
python -m venv .venv
```

Aktivierung unter macOS/Linux:

```bash
source .venv/bin/activate
```

Aktivierung unter Windows (PowerShell):

```powershell
.venv\Scripts\Activate.ps1
```

Installiere PyTorch passend zu deinem Betriebssystem und gegebenenfalls deiner CUDA-Version entsprechend der offiziellen PyTorch-Installationsanleitung. Danach können die Projektpakete installiert werden:

```bash
python -m pip install --upgrade pip
python -m pip install numpy pandas scikit-learn openpyxl xgboost \
  torch lightning pytorch-forecasting matplotlib seaborn tqdm \
  scipy joblib threadpoolctl yfinance
```

Die exakte Kompatibilität der Paketversionen ist nicht durch eine im Repository enthaltene Lockdatei festgelegt. Für reproduzierbare Läufe sollten Python- und Paketversionen daher in der eigenen Umgebung dokumentiert bzw. fixiert werden.

## Schnellstart

Führe die Befehle aus dem Projektverzeichnis aus, damit die relativen Pfade zu den Excel-Dateien stimmen.

### Benchmark über vordefinierte Kovariaten-Sets

```bash
python covariate_benchmark.py \
  --modelle naive_rw naive_hold linreg xgboost \
  --kovariaten-modus presets \
  --output Results/covariate_benchmark_results.csv \
  --viz all
```

Der Benchmark schreibt Ergebnisse in eine CSV und erstellt mit `--viz all` Visualisierungen. Die Ausgabeordner müssen existieren, bevor ein Ausgabepfad in einem noch nicht vorhandenen Unterordner verwendet wird.

### Rolling-Origin-Auswertung

```bash
python covariate_benchmark.py \
  --modelle naive_rw naive_hold linreg xgboost \
  --kovariaten-modus presets \
  --rolling \
  --rolling-freq QS \
  --rolling-output Results/covariate_benchmark_rolling.csv
```

Rolling-Origin-Benchmarks können je nach Anzahl der Modelle, Kovariaten-Sets und Origins erheblich länger dauern. TFT ist insbesondere rechenintensiv und muss explizit mit `--modelle tft` ausgewählt werden.

### Hilfe zur Kommandozeile

```bash
python covariate_benchmark.py --help
python forecast_plot.py --help
```

## Benchmark-Optionen

`covariate_benchmark.py` unterstützt unter anderem:

| Option | Bedeutung |
| --- | --- |
| `--modelle ...` | Modelle aus `naive_rw`, `naive_hold`, `linreg`, `ffn`, `xgboost`, `tft` auswählen. |
| `--kovariaten-modus presets` | Vordefinierte Kovariaten-Sets verwenden. |
| `--kovariaten-modus singletons` | Jede Kovariate einzeln testen. |
| `--kovariaten-modus pairs` | Alle Kovariaten-Paare testen. |
| `--kovariaten-modus all` | Presets, Einzelvariablen und Paare testen. |
| `--kovariaten-modus full` | Alle nichtleeren Teilmengen der sechs Kovariaten testen; umfangreich. |
| `--rolling` | Rolling-Origin-Auswertung aktivieren. |
| `--rolling-freq MS` | Häufigkeit automatisch erzeugter Origins festlegen, z. B. `MS` (monatlich) oder `QS` (quartalsweise). |
| `--rolling-origins DATUM ...` | Explizite Trainingsendpunkte verwenden. |
| `--lags` | Verzögerte Kovariaten für Sequenzmodelle ergänzen. |
| `--tune` | GridSearchCV-Tuning für Sequenzmodelle einschalten. |
| `--regime` | Bei Rolling-Auswertungen Origins nach der standardmäßigen €STR-Regimeeinteilung annotieren. |
| `--dm-test` | Bei Rolling-Auswertungen einen paarweisen Diebold-Mariano-Testbericht ergänzen. |
| `--viz all` | Heatmap, Boxplot, Scatterplot und Top-5-Tabelle erzeugen. |

Die Modi `all` und besonders `full`, Kombinationen mit `--rolling` sowie Hyperparameter-Tuning multiplizieren die Anzahl der Läufe. Starte zum Prüfen der Umgebung zunächst mit wenigen Modellen und einem kleineren Kovariaten-Modus.

## Prognoseplot erstellen

Beispiel für einen XGBoost-Forecast mit dem vordefinierten Set `nur_makro`:

```bash
python forecast_plot.py \
  --modell xgboost \
  --label nur_makro \
  --simulationen 5 \
  --output Results/forecast_xgboost_nur_makro.png
```

Weitere Beispiele:

```bash
# Lineare Regression mit allen Standard-Kovariaten
python forecast_plot.py --modell linreg --label alle

# XGBoost mit Tuning und verzögerten Kovariaten
python forecast_plot.py --modell xgboost --label nur_zinsen --lags --tune

# TFT mit explizitem Trainingsende
python forecast_plot.py --modell tft --label alle --training-end 2023-11-30
```

Mit `--show` wird der Plot interaktiv angezeigt, anstatt ihn zu speichern. `--kovariaten` erlaubt statt eines vordefinierten Labels eine explizite Variablenliste.

## Daten

Die gemeinsame Pipeline erwartet standardmäßig `Data all variables.xlsx` im Projektverzeichnis. Die Datei muss mindestens die in `common.py` konfigurierten Spalten enthalten:

```text
Datum, Einlagevolumen, €STR, Einlagezinssatz, GPRC_DEU,
MoM Inflation, DAX, 10Y Bond
```

`Datum` muss als Datum interpretierbar sein; die Zeitreihe wird chronologisch sortiert. Die bereitgestellte Excel-Datei ist für die Ausführung der zentralen Pipeline vorgesehen. Skripte in `Preprocessing/` können weitere Rohdaten oder lokale Eingabedateien voraussetzen, die nicht notwendigerweise im Repository enthalten sind. Prüfe deshalb vor deren Ausführung die im jeweiligen Skript konfigurierten Eingabe- und Ausgabepfade.

**Datenschutz und Nutzungsrechte:** Prüfe vor Veröffentlichung oder Weitergabe, ob die enthaltenen Daten öffentlich genutzt und weiterverteilt werden dürfen. Entferne vertrauliche oder nicht freigegebene Daten aus dem Repository.

## Ergebnisse und Interpretation

Der Ordner `Results/` enthält beispielhafte Forecast-Plots und Benchmark-Ausgaben, darunter Vergleiche der Modelle und Kovariaten. Beispielhafte Übersichten:

- [Modellvergleich](Results/Comparison.jpeg)
- [MAE nach Kovariate und Zeitraum](Results/MAE%20per%20covariate%20per%20time.jpeg)
- [Performance und Stabilität](Results/Performance%20vs.%20Stability.jpeg)
- [Beste Kovariaten-Sets je Modell](Results/best%20set%20per%20model.jpeg)
- [Rolling-Origin-Zusammenfassung](Results/covariate_benchmark_rolling_summary.png)

Ein einzelnes Testfenster ist keine ausreichende Grundlage für Aussagen zur allgemeinen Modellgüte. Bevorzuge Rolling-Origin-Ergebnisse und interpretiere Rangfolgen angesichts der getesteten Modell- und Kovariatenkombinationen als explorativ. Ein niedriger MAE in einem Zeitraum belegt nicht automatisch Robustheit in anderen Markt- oder Zinsphasen.

## Reproduzierbarkeit und bekannte Einschränkungen

- Zentrale Einstellungen wie Zielvariable, Trainingsende, Horizont, Kovariaten und Zufalls-Seed stehen in `common.py`.
- Das Verhalten hängt von den installierten Bibliotheksversionen sowie – bei PyTorch – von CPU/GPU und CUDA ab.
- Tuning, TFT und umfangreiche Rolling-Origin- bzw. Kovariaten-Suchen können viel Rechenzeit und Arbeitsspeicher benötigen.
- Der TFT verwendet im Prognosezeitraum fortgeschriebene letzte bekannte Werte für externe Kovariaten; Kalendermerkmale werden dagegen aus den zukünftigen Datumswerten berechnet. Das ist eine Modellierungsannahme und ersetzt keine tatsächlich bekannten Zukunftswerte.
- `TFT.py` ist ein separates Beispielskript mit eigener Daten- und Spaltenkonfiguration. Es ist nicht identisch mit der zentralen Modellpipeline in `common.py`.
- Vor einer wissenschaftlichen oder produktiven Nutzung sollten Datenaufbereitung, zeitliche Splits, Modellannahmen und Metriken fachlich geprüft werden.

## Lizenz und Quellen

Im Repository ist derzeit keine Lizenzdatei ausgewiesen. Damit sind Nutzungs- und Weiterverwendungsrechte am Code nicht automatisch geklärt. Ergänze eine passende `LICENSE`-Datei, wenn das Projekt öffentlich genutzt oder weiterverwendet werden soll.

Für verwendete externe Datensätze, Indizes und Softwarebibliotheken sollten die jeweiligen Quellen, Versionen und Lizenzbedingungen dokumentiert werden. Insbesondere bei Finanz- und Makrodaten gelten gegebenenfalls separate Nutzungsbedingungen.

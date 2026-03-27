# EEGSynthesizer

> **Versión en español** · [English version below](#english-version)

---

## Tabla de contenidos

1. [Descripción general](#descripción-general)
2. [Motivación y contexto](#motivación-y-contexto)
3. [Estructura del repositorio](#estructura-del-repositorio)
4. [Arquitectura del sintetizador](#arquitectura-del-sintetizador)
5. [Parámetros de diseño](#parámetros-de-diseño)
6. [Criterios de calidad](#criterios-de-calidad)
7. [Validación cruzada con CHB-MIT](#validación-cruzada-con-chb-mit)
8. [Resultados del benchmark interno](#resultados-del-benchmark-interno)
9. [Instalación y uso](#instalación-y-uso)
10. [Reglas de desarrollo](#reglas-de-desarrollo)
11. [Advertencias críticas](#advertencias-críticas)
12. [Publicación objetivo](#publicación-objetivo)
13. [Licencia](#licencia)

---

## Descripción general

**EEGSynthesizer v3** es un sintetizador estocástico de señales EEG de cuero cabelludo con morfología ictal fisiológicamente coherente. Genera datasets sintéticos de pacientes virtuales para entrenar modelos de detección de epilepsia en escenarios donde los datos reales etiquetados son escasos, costosos o éticamente restringidos.

El sintetizador **no** tiene como objetivo replicar exactamente el EEG real. Su objetivo es generar señales con morfología ictal *plausible* que permitan entrenar modelos con capacidad de transferencia al dominio clínico real.

---

## Motivación y contexto

El entrenamiento de detectores automáticos de crisis epilépticas requiere grandes volúmenes de EEG ictal etiquetado. La obtención de estos datos en la práctica clínica enfrenta barreras sistemáticas:

- **Escasez:** las crisis son eventos esporádicos; un paciente puede pasar días en monitoreo sin presentar ninguna.
- **Costo:** la adquisición y el etiquetado manual requieren neurólogos especializados y tiempo de UCI de epilepsia.
- **Restricciones éticas y de privacidad:** los datos EEG de pacientes son altamente sensibles; compartirlos públicamente tiene limitaciones legales.

El enfoque de este proyecto es la síntesis estocástica paramétrica: modelar los fenómenos fisiológicos clave (ruido 1/f, patrón spike-wave, correlación espacial inter-canal, artefactos realistas) con distribuciones calibradas sobre datos reales del CHB-MIT Scalp EEG Dataset.

Este trabajo forma parte de una **tesis doctoral en ingeniería biomédica** y apunta a publicación en revista **Q1** del área (Biomedical Signal Processing and Control, Computers in Biology and Medicine o equivalente). El *framing* del paper es **utilidad de entrenamiento**, no fidelidad estadística perfecta.

---

## Estructura del repositorio

```
EEGSynthesizer/
│
├── 1_0_EEGSynthesizer_DATASET.ipynb     # Generación del dataset sintético
├── 1_1_EEGSynthesizer_VALIDATION.ipynb  # Validación cruzada contra CHB-MIT
├── requirements.txt                      # Dependencias Python
├── README.md                             # Este archivo
│
├── dataset_eeg_final/                    # Dataset sintético generado (creado al ejecutar)
│   ├── X_train.npy                       # (125 793, 500, 19) float32 µV
│   ├── y_train.npy                       # (125 793,) int8
│   ├── y_train_soft.npy                  # (125 793,) float32
│   ├── X_val.npy                         # (26 964, 500, 19) float32 µV
│   ├── y_val.npy
│   ├── y_val_soft.npy
│   ├── X_test.npy                        # (26 875, 500, 19) float32 µV
│   ├── y_test.npy
│   ├── y_test_soft.npy
│   ├── cohort_metadata.csv               # Metadata por paciente virtual
│   └── benchmark_modelos.csv            # Resultados del benchmark interno
│
└── dataset_doctorado_final/              # Datos reales y activos de validación (creado al ejecutar)
    ├── chb_mit_cache/                    # Archivos EDF del CHB-MIT (descarga automática)
    ├── X_real.npy                        # Ventanas reales del CHB-MIT
    ├── y_real.npy
    ├── pid_real.npy                      # ID de sujeto por ventana
    └── validation_q1_assets/            # Figuras, CSVs y JSON de validación
        ├── 05_final_summary.json
        └── 05_final_verdict_lines.csv
```

---

## Arquitectura del sintetizador

### Notebook 1 — Generación (`1_0_EEGSynthesizer_DATASET.ipynb`)

| Bloque | Función |
|--------|---------|
| **0** | Limpieza de archivos previos |
| **1** | Parámetros de laboratorio ← único bloque editable |
| **2** | Generadores de señal puros: `colored_noise`, `create_spike_wave_pattern`, `create_rhythmic_confuser` |
| **3** | Clase `EEGSynthesizer` + pipeline de generación de 3 000 pacientes |
| **4** | Verificación de criterios de calidad (árbitro de validez del dataset) |
| **5** | Visualizaciones en tiempo, frecuencia y PSD |
| **6** | Tabla de amplitudes por canal |
| **7** | Benchmark interno con modelos clásicos (diagnóstico de fuente del AUC) |
| **8** | Fingerprint y resumen final |

### Notebook 2 — Validación (`1_1_EEGSynthesizer_VALIDATION.ipynb`)

| Bloque | Función |
|--------|---------|
| **R0** | Configuración y descarga automática del corpus CHB-MIT |
| **R1** | Construcción de `X_real.npy` / `y_real.npy` desde archivos EDF |
| **2** | Escala a µV + helpers de muestreo balanceado |
| **3** | Extracción de features compactas (13 métricas por ventana) |
| **4** | Subsets base para validación |
| **5** | Separabilidad interna SYN (control de fuga espectral vs amplitud) |
| **6** | Fidelidad estadística SYN↔REAL (desplazamiento por IQR) |
| **7** | Utilidad de transferencia: R2R, TSTR, TRTS, Augmentación |
| **8** | Privacidad y memorización |
| **9** | Diversidad inter-paciente |
| **10** | Síntesis ejecutiva y JSON de hallazgos |

---

## Parámetros de diseño

### Configuración global

| Parámetro | Valor | Descripción |
|-----------|-------|-------------|
| `N_PATIENTS` | 3 000 | Pacientes virtuales totales |
| `DURATION` | 120 s | Duración de señal por paciente |
| `FS` | 250 Hz | Frecuencia de muestreo |
| `WIN_SEC` | 2.0 s | Tamaño de ventana |
| `STEP_SEC` | 2.0 s | Paso entre ventanas (sin solapamiento) |
| `WIN_PTS` | 500 | Puntos por ventana |
| `N_CHANNELS` | 19 | Montaje 10-20: Fp1, Fp2, F7, F3, Fz, F4, F8, T3, C3, Cz, C4, T4, T5, P3, Pz, P4, T6, O1, O2 |
| `BIN_THR` | 0.50 | Umbral de fracción ictal para etiquetar ventana como ictal |
| `SEIZURE_PROB` | 0.50 | Fracción de pacientes con crisis (balance de cohorte) |
| `SEIZURE_TYPES` | `generalized_absence`, `focal_temporal` | Fenotipos epilépticos |

### Fondo interictal (ruido 1/f)

| Parámetro | Valor |
|-----------|-------|
| `PATIENT_BETA_MU` | 2.0 |
| `PATIENT_BETA_SIG` | 0.40 |
| `PATIENT_BETA_MIN / MAX` | 1.2 / 3.0 |
| `PATIENT_BASE_AMP_MU` | 45 µV |
| `PATIENT_BASE_AMP_MIN / MAX` | 20 µV / 120 µV |
| `PATIENT_WHITE_NOISE_FRAC` | 0.06 |

### Actividad ictal

| Parámetro | Valor | Justificación clínica |
|-----------|-------|-----------------------|
| `SEIZ_AMP_MIN` | 150 µV | Límite inferior del rango clínico real |
| `SEIZ_AMP_MAX` | 600 µV | Límite superior del rango clínico real |
| `SEIZ_LP_HZ` | 6 Hz | Filtro del spike-wave (no bajar de 14 Hz sin ajustar morfología) |
| `SEIZ_FREQ_MU` | 3.0 Hz | Frecuencia central del spike-wave (ref. clínica: 2.4–4.2 Hz) |
| `SEIZ_FREQ_MIN / MAX` | 2.4 Hz / 4.2 Hz | Rango clínico ictal |
| `SEIZ_HIGH_AMP_PROB` | 0.60 | Probabilidad de crisis de alta amplitud |
| `SEIZ_HIGH_AMP_FACTOR` | (2.2, 3.0)× | Factor multiplicador para crisis de alta amplitud |

### Correlación espacial inter-canal

| Parámetro | Valor |
|-----------|-------|
| `COMMON_FIELD_MIX_MIN` | 0.18 |
| `COMMON_FIELD_MIX_MAX` | 0.42 |

### Artefactos y confusores

| Componente | Descripción |
|------------|-------------|
| Ritmo alfa | 8–12 Hz, waxing-waning, canales occipitales/parietales |
| Parpadeo | Fp1/Fp2 con propagación frontal reducida |
| EMG | Canales temporales/frontales, filtrado paso-alto >30 Hz |
| Dropout | Canal con atenuación ×0.05–0.15, probabilidad 5% |
| Confusores (RMTD, POSTS, Wicket, SubSW) | Bursts no-ictales que simulan morfología ictal inespecífica |

### Splits del dataset

| Split | Pacientes | Ventanas | % ictal |
|-------|-----------|----------|---------|
| TRAIN | 2 100 (70%) | 125 793 | ~7.8% |
| VAL | 450 (15%) | 26 964 | ~7.8% |
| TEST | 450 (15%) | 26 875 | ~7.6% |

> El ~7–8% de ventanas ictales es **correcto por diseño**: 50% de pacientes tienen crisis, pero cada crisis ocupa solo una fracción de los 120 s, y el muestreo balanceado toma el 15–30% de ventanas ictales del total disponible por paciente epiléptico.

### Formato de salida

```
X: (N, T, C) = (N, 500, 19) float32 en µV raw
y: (N,)       int8    {0=interictal, 1=ictal}
y_soft: (N,)  float32  fracción ictal de la ventana [0.0, 1.0]
```

> **Importante:** las señales se guardan en µV **sin normalizar**. El z-score se aplica en el dataloader del modelo:
> ```python
> win = (win - win.mean(axis=0)) / (win.std(axis=0) + 1e-8)
> ```

---

## Criterios de calidad

El Bloque 4 es el **árbitro de calidad**. Todos los criterios deben cumplirse simultáneamente. **Nunca modificar el Bloque 4 para que pasen los criterios; solo se modifica el Bloque 1.**

| Criterio | Condición | Referencia real (CHB-MIT) |
|----------|-----------|--------------------------|
| **C1a** PTP sano | 20–400 µV | ~140 µV |
| **C1b** PTP ictal | 50–800 µV | ~440 µV |
| **C2** Ratio espectral | Ratio 2–6 Hz / 6–20 Hz ictal > 2.0 **Y** ictal > sano | Sano ~1.2 / Ictal ~6.0 |
| **C3** Beta espectral | Mediana entre −1.8 y −3.2 (con z-score previo al Welch) | ~−2.5 |
| **C4** Entropía espectral | Std > 0.05 | — |
| **C5** Correlación inter-canal | Media > 0.18 | ~0.30 |

> **Nota sobre C3:** `est_beta()` debe aplicar z-score a la señal antes de calcular Welch. Sin z-score, la potencia en µV² aplana el fit log-log y produce valores incorrectos (~−0.58 en lugar de ~−2.5).

---

## Validación cruzada con CHB-MIT

### Corpus real utilizado

| Sujeto | Archivos EDF | Archivos con crisis | Segundos ictales |
|--------|-------------|---------------------|------------------|
| chb01 | 5 | 3 | ~108 s |
| chb02 | 4 | 2 | ~91 s |
| chb03 | 4 | 3 | ~226 s |
| chb05 | 6 | 4 | ~459 s |
| chb06 | 5 | 3 | ~42 s |
| **Total** | **24** | **15** | **~926 s** |

El split usa **leave-one-subject-out** (`GroupShuffleSplit`) porque hay múltiples sujetos. Los datos reales se descargan automáticamente desde PhysioNet al ejecutar el Bloque R0.

### Protocolos de validación

| Protocolo | Descripción |
|-----------|-------------|
| **Separabilidad interna** | Controla fuga espectral vs amplitud dentro del dataset SYN |
| **R2R** (Real→Real) | Baseline: entrena y evalúa en datos reales |
| **TSTR** (Synthetic→Real) | Entrena solo en sintético, evalúa en real |
| **TRTS** (Real→Synthetic) | Entrena en real, evalúa en sintético |
| **Augmentación** (Real+Syn→Real) | Combinación de dominios |
| **Fidelidad estadística** | Desplazamiento mediana/IQR por feature |
| **Privacidad** | Fracción de ventanas sintéticas memorizadas del real |
| **Diversidad** | Ratio de variabilidad inter-paciente SYN vs REAL |

### Features evaluadas (13 por ventana)

| Feature | Descripción |
|---------|-------------|
| `ptp_med`, `ptp_p95` | Peak-to-peak mediana y percentil 95 |
| `std_med`, `std_p95` | Desviación estándar mediana y percentil 95 |
| `bp1_4`, `bp4_8`, `bp8_13`, `bp13_30` | Potencia de banda delta, theta, alfa, beta |
| `ratio_2_6__6_20` | Ratio banda 2–6 Hz / 6–20 Hz (firma ictal) |
| `beta` | Pendiente espectral 1/f |
| `Hspec` | Entropía espectral normalizada |
| `corr_abs_mean`, `corr_abs_p95` | Correlación inter-canal absoluta |

### Resultados esperados por métrica

| Métrica | Umbral | Interpretación |
|---------|--------|----------------|
| `separability_spectral_features_auroc` | > 0.85 | Confirma firma espectral del spike-wave |
| `separability_amplitude_only_auroc` | Cualquier valor | Diferencia fisiológica esperada; se neutraliza con z-score |
| `transfer_real_to_real_auroc` (R2R) | Baseline 0.65–0.99 | Depende del número y homogeneidad de sujetos |
| `transfer_synthetic_to_real_auroc` (TSTR) | 0.45–0.85 | Resultado exploratorio con 5 sujetos heterogéneos |
| `transfer_real_to_synthetic_auroc` (TRTS) | > 0.70 | Confirma que el espacio sintético es alcanzable |
| `fidelity_median_displacement_over_iqr` | < 1.5 (objetivo < 1.0) | Fidelidad estadística |
| `privacy_fraction_memorized` | < 0.01 | Sin memorización de pacientes reales |
| `diversity_internal_vs_real_ratio` | > 0.40 | Diversidad inter-paciente suficiente |

### Interpretación correcta del TSTR multi-sujeto

El sintetizador no está diseñado para cubrir la variabilidad inter-sujeto del CHB-MIT. Un TSTR bajo con evaluación multi-sujeto **no invalida** el sintetizador: refleja que el problema de generalización inter-sujeto en epilepsia es abierto incluso con datos reales. El claim del paper es **fidelidad morfológica y utilidad para preentrenar**, no generalización inter-sujeto perfecta.

---

## Resultados del benchmark interno

Los modelos clásicos del Bloque 7 del dataset sirven como diagnóstico de la **fuente del AUC**:

| Diagnóstico | Condición | Significado |
|-------------|-----------|-------------|
| AUC amplitud sola (std+rms post z-score) | **< 0.70** | El z-score elimina el atajo de escala |
| AUC morfología sola (bandpowers relativos) | **≥ 0.75** | El modelo aprende la firma ictal |
| **Dataset válido para Q1** | Ambas condiciones simultáneas | AUC alto por morfología, no por amplitud |

### Separabilidad interna SYN (Bloque 5 del validador)

| Modelo | Accuracy | F1 | ROC-AUC | PR-AUC |
|--------|----------|----|---------|--------|
| All_RF | 0.997 | 0.997 | 0.9995 | 0.9989 |
| All_LogReg | 0.995 | 0.995 | 0.9998 | 0.9997 |
| Spec_LogReg | 0.994 | 0.994 | 0.9995 | 0.9994 |
| Amp_LogReg | 0.963 | 0.962 | 0.9947 | 0.9943 |

> La brecha entre `Spec_LogReg` y `Amp_LogReg` confirma que la firma espectral domina la separación, no la amplitud bruta.

---

## Instalación y uso

### Requisitos

- Python ≥ 3.10 (probado en 3.13.7)
- Windows / Linux / macOS

### Instalación

```bash
git clone https://github.com/<usuario>/EEGSynthesizer.git
cd EEGSynthesizer
python -m venv .venv

# Windows
.venv\Scripts\activate

# Linux / macOS
source .venv/bin/activate

pip install -r requirements.txt
```

### Ejecución

**Paso 1 — Generar el dataset sintético:**

```bash
jupyter notebook 1_0_EEGSynthesizer_DATASET.ipynb
```

Ejecutar todos los bloques en orden. El dataset se guarda en `dataset_eeg_final/`. Tiempo estimado: ~15–25 minutos en CPU estándar.

**Paso 2 — Validar contra CHB-MIT:**

```bash
jupyter notebook 1_1_EEGSynthesizer_VALIDATION.ipynb
```

Ejecutar todos los bloques en orden. El Bloque R0 descarga automáticamente los 24 archivos EDF del CHB-MIT desde PhysioNet (~1.7 GB). Los activos de validación se guardan en `dataset_doctorado_final/validation_q1_assets/`.

### Uso del dataset en tu dataloader

```python
import numpy as np

X = np.load('dataset_eeg_final/X_train.npy', mmap_mode='r')
y = np.load('dataset_eeg_final/y_train.npy')

# Normalización por ventana (aplicar en el dataloader, no en el dataset)
def normalize_window(win):
    # win: (T, C) en µV
    return (win - win.mean(axis=0)) / (win.std(axis=0) + 1e-8)
```

> **Advertencia de mmap en Windows:** cualquier slice de un array cargado con `mmap_mode='r'` requiere `.copy()` antes de operaciones in-place:
> ```python
> segment = X[i].copy()   # correcto
> segment -= segment.mean()
> ```

---

## Reglas de desarrollo

1. **El Bloque 2 es inmutable.** Los generadores `colored_noise`, `create_spike_wave_pattern` y `create_rhythmic_confuser` están validados. No modificar.

2. **El Bloque 4 es el árbitro.** Si un criterio falla, el primer diagnóstico es siempre revisar si es un **bug de medición** (e.g., `est_beta()` sin z-score previo) antes de cambiar parámetros del Bloque 1.

3. **`est_beta()` siempre con z-score previo.** Sin z-score la potencia en µV² aplana el fit log-log: valor incorrecto ~−0.58 vs correcto ~−2.5.

4. **Slices de mmap requieren `.copy()`.** En Windows, `s -= s.mean()` falla sin `.copy()` sobre arrays mapeados en memoria.

5. **Guardar en Windows con `os.path.abspath()`** y borrar el archivo previo antes de `np.save()` para evitar `OSError 22` por lock de mmap.

6. **Nunca modificar el Bloque 4** para que pasen los criterios. Solo se edita el Bloque 1.

7. **El warning de matplotlib** `labels → tick_labels` en `boxplot()` es cosmético. Corregir con `tick_labels=` en lugar de `labels=`.

8. **Las métricas del validador** se interpretan siempre en el contexto del objetivo del sintetizador: utilidad morfológica para entrenamiento, no fidelidad distribucional perfecta.

---

## Advertencias críticas

### Cambios que **nunca** debes hacer

| Cambio prohibido | Consecuencia |
|-----------------|--------------|
| `SEIZ_LP_HZ < 14` | Destruye la morfología del spike-wave, lo convierte en onda sinusoidal pura |
| `SEIZ_AMP_MIN > 600e-6` | Fuera del rango clínico real; indefendible en paper revisado por pares |
| `SEIZURE_PROB ≠ 0.50` | Rompe el balance de cohorte |
| Z-score al **guardar** el dataset | Las comparaciones con CHB-MIT requieren escala µV |
| Modificar el Bloque 2 | Invalida toda la cadena de validación cruzada |
| Modificar el Bloque 4 para que pasen los criterios | Equivale a falsificar el control de calidad |

---

## Publicación objetivo

Este proyecto apunta a publicación en una revista **Q1** del área biomédica:

- Biomedical Signal Processing and Control (Elsevier)
- Computers in Biology and Medicine (Elsevier)
- O equivalente en cuartil Q1 (SCImago / JCR)

El argumento central del paper es la **utilidad de entrenamiento**: los datos sintéticos generados permiten preentrenar detectores de crisis que luego se afinen (fine-tune) con datos reales escasos, reduciendo significativamente el volumen de anotaciones clínicas necesarias.

El argumento **no** es fidelidad estadística perfecta: se declara explícitamente que las diferencias de escala absoluta entre dominios SYN y REAL son una limitación conocida y que el z-score en entrenamiento las neutraliza funcionalmente.

---

## Referencia del dataset real

Shoeb, A. H. (2009). *Application of machine learning to epileptic seizure onset detection and treatment*. PhD Thesis, Massachusetts Institute of Technology.

Goldberger, A. L. et al. (2000). PhysioBank, PhysioToolkit, and PhysioNet: Components of a new research resource for complex physiologic signals. *Circulation*, 101(23), e215–e220.

Dataset: [CHB-MIT Scalp EEG Database v1.0.0](https://physionet.org/content/chbmit/1.0.0/) — PhysioNet.

---

## Licencia

Este repositorio se distribuye bajo licencia **CC-BY-4.0** (Creative Commons Attribution 4.0 International). Puedes usar, copiar, modificar y redistribuir el contenido libremente, incluyendo para fines comerciales, siempre que se cite al autor original. Los datos reales del CHB-MIT están sujetos a las condiciones de uso de PhysioNet (acceso libre con registro). Los datos sintéticos generados por este proyecto no contienen información de pacientes reales.

---
---

# English Version

> **English version** · [Versión en español arriba](#eegSynthesizer)

---

## Table of Contents

1. [Overview](#overview)
2. [Motivation and Context](#motivation-and-context)
3. [Repository Structure](#repository-structure)
4. [Synthesizer Architecture](#synthesizer-architecture)
5. [Design Parameters](#design-parameters)
6. [Quality Criteria](#quality-criteria)
7. [Cross-Validation Against CHB-MIT](#cross-validation-against-chb-mit)
8. [Internal Benchmark Results](#internal-benchmark-results)
9. [Installation and Usage](#installation-and-usage)
10. [Development Rules](#development-rules)
11. [Critical Warnings](#critical-warnings)
12. [Target Publication](#target-publication)
13. [License](#license-1)

---

## Overview

**EEGSynthesizer v3** is a stochastic synthesizer of scalp EEG signals with physiologically coherent ictal morphology. It generates synthetic datasets of virtual patients for training epileptic seizure detection models in scenarios where real labeled data is scarce, costly, or ethically restricted.

The synthesizer does **not** aim to perfectly replicate real EEG. Its goal is to generate signals with *plausible* ictal morphology that enable training models transferable to the real clinical domain.

---

## Motivation and Context

Training automatic seizure detectors requires large volumes of labeled ictal EEG. Obtaining such data in clinical practice faces systematic barriers:

- **Scarcity:** seizures are sporadic events; a patient may spend days under monitoring without presenting any.
- **Cost:** acquisition and manual labeling require specialist neurologists and epilepsy ICU time.
- **Ethical and privacy restrictions:** patient EEG data is highly sensitive; sharing it publicly has legal limitations.

This project's approach is parametric stochastic synthesis: modeling key physiological phenomena (1/f noise, spike-wave pattern, inter-channel spatial correlation, realistic artifacts) with distributions calibrated on real data from the CHB-MIT Scalp EEG Dataset.

This work is part of a **doctoral thesis in biomedical engineering** and targets publication in a **Q1 journal** (Biomedical Signal Processing and Control, Computers in Biology and Medicine, or equivalent). The paper's framing is **training utility**, not perfect statistical fidelity.

---

## Repository Structure

```
EEGSynthesizer/
│
├── 1_0_EEGSynthesizer_DATASET.ipynb     # Synthetic dataset generation
├── 1_1_EEGSynthesizer_VALIDATION.ipynb  # Cross-validation against CHB-MIT
├── requirements.txt                      # Python dependencies
├── README.md                             # This file
│
├── dataset_eeg_final/                    # Generated synthetic dataset (created at runtime)
│   ├── X_train.npy                       # (125,793, 500, 19) float32 µV
│   ├── y_train.npy                       # (125,793,) int8
│   ├── y_train_soft.npy                  # (125,793,) float32
│   ├── X_val.npy                         # (26,964, 500, 19) float32 µV
│   ├── y_val.npy
│   ├── y_val_soft.npy
│   ├── X_test.npy                        # (26,875, 500, 19) float32 µV
│   ├── y_test.npy
│   ├── y_test_soft.npy
│   ├── cohort_metadata.csv               # Per-virtual-patient metadata
│   └── benchmark_modelos.csv            # Internal benchmark results
│
└── dataset_doctorado_final/              # Real data and validation assets (created at runtime)
    ├── chb_mit_cache/                    # CHB-MIT EDF files (auto-downloaded)
    ├── X_real.npy                        # CHB-MIT windowed EEG
    ├── y_real.npy
    ├── pid_real.npy                      # Subject ID per window
    └── validation_q1_assets/            # Figures, CSVs and validation JSON
        ├── 05_final_summary.json
        └── 05_final_verdict_lines.csv
```

---

## Synthesizer Architecture

### Notebook 1 — Generation (`1_0_EEGSynthesizer_DATASET.ipynb`)

| Block | Function |
|-------|----------|
| **0** | Cleanup of previous output files |
| **1** | Laboratory parameters ← only editable block |
| **2** | Pure signal generators: `colored_noise`, `create_spike_wave_pattern`, `create_rhythmic_confuser` |
| **3** | `EEGSynthesizer` class + 3,000-patient generation pipeline |
| **4** | Quality criteria verification (dataset validity arbiter) |
| **5** | Visualizations: time domain, frequency, PSD |
| **6** | Per-channel amplitude table |
| **7** | Internal benchmark with classical models (AUC source diagnosis) |
| **8** | Fingerprint and final summary |

### Notebook 2 — Validation (`1_1_EEGSynthesizer_VALIDATION.ipynb`)

| Block | Function |
|-------|----------|
| **R0** | Configuration and automatic CHB-MIT corpus download |
| **R1** | Construction of `X_real.npy` / `y_real.npy` from EDF files |
| **2** | µV scaling + balanced sampling helpers |
| **3** | Compact feature extraction (13 metrics per window) |
| **4** | Base subsets for validation |
| **5** | Internal SYN separability (spectral leakage vs amplitude control) |
| **6** | Statistical fidelity SYN↔REAL (displacement per IQR) |
| **7** | Transfer utility: R2R, TSTR, TRTS, Augmentation |
| **8** | Privacy and memorization |
| **9** | Inter-patient diversity |
| **10** | Executive synthesis and findings JSON |

---

## Design Parameters

### Global Configuration

| Parameter | Value | Description |
|-----------|-------|-------------|
| `N_PATIENTS` | 3,000 | Total virtual patients |
| `DURATION` | 120 s | Signal duration per patient |
| `FS` | 250 Hz | Sampling frequency |
| `WIN_SEC` | 2.0 s | Window size |
| `STEP_SEC` | 2.0 s | Step between windows (no overlap) |
| `WIN_PTS` | 500 | Points per window |
| `N_CHANNELS` | 19 | 10-20 montage: Fp1, Fp2, F7, F3, Fz, F4, F8, T3, C3, Cz, C4, T4, T5, P3, Pz, P4, T6, O1, O2 |
| `BIN_THR` | 0.50 | Ictal fraction threshold for window labeling |
| `SEIZURE_PROB` | 0.50 | Fraction of patients with seizures (cohort balance) |
| `SEIZURE_TYPES` | `generalized_absence`, `focal_temporal` | Epileptic phenotypes |

### Interictal Background (1/f noise)

| Parameter | Value |
|-----------|-------|
| `PATIENT_BETA_MU` | 2.0 |
| `PATIENT_BETA_SIG` | 0.40 |
| `PATIENT_BETA_MIN / MAX` | 1.2 / 3.0 |
| `PATIENT_BASE_AMP_MU` | 45 µV |
| `PATIENT_BASE_AMP_MIN / MAX` | 20 µV / 120 µV |
| `PATIENT_WHITE_NOISE_FRAC` | 0.06 |

### Ictal Activity

| Parameter | Value | Clinical Justification |
|-----------|-------|----------------------|
| `SEIZ_AMP_MIN` | 150 µV | Lower bound of real clinical range |
| `SEIZ_AMP_MAX` | 600 µV | Upper bound of real clinical range |
| `SEIZ_LP_HZ` | 6 Hz | Spike-wave low-pass filter (do not lower below 14 Hz without adjusting morphology) |
| `SEIZ_FREQ_MU` | 3.0 Hz | Spike-wave central frequency (clinical ref: 2.4–4.2 Hz) |
| `SEIZ_FREQ_MIN / MAX` | 2.4 Hz / 4.2 Hz | Clinical ictal range |
| `SEIZ_HIGH_AMP_PROB` | 0.60 | Probability of high-amplitude seizure |
| `SEIZ_HIGH_AMP_FACTOR` | (2.2, 3.0)× | Multiplier factor for high-amplitude seizures |

### Inter-Channel Spatial Correlation

| Parameter | Value |
|-----------|-------|
| `COMMON_FIELD_MIX_MIN` | 0.18 |
| `COMMON_FIELD_MIX_MAX` | 0.42 |

### Artifacts and Confounders

| Component | Description |
|-----------|-------------|
| Alpha rhythm | 8–12 Hz, waxing-waning, occipital/parietal channels |
| Eye blink | Fp1/Fp2 with reduced frontal propagation |
| EMG | Temporal/frontal channels, high-pass filtered >30 Hz |
| Channel dropout | Attenuation ×0.05–0.15, 5% probability |
| Confounders (RMTD, POSTS, Wicket, SubSW) | Non-ictal bursts mimicking nonspecific ictal morphology |

### Dataset Splits

| Split | Patients | Windows | % ictal |
|-------|----------|---------|---------|
| TRAIN | 2,100 (70%) | 125,793 | ~7.8% |
| VAL | 450 (15%) | 26,964 | ~7.8% |
| TEST | 450 (15%) | 26,875 | ~7.6% |

> The ~7–8% ictal windows is **correct by design**: 50% of patients have seizures, but each seizure occupies only a fraction of the 120 s, and balanced sampling takes 15–30% of the ictal windows available per epileptic patient.

### Output Format

```
X:      (N, T, C) = (N, 500, 19) float32 in raw µV
y:      (N,)       int8     {0=interictal, 1=ictal}
y_soft: (N,)       float32  ictal fraction of window [0.0, 1.0]
```

> **Important:** signals are saved in µV **without normalization**. Z-score is applied in the model's dataloader:
> ```python
> win = (win - win.mean(axis=0)) / (win.std(axis=0) + 1e-8)
> ```

---

## Quality Criteria

Block 4 is the **quality arbiter**. All criteria must be met simultaneously. **Never modify Block 4 to make criteria pass; only Block 1 is edited.**

| Criterion | Condition | Real reference (CHB-MIT) |
|-----------|-----------|--------------------------|
| **C1a** Healthy PTP | 20–400 µV | ~140 µV |
| **C1b** Ictal PTP | 50–800 µV | ~440 µV |
| **C2** Spectral ratio | 2–6 Hz / 6–20 Hz ratio ictal > 2.0 **AND** ictal > healthy | Healthy ~1.2 / Ictal ~6.0 |
| **C3** Spectral beta | Median between −1.8 and −3.2 (z-score applied before Welch) | ~−2.5 |
| **C4** Spectral entropy | Std > 0.05 | — |
| **C5** Inter-channel correlation | Mean > 0.18 | ~0.30 |

> **Note on C3:** `est_beta()` must apply z-score to the signal before computing Welch. Without z-score, power in µV² flattens the log-log fit, producing incorrect values (~−0.58 instead of ~−2.5).

---

## Cross-Validation Against CHB-MIT

### Real Corpus Used

| Subject | EDF files | Files with seizures | Ictal seconds |
|---------|-----------|---------------------|---------------|
| chb01 | 5 | 3 | ~108 s |
| chb02 | 4 | 2 | ~91 s |
| chb03 | 4 | 3 | ~226 s |
| chb05 | 6 | 4 | ~459 s |
| chb06 | 5 | 3 | ~42 s |
| **Total** | **24** | **15** | **~926 s** |

Split uses **leave-one-subject-out** (`GroupShuffleSplit`) due to multiple subjects. Real data is automatically downloaded from PhysioNet when running Block R0.

### Validation Protocols

| Protocol | Description |
|----------|-------------|
| **Internal separability** | Controls spectral leakage vs amplitude within SYN dataset |
| **R2R** (Real→Real) | Baseline: train and evaluate on real data |
| **TSTR** (Synthetic→Real) | Train only on synthetic, evaluate on real |
| **TRTS** (Real→Synthetic) | Train on real, evaluate on synthetic |
| **Augmentation** (Real+Syn→Real) | Combined domain training |
| **Statistical fidelity** | Median/IQR displacement per feature |
| **Privacy** | Fraction of synthetic windows memorized from real |
| **Diversity** | Ratio of inter-patient variability SYN vs REAL |

### Evaluated Features (13 per window)

| Feature | Description |
|---------|-------------|
| `ptp_med`, `ptp_p95` | Peak-to-peak median and 95th percentile |
| `std_med`, `std_p95` | Standard deviation median and 95th percentile |
| `bp1_4`, `bp4_8`, `bp8_13`, `bp13_30` | Band power: delta, theta, alpha, beta |
| `ratio_2_6__6_20` | 2–6 Hz / 6–20 Hz ratio (ictal signature) |
| `beta` | 1/f spectral slope |
| `Hspec` | Normalized spectral entropy |
| `corr_abs_mean`, `corr_abs_p95` | Absolute inter-channel correlation |

### Expected Results Per Metric

| Metric | Threshold | Interpretation |
|--------|-----------|----------------|
| `separability_spectral_features_auroc` | > 0.85 | Confirms spike-wave spectral signature |
| `separability_amplitude_only_auroc` | Any value | Expected physiological difference; neutralized by z-score |
| `transfer_real_to_real_auroc` (R2R) | Baseline 0.65–0.99 | Depends on number and homogeneity of subjects |
| `transfer_synthetic_to_real_auroc` (TSTR) | 0.45–0.85 | Exploratory result with 5 heterogeneous subjects |
| `transfer_real_to_synthetic_auroc` (TRTS) | > 0.70 | Confirms synthetic space is reachable |
| `fidelity_median_displacement_over_iqr` | < 1.5 (target < 1.0) | Statistical fidelity |
| `privacy_fraction_memorized` | < 0.01 | No real patient memorization |
| `diversity_internal_vs_real_ratio` | > 0.40 | Sufficient inter-patient diversity |

### Correct Interpretation of Multi-Subject TSTR

The synthesizer is not designed to cover the inter-subject variability of the CHB-MIT. A low TSTR with multi-subject evaluation **does not invalidate** the synthesizer: it reflects that the inter-subject generalization problem in epilepsy is open even with real data. The paper's claim is **morphological fidelity and pre-training utility**, not perfect inter-subject generalization.

---

## Internal Benchmark Results

The classical models in Block 7 of the dataset notebook serve as a diagnostic of the **AUC source**:

| Diagnostic | Condition | Meaning |
|------------|-----------|---------|
| Amplitude-only AUC (std+rms post z-score) | **< 0.70** | Z-score eliminates scale shortcut |
| Morphology-only AUC (relative bandpowers) | **≥ 0.75** | Model learns ictal signature |
| **Dataset valid for Q1** | Both conditions simultaneously | High AUC from morphology, not amplitude |

### Internal SYN Separability (Validator Block 5)

| Model | Accuracy | F1 | ROC-AUC | PR-AUC |
|-------|----------|----|---------|--------|
| All_RF | 0.997 | 0.997 | 0.9995 | 0.9989 |
| All_LogReg | 0.995 | 0.995 | 0.9998 | 0.9997 |
| Spec_LogReg | 0.994 | 0.994 | 0.9995 | 0.9994 |
| Amp_LogReg | 0.963 | 0.962 | 0.9947 | 0.9943 |

> The gap between `Spec_LogReg` and `Amp_LogReg` confirms that the spectral signature dominates separation, not raw amplitude.

---

## Installation and Usage

### Requirements

- Python ≥ 3.10 (tested on 3.13.7)
- Windows / Linux / macOS

### Installation

```bash
git clone https://github.com/<username>/EEGSynthesizer.git
cd EEGSynthesizer
python -m venv .venv

# Windows
.venv\Scripts\activate

# Linux / macOS
source .venv/bin/activate

pip install -r requirements.txt
```

### Execution

**Step 1 — Generate synthetic dataset:**

```bash
jupyter notebook 1_0_EEGSynthesizer_DATASET.ipynb
```

Execute all blocks in order. Dataset is saved to `dataset_eeg_final/`. Estimated time: ~15–25 minutes on standard CPU.

**Step 2 — Validate against CHB-MIT:**

```bash
jupyter notebook 1_1_EEGSynthesizer_VALIDATION.ipynb
```

Execute all blocks in order. Block R0 automatically downloads the 24 CHB-MIT EDF files from PhysioNet (~1.7 GB). Validation assets are saved to `dataset_doctorado_final/validation_q1_assets/`.

### Using the Dataset in Your Dataloader

```python
import numpy as np

X = np.load('dataset_eeg_final/X_train.npy', mmap_mode='r')
y = np.load('dataset_eeg_final/y_train.npy')

# Per-window normalization (apply in the dataloader, not in the dataset)
def normalize_window(win):
    # win: (T, C) in µV
    return (win - win.mean(axis=0)) / (win.std(axis=0) + 1e-8)
```

> **mmap warning on Windows:** any slice from an array loaded with `mmap_mode='r'` requires `.copy()` before in-place operations:
> ```python
> segment = X[i].copy()   # correct
> segment -= segment.mean()
> ```

---

## Development Rules

1. **Block 2 is immutable.** The generators `colored_noise`, `create_spike_wave_pattern`, and `create_rhythmic_confuser` are validated. Do not modify.

2. **Block 4 is the arbiter.** If a criterion fails, the first diagnostic is always to check for a **measurement bug** (e.g., `est_beta()` without prior z-score) before changing Block 1 parameters.

3. **`est_beta()` always with prior z-score.** Without z-score, power in µV² flattens the log-log fit: incorrect value ~−0.58 vs correct ~−2.5.

4. **mmap slices require `.copy()`.** On Windows, `s -= s.mean()` fails without `.copy()` on memory-mapped arrays.

5. **Save on Windows with `os.path.abspath()`** and delete the previous file before `np.save()` to avoid `OSError 22` from mmap lock.

6. **Never modify Block 4** to make criteria pass. Only Block 1 is edited.

7. **The matplotlib warning** `labels → tick_labels` in `boxplot()` is cosmetic. Fix with `tick_labels=` instead of `labels=`.

8. **Validator metrics** are always interpreted in the context of the synthesizer's objective: morphological utility for training, not perfect distributional fidelity.

---

## Critical Warnings

### Changes you must **never** make

| Prohibited change | Consequence |
|------------------|-------------|
| `SEIZ_LP_HZ < 14` | Destroys spike-wave morphology, converts it to a pure sinusoidal wave |
| `SEIZ_AMP_MIN > 600e-6` | Outside real clinical range; indefensible in peer-reviewed paper |
| `SEIZURE_PROB ≠ 0.50` | Breaks cohort balance |
| Z-score when **saving** the dataset | CHB-MIT comparisons require µV scale |
| Modifying Block 2 | Invalidates the entire cross-validation chain |
| Modifying Block 4 to pass criteria | Equivalent to falsifying quality control |

---

## Target Publication

This project targets publication in a **Q1 journal** in the biomedical area:

- Biomedical Signal Processing and Control (Elsevier)
- Computers in Biology and Medicine (Elsevier)
- Or Q1 equivalent (SCImago / JCR)

The paper's central argument is **training utility**: the generated synthetic data enables pre-training seizure detectors that are then fine-tuned with scarce real data, significantly reducing the volume of clinical annotations required.

The argument is **not** perfect statistical fidelity: it is explicitly declared that absolute scale differences between SYN and REAL domains are a known limitation, and that z-score at training time functionally neutralizes them.

---

## Real Dataset Reference

Shoeb, A. H. (2009). *Application of machine learning to epileptic seizure onset detection and treatment*. PhD Thesis, Massachusetts Institute of Technology.

Goldberger, A. L. et al. (2000). PhysioBank, PhysioToolkit, and PhysioNet: Components of a new research resource for complex physiologic signals. *Circulation*, 101(23), e215–e220.

Dataset: [CHB-MIT Scalp EEG Database v1.0.0](https://physionet.org/content/chbmit/1.0.0/) — PhysioNet.

---

## License

This repository is distributed under the **CC-BY-4.0** license (Creative Commons Attribution 4.0 International). You are free to use, copy, modify, and redistribute the content, including for commercial purposes, as long as the original author is credited. Real CHB-MIT data is subject to PhysioNet's terms of use (open access with registration). Synthetic data generated by this project contains no information from real patients.

# EEGSynthesizer

Generador estocástico paramétrico de EEG sintético para investigación en detección
de actividad ictal y evaluación de transferencia entre dominios.

EEGSynthesizer genera 3 000 pacientes virtuales con 19 canales del sistema 10–20,
250 Hz y ventanas de 2 segundos. Incluye dos escenarios paramétricos:
`generalized_absence` y `focal_temporal`. Estos nombres describen reglas internas
del generador; no equivalen a diagnósticos ni representan toda la variabilidad del
EEG clínico.

## Qué está respaldado

- Generación determinista por paciente y reproducible bajo el entorno congelado.
- Integridad de archivos mediante manifiestos y SHA-256.
- Amplitud observable final bajo el contrato operacional de 150–600 µV.
- Morfologías sintéticas diferenciadas, evolución focal y topografía controlada.
- Separación de pacientes entre TRAIN, VAL y TEST.
- Transferencia SYN→CHB-MIT bajo el protocolo externo declarado.
- Transferencia SYN→Siena respaldada para el contraste ictal frente a preictal
  limpio en 14 sujetos. El contraste con interictal estricto fue evaluable solo en
  cuatro sujetos y no permite inferencia multicorpus general.
- Descarga oficial, reanudable y verificable de CHB-MIT y Siena.

## Qué no debe afirmarse

- Equivalencia clínica entre SYN y EEG humano.
- Que las señales sintéticas sean indistinguibles de señales reales.
- Cobertura completa de la diversidad morfológica o distribucional clínica.
- Capacidad diagnóstica o de predicción clínica.
- Privacidad formal a partir de la proximidad en características.

El dictamen actual es **viable reforzado con limitaciones**: el framework es útil
para investigación y transferencia, pero conserva desplazamiento de dominio frente
al EEG real.

## Archivos principales

| Archivo | Función |
|---|---|
| `1_0_EEGSynthesizer_DATASET.ipynb` | Generación, pilotos, contrato morfológico y promoción atómica del SYN |
| `1_1_EEGSynthesizer_VALIDATION.ipynb` | Desarrollo, CHB-MIT, Siena, fidelidad, diversidad y transferencia |
| `requirements.txt` | Entorno exacto utilizado para reproducir los resultados |
| `CITATION.cff` | Metadatos de citación del software |

Los EDF y arrays grandes no se almacenan en Git. Se descargan desde sus fuentes
oficiales o se generan localmente.

## Requisitos

- Python 3.13.7 de 64 bits.
- Al menos 60 GiB libres para la reproducción completa.
- Conexión estable para las descargas iniciales.
- Aproximadamente 20.3 GiB para Siena y 11.8 GiB para los EDF CHB-MIT seleccionados.

Las descargas pueden interrumpirse y reanudarse posteriormente.

## Instalación

```bash
git clone https://github.com/EMENESES86/EEGSynthesizer.git
cd EEGSynthesizer
python -m venv .venv
```

Windows PowerShell:

```powershell
.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
python -m pip install -r requirements.txt
```

Linux o macOS:

```bash
source .venv/bin/activate
python -m pip install --upgrade pip
python -m pip install -r requirements.txt
```

Iniciar Jupyter desde la raíz del repositorio:

```bash
jupyter lab
```

No debe abrirse el notebook desde otro directorio: ambos notebooks comprueban que
el directorio de trabajo sea la raíz correcta.

## Reproducción completa desde cero

### Paso 1 — generación del SYN

Abra `1_0_EEGSynthesizer_DATASET.ipynb` y mantenga:

```python
GENERATION_MODE = "full"
CALIBRATION_MODE = "frozen"
```

Ejecute **Run All**.

`frozen` reproduce la decisión metodológica publicada: el candidato calibrado no
fue promovido porque empeoró el dominio espacial, por lo que se conserva el
baseline científicamente evaluado. El snapshot es pequeño, público y se verifica
contra el perfil de desarrollo mediante SHA-256.

Si `dataset_eeg_final` no existe, el baseline se reconstruye completamente desde
los parámetros y semillas congelados. Si existe y todos sus hashes son válidos,
se muestra que ya está generado y no se modifica.

El modo siguiente es un experimento metodológico nuevo, no la reproducción exacta
de la versión publicada:

```python
CALIBRATION_MODE = "recompute"
```

Este modo vuelve a calcular el contraste baseline–candidato usando exclusivamente
la cohorte de desarrollo. Nunca utiliza CHB-MIT externo ni Siena para recalibrar.

### Paso 2 — validación externa CHB-MIT y Siena

Abra `1_1_EEGSynthesizer_VALIDATION.ipynb`, conserve el modo predeterminado y
ejecute **Run All** una sola vez:

```python
NOTEBOOK_MODE = "siena_external"
```

El notebook construye o reutiliza primero CHB-MIT, excluye los sujetos de
desarrollo y ejecuta:

- R2R;
- TSTR;
- TRTS;
- R+S→R;
- SMOTE→R;
- fidelidad, cobertura, diversidad y proximidad en características;
- bootstrap por sujeto.

A continuación, en la misma ejecución, descarga o reutiliza Siena desde PhysioNet,
verifica 58 entradas oficiales —incluidos 41 EDF— y procesa 14 sujetos.

La primera ejecución puede descargar aproximadamente 20.3 GiB. Si los archivos ya
existen, aparecerá un mensaje como:

```text
Siena: 58/58 archivos verificados; EDF oficiales: 41; pendientes: 0
Siena ya está descargado y verificado; no es necesaria una nueva descarga.
```

### Modos opcionales de mantenimiento

`prepare_dev` vuelve a descargar/reutilizar los EDF de desarrollo y reconstruye el
perfil 5.1 con `chb01`, `chb02`, `chb03`, `chb05` y `chb06`; `chb21` se trata como
la misma persona que `chb01`. No es necesario para la reproducción normal porque
el perfil congelado y sus hashes forman parte del repositorio.

`external` ejecuta únicamente CHB-MIT. `status` inspecciona sin recalcular:

Para conocer qué etapas existen:

```python
NOTEBOOK_MODE = "status"
```

Este modo no descarga ni ejecuta la validación. Solo informa entorno, espacio libre,
estado del SYN y checkpoints disponibles.

## Ejecución automatizada

La configuración visible es suficiente para Jupyter. Para automatización también
se aceptan variables del sistema.

Windows PowerShell:

```powershell
$env:EEGSYNTH_MODE='full'
$env:EEGSYNTH_CALIBRATION_MODE='frozen'
jupyter nbconvert --to notebook --execute 1_0_EEGSynthesizer_DATASET.ipynb --output generation_executed.ipynb --ExecutePreprocessor.timeout=-1

$env:EEGSYN_VALIDATION_MODE='siena_external'
jupyter nbconvert --to notebook --execute 1_1_EEGSynthesizer_VALIDATION.ipynb --output siena_executed.ipynb --ExecutePreprocessor.timeout=-1
```

Esta es la secuencia completa de reproducción. Los modos `prepare_dev` y `external`
son opciones de mantenimiento descritas anteriormente y no forman parte de la
ejecución normal. Linux o macOS usa la misma secuencia anteponiendo la variable al
comando.

## Descargas, reanudación e integridad

Cada archivo se acepta únicamente cuando coincide con el SHA-256 oficial.

- Archivo completo y válido: se reutiliza.
- Archivo `.part`: la descarga continúa desde el último byte disponible.
- Archivo completo corrupto: se reemplaza individualmente.
- Resultado derivado incompleto: se reconstruye en un candidato temporal.
- Resultado derivado válido: se reutiliza mediante su manifiesto.

La promoción de datasets es atómica. Una interrupción no sustituye un dataset
válido por archivos parciales.

## Checkpoints

`dataset_doctorado_final/reproducibility_state.json` registra:

| Etapa | Significado |
|---|---|
| S1 | Perfil de desarrollo terminado |
| S3 | Dataset sintético congelado |
| S4 | CHB-MIT derivado e íntegro |
| S5 | Corpus Siena oficial verificado |
| S6 | Ventanas Siena derivadas e íntegras |
| S7 | Snapshots compactos de resultados externos disponibles |

Cada etapa incluye estado, fecha UTC, hashes, tamaños y el siguiente paso. Los
checkpoints no sustituyen los datos fuente: permiten decidir objetivamente qué
puede reutilizarse y qué debe reconstruirse.

Por ello, en un clon nuevo puede aparecer el snapshot S7 disponible mientras S3,
S4 o S6 figuran incompletos: las tablas compactas se distribuyen con el repositorio,
pero los arrays pesados deben generarse o descargarse antes de repetir los análisis.

## Resultados de referencia que deben reproducirse

### Productos para tesis y artículo

El bloque final de `1_1_EEGSynthesizer_VALIDATION.ipynb` muestra una tabla
multicorpus y reconstruye automáticamente seis figuras a partir de los artefactos
verificados. Se guardan versiones PNG y PDF, una tabla CSV/LaTeX, leyendas y un
manifiesto SHA-256 dentro de
`dataset_doctorado_final/validation_q1_assets/`. Las figuras distinguen criterios
respaldados, criterios completos no alcanzados y estimaciones exploratorias. Su
alcance inferencial comprende plausibilidad morfológica operacional, cobertura por
rasgos y utilidad de transferencia externa; los CSV y JSON conservan la evidencia
numérica completa.

La misma carpeta incluye `16_publication_figure_audit.csv`, con la fuente, el uso
permitido y la limitación de cada figura, y
`16_publication_representative_signals.csv`, que identifica los sujetos, eventos,
ventanas, derivaciones y amplitudes crudas de las señales mostradas. Las señales y
la PSD utilizan únicamente ventanas con ocupación ictal completa.

### Integridad del generador

- 3 000 pacientes y 180 000 ventanas.
- TRAIN/VAL/TEST sin pacientes compartidos.
- Amplitud: `1500/1500` eventos dentro del contrato.
- Medición en ventana completamente limpia: `1459/1500` (`97.27 %`). Los otros
  `41/1500` (`2.73 %`) utilizaron la medición fallback y conservan amplitud válida;
  por ello, la medición limpia universal es `NOT SUPPORTED`.
- Error mediano objetivo–observado: `0.00000257` relativo.
- Masa exacta en límites: `0 %`.
- Frecuencia generalizada de 2.5–4 Hz: `750/750` (`100 %`).
- Máximo frontocentral generalizado: `747/750` (`99.6 %`).
- Simetría bilateral en pares frontales no degradados: `750/750` (`100 %`, `SUPPORTED`).
- Simetría observable en la señal final, incluyendo degradación: `603/750` (`80.4 %`,
  `NOT SUPPORTED`); los 147 fallos tuvieron degradación frontal.
- Los controles internos programados para el escenario focal fueron aprobados.

Estos controles demuestran cumplimiento de las reglas programadas; por sí solos no
constituyen validación clínica independiente.

### CHB-MIT externo

- 18 sujetos externos.
- TSTR RF z-score: AUROC macro `0.6431`.
- IC95 % bootstrap por sujeto: `[0.5460, 0.7333]`.
- Estado de TSTR: `SUPPORTED`.
- Fidelidad distribucional: `NOT SUPPORTED`.
- Cobertura morfológica externa completa: `NOT SUPPORTED`.
- Diversidad comparable con REAL: `NOT SUPPORTED`.

### Siena

Siena contiene 14 sujetos y 47 crisis declaradas. El procesamiento aceptó 46 y
excluyó una mediante una regla documentada.

Endpoint ictal frente a preictal limpio de 30 a 5 minutos, con exclusión postictal
prioritaria, evaluable en 14 sujetos:

- 5 600 ventanas preictales; solapamiento con los 30 minutos postictales: `0`.
- Ventanas preictales fuera del intervalo 30–5 min: `0`.
- TSTR RF z-score: AUROC macro `0.7008`.
- IC95 %: `[0.6212, 0.7762]`.
- Estado: `SUPPORTED`.
- CHB+SYN frente a CHB: diferencia media `+0.0563`, IC95 % `[0.0329, 0.0808]`;
  13 de 14 sujetos mejoraron.

Endpoint ictal frente a interictal estricto:

- solo cuatro sujetos fueron evaluables;
- se reporta como `NOT EVALUABLE` para inferencia multicorpus general.

La semejanza morfológica con Siena es parcial: varias características se encuentran
dentro de la variación REAL–REAL, pero existen diferencias distribucionales,
diversidad sintética menor y lateralidad imperfecta. Esto obliga a conservar el
dictamen con limitaciones.

## Fuentes reales

- CHB-MIT Scalp EEG Database 1.0.0: <https://physionet.org/content/chbmit/1.0.0/>
  — DOI `10.13026/C2K01R`.
- Siena Scalp EEG Database 1.0.0: <https://physionet.org/content/siena-scalp-eeg/1.0.0/>
  — DOI `10.13026/5d4a-j060`.

Debe citarse cada corpus conforme a su página oficial y respetarse su licencia.

## Solución de problemas

### “Falta el perfil de desarrollo”

Restaure los tres artefactos públicos de desarrollo o ejecute `1_1` con
`NOTEBOOK_MODE="prepare_dev"`. En una copia completa del repositorio no se necesita
este paso.

### “Faltan prerrequisitos de Siena”

Ejecute primero `1_0` en `full/frozen` y después `1_1` en `siena_external`.

### Descarga interrumpida

No elimine los `.part`. Ejecute nuevamente la misma etapa.

### Hash inválido

El notebook reemplaza solamente el archivo afectado. No elimine todo el corpus.

### Espacio insuficiente

Libere espacio hasta disponer de al menos 60 GiB para una reconstrucción completa.

### Tiempo de ejecución

En el equipo de referencia, usando el entorno fijado y datos ya verificados:

- generación con dataset SYN reutilizado, pilotos y auditoría: aproximadamente 1 minuto;
- `prepare_dev`: aproximadamente 3 minutos;
- validación CHB-MIT determinista: aproximadamente 13 minutos;
- validación Siena determinista: aproximadamente 22–24 minutos.

La descarga inicial depende de la red. El modo determinista ejecuta Random Forest
en un solo hilo para obtener resultados numéricos repetibles; por eso puede ser más
lento que una ejecución paralela.

### El notebook crea archivos en una ubicación incorrecta

Cierre Jupyter, abra una terminal en la raíz de EEGSynthesizer e inicie Jupyter desde
allí.

## Alcance científico

El aporte es un framework paramétrico controlado, reproducible y auditable para
investigación de transferencia. Los resultados respaldan utilidad experimental,
pero no reproducción completa del EEG clínico ni equivalencia diagnóstica.

Los resultados desfavorables de fidelidad, cobertura o diversidad forman parte de
la evidencia y no se ocultan ni se convierten en positivos mediante umbrales
posteriores.

## Licencia y citación

Consulte `LICENSE_DATA` y `CITATION.cff`. Los datos CHB-MIT y Siena conservan las
licencias y requisitos de citación establecidos por PhysioNet.

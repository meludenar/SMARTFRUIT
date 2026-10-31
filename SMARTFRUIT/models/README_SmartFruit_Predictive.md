# SmartFruit · Predictive Loss v1.3
### Modelo v0 del componente predictivo para el entregable E5

SmartFruit es un prototipo de apoyo al proceso de inspección de limones en recepción.  
Este repositorio/notebook implementa el primer componente predictivo funcional del proyecto, denominado técnicamente **Predictive Loss v1.3**.

> Para efectos del entregable E5, esta versión corresponde al **modelo v0** del componente predictivo.

El objetivo del modelo es evaluar si la información disponible al finalizar la primera inspección contiene señal suficiente para estimar el porcentaje de pérdida posterior (`loss%`) y aportar una señal adicional al **Decision Layer**.

---

## 1. Flujo general

```text
Scan / Reception Manifest
        ↓
Primera muestra QC
        ↓
Vision AI
        ↓
Predictive Loss
        ↓
Rules / Decision Layer
        ↓
Standard / Expand / Escalate
```

El modelo predictivo **no toma la decisión final**.

Su salida principal es:

```text
predicted_loss_pct
```

La decisión `Standard / Expand / Escalate` corresponde al **Decision Layer**, que combinará:

- reglas del proceso;
- evidencia de Vision AI;
- contexto de recepción;
- señal predictiva;
- revisión del inspector.

---

## 2. Objetivo del notebook

El notebook construye un pipeline reproducible para:

1. leer los reportes históricos de QC;
2. leer la información histórica de pérdidas de IMP;
3. auditar y normalizar variables;
4. construir un dataset analítico sin atribuir pérdidas ambiguas de Work Orders multilote;
5. separar train, validation y test por lote;
6. comparar un baseline y dos modelos de regresión;
7. seleccionar el modelo usando únicamente validation;
8. evaluar el modelo seleccionado una sola vez en test congelado;
9. analizar errores por segmentos;
10. demostrar el contrato de integración `Scan + QC + Vision → Predictive`.

---

## 3. Estructura esperada en Google Drive

```text
MyDrive/
└── SMARTFRUIT/
    ├── data/
    │   ├── reportes.zip
    │   └── Weight IN - OUT by lot - item.xlsx
    ├── integration/
    │   └── SmartFruit_reception_manifest.csv
    ├── vision/
    └── models/
```

### Fuentes principales

**QC histórico**

```text
SMARTFRUIT/data/reportes.zip
```

Contiene los reportes individuales de inspección.

**IMP / pérdidas**

```text
SMARTFRUIT/data/Weight IN - OUT by lot - item.xlsx
```

Hoja utilizada:

```text
REPACK LOSS REPORT
```

**Manifest de integración**

```text
SMARTFRUIT/integration/SmartFruit_reception_manifest.csv
```

El manifest se utiliza únicamente para demostrar el flujo de integración.  
**No forma parte del entrenamiento del modelo.**

---

## 4. Datos utilizados

La corrida v1.3 procesó:

| Fuente / etapa | Resultado |
|---|---:|
| Reportes QC únicos | 106 |
| Lotes QC | 62 |
| Filas QC / pallet / muestra | 515 |
| Work Orders IMP | 557 |
| Work Orders multilote | 343 |
| Work Orders de un solo lote | 214 |
| Work Orders single-lot con QC enlazado | 207 |
| Work Orders usados en modelado | 201 |
| Lotes únicos del dataset analítico | 52 |

Se identificaron **15 Work Orders con `loss%` negativo**.  
Estos casos permanecen auditables, pero se excluyen del entrenamiento principal de esta versión.

---

## 5. Construcción del target

La variable objetivo es:

```text
loss_calc
```

y se calcula como:

```text
loss% = (ORDERInitialWeight - ORDERFinalWeight)
        / ORDERInitialWeight * 100
```

### Unidad de predicción

La unidad analítica es:

```text
Work Order de un solo lote
```

enlazado al **primer QC disponible del lote**.

Esta restricción evita asignar artificialmente el mismo resultado de pérdida a varios lotes cuando un Work Order contiene más de uno.

---

## 6. Prevención de leakage

Para reducir fuga de información:

- se usa solamente el **primer QC del lote**;
- se excluyen Work Orders multilote;
- se excluyen variables posteriores a la decisión;
- `ORDERInitialWeight`, `ORDERFinalWeight` y componentes directos del target no se usan como features;
- el split se realiza por `lot_id`;
- ningún lote puede aparecer simultáneamente en train, validation y test;
- el test permanece congelado hasta la evaluación final.

---

## 7. EDA y Data Quality

Antes del modelado se revisan:

- `label_brand`;
- `country_origin`;
- `size`;
- missingness;
- distribuciones numéricas;
- conteos de defectos;
- porcentajes fuera del rango esperado;
- valores extremos.

### Brand

En la versión utilizada para esta corrida:

| Métrica | Valor |
|---|---:|
| Brands raw únicos | 42 |
| Brands después de normalización básica | 31 |
| Brands estandarizados | 29 |

Las variantes de **DON VITO** se consolidaron.

Ejemplos:

```text
Don Vito
Don Vito's
Don Vitos
DON VITO'S
```

→

```text
DON VITO
```

### Importante: estado de aliases en esta corrida

Este README reproduce exactamente el estado del notebook v1.3 original.

Por lo tanto:

```text
RDL PRODUCCE
```

se conserva como categoría independiente y permanece marcada como posible typo para revisión.

Asimismo:

```text
LIMON FRESH LIMES
```

y

```text
FRESH LIMES
```

se mantienen como categorías distintas en esta versión.

No se aplican merges automáticos basados únicamente en similitud textual.

### País de origen

Los datos QC contienen principalmente:

```text
MEXICO
COLOMBIA
```

### Hallazgos de calidad de datos

Se detectaron, entre otros:

- 1 fila con `oleocellosis > sample_count`;
- 1 `oleocellosis_pct > 100`;
- valor extremo `pulp_temp = 521`;
- variantes de nombres de brands que requieren revisión manual.

Los outliers se reportan, pero **no se eliminan automáticamente**.

---

## 8. Features

### Numéricas

```text
inspection_month
sample_count_total
n_pallets

avg_count_per_box_mean
avg_count_per_box_max

pulp_temp_mean
pulp_temp_max

oleocellosis_pct
oleocellosis_pct_max_pallet

styler_pct
styler_pct_max_pallet

skin_breakdown_pct
skin_breakdown_pct_max_pallet

decay_pct
decay_pct_max_pallet

mold_pct
mold_pct_max_pallet

scars_pct
scars_pct_max_pallet

blanching_pct
blanching_pct_max_pallet

yellow_color_pct
yellow_color_pct_max_pallet
```

### Categóricas

```text
label_brand_std
country_origin_std
size_norm
```

### Feature puente con Vision AI

La principal variable prevista para conectar Vision con Predictive es:

```text
oleocellosis_pct
```

Durante el entrenamiento histórico esta variable proviene de los registros QC.

En la integración futura deberá provenir de Vision AI, después de validar que la medición automática sea comparable con la medición histórica/manual.

---

## 9. Modelos evaluados

### Baseline 0 — mediana del TRAIN

No aprende ninguna relación entre features y target.

Predice siempre:

```text
mediana(loss% del TRAIN)
```

Sirve como referencia mínima para responder:

> ¿El modelo de ML aporta valor frente a una predicción trivial?

---

### Modelo simple — Huber Regression

Regresión lineal robusta.

Se utiliza como baseline ML interpretable porque reduce la influencia de observaciones extremas mediante la función de pérdida Huber.

Configuración principal:

```python
HuberRegressor(
    epsilon=1.35,
    alpha=0.01,
    max_iter=1000
)
```

---

### Modelo v0 candidato — Gradient Boosting Regressor

Modelo no lineal basado en árboles secuenciales.

Configuración:

```python
GradientBoostingRegressor(
    n_estimators=150,
    learning_rate=0.03,
    max_depth=2,
    loss="huber"
)
```

Permite representar relaciones e interacciones no lineales entre las variables.

---

## 10. Preprocesamiento

### Variables numéricas para Huber

```text
SimpleImputer(strategy="median")
→ StandardScaler()
```

### Variables numéricas para Gradient Boosting

```text
SimpleImputer(strategy="median")
```

### Variables categóricas

```text
SimpleImputer(strategy="most_frequent")
→ OneHotEncoder(
      handle_unknown="infrequent_if_exist",
      min_frequency=2
   )
```

---

## 11. Split de desarrollo

El split se realiza mediante `GroupShuffleSplit` usando `lot_id` como grupo.

| Split | Work Orders | Lotes |
|---|---:|---:|
| Train | 123 | 29 |
| Validation | 22 | 10 |
| Test | 56 | 13 |

Los grupos son disjuntos.

```text
Train ∩ Validation = ∅
Train ∩ Test       = ∅
Validation ∩ Test  = ∅
```

---

## 12. Criterio de selección

La métrica principal es:

```text
MAE — Mean Absolute Error
```

También se reportan:

```text
RMSE
MedAE
R²
```

La selección de modelo utiliza **únicamente validation**.

El test no se utiliza para ajustar hiperparámetros ni para cambiar de modelo después de observar sus resultados.

---

## 13. Resultados en validation

Baseline de referencia:

```text
Mediana TRAIN = 16.4
```

| Modelo | MAE validation | RMSE validation | Mejora MAE vs baseline |
|---|---:|---:|---:|
| Huber Regression | **13.569** | 17.837 | **+3.921%** |
| Gradient Boosting v0 | 15.013 | 17.857 | -6.302% |

### Modelo seleccionado

```text
Huber Regression
```

Huber obtuvo el menor MAE en validation, por lo que fue seleccionado **antes de abrir el test**.

---

## 14. Resultados en test congelado

| Modelo | MAE | IC 95% MAE | RMSE | MedAE | R² | Mejora vs baseline |
|---|---:|---:|---:|---:|---:|---:|
| Gradient Boosting v0 | 16.003 | [11.782, 20.546] | 21.683 | 12.291 | -0.126 | +2.828% |
| Baseline mediana | 16.468 | [11.882, 21.186] | 22.480 | 11.168 | -0.211 | 0.000% |
| **Huber Regression** | **27.353** | **[16.975, 43.706]** | **40.341** | **16.700** | **-2.899** | **-66.097%** |

Los intervalos del MAE se calculan mediante **bootstrap agrupado** para preservar la dependencia por lote.

---

## 15. Interpretación del resultado

Huber fue el modelo seleccionado correctamente en validation.

Sin embargo, en test:

```text
Baseline MAE = 16.468
Huber MAE    = 27.353
```

Por lo tanto, Huber **no generalizó** al conjunto de prueba.

Aunque Gradient Boosting obtuvo:

```text
MAE = 16.003
```

no se cambia el modelo seleccionado después de observar el test.

Hacerlo convertiría el test en un nuevo conjunto de selección.

### Veredicto

```text
Predictive Loss v1.3:
EXPLORATORIO / NO OPERATIVO
```

La versión actual **no pasa el gate de desempeño** definido para justificar su uso operativo.

El resultado negativo se conserva porque forma parte de una evaluación metodológicamente válida.

---

## 16. Análisis por segmentos

El análisis se realiza sobre el modelo seleccionado: **Huber Regression**.

### País de origen

Todo el test congelado quedó compuesto por casos de:

```text
MEXICO
```

Por lo tanto, el test actual no permite comparar generalización entre países.

| Segmento | n | MAE |
|---|---:|---:|
| México | 56 | 27.353 |

---

### Oleocelosis

| Segmento | n | MAE |
|---|---:|---:|
| Oleocelosis ≤ mediana DEV | 23 | 30.030 |
| Oleocelosis > mediana DEV | 33 | 25.488 |

La variable de oleocelosis por sí sola no explica adecuadamente la pérdida posterior.

---

### Brand

| Brand | n | MAE |
|---|---:|---:|
| MEDRANO | 20 | 43.258 |
| DON VITO | 32 | 19.383 |
| LIMES | 4 | 11.594 |

En **MEDRANO** se observa el mayor error.

Promedios del segmento MEDRANO:

```text
loss real medio      = 28.493
loss predicho medio  = -8.008
```

Esto evidencia una extrapolación deficiente del modelo lineal.

---

## 17. Contrato de integración

### A. Scan context

La recepción ya conoce:

```text
scan_code
shipment_id
lot_id
label_brand
country_origin
pallet_id
packing_date
size_declared
expected_count_per_box
```

Estos campos **no deben inferirse desde la imagen**.

---

### B. QC context

Información producida durante la primera inspección:

```text
temperatura
sample_count
count por caja
otros defectos disponibles
```

---

### C. Vision output

Vision AI debe entregar al menos:

```text
image_id
total_fruits
oleocellosis_count
oleocellosis_pct
```

Opcionalmente:

```text
mean_confidence
```

Si Vision entrega:

```text
total_fruits
oleocellosis_count
```

el notebook puede calcular:

```text
oleocellosis_pct =
oleocellosis_count / total_fruits * 100
```

---

## 18. Demo técnica de integración

Caso utilizado:

```text
scan_code = SF-5815-5645
```

Contexto recuperado:

```text
lot_id         = 5815
brand          = Paramount
country        = Mexico
pallet         = 5645
size           = 110
```

Vision fue simulado para probar el contrato técnico:

```text
image_id              = IMG_DEMO_001
total_fruits           = 120
oleocellosis_count     = 9
oleocellosis_pct       = 7.5%
mean_confidence        = 0.90
```

Resultado del componente Predictive:

```text
predicted_loss_pct = 31.6898%
```

### Importante

Este valor:

- **no valida el accuracy de Vision**;
- no representa una decisión operacional;
- solamente demuestra que el flujo de integración puede ejecutarse.

La salida final mantiene:

```python
decision = None
```

porque la decisión corresponde al Decision Layer.

---

## 19. Artefactos generados

El notebook guarda:

```text
SMARTFRUIT/models/predictive_loss_v1_3.joblib
```

y:

```text
SMARTFRUIT/models/predictive_loss_v1_3_metadata.json
```

El archivo de metadata documenta:

- modelo seleccionado;
- target;
- features;
- método de selección;
- grupos del test;
- contrato con Vision;
- contrato de salida;
- advertencias de uso.

---

## 20. Cómo ejecutar

### Google Colab

1. Abrir el notebook.
2. Montar Google Drive cuando sea solicitado.
3. Confirmar que existan:

```text
MyDrive/SMARTFRUIT/data/reportes.zip
MyDrive/SMARTFRUIT/data/Weight IN - OUT by lot - item.xlsx
```

4. Para la demo de integración, agregar:

```text
MyDrive/SMARTFRUIT/integration/SmartFruit_reception_manifest.csv
```

5. Ejecutar las celdas **en orden desde el inicio**.

No se recomienda ejecutar únicamente las celdas de entrenamiento porque el notebook depende de las transformaciones y auditorías realizadas previamente.

---

## 21. Dependencias principales

```text
pandas
numpy
scikit-learn
matplotlib
joblib
openpyxl
```

Además se utilizan librerías estándar de Python para:

```text
zipfile
pathlib
hashlib
json
re
unicodedata
```

---

## 22. Limitaciones

La versión v1.3 tiene varias limitaciones que deben mantenerse explícitas:

1. El dataset analítico final contiene solo **52 lotes**.
2. El test contiene únicamente **13 lotes**.
3. Todos los casos del test quedaron en México.
4. `loss%` es un **proxy** de resultado posterior; no equivale a Hidden Quality Risk.
5. Los Work Orders multilote se excluyen porque el target no puede atribuirse correctamente a un lote individual.
6. Se detectaron anomalías de Data Quality pendientes de revisión.
7. Algunas categorías de brand permanecen fragmentadas intencionalmente porque no existía confirmación suficiente para consolidarlas.
8. La medición histórica de oleocelosis proviene de QC manual; Vision todavía debe calibrarse contra esa referencia.
9. Huber produce predicciones negativas en algunos segmentos, señal de extrapolación deficiente.
10. El test actual no debe utilizarse para ajustar esta misma iteración.

---

## 23. Próxima iteración

Antes de volver a entrenar:

- revisar anomalías de Data Quality;
- cerrar equivalencias de categorías únicamente con confirmación de negocio;
- ampliar cobertura de lotes y orígenes;
- validar la salida de Vision AI contra QC manual;
- incorporar más datos representativos;
- definir un nuevo experimento con un nuevo test congelado;
- avanzar hacia un target de **Hidden Quality Risk** cuando exista un outcome posterior por lote con timestamp y atribución confiables.

---

## 24. Estado del componente

```text
Componente: Predictive Loss
Versión técnica: v1.3
Rol en E5: Modelo v0 del componente predictivo
Modelo seleccionado por validation: Huber Regression
Estado: Exploratorio
Uso operativo: No aprobado
Salida: predicted_loss_pct
Decisión final: Decision Layer
```

---

## 25. Principio metodológico

El objetivo de esta versión no es demostrar que el modelo funciona a toda costa.

El objetivo es demostrar una evaluación reproducible y defendible:

```text
datos
→ auditoría
→ features
→ split por lote
→ baseline
→ selección en validation
→ test congelado
→ análisis de errores
→ integración
```

El hecho de que el modelo seleccionado no supere al baseline en test es un resultado válido y se reporta explícitamente, sin re-seleccionar el modelo después de observar el conjunto de prueba.

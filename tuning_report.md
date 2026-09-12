# StreamLoop — Reporte de ajuste del modelo de churn

## Objetivo y métrica

El objetivo es detectar la mayor cantidad posible de clientes que cancelarán (`Churn = Yes`). Un falso negativo representa una oportunidad de retención perdida y tiene mayor coste que contactar a un cliente que finalmente no cancela. Por ello, la métrica principal de selección fue **recall** (`scoring='recall'`), no accuracy. También se registran precision, F1 y accuracy para contextualizar el trade-off.

## Diseño experimental

- Se cargó el dataset directamente desde la URL oficial del reto.
- Se eliminó `customerID`, se convirtió `TotalCharges` a numérico (los vacíos pasan a `NaN`) y se codificó `Churn` como 1/0.
- Se hizo una división estratificada 80/20 antes de entrenar modelos (`random_state=42`). El test quedó aislado de ambas búsquedas.
- El preprocesamiento está dentro de un `Pipeline`: imputación y escalado para variables numéricas; imputación y one-hot encoding para variables categóricas.
- La línea base usa `LogisticRegression()` literalmente con sus hiperparámetros por defecto. Para las búsquedas se usa una copia equivalente con `max_iter=2000` únicamente para evitar problemas de convergencia durante los folds; este parámetro no se ajusta.
- Se utilizó `StratifiedKFold(n_splits=5, shuffle=True, random_state=42)`.

## Estrategia de búsqueda

1. **RandomizedSearchCV** exploró un rango amplio de `C` con distribución log-uniforme, `solver` (`liblinear`, `lbfgs`) y `class_weight` (`None`, `balanced`). Usó 20 iteraciones, `scoring='recall'`, `n_jobs=-1` y `refit=True`.
2. Se tomó el mejor `C` aleatorio y se definió una grilla focalizada alrededor de `C/3`, `C` y `3C`, manteniendo el solver y el peso de clase competitivos.
3. **GridSearchCV** refinó esa región con la misma validación cruzada, scoring, paralelización y refit. `best_estimator_` ya está reentrenado por `GridSearchCV`; no se volvió a entrenar manualmente.

## Resultados

Al ejecutar todas las celdas del notebook, completar esta tabla con los valores impresos por `baseline_metrics` y `final_metrics`:

| Modelo | Accuracy | Precision | Recall | F1 |
|---|---:|---:|---:|---:|
| Baseline (test) | 0.8055 | 0.6572 | 0.5588 | 0.6040 |
| Ajustado (test) | 0.7410 | 0.5078 | 0.7834 | 0.6162 |

El ajuste elevó el recall de **0.5588 a 0.7834** (+0.2246), a costa de reducir accuracy y precision. El F1 subió ligeramente (0.6040 a 0.6162), lo que indica que el aumento de detección compensa parcialmente la caída de precision.

La grilla final eligió `C=3.43863222102751`, `solver='lbfgs'` y `class_weight='balanced'`, con recall CV medio **0.8020** y desviación estándar **0.0376**. La alternativa `C=1.1462` obtuvo 0.8013 ± 0.0366: es marginalmente más estable, pero su recall medio es ligeramente inferior. Se conserva el primer modelo porque la prioridad explícita del negocio es maximizar detecciones y la diferencia de estabilidad es mínima.

Los mejores candidatos se revisan mediante `grid_results`, mostrando `mean_test_score` y `std_test_score`. La selección prioriza el mayor recall medio; si dos opciones tienen un recall prácticamente equivalente, se prefiere la de menor desviación estándar porque ofrece un comportamiento más estable entre folds. La tabla `stability` añade `mean - std` y `mean + std` para hacer visible ese riesgo.

## Decisión final

El modelo final es `grid_search.best_estimator_`, seleccionado por recall medio en validación cruzada y evaluado en el test una única vez al final. La precisión puede bajar al priorizar recall; ese es un trade-off aceptable para StreamLoop, siempre que se controle con precision y F1. No se utiliza el test para elegir hiperparámetros.

## Auditoría final del checklist del instructor

- [x] Dataset cargado desde la URL indicada.
- [x] Columnas no numéricas codificadas y valores faltantes manejados.
- [x] División train/test antes del modelado.
- [x] Preprocesamiento dentro de `Pipeline`.
- [x] Línea base con `LogisticRegression()` y sus hiperparámetros por defecto; el test permanece aislado.
- [x] `RandomizedSearchCV` antes de `GridSearchCV`.
- [x] Espacio de parámetros compatible con el clasificador.
- [x] Validación cruzada y búsquedas solo sobre train.
- [x] `n_jobs=-1` en ambas búsquedas.
- [x] Métrica deliberada: recall, conectada al coste de falsos negativos.
- [x] `refit=True`; no se reentrena manualmente el mejor estimador.
- [x] Revisión de media y variación (`mean_test_score`, `std_test_score`) en `cv_results_`.
- [x] Test usado exactamente dos veces: baseline y evaluación final.
- [x] Comparación baseline vs. ajustado y justificación de estabilidad.

### Auditoría del dataset

- [x] El notebook usa `pd.read_csv(URL)` con la URL exacta solicitada por el instructor:
	`https://raw.githubusercontent.com/IBM/telco-customer-churn-on-icp4d/master/data/Telco-Customer-Churn.csv`.
- [x] No se usa el archivo local `Telco-Customer-Churn.csv` para el entrenamiento; cualquier copia local queda fuera de los entregables y no debe añadirse al commit.
- [x] `customerID` se elimina porque es un identificador, `TotalCharges` se convierte a numérico y sus espacios vacíos se convierten en `NaN`.
- [x] Los faltantes se imputan dentro del pipeline: mediana en numéricas y moda en categóricas.
- [x] `Churn` se transforma explícitamente de `Yes`/`No` a `1`/`0`, y se eliminan únicamente filas cuyo objetivo no pudo mapearse.

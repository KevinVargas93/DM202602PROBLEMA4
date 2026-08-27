# Problema #4 — Utilización de los meses de pandemia

Experimento de Training Strategy sobre predicción de bajas de clientes bancarios.

**Materia:** Data Mining — ITBA, cohorte 2026B
**Alumno:** Vargas Teran, Kevin Leonardo
**Modalidad:** Analista Jr
**Semilla primigenia:** 100103

---

## Pregunta

Al entrenar un modelo de churn con datos que atraviesan la pandemia de 2020, ¿conviene excluir esos meses del entrenamiento?

Se comparan cuatro configuraciones, cada una con 10 semillas distintas (40 corridas en total).

| Celda | Nombre | Meses excluidos | Meses de training | Meses de final train |
|---|---|---|---|---|
| A | Baseline | ninguno | 27 | 29 |
| B | Experimento 1 | 202003–202004 | 25 | 27 |
| C | Experimento 2 | 202006 | 26 | 28 |
| D | Experimento 3 | 202003–202012 | 17 | 19 |

---

## Resultado

| Celda | Ganancia promedio | Desvío | Mínimo | Mejora vs baseline | p-valor |
|---|---|---|---|---|---|
| A | 92.869.250 | 2.625.000 | 88.678.000 | — | — |
| B | 93.681.500 | 1.880.000 | 89.690.000 | 6 de 10 | 0,5566 |
| C | 93.857.000 | 2.300.000 | 90.490.000 | 5 de 10 | 0,7695 |
| **D** | **98.084.250** | **1.184.000** | **96.520.000** | **10 de 10** | **0,0020** |

**Conclusión:** excluir marzo–diciembre de 2020 mejora la ganancia de forma consistente. Las exclusiones puntuales (B y C) son indistinguibles del baseline.

Con n=10, el p-valor de 0,0020 es el mínimo alcanzable en un test de Wilcoxon pareado.

---

## Diseño experimental

**Métrica:** ganancia con matriz de costos asimétrica de +975.000 por acierto (BAJA+2) y −25.000 por error.

**Ventana temporal:**
- Training (grid search): 201901 → 202103
- Validación: 202105
- Final train: 201901 → 202105
- Predicción: **202107**

El mes 202106 se omite deliberadamente: su `clase_ternaria` depende de quién desaparece en 202107, que es el mes a predecir.

**Semillas** (derivadas de la primigenia 100103):

```
100103, 315967, 352357, 403681, 439567, 539503, 691787, 761263, 778567, 801137
```

**Test estadístico:** Wilcoxon de rangos con signo para muestras pareadas. Cada par compara la misma semilla en dos configuraciones, lo que cancela la varianza entre semillas.

**Nota sobre 202107:** este mes tiene ground truth en el dataset (244 BAJA+2 sobre 32.938 clientes). Las ganancias reportadas son medidas, no estimadas.

---

## Estructura del repositorio

```
notebooks/     Workflows de las cuatro celdas (R, formato Jupyter)
scripts/       Consolidación de resultados y análisis estadístico
resultados/    40 carpetas WF*, una por corrida
analisis/      Tablas consolidadas listas para analizar
```

Cada carpeta en `resultados/` contiene:

| Archivo | Contenido |
|---|---|
| `PARAM.yml` | Configuración completa, hiperparámetros elegidos y resultado |
| `tb_grid_search_01.txt` | Las 60 combinaciones del grid search con su AUC |
| `impo.txt` | Importancia de variables del modelo final |

---

## Reproducir

### Requisitos

- R 4.5 o superior
- Paquetes: `data.table`, `lightgbm`, `yaml`, `rlist`, `primes`, `ggplot2`
- Máquina con al menos 32 GB de RAM (las corridas se hicieron en una VM de 12 vCPU)

### Dataset

**No está incluido en este repositorio** por su tamaño y porque es material del curso.

El archivo es `analistajr_competencia_2026.csv.gz` (~90 MB comprimido, 976.925 filas, 33 meses de 201901 a 202109, 55 columnas más `clase_ternaria`).

Ubicalo en `~/datasets/` antes de correr nada.

### Notebooks por celda

| Celda | Notebook | Meses excluidos |
|---|---|---|
| A (Baseline) | `z911_WorkFlow_01_junior_julio.ipynb` | ninguno |
| B (Experimento 1) | `z911_WorkFlow_01_junior_julio-EXP01.ipynb` | 202003–202004 |
| C (Experimento 2) | `z911_WorkFlow_01_junior_julio-EXP02.ipynb` | 202006 |
| D (Experimento 3) | `z911_WorkFlow_01_junior_julio-EXP03.ipynb` | 202003–202012 |

### Correr una celda

```bash
NB="z911_WorkFlow_01_junior_julio-EXP03"   # notebook de la celda
CELDA="20260801_D"                          # prefijo del experimento
SEMILLA=100103

jupyter nbconvert --to script "notebooks/${NB}.ipynb"

sed "s|PARAM\$semilla_primigenia <-.*|PARAM\$semilla_primigenia <- $SEMILLA|; \
     s|PARAM\$experimento <-.*|PARAM\$experimento <- \"${CELDA}_${SEMILLA}\"|" \
  "notebooks/${NB}.r" > "/tmp/run_${SEMILLA}.r"

Rscript "/tmp/run_${SEMILLA}.r"
```

Repetir para las 10 semillas de cada celda. Conviene lanzarlo dentro de `tmux`.

### Consolidar y analizar

```bash
Rscript scripts/consolidar.r
```

Genera `consolidado_resumen.csv` (una fila por corrida) y `consolidado_curvas.csv` (curvas de ganancia muestreadas).

---

## Referencias sin modelo

Para dimensionar el resultado se calcularon cuatro puntos de comparación sobre el dataset crudo de 202107, sin entrenar ningún modelo.

| Estrategia | Envíos | Aciertos | Ganancia |
|---|---|---|---|
| Enviar a todos | 32.938 | 244 | −579.450.000 |
| Azar | 2.100 | 16 | −36.500.000 |
| No enviar a nadie | 0 | 0 | 0 |
| Regla simple (`ctrx_quarter` ascendente) | 1.996 | 116 | 66.100.000 |
| Experimento 3 | 2.251 | 154 | 97.725.000 |
| Oráculo perfecto | 244 | 244 | 237.900.000 |

El Experimento 3 captura el **41,2%** de la ganancia máxima teórica.

---

## Limitaciones

**Un solo período de evaluación.** La conclusión se apoya en 202107. La mejora es consistente en las diez semillas, pero no está verificada en múltiples cortes temporales.

**Punto de corte elegido ex-post.** El óptimo se determina con la verdad conocida de 202107, así que las ganancias reportadas son optimistas respecto de un escenario operativo real. Como el criterio es idéntico en las cuatro celdas, la comparación entre ellas no se ve afectada.

**Confundidor de `num_iterations`.** El grid search determina la cantidad de árboles y su resultado cambia al cambiar los meses de training. La mediana va de 288 en el baseline a 193 en el Experimento 3. Parte del efecto observado puede provenir de esta vía indirecta y no de la exclusión en sí. El diseño no separa ambos mecanismos.

**Óptimo en el borde del grid.** En varias corridas el mejor valor cayó en el extremo superior de la grilla (`num_leaves` en 1024–2048, `min_data_in_leaf` en 2048). El óptimo real podría estar más allá del rango explorado. No afecta la comparación porque todas las celdas usan la misma grilla.

**Comparaciones múltiples.** Se corrieron 6 tests de Wilcoxon. Con corrección de Bonferroni el umbral pasa de 0,05 a 0,0083; los tres resultados significativos siguen pasando.

---

## Bibliografía

Argentina (2020). *Decreto 297/2020 — Aislamiento Social, Preventivo y Obligatorio*. Boletín Oficial, 20 de marzo de 2020.
https://www.argentina.gob.ar/normativa/nacional/335741/texto

Arjovsky, M., Bottou, L., Gulrajani, I. y Lopez-Paz, D. (2019). *Invariant Risk Minimization*. arXiv:1907.02893.
https://arxiv.org/abs/1907.02893

Gama, J., Žliobaitė, I., Bifet, A., Pechenizkiy, M. y Bouchachia, A. (2014). *A Survey on Concept Drift Adaptation*. ACM Computing Surveys, 46(4), artículo 44.
https://doi.org/10.1145/2523813

He, H. y Garcia, E. A. (2009). *Learning from Imbalanced Data*. IEEE Transactions on Knowledge and Data Engineering, 21(9), 1263–1284.
https://doi.org/10.1109/TKDE.2008.239

Ke, G., Meng, Q., Finley, T., Wang, T., Chen, W., Ma, W., Ye, Q. y Liu, T.-Y. (2017). *LightGBM: A Highly Efficient Gradient Boosting Decision Tree*. Advances in Neural Information Processing Systems 30.
https://proceedings.neurips.cc/paper/2017/file/6449f44a102fde848669bdd9eb6b76fa-Paper.pdf

Quiñonero-Candela, J., Sugiyama, M., Schwaighofer, A. y Lawrence, N. D. (eds.) (2009). *Dataset Shift in Machine Learning*. MIT Press.
https://direct.mit.edu/books/edited-volume/3841/Dataset-Shift-in-Machine-Learning

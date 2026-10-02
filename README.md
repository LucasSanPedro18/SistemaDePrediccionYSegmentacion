# ✈️ DS Airlines — Predicción de demoras y segmentación de pasajeros

Trabajo Práctico Integrador de **Ciencia de Datos** (Ingeniería en Sistemas de Información, 2026).

Documento Word: https://docs.google.com/document/d/180EeaR6hsr2v_QJ5kf0YTaYQrpo-1HYiMk4-XFGFoBE/edit?usp=sharing

Agente inteligente que **anticipa qué vuelos se van a demorar** y **asigna un beneficio compensatorio a cada pasajero afectado** según su perfil (voucher de comida o acceso a sala VIP). Así la aerolínea pasa de una gestión reactiva de las demoras a una proactiva.

El proyecto sigue la metodología **CRISP-DM** y combina un modelo de clasificación supervisada con un modelo de segmentación no supervisada.

---

## 🧩 El problema

DS Airlines necesita resolver dos cosas:

1. **Clasificación:** predecir si un vuelo nuevo va a tener demora (`demora`: 0 = no, 1 = sí), a partir de clima, congestión aérea, visibilidad, temporada, etc.
2. **Segmentación:** agrupar a los pasajeros afectados en perfiles para decidir qué beneficio recibe cada uno.

El criterio de éxito es **minimizar costos**. Dejar a un pasajero demorado sin aviso (falso negativo) es mucho más grave que dar un beneficio innecesario (falso positivo), así que se usa una matriz de costos **FN = 5, FP = 1**:

```
Costo total = (FP × 1) + (FN × 5)
```

---

## 🔄 Pipeline

```
Vuelos.xlsx ──► Preparación ──► Árbol de Decisión ──► Vuelos nuevos clasificados
                                                              │
                                                    vuelos demorados
                                                              ▼
casosNuevos.xlsx (VuelosNuevos_Clientes) ──► Pasajeros afectados
                                                              │
Clientes.xlsx ──────────────────────────────────────────────►─┤
                                                              ▼
                                       K-Medias (k = 3) ──► Segmento ──► Beneficio
                                                              ▼
                                            reporte_final_DSAirlines.xlsx
```

---

## 📊 Datos

| Dataset | Registros | Descripción |
|---|---|---|
| `Vuelos.xlsx` | 15.000 | Vuelos históricos con variable objetivo `demora` (68 % / 32 %) |
| `Clientes.xlsx` | 50.000 | Perfil sociodemográfico, comportamiento de compra y fidelización |
| `casosNuevos.xlsx` | 5.000 vuelos / 356.416 pares vuelo–pasajero | Casos a predecir y tabla relacional vuelo–cliente |

Ninguno de los datasets tiene valores nulos ni duplicados. Los outliers se conservaron porque representan casos reales del dominio (vuelos transoceánicos, clientes de alto poder adquisitivo, viajeros muy frecuentes).

---

## 🛠️ Preparación de los datos

**Vuelos**
- Se eliminó `tiempo_estimado_vuelo` por colinealidad con `distancia_vuelo` (r = 0,999).
- Se excluyeron variables sin poder discriminante: `puerta_embarque`, `tipo_avion`, `aeropuerto_origen`, `aeropuerto_destino`.
- `hora_salida_programada` se convirtió de `"HH:MM"` a número decimal.
- One-hot encoding con categorías de referencia: `Lunes`, `Despejado`, `Baja`.
- Escalamiento min-max ajustado sólo sobre entrenamiento.
- Partición 70/30 estratificada (`random_state = 42`).

**Clientes**
- Se eliminó `gasto_acumulado_extra` por redundancia con `gasto_acumulado` (r = 0,906).
- Se excluyeron `sexo` (sin poder discriminante) y `provincia` (demasiadas categorías sin patrón).
- Seis variables numéricas estandarizadas con z-score: `edad`, `cant_vuelos`, `cantidad_millas`, `ingreso_mensual`, `anticipacion_compra_promedio`, `gasto_acumulado`.
- Las categóricas (`ocupacion`, `clase_preferida`, `programaMillas`, `canal_compra`) se usaron para caracterizar los clusters.

---

## 🤖 Modelo 1 — Clasificación de vuelos

Se compararon cuatro técnicas sobre el mismo conjunto de prueba (4.500 vuelos):

| Modelo | Herramienta | Accuracy | Sensibilidad | FP | FN | Costo total |
|---|---|---|---|---|---|---|
| **Árbol de Decisión** (`max_depth=5`, Gini) | Python | **70,9 %** | **24,5 %** | 229 | 1.082 | **5.639** |
| KNN (`k=41`, euclídea) | Python | 70,4 % | 22,5 % | 222 | 1.110 | 5.772 |
| Regresión Logística | SPSS Statistics | 70,6 % | 22,0 % | 204 | 1.118 | 5.794 |
| Análisis Discriminante | SPSS Statistics | 68,4 % | 2,2 % | 23 | 1.401 | 7.028 |

✅ **Modelo elegido: Árbol de Decisión.** Tiene el menor costo total, la mayor sensibilidad sobre la clase demorado, es interpretable y no muestra sobreajuste a profundidad 5.

Variables más importantes: **visibilidad** (≈ 0,40), **congestión aérea alta** (≈ 0,23) y **temporada alta** (≈ 0,10). La regresión logística confirma el peso del clima: una tormenta multiplica casi por 5 las chances de demora (odds ratio = 4,85).

---

## 👥 Modelo 2 — Segmentación de pasajeros

Se aplicaron K-Medias, Cluster Jerárquico (Ward) y Bietápico (SPSS Two-Step) sobre los pasajeros afectados por vuelos demorados, y ACP para validar la estructura y visualizar los clusters.

| Modelo | Silhouette | Distribución | Observación |
|---|---|---|---|
| **K-Medias** (`k=3`) | 0,219 | 8,7 % / 46,6 % / 44,7 % | Segmento Premium como cluster propio |
| Cluster Jerárquico | 0,21 | 11,1 % / 43,8 % / 45,1 % | Estructura casi idéntica a K-Medias |
| Bietápico | ~0,1 | 56 % / 24,2 % / 16,5 % + 3,3 % outliers | Trata al Premium como ruido |

✅ **Modelo elegido: K-Medias**, por aislar al segmento Premium y alinearse con la lógica de beneficios.

| Segmento | Clientes | Perfil | Beneficio |
|---|---|---|---|
| **Cliente Premium** | 2.654 (8,7 %) | ~40 años, ~56 vuelos, ~50.000 millas, ingreso ~8.200 USD, mayoría Ejecutivos/Empresarios | 🛋️ Sala VIP |
| **Cliente Adulto Frecuente** | 14.216 (46,6 %) | ~47 años, ~15 vuelos, clase Económica | 🍔 Voucher comida |
| **Cliente Joven Ocasional** | 13.648 (44,7 %) | ~28 años, ~12 vuelos, compra con más anticipación y por App | 🍔 Voucher comida |

El ACP muestra dos ejes principales: uno de **valor económico** (gasto, vuelos, millas, ingreso) y uno **generacional** (edad vs. anticipación de compra). Los tres primeros componentes explican el 72,25 % de la varianza.

---

## 📋 Resultado final sobre los casos nuevos

| Métrica | Valor |
|---|---|
| Vuelos clasificados | 5.000 |
| Vuelos predichos como demorados | 659 (13,2 %) |
| Pasajeros únicos afectados | 30.518 |
| Pares vuelo–pasajero con beneficio | 47.247 |
| Asignaciones de Sala VIP | 4.044 |
| Asignaciones de Voucher comida | 43.203 |

El archivo `reporte_final_DSAirlines.xlsx` tiene dos hojas:
- `asignacion_beneficios`: los 356.416 pares vuelo–pasajero con las columnas `ID vuelo | ID pasajero | Predicho_demora_vuelo | Tipo beneficio`.
- `solo_demorados`: la lista accionable con los 47.247 pares cuyo vuelo se predice demorado.

---

## 📁 Estructura del repositorio

> Ajustá esta sección a los nombres reales de tus archivos.

```
├── notebooks/
│   ├── Preparacion_Vista_Minable_SPSS.ipynb   # Vista minable y exportación .sav para SPSS
│   ├── Aplicacion_Arbol_CasosNuevos.ipynb      # Árbol de Decisión sobre casos nuevos
│   ├── KMedias_ClientesDemorados.ipynb         # Segmentación con K-Medias
│   └── Reporte_Final_DSAirlines.ipynb          # Pipeline integrador completo
├── data/
│   ├── Vuelos.xlsx
│   ├── Clientes.xlsx
│   └── casosNuevos.xlsx
├── docs/
│   └── TPI_Ciencia_de_Datos.pdf                # Informe completo
└── README.md
```

---

## ▶️ Cómo ejecutarlo

1. Instalar dependencias:
   ```bash
   pip install pandas numpy scikit-learn openpyxl
   ```
2. Colocar `Vuelos.xlsx`, `Clientes.xlsx` y `casosNuevos.xlsx` en la misma carpeta que el notebook (o subirlos a Google Colab).
3. Ejecutar `Reporte_Final_DSAirlines.ipynb` de principio a fin.
4. Se genera `reporte_final_DSAirlines.xlsx`.

El pipeline usa `random_state = 42` en todos los pasos, así que los resultados son reproducibles.

---

## 🧰 Tecnologías

- **Python** · pandas · NumPy · scikit-learn · SciPy · matplotlib / seaborn
- **Google Colab**
- **IBM SPSS Statistics** (Regresión Logística, Análisis Discriminante, Two-Step)
- **IBM SPSS Modeler**

---

## ⚠️ Limitaciones y mejoras futuras

- La sensibilidad de todos los clasificadores sobre la clase demorado ronda el **22–25 %**: cerca de 3 de cada 4 demoras reales no se anticipan. Es una limitación de la información disponible y del desbalance 68/32.
- Bajar el umbral de decisión (hoy 0,5) aumentaría la detección de demoras a cambio de más falsos positivos; es una decisión de negocio.
- En producción haría falta reentrenamiento periódico y monitoreo del desempeño y de la aparición de categorías nuevas.

---

## 👨‍💻 Autores

- Lucas San Pedro
- Joaquín Giménez
- Sofía Elisa Coppari
- Jerónimo Álvarez

**Docentes:** Silvia Scime y Juan Moine — Comisión 503, turno noche.

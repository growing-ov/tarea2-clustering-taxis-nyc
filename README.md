# Segmentación de viajes de taxi en Nueva York — aprendizaje no supervisado

Notebook: [`aprendizaje_no_supervisado.ipynb`](aprendizaje_no_supervisado.ipynb). Está ejecutado, así que
se ven todas sus salidas: tablas, gráficas, métricas e imágenes.

---

## El problema en una frase

Nadie ha dicho qué tipos de viaje existen en la ciudad de Nueva York. Este trabajo los descubre
**sin usar ninguna etiqueta**: agrupa los viajes por cómo se parecen entre sí y después explica qué
hay dentro de cada grupo.

---

## Los datos

**NYC TLC — Yellow Taxi Trip Records, enero de 2024.** 2.964.624 viajes reales, descargados del CDN
público de la comisión de taxis, sin cuenta ni API key:

```
https://d37ci6vzurychx.cloudfront.net/trip-data/yellow_tripdata_2024-01.parquet
```

Después de limpiar, quedan **2.265.576 viajes (2,27 M)** para agrupar, muy por encima de los 100.000
exigidos.

Se eligió este dataset porque es real, masivo, se lee en menos de un segundo (Parquet columnar) y
tiene la mezcla de variables que hace falta: numéricas (distancia, tarifa, propina, peajes),
temporales (hora, día) y categóricas (tipo de pago, tarifa, vendor). Online Retail II, en cambio, se
queda en unos 5.900 clientes después de agregar por cliente (RFM).

---

## Cómo se resuelve

| Paso | Qué se hace |
|---|---|
| 1 | Nulos estructurales (`payment_type = 0`), filtro a pagos con tarjeta (la propina solo se registra con tarjeta), duplicados exactos y fechas fuera de enero |
| 2 | Reglas de negocio sobre valores imposibles: distancia 0,1-100 mi, duración 1-180 min, velocidad 1-80 mph, tarifa 3-500 USD |
| 3 | Extremos por criterio IQR, tratados con *winsorización* (p0,5-p99,5) en lugar de borrarlos |
| 4 | 9 variables derivadas y estandarizadas con `StandardScaler` |
| 5 | Número de grupos elegido con **codo + silueta** (apoyados en Davies-Bouldin y Calinski-Harabasz) → **k = 6** |
| 6 | **MiniBatch K-Means** entrenado con los 2,27 M de registros; **GMM** y **DBSCAN** como contrincantes |
| 7 | Evaluación con **silueta**, **Davies-Bouldin** y **Calinski-Harabasz**, más visualizaciones, perfil de cada grupo y zonas de origen/destino |

Las variables que entran al modelo son distancia, duración, velocidad, tarifa y porcentaje de propina
(en escala logarítmica), la hora codificada de forma cíclica, el indicador de fin de semana y el
indicador de aeropuerto. Ninguna de ellas es una variable objetivo: **no se usa ninguna etiqueta
conocida para entrenar**.

---

## Resultados

Muestra de evaluación de 30.000 registros (misma muestra para los tres modelos):

| Modelo | Silueta ↑ | Davies-Bouldin ↓ | Calinski-Harabasz ↑ |
|---|---|---|---|
| **MiniBatch K-Means (k = 6)** | **0,230** | **1,454** | **8.343** |
| GMM covarianza completa (k = 6) | 0,175 | 2,564 | 5.453 |
| DBSCAN (4 grupos, sin ruido) | 0,219 | 1,809 | 6.677 |

Sobre una muestra de 500.000 registros: MiniBatch K-Means deja **Davies-Bouldin 1,453** y
**Calinski-Harabasz 140.045**, frente a 2,606 y 92.560 del GMM.

La silueta es estable: en 5 muestras distintas de 20.000 registros la media es 0,2275 y la desviación
0,0010. Un valor de ≈ 0,23 es **moderado**: los grupos existen, pero sus fronteras se solapan, que es
lo normal en datos de comportamiento.

---

## Los seis grupos

| Grupo | Nombre | % viajes | % ingresos |
|---|---|---|---|
| C1 | ✈️ Aeropuerto / larga distancia | 9,9 % | **28,9 %** |
| C3 | 🏠 Regreso a casa entre semana (tarde-noche) | 22,1 % | 23,5 % |
| C2 | 🏢 Desplazamientos de mañana/mediodía entre semana | 21,1 % | 15,9 % |
| C0 | 🏙 Cortos urbanos entre semana (tarde-noche) | 23,6 % | 14,0 % |
| C5 | 🎉 Ocio de fin de semana | 13,3 % | 12,3 % |
| C4 | 🚕 Cortos de fin de semana | 10,0 % | 5,3 % |

El detalle de cada grupo (medianas de distancia, duración, velocidad, tarifa, propina, franja
horaria, zonas de origen y destino más frecuentes) está en el notebook.

---

## Cómo ejecutarlo

**Google Colab.** Sube el `.ipynb` y pulsa *Ejecutar todo*. La primera celda instala lo que falte y la
segunda descarga el dataset. Tarda unos 3-5 minutos.

**Local.**

```bash
python3 -m venv .venv            # o: uv venv .venv --python 3.12
source .venv/bin/activate
pip install -r requirements.txt
jupyter nbconvert --to notebook --execute --inplace aprendizaje_no_supervisado.ipynb
```

---

## Estructura del repositorio

```
.
├── aprendizaje_no_supervisado.ipynb   # notebook ejecutado (código, salidas, gráficas, conclusiones)
├── data/
│   ├── yellow_tripdata_2024-01.parquet # dataset original (se descarga solo si no está)
│   └── taxi_zone_lookup.csv            # tabla pública de zonas TLC (nombres de barrio)
├── requirements.txt
└── README.md
```

> El directorio `data/` se entrega con el dataset ya descargado para que se pueda revisar sin
> conexión, pero está en `.gitignore`: versionar un Parquet de 48 MB no es buena práctica. El
> notebook lo baja solo cuando no lo encuentra.

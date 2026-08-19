# Demostración 4. Construir un tablero FinOps básico

## Metadata

| Campo | Valor |
|-------|-------|
| **Duración** | 60 minutos |
| **Complejidad** | Alta |
| **Nivel Bloom** | Crear |
| **Entregable principal** | `novatech_finops_dashboard_v1.xlsx` |

## Descripción General

Este laboratorio culminante consolida todos los artefactos generados en los laboratorios previos del curso para construir un tablero FinOps funcional para NovaTech SRL. Trabajarás en dos fases: primero prepararás los datos y calcularás KPIs usando Python en Jupyter Notebook, y luego construirás el tablero visual en LibreOffice Calc con semáforos RAG, presupuestos con umbrales de alerta, forecast por regresión lineal y un log de anomalías con flujo de triage.

## Objetivos de Aprendizaje

- [ ] Construir un tablero FinOps funcional en LibreOffice Calc que integre KPIs de gasto, variación, cobertura, utilización, desperdicio y unit cost
- [ ] Implementar un modelo de presupuesto con umbrales de alerta y responsables asignados usando los datos de asignación de costos de NovaTech SRL
- [ ] Desarrollar una proyección de gasto (forecast) para Q1 2024 usando regresión lineal simple con `numpy.polyfit`
- [ ] Diseñar un proceso de gestión de anomalías con flujo de triage y seguimiento integrado en el tablero FinOps

## Prerrequisitos

### Conocimientos Requeridos

| Tema | Nivel |
|------|-------|
| Conceptos FinOps: KPIs (gasto, variación, cobertura, utilización, desperdicio, unit cost) | Intermedio |
| Python: pandas, numpy, openpyxl | Intermedio |
| LibreOffice Calc: fórmulas condicionales, gráficas, formato condicional | Intermedio |
| Modelo RAG (Red-Amber-Green) para semáforos | Básico |
| Material teórico Módulo 4: secciones 4.1 a 4.5 | Completado |

### Archivos de Entrada Obligatorios

Todos los archivos deben existir en `~/finops-course/data/processed/` o `~/finops-course/outputs/`:

| Archivo | Origen |
|---------|--------|
| `diagnostico_finops_novatech_v1.xlsx` | Lab 01-00-01 |
| `novatech_billing_summary_jan2024.xlsx` | Lab 02-00-01 |
| `novatech_trend_report.xlsx` | Lab 02-00-02 |
| `novatech_cost_allocation_q4_2023.xlsx` | Lab 03-00-01 |
| `novatech_billing_90days.csv` | Dataset provisto por instructor |

> **Nota:** Si no completaste algún laboratorio previo, solicita al instructor los archivos de referencia pre-completados.

## Entorno del Laboratorio

### Software Requerido

| Herramienta | Versión | Propósito |
|-------------|---------|-----------|
| Python | 3.12.1 | Motor de cálculo |
| pandas | 2.2.1 | Consolidación de datos |
| numpy | 1.26.4 | Regresión lineal (polyfit) |
| matplotlib | 3.8.3 | Gráficas exportadas |
| openpyxl | 3.1.2 | Generación de Excel |
| Jupyter Notebook | 7.1.2 | Entorno interactivo |
| LibreOffice Calc | 24.2.1 | Tablero visual final |

### Configuración Inicial

```bash
# Verificar directorio de trabajo
cd ~/finops-course

# Verificar que los archivos de entrada existen
ls data/processed/diagnostico_finops_novatech_v1.xlsx
ls outputs/novatech_billing_summary_jan2024.xlsx
ls outputs/novatech_trend_report.xlsx
ls outputs/novatech_cost_allocation_q4_2023.xlsx
ls data/raw/novatech_billing_90days.csv

# Verificar dependencias Python
pip install -r requirements.txt

# Crear directorio para outputs de este lab
mkdir -p outputs/dashboard

# Iniciar Jupyter Notebook
jupyter notebook --port=8888 --notebook-dir=~/finops-course/notebooks/
```

---

## Instrucciones Paso a Paso

---

### Paso 1: Crear el Notebook de Preparación de Datos

**Objetivo:** Crear el notebook `lab_04_01_dashboard_data_prep.ipynb` e importar todos los datasets necesarios de laboratorios anteriores.

**Instrucciones:**

1. En Jupyter Notebook (http://localhost:8888), crea un nuevo notebook llamado `lab_04_01_dashboard_data_prep.ipynb`.

2. En la primera celda, escribe las importaciones y configuración base:

```python
# Celda 1: Importaciones y configuración
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
from openpyxl import Workbook
from openpyxl.styles import Font, PatternFill, Alignment, Border, Side
from openpyxl.chart import BarChart, LineChart, Reference
from openpyxl.utils.dataframe import dataframe_to_rows
from pathlib import Path
import warnings
warnings.filterwarnings('ignore')

# Rutas base
BASE_DIR = Path.home() / 'finops-course'
DATA_RAW = BASE_DIR / 'data' / 'raw'
DATA_PROC = BASE_DIR / 'data' / 'processed'
OUTPUTS = BASE_DIR / 'outputs'
DASH_OUT = OUTPUTS / 'dashboard'
DASH_OUT.mkdir(parents=True, exist_ok=True)

print("✅ Librerías importadas correctamente")
print(f"📁 Directorio de salida: {DASH_OUT}")
```

3. En la segunda celda, carga todos los archivos de entrada:

```python
# Celda 2: Carga de datos de laboratorios previos
# Billing diario 90 días (Q4 2023 + enero 2024)
billing_90d = pd.read_csv(DATA_RAW / 'novatech_billing_90days.csv', parse_dates=['date'])

# Resumen de facturación enero 2024 (Lab 02-00-01)
billing_jan = pd.read_excel(OUTPUTS / 'novatech_billing_summary_jan2024.xlsx')

# Reporte de tendencias (Lab 02-00-02)
trend_report = pd.read_excel(OUTPUTS / 'novatech_trend_report.xlsx')

# Asignación de costos Q4 2023 (Lab 03-00-01)
cost_alloc = pd.read_excel(OUTPUTS / 'novatech_cost_allocation_q4_2023.xlsx')

print("✅ Datasets cargados:")
print(f"   billing_90d: {billing_90d.shape[0]} filas")
print(f"   billing_jan: {billing_jan.shape[0]} filas")
print(f"   cost_alloc:  {cost_alloc.shape[0]} filas")
```

**Resultado Esperado:**

```
✅ Librerías importadas correctamente
📁 Directorio de salida: /home/<usuario>/finops-course/outputs/dashboard
✅ Datasets cargados:
   billing_90d: [número] filas
   billing_jan: [número] filas
   cost_alloc:  [número] filas
```

**Verificación:** Confirma que no hay errores de `FileNotFoundError`. Si algún archivo falta, solicítalo al instructor.

---

### Paso 2: Calcular KPIs FinOps Principales

**Objetivo:** Calcular los 6 KPIs FinOps requeridos para el Executive Summary: gasto total, variación mensual, cobertura de reservas, utilización de reservas, desperdicio estimado y costo unitario.

**Instrucciones:**

1. En una nueva celda, calcula el gasto total y variación:

```python
# Celda 3: KPI 1 y 2 - Gasto total y variación mensual
# Gasto por mes
billing_90d['month'] = billing_90d['date'].dt.to_period('M')
gasto_mensual = billing_90d.groupby('month')['cost'].sum().reset_index()
gasto_mensual.columns = ['month', 'total_cost']
gasto_mensual['month_str'] = gasto_mensual['month'].astype(str)

print("Gasto mensual NovaTech SRL:")
print(gasto_mensual[['month_str', 'total_cost']].to_string(index=False))

# KPI: Gasto total enero 2024 (último mes)
gasto_enero = gasto_mensual[gasto_mensual['month_str'] == '2024-01']['total_cost'].values[0]

# KPI: Variación MoM (enero vs diciembre)
gasto_diciembre = gasto_mensual[gasto_mensual['month_str'] == '2023-12']['total_cost'].values[0]
variacion_mom = ((gasto_enero - gasto_diciembre) / gasto_diciembre) * 100

print(f"\n📊 KPI - Gasto Total Enero 2024: ${gasto_enero:,.2f}")
print(f"📊 KPI - Variación MoM (Ene vs Dic): {variacion_mom:+.1f}%")
```

2. En la siguiente celda, calcula cobertura y utilización de reservas (valores simulados basados en el diagnóstico del Lab 01):

```python
# Celda 4: KPI 3 y 4 - Cobertura y Utilización de reservas
# Datos derivados del diagnóstico FinOps (Lab 01-00-01)
# NovaTech tiene reservas parciales - valores del diagnóstico
gasto_on_demand = gasto_enero * 0.65  # 65% sigue siendo on-demand
gasto_reservas = gasto_enero * 0.35   # 35% cubierto por reservas/savings plans

# Cobertura = % del gasto elegible cubierto por compromisos
cobertura_reservas = 35.0  # porcentaje

# Utilización = % de las reservas compradas que realmente se usan
# Del diagnóstico: hay reservas infrautilizadas
utilizacion_reservas = 72.0  # porcentaje (28% desperdiciado)

# KPI 5: Desperdicio estimado
desperdicio_reservas = gasto_reservas * (1 - utilizacion_reservas / 100)
desperdicio_recursos_idle = gasto_enero * 0.08  # 8% en recursos ociosos (del diagnóstico)
desperdicio_total = desperdicio_reservas + desperdicio_recursos_idle

# KPI 6: Costo unitario (cost per active user)
# NovaTech: PayCore procesa ~500,000 usuarios activos mensuales
usuarios_activos = 500_000
costo_por_usuario = gasto_enero / usuarios_activos

print(f"📊 KPI - Cobertura de Reservas: {cobertura_reservas:.0f}%")
print(f"📊 KPI - Utilización de Reservas: {utilizacion_reservas:.0f}%")
print(f"📊 KPI - Desperdicio Total Estimado: ${desperdicio_total:,.2f}")
print(f"📊 KPI - Costo por Usuario Activo: ${costo_por_usuario:.4f}")
```

3. Consolida todos los KPIs en un DataFrame:

```python
# Celda 5: Consolidar KPIs en DataFrame
kpis = pd.DataFrame({
    'KPI': [
        'Gasto Total Mensual',
        'Variación MoM',
        'Cobertura de Reservas',
        'Utilización de Reservas',
        'Desperdicio Estimado',
        'Costo por Usuario Activo'
    ],
    'Valor': [
        f"${gasto_enero:,.2f}",
        f"{variacion_mom:+.1f}%",
        f"{cobertura_reservas:.0f}%",
        f"{utilizacion_reservas:.0f}%",
        f"${desperdicio_total:,.2f}",
        f"${costo_por_usuario:.4f}"
    ],
    'Meta': [
        'Dentro de presupuesto',
        '< ±10%',
        '> 70%',
        '> 80%',
        '< 5% del gasto total',
        '< $0.10'
    ],
    'Estado_RAG': [
        'AMBER' if variacion_mom > 5 else 'GREEN',
        'RED' if abs(variacion_mom) > 10 else ('AMBER' if abs(variacion_mom) > 5 else 'GREEN'),
        'RED' if cobertura_reservas < 50 else ('AMBER' if cobertura_reservas < 70 else 'GREEN'),
        'RED' if utilizacion_reservas < 60 else ('AMBER' if utilizacion_reservas < 80 else 'GREEN'),
        'RED' if (desperdicio_total/gasto_enero*100) > 10 else ('AMBER' if (desperdicio_total/gasto_enero*100) > 5 else 'GREEN'),
        'GREEN' if costo_por_usuario < 0.10 else 'AMBER'
    ]
})

print("═══ RESUMEN EJECUTIVO KPIs FinOps - NovaTech SRL ═══")
print(kpis.to_string(index=False))
```

**Resultado Esperado:**

Una tabla con 6 KPIs, cada uno con su valor calculado, meta definida y estado RAG (RED/AMBER/GREEN).

**Verificación:** Confirma que `Estado_RAG` contiene al menos un valor `RED` o `AMBER` (indicando áreas de mejora en NovaTech).

---

### Paso 3: Generar Datos de Presupuesto vs. Actuals

**Objetivo:** Crear la tabla de Budget vs. Actuals por equipo con umbrales de alerta al 80% y 100%.

**Instrucciones:**

1. Crea la celda de presupuestos por equipo:

```python
# Celda 6: Presupuesto vs. Actuals por equipo
# Presupuestos mensuales asignados (definidos en Lab 03-00-01)
presupuestos = pd.DataFrame({
    'Equipo': ['Backend', 'Frontend', 'Data', 'Platform'],
    'Presupuesto_Mensual': [18000, 8000, 15000, 12000],
    'Responsable': ['Carlos Méndez', 'Ana Torres', 'Luis Paredes', 'María Castillo']
})

# Gasto real por equipo en enero 2024 (del billing)
if 'team' in billing_90d.columns:
    gasto_equipo_ene = billing_90d[billing_90d['month'].astype(str) == '2024-01'].groupby('team')['cost'].sum().reset_index()
    gasto_equipo_ene.columns = ['Equipo', 'Gasto_Real']
else:
    # Si la columna team no existe, usar cost_alloc como referencia y escalar
    gasto_equipo_ene = pd.DataFrame({
        'Equipo': ['Backend', 'Frontend', 'Data', 'Platform'],
        'Gasto_Real': [19200, 7500, 16800, 11500]
    })

budget_vs_actual = presupuestos.merge(gasto_equipo_ene, on='Equipo', how='left')
budget_vs_actual['Porcentaje_Ejecucion'] = (budget_vs_actual['Gasto_Real'] / budget_vs_actual['Presupuesto_Mensual'] * 100)
budget_vs_actual['Alerta_80'] = budget_vs_actual['Presupuesto_Mensual'] * 0.80
budget_vs_actual['Alerta_100'] = budget_vs_actual['Presupuesto_Mensual'] * 1.00

# Estado de alerta
def estado_presupuesto(row):
    pct = row['Porcentaje_Ejecucion']
    if pct > 100:
        return 'RED - Sobre presupuesto'
    elif pct > 80:
        return 'AMBER - Alerta preventiva'
    else:
        return 'GREEN - Dentro de rango'

budget_vs_actual['Estado'] = budget_vs_actual.apply(estado_presupuesto, axis=1)

print("═══ BUDGET vs. ACTUALS - Enero 2024 ═══")
print(budget_vs_actual[['Equipo', 'Presupuesto_Mensual', 'Gasto_Real', 
                         'Porcentaje_Ejecucion', 'Estado']].to_string(index=False))
```

**Resultado Esperado:**

```
═══ BUDGET vs. ACTUALS - Enero 2024 ═══
   Equipo  Presupuesto_Mensual  Gasto_Real  Porcentaje_Ejecucion                    Estado
  Backend                18000       19200                106.7  RED - Sobre presupuesto
 Frontend                 8000        7500                 93.8  AMBER - Alerta preventiva
     Data                15000       16800                112.0  RED - Sobre presupuesto
 Platform                12000       11500                 95.8  AMBER - Alerta preventiva
```

**Verificación:** Al menos un equipo debe tener estado `RED` para validar que el tablero mostrará alertas activas.

---

### Paso 4: Ejecutar Forecast con Regresión Lineal

**Objetivo:** Proyectar el gasto de Q1 2024 (febrero, marzo, abril) usando `numpy.polyfit` de grado 1 con banda de confianza.

**Instrucciones:**

1. Prepara los datos de serie temporal y ejecuta la regresión:

```python
# Celda 7: Forecast Q1 2024 con regresión lineal
# Preparar serie temporal diaria
daily_cost = billing_90d.groupby('date')['cost'].sum().reset_index()
daily_cost = daily_cost.sort_values('date').reset_index(drop=True)

# Variable independiente: días desde el inicio (numérico)
daily_cost['day_num'] = (daily_cost['date'] - daily_cost['date'].min()).dt.days

# Regresión lineal con numpy polyfit grado 1
x = daily_cost['day_num'].values
y = daily_cost['cost'].values

coefficients = np.polyfit(x, y, 1)
slope = coefficients[0]
intercept = coefficients[1]

print(f"Modelo lineal: costo = {slope:.2f} * día + {intercept:.2f}")
print(f"Tendencia diaria: ${slope:+.2f}/día")

# Predicción para los próximos 90 días (Feb-Mar-Abr 2024)
ultimo_dia = daily_cost['day_num'].max()
dias_forecast = np.arange(ultimo_dia + 1, ultimo_dia + 91)
fechas_forecast = pd.date_range(start=daily_cost['date'].max() + pd.Timedelta(days=1), periods=90)

# Predicción puntual
forecast_values = slope * dias_forecast + intercept

# Banda de confianza (±1 desviación estándar de los residuos)
y_pred_historico = slope * x + intercept
residuos = y - y_pred_historico
std_residuos = np.std(residuos)

forecast_upper = forecast_values + 1.96 * std_residuos
forecast_lower = forecast_values - 1.96 * std_residuos

# Crear DataFrame de forecast
forecast_df = pd.DataFrame({
    'Fecha': fechas_forecast,
    'Forecast_USD': forecast_values,
    'Banda_Superior': forecast_upper,
    'Banda_Inferior': forecast_lower
})

# Resumen mensual del forecast
forecast_df['Mes'] = forecast_df['Fecha'].dt.to_period('M')
forecast_mensual = forecast_df.groupby('Mes').agg({
    'Forecast_USD': 'sum',
    'Banda_Superior': 'sum',
    'Banda_Inferior': 'sum'
}).reset_index()
forecast_mensual['Mes'] = forecast_mensual['Mes'].astype(str)

print("\n═══ FORECAST Q1 2024 ═══")
print(forecast_mensual.to_string(index=False))
print(f"\nTotal Q1 2024 proyectado: ${forecast_mensual['Forecast_USD'].sum():,.2f}")
```

2. Genera la gráfica de forecast:

```python
# Celda 8: Gráfica de forecast
fig, ax = plt.subplots(figsize=(12, 5))

# Datos históricos
ax.plot(daily_cost['date'], daily_cost['cost'], color='#2196F3', alpha=0.6, 
        linewidth=0.8, label='Gasto diario real')

# Línea de tendencia histórica
ax.plot(daily_cost['date'], y_pred_historico, color='#FF5722', 
        linewidth=2, linestyle='--', label='Tendencia lineal')

# Forecast
ax.plot(fechas_forecast, forecast_values, color='#4CAF50', 
        linewidth=2, label='Forecast Q1 2024')
ax.fill_between(fechas_forecast, forecast_lower, forecast_upper, 
                color='#4CAF50', alpha=0.15, label='Banda de confianza (95%)')

ax.set_xlabel('Fecha')
ax.set_ylabel('Costo diario (USD)')
ax.set_title('NovaTech SRL - Forecast de Gasto Cloud Q1 2024', fontsize=14, fontweight='bold')
ax.legend(loc='upper left')
ax.grid(True, alpha=0.3)

plt.tight_layout()
plt.savefig(DASH_OUT / 'forecast_q1_2024.png', dpi=150, bbox_inches='tight')
plt.show()
print(f"✅ Gráfica guardada: {DASH_OUT / 'forecast_q1_2024.png'}")
```

**Resultado Esperado:**

- Coeficientes de regresión impresos (pendiente y ordenada al origen)
- Tabla con forecast mensual para febrero, marzo y abril 2024
- Gráfica PNG guardada en `~/finops-course/outputs/dashboard/forecast_q1_2024.png`

**Verificación:** La pendiente (`slope`) debe ser positiva si el gasto tiene tendencia creciente, o cercana a cero si es estable.

---

### Paso 5: Generar el Archivo Consolidado de Datos del Tablero

**Objetivo:** Exportar todos los datos calculados a `novatech_dashboard_data.xlsx` con múltiples hojas para alimentar el tablero en LibreOffice Calc.

**Instrucciones:**

1. Crea el log de anomalías y exporta todo:

```python
# Celda 9: Log de anomalías (datos iniciales para el tablero)
anomaly_log = pd.DataFrame({
    'Fecha_Deteccion': ['2024-01-08', '2024-01-15', '2024-01-22'],
    'Servicio': ['EC2 - Backend', 'RDS - Data', 'S3 - Platform'],
    'Monto_Anomalia_USD': [2400, 1800, 950],
    'Desviacion_Porcentual': ['+45%', '+32%', '+28%'],
    'Equipo_Responsable': ['Backend', 'Data', 'Platform'],
    'Responsable': ['Carlos Méndez', 'Luis Paredes', 'María Castillo'],
    'Estado': ['En investigación', 'Resuelto', 'Pendiente triage'],
    'Accion_Tomada': [
        'Escalado a ingeniería - posible auto-scaling descontrolado',
        'Query ineficiente identificada y corregida',
        'Pendiente revisión de lifecycle policies'
    ],
    'Fecha_Resolucion': ['', '2024-01-18', '']
})

print("═══ ANOMALY LOG ═══")
print(anomaly_log[['Fecha_Deteccion', 'Servicio', 'Monto_Anomalia_USD', 'Estado']].to_string(index=False))
```

2. Exporta todo a Excel:

```python
# Celda 10: Exportar archivo consolidado para el tablero
output_path = DASH_OUT / 'novatech_dashboard_data.xlsx'

with pd.ExcelWriter(output_path, engine='openpyxl') as writer:
    # Hoja 1: KPIs Executive Summary
    kpis.to_excel(writer, sheet_name='KPIs', index=False)
    
    # Hoja 2: Gasto por servicio y equipo
    if 'service' in billing_90d.columns:
        gasto_servicio = billing_90d[billing_90d['month'].astype(str) == '2024-01'].groupby('service')['cost'].sum().reset_index()
        gasto_servicio.columns = ['Servicio', 'Gasto_USD']
        gasto_servicio = gasto_servicio.sort_values('Gasto_USD', ascending=False)
    else:
        gasto_servicio = pd.DataFrame({
            'Servicio': ['EC2', 'RDS', 'S3', 'Lambda', 'CloudFront', 'EKS', 'DynamoDB', 'Others'],
            'Gasto_USD': [18500, 12300, 5400, 3200, 2800, 6500, 3100, 3200]
        })
    gasto_servicio.to_excel(writer, sheet_name='Gasto_Servicio', index=False)
    
    # Hoja 3: Budget vs Actuals
    budget_vs_actual.to_excel(writer, sheet_name='Budget_vs_Actual', index=False)
    
    # Hoja 4: Forecast
    forecast_mensual.to_excel(writer, sheet_name='Forecast_Q1', index=False)
    forecast_df.to_excel(writer, sheet_name='Forecast_Diario', index=False)
    
    # Hoja 5: Anomaly Log
    anomaly_log.to_excel(writer, sheet_name='Anomaly_Log', index=False)
    
    # Hoja 6: Serie temporal histórica
    gasto_mensual_export = gasto_mensual[['month_str', 'total_cost']].copy()
    gasto_mensual_export.columns = ['Mes', 'Gasto_Total_USD']
    gasto_mensual_export.to_excel(writer, sheet_name='Historico_Mensual', index=False)

print(f"✅ Archivo consolidado exportado: {output_path}")
print(f"   Hojas: KPIs, Gasto_Servicio, Budget_vs_Actual, Forecast_Q1, Forecast_Diario, Anomaly_Log, Historico_Mensual")
```

**Resultado Esperado:**

```
✅ Archivo consolidado exportado: /home/<usuario>/finops-course/outputs/dashboard/novatech_dashboard_data.xlsx
   Hojas: KPIs, Gasto_Servicio, Budget_vs_Actual, Forecast_Q1, Forecast_Diario, Anomaly_Log, Historico_Mensual
```

**Verificación:** Abre el archivo en LibreOffice Calc y confirma que las 7 hojas existen y contienen datos.

---

### Paso 6: Construir Hoja 1 — Executive Summary con Semáforos RAG

**Objetivo:** Crear la primera hoja del tablero visual en LibreOffice Calc con los KPIs principales y formato condicional RAG.

**Instrucciones:**

1. Abre LibreOffice Calc y crea un nuevo archivo. Guárdalo como `novatech_finops_dashboard_v1.xlsx` en `~/finops-course/outputs/`.

2. Renombra la primera hoja como `Executive Summary`.

3. Construye la estructura del encabezado:

| Celda | Contenido | Formato |
|-------|-----------|---------|
| A1 | `NovaTech SRL - FinOps Dashboard` | Negrita, 18pt, fusionar A1:F1 |
| A2 | `Período: Enero 2024 | Generado: [fecha actual]` | Itálica, 11pt |
| A4 | `KPI` | Negrita, fondo azul oscuro (#1565C0), texto blanco |
| B4 | `Valor` | Negrita, fondo azul oscuro, texto blanco |
| C4 | `Meta` | Negrita, fondo azul oscuro, texto blanco |
| D4 | `Estado` | Negrita, fondo azul oscuro, texto blanco |

4. Ingresa los 6 KPIs en las filas 5 a 10 copiando los valores de la hoja `KPIs` del archivo `novatech_dashboard_data.xlsx`.

5. Aplica formato condicional en la columna D (Estado):
   - Selecciona D5:D10
   - Menú: **Formato → Condicional → Condición**
   - Condición 1: `La celda contiene "GREEN"` → Fondo verde (#4CAF50), texto blanco
   - Condición 2: `La celda contiene "AMBER"` → Fondo naranja (#FF9800), texto blanco
   - Condición 3: `La celda contiene "RED"` → Fondo rojo (#F44336), texto blanco

6. Debajo de los KPIs (fila 12), agrega una sección de resumen:

| Celda | Contenido |
|-------|-----------|
| A12 | `Resumen Ejecutivo:` (Negrita) |
| A13 | `El gasto de enero 2024 muestra un incremento respecto a diciembre.` |
| A14 | `La cobertura de reservas está por debajo del objetivo (70%).` |
| A15 | `Se detectaron 3 anomalías de costo en el período.` |
| A16 | `Acción requerida: Revisar sizing de Backend y optimizar queries de Data.` |

**Resultado Esperado:**

Una hoja con encabezado profesional, tabla de 6 KPIs con semáforos de colores visibles, y un párrafo de resumen ejecutivo.

**Verificación:** Los colores RAG deben ser visualmente distinguibles. Confirma que al menos un indicador está en rojo.

---

### Paso 7: Construir Hoja 2 — Gasto por Servicio y Equipo

**Objetivo:** Crear la segunda hoja con tablas y gráficas de barras que muestren la distribución del gasto.

**Instrucciones:**

1. Crea una nueva hoja llamada `Gasto por Servicio`.

2. Copia los datos de la hoja `Gasto_Servicio` del archivo `novatech_dashboard_data.xlsx` a las celdas A1:B9 (encabezados + 8 servicios).

3. Crea una gráfica de barras horizontales:
   - Selecciona el rango A1:B9
   - Menú: **Insertar → Gráfico**
   - Tipo: **Barras** → Subtipo: **Barras horizontales**
   - Título: `Gasto por Servicio Cloud - Enero 2024 (USD)`
   - Ubica la gráfica en las celdas D1:K16

4. En la celda A12, agrega una segunda tabla con gasto por equipo:

| Equipo | Gasto Real (USD) | % del Total |
|--------|------------------|-------------|
| Backend | 19,200 | 35% |
| Data | 16,800 | 31% |
| Platform | 11,500 | 21% |
| Frontend | 7,500 | 14% |

5. Crea una segunda gráfica (barras verticales) para el gasto por equipo, ubicándola en D18:K32.

6. Aplica formato:
   - Encabezados de tabla con fondo gris claro (#EEEEEE)
   - Valores monetarios con formato `$#,##0`
   - Porcentajes con formato `0%`

**Resultado Esperado:**

Hoja con dos tablas y dos gráficas de barras que muestran claramente dónde se concentra el gasto.

**Verificación:** Las gráficas deben ser legibles y los valores deben sumar el gasto total de enero.

---

### Paso 8: Construir Hoja 3 — Budget vs. Actuals con Alertas

**Objetivo:** Implementar la vista de presupuesto con umbrales de alerta visual al 80% y 100%.

**Instrucciones:**

1. Crea una nueva hoja llamada `Budget vs Actuals`.

2. Copia los datos de `Budget_vs_Actual` del archivo de datos. Estructura la tabla desde A1:

| Equipo | Presupuesto | Gasto Real | % Ejecución | Umbral 80% | Umbral 100% | Estado | Responsable |
|--------|-------------|------------|-------------|-------------|-------------|--------|-------------|

3. En la columna D (% Ejecución), usa la fórmula:
```
=C2/B2*100
```

4. En la columna E (Umbral 80%):
```
=B2*0.8
```

5. En la columna F (Umbral 100%):
```
=B2
```

6. En la columna G (Estado), usa una fórmula condicional:
```
=IF(D2>100,"🔴 SOBRE PRESUPUESTO",IF(D2>80,"🟡 ALERTA PREVENTIVA","🟢 OK"))
```

7. Aplica formato condicional a la columna D:
   - Valor > 100: fondo rojo claro (#FFCDD2)
   - Valor entre 80 y 100: fondo amarillo claro (#FFF9C4)
   - Valor ≤ 80: fondo verde claro (#C8E6C9)

8. Crea una gráfica de barras agrupadas:
   - Selecciona columnas Equipo, Presupuesto y Gasto Real
   - Tipo: Barras agrupadas verticales
   - Título: `Presupuesto vs. Gasto Real por Equipo - Enero 2024`
   - Agrega una línea horizontal en el nivel del presupuesto (serie adicional)

9. Debajo de la gráfica, agrega una nota:
```
Política de alertas NovaTech SRL:
- 🟢 Verde (≤80%): Sin acción requerida
- 🟡 Amarillo (80-100%): Notificación al responsable del equipo
- 🔴 Rojo (>100%): Escalamiento a Director de Tecnología + plan de contención en 48h
```

**Resultado Esperado:**

Hoja con tabla de Budget vs. Actuals, formato condicional por colores, gráfica comparativa y política de alertas documentada.

**Verificación:** Los equipos Backend y Data deben aparecer en rojo (sobre presupuesto).

---

### Paso 9: Construir Hoja 4 — Forecast Q1 2024

**Objetivo:** Presentar la proyección de gasto con datos del modelo de regresión lineal y banda de confianza.

**Instrucciones:**

1. Crea una nueva hoja llamada `Forecast Q1 2024`.

2. En A1:A3, escribe el contexto del modelo:

| Celda | Contenido |
|-------|-----------|
| A1 | `Modelo de Proyección: Regresión Lineal (numpy polyfit grado 1)` |
| A2 | `Datos base: 92 días (Oct 2023 - Ene 2024)` |
| A3 | `Banda de confianza: ±1.96 desviaciones estándar (95%)` |

3. En A5:D8, crea la tabla de forecast mensual:

| Mes | Forecast (USD) | Banda Inferior | Banda Superior |
|-----|----------------|----------------|----------------|
| 2024-02 | [valor] | [valor] | [valor] |
| 2024-03 | [valor] | [valor] | [valor] |
| 2024-04 | [valor] | [valor] | [valor] |

Copia los valores de la hoja `Forecast_Q1` del archivo de datos.

4. En la celda A10, calcula el total:
```
=SUM(B6:B8)
```
Con etiqueta: `Total Q1 2024 Proyectado:`

5. Inserta la imagen `forecast_q1_2024.png` generada en el Paso 4:
   - Menú: **Insertar → Imagen**
   - Selecciona: `~/finops-course/outputs/dashboard/forecast_q1_2024.png`
   - Ubícala en el rango E1:N20

6. Debajo de la tabla (fila 12), agrega notas de interpretación:
```
Interpretación:
- La tendencia lineal indica un crecimiento de ~$[slope*30]USD/mes
- Si no se toman acciones de optimización, el gasto Q1 alcanzará $[total]
- Recomendación: implementar rightsizing en Backend para reducir la pendiente
- Riesgo: si se mantiene la tendencia, el presupuesto Q1 ($159,000) se excederá en [%]
```

**Resultado Esperado:**

Hoja con metadatos del modelo, tabla de forecast mensual, gráfica incrustada y notas de interpretación.

**Verificación:** La suma de los 3 meses proyectados debe ser coherente con la tendencia observada (mayor que el gasto Q4 si la tendencia es creciente).

---

### Paso 10: Construir Hoja 5 — Anomaly Log con Flujo de Triage

**Objetivo:** Diseñar el registro de anomalías con campos de triage, estados y seguimiento.

**Instrucciones:**

1. Crea una nueva hoja llamada `Anomaly Log`.

2. En A1, escribe el título: `Registro de Anomalías de Costo - NovaTech SRL` (Negrita, 14pt).

3. En A3:I3, crea los encabezados de la tabla:

| Fecha Detección | Servicio | Monto Anomalía (USD) | Desviación % | Equipo | Responsable | Estado | Acción Tomada | Fecha Resolución |

4. Formatea los encabezados con fondo naranja oscuro (#E65100) y texto blanco.

5. Copia las 3 anomalías del archivo `novatech_dashboard_data.xlsx` (hoja `Anomaly_Log`).

6. Aplica formato condicional a la columna G (Estado):
   - `Pendiente triage` → Fondo rojo claro
   - `En investigación` → Fondo amarillo
   - `Resuelto` → Fondo verde claro

7. En la fila 8 (después de los datos), deja una fila vacía y en A9 escribe: `[Agregar nuevas anomalías aquí]` en itálica gris.

8. En la sección inferior (A11:E16), documenta el flujo de triage:

```
FLUJO DE TRIAGE DE ANOMALÍAS:

1. DETECCIÓN → Alerta automática cuando gasto diario > media + 2σ
2. TRIAGE → FinOps lead asigna prioridad (P1/P2/P3) y responsable en <4h
3. INVESTIGACIÓN → Responsable identifica causa raíz en <24h (P1) o <72h (P2/P3)
4. RESOLUCIÓN → Implementar corrección y documentar en este log
5. POST-MORTEM → Revisar en reunión semanal FinOps si monto > $1,000

Estados válidos: Pendiente triage | En investigación | Resuelto | Falso positivo
SLA de resolución: P1 (<$5K) = 24h | P2 ($1K-$5K) = 72h | P3 (<$1K) = 1 semana
```

9. Agrega una tabla de resumen de anomalías:

| Métrica | Valor |
|---------|-------|
| Total anomalías detectadas (enero) | 3 |
| Monto total en anomalías | $5,150 |
| Anomalías resueltas | 1 (33%) |
| Anomalías pendientes | 2 (67%) |
| Tiempo promedio de resolución | 3 días |

**Resultado Esperado:**

Hoja con tabla de anomalías formateada, flujo de triage documentado, y métricas de gestión de anomalías.

**Verificación:** Los tres estados (Pendiente, En investigación, Resuelto) deben tener colores distintos visibles.

---

### Paso 11: Revisión Final y Guardado del Tablero

**Objetivo:** Verificar la integridad del tablero completo y guardar el entregable final.

**Instrucciones:**

1. Navega por las 5 hojas del archivo y verifica:

| Hoja | Verificación |
|------|-------------|
| Executive Summary | 6 KPIs con semáforos RAG visibles |
| Gasto por Servicio | 2 gráficas de barras legibles |
| Budget vs Actuals | Formato condicional + gráfica comparativa |
| Forecast Q1 2024 | Tabla + gráfica PNG incrustada |
| Anomaly Log | 3 registros + flujo de triage documentado |

2. Ajusta el ancho de columnas para que todo sea legible sin scroll horizontal (Ctrl+A → Formato → Columna → Ancho óptimo).

3. Configura la hoja `Executive Summary` como la hoja activa al abrir (debe ser la primera que se ve).

4. Guarda el archivo final:
   - **Archivo → Guardar como**
   - Nombre: `novatech_finops_dashboard_v1.xlsx`
   - Ubicación: `~/finops-course/outputs/`
   - Formato: `.xlsx` (Microsoft Excel 2007-365)

5. Verifica el tamaño del archivo (debería estar entre 200 KB y 2 MB dependiendo de la imagen incrustada).

```bash
# Verificación desde terminal
ls -lh ~/finops-course/outputs/novatech_finops_dashboard_v1.xlsx
```

**Resultado Esperado:**

```
-rw-r--r-- 1 usuario usuario 850K [fecha] novatech_finops_dashboard_v1.xlsx
```

**Verificación:** Cierra y vuelve a abrir el archivo para confirmar que todos los formatos, gráficas y la imagen se preservaron correctamente.

---

## Validación y Pruebas

Ejecuta la siguiente lista de verificación para confirmar que el laboratorio está completo:

| # | Criterio de Validación | ✅/❌ |
|---|------------------------|-------|
| 1 | El notebook `lab_04_01_dashboard_data_prep.ipynb` ejecuta sin errores de principio a fin | |
| 2 | El archivo `novatech_dashboard_data.xlsx` existe con 7 hojas | |
| 3 | La gráfica `forecast_q1_2024.png` existe en `outputs/dashboard/` | |
| 4 | El tablero `novatech_finops_dashboard_v1.xlsx` tiene exactamente 5 hojas | |
| 5 | La hoja Executive Summary muestra 6 KPIs con colores RAG | |
| 6 | La hoja Budget vs Actuals tiene al menos un equipo en estado RED | |
| 7 | El forecast muestra valores para 3 meses futuros con banda de confianza | |
| 8 | El Anomaly Log tiene 3 registros con estados diferenciados por color | |
| 9 | El flujo de triage está documentado con SLAs definidos | |
| 10 | Todas las gráficas son legibles y tienen títulos descriptivos | |

### Script de Validación Automatizada (opcional)

Ejecuta en una celda nueva del notebook:

```python
# Celda de validación
from pathlib import Path
import openpyxl

checks = []

# Check 1: Archivo de datos existe
f1 = DASH_OUT / 'novatech_dashboard_data.xlsx'
checks.append(("Archivo dashboard_data.xlsx", f1.exists()))

# Check 2: Gráfica existe
f2 = DASH_OUT / 'forecast_q1_2024.png'
checks.append(("Gráfica forecast PNG", f2.exists()))

# Check 3: Hojas del archivo de datos
if f1.exists():
    wb = openpyxl.load_workbook(f1, read_only=True)
    checks.append(("7 hojas en dashboard_data", len(wb.sheetnames) >= 6))
    wb.close()

# Check 4: Tablero final existe
f3 = OUTPUTS / 'novatech_finops_dashboard_v1.xlsx'
checks.append(("Tablero final existe", f3.exists()))

# Check 5: Hojas del tablero
if f3.exists():
    wb = openpyxl.load_workbook(f3, read_only=True)
    expected_sheets = ['Executive Summary', 'Gasto por Servicio', 'Budget vs Actuals', 
                       'Forecast Q1 2024', 'Anomaly Log']
    for sheet in expected_sheets:
        checks.append((f"Hoja '{sheet}'", sheet in wb.sheetnames))
    wb.close()

print("═══ VALIDACIÓN DEL LABORATORIO ═══")
all_pass = True
for name, passed in checks:
    status = "✅" if passed else "❌"
    if not passed:
        all_pass = False
    print(f"  {status} {name}")

print(f"\n{'🎉 LABORATORIO COMPLETADO' if all_pass else '⚠️ Revisa los items marcados con ❌'}")
```

---

## Resolución de Problemas

### Problema 1: Error `FileNotFoundError` al cargar archivos de laboratorios previos

**Síntomas:** Al ejecutar la Celda 2, Python lanza `FileNotFoundError: [Errno 2] No such file or directory: '.../novatech_billing_summary_jan2024.xlsx'`.

**Causa:** Los archivos de salida de laboratorios anteriores no se encuentran en las rutas esperadas. Esto ocurre si el estudiante no completó un lab previo, guardó los archivos con otro nombre, o los ubicó en un directorio diferente.

**Solución:**

```bash
# 1. Verificar qué archivos existen
find ~/finops-course/ -name "novatech_*" -type f

# 2. Si los archivos están en otra ubicación, copiarlos:
cp ~/finops-course/[ruta_encontrada]/novatech_billing_summary_jan2024.xlsx ~/finops-course/outputs/

# 3. Si no existen, solicitar al instructor los archivos de referencia
# y colocarlos en ~/finops-course/outputs/ y ~/finops-course/data/processed/

# 4. Alternativa: modificar las rutas en el notebook
# Cambiar la línea:
# billing_jan = pd.read_excel(OUTPUTS / 'novatech_billing_summary_jan2024.xlsx')
# por la ruta donde realmente está el archivo
```

### Problema 2: La imagen PNG no se incrusta correctamente en LibreOffice Calc

**Síntomas:** Al insertar la imagen `forecast_q1_2024.png` en la hoja Forecast Q1 2024, la imagen aparece distorsionada, no se muestra, o desaparece al guardar como `.xlsx`.

**Causa:** LibreOffice Calc puede tener problemas con imágenes de alta resolución al exportar a formato `.xlsx`, o la ruta de la imagen se pierde si se usó una referencia relativa.

**Solución:**

1. Asegúrate de que la imagen se **incrusta** (no se enlaza):
   - Al insertar: Menú **Insertar → Imagen** → selecciona el archivo
   - **NO** marcar la opción "Enlazar" si aparece

2. Si la imagen se distorsiona, redimensiónala manteniendo la proporción:
   - Clic derecho sobre la imagen → **Posición y tamaño**
   - Marca "Mantener proporción"
   - Ajusta el ancho a ~15 cm

3. Si desaparece al guardar como `.xlsx`:
   - Guarda primero en formato nativo `.ods`
   - Luego usa **Archivo → Guardar una copia** en formato `.xlsx`
   - Verifica reabriendo el `.xlsx`

4. Alternativa si persiste el problema: en lugar de incrustar la imagen, crea la gráfica directamente en LibreOffice Calc usando los datos de la hoja `Forecast_Diario` del archivo de datos.

---

## Limpieza

Este laboratorio genera el entregable final del curso, por lo que **no se recomienda eliminar archivos**. Sin embargo, si necesitas liberar espacio:

```bash
# Solo eliminar archivos intermedios (NO el tablero final)
# rm ~/finops-course/outputs/dashboard/novatech_dashboard_data.xlsx  # Intermedio

# Mantener obligatoriamente:
# ~/finops-course/outputs/novatech_finops_dashboard_v1.xlsx  ← ENTREGABLE FINAL
# ~/finops-course/outputs/dashboard/forecast_q1_2024.png     ← Respaldo de gráfica
# ~/finops-course/notebooks/lab_04_01_dashboard_data_prep.ipynb ← Código fuente
```

Para detener Jupyter Notebook:
```bash
# En la terminal donde se ejecuta Jupyter, presionar Ctrl+C dos veces
# O desde el navegador: File → Shut Down
```

---

## Resumen

### Lo que Construiste

En este laboratorio culminante integraste todos los conocimientos y artefactos del curso para crear un tablero FinOps funcional para NovaTech SRL:

| Componente | Técnica Utilizada | Herramienta |
|------------|-------------------|-------------|
| KPIs con semáforos RAG | Lógica condicional + formato visual | Python → LibreOffice |
| Budget vs. Actuals | Umbrales 80%/100% con alertas | Fórmulas condicionales |
| Forecast Q1 2024 | Regresión lineal (`np.polyfit` grado 1) | Python (numpy) |
| Banda de confianza | ±1.96σ de residuos | Python (numpy) |
| Anomaly Log | Flujo de triage con SLAs | Diseño de proceso |

### Conceptos Clave Aplicados

- **Pirámide de reportes FinOps**: El tablero integra las tres capas (operativa, táctica, estratégica) en un solo artefacto con hojas diferenciadas por audiencia.
- **Transformación de datos técnicos a decisiones**: Los datos brutos de billing se convirtieron en KPIs accionables con contexto, metas y llamadas a la acción.
- **Modelo RAG**: Semáforos verde/amarillo/rojo permiten identificar instantáneamente las áreas que requieren atención.
- **Forecasting cuantitativo**: La regresión lineal simple proporciona una base objetiva para la planificación financiera del próximo trimestre.

### Recursos Adicionales

- [FinOps Foundation — KPIs and Metrics](https://www.finops.org/framework/kpis/)
- [FinOps Foundation — Forecasting Capability](https://www.finops.org/framework/capabilities/forecasting/)
- [FinOps Foundation — Anomaly Management](https://www.finops.org/framework/capabilities/anomaly-management/)
- [LibreOffice Calc — Documentación de formato condicional](https://help.libreoffice.org/latest/es/text/scalc/01/05120000.html)
- [NumPy — numpy.polyfit documentation](https://numpy.org/doc/stable/reference/generated/numpy.polyfit.html)

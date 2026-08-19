# Demostración 1. Lectura de un reporte de consumo cloud

## Metadata

| Campo | Valor |
|-------|-------|
| **Duración** | 25 minutos |
| **Complejidad** | Media |
| **Nivel Bloom** | Aplicar |
| **Módulo** | 2 — Reportes de consumo cloud |
| **Organización ficticia** | NovaTech SRL |

---

## Visión General

En este laboratorio trabajarás con un dataset sintético de facturación multicloud (`novatech_billing_raw.csv`) que contiene 500 registros correspondientes a enero 2024. Cargarás el archivo en un Jupyter Notebook, identificarás los campos clave del estándar FOCUS 1.0, diferenciarás tipos de costo (on-demand, amortizado, reservas) y calcularás métricas financieras básicas por servicio y proveedor. El resultado será un archivo Excel de resumen (`novatech_billing_summary_jan2024.xlsx`) que servirá como insumo para los laboratorios posteriores.

---

## Objetivos de Aprendizaje

- [ ] Interpretar correctamente los campos y métricas de un reporte de facturación cloud usando terminología FinOps estándar
- [ ] Diferenciar entre consumo, gasto, uso, costo amortizado, costo neto y costos de reservas en un dataset de facturación cloud
- [ ] Aplicar los principios de normalización FOCUS para estandarizar datos de costos provenientes de múltiples proveedores cloud
- [ ] Calcular métricas básicas de costo (total por servicio, costo promedio diario, participación porcentual) usando pandas sobre datos reales

---

## Prerrequisitos

### Conocimiento previo

| Requisito | Descripción |
|-----------|-------------|
| Lab 01-00-01 completado | Contexto organizacional de NovaTech SRL establecido |
| Pandas básico | Operaciones `read_csv`, `groupby`, `sum`, `describe` |
| Conceptos FinOps Módulo 2 | Costo amortizado, reservas, modelos IaaS/PaaS/SaaS |
| Python básico | Variables, funciones, f-strings |

### Acceso y archivos requeridos

| Recurso | Ubicación |
|---------|-----------|
| Dataset de facturación | `~/finops-course/data/raw/novatech_billing_raw.csv` |
| Notebook pre-estructurado | `~/finops-course/notebooks/lab_02_01_billing_reader.ipynb` |
| Archivo requirements.txt | `~/finops-course/requirements.txt` |

---

## Entorno de Laboratorio

### Hardware mínimo

| Componente | Especificación |
|------------|---------------|
| Procesador | Intel Core i5 8ª gen / AMD Ryzen 5 3000 (4 núcleos) |
| RAM | 8 GB DDR4 mínimo |
| Almacenamiento | 10 GB libres en SSD |
| Pantalla | 1366×768 mínimo (recomendado 1920×1080) |
| Internet | 10 Mbps |

### Software requerido

| Herramienta | Versión |
|-------------|---------|
| Python | 3.12.1 |
| pandas | 2.2.1 |
| numpy | 1.26.4 |
| openpyxl | 3.1.2 |
| Jupyter Notebook | 7.1.2 |

### Configuración inicial del entorno

Ejecuta estos comandos **solo si no lo hiciste en el Lab 01-00-01**:

```bash
# Crear estructura de directorios
mkdir -p ~/finops-course/data/raw
mkdir -p ~/finops-course/data/processed
mkdir -p ~/finops-course/notebooks
mkdir -p ~/finops-course/outputs

# Verificar requirements.txt
cat ~/finops-course/requirements.txt
```

Contenido esperado de `requirements.txt`:

```
pandas==2.2.1
numpy==1.26.4
openpyxl==3.1.2
matplotlib==3.8.3
notebook==7.1.2
```

```bash
# Instalar dependencias (si no se hizo previamente)
pip install -r ~/finops-course/requirements.txt

# Verificar versiones
python -c "import pandas; print(f'pandas: {pandas.__version__}')"
python -c "import numpy; print(f'numpy: {numpy.__version__}')"
```

---

## Paso a Paso

### Paso 1: Generar el dataset sintético de facturación

**Objetivo:** Crear el archivo `novatech_billing_raw.csv` con 500 registros de facturación multicloud que sigan parcialmente el estándar FOCUS 1.0.

> **Nota:** Si el instructor ya proporcionó este archivo en `~/finops-course/data/raw/`, puedes omitir este paso y pasar directamente al Paso 2. Verifica con `ls ~/finops-course/data/raw/novatech_billing_raw.csv`.

**Instrucciones:**

1. Abre una terminal y navega al directorio de trabajo:

```bash
cd ~/finops-course/data/raw
```

2. Crea un script Python para generar el dataset sintético:

```bash
cat > generate_billing_data.py << 'EOF'
import pandas as pd
import numpy as np
from datetime import datetime, timedelta

np.random.seed(42)

# Configuración NovaTech SRL
providers = ['AWS', 'Azure', 'GCP']
provider_weights = [0.50, 0.30, 0.20]

services_by_provider = {
    'AWS': {
        'IaaS': ['Amazon EC2', 'Amazon EBS'],
        'PaaS': ['Amazon RDS', 'AWS Lambda', 'Amazon S3'],
        'SaaS': ['Amazon WorkSpaces']
    },
    'Azure': {
        'IaaS': ['Azure Virtual Machines', 'Azure Disk Storage'],
        'PaaS': ['Azure SQL Database', 'Azure Functions', 'Azure Blob Storage'],
        'SaaS': ['Microsoft 365']
    },
    'GCP': {
        'IaaS': ['Compute Engine', 'Persistent Disk'],
        'PaaS': ['Cloud SQL', 'Cloud Functions', 'Cloud Storage'],
        'SaaS': ['Google Workspace']
    }
}

products = ['PayCore', 'AnalyticsHub', 'DevPortal']
teams = ['Backend', 'Frontend', 'Data', 'Platform']
environments = ['prod', 'staging', 'dev']
env_weights = [0.60, 0.25, 0.15]

regions = {
    'AWS': ['us-east-1', 'eu-west-1', 'sa-east-1'],
    'Azure': ['eastus', 'westeurope', 'brazilsouth'],
    'GCP': ['us-central1', 'europe-west1', 'southamerica-east1']
}

charge_types = ['Usage', 'Usage', 'Usage', 'Usage', 'Purchase', 'Tax', 'Credit']
usage_types = ['OnDemand', 'OnDemand', 'OnDemand', 'Reserved', 'Reserved', 'SpotInstance']

# Generar 500 registros
n_records = 500
records = []

start_date = datetime(2024, 1, 1)
end_date = datetime(2024, 1, 31)

for i in range(n_records):
    provider = np.random.choice(providers, p=provider_weights)
    
    # Seleccionar modelo y servicio
    model = np.random.choice(['IaaS', 'PaaS', 'SaaS'], p=[0.45, 0.40, 0.15])
    service_list = services_by_provider[provider][model]
    service = np.random.choice(service_list)
    
    # Fecha aleatoria en enero 2024
    day_offset = np.random.randint(0, 31)
    billing_date = start_date + timedelta(days=day_offset)
    
    # Producto, equipo, ambiente
    product = np.random.choice(products)
    team = np.random.choice(teams)
    environment = np.random.choice(environments, p=env_weights)
    
    # Región
    region = np.random.choice(regions[provider])
    
    # Tipo de cargo y uso
    charge_type = np.random.choice(charge_types)
    usage_type = np.random.choice(usage_types) if charge_type == 'Usage' else 'N/A'
    
    # Costos base según modelo
    if model == 'IaaS':
        base_cost = np.random.uniform(5.0, 250.0)
    elif model == 'PaaS':
        base_cost = np.random.uniform(2.0, 180.0)
    else:  # SaaS
        base_cost = np.random.uniform(10.0, 50.0)
    
    # Ajuste por ambiente
    if environment == 'prod':
        base_cost *= 1.5
    elif environment == 'staging':
        base_cost *= 0.7
    else:
        base_cost *= 0.4
    
    # Calcular diferentes tipos de costo
    billed_cost = round(base_cost, 4)
    
    # Costo efectivo (con descuentos aplicados)
    if usage_type == 'Reserved':
        effective_cost = round(billed_cost * 0.65, 4)  # ~35% descuento
    elif usage_type == 'SpotInstance':
        effective_cost = round(billed_cost * 0.30, 4)  # ~70% descuento
    else:
        effective_cost = billed_cost
    
    # Costo amortizado (distribuye compromisos)
    if charge_type == 'Purchase':
        amortized_cost = round(billed_cost / 12, 4)  # Amortizado a 12 meses
    elif usage_type == 'Reserved':
        amortized_cost = round(effective_cost * 1.05, 4)  # Incluye amortización
    else:
        amortized_cost = effective_cost
    
    # Costo neto (después de créditos)
    if charge_type == 'Credit':
        net_cost = round(-abs(billed_cost) * 0.1, 4)
        billed_cost = round(-abs(billed_cost) * 0.1, 4)
        effective_cost = net_cost
        amortized_cost = net_cost
    elif charge_type == 'Tax':
        net_cost = round(billed_cost * 0.21, 4)
        billed_cost = net_cost
        effective_cost = net_cost
        amortized_cost = net_cost
    else:
        net_cost = amortized_cost
    
    # Resource ID
    resource_id = f"{provider.lower()}-{service.lower().replace(' ', '-')[:10]}-{i:04d}"
    
    # Introducir algunas inconsistencias intencionales
    # ~5% de registros con campos vacíos en tags
    tag_product = product if np.random.random() > 0.05 else ''
    tag_team = team if np.random.random() > 0.05 else ''
    tag_environment = environment if np.random.random() > 0.03 else ''
    
    # ~3% con formatos de fecha inconsistentes (ya corregido en generación)
    # ~2% con costos negativos legítimos (créditos)
    
    records.append({
        'InvoiceId': f"INV-2024-01-{provider[:3]}-{i:04d}",
        'BillingPeriodStart': '2024-01-01',
        'BillingPeriodEnd': '2024-01-31',
        'BillingDate': billing_date.strftime('%Y-%m-%d'),
        'Provider': provider,
        'ServiceName': service,
        'ServiceModel': model,
        'ResourceId': resource_id,
        'Region': region,
        'ChargeType': charge_type,
        'UsageType': usage_type,
        'UsageQuantity': round(np.random.uniform(1, 744), 2),
        'UsageUnit': 'Hours' if model == 'IaaS' else ('Requests' if 'Lambda' in service or 'Functions' in service else 'GB-Month'),
        'BilledCost': billed_cost,
        'EffectiveCost': effective_cost,
        'AmortizedCost': amortized_cost,
        'NetCost': net_cost,
        'Currency': 'USD',
        'TagProduct': tag_product,
        'TagTeam': tag_team,
        'TagEnvironment': tag_environment
    })

df = pd.DataFrame(records)
df.to_csv('novatech_billing_raw.csv', index=False)
print(f"Dataset generado: {len(df)} registros")
print(f"Proveedores: {df['Provider'].value_counts().to_dict()}")
print(f"Modelos: {df['ServiceModel'].value_counts().to_dict()}")
print(f"Archivo guardado en: novatech_billing_raw.csv")
EOF

python generate_billing_data.py
```

**Salida esperada:**

```
Dataset generado: 500 registros
Proveedores: {'AWS': 262, 'Azure': 138, 'GCP': 100}
Modelos: {'IaaS': 224, 'PaaS': 200, 'SaaS': 76}
Archivo guardado en: novatech_billing_raw.csv
```

**Verificación:**

```bash
wc -l novatech_billing_raw.csv
# Esperado: 501 (500 registros + 1 header)

head -2 novatech_billing_raw.csv
```

---

### Paso 2: Iniciar Jupyter Notebook y crear el notebook de análisis

**Objetivo:** Abrir el entorno Jupyter y preparar el notebook de trabajo para el análisis de facturación.

**Instrucciones:**

1. Inicia Jupyter Notebook en el puerto designado:

```bash
cd ~/finops-course
jupyter notebook --port=8888 --notebook-dir=~/finops-course/notebooks/
```

2. En el navegador, accede a `http://localhost:8888` y crea un nuevo notebook Python 3 con el nombre `lab_02_01_billing_reader.ipynb`.

3. En la **primera celda** del notebook, escribe el encabezado y las importaciones:

```python
# =============================================================
# Lab 02-00-01: Lectura de un reporte de consumo cloud
# Organización: NovaTech SRL
# Período: Enero 2024
# Dataset: novatech_billing_raw.csv (500 registros multicloud)
# =============================================================

import pandas as pd
import numpy as np
from pathlib import Path

# Configuración de visualización
pd.set_option('display.max_columns', None)
pd.set_option('display.float_format', '{:.2f}'.format)

# Rutas
DATA_RAW = Path('../data/raw')
DATA_PROCESSED = Path('../data/processed')
OUTPUTS = Path('../outputs')

print(f"pandas version: {pd.__version__}")
print(f"numpy version: {np.__version__}")
```

4. Ejecuta la celda con `Shift + Enter`.

**Salida esperada:**

```
pandas version: 2.2.1
numpy version: 1.26.4
```

**Verificación:** Ambas versiones deben coincidir exactamente con las especificadas.

---

### Paso 3: Cargar y explorar la estructura del dataset

**Objetivo:** Cargar el CSV de facturación, identificar los campos FOCUS 1.0 y comprender la estructura del reporte.

**Instrucciones:**

1. En una **nueva celda**, carga el dataset:

```python
# Cargar dataset de facturación NovaTech SRL - Enero 2024
df = pd.read_csv(DATA_RAW / 'novatech_billing_raw.csv')

print(f"Registros cargados: {df.shape[0]}")
print(f"Columnas: {df.shape[1]}")
print(f"\n{'='*60}")
print("ESTRUCTURA DEL REPORTE DE FACTURACIÓN CLOUD")
print(f"{'='*60}")
print(f"\nColumnas disponibles:")
for i, col in enumerate(df.columns, 1):
    print(f"  {i:2d}. {col}")
```

2. En la **siguiente celda**, examina los tipos de datos y valores nulos:

```python
# Inspección de tipos de datos y completitud
print("TIPOS DE DATOS Y VALORES NULOS")
print("="*60)
info_df = pd.DataFrame({
    'Tipo': df.dtypes,
    'No Nulos': df.count(),
    'Nulos': df.isnull().sum(),
    '% Completitud': ((df.count() / len(df)) * 100).round(1)
})
print(info_df.to_string())
```

3. En otra celda, muestra las primeras filas para inspección visual:

```python
# Vista previa de los datos
print("\nPRIMEROS 5 REGISTROS:")
print("="*60)
df.head()
```

**Salida esperada:**

```
Registros cargados: 500
Columnas: 21

============================================================
ESTRUCTURA DEL REPORTE DE FACTURACIÓN CLOUD
============================================================

Columnas disponibles:
   1. InvoiceId
   2. BillingPeriodStart
   3. BillingPeriodEnd
   4. BillingDate
   5. Provider
   6. ServiceName
   7. ServiceModel
   8. ResourceId
   9. Region
  10. ChargeType
  11. UsageType
  12. UsageQuantity
  13. UsageUnit
  14. BilledCost
  15. EffectiveCost
  16. AmortizedCost
  17. NetCost
  18. Currency
  19. TagProduct
  20. TagTeam
  21. TagEnvironment
```

**Verificación:** Confirma que se cargaron exactamente 500 registros y 21 columnas. Las columnas de tags (`TagProduct`, `TagTeam`, `TagEnvironment`) deben mostrar algunos valores nulos (~3-5%).

---

### Paso 4: Interpretar los campos FOCUS 1.0 y tipos de costo

**Objetivo:** Documentar y diferenciar los campos de costo según la terminología FinOps: BilledCost, EffectiveCost, AmortizedCost y NetCost.

**Instrucciones:**

1. Crea una celda de tipo **Markdown** con la documentación de campos:

```markdown
## Glosario de Campos FOCUS 1.0 en el Dataset

| Campo | Definición FinOps | Uso en análisis |
|-------|-------------------|-----------------|
| **BilledCost** | Monto facturado por el proveedor antes de descuentos | Conciliación con factura real |
| **EffectiveCost** | Costo después de aplicar descuentos (reservas, spot) | Costo real pagado |
| **AmortizedCost** | Costo con compromisos distribuidos en el tiempo | Análisis de tendencias |
| **NetCost** | Costo final después de créditos y ajustes | Presupuesto y forecast |
| **ChargeType** | Tipo de cargo: Usage, Purchase, Tax, Credit | Clasificación de gastos |
| **UsageType** | Modelo de pricing: OnDemand, Reserved, SpotInstance | Análisis de cobertura |
| **ServiceModel** | Clasificación IaaS/PaaS/SaaS | Estrategia de optimización |
```

2. En una celda de código, analiza la distribución de tipos de costo:

```python
# ANÁLISIS DE TIPOS DE COSTO
print("DISTRIBUCIÓN POR TIPO DE CARGO (ChargeType)")
print("="*60)
charge_dist = df['ChargeType'].value_counts()
charge_pct = (charge_dist / len(df) * 100).round(1)
for charge, count in charge_dist.items():
    print(f"  {charge:<12} → {count:>4} registros ({charge_pct[charge]}%)")

print(f"\n{'='*60}")
print("DISTRIBUCIÓN POR TIPO DE USO (UsageType)")
print("="*60)
usage_dist = df[df['ChargeType'] == 'Usage']['UsageType'].value_counts()
usage_pct = (usage_dist / usage_dist.sum() * 100).round(1)
for usage, count in usage_dist.items():
    print(f"  {usage:<14} → {count:>4} registros ({usage_pct[usage]}%)")
```

3. Compara las diferentes métricas de costo:

```python
# COMPARACIÓN DE MÉTRICAS DE COSTO
print("\nCOMPARACIÓN DE MÉTRICAS DE COSTO (solo registros Usage)")
print("="*60)

usage_df = df[df['ChargeType'] == 'Usage'].copy()

cost_comparison = pd.DataFrame({
    'Métrica': ['BilledCost', 'EffectiveCost', 'AmortizedCost', 'NetCost'],
    'Total (USD)': [
        usage_df['BilledCost'].sum(),
        usage_df['EffectiveCost'].sum(),
        usage_df['AmortizedCost'].sum(),
        usage_df['NetCost'].sum()
    ]
})
cost_comparison['Diferencia vs Billed'] = (
    (cost_comparison['Total (USD)'] - cost_comparison['Total (USD)'].iloc[0]) 
    / cost_comparison['Total (USD)'].iloc[0] * 100
).round(2)

print(cost_comparison.to_string(index=False))
print(f"\n→ El EffectiveCost es menor que BilledCost debido a descuentos")
print(f"  por reservas ({usage_df[usage_df['UsageType']=='Reserved'].shape[0]} registros)")
print(f"  y spot instances ({usage_df[usage_df['UsageType']=='SpotInstance'].shape[0]} registros)")
```

**Salida esperada (valores aproximados):**

```
DISTRIBUCIÓN POR TIPO DE CARGO (ChargeType)
============================================================
  Usage        → ~285 registros (~57%)
  Purchase     →  ~72 registros (~14%)
  Tax          →  ~72 registros (~14%)
  Credit       →  ~71 registros (~14%)

COMPARACIÓN DE MÉTRICAS DE COSTO (solo registros Usage)
============================================================
     Métrica  Total (USD)  Diferencia vs Billed
  BilledCost     XXXXX.XX                  0.00
EffectiveCost    XXXXX.XX                -XX.XX
AmortizedCost    XXXXX.XX                -XX.XX
      NetCost    XXXXX.XX                -XX.XX
```

**Verificación:** El `EffectiveCost` total debe ser menor que el `BilledCost` total (los descuentos por reservas y spot reducen el costo efectivo). El `AmortizedCost` debe estar entre ambos valores.

---

### Paso 5: Identificar y corregir inconsistencias de formato

**Objetivo:** Detectar problemas de calidad en los datos (tags vacíos, valores inconsistentes) y aplicar correcciones básicas de normalización.

**Instrucciones:**

1. Diagnostica la calidad de los datos:

```python
# DIAGNÓSTICO DE CALIDAD DE DATOS
print("ANÁLISIS DE CALIDAD - TAGS DE ASIGNACIÓN")
print("="*60)

tag_cols = ['TagProduct', 'TagTeam', 'TagEnvironment']
for col in tag_cols:
    empty_count = (df[col] == '').sum() + df[col].isnull().sum()
    coverage = ((len(df) - empty_count) / len(df) * 100).round(1)
    print(f"  {col:<18} → Cobertura: {coverage}% | Sin etiquetar: {empty_count} registros")

print(f"\n{'='*60}")
print("REGISTROS CON COSTOS NEGATIVOS (Créditos)")
print("="*60)
negative_costs = df[df['BilledCost'] < 0]
print(f"  Total registros con costo negativo: {len(negative_costs)}")
print(f"  Suma de créditos: ${negative_costs['BilledCost'].sum():,.2f}")
print(f"  Tipos de cargo: {negative_costs['ChargeType'].unique().tolist()}")
```

2. Aplica correcciones y normalización:

```python
# CORRECCIONES Y NORMALIZACIÓN
print("APLICANDO CORRECCIONES...")
print("="*60)

# 1. Reemplazar strings vacíos por NaN para manejo consistente
df_clean = df.copy()
df_clean[tag_cols] = df_clean[tag_cols].replace('', np.nan)

# 2. Asegurar tipos numéricos en columnas de costo
cost_cols = ['BilledCost', 'EffectiveCost', 'AmortizedCost', 'NetCost', 'UsageQuantity']
for col in cost_cols:
    df_clean[col] = pd.to_numeric(df_clean[col], errors='coerce')

# 3. Convertir fechas a datetime
df_clean['BillingDate'] = pd.to_datetime(df_clean['BillingDate'])
df_clean['BillingPeriodStart'] = pd.to_datetime(df_clean['BillingPeriodStart'])
df_clean['BillingPeriodEnd'] = pd.to_datetime(df_clean['BillingPeriodEnd'])

# 4. Crear columna de día para análisis temporal
df_clean['BillingDay'] = df_clean['BillingDate'].dt.day

# Verificación post-limpieza
print(f"  ✓ Tags vacíos convertidos a NaN")
print(f"  ✓ Columnas de costo validadas como numéricas")
print(f"  ✓ Fechas convertidas a datetime64")
print(f"  ✓ Columna BillingDay creada")
print(f"\n  Registros finales: {len(df_clean)}")
print(f"  Tipos de datos actualizados:")
print(f"    BillingDate: {df_clean['BillingDate'].dtype}")
print(f"    BilledCost: {df_clean['BilledCost'].dtype}")
```

**Salida esperada:**

```
ANÁLISIS DE CALIDAD - TAGS DE ASIGNACIÓN
============================================================
  TagProduct         → Cobertura: ~95% | Sin etiquetar: ~25 registros
  TagTeam            → Cobertura: ~95% | Sin etiquetar: ~25 registros
  TagEnvironment     → Cobertura: ~97% | Sin etiquetar: ~15 registros

APLICANDO CORRECCIONES...
============================================================
  ✓ Tags vacíos convertidos a NaN
  ✓ Columnas de costo validadas como numéricas
  ✓ Fechas convertidas a datetime64
  ✓ Columna BillingDay creada

  Registros finales: 500
  Tipos de datos actualizados:
    BillingDate: datetime64[ns]
    BilledCost: float64
```

**Verificación:** Los tags deben tener ~95-97% de cobertura. Las columnas de costo deben ser `float64` y las fechas `datetime64[ns]`.

---

### Paso 6: Calcular métricas de costo por servicio y proveedor

**Objetivo:** Generar tablas resumen con costo total, promedio diario y participación porcentual por servicio y por proveedor, relacionando cada servicio con su modelo (IaaS/PaaS/SaaS).

**Instrucciones:**

1. Calcula el resumen por proveedor:

```python
# RESUMEN DE COSTOS POR PROVEEDOR
print("COSTO TOTAL POR PROVEEDOR CLOUD (AmortizedCost)")
print("="*60)

provider_summary = df_clean.groupby('Provider').agg(
    Total_AmortizedCost=('AmortizedCost', 'sum'),
    Total_BilledCost=('BilledCost', 'sum'),
    Num_Registros=('InvoiceId', 'count'),
    Costo_Promedio_Diario=('AmortizedCost', lambda x: x.sum() / 31)
).round(2)

provider_summary['Participacion_%'] = (
    provider_summary['Total_AmortizedCost'] / 
    provider_summary['Total_AmortizedCost'].sum() * 100
).round(1)

provider_summary = provider_summary.sort_values('Total_AmortizedCost', ascending=False)
print(provider_summary.to_string())
print(f"\n  TOTAL GENERAL: ${provider_summary['Total_AmortizedCost'].sum():,.2f}")
```

2. Calcula el resumen por servicio (top 10):

```python
# RESUMEN DE COSTOS POR SERVICIO (Top 10)
print("\nTOP 10 SERVICIOS POR COSTO AMORTIZADO")
print("="*60)

service_summary = df_clean.groupby(['ServiceName', 'ServiceModel', 'Provider']).agg(
    Total_AmortizedCost=('AmortizedCost', 'sum'),
    Num_Registros=('InvoiceId', 'count'),
    Costo_Promedio=('AmortizedCost', 'mean')
).round(2)

service_summary['Participacion_%'] = (
    service_summary['Total_AmortizedCost'] / 
    service_summary['Total_AmortizedCost'].sum() * 100
).round(1)

service_summary = service_summary.sort_values('Total_AmortizedCost', ascending=False)
print(service_summary.head(10).to_string())
```

3. Analiza el costo por modelo de servicio (IaaS/PaaS/SaaS):

```python
# COSTO POR MODELO DE SERVICIO
print("\nCOSTO POR MODELO DE SERVICIO (IaaS / PaaS / SaaS)")
print("="*60)

model_summary = df_clean.groupby('ServiceModel').agg(
    Total_AmortizedCost=('AmortizedCost', 'sum'),
    Total_BilledCost=('BilledCost', 'sum'),
    Ahorro_vs_Billed=('BilledCost', lambda x: 0),  # placeholder
    Num_Registros=('InvoiceId', 'count')
).round(2)

# Calcular ahorro (diferencia entre billed y amortized)
model_summary['Ahorro_vs_Billed'] = (
    model_summary['Total_BilledCost'] - model_summary['Total_AmortizedCost']
).round(2)

model_summary['Participacion_%'] = (
    model_summary['Total_AmortizedCost'] / 
    model_summary['Total_AmortizedCost'].sum() * 100
).round(1)

print(model_summary.to_string())
print(f"\n→ Interpretación FinOps:")
print(f"  • IaaS: Mayor oportunidad de rightsizing y scheduling")
print(f"  • PaaS: Optimizar tier selection y escalado automático")
print(f"  • SaaS: Auditar licencias activas vs. contratadas")
```

**Salida esperada (estructura):**

```
COSTO TOTAL POR PROVEEDOR CLOUD (AmortizedCost)
============================================================
          Total_AmortizedCost  Total_BilledCost  Num_Registros  Costo_Promedio_Diario  Participacion_%
Provider                                                                                              
AWS                  XXXXX.XX          XXXXX.XX            262                XXXX.XX             50.X
Azure                XXXXX.XX          XXXXX.XX            138                XXXX.XX             30.X
GCP                  XXXXX.XX          XXXXX.XX            100                XXXX.XX             20.X

  TOTAL GENERAL: $XX,XXX.XX
```

**Verificación:** AWS debe representar ~50% del gasto total, Azure ~30% y GCP ~20%, consistente con los pesos de distribución. IaaS debe ser el segmento con mayor gasto absoluto.

---

### Paso 7: Analizar cobertura de reservas vs. on-demand

**Objetivo:** Cuantificar la proporción de gasto cubierto por reservas frente a on-demand, e identificar el ahorro generado por compromisos.

**Instrucciones:**

1. Calcula la cobertura de reservas:

```python
# ANÁLISIS DE COBERTURA: RESERVAS vs ON-DEMAND
print("ANÁLISIS DE COBERTURA DE COMPROMISOS")
print("="*60)

# Filtrar solo registros de Usage
usage_only = df_clean[df_clean['ChargeType'] == 'Usage'].copy()

coverage = usage_only.groupby('UsageType').agg(
    Total_BilledCost=('BilledCost', 'sum'),
    Total_EffectiveCost=('EffectiveCost', 'sum'),
    Num_Registros=('InvoiceId', 'count')
).round(2)

coverage['Ahorro_USD'] = (coverage['Total_BilledCost'] - coverage['Total_EffectiveCost']).round(2)
coverage['Descuento_%'] = (
    (coverage['Total_BilledCost'] - coverage['Total_EffectiveCost']) / 
    coverage['Total_BilledCost'] * 100
).round(1)

print(coverage.to_string())

total_billed = usage_only['BilledCost'].sum()
total_effective = usage_only['EffectiveCost'].sum()
total_savings = total_billed - total_effective

print(f"\n{'='*60}")
print(f"  Gasto total facturado (on-demand rates): ${total_billed:,.2f}")
print(f"  Gasto efectivo (con descuentos):         ${total_effective:,.2f}")
print(f"  Ahorro total por compromisos:            ${total_savings:,.2f}")
print(f"  Porcentaje de ahorro:                    {(total_savings/total_billed*100):.1f}%")

# Cobertura de reservas
reserved_cost = usage_only[usage_only['UsageType'] == 'Reserved']['BilledCost'].sum()
coverage_pct = (reserved_cost / total_billed * 100)
print(f"\n  Cobertura de reservas:                   {coverage_pct:.1f}% del gasto Usage")
```

**Salida esperada (estructura):**

```
ANÁLISIS DE COBERTURA DE COMPROMISOS
============================================================
               Total_BilledCost  Total_EffectiveCost  Num_Registros  Ahorro_USD  Descuento_%
UsageType                                                                                    
OnDemand              XXXXX.XX             XXXXX.XX            XXX        0.00          0.0
Reserved              XXXXX.XX             XXXXX.XX             XX     XXXX.XX         35.0
SpotInstance          XXXXX.XX             XXXXX.XX             XX     XXXX.XX         70.0
```

**Verificación:** Los registros `Reserved` deben mostrar ~35% de descuento y los `SpotInstance` ~70%, consistente con la lógica de generación del dataset.

---

### Paso 8: Generar el archivo de resumen Excel

**Objetivo:** Exportar los resultados del análisis a un archivo Excel multi-hoja que servirá como insumo para laboratorios posteriores.

**Instrucciones:**

1. Genera el archivo Excel con múltiples hojas:

```python
# EXPORTAR RESUMEN A EXCEL
print("GENERANDO ARCHIVO DE RESUMEN EXCEL")
print("="*60)

output_file = OUTPUTS / 'novatech_billing_summary_jan2024.xlsx'

# Asegurar que el directorio existe
OUTPUTS.mkdir(parents=True, exist_ok=True)

with pd.ExcelWriter(output_file, engine='openpyxl') as writer:
    
    # Hoja 1: Resumen por proveedor
    provider_summary.to_excel(writer, sheet_name='Por_Proveedor')
    print("  ✓ Hoja 'Por_Proveedor' creada")
    
    # Hoja 2: Resumen por servicio
    service_summary.reset_index().to_excel(writer, sheet_name='Por_Servicio', index=False)
    print("  ✓ Hoja 'Por_Servicio' creada")
    
    # Hoja 3: Resumen por modelo (IaaS/PaaS/SaaS)
    model_summary.to_excel(writer, sheet_name='Por_Modelo')
    print("  ✓ Hoja 'Por_Modelo' creada")
    
    # Hoja 4: Análisis de cobertura
    coverage.to_excel(writer, sheet_name='Cobertura_Reservas')
    print("  ✓ Hoja 'Cobertura_Reservas' creada")
    
    # Hoja 5: Datos limpios completos
    df_clean.to_excel(writer, sheet_name='Datos_Completos', index=False)
    print("  ✓ Hoja 'Datos_Completos' creada")
    
    # Hoja 6: Metadata del análisis
    metadata = pd.DataFrame({
        'Campo': ['Organización', 'Período', 'Total Registros', 
                  'Proveedores', 'Fecha Análisis', 'Analista',
                  'Costo Total Amortizado', 'Moneda'],
        'Valor': ['NovaTech SRL', 'Enero 2024', str(len(df_clean)),
                  'AWS, Azure, GCP', pd.Timestamp.now().strftime('%Y-%m-%d %H:%M'),
                  'Equipo FinOps',
                  f"${df_clean['AmortizedCost'].sum():,.2f}", 'USD']
    })
    metadata.to_excel(writer, sheet_name='Metadata', index=False)
    print("  ✓ Hoja 'Metadata' creada")

print(f"\n  Archivo guardado: {output_file}")
print(f"  Tamaño: {output_file.stat().st_size / 1024:.1f} KB")
```

2. Verifica la integridad del archivo generado:

```python
# VERIFICACIÓN DEL ARCHIVO GENERADO
print("\nVERIFICACIÓN DE INTEGRIDAD")
print("="*60)

# Releer el archivo para confirmar
verify = pd.ExcelFile(output_file)
print(f"  Hojas en el archivo: {verify.sheet_names}")
print(f"  Total de hojas: {len(verify.sheet_names)}")

# Verificar hoja principal
df_verify = pd.read_excel(output_file, sheet_name='Datos_Completos')
print(f"  Registros en 'Datos_Completos': {len(df_verify)}")
print(f"\n  ✓ Archivo listo para Lab 02-00-02 y Lab 03-00-01")
```

**Salida esperada:**

```
GENERANDO ARCHIVO DE RESUMEN EXCEL
============================================================
  ✓ Hoja 'Por_Proveedor' creada
  ✓ Hoja 'Por_Servicio' creada
  ✓ Hoja 'Por_Modelo' creada
  ✓ Hoja 'Cobertura_Reservas' creada
  ✓ Hoja 'Datos_Completos' creada
  ✓ Hoja 'Metadata' creada

  Archivo guardado: ../outputs/novatech_billing_summary_jan2024.xlsx
  Tamaño: ~XXX.X KB

VERIFICACIÓN DE INTEGRIDAD
============================================================
  Hojas en el archivo: ['Por_Proveedor', 'Por_Servicio', 'Por_Modelo', 'Cobertura_Reservas', 'Datos_Completos', 'Metadata']
  Total de hojas: 6
  Registros en 'Datos_Completos': 500

  ✓ Archivo listo para Lab 02-00-02 y Lab 03-00-01
```

**Verificación:** El archivo debe contener exactamente 6 hojas y la hoja `Datos_Completos` debe tener 500 registros.

---

### Paso 9: Resumen ejecutivo y conclusiones del análisis

**Objetivo:** Sintetizar los hallazgos clave en un formato de reporte ejecutivo que conecte los datos con decisiones FinOps.

**Instrucciones:**

1. Genera el resumen ejecutivo final:

```python
# RESUMEN EJECUTIVO
print("="*60)
print("  RESUMEN EJECUTIVO - REPORTE DE FACTURACIÓN")
print("  NovaTech SRL | Enero 2024")
print("="*60)

total_amortized = df_clean['AmortizedCost'].sum()
total_billed_all = df_clean['BilledCost'].sum()
days_in_period = 31

print(f"""
┌─────────────────────────────────────────────────────────┐
│ MÉTRICAS CLAVE                                          │
├─────────────────────────────────────────────────────────┤
│ Costo Total Amortizado:    ${total_amortized:>12,.2f}           │
│ Costo Promedio Diario:     ${total_amortized/days_in_period:>12,.2f}           │
│ Registros procesados:      {len(df_clean):>12,}           │
│ Proveedores activos:       {df_clean['Provider'].nunique():>12}           │
│ Servicios únicos:          {df_clean['ServiceName'].nunique():>12}           │
├─────────────────────────────────────────────────────────┤
│ DISTRIBUCIÓN POR MODELO                                 │
├─────────────────────────────────────────────────────────┤""")

for model in ['IaaS', 'PaaS', 'SaaS']:
    model_cost = df_clean[df_clean['ServiceModel'] == model]['AmortizedCost'].sum()
    model_pct = model_cost / total_amortized * 100
    print(f"│ {model:<6} ${model_cost:>10,.2f}  ({model_pct:>5.1f}%)                      │")

print(f"""├─────────────────────────────────────────────────────────┤
│ HALLAZGOS FINOPS                                        │
├─────────────────────────────────────────────────────────┤
│ • IaaS representa el mayor gasto → priorizar           │
│   rightsizing y scheduling en EC2/VMs                   │
│ • Cobertura de reservas: revisar oportunidades          │
│   de aumentar compromisos en cargas estables            │
│ • Tags incompletos (~5%): mejorar governance            │
│   para asignación precisa de costos                     │
│ • Créditos activos: validar vigencia y aplicación       │
└─────────────────────────────────────────────────────────┘
""")

print("Archivos generados en este laboratorio:")
print(f"  1. {output_file}")
print(f"  2. ~/finops-course/notebooks/lab_02_01_billing_reader.ipynb")
print(f"\n→ Estos archivos son REQUERIDOS para Lab 02-00-02 y Lab 03-00-01")
```

2. Guarda el notebook con `Ctrl + S`.

**Verificación:** El resumen ejecutivo debe mostrar valores coherentes donde IaaS > PaaS > SaaS en costo total, reflejando la distribución del dataset.

---

## Validación y Pruebas

Ejecuta estas verificaciones finales para confirmar que el laboratorio se completó correctamente:

```python
# VALIDACIÓN FINAL DEL LABORATORIO
print("CHECKLIST DE VALIDACIÓN - Lab 02-00-01")
print("="*60)

checks = []

# Check 1: Dataset cargado correctamente
checks.append(("Dataset con 500 registros", len(df_clean) == 500))

# Check 2: Columnas correctas
expected_cols = 22  # 21 originales + BillingDay
checks.append(("Columnas esperadas (22)", len(df_clean.columns) == expected_cols))

# Check 3: Tipos de dato correctos
checks.append(("BillingDate es datetime", df_clean['BillingDate'].dtype == 'datetime64[ns]'))
checks.append(("BilledCost es float", df_clean['BilledCost'].dtype == 'float64'))

# Check 4: Archivo Excel existe
checks.append(("Excel generado existe", output_file.exists()))

# Check 5: Excel tiene 6 hojas
excel_sheets = pd.ExcelFile(output_file).sheet_names
checks.append(("Excel tiene 6 hojas", len(excel_sheets) == 6))

# Check 6: Tres proveedores presentes
checks.append(("3 proveedores (AWS/Azure/GCP)", df_clean['Provider'].nunique() == 3))

# Check 7: Tres modelos presentes
checks.append(("3 modelos (IaaS/PaaS/SaaS)", df_clean['ServiceModel'].nunique() == 3))

# Check 8: Costos numéricos sin NaN en columnas clave
checks.append(("Sin NaN en BilledCost", df_clean['BilledCost'].notna().all()))

# Imprimir resultados
all_passed = True
for description, passed in checks:
    status = "✓ PASS" if passed else "✗ FAIL"
    print(f"  [{status}] {description}")
    if not passed:
        all_passed = False

print(f"\n{'='*60}")
if all_passed:
    print("  ✓ LABORATORIO COMPLETADO EXITOSAMENTE")
    print("  → Proceder a Lab 02-00-02")
else:
    print("  ✗ HAY VERIFICACIONES FALLIDAS - Revisar pasos anteriores")
```

**Resultado esperado:** Todas las verificaciones deben mostrar `✓ PASS`.

---

## Solución de Problemas

### Problema 1: Error `FileNotFoundError` al cargar el CSV

**Síntomas:**
```
FileNotFoundError: [Errno 2] No such file or directory: '../data/raw/novatech_billing_raw.csv'
```

**Causa:** El notebook no se ejecutó desde el directorio `~/finops-course/notebooks/`, por lo que la ruta relativa `../data/raw/` no apunta al directorio correcto.

**Solución:**

```python
# Opción 1: Verificar directorio actual del notebook
import os
print(f"Directorio actual: {os.getcwd()}")

# Opción 2: Usar ruta absoluta
from pathlib import Path
DATA_RAW = Path.home() / 'finops-course' / 'data' / 'raw'
print(f"Ruta absoluta: {DATA_RAW}")
print(f"Archivo existe: {(DATA_RAW / 'novatech_billing_raw.csv').exists()}")

# Opción 3: Reiniciar Jupyter desde el directorio correcto
# jupyter notebook --port=8888 --notebook-dir=~/finops-course/notebooks/
```

---

### Problema 2: Error `ModuleNotFoundError: No module named 'openpyxl'` al exportar Excel

**Síntomas:**
```
ModuleNotFoundError: No module named 'openpyxl'
```
O bien:
```
ImportError: Missing optional dependency 'openpyxl'. Use pip or conda to install openpyxl.
```

**Causa:** El paquete `openpyxl` no está instalado en el entorno Python que usa el kernel de Jupyter, o se instaló en un entorno virtual diferente al que ejecuta el notebook.

**Solución:**

```python
# Instalar directamente desde el notebook
import subprocess
subprocess.check_call(['pip', 'install', 'openpyxl==3.1.2'])

# Verificar instalación
import openpyxl
print(f"openpyxl version: {openpyxl.__version__}")

# Si persiste, verificar que el kernel usa el Python correcto:
import sys
print(f"Python ejecutable: {sys.executable}")
# Instalar en ese Python específico:
# !{sys.executable} -m pip install openpyxl==3.1.2
```

Después de instalar, reiniciar el kernel del notebook: menú **Kernel → Restart Kernel** y re-ejecutar todas las celdas.

---

## Limpieza

Este laboratorio no requiere limpieza de recursos cloud ni contenedores. Los archivos generados son necesarios para laboratorios posteriores.

**No eliminar:**
- `~/finops-course/outputs/novatech_billing_summary_jan2024.xlsx` (requerido por Lab 02-00-02 y Lab 03-00-01)
- `~/finops-course/notebooks/lab_02_01_billing_reader.ipynb` (referencia)
- `~/finops-course/data/raw/novatech_billing_raw.csv` (dataset base)

**Opcional — eliminar archivos temporales:**

```bash
# Solo si deseas limpiar el script generador (no afecta labs posteriores)
rm -f ~/finops-course/data/raw/generate_billing_data.py
```

---

## Resumen

### Lo que aprendiste en este laboratorio

| Concepto | Aplicación realizada |
|----------|---------------------|
| Campos FOCUS 1.0 | Identificaste BilledCost, EffectiveCost, AmortizedCost, NetCost y su significado |
| Tipos de cargo | Diferenciaste Usage, Purchase, Tax y Credit en un dataset real |
| Modelos de servicio | Clasificaste gastos por IaaS, PaaS y SaaS y sus implicaciones de optimización |
| Cobertura de reservas | Cuantificaste el ahorro por compromisos (Reserved, Spot) vs. On-Demand |
| Calidad de datos | Detectaste y corregiste tags vacíos e inconsistencias de formato |
| Exportación estructurada | Generaste un Excel multi-hoja como entregable FinOps reutilizable |

### Conexión con la lección teórica

Los modelos IaaS, PaaS y SaaS que estudiaste en la lección 2.1 se materializan directamente en el dataset: los servicios de cómputo (EC2, Virtual Machines, Compute Engine) representan IaaS con alta granularidad de costo; las bases de datos gestionadas y funciones serverless representan PaaS con facturación por solicitud o tier; y los servicios de productividad representan SaaS con costo por usuario. Cada modelo requiere una estrategia de optimización diferente, como confirmaste en el análisis.

### Archivos generados (dependencias para labs futuros)

```
~/finops-course/
├── data/raw/
│   └── novatech_billing_raw.csv          ← Dataset fuente (500 registros)
├── notebooks/
│   └── lab_02_01_billing_reader.ipynb     ← Notebook completado
└── outputs/
    └── novatech_billing_summary_jan2024.xlsx  ← CRÍTICO para Lab 02-00-02
```

### Recursos adicionales

- [FOCUS Specification 1.0](https://focus.finops.org/) — Estándar oficial de normalización de datos de costo cloud
- [FinOps Foundation — Data Ingestion](https://www.finops.org/framework/capabilities/data-ingestion/) — Capacidad de ingestión de datos en el marco FinOps
- [pandas Documentation — GroupBy](https://pandas.pydata.org/docs/user_guide/groupby.html) — Referencia para operaciones de agrupación utilizadas en este lab

---

# Demostración 2. Análisis de tendencias de consumo por servicio

## Metadata

| Campo | Valor |
|-------|-------|
| **Duración** | 25 minutos |
| **Complejidad** | Media |
| **Nivel Bloom** | Aplicar |
| **Prerrequisito directo** | Lab 02-00-01 completado |
| **Archivos de salida** | `novatech_trend_report.xlsx`, `trend_line_top5.png`, `stacked_bar_category.png`, `anomalies_highlight.png` |

## Descripción General

En este laboratorio construirás series de tiempo de consumo cloud por servicio usando datos históricos de 90 días (Q4 2023) concatenados con el resumen de enero 2024 generado en el laboratorio anterior. Aplicarás agrupación semanal, cálculo de variaciones porcentuales Week-over-Week (WoW), detección de anomalías mediante z-score y generarás visualizaciones profesionales que comuniquen tendencias de gasto IaaS, PaaS y SaaS de NovaTech SRL a audiencias técnicas y ejecutivas.

## Objetivos de Aprendizaje

- [ ] Construir series de tiempo de consumo cloud agrupadas por semana y servicio a partir de datos de facturación de 4 meses
- [ ] Detectar anomalías de consumo aplicando z-score (umbral > 2.0) sobre variaciones semanales de gasto por servicio
- [ ] Comparar la evolución del mix de servicios IaaS, PaaS y SaaS mediante gráficas de barras apiladas
- [ ] Generar visualizaciones de tendencias con matplotlib exportadas como PNG de alta resolución (150 DPI)
- [ ] Producir un reporte Excel consolidado (`novatech_trend_report.xlsx`) con múltiples hojas para consumo en laboratorios posteriores

## Prerrequisitos

### Conocimiento requerido

| Tema | Nivel |
|------|-------|
| Series de tiempo y agrupación temporal | Básico |
| Variación porcentual (WoW) | Básico |
| Diferencias IaaS / PaaS / SaaS | Comprendido (Lección 2.1) |
| pandas DataFrames y groupby | Intermedio |
| matplotlib gráficas de líneas y barras | Básico |

### Acceso y archivos requeridos

| Recurso | Ubicación |
|---------|-----------|
| Dataset 90 días | `~/finops-course/data/raw/novatech_billing_90days.csv` |
| Resumen enero 2024 (Lab 02-00-01) | `~/finops-course/data/processed/novatech_billing_summary_jan2024.xlsx` |
| Notebook pre-estructurado | `~/finops-course/notebooks/lab_02_02_trend_analysis.ipynb` |

> **Nota:** Si no completaste el Lab 02-00-01, solicita al instructor los archivos de referencia pre-completados.

## Entorno de Laboratorio

### Software requerido

| Herramienta | Versión | Propósito |
|-------------|---------|-----------|
| Python | 3.12.1 | Runtime |
| pandas | 2.2.1 | Manipulación de datos |
| numpy | 1.26.4 | Cálculo de z-score |
| matplotlib | 3.8.3 | Visualizaciones |
| openpyxl | 3.1.2 | Exportación Excel |
| Jupyter Notebook | 7.1.2 | Entorno interactivo |

### Comandos de preparación del entorno

```bash
# Verificar que el entorno está activo
cd ~/finops-course
pip install -r requirements.txt --quiet

# Verificar archivos de entrada
ls -la data/raw/novatech_billing_90days.csv
ls -la data/processed/novatech_billing_summary_jan2024.xlsx

# Crear directorio de salida si no existe
mkdir -p outputs/lab_02_02
mkdir -p data/processed

# Iniciar Jupyter Notebook
jupyter notebook --port=8888 --notebook-dir=~/finops-course/notebooks/
```

## Estructura del Dataset `novatech_billing_90days.csv`

Antes de iniciar, es importante comprender la estructura del dataset de 90 días:

| Columna | Tipo | Descripción | Ejemplo |
|---------|------|-------------|---------|
| `date` | string (YYYY-MM-DD) | Fecha de facturación | 2023-10-01 |
| `provider` | string | Proveedor cloud | AWS, Azure, GCP |
| `service_name` | string | Nombre del servicio | EC2, Azure SQL, BigQuery |
| `service_category` | string | Categoría IaaS/PaaS/SaaS | IaaS |
| `product` | string | Producto de NovaTech | PayCore, AnalyticsHub, DevPortal |
| `team` | string | Equipo responsable | Backend, Frontend, Data, Platform |
| `environment` | string | Ambiente | prod, staging, dev |
| `cost_usd` | float | Costo en USD | 127.45 |
| `usage_quantity` | float | Cantidad de uso | 720 |
| `usage_unit` | string | Unidad de medida | hours, GB, requests |

## Paso a Paso

### Paso 1: Cargar y explorar los datasets

**Objetivo:** Cargar el dataset de 90 días y el resumen de enero 2024, verificar su integridad y preparar la concatenación.

**Instrucciones:**

1. Abre el notebook `lab_02_02_trend_analysis.ipynb` en Jupyter.

2. En la primera celda, importa las bibliotecas necesarias:

```python
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
from datetime import datetime, timedelta
import warnings
warnings.filterwarnings('ignore')

# Configuración de matplotlib para español
plt.rcParams['figure.figsize'] = (12, 6)
plt.rcParams['figure.dpi'] = 150
plt.rcParams['font.size'] = 10

print("Bibliotecas cargadas correctamente")
print(f"pandas: {pd.__version__}")
print(f"numpy: {np.__version__}")
```

3. Carga el dataset de 90 días:

```python
# Cargar dataset Q4 2023 (90 días)
df_90d = pd.read_csv('../data/raw/novatech_billing_90days.csv', parse_dates=['date'])

print(f"Dataset 90 días: {df_90d.shape[0]} registros, {df_90d.shape[1]} columnas")
print(f"Rango de fechas: {df_90d['date'].min().strftime('%Y-%m-%d')} a {df_90d['date'].max().strftime('%Y-%m-%d')}")
print(f"\nServicios únicos: {df_90d['service_name'].nunique()}")
print(f"Categorías: {df_90d['service_category'].unique().tolist()}")
print(f"\nGasto total Q4 2023: ${df_90d['cost_usd'].sum():,.2f}")
```

4. Carga el resumen de enero 2024 del laboratorio anterior:

```python
# Cargar resumen de enero 2024 (output del Lab 02-00-01)
df_jan = pd.read_excel('../data/processed/novatech_billing_summary_jan2024.xlsx', 
                       sheet_name='daily_detail')

# Asegurar que la columna date sea datetime
df_jan['date'] = pd.to_datetime(df_jan['date'])

print(f"\nDataset enero 2024: {df_jan.shape[0]} registros")
print(f"Rango: {df_jan['date'].min().strftime('%Y-%m-%d')} a {df_jan['date'].max().strftime('%Y-%m-%d')}")
print(f"Gasto total enero 2024: ${df_jan['cost_usd'].sum():,.2f}")
```

5. Concatena ambos datasets:

```python
# Concatenar datasets para serie temporal completa (4 meses)
df = pd.concat([df_90d, df_jan], ignore_index=True)
df = df.sort_values('date').reset_index(drop=True)

print(f"\n{'='*50}")
print(f"DATASET CONSOLIDADO")
print(f"{'='*50}")
print(f"Total registros: {df.shape[0]:,}")
print(f"Período: {df['date'].min().strftime('%Y-%m-%d')} a {df['date'].max().strftime('%Y-%m-%d')}")
print(f"Gasto total 4 meses: ${df['cost_usd'].sum():,.2f}")
print(f"Promedio diario: ${df.groupby('date')['cost_usd'].sum().mean():,.2f}")
```

**Resultado esperado:**

```
Dataset 90 días: ~2700 registros, 10 columnas
Rango de fechas: 2023-10-01 a 2023-12-31
Servicios únicos: 12-15
Categorías: ['IaaS', 'PaaS', 'SaaS']
Gasto total Q4 2023: ~$185,000-$220,000

Dataset enero 2024: ~900 registros
Gasto total enero 2024: ~$68,000-$75,000

DATASET CONSOLIDADO
Total registros: ~3,600
Período: 2023-10-01 a 2024-01-31
Gasto total 4 meses: ~$255,000-$295,000
```

**Verificación:** Confirma que el rango de fechas cubre exactamente del 1 de octubre 2023 al 31 de enero 2024 y que las tres categorías (IaaS, PaaS, SaaS) están presentes.

---

### Paso 2: Agrupación semanal por servicio

**Objetivo:** Crear una serie temporal semanal que permita identificar tendencias de consumo por servicio y categoría.

**Instrucciones:**

1. Crea la columna de semana y agrupa los datos:

```python
# Crear columna de semana (inicio lunes)
df['week'] = df['date'].dt.to_period('W').apply(lambda r: r.start_time)

# Agrupación semanal por servicio
weekly_service = df.groupby(['week', 'service_name', 'service_category']).agg(
    cost_usd=('cost_usd', 'sum'),
    records=('cost_usd', 'count')
).reset_index()

print(f"Semanas en el período: {weekly_service['week'].nunique()}")
print(f"Combinaciones servicio-semana: {weekly_service.shape[0]}")
print(f"\nGasto semanal promedio total: ${weekly_service.groupby('week')['cost_usd'].sum().mean():,.2f}")
```

2. Crea la tabla pivote para análisis de tendencias:

```python
# Tabla pivote: semanas como filas, servicios como columnas
pivot_services = weekly_service.pivot_table(
    index='week', 
    columns='service_name', 
    values='cost_usd', 
    aggfunc='sum',
    fill_value=0
)

print(f"\nTabla pivote: {pivot_services.shape[0]} semanas x {pivot_services.shape[1]} servicios")
print(f"\nTop 5 servicios por gasto total:")
top5_total = pivot_services.sum().sort_values(ascending=False).head(5)
for svc, cost in top5_total.items():
    print(f"  {svc}: ${cost:,.2f}")
```

3. Agrupa por categoría de servicio (IaaS/PaaS/SaaS):

```python
# Agrupación semanal por categoría
weekly_category = df.groupby(['week', 'service_category']).agg(
    cost_usd=('cost_usd', 'sum')
).reset_index()

# Pivote por categoría
pivot_category = weekly_category.pivot_table(
    index='week',
    columns='service_category',
    values='cost_usd',
    aggfunc='sum',
    fill_value=0
)

# Calcular porcentaje de cada categoría
pivot_category_pct = pivot_category.div(pivot_category.sum(axis=1), axis=0) * 100

print("\nDistribución promedio por categoría:")
for cat in pivot_category_pct.columns:
    print(f"  {cat}: {pivot_category_pct[cat].mean():.1f}%")
```

**Resultado esperado:**

```
Semanas en el período: 17-18
Combinaciones servicio-semana: ~200-270

Top 5 servicios por gasto total:
  EC2: $XX,XXX
  RDS: $XX,XXX
  S3: $XX,XXX
  Azure SQL: $XX,XXX
  BigQuery: $XX,XXX

Distribución promedio por categoría:
  IaaS: ~55-65%
  PaaS: ~25-30%
  SaaS: ~10-15%
```

**Verificación:** La suma de porcentajes por categoría debe ser 100% en cada semana. IaaS debe representar la mayor proporción del gasto (consistente con la lección 2.1 sobre mayor control y visibilidad).

---

### Paso 3: Cálculo de variación porcentual Week-over-Week (WoW)

**Objetivo:** Calcular la variación porcentual semanal para cada servicio e identificar los servicios con mayor crecimiento de gasto.

**Instrucciones:**

1. Calcula la variación WoW para cada servicio:

```python
# Calcular variación porcentual WoW por servicio
wow_change = pivot_services.pct_change() * 100

# Reemplazar infinitos (cuando semana anterior era 0) con NaN
wow_change = wow_change.replace([np.inf, -np.inf], np.nan)

print("Variación WoW (%) - Últimas 4 semanas:")
print(wow_change.tail(4).round(1).to_string())
```

2. Identifica los top 5 servicios por crecimiento acumulado:

```python
# Crecimiento total: comparar última semana vs primera semana
first_week = pivot_services.iloc[0]
last_week = pivot_services.iloc[-1]

# Calcular crecimiento porcentual total del período
growth = ((last_week - first_week) / first_week.replace(0, np.nan)) * 100
growth = growth.dropna().sort_values(ascending=False)

print("\n" + "="*50)
print("TOP 5 SERVICIOS POR CRECIMIENTO DE GASTO (Oct 2023 → Ene 2024)")
print("="*50)
top5_growth = growth.head(5)
for i, (svc, pct) in enumerate(top5_growth.items(), 1):
    first_cost = first_week[svc]
    last_cost = last_week[svc]
    print(f"  {i}. {svc}: +{pct:.1f}% (${first_cost:,.0f}/sem → ${last_cost:,.0f}/sem)")

print(f"\n{'='*50}")
print("TOP 5 SERVICIOS CON MAYOR REDUCCIÓN")
print("="*50)
bottom5 = growth.sort_values().head(5)
for i, (svc, pct) in enumerate(bottom5.items(), 1):
    print(f"  {i}. {svc}: {pct:.1f}%")
```

3. Calcula la volatilidad semanal (desviación estándar de WoW):

```python
# Volatilidad: desviación estándar de cambios WoW
volatility = wow_change.std().sort_values(ascending=False)

print("\nServicios más volátiles (mayor variación semanal):")
for svc, vol in volatility.head(5).items():
    print(f"  {svc}: σ = {vol:.1f}% WoW")
```

**Resultado esperado:**

```
TOP 5 SERVICIOS POR CRECIMIENTO DE GASTO (Oct 2023 → Ene 2024)
==================================================
  1. BigQuery: +45.2% ($1,200/sem → $1,742/sem)
  2. Lambda: +38.7% ($800/sem → $1,110/sem)
  3. Azure SQL: +22.1% ($2,100/sem → $2,564/sem)
  ...

Servicios más volátiles (mayor variación semanal):
  BigQuery: σ = 18.5% WoW
  Lambda: σ = 15.2% WoW
  ...
```

**Verificación:** Los servicios PaaS/SaaS con mayor crecimiento suelen indicar adopción creciente o escalamiento automático. Los servicios IaaS estables reflejan cargas de trabajo predecibles. Valida que los porcentajes sean razonables (crecimiento > 100% semanal sería sospechoso).

---

### Paso 4: Detección de anomalías con z-score

**Objetivo:** Aplicar el método de z-score para identificar semanas con consumo anormalmente alto o bajo en cada servicio, usando un umbral de |z| > 2.0.

**Instrucciones:**

1. Calcula el z-score para cada servicio por semana:

```python
# Calcular z-score por servicio (basado en costo semanal)
from scipy import stats  # Alternativa: cálculo manual con numpy

# Cálculo manual de z-score (sin dependencia scipy)
def calculate_zscore(series):
    """Calcula z-score para una serie temporal."""
    mean = series.mean()
    std = series.std()
    if std == 0:
        return pd.Series(0, index=series.index)
    return (series - mean) / std

# Aplicar z-score a cada servicio
zscores = pivot_services.apply(calculate_zscore)

print("Z-scores calculados para todos los servicios")
print(f"Dimensiones: {zscores.shape}")
```

2. Identifica las anomalías (|z| > 2.0):

```python
# Definir umbral de anomalía
ZSCORE_THRESHOLD = 2.0

# Encontrar todas las anomalías
anomalies = []
for service in zscores.columns:
    for week in zscores.index:
        z = zscores.loc[week, service]
        if abs(z) > ZSCORE_THRESHOLD:
            cost = pivot_services.loc[week, service]
            mean_cost = pivot_services[service].mean()
            anomalies.append({
                'week': week,
                'service': service,
                'z_score': round(z, 2),
                'cost_usd': round(cost, 2),
                'mean_cost': round(mean_cost, 2),
                'deviation_pct': round((cost - mean_cost) / mean_cost * 100, 1),
                'type': 'PICO' if z > 0 else 'CAÍDA'
            })

df_anomalies = pd.DataFrame(anomalies)
df_anomalies = df_anomalies.sort_values('z_score', key=abs, ascending=False)

print(f"\n{'='*60}")
print(f"ANOMALÍAS DETECTADAS (|z-score| > {ZSCORE_THRESHOLD})")
print(f"{'='*60}")
print(f"Total anomalías: {len(df_anomalies)}")
print(f"  - Picos (z > +{ZSCORE_THRESHOLD}): {(df_anomalies['type'] == 'PICO').sum()}")
print(f"  - Caídas (z < -{ZSCORE_THRESHOLD}): {(df_anomalies['type'] == 'CAÍDA').sum()}")
print(f"\nTop 5 anomalías más severas:")
print(df_anomalies.head(5)[['week', 'service', 'z_score', 'cost_usd', 'deviation_pct', 'type']].to_string(index=False))
```

3. Clasifica anomalías por categoría de servicio:

```python
# Mapear categoría a cada anomalía
service_to_category = df[['service_name', 'service_category']].drop_duplicates()
service_to_category = service_to_category.set_index('service_name')['service_category'].to_dict()

df_anomalies['category'] = df_anomalies['service'].map(service_to_category)

print("\nAnomалías por categoría de servicio:")
anomaly_by_cat = df_anomalies.groupby('category').agg(
    count=('z_score', 'count'),
    avg_severity=('z_score', lambda x: abs(x).mean())
).round(2)
print(anomaly_by_cat.to_string())

print("\n💡 Insight FinOps:")
max_cat = anomaly_by_cat['count'].idxmax()
print(f"   La categoría '{max_cat}' presenta más anomalías,")
print(f"   lo que sugiere mayor variabilidad en su consumo.")
if max_cat == 'IaaS':
    print(f"   → Revisar políticas de rightsizing y scheduling")
elif max_cat == 'PaaS':
    print(f"   → Revisar configuraciones de auto-scaling")
else:
    print(f"   → Revisar patrones de uso de licencias")
```

**Resultado esperado:**

```
ANOMALÍAS DETECTADAS (|z-score| > 2.0)
============================================================
Total anomalías: 5-12
  - Picos (z > +2.0): 3-7
  - Caídas (z < -2.0): 2-5

Top 5 anomalías más severas:
       week     service  z_score  cost_usd  deviation_pct   type
 2023-12-23     EC2       2.85    8542.30        42.3      PICO
 2023-11-20     BigQuery  2.61    2890.15        55.1      PICO
 ...

Anomalías por categoría de servicio:
          count  avg_severity
IaaS        5      2.34
PaaS        4      2.21
SaaS        2      2.08
```

**Verificación:** Las anomalías deben ser coherentes con el contexto de negocio. Picos en diciembre pueden corresponder a Black Friday/Navidad (tráfico e-commerce). Caídas en fechas festivas son normales para ambientes de desarrollo.

---

### Paso 5: Generación de visualizaciones

**Objetivo:** Crear tres gráficas profesionales que comuniquen los hallazgos de tendencias, mix de servicios y anomalías.

**Instrucciones:**

1. Gráfica 1 — Tendencia de líneas para Top 5 servicios por crecimiento:

```python
# Gráfica 1: Tendencia de los top 5 servicios por crecimiento
fig, ax = plt.subplots(figsize=(14, 7))

top5_services = top5_growth.index.tolist()
colors = ['#e74c3c', '#3498db', '#2ecc71', '#f39c12', '#9b59b6']

for i, service in enumerate(top5_services):
    ax.plot(pivot_services.index, pivot_services[service], 
            marker='o', markersize=4, linewidth=2,
            color=colors[i], label=f"{service} (+{top5_growth[service]:.0f}%)")

ax.set_title('NovaTech SRL — Top 5 Servicios por Crecimiento de Gasto\n(Oct 2023 – Ene 2024)', 
             fontsize=14, fontweight='bold')
ax.set_xlabel('Semana', fontsize=11)
ax.set_ylabel('Costo Semanal (USD)', fontsize=11)
ax.legend(loc='upper left', fontsize=9, framealpha=0.9)
ax.grid(True, alpha=0.3, linestyle='--')
ax.yaxis.set_major_formatter(plt.FuncFormatter(lambda x, p: f'${x:,.0f}'))

# Rotar etiquetas del eje X
plt.xticks(rotation=45, ha='right')
plt.tight_layout()

# Guardar
output_path = '../outputs/lab_02_02/trend_line_top5.png'
plt.savefig(output_path, dpi=150, bbox_inches='tight', facecolor='white')
plt.show()
print(f"✅ Gráfica guardada: {output_path}")
```

2. Gráfica 2 — Barras apiladas por categoría (IaaS/PaaS/SaaS):

```python
# Gráfica 2: Barras apiladas por categoría de servicio
fig, (ax1, ax2) = plt.subplots(2, 1, figsize=(14, 10), gridspec_kw={'height_ratios': [2, 1]})

# Panel superior: valores absolutos
colors_cat = {'IaaS': '#2c3e50', 'PaaS': '#2980b9', 'SaaS': '#27ae60'}
pivot_category.plot(kind='bar', stacked=True, ax=ax1, 
                    color=[colors_cat.get(c, '#95a5a6') for c in pivot_category.columns],
                    width=0.8, edgecolor='white', linewidth=0.5)

ax1.set_title('NovaTech SRL — Evolución del Gasto por Categoría de Servicio\n(Barras Apiladas Semanales)', 
              fontsize=13, fontweight='bold')
ax1.set_ylabel('Costo Semanal (USD)', fontsize=11)
ax1.yaxis.set_major_formatter(plt.FuncFormatter(lambda x, p: f'${x:,.0f}'))
ax1.legend(title='Categoría', fontsize=9)
ax1.grid(True, alpha=0.2, axis='y')

# Formatear etiquetas X
week_labels = [w.strftime('%d-%b') for w in pivot_category.index]
ax1.set_xticklabels(week_labels, rotation=45, ha='right', fontsize=8)

# Panel inferior: porcentajes
pivot_category_pct.plot(kind='area', stacked=True, ax=ax2,
                        color=[colors_cat.get(c, '#95a5a6') for c in pivot_category_pct.columns],
                        alpha=0.7)
ax2.set_title('Distribución Porcentual del Mix de Servicios', fontsize=11)
ax2.set_ylabel('Porcentaje (%)', fontsize=10)
ax2.set_ylim(0, 100)
ax2.legend(title='Categoría', fontsize=9, loc='center right')
ax2.grid(True, alpha=0.2, axis='y')

plt.tight_layout()

output_path = '../outputs/lab_02_02/stacked_bar_category.png'
plt.savefig(output_path, dpi=150, bbox_inches='tight', facecolor='white')
plt.show()
print(f"✅ Gráfica guardada: {output_path}")
```

3. Gráfica 3 — Anomalías destacadas sobre la tendencia total:

```python
# Gráfica 3: Gasto total semanal con anomalías señalizadas
fig, ax = plt.subplots(figsize=(14, 7))

# Serie temporal total
weekly_total = pivot_services.sum(axis=1)
ax.plot(weekly_total.index, weekly_total.values, 
        color='#2c3e50', linewidth=2.5, marker='o', markersize=5, label='Gasto Total Semanal')

# Banda de ±2σ (zona normal)
mean_total = weekly_total.mean()
std_total = weekly_total.std()
ax.axhline(mean_total, color='#7f8c8d', linestyle='--', linewidth=1, label=f'Media: ${mean_total:,.0f}')
ax.fill_between(weekly_total.index, 
                mean_total - 2*std_total, 
                mean_total + 2*std_total, 
                alpha=0.15, color='#3498db', label='Banda ±2σ (zona normal)')

# Marcar semanas con anomalías
if not df_anomalies.empty:
    anomaly_weeks = df_anomalies['week'].unique()
    for week in anomaly_weeks:
        if week in weekly_total.index:
            total_cost = weekly_total[week]
            n_anomalies = len(df_anomalies[df_anomalies['week'] == week])
            ax.scatter(week, total_cost, color='#e74c3c', s=150, zorder=5, 
                      edgecolors='darkred', linewidth=1.5)
            ax.annotate(f'{n_anomalies} anomalía(s)', 
                       xy=(week, total_cost),
                       xytext=(10, 15), textcoords='offset points',
                       fontsize=8, color='#e74c3c',
                       arrowprops=dict(arrowstyle='->', color='#e74c3c', lw=1))

ax.set_title('NovaTech SRL — Gasto Total Semanal con Detección de Anomalías\n(Método Z-Score, umbral |z| > 2.0)', 
             fontsize=13, fontweight='bold')
ax.set_xlabel('Semana', fontsize=11)
ax.set_ylabel('Costo Semanal Total (USD)', fontsize=11)
ax.legend(loc='upper left', fontsize=9)
ax.grid(True, alpha=0.3, linestyle='--')
ax.yaxis.set_major_formatter(plt.FuncFormatter(lambda x, p: f'${x:,.0f}'))
plt.xticks(rotation=45, ha='right')
plt.tight_layout()

output_path = '../outputs/lab_02_02/anomalies_highlight.png'
plt.savefig(output_path, dpi=150, bbox_inches='tight', facecolor='white')
plt.show()
print(f"✅ Gráfica guardada: {output_path}")
```

**Resultado esperado:** Tres archivos PNG generados en `~/finops-course/outputs/lab_02_02/`:
- `trend_line_top5.png` — Líneas de tendencia ascendente para los servicios de mayor crecimiento
- `stacked_bar_category.png` — Barras apiladas mostrando el mix IaaS/PaaS/SaaS por semana
- `anomalies_highlight.png` — Serie total con puntos rojos en semanas anómalas fuera de la banda ±2σ

**Verificación:** Abre cada PNG y confirma que: (1) los ejes están etiquetados en USD, (2) las leyendas son legibles, (3) los puntos de anomalía son claramente visibles, (4) la resolución es nítida a 150 DPI.

---

### Paso 6: Exportar reporte Excel consolidado

**Objetivo:** Generar el archivo `novatech_trend_report.xlsx` con múltiples hojas que servirá como insumo para los laboratorios 03-00-01 y 04-00-01.

**Instrucciones:**

1. Prepara los DataFrames de resumen:

```python
# Resumen ejecutivo
executive_summary = pd.DataFrame({
    'Métrica': [
        'Período de análisis',
        'Gasto total (USD)',
        'Promedio semanal (USD)',
        'Servicio mayor crecimiento',
        'Crecimiento máximo (%)',
        'Total anomalías detectadas',
        'Categoría más volátil',
        'Mix IaaS promedio (%)',
        'Mix PaaS promedio (%)',
        'Mix SaaS promedio (%)'
    ],
    'Valor': [
        f"{df['date'].min().strftime('%Y-%m-%d')} a {df['date'].max().strftime('%Y-%m-%d')}",
        f"${df['cost_usd'].sum():,.2f}",
        f"${weekly_total.mean():,.2f}",
        top5_growth.index[0],
        f"{top5_growth.iloc[0]:.1f}%",
        str(len(df_anomalies)),
        anomaly_by_cat['count'].idxmax() if not df_anomalies.empty else 'N/A',
        f"{pivot_category_pct['IaaS'].mean():.1f}%" if 'IaaS' in pivot_category_pct.columns else 'N/A',
        f"{pivot_category_pct['PaaS'].mean():.1f}%" if 'PaaS' in pivot_category_pct.columns else 'N/A',
        f"{pivot_category_pct['SaaS'].mean():.1f}%" if 'SaaS' in pivot_category_pct.columns else 'N/A'
    ]
})

print("Resumen ejecutivo preparado:")
print(executive_summary.to_string(index=False))
```

2. Exporta el archivo Excel con múltiples hojas:

```python
# Exportar reporte completo
output_excel = '../data/processed/novatech_trend_report.xlsx'

with pd.ExcelWriter(output_excel, engine='openpyxl') as writer:
    # Hoja 1: Resumen ejecutivo
    executive_summary.to_excel(writer, sheet_name='resumen_ejecutivo', index=False)
    
    # Hoja 2: Serie temporal semanal por servicio
    pivot_services.to_excel(writer, sheet_name='weekly_by_service')
    
    # Hoja 3: Serie temporal por categoría
    pivot_category.to_excel(writer, sheet_name='weekly_by_category')
    
    # Hoja 4: Variación WoW
    wow_change.to_excel(writer, sheet_name='wow_change_pct')
    
    # Hoja 5: Anomalías detectadas
    if not df_anomalies.empty:
        df_anomalies.to_excel(writer, sheet_name='anomalias', index=False)
    
    # Hoja 6: Top servicios por crecimiento
    growth_df = pd.DataFrame({
        'service': growth.index,
        'growth_pct': growth.values,
        'first_week_cost': [first_week[s] for s in growth.index],
        'last_week_cost': [last_week[s] for s in growth.index]
    })
    growth_df.to_excel(writer, sheet_name='growth_ranking', index=False)

print(f"\n{'='*50}")
print(f"✅ REPORTE EXPORTADO EXITOSAMENTE")
print(f"{'='*50}")
print(f"Archivo: {output_excel}")
print(f"Hojas: resumen_ejecutivo, weekly_by_service, weekly_by_category,")
print(f"        wow_change_pct, anomalias, growth_ranking")
```

3. Verifica la integridad del archivo generado:

```python
# Verificación de integridad
verify = pd.ExcelFile(output_excel)
print(f"\nVerificación del archivo:")
print(f"  Hojas encontradas: {verify.sheet_names}")
for sheet in verify.sheet_names:
    df_check = pd.read_excel(output_excel, sheet_name=sheet)
    print(f"  - {sheet}: {df_check.shape[0]} filas x {df_check.shape[1]} columnas")

print(f"\n✅ Archivo listo para uso en Lab 03-00-01 y Lab 04-00-01")
```

**Resultado esperado:**

```
✅ REPORTE EXPORTADO EXITOSAMENTE
==================================================
Archivo: ../data/processed/novatech_trend_report.xlsx
Hojas: resumen_ejecutivo, weekly_by_service, weekly_by_category,
        wow_change_pct, anomalias, growth_ranking

Verificación del archivo:
  Hojas encontradas: ['resumen_ejecutivo', 'weekly_by_service', 'weekly_by_category', 'wow_change_pct', 'anomalias', 'growth_ranking']
  - resumen_ejecutivo: 10 filas x 2 columnas
  - weekly_by_service: 17 filas x 12 columnas
  - weekly_by_category: 17 filas x 3 columnas
  - wow_change_pct: 17 filas x 12 columnas
  - anomalias: 5-12 filas x 7 columnas
  - growth_ranking: 12 filas x 4 columnas
```

**Verificación:** Abre el archivo en LibreOffice Calc y confirma que todas las hojas contienen datos formateados correctamente, sin celdas vacías inesperadas.

---

## Validación y Pruebas

Ejecuta las siguientes verificaciones finales en una celda del notebook:

```python
import os

print("="*60)
print("VALIDACIÓN FINAL DEL LABORATORIO 02-00-02")
print("="*60)

# Verificar archivos de salida
checks = {
    'Reporte Excel': '../data/processed/novatech_trend_report.xlsx',
    'Gráfica tendencias Top 5': '../outputs/lab_02_02/trend_line_top5.png',
    'Gráfica barras categoría': '../outputs/lab_02_02/stacked_bar_category.png',
    'Gráfica anomalías': '../outputs/lab_02_02/anomalies_highlight.png'
}

all_passed = True
for name, path in checks.items():
    exists = os.path.exists(path)
    size = os.path.getsize(path) if exists else 0
    status = "✅" if exists and size > 0 else "❌"
    if not exists or size == 0:
        all_passed = False
    print(f"  {status} {name}: {'OK' if exists else 'NO ENCONTRADO'} ({size/1024:.1f} KB)")

# Verificar contenido del reporte
print(f"\n--- Verificación de contenido ---")
xl = pd.ExcelFile('../data/processed/novatech_trend_report.xlsx')
assert len(xl.sheet_names) >= 5, "Faltan hojas en el reporte"
print(f"  ✅ Reporte tiene {len(xl.sheet_names)} hojas (mínimo 5)")

anomalias_df = pd.read_excel(xl, sheet_name='anomalias')
assert len(anomalias_df) > 0, "No se detectaron anomalías"
print(f"  ✅ {len(anomalias_df)} anomalías detectadas con z-score > 2.0")

weekly_df = pd.read_excel(xl, sheet_name='weekly_by_service')
assert weekly_df.shape[0] >= 15, "Insuficientes semanas en el análisis"
print(f"  ✅ {weekly_df.shape[0]} semanas analizadas (mínimo 15)")

print(f"\n{'='*60}")
if all_passed:
    print("🎉 LABORATORIO COMPLETADO EXITOSAMENTE")
else:
    print("⚠️  LABORATORIO INCOMPLETO - Revisa los archivos faltantes")
print("="*60)
```

### Criterios de aceptación

| Criterio | Condición |
|----------|-----------|
| Dataset consolidado | 4 meses (Oct 2023 – Ene 2024) con ≥3,000 registros |
| Agrupación semanal | ≥15 semanas con datos por servicio |
| Variación WoW | Calculada para todos los servicios sin errores |
| Anomalías | Al menos 3 detectadas con |z| > 2.0 |
| PNG exportados | 3 archivos, cada uno > 50 KB |
| Excel exportado | 6 hojas con datos coherentes |

## Resolución de Problemas

### Problema 1: Error al cargar `novatech_billing_summary_jan2024.xlsx`

**Síntomas:**
```
FileNotFoundError: [Errno 2] No such file or directory: '../data/processed/novatech_billing_summary_jan2024.xlsx'
```
O bien:
```
ValueError: Worksheet named 'daily_detail' not found
```

**Causa:** El laboratorio 02-00-01 no fue completado o el archivo se guardó con un nombre de hoja diferente al esperado.

**Solución:**

```python
# Opción 1: Verificar qué hojas tiene el archivo
import os
path = '../data/processed/novatech_billing_summary_jan2024.xlsx'
if os.path.exists(path):
    xl = pd.ExcelFile(path)
    print(f"Hojas disponibles: {xl.sheet_names}")
    # Usar la primera hoja disponible
    df_jan = pd.read_excel(path, sheet_name=xl.sheet_names[0])
else:
    # Opción 2: Solicitar archivo al instructor
    print("❌ Archivo no encontrado.")
    print("   Solicita al instructor el archivo de referencia pre-completado")
    print("   y colócalo en: ~/finops-course/data/processed/")
```

Si el instructor proporciona el archivo de referencia, colocarlo en la ruta correcta y re-ejecutar la celda de carga.

---

### Problema 2: Las gráficas PNG se generan con tamaño 0 KB o no se visualizan

**Síntomas:**
- El archivo PNG existe pero tiene 0 bytes
- La gráfica aparece en blanco en Jupyter
- Error: `RuntimeError: main thread is not in main loop` (en algunos entornos)

**Causa:** Conflicto entre el backend de matplotlib y el entorno Jupyter, especialmente en sistemas sin display gráfico o cuando se usa `%matplotlib inline` incorrectamente.

**Solución:**

```python
# Agregar al inicio del notebook, ANTES de cualquier import de matplotlib
%matplotlib inline

# Si persiste el problema, forzar backend no interactivo para guardado
import matplotlib
matplotlib.use('Agg')  # Backend sin display
import matplotlib.pyplot as plt

# Verificar que el directorio de salida existe
import os
os.makedirs('../outputs/lab_02_02', exist_ok=True)

# Después de cada plt.savefig(), verificar:
output_path = '../outputs/lab_02_02/trend_line_top5.png'
plt.savefig(output_path, dpi=150, bbox_inches='tight', facecolor='white')
plt.close()  # Cerrar figura explícitamente para liberar memoria

# Verificar tamaño
size = os.path.getsize(output_path)
print(f"Archivo guardado: {size/1024:.1f} KB")
if size < 10:
    print("⚠️ Archivo sospechosamente pequeño. Reinicia el kernel de Jupyter.")
```

Si el problema persiste, reiniciar el kernel de Jupyter (`Kernel → Restart`) y ejecutar todas las celdas desde el inicio.

## Limpieza

Este laboratorio genera archivos que son **requeridos por laboratorios posteriores**. No elimines los archivos de salida.

```python
# NO ejecutar si planeas continuar con Lab 03-00-01 y Lab 04-00-01
# Este bloque es solo para limpieza final del curso

# import shutil
# shutil.rmtree('../outputs/lab_02_02/', ignore_errors=True)
# os.remove('../data/processed/novatech_trend_report.xlsx')
# print("Archivos de laboratorio eliminados")
```

**Archivos que deben preservarse para laboratorios futuros:**

| Archivo | Requerido por |
|---------|---------------|
| `~/finops-course/data/processed/novatech_trend_report.xlsx` | Lab 03-00-01, Lab 04-00-01 |
| `~/finops-course/outputs/lab_02_02/trend_line_top5.png` | Lab 04-00-01 (tablero FinOps) |
| `~/finops-course/outputs/lab_02_02/stacked_bar_category.png` | Lab 04-00-01 (tablero FinOps) |
| `~/finops-course/outputs/lab_02_02/anomalies_highlight.png` | Lab 04-00-01 (tablero FinOps) |

## Resumen

En este laboratorio aplicaste técnicas de análisis de series de tiempo a datos reales de facturación cloud de NovaTech SRL:

| Concepto aplicado | Resultado |
|-------------------|-----------|
| Agrupación temporal semanal | Serie de 17+ semanas por servicio y categoría |
| Variación WoW | Identificación de top 5 servicios con mayor crecimiento |
| Z-score para anomalías | Detección automática de picos/caídas con umbral |z| > 2.0 |
| Mix IaaS/PaaS/SaaS | Cuantificación de la evolución del portafolio de servicios |
| Visualización profesional | 3 gráficas PNG exportadas para comunicación ejecutiva |

### Conexión con la lección teórica

Los hallazgos de este laboratorio validan los conceptos de la Lección 2.1:

- **IaaS** (mayor % del gasto) ofrece la mayor oportunidad de optimización por su granularidad y control
- **PaaS** (crecimiento acelerado) refleja la tendencia de NovaTech hacia servicios gestionados, donde el control se reduce pero la agilidad aumenta
- **SaaS** (menor %, estable) presenta costos predecibles pero requiere auditoría de licencias

### Recursos adicionales

- [FinOps Foundation — Anomaly Detection](https://www.finops.org/framework/capabilities/anomaly-management/)
- [pandas — Time Series / Date Functionality](https://pandas.pydata.org/docs/user_guide/timeseries.html)
- [matplotlib — Tutorials](https://matplotlib.org/stable/tutorials/index.html)
- [Wikipedia — Standard Score (Z-Score)](https://en.wikipedia.org/wiki/Standard_score)

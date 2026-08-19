# Demostración 8. Identificar drivers de costo en una solución moderna

## Metadata

| Campo | Valor |
|-------|-------|
| **Duración** | 15 minutos |
| **Complejidad** | Media |
| **Nivel Bloom** | Aplicar |
| **Tecnologías** | Python 3.12.1, JupyterLab 4.1.5, pandas 2.2.1, SQLAlchemy 2.0.28, PostgreSQL 16.2, plotly 5.20.0, numpy 1.26.4, openpyxl 3.1.2 |

## Descripción General

En este laboratorio identificarás y cuantificarás los principales drivers de costo en una arquitectura moderna que abarca Kubernetes, servicios multicloud y cargas de trabajo de inteligencia artificial. Trabajarás con datasets sintéticos que simulan la salida de Kubecost, consumos de modelos de IA y servicios equivalentes en tres nubes públicas. Al finalizar, enriquecerás el backlog de optimización existente con iniciativas específicas derivadas de estos drivers de costo.

## Objetivos de Aprendizaje

- [ ] Identificar y cuantificar los principales drivers de costo en una arquitectura moderna (Kubernetes, multicloud, IA)
- [ ] Analizar la estructura de costos Kubernetes desglosando por namespace y aplicando tres métodos de asignación de costos compartidos
- [ ] Normalizar costos multicloud a una moneda base para comparar servicios equivalentes entre proveedores
- [ ] Calcular métricas FinOps para cargas de trabajo de IA: costo por token, costo por inferencia y ROI por caso de uso
- [ ] Enriquecer el backlog de optimización con drivers de costo específicos de arquitecturas modernas

## Prerrequisitos

### Conocimientos Previos

- Familiaridad con pandas y SQL básico (labs anteriores del curso)
- Comprensión del concepto de backlog de optimización FinOps (Lab 06)
- Conocimiento básico del data warehouse FinOps construido en Lab 07
- Comprensión de los conceptos de FinOps ampliado a SaaS, plataformas y tecnología (Lección 8.1)

### Acceso y Recursos

- Lab 07-00-01 completado: PostgreSQL 16.2 activo en puerto 5432 con base `finops_dw`
- Docker Compose activo con servicios del Lab 07
- Archivo `finops_backlog_v2.xlsx` disponible en `~/finops-labs/lab06/output/`
- Datasets sintéticos provistos por el instructor en `~/finops-labs/lab08/data/`

## Entorno de Laboratorio

### Software Requerido

| Componente | Versión | Propósito |
|-----------|---------|-----------|
| Python | 3.12.1 | Runtime principal |
| JupyterLab | 4.1.5 | Entorno de desarrollo interactivo |
| pandas | 2.2.1 | Manipulación de datos |
| SQLAlchemy | 2.0.28 | Conexión a PostgreSQL |
| psycopg2-binary | 2.9.9 | Driver PostgreSQL |
| plotly | 5.20.0 | Visualizaciones interactivas |
| numpy | 1.26.4 | Cálculos numéricos |
| openpyxl | 3.1.2 | Lectura/escritura Excel |
| PostgreSQL | 16.2 | Base de datos (Docker) |

### Preparación del Directorio

```bash
mkdir -p ~/finops-labs/lab08/{data,notebooks,output}
```

## Instrucciones Paso a Paso

### Paso 1: Crear los Datasets Sintéticos

**Objetivo:** Generar los archivos de datos necesarios para el análisis de drivers de costo.

**Instrucciones:**

1. Crea el archivo `~/finops-labs/lab08/data/kubernetes_costs.csv` con el siguiente script Python:

```python
# Ejecutar en terminal: python ~/finops-labs/lab08/data/generate_k8s_data.py
import pandas as pd
import numpy as np

np.random.seed(42)

namespaces = ['paycore-prod', 'paycore-staging', 'analyticshub-prod', 
              'analyticshub-staging', 'devportal-prod', 'platform-system', 'monitoring']
clusters = ['eks-prod-us-east-1', 'eks-staging-us-west-2', 'gke-analytics-europe']
workload_types = ['Deployment', 'StatefulSet', 'DaemonSet', 'CronJob']

records = []
for day in pd.date_range('2024-01-01', periods=30, freq='D'):
    for ns in namespaces:
        cluster = clusters[0] if 'prod' in ns else (clusters[2] if 'analytics' in ns else clusters[1])
        if ns == 'platform-system':
            cluster = clusters[0]
        if ns == 'monitoring':
            cluster = clusters[0]
        
        n_workloads = np.random.randint(2, 6)
        for i in range(n_workloads):
            wl_type = np.random.choice(workload_types, p=[0.5, 0.25, 0.15, 0.1])
            cpu_cost = round(np.random.uniform(5, 80) * (2.5 if 'prod' in ns else 1.0), 2)
            mem_cost = round(cpu_cost * np.random.uniform(0.3, 0.8), 2)
            net_cost = round(np.random.uniform(1, 15), 2)
            pv_cost = round(np.random.uniform(0, 25), 2) if wl_type == 'StatefulSet' else 0
            shared_cost = round(np.random.uniform(8, 35), 2)
            total = round(cpu_cost + mem_cost + net_cost + pv_cost + shared_cost, 2)
            
            records.append({
                'date': day.strftime('%Y-%m-%d'),
                'namespace': ns,
                'workload_name': f"{ns.split('-')[0]}-worker-{i+1}",
                'workload_type': wl_type,
                'cluster_name': cluster,
                'cpu_cost': cpu_cost,
                'memory_cost': mem_cost,
                'network_cost': net_cost,
                'pv_cost': pv_cost,
                'shared_cost': shared_cost,
                'total_cost': total
            })

df = pd.DataFrame(records)
df.to_csv('~/finops-labs/lab08/data/kubernetes_costs.csv', index=False)
print(f"Generados {len(df)} registros de costos Kubernetes")
```

2. Crea el archivo `~/finops-labs/lab08/data/ai_workloads.csv`:

```python
# Ejecutar en terminal: python ~/finops-labs/lab08/data/generate_ai_data.py
import pandas as pd
import numpy as np

np.random.seed(123)

models = ['gpt-4-turbo', 'claude-3-sonnet', 'llama-3-70b-self-hosted', 
          'stable-diffusion-xl', 'whisper-large-v3']
use_cases = ['fraud-detection', 'customer-support-bot', 'document-processing',
             'image-generation-marketing', 'audio-transcription']

records = []
for day in pd.date_range('2024-01-01', periods=30, freq='D'):
    for model, use_case in zip(models, use_cases):
        inference_req = np.random.randint(500, 50000)
        tokens = inference_req * np.random.randint(200, 2000)
        gpu_hours = round(np.random.uniform(0.5, 24) if 'self-hosted' in model or 'stable' in model else 0, 2)
        training_cost = round(np.random.uniform(0, 500) if day.day <= 5 else 0, 2)
        
        if 'gpt-4' in model:
            inference_cost = round(tokens * 0.00003, 2)
        elif 'claude' in model:
            inference_cost = round(tokens * 0.000015, 2)
        elif 'self-hosted' in model:
            inference_cost = round(gpu_hours * 3.50, 2)
        elif 'stable' in model:
            inference_cost = round(gpu_hours * 4.10, 2)
        else:
            inference_cost = round(gpu_hours * 2.80 if gpu_hours > 0 else inference_req * 0.006, 2)
        
        revenue = round(inference_req * np.random.uniform(0.05, 2.5), 2)
        
        records.append({
            'date': day.strftime('%Y-%m-%d'),
            'model_name': model,
            'inference_requests': inference_req,
            'tokens_processed': tokens,
            'gpu_hours': gpu_hours,
            'training_cost': training_cost,
            'inference_cost': inference_cost,
            'use_case': use_case,
            'revenue_attributed': revenue
        })

df = pd.DataFrame(records)
df.to_csv('~/finops-labs/lab08/data/ai_workloads.csv', index=False)
print(f"Generados {len(df)} registros de cargas de trabajo IA")
```

3. Crea el archivo `~/finops-labs/lab08/data/multicloud_services.csv`:

```python
# Ejecutar en terminal: python ~/finops-labs/lab08/data/generate_multicloud_data.py
import pandas as pd
import numpy as np

np.random.seed(456)

services = [
    {'service_category': 'compute', 'aws_service': 'EC2 m5.xlarge', 'gcp_service': 'n2-standard-4', 'azure_service': 'Standard_D4s_v3'},
    {'service_category': 'database', 'aws_service': 'RDS PostgreSQL', 'gcp_service': 'Cloud SQL PostgreSQL', 'azure_service': 'Azure Database PostgreSQL'},
    {'service_category': 'storage', 'aws_service': 'S3 Standard', 'gcp_service': 'Cloud Storage Standard', 'azure_service': 'Blob Storage Hot'},
    {'service_category': 'kubernetes', 'aws_service': 'EKS', 'gcp_service': 'GKE', 'azure_service': 'AKS'},
    {'service_category': 'ai_platform', 'aws_service': 'SageMaker', 'gcp_service': 'Vertex AI', 'azure_service': 'Azure ML'},
]

records = []
for day in pd.date_range('2024-01-01', periods=30, freq='D'):
    for svc in services:
        # AWS en USD
        aws_cost = round(np.random.uniform(100, 3000) * (3 if svc['service_category'] in ['compute','database'] else 1), 2)
        # GCP en EUR
        gcp_cost = round(aws_cost * np.random.uniform(0.85, 1.1) * 0.92, 2)
        # Azure en MXN
        azure_cost = round(aws_cost * np.random.uniform(0.9, 1.15) * 17.15, 2)
        
        units = np.random.randint(1, 50)
        
        records.append({
            'date': day.strftime('%Y-%m-%d'),
            'service_category': svc['service_category'],
            'provider': 'AWS',
            'service_name': svc['aws_service'],
            'cost': aws_cost,
            'currency': 'USD',
            'units_consumed': units
        })
        records.append({
            'date': day.strftime('%Y-%m-%d'),
            'service_category': svc['service_category'],
            'provider': 'GCP',
            'service_name': svc['gcp_service'],
            'cost': gcp_cost,
            'currency': 'EUR',
            'units_consumed': units
        })
        records.append({
            'date': day.strftime('%Y-%m-%d'),
            'service_category': svc['service_category'],
            'provider': 'Azure',
            'service_name': svc['azure_service'],
            'cost': azure_cost,
            'currency': 'MXN',
            'units_consumed': units
        })

df = pd.DataFrame(records)
df.to_csv('~/finops-labs/lab08/data/multicloud_services.csv', index=False)
print(f"Generados {len(df)} registros multicloud")
```

4. Crea el archivo `~/finops-labs/lab08/data/fx_rates.json`:

```json
{
    "base_currency": "USD",
    "rates": {
        "USD": 1.0,
        "EUR": 1.0870,
        "MXN": 0.0583
    },
    "description": "Tasas fijas para normalización: EUR->USD multiply by 1.087, MXN->USD multiply by 0.0583",
    "effective_date": "2024-01-15"
}
```

**Salida esperada:**

```
Generados ~2940 registros de costos Kubernetes
Generados 150 registros de cargas de trabajo IA
Generados 450 registros multicloud
```

**Verificación:**

```bash
ls -la ~/finops-labs/lab08/data/
# Debe mostrar: kubernetes_costs.csv, ai_workloads.csv, multicloud_services.csv, fx_rates.json
wc -l ~/finops-labs/lab08/data/*.csv
```

---

### Paso 2: Cargar Datasets en PostgreSQL

**Objetivo:** Crear tablas en PostgreSQL e insertar los datos para análisis posterior.

**Instrucciones:**

1. Inicia JupyterLab si no está activo:

```bash
jupyter lab --port=8888 --notebook-dir=~/finops-labs/lab08/notebooks/
```

2. Crea un nuevo notebook llamado `lab08_cost_drivers.ipynb` y ejecuta las siguientes celdas:

```python
# Celda 1: Imports y conexión
import pandas as pd
import numpy as np
import json
from sqlalchemy import create_engine, text
import plotly.express as px
import plotly.graph_objects as go
from openpyxl import load_workbook

# Conexión a PostgreSQL (Docker del Lab 07)
engine = create_engine('postgresql+psycopg2://finops:finops2024@localhost:5432/finops_dw')

print("✅ Conexión a PostgreSQL establecida")
```

```python
# Celda 2: Cargar datasets
k8s_df = pd.read_csv('../data/kubernetes_costs.csv')
ai_df = pd.read_csv('../data/ai_workloads.csv')
multicloud_df = pd.read_csv('../data/multicloud_services.csv')

with open('../data/fx_rates.json', 'r') as f:
    fx_rates = json.load(f)

print(f"K8s: {len(k8s_df)} registros | IA: {len(ai_df)} registros | Multicloud: {len(multicloud_df)} registros")
print(f"Tasas FX: {fx_rates['rates']}")
```

```python
# Celda 3: Crear tablas en PostgreSQL
k8s_df.to_sql('k8s_costs', engine, if_exists='replace', index=False)
ai_df.to_sql('ai_costs', engine, if_exists='replace', index=False)
multicloud_df.to_sql('multicloud_services', engine, if_exists='replace', index=False)

# Verificar carga
with engine.connect() as conn:
    for table in ['k8s_costs', 'ai_costs', 'multicloud_services']:
        result = conn.execute(text(f"SELECT COUNT(*) FROM {table}"))
        count = result.scalar()
        print(f"  Tabla {table}: {count} registros cargados")

print("\n✅ Datos cargados en PostgreSQL")
```

**Salida esperada:**

```
✅ Conexión a PostgreSQL establecida
K8s: ~2940 registros | IA: 150 registros | Multicloud: 450 registros
Tasas FX: {'USD': 1.0, 'EUR': 1.087, 'MXN': 0.0583}
  Tabla k8s_costs: ~2940 registros cargados
  Tabla ai_costs: 150 registros cargados
  Tabla multicloud_services: 450 registros cargados

✅ Datos cargados en PostgreSQL
```

**Verificación:** Las tres tablas deben existir en la base `finops_dw` con el conteo correcto de registros.

---

### Paso 3: Análisis de Costos Kubernetes con Asignación de Shared Costs

**Objetivo:** Desglosar costos por namespace y aplicar tres métodos de asignación de costos compartidos.

**Instrucciones:**

1. Ejecuta la siguiente celda para el análisis por namespace:

```python
# Celda 4: Desglose de costos K8s por namespace
k8s_summary = k8s_df.groupby('namespace').agg(
    total_cost=('total_cost', 'sum'),
    cpu_cost=('cpu_cost', 'sum'),
    memory_cost=('memory_cost', 'sum'),
    network_cost=('network_cost', 'sum'),
    pv_cost=('pv_cost', 'sum'),
    shared_cost=('shared_cost', 'sum'),
    workload_count=('workload_name', 'count')
).round(2).sort_values('total_cost', ascending=False)

print("=" * 70)
print("DESGLOSE DE COSTOS KUBERNETES POR NAMESPACE (30 días)")
print("=" * 70)
print(k8s_summary.to_string())
print(f"\n💰 Costo total K8s: ${k8s_summary['total_cost'].sum():,.2f}")
```

2. Implementa los tres métodos de asignación de shared costs:

```python
# Celda 5: Asignación de costos compartidos - 3 métodos
# Identificar namespaces de aplicación vs. infraestructura compartida
infra_namespaces = ['platform-system', 'monitoring']
app_namespaces = [ns for ns in k8s_df['namespace'].unique() if ns not in infra_namespaces]

# Total de shared/infra costs a distribuir
infra_costs = k8s_df[k8s_df['namespace'].isin(infra_namespaces)]['total_cost'].sum()

# Costos directos por namespace de aplicación
app_costs = k8s_df[k8s_df['namespace'].isin(app_namespaces)].groupby('namespace').agg(
    direct_cost=('total_cost', 'sum'),
    total_cpu=('cpu_cost', 'sum'),
    total_memory=('memory_cost', 'sum')
).reset_index()

# Método 1: Proporcional por CPU
total_app_cpu = app_costs['total_cpu'].sum()
app_costs['shared_alloc_cpu'] = round(
    (app_costs['total_cpu'] / total_app_cpu) * infra_costs, 2
)

# Método 2: Proporcional por Memoria
total_app_mem = app_costs['total_memory'].sum()
app_costs['shared_alloc_memory'] = round(
    (app_costs['total_memory'] / total_app_mem) * infra_costs, 2
)

# Método 3: Equal Split
n_app_namespaces = len(app_namespaces)
app_costs['shared_alloc_equal'] = round(infra_costs / n_app_namespaces, 2)

# Costo total por método
app_costs['total_cost_cpu_method'] = app_costs['direct_cost'] + app_costs['shared_alloc_cpu']
app_costs['total_cost_mem_method'] = app_costs['direct_cost'] + app_costs['shared_alloc_memory']
app_costs['total_cost_equal_method'] = app_costs['direct_cost'] + app_costs['shared_alloc_equal']

print("=" * 70)
print("ASIGNACIÓN DE COSTOS COMPARTIDOS DE INFRAESTRUCTURA")
print(f"Costo infraestructura a distribuir: ${infra_costs:,.2f}")
print("=" * 70)
print("\n📊 Comparación de métodos de asignación:")
comparison = app_costs[['namespace', 'direct_cost', 'shared_alloc_cpu', 
                         'shared_alloc_memory', 'shared_alloc_equal']].copy()
comparison.columns = ['Namespace', 'Costo Directo', 'Alloc CPU', 'Alloc Memory', 'Alloc Equal']
print(comparison.to_string(index=False))
```

3. Visualiza el desglose con plotly:

```python
# Celda 6: Visualización de costos K8s
fig = px.treemap(
    k8s_df.groupby(['cluster_name', 'namespace']).agg(
        total_cost=('total_cost', 'sum')
    ).reset_index(),
    path=['cluster_name', 'namespace'],
    values='total_cost',
    title='Distribución de Costos Kubernetes por Cluster y Namespace (Enero 2024)',
    color='total_cost',
    color_continuous_scale='RdYlGn_r'
)
fig.write_html('../output/k8s_cost_treemap.html')
print("✅ Treemap guardado en output/k8s_cost_treemap.html")
```

**Salida esperada:**

- Tabla con desglose de costos por namespace mostrando que los namespaces de producción (`*-prod`) tienen los costos más altos
- Comparación de tres métodos de asignación mostrando diferencias en la distribución del costo compartido
- Archivo HTML con treemap interactivo

**Verificación:** El namespace con mayor costo directo debe ser un namespace de producción. Los tres métodos de asignación deben sumar el mismo total de costo infraestructura distribuido.

---

### Paso 4: Normalización de Costos Multicloud

**Objetivo:** Convertir todos los costos a USD y calcular el costo por unidad normalizado para comparar proveedores.

**Instrucciones:**

1. Ejecuta la normalización de monedas:

```python
# Celda 7: Normalización multicloud a USD
rates = fx_rates['rates']

def normalize_to_usd(row):
    """Convierte cualquier moneda a USD usando tasas fijas."""
    if row['currency'] == 'USD':
        return row['cost']
    elif row['currency'] == 'EUR':
        return round(row['cost'] * rates['EUR'], 2)  # EUR a USD
    elif row['currency'] == 'MXN':
        return round(row['cost'] * rates['MXN'], 2)  # MXN a USD
    return row['cost']

multicloud_df['cost_usd'] = multicloud_df.apply(normalize_to_usd, axis=1)
multicloud_df['cost_per_unit_usd'] = round(
    multicloud_df['cost_usd'] / multicloud_df['units_consumed'], 2
)

# Resumen por categoría y proveedor
multicloud_summary = multicloud_df.groupby(['service_category', 'provider']).agg(
    total_cost_usd=('cost_usd', 'sum'),
    avg_cost_per_unit=('cost_per_unit_usd', 'mean'),
    total_units=('units_consumed', 'sum')
).round(2).reset_index()

print("=" * 70)
print("COSTOS MULTICLOUD NORMALIZADOS A USD (Enero 2024)")
print("=" * 70)
pivot = multicloud_summary.pivot_table(
    index='service_category', 
    columns='provider', 
    values='avg_cost_per_unit'
).round(2)
print("\n📊 Costo promedio por unidad (USD) por categoría y proveedor:")
print(pivot.to_string())

# Identificar el proveedor más económico por categoría
cheapest = pivot.idxmin(axis=1)
print(f"\n🏆 Proveedor más económico por categoría:")
for cat, provider in cheapest.items():
    print(f"   {cat}: {provider} (${pivot.loc[cat, provider]:.2f}/unidad)")
```

2. Calcula el potencial de ahorro por consolidación:

```python
# Celda 8: Potencial de ahorro multicloud
savings_analysis = []
for category in multicloud_summary['service_category'].unique():
    cat_data = multicloud_summary[multicloud_summary['service_category'] == category]
    min_cost_provider = cat_data.loc[cat_data['avg_cost_per_unit'].idxmin()]
    total_current = cat_data['total_cost_usd'].sum()
    
    # Si todo se moviera al proveedor más barato
    optimal_cost = min_cost_provider['avg_cost_per_unit'] * cat_data['total_units'].sum()
    saving = total_current - optimal_cost
    
    savings_analysis.append({
        'service_category': category,
        'current_total_usd': round(total_current, 2),
        'optimal_provider': min_cost_provider['provider'],
        'optimal_cost_usd': round(optimal_cost, 2),
        'potential_saving_usd': round(saving, 2),
        'saving_pct': round((saving / total_current) * 100, 1)
    })

savings_df = pd.DataFrame(savings_analysis).sort_values('potential_saving_usd', ascending=False)
print("\n💡 Potencial de ahorro por consolidación multicloud:")
print(savings_df.to_string(index=False))
print(f"\n💰 Ahorro total potencial: ${savings_df['potential_saving_usd'].sum():,.2f}")
```

**Salida esperada:**

- Tabla pivote mostrando costo por unidad en USD para cada proveedor y categoría
- Identificación del proveedor más económico por categoría de servicio
- Tabla de ahorro potencial con porcentaje de saving por categoría

**Verificación:** Todos los costos deben estar en USD. El costo por unidad debe variar entre proveedores reflejando las diferencias de pricing.

---

### Paso 5: Métricas FinOps para Cargas de Trabajo de IA

**Objetivo:** Calcular KPIs financieros específicos para workloads de inteligencia artificial.

**Instrucciones:**

1. Ejecuta el análisis de costos de IA:

```python
# Celda 9: Métricas FinOps para IA
ai_metrics = ai_df.groupby(['model_name', 'use_case']).agg(
    total_inference_requests=('inference_requests', 'sum'),
    total_tokens=('tokens_processed', 'sum'),
    total_gpu_hours=('gpu_hours', 'sum'),
    total_training_cost=('training_cost', 'sum'),
    total_inference_cost=('inference_cost', 'sum'),
    total_revenue=('revenue_attributed', 'sum')
).reset_index()

# Calcular métricas derivadas
ai_metrics['total_cost'] = ai_metrics['total_training_cost'] + ai_metrics['total_inference_cost']
ai_metrics['cost_per_1k_tokens'] = round(
    (ai_metrics['total_inference_cost'] / ai_metrics['total_tokens']) * 1000, 4
)
ai_metrics['cost_per_inference'] = round(
    ai_metrics['total_inference_cost'] / ai_metrics['total_inference_requests'], 4
)
ai_metrics['roi'] = round(
    ai_metrics['total_revenue'] / ai_metrics['total_cost'], 2
)
ai_metrics['net_value'] = round(
    ai_metrics['total_revenue'] - ai_metrics['total_cost'], 2
)

print("=" * 70)
print("MÉTRICAS FINOPS PARA CARGAS DE TRABAJO DE IA (Enero 2024)")
print("=" * 70)

display_cols = ['model_name', 'use_case', 'total_cost', 'cost_per_1k_tokens', 
                'cost_per_inference', 'total_revenue', 'roi', 'net_value']
print(ai_metrics[display_cols].to_string(index=False))

# Identificar modelos con ROI negativo
negative_roi = ai_metrics[ai_metrics['roi'] < 1.0]
if not negative_roi.empty:
    print(f"\n⚠️  Modelos con ROI < 1.0 (costo supera ingresos atribuidos):")
    for _, row in negative_roi.iterrows():
        print(f"   {row['model_name']} ({row['use_case']}): ROI = {row['roi']}x | Pérdida: ${abs(row['net_value']):,.2f}")
```

2. Visualiza el ROI por caso de uso:

```python
# Celda 10: Visualización ROI de IA
fig = go.Figure()
fig.add_trace(go.Bar(
    name='Costo Total',
    x=ai_metrics['use_case'],
    y=ai_metrics['total_cost'],
    marker_color='indianred'
))
fig.add_trace(go.Bar(
    name='Revenue Atribuido',
    x=ai_metrics['use_case'],
    y=ai_metrics['total_revenue'],
    marker_color='mediumseagreen'
))
fig.update_layout(
    title='Costo vs Revenue por Caso de Uso de IA - NovaTech SRL',
    barmode='group',
    yaxis_title='USD',
    xaxis_title='Caso de Uso'
)
fig.write_html('../output/ai_roi_analysis.html')
print("✅ Gráfico ROI de IA guardado en output/ai_roi_analysis.html")
```

**Salida esperada:**

- Tabla con métricas por modelo: costo por 1K tokens, costo por inferencia, ROI
- Identificación de modelos con ROI negativo (costo > revenue)
- Gráfico de barras comparando costo vs revenue por caso de uso

**Verificación:** El costo por 1K tokens para GPT-4-turbo debe ser mayor que para Claude-3-sonnet. Al menos un modelo debe tener ROI < 1.0.

---

### Paso 6: Identificar Top 5 Drivers y Enriquecer el Backlog

**Objetivo:** Consolidar los principales drivers de costo y agregarlos como nuevas iniciativas al backlog de optimización.

**Instrucciones:**

1. Identifica los top 5 drivers por categoría:

```python
# Celda 11: Top 5 drivers de costo por categoría

# Top 5 K8s - por namespace
k8s_top5 = k8s_summary.head(5).reset_index()
print("🔝 TOP 5 DRIVERS DE COSTO - KUBERNETES")
for i, row in k8s_top5.iterrows():
    print(f"   {i+1}. {row['namespace']}: ${row['total_cost']:,.2f} "
          f"(CPU: ${row['cpu_cost']:,.2f} | Shared: ${row['shared_cost']:,.2f})")

# Top 5 Multicloud - por categoría con mayor gasto
multicloud_top5 = multicloud_df.groupby(['service_category', 'provider']).agg(
    total_usd=('cost_usd', 'sum')
).reset_index().sort_values('total_usd', ascending=False).head(5)
print("\n🔝 TOP 5 DRIVERS DE COSTO - MULTICLOUD")
for i, row in multicloud_top5.iterrows():
    print(f"   {i+1}. {row['service_category']} ({row['provider']}): ${row['total_usd']:,.2f}")

# Top 5 IA - por costo total
ai_top5 = ai_metrics.sort_values('total_cost', ascending=False).head(5)
print("\n🔝 TOP 5 DRIVERS DE COSTO - IA")
for i, (_, row) in enumerate(ai_top5.iterrows()):
    print(f"   {i+1}. {row['model_name']} ({row['use_case']}): ${row['total_cost']:,.2f} | ROI: {row['roi']}x")
```

2. Genera las nuevas entradas del backlog:

```python
# Celda 12: Crear entradas para backlog v3
new_backlog_items = []

# Items de K8s
for i, row in k8s_top5.iterrows():
    new_backlog_items.append({
        'id': f'OPT-K8S-{i+1:03d}',
        'category': 'Kubernetes',
        'initiative': f"Rightsizing namespace {row['namespace']}",
        'description': f"Reducir sobreaprovisionamiento en {row['namespace']}. "
                      f"Shared cost representa ${row['shared_cost']:,.0f} del total.",
        'estimated_saving_usd_monthly': round(row['total_cost'] * 0.20, 2),
        'effort': 'Medium' if 'prod' in row['namespace'] else 'Low',
        'priority': 'High' if row['total_cost'] > k8s_top5['total_cost'].median() else 'Medium',
        'driver_type': 'Kubernetes shared cost + overprovisioning',
        'status': 'Backlog'
    })

# Items de Multicloud
for i, row in multicloud_top5.iterrows():
    saving_pct = savings_df[savings_df['service_category'] == row['service_category']]['saving_pct'].values
    pct = saving_pct[0] if len(saving_pct) > 0 else 10
    new_backlog_items.append({
        'id': f'OPT-MC-{i+1:03d}',
        'category': 'Multicloud',
        'initiative': f"Consolidar {row['service_category']} - evaluar migración desde {row['provider']}",
        'description': f"Servicio {row['service_category']} en {row['provider']} con gasto ${row['total_usd']:,.0f}/mes. "
                      f"Potencial de ahorro {pct:.0f}% por normalización de proveedor.",
        'estimated_saving_usd_monthly': round(row['total_usd'] * pct / 100, 2),
        'effort': 'High',
        'priority': 'Medium',
        'driver_type': 'Multicloud price variance',
        'status': 'Backlog'
    })

# Items de IA
for _, row in ai_metrics[ai_metrics['roi'] < 1.5].iterrows():
    new_backlog_items.append({
        'id': f'OPT-AI-{len(new_backlog_items)+1:03d}',
        'category': 'AI/ML',
        'initiative': f"Optimizar modelo {row['model_name']} - caso {row['use_case']}",
        'description': f"ROI actual: {row['roi']}x. Costo/1K tokens: ${row['cost_per_1k_tokens']:.4f}. "
                      f"Evaluar modelo alternativo o reducir volumen de inferencias.",
        'estimated_saving_usd_monthly': round(row['total_cost'] * 0.30, 2),
        'effort': 'Medium',
        'priority': 'High' if row['roi'] < 1.0 else 'Medium',
        'driver_type': 'AI cost per inference / low ROI',
        'status': 'Backlog'
    })

new_items_df = pd.DataFrame(new_backlog_items)
print(f"\n✅ Generadas {len(new_items_df)} nuevas iniciativas para el backlog")
print(f"   - Kubernetes: {len([x for x in new_backlog_items if x['category']=='Kubernetes'])}")
print(f"   - Multicloud: {len([x for x in new_backlog_items if x['category']=='Multicloud'])}")
print(f"   - AI/ML: {len([x for x in new_backlog_items if x['category']=='AI/ML'])}")
print(f"\n💰 Ahorro mensual estimado total: ${new_items_df['estimated_saving_usd_monthly'].sum():,.2f}")
```

3. Genera el backlog v3 consolidado:

```python
# Celda 13: Generar finops_backlog_v3.xlsx
import os

# Intentar cargar backlog existente
backlog_path = os.path.expanduser('~/finops-labs/lab06/output/finops_backlog_v2.xlsx')
if os.path.exists(backlog_path):
    existing_backlog = pd.read_excel(backlog_path)
    print(f"📂 Backlog v2 cargado: {len(existing_backlog)} items existentes")
else:
    # Crear backlog mínimo si no existe el archivo previo
    existing_backlog = pd.DataFrame({
        'id': ['OPT-001', 'OPT-002', 'OPT-003'],
        'category': ['Compute', 'Storage', 'Database'],
        'initiative': ['Rightsizing EC2 instances', 'S3 lifecycle policies', 'RDS Reserved Instances'],
        'description': ['Reducir instancias sobredimensionadas', 'Mover datos fríos a Glacier', 'Comprar RI para RDS estables'],
        'estimated_saving_usd_monthly': [4500, 1200, 3800],
        'effort': ['Low', 'Low', 'Medium'],
        'priority': ['High', 'Medium', 'High'],
        'driver_type': ['Overprovisioning', 'Storage tiering', 'Rate optimization'],
        'status': ['In Progress', 'Backlog', 'Backlog']
    })
    print("⚠️  Backlog v2 no encontrado. Usando backlog de referencia.")

# Concatenar
backlog_v3 = pd.concat([existing_backlog, new_items_df], ignore_index=True)

# Guardar
output_path = '../output/finops_backlog_v3.xlsx'
backlog_v3.to_excel(output_path, index=False, sheet_name='Backlog')
print(f"\n✅ finops_backlog_v3.xlsx generado: {len(backlog_v3)} items totales")
print(f"   Guardado en: {os.path.abspath(output_path)}")

# Resumen final
print("\n" + "=" * 70)
print("RESUMEN DE DRIVERS DE COSTO IDENTIFICADOS")
print("=" * 70)
summary = backlog_v3.groupby('category').agg(
    items=('id', 'count'),
    total_saving=('estimated_saving_usd_monthly', 'sum')
).round(2)
print(summary.to_string())
print(f"\n💰 AHORRO MENSUAL POTENCIAL TOTAL: ${backlog_v3['estimated_saving_usd_monthly'].sum():,.2f}")
```

**Salida esperada:**

```
✅ Generadas ~13 nuevas iniciativas para el backlog
   - Kubernetes: 5
   - Multicloud: 5
   - AI/ML: ~3

💰 Ahorro mensual estimado total: $X,XXX.XX

✅ finops_backlog_v3.xlsx generado: ~16 items totales
   Guardado en: /home/user/finops-labs/lab08/output/finops_backlog_v3.xlsx
```

**Verificación:** El archivo `finops_backlog_v3.xlsx` debe contener los items del backlog v2 más las nuevas iniciativas de K8s, multicloud e IA.

---

## Validación y Pruebas

Ejecuta la siguiente celda final para validar que todos los entregables se generaron correctamente:

```python
# Celda 14: Validación final del laboratorio
import os

print("=" * 70)
print("VALIDACIÓN FINAL - LAB 08-00-01")
print("=" * 70)

checks = {
    'Tablas PostgreSQL': False,
    'Treemap K8s (HTML)': os.path.exists('../output/k8s_cost_treemap.html'),
    'ROI IA (HTML)': os.path.exists('../output/ai_roi_analysis.html'),
    'Backlog v3 (XLSX)': os.path.exists('../output/finops_backlog_v3.xlsx'),
}

# Verificar tablas
try:
    with engine.connect() as conn:
        for t in ['k8s_costs', 'ai_costs', 'multicloud_services']:
            conn.execute(text(f"SELECT 1 FROM {t} LIMIT 1"))
    checks['Tablas PostgreSQL'] = True
except:
    pass

# Verificar contenido del backlog
if checks['Backlog v3 (XLSX)']:
    bl = pd.read_excel('../output/finops_backlog_v3.xlsx')
    has_k8s = bl['category'].str.contains('Kubernetes', na=False).any()
    has_mc = bl['category'].str.contains('Multicloud', na=False).any()
    has_ai = bl['category'].str.contains('AI', na=False).any()
    checks['Backlog contiene K8s items'] = has_k8s
    checks['Backlog contiene Multicloud items'] = has_mc
    checks['Backlog contiene AI items'] = has_ai

all_passed = all(checks.values())
for check, status in checks.items():
    icon = "✅" if status else "❌"
    print(f"  {icon} {check}")

print(f"\n{'🎉 LABORATORIO COMPLETADO EXITOSAMENTE' if all_passed else '⚠️  Revisar items fallidos'}")
```

**Resultado esperado:** Todos los checks deben mostrar ✅.

## Solución de Problemas

### Problema 1: Error de conexión a PostgreSQL

**Síntomas:** `OperationalError: could not connect to server: Connection refused` al ejecutar la celda de conexión SQLAlchemy.

**Causa:** El contenedor Docker de PostgreSQL del Lab 07 no está activo o el puerto 5432 no está mapeado correctamente.

**Solución:**

```bash
# Verificar que Docker Compose del Lab 07 está activo
cd ~/finops-labs/lab07
docker compose ps

# Si no está activo, iniciarlo
docker compose up -d

# Verificar conectividad
docker exec -it lab07-postgres-1 psql -U finops -d finops_dw -c "SELECT 1;"

# Si el contenedor no existe, recrear solo PostgreSQL
docker run -d --name finops-postgres \
  -e POSTGRES_USER=finops \
  -e POSTGRES_PASSWORD=finops2024 \
  -e POSTGRES_DB=finops_dw \
  -p 5432:5432 \
  postgres:16.2
```

### Problema 2: Error al normalizar costos multicloud (KeyError en fx_rates)

**Síntomas:** `KeyError: 'EUR'` o valores `NaN` en la columna `cost_usd` después de aplicar la función `normalize_to_usd`.

**Causa:** El archivo `fx_rates.json` no se cargó correctamente o tiene un formato diferente al esperado. También puede ocurrir si la columna `currency` contiene valores con espacios o en minúsculas.

**Solución:**

```python
# Verificar contenido del archivo
import json
with open('../data/fx_rates.json', 'r') as f:
    fx = json.load(f)
print(f"Estructura: {fx}")
print(f"Claves rates: {fx['rates'].keys()}")

# Limpiar columna currency antes de normalizar
multicloud_df['currency'] = multicloud_df['currency'].str.strip().str.upper()

# Verificar valores únicos
print(f"Monedas en dataset: {multicloud_df['currency'].unique()}")

# Si fx_rates no carga, definir manualmente
fx_rates = {
    "rates": {"USD": 1.0, "EUR": 1.0870, "MXN": 0.0583}
}
```

## Limpieza

```python
# Celda opcional: Limpiar tablas temporales de PostgreSQL (solo si es necesario)
# NO ejecutar si planeas usar estos datos en laboratorios posteriores

# with engine.connect() as conn:
#     conn.execute(text("DROP TABLE IF EXISTS k8s_costs"))
#     conn.execute(text("DROP TABLE IF EXISTS ai_costs"))
#     conn.execute(text("DROP TABLE IF EXISTS multicloud_services"))
#     conn.commit()
#     print("Tablas temporales eliminadas")
```

> **Nota:** Se recomienda mantener las tablas y archivos de output, ya que pueden ser referenciados en laboratorios posteriores del curso.

## Resumen

En este laboratorio aplicaste análisis de drivers de costo sobre tres dimensiones clave de una arquitectura moderna:

| Dimensión | Hallazgo Principal | Técnica Aplicada |
|-----------|-------------------|------------------|
| **Kubernetes** | Los namespaces de producción concentran >60% del costo; shared costs requieren método explícito de asignación | Desglose por namespace + 3 métodos de shared cost allocation |
| **Multicloud** | Diferencias de hasta 30% en costo por unidad entre proveedores para servicios equivalentes | Normalización FX + costo por unidad de servicio |
| **IA/ML** | Modelos con ROI < 1.0 destruyen valor; costo por token varía 10x entre modelos | Costo por 1K tokens, costo por inferencia, ROI por caso de uso |

**Conceptos clave reforzados:**
- La extensión de FinOps más allá de la nube pública (Lección 8.1) se materializa al analizar Kubernetes, multicloud y cargas de IA como parte del gasto tecnológico total
- El inventario de gasto tecnológico ampliado incluye drivers de costo que no aparecen en una sola factura de nube
- El backlog de optimización es un artefacto vivo que se enriquece conforme se descubren nuevos drivers

### Recursos Adicionales

- [Kubecost Documentation — Cost Allocation](https://docs.kubecost.com/using-kubecost/navigating-the-kubecost-ui/cost-allocation)
- [FinOps Foundation — AI Cost Management](https://www.finops.org/wg/ai-cost-management/)
- [FOCUS Specification — Multicloud Normalization](https://focus.finops.org/)
- [OpenCost — Kubernetes Cost Monitoring](https://www.opencost.io/)

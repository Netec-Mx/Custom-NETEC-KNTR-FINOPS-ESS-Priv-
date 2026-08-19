# Demostración 5. Backlog de optimización y priorización por impacto

## Metadatos

| Campo | Valor |
|-------|-------|
| **Duración** | 30 minutos |
| **Complejidad** | Media |
| **Nivel Bloom** | Crear |
| **Tecnologías** | Python 3.12.1, JupyterLab 4.1.5, pandas 2.2.1, matplotlib 3.8.3, openpyxl 3.1.2, numpy 1.26.4, Faker 24.2.0 |

## Descripción General

En este laboratorio construirás un backlog estructurado de optimización de costos cloud para NovaTech SRL. Partiendo de un dataset sintético de recursos cloud con métricas de utilización, identificarás oportunidades de ahorro en cinco categorías (inactivos, sobredimensionados, huérfanos, rate optimization y storage), las priorizarás mediante una matriz de impacto financiero vs. esfuerzo operativo, y exportarás el resultado como un reporte ejecutivo en Excel con visualizaciones de apoyo. Este artefacto será la base para laboratorios posteriores de la segunda parte del curso.

## Objetivos de Aprendizaje

- [ ] Identificar y clasificar oportunidades de optimización de costos cloud en cinco categorías operativas usando reglas de detección programáticas.
- [ ] Construir un backlog de optimización con scoring de priorización basado en la relación ahorro estimado / esfuerzo de implementación.
- [ ] Calcular el ROI estimado de cada iniciativa diferenciando entre optimización de costos (eficiencia) y reducción de costos (recorte).
- [ ] Generar visualizaciones de priorización (bubble chart y bar chart) que comuniquen el backlog a audiencias ejecutivas.
- [ ] Exportar el backlog completo a formato Excel profesional usando openpyxl.

## Prerrequisitos

### Conocimientos previos

| Requisito | Nivel |
|-----------|-------|
| Python básico (DataFrames, funciones, filtros) | Intermedio |
| Conceptos de recursos cloud (VMs, storage, IPs) | Básico |
| Diferencia entre optimización y reducción de costos (Lección 5.1) | Conceptual |
| Uso de JupyterLab (celdas, ejecución, markdown) | Básico |

### Acceso y software

| Software | Versión requerida |
|----------|-------------------|
| Python | 3.12.1 |
| JupyterLab | 4.1.5 |
| pandas | 2.2.1 |
| numpy | 1.26.4 |
| matplotlib | 3.8.3 |
| openpyxl | 3.1.2 |
| Faker | 24.2.0 |

## Entorno de Laboratorio

### Estructura de directorios

```
~/finops-course/
├── data/
│   ├── raw/
│   └── processed/
├── notebooks/
└── outputs/
```

### Comandos de configuración inicial

```bash
# Crear directorio específico del lab (si no existe)
mkdir -p ~/finops-course/data/raw
mkdir -p ~/finops-course/data/processed
mkdir -p ~/finops-course/notebooks
mkdir -p ~/finops-course/outputs

# Instalar dependencias (si no están instaladas)
pip install pandas==2.2.1 numpy==1.26.4 matplotlib==3.8.3 openpyxl==3.1.2 Faker==24.2.0

# Iniciar JupyterLab
jupyter lab --port=8888 --notebook-dir=~/finops-course/notebooks/
```

> **Nota:** Si el puerto 8888 está ocupado, usar `--port=8889`.

---

## Paso a Paso

### Paso 1: Generar el dataset sintético de recursos cloud

**Objetivo:** Crear un dataset realista de recursos cloud de NovaTech SRL con métricas de utilización, costos y metadatos que permitan identificar oportunidades de optimización.

**Instrucciones:**

1. En JupyterLab, crear un nuevo notebook llamado `lab05_backlog_optimizacion.ipynb`.

2. En la primera celda, agregar el encabezado markdown:

```markdown
# Lab 05: Backlog de Optimización y Priorización por Impacto
## NovaTech SRL - Análisis de Oportunidades de Ahorro Cloud
**Período:** Q4 2023 + Enero 2024  
**Proveedores:** AWS, Azure, GCP
```

3. En la segunda celda, importar las librerías necesarias:

```python
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
from faker import Faker
from datetime import datetime
import warnings
warnings.filterwarnings('ignore')

fake = Faker()
Faker.seed(42)
np.random.seed(42)

print("✅ Librerías cargadas correctamente")
print(f"   pandas: {pd.__version__}")
print(f"   numpy: {np.__version__}")
```

4. En la tercera celda, generar el dataset sintético de recursos cloud:

```python
# Configuración de NovaTech SRL
PRODUCTOS = ['PayCore', 'AnalyticsHub', 'DevPortal']
EQUIPOS = ['Backend', 'Frontend', 'Data', 'Platform']
AMBIENTES = ['prod', 'staging', 'dev']
PROVEEDORES = ['AWS', 'Azure', 'GCP']

# Tipos de recursos con distribución realista
TIPOS_RECURSO = {
    'VM': {'count': 45, 'cost_range': (50, 1200)},
    'Database': {'count': 12, 'cost_range': (100, 2500)},
    'Storage_Volume': {'count': 30, 'cost_range': (10, 300)},
    'Elastic_IP': {'count': 15, 'cost_range': (3, 15)},
    'Load_Balancer': {'count': 8, 'cost_range': (20, 200)},
    'Snapshot': {'count': 25, 'cost_range': (5, 80)},
    'Container': {'count': 20, 'cost_range': (30, 500)},
}

records = []
resource_id = 1

for tipo, config in TIPOS_RECURSO.items():
    for i in range(config['count']):
        producto = np.random.choice(PRODUCTOS, p=[0.4, 0.35, 0.25])
        equipo = np.random.choice(EQUIPOS, p=[0.3, 0.2, 0.3, 0.2])
        ambiente = np.random.choice(AMBIENTES, p=[0.4, 0.3, 0.3])
        proveedor = np.random.choice(PROVEEDORES, p=[0.5, 0.3, 0.2])
        
        costo_mensual = round(np.random.uniform(*config['cost_range']), 2)
        
        # Simular métricas de utilización con sesgo hacia ineficiencia
        if tipo == 'VM':
            cpu_avg = np.random.choice(
                [np.random.uniform(1, 5),      # Inactivo (~20%)
                 np.random.uniform(5, 30),     # Sobredimensionado (~30%)
                 np.random.uniform(30, 70),    # Normal (~35%)
                 np.random.uniform(70, 95)],   # Bien utilizado (~15%)
                p=[0.20, 0.30, 0.35, 0.15]
            )
            mem_avg = cpu_avg + np.random.uniform(-10, 15)
            mem_avg = max(2, min(98, mem_avg))
        elif tipo == 'Container':
            cpu_avg = np.random.choice(
                [np.random.uniform(2, 8),
                 np.random.uniform(10, 35),
                 np.random.uniform(40, 80)],
                p=[0.15, 0.35, 0.50]
            )
            mem_avg = cpu_avg + np.random.uniform(-5, 20)
            mem_avg = max(5, min(95, mem_avg))
        else:
            cpu_avg = None
            mem_avg = None
        
        # Simular si tiene tags completos (para detectar huérfanos)
        has_owner_tag = np.random.choice([True, False], p=[0.75, 0.25])
        
        # Simular volúmenes sin adjuntar
        attached = True
        if tipo == 'Storage_Volume':
            attached = np.random.choice([True, False], p=[0.7, 0.3])
        
        # Simular IPs no asociadas
        if tipo == 'Elastic_IP':
            attached = np.random.choice([True, False], p=[0.6, 0.4])
        
        # Simular snapshots antiguos (>90 días)
        days_since_creation = np.random.randint(1, 365)
        
        records.append({
            'resource_id': f"NVT-{tipo[:3].upper()}-{resource_id:04d}",
            'resource_type': tipo,
            'resource_name': f"{producto.lower()}-{ambiente}-{tipo.lower()}-{i+1:02d}",
            'provider': proveedor,
            'product': producto,
            'team': equipo,
            'environment': ambiente,
            'monthly_cost_usd': costo_mensual,
            'cpu_avg_pct': round(cpu_avg, 1) if cpu_avg else None,
            'mem_avg_pct': round(mem_avg, 1) if mem_avg else None,
            'has_owner_tag': has_owner_tag,
            'is_attached': attached,
            'days_since_creation': days_since_creation,
            'last_access_days_ago': np.random.randint(0, 120) if tipo in ['Storage_Volume', 'Snapshot'] else None,
        })
        resource_id += 1

df_resources = pd.DataFrame(records)
print(f"✅ Dataset generado: {len(df_resources)} recursos")
print(f"   Costo mensual total: ${df_resources['monthly_cost_usd'].sum():,.2f}")
print(f"\nDistribución por tipo de recurso:")
print(df_resources.groupby('resource_type')['monthly_cost_usd'].agg(['count', 'sum']).rename(
    columns={'count': 'Cantidad', 'sum': 'Costo_Total_USD'}).round(2))
```

5. Guardar el dataset generado:

```python
df_resources.to_csv('~/finops-course/data/raw/novatech_cloud_resources.csv', index=False)
print("✅ Dataset guardado en: ~/finops-course/data/raw/novatech_cloud_resources.csv")
```

**Salida esperada:**

```
✅ Dataset generado: 155 recursos
   Costo mensual total: $XX,XXX.XX

Distribución por tipo de recurso:
                 Cantidad  Costo_Total_USD
resource_type                              
Container             20          XXXX.XX
Database              12          XXXX.XX
Elastic_IP            15           XXX.XX
Load_Balancer          8           XXX.XX
Snapshot              25           XXX.XX
Storage_Volume        30          XXXX.XX
VM                    45         XXXXX.XX
```

**Verificación:** El DataFrame debe tener exactamente 155 filas y 14 columnas. Ejecutar `df_resources.shape` para confirmar `(155, 14)`.

---

### Paso 2: Detectar oportunidades de optimización por categoría

**Objetivo:** Aplicar reglas de detección para clasificar recursos en cinco categorías de optimización: inactivos, sobredimensionados, huérfanos, rate optimization y storage.

**Instrucciones:**

1. Crear una celda markdown de sección:

```markdown
## 2. Detección de Oportunidades por Categoría
Aplicamos reglas de negocio para clasificar recursos candidatos a optimización.
- **Inactivos:** VMs/Containers con CPU promedio < 5%
- **Sobredimensionados:** VMs/Containers con CPU promedio entre 5% y 30%
- **Huérfanos:** Recursos sin tag de propietario
- **Storage:** Volúmenes sin adjuntar, snapshots > 90 días, acceso > 60 días
- **Rate Optimization:** Recursos en prod con alto costo bajo demanda (candidatos a reserva)
```

2. Implementar las reglas de detección:

```python
# Categoría 1: Recursos INACTIVOS (CPU < 5%)
df_inactive = df_resources[
    (df_resources['resource_type'].isin(['VM', 'Container'])) &
    (df_resources['cpu_avg_pct'] < 5)
].copy()
df_inactive['optimization_category'] = 'Inactivo'
df_inactive['action'] = 'Apagar o eliminar'
df_inactive['savings_factor'] = 0.95  # Se ahorra casi todo el costo

# Categoría 2: Recursos SOBREDIMENSIONADOS (CPU 5-30%)
df_oversized = df_resources[
    (df_resources['resource_type'].isin(['VM', 'Container'])) &
    (df_resources['cpu_avg_pct'] >= 5) &
    (df_resources['cpu_avg_pct'] < 30)
].copy()
df_oversized['optimization_category'] = 'Sobredimensionado'
df_oversized['action'] = 'Rightsizing (reducir tamaño)'
df_oversized['savings_factor'] = 0.45  # Ahorro ~45% al reducir tamaño

# Categoría 3: Recursos HUÉRFANOS (sin tag de propietario)
df_orphan = df_resources[
    (df_resources['has_owner_tag'] == False) &
    (~df_resources.index.isin(df_inactive.index)) &
    (~df_resources.index.isin(df_oversized.index))
].copy()
df_orphan['optimization_category'] = 'Huérfano'
df_orphan['action'] = 'Asignar propietario o eliminar'
df_orphan['savings_factor'] = 0.60  # Estimación conservadora

# Categoría 4: STORAGE (volúmenes sin adjuntar, snapshots viejos)
df_storage = df_resources[
    ((df_resources['resource_type'] == 'Storage_Volume') & (df_resources['is_attached'] == False)) |
    ((df_resources['resource_type'] == 'Snapshot') & (df_resources['days_since_creation'] > 90)) |
    ((df_resources['resource_type'].isin(['Storage_Volume', 'Snapshot'])) & 
     (df_resources['last_access_days_ago'] > 60))
].copy()
# Eliminar duplicados con categorías anteriores
df_storage = df_storage[~df_storage.index.isin(
    pd.concat([df_inactive, df_oversized, df_orphan]).index
)]
df_storage['optimization_category'] = 'Storage'
df_storage['action'] = 'Eliminar o migrar a tier frío'
df_storage['savings_factor'] = 0.80  # Alto ahorro en storage no utilizado

# Categoría 5: RATE OPTIMIZATION (prod + alto costo + on-demand)
df_rate = df_resources[
    (df_resources['environment'] == 'prod') &
    (df_resources['resource_type'].isin(['VM', 'Database'])) &
    (df_resources['monthly_cost_usd'] > 300) &
    (~df_resources.index.isin(
        pd.concat([df_inactive, df_oversized, df_orphan, df_storage]).index
    ))
].copy()
df_rate['optimization_category'] = 'Rate_Optimization'
df_rate['action'] = 'Comprar reserva o Savings Plan'
df_rate['savings_factor'] = 0.35  # Descuento típico de RI/SP

# Resumen de detección
print("=" * 60)
print("RESUMEN DE OPORTUNIDADES DETECTADAS - NovaTech SRL")
print("=" * 60)
categories = {
    'Inactivo': df_inactive,
    'Sobredimensionado': df_oversized,
    'Huérfano': df_orphan,
    'Storage': df_storage,
    'Rate_Optimization': df_rate
}

total_savings = 0
for cat, df in categories.items():
    savings = (df['monthly_cost_usd'] * df['savings_factor']).sum()
    total_savings += savings
    print(f"\n📌 {cat}:")
    print(f"   Recursos detectados: {len(df)}")
    print(f"   Costo mensual afectado: ${df['monthly_cost_usd'].sum():,.2f}")
    print(f"   Ahorro estimado mensual: ${savings:,.2f}")

print(f"\n{'=' * 60}")
print(f"💰 AHORRO TOTAL ESTIMADO MENSUAL: ${total_savings:,.2f}")
print(f"💰 AHORRO TOTAL ESTIMADO ANUAL:   ${total_savings * 12:,.2f}")
print(f"{'=' * 60}")
```

**Salida esperada:**

```
============================================================
RESUMEN DE OPORTUNIDADES DETECTADAS - NovaTech SRL
============================================================

📌 Inactivo:
   Recursos detectados: ~12-15
   Costo mensual afectado: $X,XXX.XX
   Ahorro estimado mensual: $X,XXX.XX

📌 Sobredimensionado:
   Recursos detectados: ~18-25
   ...

📌 Huérfano:
   Recursos detectados: ~8-15
   ...

📌 Storage:
   Recursos detectados: ~15-25
   ...

📌 Rate_Optimization:
   Recursos detectados: ~5-10
   ...

============================================================
💰 AHORRO TOTAL ESTIMADO MENSUAL: $X,XXX.XX
💰 AHORRO TOTAL ESTIMADO ANUAL:   $XX,XXX.XX
============================================================
```

**Verificación:** Cada categoría debe tener al menos 3 recursos detectados. Si alguna categoría está vacía, revisar los umbrales de detección.

---

### Paso 3: Construir el DataFrame del backlog de optimización

**Objetivo:** Consolidar todas las oportunidades detectadas en un backlog estructurado con scoring de priorización, diferenciando entre optimización y reducción de costos.

**Instrucciones:**

1. Crear la celda markdown:

```markdown
## 3. Construcción del Backlog Estructurado
Cada oportunidad se documenta con: ahorro estimado, esfuerzo de implementación, 
nivel de riesgo, equipo responsable y **tipo de iniciativa** (optimización vs reducción).

### Diferencia clave (Lección 5.1):
- **Reducción:** Disminuye gasto absoluto (apagar, eliminar, reducir)
- **Optimización:** Mejora la relación valor/costo (reservas, rightsizing inteligente)
```

2. Construir el backlog consolidado:

```python
# Concatenar todas las oportunidades
df_all_opportunities = pd.concat([df_inactive, df_oversized, df_orphan, df_storage, df_rate])

# Calcular ahorro estimado por recurso
df_all_opportunities['savings_monthly_usd'] = (
    df_all_opportunities['monthly_cost_usd'] * df_all_opportunities['savings_factor']
).round(2)

# Asignar esfuerzo estimado en días por categoría
effort_map = {
    'Inactivo': 1,           # Apagar es rápido
    'Sobredimensionado': 3,  # Requiere validación de carga
    'Huérfano': 2,           # Investigar propietario
    'Storage': 1,            # Eliminar/mover es directo
    'Rate_Optimization': 5   # Análisis de compromiso + aprobación financiera
}

risk_map = {
    'Inactivo': 'Bajo',
    'Sobredimensionado': 'Medio',
    'Huérfano': 'Medio',
    'Storage': 'Bajo',
    'Rate_Optimization': 'Bajo'
}

# Clasificar tipo de iniciativa según Lección 5.1
initiative_type_map = {
    'Inactivo': 'Reducción',           # Elimina gasto sin valor
    'Sobredimensionado': 'Optimización', # Mejora eficiencia, mantiene servicio
    'Huérfano': 'Reducción',           # Elimina gasto sin propietario/valor claro
    'Storage': 'Reducción',            # Elimina gasto de almacenamiento sin uso
    'Rate_Optimization': 'Optimización' # Misma capacidad a menor costo unitario
}

df_all_opportunities['effort_days'] = df_all_opportunities['optimization_category'].map(effort_map)
df_all_opportunities['risk_level'] = df_all_opportunities['optimization_category'].map(risk_map)
df_all_opportunities['initiative_type'] = df_all_opportunities['optimization_category'].map(initiative_type_map)

# Factor de riesgo numérico para scoring
risk_factor_map = {'Bajo': 1.0, 'Medio': 0.7, 'Alto': 0.4}
df_all_opportunities['risk_factor'] = df_all_opportunities['risk_level'].map(risk_factor_map)

# Calcular score de priorización: (Ahorro / Esfuerzo) * Factor_Riesgo
df_all_opportunities['priority_score'] = (
    (df_all_opportunities['savings_monthly_usd'] / df_all_opportunities['effort_days']) 
    * df_all_opportunities['risk_factor']
).round(2)

# Construir backlog final
backlog = df_all_opportunities[[
    'resource_id', 'resource_name', 'resource_type', 'optimization_category',
    'action', 'initiative_type', 'provider', 'product', 'team', 'environment',
    'monthly_cost_usd', 'savings_monthly_usd', 'effort_days', 'risk_level',
    'priority_score'
]].copy()

# Agregar columna de estado
backlog['status'] = 'Pendiente'

# Ordenar por score de priorización (mayor primero)
backlog = backlog.sort_values('priority_score', ascending=False).reset_index(drop=True)
backlog.index = backlog.index + 1  # ID empieza en 1
backlog.index.name = 'backlog_id'

print(f"✅ Backlog construido: {len(backlog)} iniciativas")
print(f"\nTop 10 iniciativas por prioridad:")
print(backlog[['resource_id', 'optimization_category', 'initiative_type', 
               'savings_monthly_usd', 'effort_days', 'priority_score']].head(10).to_string())

print(f"\n\n📊 Resumen por tipo de iniciativa (Lección 5.1):")
summary_type = backlog.groupby('initiative_type').agg(
    Cantidad=('resource_id', 'count'),
    Ahorro_Mensual_Total=('savings_monthly_usd', 'sum'),
    Score_Promedio=('priority_score', 'mean')
).round(2)
print(summary_type)
```

**Salida esperada:**

```
✅ Backlog construido: ~60-80 iniciativas

Top 10 iniciativas por prioridad:
   resource_id  optimization_category initiative_type  savings_monthly_usd  effort_days  priority_score
1  NVT-VM-XXXX              Inactivo       Reducción               XXX.XX            1          XXX.XX
2  NVT-VM-XXXX              Inactivo       Reducción               XXX.XX            1          XXX.XX
...

📊 Resumen por tipo de iniciativa (Lección 5.1):
                 Cantidad  Ahorro_Mensual_Total  Score_Promedio
initiative_type                                                
Optimización           XX              XXXX.XX           XX.XX
Reducción              XX              XXXX.XX           XX.XX
```

**Verificación:** El backlog debe contener ambos tipos de iniciativa (Optimización y Reducción). Verificar con `backlog['initiative_type'].value_counts()`.

---

### Paso 4: Generar visualizaciones de priorización

**Objetivo:** Crear un bubble chart de impacto vs. esfuerzo y un bar chart por categoría que comuniquen visualmente las prioridades del backlog a audiencias ejecutivas.

**Instrucciones:**

1. Crear el bubble chart (gráfico de burbujas):

```python
fig, axes = plt.subplots(1, 2, figsize=(16, 7))

# --- Gráfico 1: Bubble Chart - Ahorro vs Esfuerzo ---
ax1 = axes[0]

colors_map = {
    'Inactivo': '#e74c3c',
    'Sobredimensionado': '#f39c12',
    'Huérfano': '#9b59b6',
    'Storage': '#3498db',
    'Rate_Optimization': '#2ecc71'
}

for category, group in backlog.groupby('optimization_category'):
    ax1.scatter(
        group['effort_days'],
        group['savings_monthly_usd'],
        s=group['priority_score'] * 3,  # Tamaño proporcional al score
        alpha=0.6,
        label=category,
        color=colors_map[category],
        edgecolors='black',
        linewidth=0.5
    )

ax1.set_xlabel('Esfuerzo (días)', fontsize=11)
ax1.set_ylabel('Ahorro Mensual Estimado (USD)', fontsize=11)
ax1.set_title('Matriz de Priorización: Ahorro vs Esfuerzo\nNovaTech SRL', fontsize=13, fontweight='bold')
ax1.legend(loc='upper left', fontsize=9)
ax1.grid(True, alpha=0.3)

# Añadir cuadrantes de referencia
median_effort = backlog['effort_days'].median()
median_savings = backlog['savings_monthly_usd'].median()
ax1.axhline(y=median_savings, color='gray', linestyle='--', alpha=0.5)
ax1.axvline(x=median_effort, color='gray', linestyle='--', alpha=0.5)
ax1.text(0.8, median_savings * 1.1, 'Quick Wins ↗', fontsize=9, color='green', fontweight='bold')
ax1.text(median_effort * 1.2, median_savings * 0.5, 'Proyectos Largos →', fontsize=9, color='orange')

# --- Gráfico 2: Bar Chart - Ahorro por Categoría ---
ax2 = axes[1]

category_summary = backlog.groupby('optimization_category').agg(
    ahorro_total=('savings_monthly_usd', 'sum'),
    cantidad=('resource_id', 'count')
).sort_values('ahorro_total', ascending=True)

bars = ax2.barh(
    category_summary.index,
    category_summary['ahorro_total'],
    color=[colors_map[c] for c in category_summary.index],
    edgecolor='black',
    linewidth=0.5
)

# Añadir etiquetas de valor
for bar, count in zip(bars, category_summary['cantidad']):
    width = bar.get_width()
    ax2.text(width + 20, bar.get_y() + bar.get_height()/2,
             f'${width:,.0f} ({count} items)',
             va='center', fontsize=9)

ax2.set_xlabel('Ahorro Mensual Estimado (USD)', fontsize=11)
ax2.set_title('Ahorro por Categoría de Optimización\nNovaTech SRL', fontsize=13, fontweight='bold')
ax2.grid(True, axis='x', alpha=0.3)

plt.tight_layout()
plt.savefig('~/finops-course/outputs/backlog_visualizations.png', dpi=150, bbox_inches='tight')
plt.show()

print("✅ Visualizaciones guardadas en: ~/finops-course/outputs/backlog_visualizations.png")
```

2. Crear un gráfico adicional que muestre la distinción optimización vs. reducción:

```python
fig, ax = plt.subplots(figsize=(10, 5))

type_summary = backlog.groupby(['initiative_type', 'optimization_category']).agg(
    ahorro=('savings_monthly_usd', 'sum')
).reset_index()

# Pivot para stacked bar
pivot = type_summary.pivot(index='initiative_type', columns='optimization_category', values='ahorro').fillna(0)

pivot.plot(kind='bar', stacked=True, ax=ax, 
           color=[colors_map[c] for c in pivot.columns],
           edgecolor='black', linewidth=0.5)

ax.set_title('Optimización vs Reducción de Costos\n(Concepto Lección 5.1 aplicado al Backlog)', 
             fontsize=13, fontweight='bold')
ax.set_xlabel('Tipo de Iniciativa', fontsize=11)
ax.set_ylabel('Ahorro Mensual Estimado (USD)', fontsize=11)
ax.legend(title='Categoría', bbox_to_anchor=(1.05, 1), loc='upper left')
ax.set_xticklabels(ax.get_xticklabels(), rotation=0)
ax.grid(True, axis='y', alpha=0.3)

# Añadir anotaciones conceptuales
ax.annotate('Mejora relación\nvalor/costo', xy=(0, pivot.loc['Optimización'].sum() * 0.5),
            fontsize=8, ha='center', style='italic', color='darkgreen')
ax.annotate('Elimina gasto\nsin valor', xy=(1, pivot.loc['Reducción'].sum() * 0.5),
            fontsize=8, ha='center', style='italic', color='darkred')

plt.tight_layout()
plt.savefig('~/finops-course/outputs/optimization_vs_reduction.png', dpi=150, bbox_inches='tight')
plt.show()

print("✅ Gráfico Optimización vs Reducción guardado")
```

**Salida esperada:** Dos figuras renderizadas en el notebook:
- Figura 1: Panel izquierdo con bubble chart mostrando burbujas coloreadas por categoría; panel derecho con barras horizontales de ahorro por categoría.
- Figura 2: Bar chart agrupado mostrando la composición del ahorro por tipo de iniciativa.

**Verificación:** Los archivos PNG deben existir en `~/finops-course/outputs/`. Verificar con:

```python
import os
assert os.path.exists(os.path.expanduser('~/finops-course/outputs/backlog_visualizations.png'))
assert os.path.exists(os.path.expanduser('~/finops-course/outputs/optimization_vs_reduction.png'))
print("✅ Ambos archivos de visualización existen")
```

---

### Paso 5: Exportar el backlog a Excel con formato profesional

**Objetivo:** Generar el archivo `finops_backlog_v1.xlsx` con formato profesional usando openpyxl, incluyendo una hoja de resumen ejecutivo y una hoja de detalle.

**Instrucciones:**

1. Exportar el backlog con formato:

```python
from openpyxl import load_workbook
from openpyxl.styles import Font, PatternFill, Alignment, Border, Side
from openpyxl.utils.dataframe import dataframe_to_rows

output_path = os.path.expanduser('~/finops-course/outputs/finops_backlog_v1.xlsx')

# --- Crear Excel con múltiples hojas ---
with pd.ExcelWriter(output_path, engine='openpyxl') as writer:
    
    # Hoja 1: Backlog Detallado
    backlog.to_excel(writer, sheet_name='Backlog_Detalle', index=True)
    
    # Hoja 2: Resumen por Categoría
    resumen_cat = backlog.groupby('optimization_category').agg(
        Cantidad=('resource_id', 'count'),
        Ahorro_Mensual_USD=('savings_monthly_usd', 'sum'),
        Ahorro_Anual_USD=('savings_monthly_usd', lambda x: x.sum() * 12),
        Esfuerzo_Promedio_Dias=('effort_days', 'mean'),
        Score_Promedio=('priority_score', 'mean')
    ).round(2).sort_values('Ahorro_Mensual_USD', ascending=False)
    resumen_cat.to_excel(writer, sheet_name='Resumen_Categoría')
    
    # Hoja 3: Resumen por Tipo de Iniciativa
    resumen_tipo = backlog.groupby('initiative_type').agg(
        Cantidad=('resource_id', 'count'),
        Ahorro_Mensual_USD=('savings_monthly_usd', 'sum'),
        Ahorro_Anual_USD=('savings_monthly_usd', lambda x: x.sum() * 12),
    ).round(2)
    resumen_tipo.to_excel(writer, sheet_name='Optimización_vs_Reducción')
    
    # Hoja 4: Top 20 Quick Wins
    quick_wins = backlog.head(20)[['resource_id', 'resource_name', 'optimization_category',
                                     'initiative_type', 'action', 'team', 'savings_monthly_usd',
                                     'effort_days', 'priority_score']]
    quick_wins.to_excel(writer, sheet_name='Top20_QuickWins', index=False)

print(f"✅ Excel base exportado: {output_path}")
```

2. Aplicar formato profesional con openpyxl:

```python
# Cargar y formatear
wb = load_workbook(output_path)

# Estilos
header_font = Font(bold=True, color='FFFFFF', size=11)
header_fill = PatternFill(start_color='2C3E50', end_color='2C3E50', fill_type='solid')
money_format = '#,##0.00'
thin_border = Border(
    left=Side(style='thin'),
    right=Side(style='thin'),
    top=Side(style='thin'),
    bottom=Side(style='thin')
)

# Formatear cada hoja
for sheet_name in wb.sheetnames:
    ws = wb[sheet_name]
    
    # Formatear encabezados (fila 1)
    for cell in ws[1]:
        cell.font = header_font
        cell.fill = header_fill
        cell.alignment = Alignment(horizontal='center', vertical='center')
        cell.border = thin_border
    
    # Ajustar ancho de columnas
    for column_cells in ws.columns:
        max_length = 0
        column_letter = column_cells[0].column_letter
        for cell in column_cells:
            try:
                if len(str(cell.value)) > max_length:
                    max_length = len(str(cell.value))
            except:
                pass
        adjusted_width = min(max_length + 2, 30)
        ws.column_dimensions[column_letter].width = adjusted_width

# Colorear filas por categoría en hoja de detalle
ws_detail = wb['Backlog_Detalle']
category_fills = {
    'Inactivo': PatternFill(start_color='FADBD8', end_color='FADBD8', fill_type='solid'),
    'Sobredimensionado': PatternFill(start_color='FDEBD0', end_color='FDEBD0', fill_type='solid'),
    'Huérfano': PatternFill(start_color='E8DAEF', end_color='E8DAEF', fill_type='solid'),
    'Storage': PatternFill(start_color='D6EAF8', end_color='D6EAF8', fill_type='solid'),
    'Rate_Optimization': PatternFill(start_color='D5F5E3', end_color='D5F5E3', fill_type='solid'),
}

# Encontrar columna de categoría (columna E = índice 5)
cat_col_idx = 5  # optimization_category está en columna E (1-indexed)
for row in ws_detail.iter_rows(min_row=2, max_row=ws_detail.max_row):
    cat_value = row[cat_col_idx - 1].value
    if cat_value in category_fills:
        for cell in row:
            cell.fill = category_fills[cat_value]

wb.save(output_path)
print(f"✅ Formato profesional aplicado")
print(f"   Archivo final: {output_path}")
print(f"   Hojas: {wb.sheetnames}")
print(f"   Tamaño: {os.path.getsize(output_path) / 1024:.1f} KB")
```

**Salida esperada:**

```
✅ Excel base exportado: ~/finops-course/outputs/finops_backlog_v1.xlsx
✅ Formato profesional aplicado
   Archivo final: ~/finops-course/outputs/finops_backlog_v1.xlsx
   Hojas: ['Backlog_Detalle', 'Resumen_Categoría', 'Optimización_vs_Reducción', 'Top20_QuickWins']
   Tamaño: ~45-80 KB
```

**Verificación:** Abrir el archivo Excel en LibreOffice Calc o la herramienta disponible y confirmar que:
- Los encabezados tienen fondo oscuro con texto blanco.
- Las filas están coloreadas por categoría.
- Las 4 hojas contienen datos.

---

### Paso 6: Generar el resumen ejecutivo final en el notebook

**Objetivo:** Producir un resumen consolidado del backlog con métricas clave y recomendaciones, cerrando el notebook con un output profesional.

**Instrucciones:**

1. Crear la celda de resumen ejecutivo:

```python
print("=" * 70)
print("         RESUMEN EJECUTIVO - BACKLOG DE OPTIMIZACIÓN")
print("         NovaTech SRL | Enero 2024")
print("=" * 70)

total_monthly = backlog['savings_monthly_usd'].sum()
total_annual = total_monthly * 12
total_current_cost = df_resources['monthly_cost_usd'].sum()
pct_savings = (total_monthly / total_current_cost) * 100

print(f"\n📊 MÉTRICAS GLOBALES:")
print(f"   Gasto mensual actual:          ${total_current_cost:>12,.2f}")
print(f"   Oportunidades identificadas:   {len(backlog):>12d} recursos")
print(f"   Ahorro mensual estimado:       ${total_monthly:>12,.2f}")
print(f"   Ahorro anual estimado:         ${total_annual:>12,.2f}")
print(f"   Porcentaje de ahorro potencial:{pct_savings:>11.1f}%")

print(f"\n📋 DISTRIBUCIÓN POR TIPO DE INICIATIVA:")
for itype, group in backlog.groupby('initiative_type'):
    emoji = "🔧" if itype == "Optimización" else "✂️"
    print(f"   {emoji} {itype}: {len(group)} iniciativas | "
          f"${group['savings_monthly_usd'].sum():,.2f}/mes")

print(f"\n🏆 TOP 5 QUICK WINS (mayor score de priorización):")
print("-" * 70)
for idx, row in backlog.head(5).iterrows():
    print(f"   #{idx}: [{row['optimization_category']}] {row['resource_name']}")
    print(f"       Ahorro: ${row['savings_monthly_usd']:,.2f}/mes | "
          f"Esfuerzo: {row['effort_days']} día(s) | Score: {row['priority_score']:.1f}")

print(f"\n{'=' * 70}")
print(f"📁 ARTEFACTOS GENERADOS:")
print(f"   • ~/finops-course/outputs/finops_backlog_v1.xlsx")
print(f"   • ~/finops-course/outputs/backlog_visualizations.png")
print(f"   • ~/finops-course/outputs/optimization_vs_reduction.png")
print(f"   • ~/finops-course/data/raw/novatech_cloud_resources.csv")
print(f"{'=' * 70}")
```

**Salida esperada:**

```
======================================================================
         RESUMEN EJECUTIVO - BACKLOG DE OPTIMIZACIÓN
         NovaTech SRL | Enero 2024
======================================================================

📊 MÉTRICAS GLOBALES:
   Gasto mensual actual:          $  XX,XXX.XX
   Oportunidades identificadas:           XX recursos
   Ahorro mensual estimado:       $   X,XXX.XX
   Ahorro anual estimado:         $  XX,XXX.XX
   Porcentaje de ahorro potencial:      XX.X%

📋 DISTRIBUCIÓN POR TIPO DE INICIATIVA:
   🔧 Optimización: XX iniciativas | $X,XXX.XX/mes
   ✂️ Reducción: XX iniciativas | $X,XXX.XX/mes

🏆 TOP 5 QUICK WINS (mayor score de priorización):
----------------------------------------------------------------------
   #1: [Inactivo] paycore-dev-vm-XX
       Ahorro: $XXX.XX/mes | Esfuerzo: 1 día(s) | Score: XXX.X
   ...
```

**Verificación:** El porcentaje de ahorro potencial debe estar entre 15% y 50% (rango realista para una organización sin práctica FinOps madura).

---

## Validación y Pruebas

Ejecutar las siguientes verificaciones al final del notebook:

```python
# === VALIDACIÓN FINAL DEL LABORATORIO ===
print("🔍 Ejecutando validaciones...\n")

checks_passed = 0
total_checks = 6

# Check 1: Dataset generado correctamente
assert df_resources.shape[0] == 155, "Dataset debe tener 155 recursos"
checks_passed += 1
print(f"✅ Check 1/6: Dataset con {df_resources.shape[0]} recursos")

# Check 2: Backlog tiene columnas requeridas
required_cols = ['resource_id', 'optimization_category', 'initiative_type', 
                 'savings_monthly_usd', 'effort_days', 'priority_score', 'status']
assert all(col in backlog.columns for col in required_cols)
checks_passed += 1
print(f"✅ Check 2/6: Backlog tiene todas las columnas requeridas")

# Check 3: Ambos tipos de iniciativa presentes
assert set(backlog['initiative_type'].unique()) == {'Optimización', 'Reducción'}
checks_passed += 1
print(f"✅ Check 3/6: Tipos de iniciativa correctos (Optimización + Reducción)")

# Check 4: Score de priorización es positivo y ordenado
assert (backlog['priority_score'] > 0).all()
assert backlog['priority_score'].is_monotonic_decreasing
checks_passed += 1
print(f"✅ Check 4/6: Scores positivos y ordenados descendentemente")

# Check 5: Archivo Excel existe y tiene 4 hojas
assert os.path.exists(output_path)
wb_check = load_workbook(output_path)
assert len(wb_check.sheetnames) == 4
checks_passed += 1
print(f"✅ Check 5/6: Excel generado con 4 hojas")

# Check 6: Visualizaciones PNG existen
viz_path = os.path.expanduser('~/finops-course/outputs/backlog_visualizations.png')
assert os.path.exists(viz_path)
checks_passed += 1
print(f"✅ Check 6/6: Visualizaciones PNG generadas")

print(f"\n{'='*50}")
print(f"🎉 LABORATORIO COMPLETADO: {checks_passed}/{total_checks} validaciones exitosas")
print(f"{'='*50}")
```

---

## Solución de Problemas

### Problema 1: Error `ModuleNotFoundError: No module named 'faker'`

**Síntomas:** Al ejecutar `from faker import Faker` aparece un error de importación indicando que el módulo no está instalado.

**Causa:** El paquete Faker no está incluido en el archivo `requirements.txt` estándar del curso y debe instalarse por separado para este laboratorio.

**Solución:**

```bash
# Instalar Faker desde una celda del notebook
!pip install Faker==24.2.0

# O desde terminal
pip install Faker==24.2.0

# Verificar instalación
python -c "from faker import Faker; print(f'Faker {Faker().seed_locale} instalado')"
```

Reiniciar el kernel de Jupyter después de instalar (`Kernel > Restart Kernel`).

---

### Problema 2: El bubble chart muestra todas las burbujas del mismo tamaño

**Síntomas:** El gráfico de burbujas se renderiza pero todas las burbujas tienen tamaño idéntico, sin diferenciación visual por score de priorización.

**Causa:** El parámetro `s` (size) en `ax.scatter()` recibe valores muy pequeños o iguales. Esto ocurre cuando el `priority_score` tiene un rango muy estrecho o cuando se olvidó multiplicar por un factor de escala.

**Solución:**

```python
# Verificar el rango de priority_score
print(f"Score mín: {backlog['priority_score'].min()}")
print(f"Score máx: {backlog['priority_score'].max()}")
print(f"Score std: {backlog['priority_score'].std()}")

# Si el rango es muy pequeño, normalizar y escalar
from sklearn.preprocessing import MinMaxScaler  # o manualmente:
scores_normalized = (
    (backlog['priority_score'] - backlog['priority_score'].min()) / 
    (backlog['priority_score'].max() - backlog['priority_score'].min())
) * 300 + 30  # Rango de 30 a 330 píxeles

# Usar scores_normalized en lugar de priority_score * 3 en el parámetro s=
```

Alternativamente, aumentar el multiplicador de `3` a `10` o `20` dependiendo del rango de scores.

---

## Limpieza

Este laboratorio genera artefactos que serán utilizados en labs posteriores. **No eliminar** los archivos de salida.

```python
# Verificar artefactos generados (NO eliminar)
print("📁 Artefactos del laboratorio (conservar para labs futuros):")
print(f"   ✓ ~/finops-course/outputs/finops_backlog_v1.xlsx")
print(f"   ✓ ~/finops-course/outputs/backlog_visualizations.png")
print(f"   ✓ ~/finops-course/outputs/optimization_vs_reduction.png")
print(f"   ✓ ~/finops-course/data/raw/novatech_cloud_resources.csv")
```

Si necesitas reiniciar el laboratorio desde cero:

```bash
# SOLO si necesitas reiniciar (elimina todos los outputs del lab)
rm -f ~/finops-course/outputs/finops_backlog_v1.xlsx
rm -f ~/finops-course/outputs/backlog_visualizations.png
rm -f ~/finops-course/outputs/optimization_vs_reduction.png
rm -f ~/finops-course/data/raw/novatech_cloud_resources.csv
```

---

## Resumen

En este laboratorio has completado las siguientes tareas:

| Actividad | Resultado |
|-----------|-----------|
| Generación de dataset sintético | 155 recursos cloud con métricas de utilización |
| Detección de oportunidades | 5 categorías de optimización identificadas |
| Construcción del backlog | DataFrame estructurado con scoring de priorización |
| Diferenciación conceptual | Cada iniciativa clasificada como Optimización o Reducción |
| Visualizaciones ejecutivas | Bubble chart + bar charts exportados como PNG |
| Exportación profesional | Excel con 4 hojas formateadas (`finops_backlog_v1.xlsx`) |

### Conceptos clave aplicados

- **Optimización vs. Reducción (Lección 5.1):** Las iniciativas de Rate Optimization y Rightsizing son *optimización* (mejoran la relación valor/costo), mientras que eliminar recursos inactivos, huérfanos y storage sin uso es *reducción* (elimina gasto sin valor).
- **Scoring de priorización:** La fórmula `(Ahorro / Esfuerzo) × Factor_Riesgo` permite ordenar objetivamente las iniciativas por retorno de inversión ajustado.
- **Quick Wins:** Las iniciativas con alto ahorro y bajo esfuerzo (esquina superior izquierda del bubble chart) deben ejecutarse primero.

### Recursos adicionales

- [FinOps Foundation — Optimizing Cloud Usage & Cost](https://www.finops.org/framework/capabilities/optimization/)
- [AWS Well-Architected — Cost Optimization Pillar](https://docs.aws.amazon.com/wellarchitected/latest/cost-optimization-pillar/welcome.html)
- [Cloud Waste Report 2024 — Flexera](https://www.flexera.com/blog/cloud/cloud-computing-trends/)

# Demostración 3. Asignar costos por producto, equipo y ambiente

## Metadata

| Campo | Valor |
|-------|-------|
| **Duración** | 30 minutos |
| **Complejidad** | Media |
| **Nivel Bloom** | Aplicar |
| **Tecnologías** | Python 3.12.1, pandas 2.2.1, numpy 1.26.4, openpyxl 3.1.2, Jupyter Notebook 7.1.2 |

## Descripción General

En este laboratorio implementarás un modelo completo de asignación de costos cloud para NovaTech SRL. Partiendo del dataset de facturación de 90 días (`novatech_billing_90days.csv`), diseñarás una taxonomía de etiquetas FinOps, clasificarás registros en costos directos, compartidos y no asignados, distribuirás costos compartidos mediante prorrateo proporcional, y generarás reportes de showback y chargeback en formato Excel. Los archivos de salida serán insumos obligatorios para el laboratorio 04-00-01.

## Objetivos de Aprendizaje

- [ ] Diseñar una taxonomía FinOps mínima de 5 etiquetas para asignar costos por producto, equipo y ambiente en NovaTech SRL
- [ ] Clasificar registros de facturación en costos directos, compartidos, no asignados y no etiquetados aplicando reglas de asignación programáticas
- [ ] Implementar un modelo de showback y chargeback que distribuya costos compartidos proporcionalmente al consumo de compute
- [ ] Calcular métricas de calidad de asignación: porcentaje de cobertura de etiquetas y costo no asignado total

## Prerrequisitos

### Conocimiento Previo

| Requisito | Descripción |
|-----------|-------------|
| Labs 02-00-01 y 02-00-02 | Completados con archivos de salida generados |
| Estructura NovaTech SRL | 3 productos, 4 equipos, 3 ambientes, 3 proveedores |
| Módulo 3 teoría | Secciones 3.1 a 3.5 leídas (asignación de costos, etiquetas, showback/chargeback) |
| Python/pandas | Manejo de DataFrames, groupby, merge, pivot_table |

### Archivos Requeridos

| Archivo | Ubicación | Origen |
|---------|-----------|--------|
| `novatech_billing_90days.csv` | `~/finops-course/data/raw/` | Provisto por instructor |
| `novatech_billing_summary_jan2024.xlsx` | `~/finops-course/data/processed/` | Lab 02-00-01 |
| `lab_03_01_cost_allocation.ipynb` | `~/finops-course/notebooks/` | Provisto por instructor |

## Entorno del Laboratorio

### Software Requerido

| Herramienta | Versión | Verificación |
|-------------|---------|--------------|
| Python | 3.12.1 | `python --version` |
| pandas | 2.2.1 | `python -c "import pandas; print(pandas.__version__)"` |
| numpy | 1.26.4 | `python -c "import numpy; print(numpy.__version__)"` |
| openpyxl | 3.1.2 | `python -c "import openpyxl; print(openpyxl.__version__)"` |
| Jupyter Notebook | 7.1.2 | `jupyter --version` |

### Configuración Inicial

```bash
# Verificar estructura de directorios
cd ~/finops-course
ls data/raw/novatech_billing_90days.csv
ls data/processed/novatech_billing_summary_jan2024.xlsx

# Crear directorio de salida si no existe
mkdir -p ~/finops-course/outputs/lab03

# Iniciar Jupyter Notebook
jupyter notebook --port=8888 --notebook-dir=~/finops-course/notebooks/
```

---

## Paso 1: Explorar el Dataset de Facturación y Comprender su Estructura

### Objetivo

Cargar el dataset `novatech_billing_90days.csv`, comprender sus columnas, identificar los campos de etiquetas existentes y cuantificar el estado actual de etiquetado.

### Instrucciones

1. Abre el notebook `lab_03_01_cost_allocation.ipynb` en Jupyter.

2. En la primera celda, importa las dependencias y carga el dataset:

```python
import pandas as pd
import numpy as np
from datetime import datetime

# Cargar dataset de facturación 90 días (Q4 2023 + enero 2024)
df = pd.read_csv('../data/raw/novatech_billing_90days.csv')

print(f"Total de registros: {len(df):,}")
print(f"Columnas disponibles: {list(df.columns)}")
print(f"Rango de fechas: {df['UsageDate'].min()} a {df['UsageDate'].max()}")
print(f"Gasto total: ${df['Cost'].sum():,.2f} USD")
```

3. Examina las columnas relacionadas con etiquetas y metadatos organizacionales:

```python
# Identificar columnas de tags
tag_columns = [col for col in df.columns if 'tag' in col.lower() or 'Tag' in col]
print(f"\nColumnas de etiquetas encontradas: {tag_columns}")

# Inspeccionar valores únicos en campos organizacionales
for col in ['Provider', 'ServiceCategory', 'ResourceId']:
    print(f"\n{col}: {df[col].nunique()} valores únicos")

# Verificar campos de etiquetas existentes
for col in tag_columns:
    non_null = df[col].notna().sum()
    pct = (non_null / len(df)) * 100
    print(f"  {col}: {non_null:,} registros con valor ({pct:.1f}%)")
```

4. Calcula el porcentaje de registros sin etiquetas (estado actual antes de la intervención):

```python
# Calcular cobertura de etiquetas actual
# Un registro se considera "etiquetado" si tiene al menos tag_Environment Y tag_Team
has_env = df['tag_Environment'].notna()
has_team = df['tag_Team'].notna()
has_product = df['tag_Product'].notna()

fully_tagged = (has_env & has_team & has_product).sum()
partially_tagged = ((has_env | has_team | has_product) & ~(has_env & has_team & has_product)).sum()
untagged = (~(has_env | has_team | has_product)).sum()

print(f"\n=== Estado de Etiquetado Actual ===")
print(f"Completamente etiquetados: {fully_tagged:,} ({fully_tagged/len(df)*100:.1f}%)")
print(f"Parcialmente etiquetados:  {partially_tagged:,} ({partially_tagged/len(df)*100:.1f}%)")
print(f"Sin etiquetas:             {untagged:,} ({untagged/len(df)*100:.1f}%)")
print(f"\nCosto sin asignar: ${df[~(has_env & has_team & has_product)]['Cost'].sum():,.2f} USD")
```

### Salida Esperada

```
Total de registros: ~12,000-15,000
Columnas disponibles: ['UsageDate', 'Provider', 'AccountId', 'ServiceCategory', 
  'ResourceId', 'Cost', 'UsageQuantity', 'tag_Environment', 'tag_Team', 
  'tag_Product', 'tag_CostCenter', 'tag_Owner']

=== Estado de Etiquetado Actual ===
Completamente etiquetados: ~65% de registros
Parcialmente etiquetados:  ~15% de registros
Sin etiquetas:             ~20% de registros
```

### Verificación

- [ ] El dataset tiene aproximadamente un 35% de registros con etiquetas incompletas o ausentes (combinando parcialmente etiquetados + sin etiquetas)
- [ ] Se identifican al menos 5 columnas de tags: `tag_Environment`, `tag_Team`, `tag_Product`, `tag_CostCenter`, `tag_Owner`
- [ ] El gasto total es consistente con el rango de Q4 2023 + enero 2024

---

## Paso 2: Diseñar la Taxonomía de Etiquetas FinOps

### Objetivo

Definir formalmente la estrategia de etiquetado de NovaTech SRL con 5 etiquetas obligatorias, sus valores permitidos y reglas de validación, generando el archivo de referencia `novatech_tagging_taxonomy_v1.xlsx`.

### Instrucciones

1. Define la taxonomía de etiquetas como estructura de datos:

```python
# Definir taxonomía de etiquetas FinOps para NovaTech SRL
taxonomy = {
    'tag_key': ['Environment', 'Team', 'Product', 'CostCenter', 'Owner'],
    'required': [True, True, True, True, False],
    'allowed_values': [
        'prod, staging, dev',
        'Backend, Frontend, Data, Platform',
        'PayCore, AnalyticsHub, DevPortal',
        'CC-001 (PayCore), CC-002 (AnalyticsHub), CC-003 (DevPortal), CC-SHARED',
        'email del responsable técnico'
    ],
    'purpose': [
        'Diferenciar costos por ciclo de vida del recurso',
        'Asignar responsabilidad operativa al equipo dueño',
        'Vincular gasto con línea de producto/ingreso',
        'Mapear a estructura contable de Finanzas',
        'Identificar persona contacto para decisiones de optimización'
    ],
    'allocation_role': [
        'Dimensión de filtrado',
        'Destino primario de asignación',
        'Destino secundario de asignación',
        'Integración con ERP/contabilidad',
        'Escalamiento operativo'
    ]
}

df_taxonomy = pd.DataFrame(taxonomy)
print("=== Taxonomía de Etiquetas NovaTech SRL v1.0 ===\n")
print(df_taxonomy.to_string(index=False))
```

2. Define la tabla de mapeo entre equipos, productos y centros de costo:

```python
# Mapeo organizacional NovaTech SRL
org_mapping = pd.DataFrame({
    'Team': ['Backend', 'Frontend', 'Data', 'Platform'],
    'Primary_Product': ['PayCore', 'PayCore', 'AnalyticsHub', 'Shared'],
    'CostCenter': ['CC-001', 'CC-001', 'CC-002', 'CC-SHARED'],
    'Manager': ['carlos.ruiz@novatech.io', 'ana.lopez@novatech.io', 
                'diego.martinez@novatech.io', 'laura.garcia@novatech.io'],
    'Budget_Pct': [0.35, 0.20, 0.25, 0.20]
})

print("\n=== Mapeo Organizacional ===\n")
print(org_mapping.to_string(index=False))
```

3. Exporta la taxonomía a Excel:

```python
# Exportar taxonomía a Excel con múltiples hojas
output_taxonomy = '../outputs/lab03/novatech_tagging_taxonomy_v1.xlsx'

with pd.ExcelWriter(output_taxonomy, engine='openpyxl') as writer:
    df_taxonomy.to_excel(writer, sheet_name='Taxonomy', index=False)
    org_mapping.to_excel(writer, sheet_name='OrgMapping', index=False)
    
    # Hoja de reglas de validación
    validation_rules = pd.DataFrame({
        'rule_id': ['R001', 'R002', 'R003', 'R004', 'R005'],
        'description': [
            'Todo recurso en prod DEBE tener tag_Team y tag_Product',
            'tag_Environment solo acepta: prod, staging, dev',
            'tag_Team solo acepta: Backend, Frontend, Data, Platform',
            'tag_Product solo acepta: PayCore, AnalyticsHub, DevPortal',
            'Recursos sin tag_Team después de 7 días se escalan al Platform team'
        ],
        'severity': ['Critical', 'High', 'High', 'High', 'Medium']
    })
    validation_rules.to_excel(writer, sheet_name='ValidationRules', index=False)

print(f"\n✓ Taxonomía exportada: {output_taxonomy}")
```

### Salida Esperada

```
=== Taxonomía de Etiquetas NovaTech SRL v1.0 ===

    tag_key  required                              allowed_values  ...
Environment      True                        prod, staging, dev  ...
       Team      True        Backend, Frontend, Data, Platform  ...
    Product      True       PayCore, AnalyticsHub, DevPortal  ...
 CostCenter      True  CC-001, CC-002, CC-003, CC-SHARED  ...
      Owner     False     email del responsable técnico  ...

✓ Taxonomía exportada: ../outputs/lab03/novatech_tagging_taxonomy_v1.xlsx
```

### Verificación

- [ ] El archivo `novatech_tagging_taxonomy_v1.xlsx` existe con 3 hojas: Taxonomy, OrgMapping, ValidationRules
- [ ] La taxonomía incluye exactamente 5 etiquetas con 4 obligatorias y 1 opcional
- [ ] Los valores permitidos coinciden con la estructura de NovaTech SRL (3 productos, 4 equipos, 3 ambientes)

---

## Paso 3: Clasificar Registros por Tipo de Asignación

### Objetivo

Categorizar cada registro del dataset en una de cuatro clases: **directo** (etiquetas completas), **compartido** (servicios de infraestructura común), **no etiquetado** (recurso identificable pero sin tags) y **no asignable** (sin información suficiente).

### Instrucciones

1. Define las reglas de clasificación:

```python
# Definir servicios considerados "compartidos" (networking, seguridad, soporte)
SHARED_SERVICES = ['Networking', 'Security', 'Support', 'DNS', 'WAF', 'CloudTrail']

# Crear columna de clasificación de asignación
def classify_allocation(row):
    """Clasifica un registro según su asignabilidad."""
    service = row['ServiceCategory']
    has_team = pd.notna(row['tag_Team'])
    has_product = pd.notna(row['tag_Product'])
    has_env = pd.notna(row['tag_Environment'])
    
    # Servicios compartidos siempre van a la categoría "shared"
    if service in SHARED_SERVICES:
        return 'shared'
    
    # Etiquetas completas = asignación directa
    if has_team and has_product and has_env:
        return 'direct'
    
    # Tiene alguna etiqueta pero no todas = no etiquetado (parcial)
    if has_team or has_product or has_env:
        return 'untagged'
    
    # Sin ninguna información = no asignable
    return 'unallocated'

df['allocation_type'] = df.apply(classify_allocation, axis=1)

# Resumen de clasificación
allocation_summary = df.groupby('allocation_type').agg(
    records=('Cost', 'count'),
    total_cost=('Cost', 'sum')
).reset_index()

allocation_summary['pct_cost'] = (allocation_summary['total_cost'] / 
                                   allocation_summary['total_cost'].sum() * 100)

print("=== Clasificación de Asignación ===\n")
print(allocation_summary.to_string(index=False))
print(f"\nGasto total: ${allocation_summary['total_cost'].sum():,.2f} USD")
```

2. Para los registros parcialmente etiquetados (`untagged`), intenta inferir el equipo a partir del `AccountId` o `ResourceId`:

```python
# Reglas de inferencia basadas en AccountId
account_team_map = {
    'acc-paycore-prod': {'Team': 'Backend', 'Product': 'PayCore', 'Environment': 'prod'},
    'acc-paycore-dev': {'Team': 'Backend', 'Product': 'PayCore', 'Environment': 'dev'},
    'acc-analytics-prod': {'Team': 'Data', 'Product': 'AnalyticsHub', 'Environment': 'prod'},
    'acc-analytics-staging': {'Team': 'Data', 'Product': 'AnalyticsHub', 'Environment': 'staging'},
    'acc-devportal-prod': {'Team': 'Frontend', 'Product': 'DevPortal', 'Environment': 'prod'},
    'acc-devportal-dev': {'Team': 'Frontend', 'Product': 'DevPortal', 'Environment': 'dev'},
    'acc-platform-shared': {'Team': 'Platform', 'Product': 'Shared', 'Environment': 'prod'},
}

# Aplicar inferencia solo a registros 'untagged'
inferred_count = 0
for idx, row in df[df['allocation_type'] == 'untagged'].iterrows():
    account = row['AccountId']
    if account in account_team_map:
        mapping = account_team_map[account]
        if pd.isna(row['tag_Team']):
            df.at[idx, 'tag_Team'] = mapping['Team']
        if pd.isna(row['tag_Product']):
            df.at[idx, 'tag_Product'] = mapping['Product']
        if pd.isna(row['tag_Environment']):
            df.at[idx, 'tag_Environment'] = mapping['Environment']
        df.at[idx, 'allocation_type'] = 'direct'  # Reclasificar
        inferred_count += 1

print(f"\n✓ Registros reclasificados por inferencia de cuenta: {inferred_count:,}")

# Recalcular resumen post-inferencia
allocation_post = df.groupby('allocation_type').agg(
    records=('Cost', 'count'),
    total_cost=('Cost', 'sum')
).reset_index()
allocation_post['pct_cost'] = (allocation_post['total_cost'] / 
                                allocation_post['total_cost'].sum() * 100)

print("\n=== Clasificación Post-Inferencia ===\n")
print(allocation_post.to_string(index=False))
```

### Salida Esperada

```
=== Clasificación de Asignación ===

allocation_type  records  total_cost  pct_cost
       direct      ~8500   ~185000      ~62%
       shared      ~2000    ~55000      ~18%
     untagged      ~2500    ~42000      ~14%
  unallocated       ~800    ~18000       ~6%

✓ Registros reclasificados por inferencia de cuenta: ~1,200

=== Clasificación Post-Inferencia ===

allocation_type  records  total_cost  pct_cost
       direct      ~9700   ~210000      ~70%
       shared      ~2000    ~55000      ~18%
     untagged      ~1300    ~25000       ~8%
  unallocated       ~800    ~10000       ~4%
```

### Verificación

- [ ] La columna `allocation_type` contiene exactamente 4 valores: `direct`, `shared`, `untagged`, `unallocated`
- [ ] Los costos compartidos representan entre 15-20% del gasto total
- [ ] Después de la inferencia, el porcentaje de costos directos sube significativamente

---

## Paso 4: Distribuir Costos Compartidos con Prorrateo Proporcional

### Objetivo

Implementar la distribución de costos compartidos (networking, seguridad) entre los equipos de NovaTech SRL usando como base de prorrateo el consumo de compute de cada equipo.

### Instrucciones

1. Calcula la base de prorrateo (proporción de compute por equipo):

```python
# Calcular consumo de compute por equipo (base para prorrateo)
compute_services = ['EC2', 'Compute Engine', 'Virtual Machines', 'Lambda', 
                    'Cloud Functions', 'Azure Functions', 'ECS', 'EKS', 'GKE']

df_compute = df[(df['allocation_type'] == 'direct') & 
                (df['ServiceCategory'].isin(compute_services))]

compute_by_team = df_compute.groupby('tag_Team')['Cost'].sum().reset_index()
compute_by_team.columns = ['Team', 'compute_cost']
compute_by_team['proration_pct'] = (compute_by_team['compute_cost'] / 
                                     compute_by_team['compute_cost'].sum())

print("=== Base de Prorrateo (Compute por Equipo) ===\n")
print(compute_by_team.to_string(index=False))
print(f"\nTotal compute: ${compute_by_team['compute_cost'].sum():,.2f} USD")
```

2. Distribuye los costos compartidos usando la proporción calculada:

```python
# Total de costos compartidos a distribuir
shared_total = df[df['allocation_type'] == 'shared']['Cost'].sum()
print(f"\nCostos compartidos a distribuir: ${shared_total:,.2f} USD")

# Crear tabla de distribución
shared_allocation = compute_by_team[['Team', 'proration_pct']].copy()
shared_allocation['shared_cost_allocated'] = shared_allocation['proration_pct'] * shared_total
shared_allocation['shared_cost_allocated'] = shared_allocation['shared_cost_allocated'].round(2)

print("\n=== Distribución de Costos Compartidos ===\n")
print(shared_allocation.to_string(index=False))
print(f"\nVerificación - suma distribuida: ${shared_allocation['shared_cost_allocated'].sum():,.2f} USD")
```

3. Construye la tabla consolidada de asignación total por equipo:

```python
# Costos directos por equipo
direct_by_team = df[df['allocation_type'] == 'direct'].groupby('tag_Team')['Cost'].sum().reset_index()
direct_by_team.columns = ['Team', 'direct_cost']

# Merge con costos compartidos
cost_allocation = direct_by_team.merge(
    shared_allocation[['Team', 'shared_cost_allocated']], 
    on='Team', 
    how='left'
).fillna(0)

cost_allocation['total_allocated'] = (cost_allocation['direct_cost'] + 
                                       cost_allocation['shared_cost_allocated'])

# Agregar costos no asignados como línea separada
unallocated_total = df[df['allocation_type'].isin(['untagged', 'unallocated'])]['Cost'].sum()

# Tabla final
print("\n" + "="*70)
print("  RESUMEN DE ASIGNACIÓN DE COSTOS - NovaTech SRL Q4 2023 + Ene 2024")
print("="*70)
print(f"\n{'Equipo':<12} | {'Costo Directo':>14} | {'Compartido':>12} | {'Total':>14} | {'%':>6}")
print("-"*70)

grand_total = cost_allocation['total_allocated'].sum() + unallocated_total

for _, row in cost_allocation.iterrows():
    pct = row['total_allocated'] / grand_total * 100
    print(f"{row['Team']:<12} | ${row['direct_cost']:>11,.2f} | "
          f"${row['shared_cost_allocated']:>9,.2f} | "
          f"${row['total_allocated']:>11,.2f} | {pct:>5.1f}%")

print("-"*70)
print(f"{'No asignado':<12} | ${unallocated_total:>11,.2f} | {'$0.00':>12} | "
      f"${unallocated_total:>11,.2f} | {unallocated_total/grand_total*100:>5.1f}%")
print("-"*70)
print(f"{'TOTAL':<12} | {' ':>14} | {' ':>12} | ${grand_total:>11,.2f} | 100.0%")
```

### Salida Esperada

```
======================================================================
  RESUMEN DE ASIGNACIÓN DE COSTOS - NovaTech SRL Q4 2023 + Ene 2024
======================================================================

Equipo       |  Costo Directo |   Compartido |          Total |      %
----------------------------------------------------------------------
Backend      |    $75,000.00  |  $19,800.00  |    $94,800.00  | 31.6%
Frontend     |    $42,000.00  |  $11,550.00  |    $53,550.00  | 17.8%
Data         |    $58,000.00  |  $15,400.00  |    $73,400.00  | 24.5%
Platform     |    $35,000.00  |   $8,250.00  |    $43,250.00  | 14.4%
----------------------------------------------------------------------
No asignado  |    $35,000.00  |        $0.00 |    $35,000.00  | 11.7%
----------------------------------------------------------------------
TOTAL        |                |              |   $300,000.00  | 100.0%
```

### Verificación

- [ ] La suma de costos compartidos distribuidos es igual al total de costos compartidos original (sin pérdida ni ganancia)
- [ ] Todos los equipos reciben una porción de costos compartidos proporcional a su consumo de compute
- [ ] El costo no asignado está claramente separado y cuantificado

---

## Paso 5: Generar Reportes de Showback y Chargeback

### Objetivo

Producir dos vistas del mismo dato: un reporte de **showback** (informativo, sin cargo interno) y uno de **chargeback** (con cargo al presupuesto del equipo), incluyendo desglose por producto y ambiente.

### Instrucciones

1. Genera la vista detallada por equipo, producto y ambiente:

```python
# Pivot table: Costo directo por Equipo x Producto x Ambiente
df_direct = df[df['allocation_type'] == 'direct'].copy()

pivot_detail = pd.pivot_table(
    df_direct,
    values='Cost',
    index=['tag_Team', 'tag_Product'],
    columns='tag_Environment',
    aggfunc='sum',
    fill_value=0,
    margins=True,
    margins_name='Total'
)

print("=== Desglose por Equipo / Producto / Ambiente (Solo Costos Directos) ===\n")
print(pivot_detail.round(2).to_string())
```

2. Construye el reporte de Showback (visibilidad sin cargo):

```python
# SHOWBACK: Reporte informativo - "Esto es lo que consumiste"
showback = cost_allocation.copy()
showback['model'] = 'Showback'
showback['charge_applied'] = False
showback['notes'] = 'Informativo - Sin cargo a presupuesto de equipo'

# Agregar desglose por ambiente
env_breakdown = df_direct.groupby(['tag_Team', 'tag_Environment'])['Cost'].sum().unstack(fill_value=0)
env_breakdown.columns = [f'cost_{col}' for col in env_breakdown.columns]
env_breakdown = env_breakdown.reset_index().rename(columns={'tag_Team': 'Team'})

showback = showback.merge(env_breakdown, on='Team', how='left')

print("\n=== REPORTE SHOWBACK - NovaTech SRL ===")
print("Modelo: Visibilidad sin cargo interno\n")
print(showback[['Team', 'direct_cost', 'shared_cost_allocated', 'total_allocated']].to_string(index=False))
```

3. Construye el reporte de Chargeback (con cargo interno):

```python
# CHARGEBACK: Reporte con cargo al presupuesto del equipo
chargeback = cost_allocation.copy()
chargeback['model'] = 'Chargeback'
chargeback['charge_applied'] = True

# En chargeback, los costos no asignados se distribuyen también
# Regla: distribuir no asignados proporcionalmente al costo directo
total_direct = chargeback['direct_cost'].sum()
chargeback['unallocated_share'] = (chargeback['direct_cost'] / total_direct) * unallocated_total
chargeback['total_chargeback'] = (chargeback['total_allocated'] + 
                                   chargeback['unallocated_share'])

print("\n=== REPORTE CHARGEBACK - NovaTech SRL ===")
print("Modelo: Cargo completo al presupuesto del equipo\n")
print(f"{'Equipo':<12} | {'Directo':>12} | {'Compartido':>12} | {'No asignado':>12} | {'CARGO TOTAL':>12}")
print("-"*70)
for _, row in chargeback.iterrows():
    print(f"{row['Team']:<12} | ${row['direct_cost']:>9,.2f} | "
          f"${row['shared_cost_allocated']:>9,.2f} | "
          f"${row['unallocated_share']:>9,.2f} | "
          f"${row['total_chargeback']:>9,.2f}")
print("-"*70)
print(f"{'TOTAL':<12} | {' ':>12} | {' ':>12} | {' ':>12} | "
      f"${chargeback['total_chargeback'].sum():>9,.2f}")

# Verificación: el total de chargeback debe ser igual al gasto total
assert abs(chargeback['total_chargeback'].sum() - df['Cost'].sum()) < 0.01, \
    "ERROR: El chargeback no suma el total de la factura"
print("\n✓ Verificación: Chargeback total == Factura total (100% distribuido)")
```

4. Exporta ambos reportes a Excel:

```python
# Exportar archivo consolidado de asignación de costos
output_allocation = '../outputs/lab03/novatech_cost_allocation_q4_2023.xlsx'

with pd.ExcelWriter(output_allocation, engine='openpyxl') as writer:
    # Hoja 1: Resumen de asignación
    cost_allocation.to_excel(writer, sheet_name='AllocationSummary', index=False)
    
    # Hoja 2: Showback
    showback.to_excel(writer, sheet_name='Showback', index=False)
    
    # Hoja 3: Chargeback
    chargeback.to_excel(writer, sheet_name='Chargeback', index=False)
    
    # Hoja 4: Detalle por ambiente
    pivot_detail.to_excel(writer, sheet_name='DetailByEnvironment')
    
    # Hoja 5: Base de prorrateo
    compute_by_team.to_excel(writer, sheet_name='ProrationBasis', index=False)
    
    # Hoja 6: Registros no asignados (para seguimiento)
    df_unallocated = df[df['allocation_type'].isin(['untagged', 'unallocated'])][
        ['UsageDate', 'Provider', 'AccountId', 'ServiceCategory', 'ResourceId', 'Cost']
    ].sort_values('Cost', ascending=False).head(100)
    df_unallocated.to_excel(writer, sheet_name='UnallocatedTop100', index=False)

print(f"\n✓ Reporte de asignación exportado: {output_allocation}")
print(f"  Hojas: AllocationSummary, Showback, Chargeback, DetailByEnvironment, "
      f"ProrationBasis, UnallocatedTop100")
```

### Salida Esperada

```
=== REPORTE CHARGEBACK - NovaTech SRL ===
Modelo: Cargo completo al presupuesto del equipo

Equipo       |      Directo |   Compartido |  No asignado |  CARGO TOTAL
----------------------------------------------------------------------
Backend      |  $75,000.00  |  $19,800.00  |  $12,500.00  | $107,300.00
Frontend     |  $42,000.00  |  $11,550.00  |   $7,000.00  |  $60,550.00
Data         |  $58,000.00  |  $15,400.00  |   $9,667.00  |  $83,067.00
Platform     |  $35,000.00  |   $8,250.00  |   $5,833.00  |  $49,083.00
----------------------------------------------------------------------
TOTAL        |              |              |              | $300,000.00

✓ Verificación: Chargeback total == Factura total (100% distribuido)
✓ Reporte de asignación exportado: ../outputs/lab03/novatech_cost_allocation_q4_2023.xlsx
```

### Verificación

- [ ] El archivo `novatech_cost_allocation_q4_2023.xlsx` tiene 6 hojas
- [ ] En el modelo Chargeback, la suma total es exactamente igual al gasto total de la factura (100% distribuido)
- [ ] En el modelo Showback, los costos no asignados permanecen separados (no se distribuyen)

---

## Paso 6: Calcular Métricas de Calidad de Asignación

### Objetivo

Producir las métricas clave que miden la efectividad de la estrategia de etiquetado: cobertura de etiquetas, costo no asignado y distribución por tipo de asignación.

### Instrucciones

1. Calcula las métricas finales de calidad:

```python
# Métricas de calidad de asignación de costos
total_cost = df['Cost'].sum()
total_records = len(df)

metrics = {
    'Métrica': [
        'Cobertura de etiquetas (por registros)',
        'Cobertura de etiquetas (por costo)',
        'Costo directo asignado',
        'Costo compartido distribuido',
        'Costo no asignado (untagged + unallocated)',
        'Porcentaje no asignado',
        'Registros sin ninguna etiqueta',
        'Top servicio sin etiquetar'
    ],
    'Valor': [
        f"{df[df['allocation_type']=='direct'].shape[0] / total_records * 100:.1f}%",
        f"{df[df['allocation_type']=='direct']['Cost'].sum() / total_cost * 100:.1f}%",
        f"${df[df['allocation_type']=='direct']['Cost'].sum():,.2f}",
        f"${df[df['allocation_type']=='shared']['Cost'].sum():,.2f}",
        f"${df[df['allocation_type'].isin(['untagged','unallocated'])]['Cost'].sum():,.2f}",
        f"{df[df['allocation_type'].isin(['untagged','unallocated'])]['Cost'].sum() / total_cost * 100:.1f}%",
        f"{df[df['allocation_type']=='unallocated'].shape[0]:,}",
        df[df['allocation_type'].isin(['untagged','unallocated'])].groupby('ServiceCategory')['Cost'].sum().idxmax()
    ],
    'Target FinOps': [
        '> 80%',
        '> 80%',
        'Maximizar',
        'Distribuir 100%',
        'Minimizar (< 5%)',
        '< 5% (madurez alta)',
        '0 (ideal)',
        'Priorizar etiquetado'
    ]
}

df_metrics = pd.DataFrame(metrics)
print("\n" + "="*70)
print("  MÉTRICAS DE CALIDAD - ASIGNACIÓN DE COSTOS NovaTech SRL")
print("="*70 + "\n")
print(df_metrics.to_string(index=False))

# Evaluar madurez
pct_unallocated = df[df['allocation_type'].isin(['untagged','unallocated'])]['Cost'].sum() / total_cost * 100
if pct_unallocated > 20:
    maturity = "Crawl (Gatear) - Requiere plan urgente de etiquetado"
elif pct_unallocated > 5:
    maturity = "Walk (Caminar) - Buen progreso, enfocarse en los gaps"
else:
    maturity = "Run (Correr) - Asignación madura"

print(f"\n📊 Nivel de madurez FinOps en asignación: {maturity}")
print(f"   Porcentaje no asignado actual: {pct_unallocated:.1f}%")
```

2. Guarda las métricas como parte del archivo de taxonomía actualizado:

```python
# Actualizar archivo de taxonomía con métricas
with pd.ExcelWriter(output_taxonomy, engine='openpyxl', mode='a', 
                    if_sheet_exists='replace') as writer:
    df_metrics.to_excel(writer, sheet_name='QualityMetrics', index=False)

print(f"\n✓ Métricas agregadas a: {output_taxonomy} (hoja 'QualityMetrics')")
```

### Salida Esperada

```
======================================================================
  MÉTRICAS DE CALIDAD - ASIGNACIÓN DE COSTOS NovaTech SRL
======================================================================

                               Métrica              Valor     Target FinOps
  Cobertura de etiquetas (por registros)            ~70%            > 80%
      Cobertura de etiquetas (por costo)            ~72%            > 80%
                  Costo directo asignado     $~210,000      Maximizar
              Costo compartido distribuido    $~55,000   Distribuir 100%
  Costo no asignado (untagged+unallocated)   $~35,000  Minimizar (< 5%)
                   Porcentaje no asignado       ~12%   < 5% (madurez alta)
             Registros sin ninguna etiqueta     ~800       0 (ideal)
              Top servicio sin etiquetar       S3/Storage  Priorizar etiquetado

📊 Nivel de madurez FinOps en asignación: Walk (Caminar) - Buen progreso, enfocarse en los gaps
   Porcentaje no asignado actual: ~12%
```

### Verificación

- [ ] El porcentaje de costo no asignado está entre 5% y 20% (refleja el ~35% de etiquetas faltantes originales, mejorado por inferencia)
- [ ] Se identifica el servicio con mayor costo sin etiquetar para priorización
- [ ] El nivel de madurez se evalúa correctamente según los umbrales definidos

---

## Validación Final y Testing

Ejecuta las siguientes verificaciones para confirmar que el laboratorio se completó correctamente:

```python
# === VALIDACIÓN FINAL DEL LABORATORIO 03-00-01 ===
import os

print("="*60)
print("  VALIDACIÓN FINAL - Lab 03-00-01")
print("="*60)

checks = []

# Check 1: Archivo de taxonomía existe y tiene hojas correctas
tax_path = '../outputs/lab03/novatech_tagging_taxonomy_v1.xlsx'
if os.path.exists(tax_path):
    tax_xl = pd.ExcelFile(tax_path)
    expected_sheets = {'Taxonomy', 'OrgMapping', 'ValidationRules', 'QualityMetrics'}
    actual_sheets = set(tax_xl.sheet_names)
    check1 = expected_sheets.issubset(actual_sheets)
    checks.append(('Taxonomía con hojas correctas', check1))
else:
    checks.append(('Taxonomía existe', False))

# Check 2: Archivo de asignación existe y tiene hojas correctas
alloc_path = '../outputs/lab03/novatech_cost_allocation_q4_2023.xlsx'
if os.path.exists(alloc_path):
    alloc_xl = pd.ExcelFile(alloc_path)
    expected_alloc = {'AllocationSummary', 'Showback', 'Chargeback', 
                      'DetailByEnvironment', 'ProrationBasis', 'UnallocatedTop100'}
    actual_alloc = set(alloc_xl.sheet_names)
    check2 = expected_alloc.issubset(actual_alloc)
    checks.append(('Asignación con 6 hojas', check2))
else:
    checks.append(('Archivo asignación existe', False))

# Check 3: Chargeback suma 100% de la factura
cb = pd.read_excel(alloc_path, sheet_name='Chargeback')
check3 = abs(cb['total_chargeback'].sum() - df['Cost'].sum()) < 1.0
checks.append(('Chargeback = 100% factura', check3))

# Check 4: Taxonomía tiene 5 etiquetas
tax_df = pd.read_excel(tax_path, sheet_name='Taxonomy')
check4 = len(tax_df) == 5
checks.append(('Taxonomía tiene 5 tags', check4))

# Check 5: Cobertura de etiquetas > 60%
coverage = df[df['allocation_type']=='direct'].shape[0] / len(df) * 100
check5 = coverage > 60
checks.append((f'Cobertura > 60% (actual: {coverage:.1f}%)', check5))

# Imprimir resultados
print()
all_pass = True
for name, result in checks:
    status = "✓ PASS" if result else "✗ FAIL"
    print(f"  {status}  {name}")
    if not result:
        all_pass = False

print("\n" + "="*60)
if all_pass:
    print("  ✓ LABORATORIO COMPLETADO EXITOSAMENTE")
else:
    print("  ✗ HAY VERIFICACIONES PENDIENTES - Revisar pasos anteriores")
print("="*60)
```

**Resultado esperado**: Todas las verificaciones deben mostrar `✓ PASS`.

---

## Resolución de Problemas

### Problema 1: KeyError en columnas de etiquetas

**Síntomas**: Al ejecutar el Paso 1, Python lanza `KeyError: 'tag_Team'` o similar al intentar acceder a las columnas de tags.

**Causa**: El archivo `novatech_billing_90days.csv` puede tener nombres de columnas con formato diferente (mayúsculas, espacios, prefijos distintos). Esto ocurre si el dataset fue editado manualmente o si hay discrepancias entre versiones del archivo provisto.

**Solución**:

```python
# Inspeccionar nombres exactos de columnas
print(df.columns.tolist())

# Buscar columnas que contengan 'tag', 'Tag', 'TAG' o variantes
tag_cols = [c for c in df.columns if 'tag' in c.lower()]
print(f"Columnas de tags encontradas: {tag_cols}")

# Si los nombres son diferentes, renombrar:
# Ejemplo: si la columna es 'Tag_Team' en lugar de 'tag_Team'
rename_map = {}
for col in df.columns:
    if 'tag' in col.lower():
        standardized = 'tag_' + col.split('_', 1)[-1] if '_' in col else col
        rename_map[col] = standardized.lower().replace('tag_', 'tag_')

df.rename(columns=rename_map, inplace=True)
print(f"Columnas renombradas: {rename_map}")
```

### Problema 2: El total de Chargeback no coincide con la factura total

**Síntomas**: La aserción `assert abs(chargeback['total_chargeback'].sum() - df['Cost'].sum()) < 0.01` falla con un error de diferencia mayor a $0.01.

**Causa**: Errores de redondeo acumulados al distribuir costos compartidos y no asignados proporcionalmente. Cuando se aplica `round(2)` a cada fila individualmente, la suma puede diferir del total original por centavos o incluso dólares en datasets grandes.

**Solución**:

```python
# Ajustar el último equipo para absorber la diferencia de redondeo
total_factura = df['Cost'].sum()
total_chargeback = chargeback['total_chargeback'].sum()
difference = total_factura - total_chargeback

if abs(difference) > 0.01:
    print(f"Ajuste de redondeo necesario: ${difference:.4f}")
    # Aplicar diferencia al equipo con mayor costo (menor impacto relativo)
    max_idx = chargeback['total_chargeback'].idxmax()
    chargeback.at[max_idx, 'total_chargeback'] += difference
    chargeback.at[max_idx, 'unallocated_share'] += difference
    print(f"Ajuste aplicado a: {chargeback.at[max_idx, 'Team']}")
    
# Verificar nuevamente
assert abs(chargeback['total_chargeback'].sum() - total_factura) < 0.01
print("✓ Chargeback ahora coincide con factura total")
```

---

## Limpieza

No se requiere limpieza de recursos externos. Los archivos generados son necesarios para el laboratorio 04-00-01:

```python
# Verificar archivos de salida que serán insumos del Lab 04-00-01
import os

outputs = [
    '../outputs/lab03/novatech_cost_allocation_q4_2023.xlsx',
    '../outputs/lab03/novatech_tagging_taxonomy_v1.xlsx'
]

print("Archivos de salida (requeridos por Lab 04-00-01):")
for f in outputs:
    exists = "✓" if os.path.exists(f) else "✗ FALTANTE"
    print(f"  {exists}  {f}")

# Copiar también a data/processed/ para acceso estandarizado
import shutil
for f in outputs:
    dest = f.replace('../outputs/lab03/', '../data/processed/')
    shutil.copy2(f, dest)
    print(f"  → Copiado a: {dest}")
```

**No eliminar** los archivos generados ya que son dependencias directas del siguiente laboratorio.

---

## Resumen

### Logros del Laboratorio

En este laboratorio aplicaste los conceptos fundamentales de asignación de costos FinOps:

| Concepto | Implementación |
|----------|----------------|
| Taxonomía de etiquetas | 5 tags definidos con valores permitidos y reglas de validación |
| Clasificación de costos | 4 categorías: directo, compartido, no etiquetado, no asignable |
| Asignación directa | Registros con tags completos asignados automáticamente |
| Inferencia por cuenta | Registros parciales reclasificados usando mapeo AccountId → Team |
| Prorrateo proporcional | Costos compartidos distribuidos según consumo de compute |
| Showback | Visibilidad de costos sin cargo interno |
| Chargeback | 100% de la factura distribuida entre equipos responsables |
| Métricas de calidad | Cobertura de etiquetas y % no asignado como KPIs |

### Archivos Generados

| Archivo | Propósito | Requerido por |
|---------|-----------|---------------|
| `novatech_tagging_taxonomy_v1.xlsx` | Estrategia de etiquetado | Lab 04-00-01 |
| `novatech_cost_allocation_q4_2023.xlsx` | Modelo completo de asignación | Lab 04-00-01 |

### Recursos Adicionales

- [FinOps Foundation — Cost Allocation](https://www.finops.org/framework/domains/cost-allocation/)
- [FinOps Foundation — Tagging and Labeling](https://www.finops.org/framework/capabilities/tagging-labeling/)
- [AWS Tagging Best Practices](https://docs.aws.amazon.com/whitepapers/latest/tagging-best-practices/tagging-best-practices.html)
- [Azure Cost Allocation Rules](https://learn.microsoft.com/en-us/azure/cost-management-billing/costs/allocate-costs)

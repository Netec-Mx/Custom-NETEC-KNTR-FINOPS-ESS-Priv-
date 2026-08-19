# Demostración 9. Caso integrador: diagnóstico, asignación, reporte y plan de optimización

## Metadata

| Campo | Valor |
|-------|-------|
| **Duración** | 45 minutos |
| **Complejidad** | Alta |
| **Nivel Bloom** | Crear |
| **Tecnologías** | Python 3.12.1, JupyterLab 4.1.5, pandas 2.2.1, plotly 5.20.0, openpyxl 3.1.2, SQLAlchemy 2.0.28, psycopg2-binary 2.9.9, PostgreSQL 16.2, pyyaml 6.0.1, rich 13.7.1, nbconvert 7.16.2, Grafana 10.3.3, Docker Compose 2.24.6 |

## Descripción General

Este laboratorio final integra todos los artefactos generados durante el curso en un diagnóstico FinOps completo para TechCorp S.A. El estudiante ejecutará un script de diagnóstico automatizado contra el data warehouse, construirá un roadmap de 30-60-90 días con responsables y KPIs, definirá rituales operativos, y generará un reporte ejecutivo HTML autocontenido como artefacto de cierre del curso.

## Objetivos de Aprendizaje

- [ ] Integrar artefactos de labs anteriores (backlog, RACI, data warehouse, análisis de drivers) en un diagnóstico FinOps completo
- [ ] Evaluar la madurez FinOps actual de TechCorp S.A. y proyectar el estado objetivo a 90 días usando el modelo Crawl-Walk-Run
- [ ] Diseñar un roadmap de implementación de 30-60-90 días con iniciativas priorizadas, responsables y KPIs medibles
- [ ] Definir KPIs operativos y rituales de gestión con responsables asignados desde la matriz RACI
- [ ] Generar un reporte ejecutivo consolidado en formato HTML autocontenido usando Jupyter y nbconvert

## Prerrequisitos

### Conocimientos Previos

- Completados los labs 05-00-01 (backlog), 06-00-01 (gobernanza), 07-00-01 (data warehouse + Grafana), 08-00-01 (priorización de drivers)
- Familiaridad con el modelo de madurez FinOps (Crawl-Walk-Run)
- Manejo básico de SQL, Python/pandas y YAML

### Artefactos Requeridos

| Artefacto | Ruta esperada |
|-----------|---------------|
| Backlog v1 | `~/finops-labs/lab05/output/finops_backlog_v1.xlsx` |
| Gobernanza v1 | `~/finops-labs/lab06/output/finops_governance_v1.xlsx` |
| Backlog v2 | `~/finops-labs/lab06/output/finops_backlog_v2.xlsx` |
| Políticas YAML | `~/finops-labs/lab06/policies/*.yaml` |
| Dashboard Grafana | `~/finops-labs/lab07/grafana/finops_overview_dashboard.json` |
| Backlog v3 | `~/finops-labs/lab08/output/finops_backlog_v3.xlsx` |

### Servicios Activos

| Servicio | Puerto | Estado requerido |
|----------|--------|-----------------|
| PostgreSQL 16.2 | 5432 | En ejecución (Docker) |
| Grafana 10.3.3 | 3000 | En ejecución (Docker) |

## Entorno del Laboratorio

### Software Requerido

| Componente | Versión |
|------------|---------|
| Python | 3.12.1 |
| pandas | 2.2.1 |
| plotly | 5.20.0 |
| openpyxl | 3.1.2 |
| SQLAlchemy | 2.0.28 |
| psycopg2-binary | 2.9.9 |
| pyyaml | 6.0.1 |
| rich | 13.7.1 |
| nbconvert | 7.16.2 |
| JupyterLab | 4.1.5 |
| Docker Compose | 2.24.6 |

### Configuración Inicial

**1. Crear estructura de directorios:**

```bash
mkdir -p ~/finops-labs/lab09/{scripts,templates,output,reports}
cd ~/finops-labs/lab09
```

**2. Verificar dependencias Python:**

```bash
pip install pandas==2.2.1 plotly==5.20.0 openpyxl==3.1.2 \
  SQLAlchemy==2.0.28 psycopg2-binary==2.9.9 pyyaml==6.0.1 \
  rich==13.7.1 nbconvert==7.16.2 jupyterlab==4.1.5
```

**3. Verificar servicios Docker activos:**

```bash
cd ~/finops-labs/lab07
docker compose ps
```

Salida esperada (ambos servicios `running`):

```
NAME                STATUS
lab07-postgres-1    Up
lab07-grafana-1     Up
```

**4. Verificar conectividad a PostgreSQL:**

```bash
python3 -c "
from sqlalchemy import create_engine, text
engine = create_engine('postgresql+psycopg2://finops:finops123@localhost:5432/finops_dw')
with engine.connect() as conn:
    result = conn.execute(text('SELECT count(*) FROM staging.billing_raw'))
    print(f'Registros en staging.billing_raw: {result.scalar()}')
"
```

> **Nota:** Si no completaste los labs previos, solicita al instructor los artefactos de referencia pre-completados y colócalos en las rutas indicadas.

---

## Paso a Paso

### Paso 1 — FASE 1: Script de Diagnóstico Automatizado (10 min)

**Objetivo:** Crear y ejecutar un script Python que consulte el data warehouse, lea los artefactos de gobernanza y genere un informe de estado actual con score de madurez, cobertura de tags, recursos sin propietario, ahorro potencial y gaps de gobernanza.

**Instrucciones:**

1. Crear el archivo `~/finops-labs/lab09/scripts/diagnostico_finops.py`:

```python
#!/usr/bin/env python3
"""
Diagnóstico FinOps Automatizado - TechCorp S.A.
Lab 09-00-01 | Caso Integrador Final
"""

import pandas as pd
import yaml
from pathlib import Path
from sqlalchemy import create_engine, text
from rich.console import Console
from rich.table import Table
from rich import box
from datetime import datetime

# === CONFIGURACIÓN ===
DB_URL = "postgresql+psycopg2://finops:finops123@localhost:5432/finops_dw"
BASE_PATH = Path.home() / "finops-labs"
OUTPUT_PATH = BASE_PATH / "lab09" / "output"
OUTPUT_PATH.mkdir(parents=True, exist_ok=True)

console = Console()

def get_engine():
    return create_engine(DB_URL)

def diagnosticar_cobertura_tags(engine):
    """Calcula % de recursos con etiquetas completas."""
    query = text("""
        SELECT 
            COUNT(*) as total_records,
            COUNT(CASE WHEN team IS NOT NULL AND team != '' THEN 1 END) as tagged_team,
            COUNT(CASE WHEN environment IS NOT NULL AND environment != '' THEN 1 END) as tagged_env,
            COUNT(CASE WHEN product IS NOT NULL AND product != '' THEN 1 END) as tagged_product
        FROM staging.billing_raw
    """)
    with engine.connect() as conn:
        result = conn.execute(query).fetchone()
    
    total = result[0] if result[0] > 0 else 1
    coverage = {
        "team": round(result[1] / total * 100, 1),
        "environment": round(result[2] / total * 100, 1),
        "product": round(result[3] / total * 100, 1),
    }
    coverage["promedio"] = round(sum(coverage.values()) / 3, 1)
    return coverage, total

def diagnosticar_recursos_sin_owner(engine):
    """Identifica recursos sin propietario asignado."""
    query = text("""
        SELECT 
            COUNT(DISTINCT resource_id) as sin_owner
        FROM staging.billing_raw
        WHERE team IS NULL OR team = ''
    """)
    with engine.connect() as conn:
        result = conn.execute(query).scalar()
    return result or 0

def diagnosticar_ahorro_backlog():
    """Lee el backlog v3 y suma el ahorro potencial total."""
    backlog_path = BASE_PATH / "lab08" / "output" / "finops_backlog_v3.xlsx"
    if not backlog_path.exists():
        return 0, 0, []
    
    df = pd.read_excel(backlog_path)
    ahorro_total = df["estimated_savings_usd"].sum() if "estimated_savings_usd" in df.columns else 0
    num_iniciativas = len(df)
    
    # Top 5 iniciativas por ahorro
    top5 = df.nlargest(5, "estimated_savings_usd")[
        ["initiative_id", "title", "estimated_savings_usd", "priority"]
    ].to_dict("records") if "estimated_savings_usd" in df.columns else []
    
    return round(ahorro_total, 2), num_iniciativas, top5

def diagnosticar_gobernanza():
    """Evalúa gaps de gobernanza leyendo artefactos del lab06."""
    gov_path = BASE_PATH / "lab06" / "output" / "finops_governance_v1.xlsx"
    policies_path = BASE_PATH / "lab06" / "policies"
    
    gaps = []
    
    if not gov_path.exists():
        gaps.append("Matriz RACI no encontrada - gobernanza no formalizada")
    else:
        df_raci = pd.read_excel(gov_path, sheet_name="RACI")
        # Verificar si hay procesos sin responsable
        if "Responsible" in df_raci.columns:
            sin_resp = df_raci[df_raci["Responsible"].isna() | (df_raci["Responsible"] == "")]
            if len(sin_resp) > 0:
                gaps.append(f"{len(sin_resp)} procesos sin responsable asignado en RACI")
    
    if not policies_path.exists() or not list(policies_path.glob("*.yaml")):
        gaps.append("Políticas YAML no definidas - enforcement no automatizable")
    else:
        num_policies = len(list(policies_path.glob("*.yaml")))
        if num_policies < 3:
            gaps.append(f"Solo {num_policies} política(s) YAML definida(s) - cobertura insuficiente")
    
    return gaps

def calcular_score_madurez(coverage, recursos_sin_owner, ahorro_total, gaps):
    """Calcula score de madurez FinOps 0-100 basado en criterios ponderados."""
    scores = {}
    
    # Asignación de costos (peso 25%) - basado en cobertura de tags
    if coverage["promedio"] >= 85:
        scores["asignacion"] = 3
    elif coverage["promedio"] >= 50:
        scores["asignacion"] = 2
    else:
        scores["asignacion"] = 1
    
    # Optimización de tarifas (peso 25%) - basado en backlog existente
    if ahorro_total > 0:
        scores["optimizacion"] = 2  # Al menos Walk si hay backlog
    else:
        scores["optimizacion"] = 1
    
    # Gobernanza (peso 25%) - basado en gaps
    if len(gaps) == 0:
        scores["gobernanza"] = 3
    elif len(gaps) <= 2:
        scores["gobernanza"] = 2
    else:
        scores["gobernanza"] = 1
    
    # Visibilidad (peso 25%) - basado en DW activo
    scores["visibilidad"] = 2  # Walk: DW existe y funciona
    
    # Score global (promedio ponderado normalizado a 100)
    score_global = round((sum(scores.values()) / (4 * 3)) * 100, 1)
    
    return scores, score_global

def generar_informe(coverage, total_records, recursos_sin_owner, 
                    ahorro_total, num_iniciativas, top5, gaps, scores, score_global):
    """Genera informe en consola y exporta a YAML."""
    
    console.print("\n[bold cyan]═══════════════════════════════════════════════════════[/bold cyan]")
    console.print("[bold cyan]   DIAGNÓSTICO FINOPS - TECHCORP S.A.                  [/bold cyan]")
    console.print(f"[bold cyan]   Fecha: {datetime.now().strftime('%Y-%m-%d %H:%M')}[/bold cyan]")
    console.print("[bold cyan]═══════════════════════════════════════════════════════[/bold cyan]\n")
    
    # Score de madurez
    color = "red" if score_global < 40 else "yellow" if score_global < 70 else "green"
    console.print(f"[bold {color}]Score de Madurez Global: {score_global}/100[/bold {color}]\n")
    
    # Tabla de capacidades
    table = Table(title="Madurez por Capacidad", box=box.ROUNDED)
    table.add_column("Capacidad", style="cyan")
    table.add_column("Nivel", justify="center")
    table.add_column("Estado", justify="center")
    
    nivel_map = {1: ("Crawl 🔴", "red"), 2: ("Walk 🟡", "yellow"), 3: ("Run 🟢", "green")}
    for cap, nivel in scores.items():
        nombre, style = nivel_map[nivel]
        table.add_row(cap.capitalize(), str(nivel), f"[{style}]{nombre}[/{style}]")
    console.print(table)
    
    # Cobertura de tags
    console.print(f"\n[bold]Cobertura de Etiquetado:[/bold]")
    console.print(f"  • Team: {coverage['team']}%")
    console.print(f"  • Environment: {coverage['environment']}%")
    console.print(f"  • Product: {coverage['product']}%")
    console.print(f"  • [bold]Promedio: {coverage['promedio']}%[/bold]")
    
    # Recursos sin owner
    console.print(f"\n[bold]Recursos sin propietario:[/bold] {recursos_sin_owner}")
    
    # Ahorro potencial
    console.print(f"\n[bold]Ahorro potencial total (backlog):[/bold] ${ahorro_total:,.2f} USD")
    console.print(f"[bold]Iniciativas en backlog:[/bold] {num_iniciativas}")
    
    # Gaps de gobernanza
    console.print(f"\n[bold]Gaps de Gobernanza ({len(gaps)}):[/bold]")
    for gap in gaps:
        console.print(f"  ⚠️  {gap}")
    
    # Exportar a YAML
    informe_data = {
        "diagnostico_finops": {
            "organizacion": "TechCorp S.A.",
            "fecha": datetime.now().strftime("%Y-%m-%d"),
            "score_global": score_global,
            "capacidades": {k: {"nivel": v, "estado": nivel_map[v][0].split()[0]} 
                          for k, v in scores.items()},
            "cobertura_tags": coverage,
            "recursos_sin_owner": recursos_sin_owner,
            "ahorro_potencial_usd": ahorro_total,
            "num_iniciativas_backlog": num_iniciativas,
            "gaps_gobernanza": gaps,
            "total_registros_dw": total_records,
        }
    }
    
    output_file = OUTPUT_PATH / "diagnostico_finops_techcorp.yaml"
    with open(output_file, "w", encoding="utf-8") as f:
        yaml.dump(informe_data, f, default_flow_style=False, allow_unicode=True)
    
    console.print(f"\n[green]✅ Informe exportado: {output_file}[/green]\n")
    return informe_data

def main():
    engine = get_engine()
    
    console.print("[dim]Ejecutando diagnóstico...[/dim]")
    
    # 1. Cobertura de tags
    coverage, total_records = diagnosticar_cobertura_tags(engine)
    
    # 2. Recursos sin owner
    recursos_sin_owner = diagnosticar_recursos_sin_owner(engine)
    
    # 3. Ahorro potencial del backlog
    ahorro_total, num_iniciativas, top5 = diagnosticar_ahorro_backlog()
    
    # 4. Gaps de gobernanza
    gaps = diagnosticar_gobernanza()
    
    # 5. Score de madurez
    scores, score_global = calcular_score_madurez(
        coverage, recursos_sin_owner, ahorro_total, gaps
    )
    
    # 6. Generar informe
    generar_informe(coverage, total_records, recursos_sin_owner,
                    ahorro_total, num_iniciativas, top5, gaps, scores, score_global)

if __name__ == "__main__":
    main()
```

2. Ejecutar el script de diagnóstico:

```bash
cd ~/finops-labs/lab09
python3 scripts/diagnostico_finops.py
```

**Salida esperada:**

```
Ejecutando diagnóstico...

═══════════════════════════════════════════════════════
   DIAGNÓSTICO FINOPS - TECHCORP S.A.
   Fecha: 2025-06-15 14:30
═══════════════════════════════════════════════════════

Score de Madurez Global: 41.7/100

┌──────────────────────────────────────────┐
│       Madurez por Capacidad              │
├──────────────┬───────┬───────────────────┤
│ Capacidad    │ Nivel │ Estado            │
├──────────────┼───────┼───────────────────┤
│ Asignacion   │ 1     │ Crawl 🔴          │
│ Optimizacion │ 2     │ Walk 🟡           │
│ Gobernanza   │ 2     │ Walk 🟡           │
│ Visibilidad  │ 2     │ Walk 🟡           │
└──────────────┴───────┴───────────────────┘

Cobertura de Etiquetado:
  • Team: 42.3%
  • Environment: 67.8%
  • Product: 55.1%
  • Promedio: 55.1%

Recursos sin propietario: 847

Ahorro potencial total (backlog): $127,450.00 USD
Iniciativas en backlog: 12

Gaps de Gobernanza (2):
  ⚠️  2 procesos sin responsable asignado en RACI
  ⚠️  Solo 2 política(s) YAML definida(s) - cobertura insuficiente

✅ Informe exportado: ~/finops-labs/lab09/output/diagnostico_finops_techcorp.yaml
```

**Verificación:**

```bash
cat ~/finops-labs/lab09/output/diagnostico_finops_techcorp.yaml
```

Debe contener las claves `diagnostico_finops.score_global`, `capacidades`, `cobertura_tags` y `gaps_gobernanza`.

---

### Paso 2 — FASE 2: Roadmap 30-60-90 Días (15 min)

**Objetivo:** Clasificar las iniciativas del backlog en 3 horizontes temporales, asignar responsables usando la matriz RACI y definir criterios de éxito medibles.

**Instrucciones:**

1. Crear el archivo `~/finops-labs/lab09/scripts/generar_roadmap.py`:

```python
#!/usr/bin/env python3
"""
Generador de Roadmap FinOps 30-60-90 días - TechCorp S.A.
"""

import pandas as pd
from pathlib import Path
from rich.console import Console

BASE_PATH = Path.home() / "finops-labs"
OUTPUT_PATH = BASE_PATH / "lab09" / "output"

console = Console()

def cargar_backlog():
    """Carga backlog v3 del lab08."""
    path = BASE_PATH / "lab08" / "output" / "finops_backlog_v3.xlsx"
    if not path.exists():
        console.print("[red]ERROR: finops_backlog_v3.xlsx no encontrado[/red]")
        console.print("[yellow]Generando backlog de ejemplo para continuar...[/yellow]")
        return generar_backlog_ejemplo()
    return pd.read_excel(path)

def generar_backlog_ejemplo():
    """Genera backlog de ejemplo si no existe el del lab08."""
    data = {
        "initiative_id": [f"FIN-{i:03d}" for i in range(1, 13)],
        "title": [
            "Implementar política de etiquetado obligatorio",
            "Comprar Reserved Instances EC2 (1yr)",
            "Rightsizing instancias sobredimensionadas",
            "Eliminar EBS volumes huérfanos",
            "Configurar Savings Plans Compute",
            "Automatizar apagado dev/staging noches",
            "Implementar showback por equipo",
            "Crear alertas de anomalía por servicio",
            "Migrar workloads a Spot Instances",
            "Consolidar cuentas AWS subutilizadas",
            "Implementar FinOps Standup semanal",
            "Definir Unit Economics por producto",
        ],
        "estimated_savings_usd": [
            0, 32000, 18500, 4200, 28000, 12600,
            0, 0, 15800, 8400, 0, 0
        ],
        "effort": [
            "medium", "low", "medium", "low", "medium", "medium",
            "high", "medium", "high", "low", "low", "high"
        ],
        "priority": [
            "P1", "P1", "P1", "P2", "P1", "P2",
            "P2", "P2", "P3", "P2", "P1", "P3"
        ],
        "category": [
            "governance", "rate_optimization", "usage_optimization",
            "usage_optimization", "rate_optimization", "usage_optimization",
            "allocation", "anomaly_mgmt", "rate_optimization",
            "governance", "culture", "metrics"
        ],
        "dependencies": [
            "none", "none", "FIN-001", "none", "FIN-002",
            "FIN-001", "FIN-001", "FIN-007", "FIN-003",
            "none", "none", "FIN-007"
        ]
    }
    return pd.DataFrame(data)

def cargar_raci():
    """Carga matriz RACI del lab06."""
    path = BASE_PATH / "lab06" / "output" / "finops_governance_v1.xlsx"
    if not path.exists():
        return generar_raci_ejemplo()
    try:
        return pd.read_excel(path, sheet_name="RACI")
    except Exception:
        return generar_raci_ejemplo()

def generar_raci_ejemplo():
    """RACI de ejemplo si no existe."""
    return pd.DataFrame({
        "process": [
            "Tag Enforcement", "RI Purchasing", "Rightsizing",
            "Budget Management", "Anomaly Detection", "Cost Allocation",
            "Showback Reporting", "Sprint Planning", "Executive Review"
        ],
        "Responsible": [
            "Platform Team", "FinOps Lead", "Engineering Leads",
            "FinOps Lead", "Platform Team", "FinOps Lead",
            "FinOps Lead", "Engineering Leads", "CFO"
        ],
        "Accountable": [
            "CTO", "CFO", "CTO",
            "CFO", "CTO", "CFO",
            "CFO", "CTO", "CEO"
        ],
        "Consulted": [
            "Engineering Leads", "Platform Team", "Platform Team",
            "Engineering Leads", "FinOps Lead", "Engineering Leads",
            "Product Owners", "FinOps Lead", "CTO"
        ],
        "Informed": [
            "CFO", "CTO", "CFO",
            "Product Owners", "CFO", "Product Owners",
            "Engineering Leads", "CFO", "Board"
        ]
    })

def clasificar_horizonte(row):
    """Clasifica iniciativa en horizonte 30/60/90 días."""
    # Criterios: prioridad + esfuerzo + dependencias
    if row["priority"] == "P1" and row["effort"] == "low" and row["dependencies"] == "none":
        return 30
    elif row["priority"] == "P1" and row["effort"] in ["low", "medium"]:
        return 30
    elif row["priority"] in ["P1", "P2"] and row["effort"] == "medium":
        return 60
    elif row["priority"] == "P2" and row["effort"] in ["low", "medium"]:
        return 60
    else:
        return 90

def asignar_responsable(row, raci_df):
    """Asigna responsable basado en categoría y RACI."""
    category_to_process = {
        "governance": "Tag Enforcement",
        "rate_optimization": "RI Purchasing",
        "usage_optimization": "Rightsizing",
        "allocation": "Cost Allocation",
        "anomaly_mgmt": "Anomaly Detection",
        "culture": "Sprint Planning",
        "metrics": "Showback Reporting",
    }
    process = category_to_process.get(row["category"], "Budget Management")
    match = raci_df[raci_df["process"] == process]
    if not match.empty:
        return match.iloc[0]["Responsible"]
    return "FinOps Lead"

def definir_kpi_exito(row):
    """Define KPI de éxito medible por iniciativa."""
    kpis = {
        "governance": "Cobertura de tags >= 85% en 30 días post-implementación",
        "rate_optimization": f"Ahorro mensual >= ${row['estimated_savings_usd']/12:,.0f} USD",
        "usage_optimization": f"Reducción de waste >= ${row['estimated_savings_usd']/12:,.0f} USD/mes",
        "allocation": "100% del spend asignado a centro de costo",
        "anomaly_mgmt": "Tiempo de detección de anomalía < 4 horas",
        "culture": "Asistencia a rituales >= 80% de stakeholders",
        "metrics": "Unit Economics definidos para 100% de productos",
    }
    return kpis.get(row["category"], "Definir en planning")

def main():
    console.print("[bold cyan]Generando Roadmap FinOps 30-60-90...[/bold cyan]\n")
    
    # Cargar datos
    backlog = cargar_backlog()
    raci = cargar_raci()
    
    # Clasificar horizontes
    backlog["horizonte_dias"] = backlog.apply(clasificar_horizonte, axis=1)
    backlog["responsable"] = backlog.apply(lambda r: asignar_responsable(r, raci), axis=1)
    backlog["kpi_exito"] = backlog.apply(definir_kpi_exito, axis=1)
    
    # Estado objetivo de madurez por horizonte
    backlog["madurez_objetivo"] = backlog["horizonte_dias"].map({
        30: "Crawl → Walk",
        60: "Walk consolidado",
        90: "Walk → Run"
    })
    
    # Generar Excel con una hoja por horizonte
    output_file = OUTPUT_PATH / "roadmap_finops.xlsx"
    
    with pd.ExcelWriter(output_file, engine="openpyxl") as writer:
        for horizonte in [30, 60, 90]:
            df_h = backlog[backlog["horizonte_dias"] == horizonte].copy()
            df_h = df_h.sort_values("estimated_savings_usd", ascending=False)
            df_h.to_excel(writer, sheet_name=f"Horizonte_{horizonte}_dias", index=False)
        
        # Hoja resumen
        resumen = backlog.groupby("horizonte_dias").agg(
            num_iniciativas=("initiative_id", "count"),
            ahorro_total_usd=("estimated_savings_usd", "sum"),
        ).reset_index()
        resumen.to_excel(writer, sheet_name="Resumen", index=False)
    
    # Imprimir resumen
    for horizonte in [30, 60, 90]:
        df_h = backlog[backlog["horizonte_dias"] == horizonte]
        ahorro = df_h["estimated_savings_usd"].sum()
        console.print(f"[bold]Horizonte {horizonte} días:[/bold] "
                     f"{len(df_h)} iniciativas | "
                     f"Ahorro estimado: ${ahorro:,.0f} USD")
        for _, row in df_h.iterrows():
            console.print(f"  • [{row['priority']}] {row['title']} → {row['responsable']}")
    
    console.print(f"\n[green]✅ Roadmap exportado: {output_file}[/green]")

if __name__ == "__main__":
    main()
```

2. Ejecutar el generador de roadmap:

```bash
python3 scripts/generar_roadmap.py
```

**Salida esperada:**

```
Generando Roadmap FinOps 30-60-90...

Horizonte 30 días: 4 iniciativas | Ahorro estimado: $54,700 USD
  • [P1] Comprar Reserved Instances EC2 (1yr) → FinOps Lead
  • [P1] Implementar política de etiquetado obligatorio → Platform Team
  • [P1] Rightsizing instancias sobredimensionadas → Engineering Leads
  • [P1] Implementar FinOps Standup semanal → Engineering Leads

Horizonte 60 días: 5 iniciativas | Ahorro estimado: $44,800 USD
  • [P1] Configurar Savings Plans Compute → FinOps Lead
  • [P2] Automatizar apagado dev/staging noches → Platform Team
  • [P2] Eliminar EBS volumes huérfanos → Engineering Leads
  • [P2] Implementar showback por equipo → FinOps Lead
  • [P2] Crear alertas de anomalía por servicio → Platform Team

Horizonte 90 días: 3 iniciativas | Ahorro estimado: $24,200 USD
  • [P3] Migrar workloads a Spot Instances → Engineering Leads
  • [P2] Consolidar cuentas AWS subutilizadas → Platform Team
  • [P3] Definir Unit Economics por producto → FinOps Lead

✅ Roadmap exportado: ~/finops-labs/lab09/output/roadmap_finops.xlsx
```

**Verificación:**

```bash
python3 -c "
import openpyxl
wb = openpyxl.load_workbook('$HOME/finops-labs/lab09/output/roadmap_finops.xlsx')
print('Hojas:', wb.sheetnames)
for sheet in wb.sheetnames:
    ws = wb[sheet]
    print(f'  {sheet}: {ws.max_row-1} filas')
"
```

Debe mostrar 4 hojas: `Horizonte_30_dias`, `Horizonte_60_dias`, `Horizonte_90_dias`, `Resumen`.

---

### Paso 3 — FASE 3: KPIs y Rituales Operativos (10 min)

**Objetivo:** Definir los 5 KPIs de eficiencia FinOps y los 5 rituales operativos con responsables asignados desde la matriz RACI.

**Instrucciones:**

1. Crear el archivo `~/finops-labs/lab09/templates/kpis_rituales.yaml`:

```yaml
# KPIs y Rituales FinOps - TechCorp S.A.
# Lab 09-00-01 | Caso Integrador Final

organizacion: "TechCorp S.A."
fecha_definicion: "2025-06-15"
version: "1.0"

kpis_eficiencia:
  - id: KPI-001
    nombre: "Unit Economics"
    descripcion: "Costo cloud por transacción procesada por producto"
    formula: "total_cloud_cost / total_transactions"
    meta_crawl: "Definido para 1 producto"
    meta_walk: "Definido para todos los productos, medido mensualmente"
    meta_run: "Automatizado, medido diariamente, con tendencia y forecast"
    meta_actual: "No definido"
    meta_90_dias: "Definido para PayCore y AnalyticsHub"
    responsable: "FinOps Lead"
    frecuencia_medicion: "semanal"
    fuente_datos: "marts.cost_by_product + sistema transaccional"

  - id: KPI-002
    nombre: "% Untagged Spend"
    descripcion: "Porcentaje del gasto cloud sin etiquetas completas"
    formula: "(spend_sin_tags / spend_total) * 100"
    meta_crawl: "< 60%"
    meta_walk: "< 20%"
    meta_run: "< 5%"
    meta_actual: "45%"
    meta_90_dias: "< 15%"
    responsable: "Platform Team"
    frecuencia_medicion: "diaria"
    fuente_datos: "staging.billing_raw → marts.tag_coverage"

  - id: KPI-003
    nombre: "Budget Variance"
    descripcion: "Desviación porcentual entre gasto real y presupuestado"
    formula: "((actual_spend - budget) / budget) * 100"
    meta_crawl: "< 30%"
    meta_walk: "< 15%"
    meta_run: "< 5%"
    meta_actual: "22%"
    meta_90_dias: "< 12%"
    responsable: "FinOps Lead"
    frecuencia_medicion: "semanal"
    fuente_datos: "marts.budget_vs_actual"

  - id: KPI-004
    nombre: "Savings Rate"
    descripcion: "Porcentaje de ahorro realizado vs. gasto on-demand equivalente"
    formula: "(on_demand_equivalent - actual_spend) / on_demand_equivalent * 100"
    meta_crawl: "< 10%"
    meta_walk: "15-30%"
    meta_run: "> 30%"
    meta_actual: "3%"
    meta_90_dias: "> 18%"
    responsable: "FinOps Lead"
    frecuencia_medicion: "mensual"
    fuente_datos: "CUR/Billing Export → cálculo de savings"

  - id: KPI-005
    nombre: "Coverage Rate"
    descripcion: "Porcentaje de compute cubierto por compromisos (RI/SP)"
    formula: "(horas_cubiertas / horas_totales) * 100"
    meta_crawl: "< 20%"
    meta_walk: "40-65%"
    meta_run: "> 70%"
    meta_actual: "0%"
    meta_90_dias: "> 45%"
    responsable: "FinOps Lead"
    frecuencia_medicion: "semanal"
    fuente_datos: "AWS Cost Explorer Reservation Utilization"

rituales_semanales:
  - id: RIT-W01
    nombre: "FinOps Standup"
    descripcion: "Revisión rápida de estado de iniciativas del backlog activo"
    duracion_minutos: 15
    dia: "Lunes"
    hora: "09:30"
    participantes:
      - rol: "Facilitador"
        persona: "FinOps Lead"
      - rol: "Participantes"
        personas: ["Platform Team Lead", "Engineering Leads", "Product Owners"]
    agenda:
      - "Revisión de KPIs clave (5 min)"
      - "Estado de iniciativas en curso (5 min)"
      - "Blockers y escalaciones (5 min)"
    artefacto_salida: "Actualización de estado en backlog"

  - id: RIT-W02
    nombre: "Anomaly Review"
    descripcion: "Triaje de anomalías de costo detectadas en la semana"
    duracion_minutos: 30
    dia: "Miércoles"
    hora: "10:00"
    participantes:
      - rol: "Facilitador"
        persona: "Platform Team Lead"
      - rol: "Participantes"
        personas: ["FinOps Lead", "On-call Engineer"]
    agenda:
      - "Revisión de alertas disparadas (10 min)"
      - "Clasificación: falso positivo / accionable (10 min)"
      - "Asignación de acciones correctivas (10 min)"
    artefacto_salida: "Anomaly log actualizado con disposición"

  - id: RIT-W03
    nombre: "Backlog Grooming"
    descripcion: "Refinamiento y repriorización del backlog de optimización"
    duracion_minutos: 30
    dia: "Viernes"
    hora: "14:00"
    participantes:
      - rol: "Facilitador"
        persona: "FinOps Lead"
      - rol: "Participantes"
        personas: ["Platform Team Lead", "Engineering Leads"]
    agenda:
      - "Nuevas oportunidades identificadas (10 min)"
      - "Re-estimación de iniciativas en curso (10 min)"
      - "Priorización para siguiente sprint (10 min)"
    artefacto_salida: "Backlog actualizado con nuevas estimaciones"

rituales_mensuales:
  - id: RIT-M01
    nombre: "Cost Review Meeting"
    descripcion: "Revisión ejecutiva mensual de costos, tendencias y cumplimiento de presupuesto"
    duracion_minutos: 60
    dia: "Primer martes del mes"
    hora: "11:00"
    participantes:
      - rol: "Presentador"
        persona: "FinOps Lead"
      - rol: "Audiencia"
        personas: ["CFO", "CTO", "VP Engineering", "Product Owners"]
    agenda:
      - "Resumen de gasto vs. presupuesto (15 min)"
      - "Tendencias y forecast próximo mes (15 min)"
      - "Progreso de optimización y savings realizados (15 min)"
      - "Decisiones y aprobaciones requeridas (15 min)"
    artefacto_salida: "Monthly Cost Report + decisiones documentadas"

  - id: RIT-M02
    nombre: "Optimization Sprint Planning"
    descripcion: "Planificación del sprint mensual de optimización"
    duracion_minutos: 45
    dia: "Primer viernes del mes"
    hora: "14:00"
    participantes:
      - rol: "Facilitador"
        persona: "FinOps Lead"
      - rol: "Participantes"
        personas: ["Platform Team Lead", "Engineering Leads", "CTO"]
    agenda:
      - "Retrospectiva del sprint anterior (10 min)"
      - "Selección de iniciativas para el mes (20 min)"
      - "Asignación de recursos y ownership (15 min)"
    artefacto_salida: "Sprint backlog del mes con owners asignados"
```

2. Validar la estructura YAML:

```bash
python3 -c "
import yaml
from pathlib import Path

path = Path.home() / 'finops-labs/lab09/templates/kpis_rituales.yaml'
with open(path) as f:
    data = yaml.safe_load(f)

print(f'KPIs definidos: {len(data[\"kpis_eficiencia\"])}')
print(f'Rituales semanales: {len(data[\"rituales_semanales\"])}')
print(f'Rituales mensuales: {len(data[\"rituales_mensuales\"])}')
print()
for kpi in data['kpis_eficiencia']:
    print(f'  [{kpi[\"id\"]}] {kpi[\"nombre\"]}: actual={kpi[\"meta_actual\"]} → objetivo 90d={kpi[\"meta_90_dias\"]}')
"
```

**Salida esperada:**

```
KPIs definidos: 5
Rituales semanales: 3
Rituales mensuales: 2

  [KPI-001] Unit Economics: actual=No definido → objetivo 90d=Definido para PayCore y AnalyticsHub
  [KPI-002] % Untagged Spend: actual=45% → objetivo 90d=< 15%
  [KPI-003] Budget Variance: actual=22% → objetivo 90d=< 12%
  [KPI-004] Savings Rate: actual=3% → objetivo 90d=> 18%
  [KPI-005] Coverage Rate: actual=0% → objetivo 90d=> 45%
```

**Verificación:**

```bash
# Copiar a output para trazabilidad
cp ~/finops-labs/lab09/templates/kpis_rituales.yaml ~/finops-labs/lab09/output/
ls ~/finops-labs/lab09/output/kpis_rituales.yaml
```

---

### Paso 4 — FASE 4: Reporte Ejecutivo en Jupyter Notebook (10 min)

**Objetivo:** Crear y ejecutar un notebook que consolide todos los análisis, genere visualizaciones con plotly y exporte el reporte como HTML autocontenido.

**Instrucciones:**

1. Crear el notebook `~/finops-labs/lab09/reports/lab09_reporte_ejecutivo.ipynb`. Utilizar el siguiente script Python para generarlo programáticamente:

```bash
cd ~/finops-labs/lab09/reports
```

```python
# Ejecutar este script para crear el notebook
# Archivo: ~/finops-labs/lab09/scripts/crear_notebook.py

import json
from pathlib import Path

notebook = {
    "cells": [
        {
            "cell_type": "markdown",
            "metadata": {},
            "source": [
                "# 📊 Reporte Ejecutivo FinOps - TechCorp S.A.\n",
                "\n",
                "**Fecha:** Junio 2025  \n",
                "**Preparado por:** Equipo FinOps  \n",
                "**Alcance:** División Cloud & Plataforma  \n",
                "**Período analizado:** Q4 2023 + Enero 2024\n",
                "\n",
                "---"
            ]
        },
        {
            "cell_type": "code",
            "metadata": {},
            "source": [
                "# Imports y configuración\n",
                "import pandas as pd\n",
                "import plotly.express as px\n",
                "import plotly.graph_objects as go\n",
                "from plotly.subplots import make_subplots\n",
                "import yaml\n",
                "from pathlib import Path\n",
                "from IPython.display import HTML, display\n",
                "\n",
                "BASE_PATH = Path.home() / 'finops-labs'\n",
                "LAB09_OUTPUT = BASE_PATH / 'lab09' / 'output'\n",
                "\n",
                "# Cargar diagnóstico\n",
                "diag_path = LAB09_OUTPUT / 'diagnostico_finops_techcorp.yaml'\n",
                "with open(diag_path) as f:\n",
                "    diagnostico = yaml.safe_load(f)['diagnostico_finops']\n",
                "\n",
                "print(f\"Diagnóstico cargado: {diagnostico['organizacion']}\")\n",
                "print(f\"Score de madurez: {diagnostico['score_global']}/100\")"
            ],
            "outputs": [],
            "execution_count": None
        },
        {
            "cell_type": "markdown",
            "metadata": {},
            "source": [
                "## 1. Diagnóstico de Madurez\n",
                "\n",
                "### 1.1 Score Global y Gauge de Madurez"
            ]
        },
        {
            "cell_type": "code",
            "metadata": {},
            "source": [
                "# Gauge de madurez\n",
                "fig_gauge = go.Figure(go.Indicator(\n",
                "    mode='gauge+number+delta',\n",
                "    value=diagnostico['score_global'],\n",
                "    domain={'x': [0, 1], 'y': [0, 1]},\n",
                "    title={'text': 'Score de Madurez FinOps', 'font': {'size': 24}},\n",
                "    delta={'reference': 70, 'increasing': {'color': 'green'}},\n",
                "    gauge={\n",
                "        'axis': {'range': [0, 100], 'tickwidth': 1},\n",
                "        'bar': {'color': 'darkblue'},\n",
                "        'steps': [\n",
                "            {'range': [0, 33], 'color': '#ff4444', 'name': 'Crawl'},\n",
                "            {'range': [33, 66], 'color': '#ffaa00', 'name': 'Walk'},\n",
                "            {'range': [66, 100], 'color': '#44bb44', 'name': 'Run'},\n",
                "        ],\n",
                "        'threshold': {\n",
                "            'line': {'color': 'red', 'width': 4},\n",
                "            'thickness': 0.75,\n",
                "            'value': 70\n",
                "        }\n",
                "    }\n",
                "))\n",
                "fig_gauge.update_layout(height=350)\n",
                "fig_gauge.show()"
            ],
            "outputs": [],
            "execution_count": None
        },
        {
            "cell_type": "markdown",
            "metadata": {},
            "source": [
                "### 1.2 Sunburst de Costos por Equipo y Servicio"
            ]
        },
        {
            "cell_type": "code",
            "metadata": {},
            "source": [
                "# Datos simulados de costos por equipo/servicio para sunburst\n",
                "# En producción estos vendrían del data warehouse\n",
                "cost_data = pd.DataFrame({\n",
                "    'team': ['Backend']*4 + ['Frontend']*3 + ['Data']*3 + ['Platform']*3,\n",
                "    'service': ['EC2', 'RDS', 'Lambda', 'S3',\n",
                "                'CloudFront', 'S3', 'Lambda',\n",
                "                'Redshift', 'S3', 'Glue',\n",
                "                'EKS', 'EC2', 'CloudWatch'],\n",
                "    'cost_usd': [28500, 15200, 3400, 2100,\n",
                "                 8900, 4200, 1800,\n",
                "                 22000, 8500, 5600,\n",
                "                 18000, 12300, 3200]\n",
                "})\n",
                "\n",
                "fig_sunburst = px.sunburst(\n",
                "    cost_data,\n",
                "    path=['team', 'service'],\n",
                "    values='cost_usd',\n",
                "    title='Distribución de Costos por Equipo y Servicio (USD)',\n",
                "    color='cost_usd',\n",
                "    color_continuous_scale='RdYlGn_r'\n",
                ")\n",
                "fig_sunburst.update_layout(height=500)\n",
                "fig_sunburst.show()"
            ],
            "outputs": [],
            "execution_count": None
        },
        {
            "cell_type": "markdown",
            "metadata": {},
            "source": [
                "## 2. Roadmap de Implementación 30-60-90 Días"
            ]
        },
        {
            "cell_type": "code",
            "metadata": {},
            "source": [
                "# Timeline del roadmap\n",
                "roadmap_path = LAB09_OUTPUT / 'roadmap_finops.xlsx'\n",
                "roadmap_data = []\n",
                "\n",
                "for horizonte in [30, 60, 90]:\n",
                "    try:\n",
                "        df = pd.read_excel(roadmap_path, sheet_name=f'Horizonte_{horizonte}_dias')\n",
                "        for _, row in df.iterrows():\n",
                "            roadmap_data.append({\n",
                "                'Iniciativa': row.get('title', row.get('initiative_id', 'N/A')),\n",
                "                'Horizonte': f'{horizonte} días',\n",
                "                'Inicio': horizonte - 25,\n",
                "                'Fin': horizonte,\n",
                "                'Ahorro_USD': row.get('estimated_savings_usd', 0),\n",
                "                'Responsable': row.get('responsable', 'TBD')\n",
                "            })\n",
                "    except Exception as e:\n",
                "        print(f'Nota: {e}')\n",
                "\n",
                "if roadmap_data:\n",
                "    df_timeline = pd.DataFrame(roadmap_data)\n",
                "    \n",
                "    fig_timeline = px.timeline(\n",
                "        df_timeline,\n",
                "        x_start='Inicio',\n",
                "        x_end='Fin',\n",
                "        y='Iniciativa',\n",
                "        color='Horizonte',\n",
                "        title='Roadmap FinOps - Timeline de Implementación',\n",
                "        color_discrete_map={\n",
                "            '30 días': '#2ecc71',\n",
                "            '60 días': '#f39c12',\n",
                "            '90 días': '#e74c3c'\n",
                "        }\n",
                "    )\n",
                "    fig_timeline.update_layout(height=500, xaxis_title='Días')\n",
                "    fig_timeline.show()\n",
                "else:\n",
                "    # Fallback con Gantt simplificado\n",
                "    from datetime import datetime, timedelta\n",
                "    base = datetime(2025, 6, 15)\n",
                "    tasks = [\n",
                "        {'Task': 'Etiquetado obligatorio', 'Start': base, 'Finish': base+timedelta(25), 'Phase': '30 días'},\n",
                "        {'Task': 'Reserved Instances', 'Start': base+timedelta(5), 'Finish': base+timedelta(28), 'Phase': '30 días'},\n",
                "        {'Task': 'Rightsizing', 'Start': base+timedelta(10), 'Finish': base+timedelta(30), 'Phase': '30 días'},\n",
                "        {'Task': 'Savings Plans', 'Start': base+timedelta(30), 'Finish': base+timedelta(55), 'Phase': '60 días'},\n",
                "        {'Task': 'Showback', 'Start': base+timedelta(35), 'Finish': base+timedelta(60), 'Phase': '60 días'},\n",
                "        {'Task': 'Spot Instances', 'Start': base+timedelta(60), 'Finish': base+timedelta(85), 'Phase': '90 días'},\n",
                "        {'Task': 'Unit Economics', 'Start': base+timedelta(65), 'Finish': base+timedelta(90), 'Phase': '90 días'},\n",
                "    ]\n",
                "    df_gantt = pd.DataFrame(tasks)\n",
                "    fig_timeline = px.timeline(df_gantt, x_start='Start', x_end='Finish', y='Task', color='Phase',\n",
                "                              title='Roadmap FinOps - Timeline de Implementación')\n",
                "    fig_timeline.update_layout(height=400)\n",
                "    fig_timeline.show()"
            ],
            "outputs": [],
            "execution_count": None
        },
        {
            "cell_type": "markdown",
            "metadata": {},
            "source": [
                "## 3. KPIs y Metas a 90 Días"
            ]
        },
        {
            "cell_type": "code",
            "metadata": {},
            "source": [
                "# Cargar KPIs\n",
                "kpis_path = LAB09_OUTPUT / 'kpis_rituales.yaml'\n",
                "with open(kpis_path) as f:\n",
                "    kpis_data = yaml.safe_load(f)\n",
                "\n",
                "# Tabla de KPIs\n",
                "kpi_rows = []\n",
                "for kpi in kpis_data['kpis_eficiencia']:\n",
                "    kpi_rows.append({\n",
                "        'KPI': kpi['nombre'],\n",
                "        'Estado Actual': kpi['meta_actual'],\n",
                "        'Meta 90 días': kpi['meta_90_dias'],\n",
                "        'Meta Run': kpi['meta_run'],\n",
                "        'Responsable': kpi['responsable']\n",
                "    })\n",
                "\n",
                "df_kpis = pd.DataFrame(kpi_rows)\n",
                "display(HTML(df_kpis.to_html(index=False, classes='table table-striped')))\n",
                "\n",
                "# Visualización de progreso esperado\n",
                "fig_kpi = go.Figure()\n",
                "kpi_names = ['Untagged\\nSpend', 'Budget\\nVariance', 'Savings\\nRate', 'Coverage\\nRate']\n",
                "actual_values = [45, 22, 3, 0]\n",
                "target_values = [15, 12, 18, 45]\n",
                "\n",
                "fig_kpi.add_trace(go.Bar(name='Estado Actual', x=kpi_names, y=actual_values,\n",
                "                         marker_color='#e74c3c'))\n",
                "fig_kpi.add_trace(go.Bar(name='Meta 90 días', x=kpi_names, y=target_values,\n",
                "                         marker_color='#2ecc71'))\n",
                "fig_kpi.update_layout(title='KPIs: Estado Actual vs. Meta 90 Días (%)',\n",
                "                      barmode='group', height=350)\n",
                "fig_kpi.show()"
            ],
            "outputs": [],
            "execution_count": None
        },
        {
            "cell_type": "markdown",
            "metadata": {},
            "source": [
                "## 4. Resumen Ejecutivo y Próximos Pasos\n",
                "\n",
                "### Hallazgos Principales\n",
                "\n",
                "| Dimensión | Estado | Acción Prioritaria |\n",
                "|-----------|--------|--------------------|\n",
                "| Madurez Global | Crawl-Walk (41.7/100) | Formalizar gobernanza y rituales |\n",
                "| Cobertura Tags | 55% promedio | Política de enforcement automatizado |\n",
                "| Ahorro Potencial | $127,450 USD/año | Ejecutar RI + Rightsizing en 30 días |\n",
                "| Gobernanza | 2 gaps críticos | Completar RACI + políticas YAML |\n",
                "\n",
                "### Inversión Requerida vs. Retorno Esperado\n",
                "\n",
                "- **Inversión:** 1 FTE FinOps Lead + 20% tiempo Platform Team (3 meses)\n",
                "- **Retorno estimado:** $127,450 USD ahorro anualizado (ROI > 300%)\n",
                "- **Madurez objetivo 90 días:** Walk consolidado (65-70/100)\n",
                "\n",
                "### Aprobaciones Requeridas\n",
                "\n",
                "1. ✅ Presupuesto para Reserved Instances ($32,000 upfront)\n",
                "2. ✅ Asignación de FinOps Lead como rol formal\n",
                "3. ✅ Mandato ejecutivo para política de etiquetado obligatorio"
            ]
        },
        {
            "cell_type": "code",
            "metadata": {},
            "source": [
                "# Resumen final con métricas consolidadas\n",
                "print('=' * 60)\n",
                "print('  REPORTE FINOPS EJECUTIVO - TECHCORP S.A.')\n",
                "print('=' * 60)\n",
                "print(f\"  Score Madurez Actual:    {diagnostico['score_global']}/100\")\n",
                "print(f\"  Score Madurez Objetivo:  70/100 (90 días)\")\n",
                "print(f\"  Ahorro Potencial Total:  ${diagnostico['ahorro_potencial_usd']:,.2f} USD\")\n",
                "print(f\"  Cobertura Tags Actual:   {diagnostico['cobertura_tags']['promedio']}%\")\n",
                "print(f\"  Cobertura Tags Objetivo: 85%\")\n",
                "print(f\"  Recursos sin Owner:      {diagnostico['recursos_sin_owner']}\")\n",
                "print(f\"  Gaps de Gobernanza:      {len(diagnostico['gaps_gobernanza'])}\")\n",
                "print('=' * 60)\n",
                "print('\\n✅ Reporte completado exitosamente.')\n",
                "print('   Exportar como HTML: jupyter nbconvert --to html lab09_reporte_ejecutivo.ipynb')"
            ],
            "outputs": [],
            "execution_count": None
        }
    ],
    "metadata": {
        "kernelspec": {
            "display_name": "Python 3",
            "language": "python",
            "name": "python3"
        },
        "language_info": {
            "name": "python",
            "version": "3.12.1"
        }
    },
    "nbformat": 4,
    "nbformat_minor": 5
}

output_path = Path.home() / "finops-labs" / "lab09" / "reports" / "lab09_reporte_ejecutivo.ipynb"
output_path.parent.mkdir(parents=True, exist_ok=True)

with open(output_path, "w", encoding="utf-8") as f:
    json.dump(notebook, f, indent=2, ensure_ascii=False)

print(f"Notebook creado: {output_path}")
```

2. Ejecutar el script de creación del notebook:

```bash
python3 ~/finops-labs/lab09/scripts/crear_notebook.py
```

3. Ejecutar el notebook completo desde línea de comandos:

```bash
cd ~/finops-labs/lab09/reports

jupyter nbconvert --to notebook --execute \
  --ExecutePreprocessor.timeout=120 \
  lab09_reporte_ejecutivo.ipynb \
  --output lab09_reporte_ejecutivo_executed.ipynb
```

4. Exportar como HTML autocontenido:

```bash
jupyter nbconvert --to html \
  --HTMLExporter.theme=dark \
  --no-input \
  lab09_reporte_ejecutivo_executed.ipynb \
  --output ../output/finops_diagnostic_report_techcorp.html
```

> **Nota:** La opción `--no-input` oculta las celdas de código en el HTML final, mostrando solo las visualizaciones y texto markdown (formato ejecutivo).

**Salida esperada:**

```
[NbConvertApp] Converting notebook lab09_reporte_ejecutivo_executed.ipynb to html
[NbConvertApp] Writing XXXX bytes to ../output/finops_diagnostic_report_techcorp.html
```

**Verificación:**

```bash
ls -lh ~/finops-labs/lab09/output/finops_diagnostic_report_techcorp.html
```

El archivo debe existir y tener un tamaño > 500 KB (incluye plotly.js embebido).

5. (Opcional) Abrir el reporte en el navegador para verificación visual:

```bash
# Linux
xdg-open ~/finops-labs/lab09/output/finops_diagnostic_report_techcorp.html

# macOS
open ~/finops-labs/lab09/output/finops_diagnostic_report_techcorp.html
```

---

## Validación y Pruebas

Ejecutar el siguiente script de validación integral para confirmar que todos los artefactos del laboratorio se generaron correctamente:

```bash
python3 -c "
from pathlib import Path
import yaml
import openpyxl

base = Path.home() / 'finops-labs' / 'lab09'
results = []

# 1. Diagnóstico YAML
diag = base / 'output' / 'diagnostico_finops_techcorp.yaml'
if diag.exists():
    with open(diag) as f:
        data = yaml.safe_load(f)
    has_score = 'score_global' in data.get('diagnostico_finops', {})
    results.append(('Diagnóstico YAML', '✅' if has_score else '⚠️'))
else:
    results.append(('Diagnóstico YAML', '❌ No encontrado'))

# 2. Roadmap Excel
roadmap = base / 'output' / 'roadmap_finops.xlsx'
if roadmap.exists():
    wb = openpyxl.load_workbook(roadmap)
    has_sheets = all(f'Horizonte_{h}_dias' in wb.sheetnames for h in [30, 60, 90])
    results.append(('Roadmap Excel', '✅' if has_sheets else '⚠️ Hojas incompletas'))
else:
    results.append(('Roadmap Excel', '❌ No encontrado'))

# 3. KPIs y Rituales YAML
kpis = base / 'output' / 'kpis_rituales.yaml'
if kpis.exists():
    with open(kpis) as f:
        kdata = yaml.safe_load(f)
    n_kpis = len(kdata.get('kpis_eficiencia', []))
    n_rit_w = len(kdata.get('rituales_semanales', []))
    n_rit_m = len(kdata.get('rituales_mensuales', []))
    ok = n_kpis == 5 and n_rit_w == 3 and n_rit_m == 2
    results.append(('KPIs y Rituales', '✅' if ok else f'⚠️ KPIs={n_kpis}, Rit_W={n_rit_w}, Rit_M={n_rit_m}'))
else:
    results.append(('KPIs y Rituales', '❌ No encontrado'))

# 4. Reporte HTML
html = base / 'output' / 'finops_diagnostic_report_techcorp.html'
if html.exists():
    size_kb = html.stat().st_size / 1024
    results.append(('Reporte HTML', f'✅ ({size_kb:.0f} KB)'))
else:
    results.append(('Reporte HTML', '❌ No encontrado'))

# Resumen
print()
print('═' * 50)
print('  VALIDACIÓN FINAL - Lab 09-00-01')
print('═' * 50)
all_ok = True
for name, status in results:
    print(f'  {status}  {name}')
    if '❌' in status:
        all_ok = False
print('═' * 50)
print(f'  {\"🎉 LABORATORIO COMPLETADO\" if all_ok else \"⚠️ REVISAR ARTEFACTOS FALTANTES\"}')
print('═' * 50)
"
```

**Resultado esperado:**

```
══════════════════════════════════════════════════════
  VALIDACIÓN FINAL - Lab 09-00-01
══════════════════════════════════════════════════════
  ✅  Diagnóstico YAML
  ✅  Roadmap Excel
  ✅  KPIs y Rituales
  ✅  Reporte HTML (1245 KB)
══════════════════════════════════════════════════════
  🎉 LABORATORIO COMPLETADO
══════════════════════════════════════════════════════
```

---

## Solución de Problemas

### Problema 1: Error de conexión a PostgreSQL

**Síntomas:**

```
sqlalchemy.exc.OperationalError: (psycopg2.OperationalError) 
connection refused... Is the server running on host "localhost" and accepting TCP/IP connections on port 5432?
```

**Causa:** El contenedor Docker de PostgreSQL del lab07 no está en ejecución o fue detenido.

**Solución:**

```bash
# Verificar estado
cd ~/finops-labs/lab07
docker compose ps

# Si está detenido, reiniciar
docker compose up -d

# Esperar 5 segundos y verificar conectividad
sleep 5
docker compose exec postgres pg_isready -U finops
```

Si el contenedor no existe (fue eliminado), recrearlo:

```bash
docker compose up -d --force-recreate
# Esperar a que PostgreSQL esté listo
sleep 10
# Verificar que el esquema staging existe
docker compose exec postgres psql -U finops -d finops_dw -c "\dt staging.*"
```

### Problema 2: nbconvert falla al exportar HTML con error de plotly

**Síntomas:**

```
ValueError: Mime type rendering requires nbformat>=4.2.0
```

o bien las gráficas plotly aparecen como espacios en blanco en el HTML exportado.

**Causa:** nbconvert no puede renderizar widgets plotly en modo offline sin la librería `plotly` disponible en el kernel, o falta el paquete `kaleido` para exportación estática.

**Solución:**

```bash
# Instalar dependencias de renderizado
pip install kaleido==0.2.1 nbformat>=5.9.0

# Alternativa: usar renderer de plotly compatible con nbconvert
# Agregar al inicio del notebook:
# import plotly.io as pio
# pio.renderers.default = "notebook_connected"

# Re-ejecutar con la opción de embed de HTML
jupyter nbconvert --to html \
  --HTMLExporter.theme=light \
  --template classic \
  lab09_reporte_ejecutivo_executed.ipynb \
  --output ../output/finops_diagnostic_report_techcorp.html
```

Si persiste, ejecutar el notebook interactivamente en JupyterLab y exportar desde la interfaz:

```bash
jupyter lab --port=8888 --notebook-dir=~/finops-labs/lab09/reports/
# Luego: File → Export Notebook As → HTML
```

---

## Limpieza

Este es el laboratorio final del curso. Los artefactos generados constituyen el entregable de cierre. Sin embargo, si deseas liberar recursos:

```bash
# Detener servicios Docker (solo si ya no los necesitas)
cd ~/finops-labs/lab07
docker compose down

# Opcional: eliminar volúmenes de datos (IRREVERSIBLE)
# docker compose down -v

# Los artefactos del curso permanecen en:
# ~/finops-labs/lab09/output/
```

**Artefactos finales generados:**

| Archivo | Descripción |
|---------|-------------|
| `diagnostico_finops_techcorp.yaml` | Diagnóstico automatizado con score de madurez |
| `roadmap_finops.xlsx` | Roadmap 30-60-90 días con 4 hojas |
| `kpis_rituales.yaml` | 5 KPIs + 5 rituales con responsables |
| `finops_diagnostic_report_techcorp.html` | Reporte ejecutivo HTML autocontenido |

---

## Resumen

En este laboratorio integrador final se consolidaron todos los conocimientos y artefactos del curso FinOps:

1. **Diagnóstico automatizado** — Se ejecutó un script que consulta el data warehouse, evalúa artefactos de gobernanza y genera un score de madurez cuantitativo (modelo Crawl-Walk-Run aplicado a 4 capacidades).

2. **Roadmap 30-60-90** — Se clasificaron las iniciativas del backlog en horizontes temporales según prioridad, esfuerzo y dependencias, con responsables del RACI y KPIs de éxito por iniciativa.

3. **KPIs y Rituales** — Se definieron 5 KPIs operativos (Unit Economics, % Untagged Spend, Budget Variance, Savings Rate, Coverage Rate) y 5 rituales (3 semanales + 2 mensuales) con ownership claro.

4. **Reporte ejecutivo** — Se generó un reporte HTML autocontenido con visualizaciones plotly (gauge de madurez, sunburst de costos, timeline de roadmap) listo para presentación a stakeholders.

### Recursos Adicionales

- [FinOps Foundation — Framework completo](https://www.finops.org/framework/)
- [FinOps Foundation — Modelo de Madurez](https://www.finops.org/framework/maturity-model/)
- [Plotly — Documentación de gráficos](https://plotly.com/python/)
- [Jupyter nbconvert — Documentación](https://nbconvert.readthedocs.io/)

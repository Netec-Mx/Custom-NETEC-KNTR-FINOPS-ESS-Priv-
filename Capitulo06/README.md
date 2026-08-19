# Demostración 6. Definición de roles, RACI y políticas mínimas

## Metadatos

| Campo | Valor |
|-------|-------|
| **Duración** | 25 minutos |
| **Complejidad** | Media |
| **Nivel Bloom** | Crear |

## Descripción General

En este laboratorio construirás los artefactos de gobernanza FinOps para la organización ficticia NovaTech SRL: definirás roles con responsabilidades específicas, crearás una matriz RACI para los procesos FinOps principales, diseñarás políticas operativas mínimas en formato YAML y evaluarás la madurez FinOps mediante un modelo de scoring. Todos los artefactos se consolidarán en un archivo Excel multi-hoja (`finops_governance_v1.xlsx`) que servirá como insumo para laboratorios posteriores.

## Objetivos de Aprendizaje

- [ ] Definir los 6 roles FinOps clave con responsabilidades específicas adaptadas a NovaTech SRL
- [ ] Construir una matriz RACI completa para 5 procesos FinOps principales
- [ ] Diseñar 4 políticas FinOps operativas mínimas en formato YAML
- [ ] Evaluar la madurez FinOps de la organización usando un scoring de 20 preguntas en 5 dimensiones
- [ ] Generar el archivo `finops_governance_v1.xlsx` con 4 hojas consolidadas

## Prerrequisitos

### Conocimiento previo

- Comprensión de los 6 roles FinOps (Finanzas, Ingeniería, Producto, Operaciones, Compras, Liderazgo)
- Familiaridad con matrices RACI (Responsible, Accountable, Consulted, Informed)
- Lab 05-00-01 completado (`finops_backlog_v1.xlsx` disponible)

### Acceso y archivos requeridos

| Recurso | Ubicación |
|---------|-----------|
| Backlog FinOps | `~/finops-course/data/processed/finops_backlog_v1.xlsx` |
| Perfil organizacional | `~/finops-course/data/raw/org_profile.json` (se creará en Paso 1) |
| JupyterLab | Puerto 8888 |

## Entorno del Laboratorio

### Software requerido

| Herramienta | Versión |
|-------------|---------|
| Python | 3.12.1 |
| pandas | 2.2.1 |
| openpyxl | 3.1.2 |
| pyyaml | 6.0.1 |
| JupyterLab | 4.1.5 |

### Configuración inicial

```bash
# Crear estructura de directorios
mkdir -p ~/finops-course/lab06/{data,output,policies}
mkdir -p ~/finops-course/notebooks

# Instalar pyyaml si no está disponible
pip install pyyaml==6.0.1

# Verificar instalación
python -c "import pandas, openpyxl, yaml; print('Dependencias OK')"
```

```bash
# Iniciar JupyterLab
jupyter lab --port=8888 --notebook-dir=~/finops-course/notebooks/
```

---

## Paso a Paso

### Paso 1: Crear el perfil organizacional y cargar datos base

**Objetivo:** Generar el archivo `org_profile.json` que describe la estructura de NovaTech SRL y cargar el backlog existente.

**Instrucciones:**

1. Crea un nuevo notebook en JupyterLab llamado `lab06_governance.ipynb`.

2. En la primera celda, crea el perfil organizacional:

```python
import json
import pandas as pd
import yaml
from pathlib import Path

# Definir rutas
LAB_DIR = Path.home() / "finops-course" / "lab06"
DATA_DIR = Path.home() / "finops-course" / "data"
OUTPUT_DIR = LAB_DIR / "output"
POLICIES_DIR = LAB_DIR / "policies"

# Perfil organizacional de NovaTech SRL
org_profile = {
    "company": "NovaTech SRL",
    "industry": "SaaS B2B",
    "cloud_spend_monthly_usd": 185000,
    "cloud_providers": ["AWS", "Azure", "GCP"],
    "products": ["PayCore", "AnalyticsHub", "DevPortal"],
    "environments": ["prod", "staging", "dev"],
    "teams": {
        "Backend": {"size": 12, "lead": "Carlos Méndez", "type": "engineering"},
        "Frontend": {"size": 8, "lead": "Laura Ríos", "type": "engineering"},
        "Data": {"size": 6, "lead": "Miguel Torres", "type": "engineering"},
        "Platform": {"size": 5, "lead": "Ana Vega", "type": "operations"},
        "Finance": {"size": 3, "lead": "Roberto Díaz", "type": "finance"},
        "Product": {"size": 4, "lead": "Sofía Castro", "type": "product"},
        "Procurement": {"size": 2, "lead": "Diego Herrera", "type": "procurement"},
        "Leadership": {"size": 3, "lead": "Patricia Morales (CTO)", "type": "leadership"}
    },
    "finops_maturity_current": "Crawl",
    "tagging_compliance": 0.45,
    "budget_alerts_configured": False,
    "reservation_coverage": 0.22
}

# Guardar perfil
profile_path = DATA_DIR / "raw" / "org_profile.json"
profile_path.parent.mkdir(parents=True, exist_ok=True)
with open(profile_path, "w", encoding="utf-8") as f:
    json.dump(org_profile, f, indent=2, ensure_ascii=False)

print(f"Perfil guardado en: {profile_path}")
print(f"Empresa: {org_profile['company']}")
print(f"Equipos: {len(org_profile['teams'])}")
print(f"Gasto mensual: ${org_profile['cloud_spend_monthly_usd']:,.0f} USD")
```

3. En la siguiente celda, carga el backlog del lab anterior (o crea uno de referencia si no existe):

```python
# Intentar cargar backlog existente
backlog_path = DATA_DIR / "processed" / "finops_backlog_v1.xlsx"

if backlog_path.exists():
    backlog_df = pd.read_excel(backlog_path)
    print(f"Backlog cargado: {len(backlog_df)} iniciativas")
else:
    # Crear backlog de referencia si no existe
    backlog_df = pd.DataFrame({
        "id": [f"FIN-{i:03d}" for i in range(1, 11)],
        "initiative": [
            "Implementar política de etiquetado obligatorio",
            "Rightsizing instancias EC2 sobredimensionadas",
            "Comprar Savings Plans para compute estable",
            "Apagado automático de entornos dev/staging",
            "Eliminar recursos huérfanos (EBS, EIP, snapshots)",
            "Migrar workloads a instancias Graviton",
            "Configurar alertas de presupuesto por equipo",
            "Implementar chargeback por centro de costo",
            "Optimizar almacenamiento S3 con lifecycle policies",
            "Negociar EDP con AWS"
        ],
        "category": ["Governance", "Optimization", "Rate", "Optimization",
                     "Waste", "Optimization", "Governance", "Allocation",
                     "Optimization", "Rate"],
        "estimated_savings_monthly_usd": [0, 12000, 18000, 8500, 4200,
                                          6800, 0, 0, 3100, 15000],
        "effort": ["Medium", "Low", "Medium", "Low", "Low",
                   "High", "Low", "High", "Medium", "High"],
        "priority": ["P1", "P1", "P1", "P2", "P2", "P2", "P1", "P3", "P2", "P3"]
    })
    backlog_df.to_excel(backlog_path, index=False)
    print(f"Backlog de referencia creado: {len(backlog_df)} iniciativas")

backlog_df.head()
```

**Salida esperada:**

```
Perfil guardado en: /home/user/finops-course/data/raw/org_profile.json
Empresa: NovaTech SRL
Equipos: 8
Gasto mensual: $185,000 USD
Backlog cargado: 10 iniciativas
```

**Verificación:** El archivo `org_profile.json` existe y el DataFrame `backlog_df` contiene al menos 10 filas.

---

### Paso 2: Definir roles FinOps con responsabilidades específicas

**Objetivo:** Crear un DataFrame estructurado con los 6 roles FinOps, sus responsabilidades y los equipos de NovaTech asignados a cada rol.

**Instrucciones:**

1. Define los roles y sus responsabilidades en una nueva celda:

```python
# Definición de roles FinOps para NovaTech SRL
roles_data = [
    {
        "role": "Finanzas",
        "team_novatech": "Finance",
        "lead": "Roberto Díaz",
        "responsibilities": (
            "1) Mantener modelo de asignación de costos (showback/chargeback)\n"
            "2) Reconciliar facturas cloud con sistemas contables\n"
            "3) Calcular costo amortizado de reservas y compromisos\n"
            "4) Emitir reportes ejecutivos mensuales de variación presupuestaria\n"
            "5) Definir centros de costo y reglas de distribución"
        ),
        "kpi_principal": "Variación presupuestaria < 10%",
        "frecuencia_participacion": "Semanal"
    },
    {
        "role": "Ingeniería",
        "team_novatech": "Backend, Frontend, Data",
        "lead": "Carlos Méndez (coord.)",
        "responsibilities": (
            "1) Etiquetar recursos desde el aprovisionamiento (IaC)\n"
            "2) Implementar rightsizing y autoscaling\n"
            "3) Revisar recomendaciones de AWS Advisor/Azure Advisor\n"
            "4) Participar en revisiones de arquitectura con perspectiva de costo\n"
            "5) Ejecutar items del backlog de optimización asignados"
        ),
        "kpi_principal": "Cobertura de etiquetado > 95%",
        "frecuencia_participacion": "Diaria"
    },
    {
        "role": "Producto",
        "team_novatech": "Product",
        "lead": "Sofía Castro",
        "responsibilities": (
            "1) Definir unit economics (costo/usuario, costo/transacción)\n"
            "2) Priorizar optimización vs. nuevas funcionalidades\n"
            "3) Comunicar impacto de decisiones de arquitectura en costo\n"
            "4) Validar ROI de iniciativas de optimización\n"
            "5) Alinear roadmap técnico con objetivos de eficiencia"
        ),
        "kpi_principal": "Costo por usuario activo mensual",
        "frecuencia_participacion": "Quincenal"
    },
    {
        "role": "Operaciones",
        "team_novatech": "Platform",
        "lead": "Ana Vega",
        "responsibilities": (
            "1) Configurar alertas de presupuesto y anomalías\n"
            "2) Implementar apagado automático en entornos no-prod\n"
            "3) Mantener inventario de recursos y detectar huérfanos\n"
            "4) Ejecutar flujos de remediación automatizados\n"
            "5) Gestionar herramientas de monitoreo de costos"
        ),
        "kpi_principal": "Tiempo de detección de anomalías < 4h",
        "frecuencia_participacion": "Diaria"
    },
    {
        "role": "Compras",
        "team_novatech": "Procurement",
        "lead": "Diego Herrera",
        "responsibilities": (
            "1) Evaluar y adquirir reservas (RI, SP, CUD)\n"
            "2) Negociar descuentos por compromiso (EDP, PPA)\n"
            "3) Gestionar renovaciones y vencimientos contractuales\n"
            "4) Coordinar con Finanzas la contabilización de compromisos\n"
            "5) Evaluar marketplace y opciones de licenciamiento"
        ),
        "kpi_principal": "Cobertura de reservas > 70%",
        "frecuencia_participacion": "Mensual"
    },
    {
        "role": "Liderazgo",
        "team_novatech": "Leadership",
        "lead": "Patricia Morales (CTO)",
        "responsibilities": (
            "1) Patrocinar la práctica FinOps y comunicar importancia estratégica\n"
            "2) Establecer objetivos de eficiencia alineados al negocio\n"
            "3) Tomar decisiones de inversión informadas por datos de costo\n"
            "4) Resolver conflictos entre equipos por prioridades\n"
            "5) Aprobar compromisos financieros > $5,000 USD/mes"
        ),
        "kpi_principal": "Cloud cost as % of revenue < 15%",
        "frecuencia_participacion": "Mensual"
    }
]

roles_df = pd.DataFrame(roles_data)
print(f"Roles definidos: {len(roles_df)}")
print(roles_df[["role", "team_novatech", "kpi_principal"]].to_string(index=False))
```

**Salida esperada:**

```
Roles definidos: 6
        role           team_novatech                       kpi_principal
    Finanzas                 Finance     Variación presupuestaria < 10%
  Ingeniería  Backend, Frontend, Data   Cobertura de etiquetado > 95%
    Producto                 Product  Costo por usuario activo mensual
 Operaciones                Platform  Tiempo de detección de anomalías < 4h
     Compras             Procurement     Cobertura de reservas > 70%
   Liderazgo              Leadership  Cloud cost as % of revenue < 15%
```

**Verificación:** El DataFrame contiene exactamente 6 filas, una por cada rol FinOps.

---

### Paso 3: Construir la matriz RACI para procesos FinOps

**Objetivo:** Crear una matriz RACI con 5 procesos FinOps como filas y 6 roles como columnas, asignando valores R (Responsible), A (Accountable), C (Consulted) e I (Informed).

**Instrucciones:**

1. Define la matriz RACI:

```python
# Matriz RACI - Procesos FinOps principales
# R = Responsible (ejecuta), A = Accountable (aprueba/rinde cuentas)
# C = Consulted (se consulta antes), I = Informed (se notifica después)

raci_data = {
    "Proceso FinOps": [
        "Asignación de costos (tagging + allocation)",
        "Reporte mensual de costos",
        "Aprobación de reservas/compromisos",
        "Gestión de anomalías de costo",
        "Revisión de backlog de optimización"
    ],
    "Finanzas": ["A", "R", "C", "I", "C"],
    "Ingeniería": ["R", "C", "C", "R", "R"],
    "Producto": ["C", "I", "I", "I", "A"],
    "Operaciones": ["R", "R", "I", "R", "C"],
    "Compras": ["I", "I", "R", "I", "I"],
    "Liderazgo": ["I", "A", "A", "I", "I"]
}

raci_df = pd.DataFrame(raci_data)
raci_df = raci_df.set_index("Proceso FinOps")

print("=" * 80)
print("MATRIZ RACI - NovaTech SRL - Procesos FinOps")
print("=" * 80)
print(raci_df.to_string())
print("\nLeyenda: R=Responsible | A=Accountable | C=Consulted | I=Informed")
```

2. Valida la integridad de la matriz (cada proceso debe tener exactamente 1 Accountable):

```python
# Validación: cada proceso debe tener exactamente 1 'A'
print("\n--- Validación de matriz RACI ---")
valid = True
for proceso in raci_df.index:
    row = raci_df.loc[proceso]
    a_count = (row == "A").sum()
    r_count = (row == "R").sum()
    if a_count != 1:
        print(f"⚠️  '{proceso}': tiene {a_count} Accountable (debe ser 1)")
        valid = False
    if r_count < 1:
        print(f"⚠️  '{proceso}': no tiene Responsible asignado")
        valid = False

if valid:
    print("✅ Matriz RACI válida: cada proceso tiene 1 Accountable y al menos 1 Responsible")
```

3. Mapea las iniciativas del backlog a roles responsables:

```python
# Mapear iniciativas del backlog al RACI
role_mapping = {
    "Governance": "Operaciones",
    "Optimization": "Ingeniería",
    "Rate": "Compras",
    "Waste": "Operaciones",
    "Allocation": "Finanzas"
}

backlog_df["responsible_role"] = backlog_df["category"].map(role_mapping)
print("\nBacklog con roles asignados:")
print(backlog_df[["id", "initiative", "category", "responsible_role"]].to_string(index=False))
```

**Salida esperada:**

```
================================================================================
MATRIZ RACI - NovaTech SRL - Procesos FinOps
================================================================================
                                          Finanzas Ingeniería Producto Operaciones Compras Liderazgo
Asignación de costos (tagging + allocation)       A          R        C           R       I         I
Reporte mensual de costos                         R          C        I           R       I         A
Aprobación de reservas/compromisos                C          C        I           I       R         A
Gestión de anomalías de costo                     I          R        I           R       I         I
Revisión de backlog de optimización               C          R        A           C       I         I

Leyenda: R=Responsible | A=Accountable | C=Consulted | I=Informed

--- Validación de matriz RACI ---
✅ Matriz RACI válida: cada proceso tiene 1 Accountable y al menos 1 Responsible
```

**Verificación:** La validación muestra el mensaje de éxito sin warnings.

---

### Paso 4: Diseñar políticas FinOps en formato YAML

**Objetivo:** Crear 4 archivos YAML con políticas operativas mínimas: etiquetado, presupuestos, excepciones y aprobaciones.

**Instrucciones:**

1. Define y guarda las 4 políticas:

```python
# Política 1: Etiquetado obligatorio
policy_tagging = {
    "policy_name": "Etiquetado Obligatorio de Recursos Cloud",
    "policy_id": "POL-TAG-001",
    "version": "1.0",
    "effective_date": "2024-02-01",
    "owner": "Ana Vega (Platform)",
    "scope": "Todos los recursos en AWS, Azure y GCP",
    "mandatory_tags": {
        "env": {
            "description": "Ambiente de despliegue",
            "allowed_values": ["prod", "staging", "dev"],
            "enforcement": "deny_deploy_if_missing"
        },
        "team": {
            "description": "Equipo propietario del recurso",
            "allowed_values": ["backend", "frontend", "data", "platform"],
            "enforcement": "deny_deploy_if_missing"
        },
        "project": {
            "description": "Producto o proyecto asociado",
            "allowed_values": ["paycore", "analyticshub", "devportal", "shared"],
            "enforcement": "deny_deploy_if_missing"
        },
        "cost-center": {
            "description": "Centro de costo contable",
            "allowed_values": ["CC-ENG-001", "CC-ENG-002", "CC-DATA-001", "CC-PLAT-001"],
            "enforcement": "alert_if_missing"
        }
    },
    "compliance_target": 0.95,
    "review_frequency": "monthly",
    "non_compliance_action": "Notificación al lead del equipo + escalación a 72h"
}

# Política 2: Presupuestos y alertas
policy_budgets = {
    "policy_name": "Presupuestos y Alertas de Costo",
    "policy_id": "POL-BUD-001",
    "version": "1.0",
    "effective_date": "2024-02-01",
    "owner": "Roberto Díaz (Finance)",
    "scope": "Todas las cuentas cloud de NovaTech SRL",
    "budget_rules": {
        "alert_thresholds": [
            {"level": "warning", "percentage": 80, "notify": ["team_lead", "finops_team"]},
            {"level": "critical", "percentage": 100, "notify": ["team_lead", "finops_team", "vp_engineering"]},
            {"level": "breach", "percentage": 120, "notify": ["cto", "cfo", "all_stakeholders"]}
        ],
        "budget_granularity": ["account", "team", "project"],
        "forecast_method": "linear_regression_90days",
        "review_cadence": "weekly"
    },
    "anomaly_detection": {
        "enabled": True,
        "sensitivity": "medium",
        "min_impact_usd": 500,
        "notification_channel": "slack_finops_alerts"
    }
}

# Política 3: Excepciones
policy_exceptions = {
    "policy_name": "Gestión de Excepciones FinOps",
    "policy_id": "POL-EXC-001",
    "version": "1.0",
    "effective_date": "2024-02-01",
    "owner": "Patricia Morales (CTO)",
    "scope": "Excepciones a políticas de etiquetado, presupuesto o apagado automático",
    "exception_flow": {
        "request_channel": "Ticket en sistema de gestión (Jira/ServiceNow)",
        "required_fields": ["justificación_negocio", "impacto_estimado_usd", "duración_excepción", "plan_remediación"],
        "approval_sla_hours": 48,
        "approvers": {
            "impact_below_5000": "Team Lead + FinOps Analyst",
            "impact_above_5000": "VP Engineering + CFO"
        },
        "max_exception_duration_days": 90,
        "renewal_required": True
    }
}

# Política 4: Aprobaciones de gasto
policy_approvals = {
    "policy_name": "Aprobaciones de Gasto Cloud",
    "policy_id": "POL-APR-001",
    "version": "1.0",
    "effective_date": "2024-02-01",
    "owner": "Patricia Morales (CTO)",
    "scope": "Nuevos compromisos, reservas y gastos recurrentes",
    "approval_matrix": {
        "thresholds": [
            {"range": "$0 - $1,000/mes", "approver": "Team Lead", "sla_hours": 24},
            {"range": "$1,001 - $5,000/mes", "approver": "Engineering Manager", "sla_hours": 48},
            {"range": "$5,001 - $20,000/mes", "approver": "VP Engineering", "sla_hours": 72},
            {"range": "> $20,000/mes", "approver": "CTO + CFO", "sla_hours": 120}
        ],
        "reservation_purchases": {
            "any_amount": "VP Engineering + Finance Lead",
            "term_1year_plus": "CTO + CFO"
        }
    },
    "documentation_required": [
        "Business justification",
        "Cost-benefit analysis",
        "Alternative options evaluated",
        "Exit strategy if commitment"
    ]
}

# Guardar todas las políticas como YAML
policies = {
    "policy_tagging.yaml": policy_tagging,
    "policy_budgets.yaml": policy_budgets,
    "policy_exceptions.yaml": policy_exceptions,
    "policy_approvals.yaml": policy_approvals
}

for filename, policy in policies.items():
    filepath = POLICIES_DIR / filename
    with open(filepath, "w", encoding="utf-8") as f:
        yaml.dump(policy, f, default_flow_style=False, allow_unicode=True, sort_keys=False)
    print(f"✅ {filename} guardado en {filepath}")

print(f"\nTotal políticas creadas: {len(policies)}")
```

2. Crea un DataFrame resumen de políticas para el Excel final:

```python
# Resumen de políticas para hoja Excel
policies_summary = pd.DataFrame([
    {
        "policy_id": "POL-TAG-001",
        "nombre": "Etiquetado Obligatorio",
        "owner": "Ana Vega (Platform)",
        "tags_obligatorios": "env, team, project, cost-center",
        "enforcement": "Deny deploy si falta tag",
        "compliance_target": "95%"
    },
    {
        "policy_id": "POL-BUD-001",
        "nombre": "Presupuestos y Alertas",
        "owner": "Roberto Díaz (Finance)",
        "alertas": "80% warning, 100% critical, 120% breach",
        "granularidad": "Account, Team, Project",
        "review": "Semanal"
    },
    {
        "policy_id": "POL-EXC-001",
        "nombre": "Gestión de Excepciones",
        "owner": "Patricia Morales (CTO)",
        "sla_aprobacion": "48 horas",
        "duracion_maxima": "90 días",
        "renovacion": "Requerida"
    },
    {
        "policy_id": "POL-APR-001",
        "nombre": "Aprobaciones de Gasto",
        "owner": "Patricia Morales (CTO)",
        "umbral_vp": "> $5,000 USD/mes",
        "umbral_cto_cfo": "> $20,000 USD/mes",
        "reservas": "VP Eng + Finance Lead"
    }
])

print("Resumen de políticas:")
print(policies_summary[["policy_id", "nombre", "owner"]].to_string(index=False))
```

**Salida esperada:**

```
✅ policy_tagging.yaml guardado en /home/user/finops-course/lab06/policies/policy_tagging.yaml
✅ policy_budgets.yaml guardado en /home/user/finops-course/lab06/policies/policy_budgets.yaml
✅ policy_exceptions.yaml guardado en /home/user/finops-course/lab06/policies/policy_exceptions.yaml
✅ policy_approvals.yaml guardado en /home/user/finops-course/lab06/policies/policy_approvals.yaml

Total políticas creadas: 4
```

**Verificación:** Los 4 archivos YAML existen en `~/finops-course/lab06/policies/` y son parseables.

---

### Paso 5: Implementar scoring de madurez FinOps

**Objetivo:** Evaluar la madurez FinOps de NovaTech SRL usando 20 preguntas binarias agrupadas en 5 dimensiones, clasificando el resultado en Crawl/Walk/Run.

**Instrucciones:**

1. Define el modelo de scoring:

```python
# Modelo de madurez FinOps - 20 preguntas, 5 dimensiones (4 preguntas cada una)
maturity_assessment = {
    "Visibilidad": [
        {"question": "¿Existe un dashboard centralizado de costos cloud?", "answer": True},
        {"question": "¿Los costos se pueden desglosar por equipo/producto?", "answer": False},
        {"question": "¿Se tiene visibilidad de costos en tiempo real (< 24h delay)?", "answer": False},
        {"question": "¿Todos los stakeholders tienen acceso a reportes de costos?", "answer": True}
    ],
    "Asignación": [
        {"question": "¿Existe una estrategia de etiquetado documentada?", "answer": True},
        {"question": "¿La cobertura de etiquetado supera el 80%?", "answer": False},
        {"question": "¿Se implementa showback o chargeback a los equipos?", "answer": False},
        {"question": "¿Los costos compartidos se distribuyen con reglas claras?", "answer": False}
    ],
    "Optimización": [
        {"question": "¿Se revisan recomendaciones de rightsizing regularmente?", "answer": True},
        {"question": "¿Existe un proceso de eliminación de recursos huérfanos?", "answer": False},
        {"question": "¿Se utilizan reservas o Savings Plans?", "answer": True},
        {"question": "¿Hay automatización de apagado en entornos no-prod?", "answer": False}
    ],
    "Gobernanza": [
        {"question": "¿Existen políticas de gasto documentadas?", "answer": False},
        {"question": "¿Hay un proceso de aprobación para gastos significativos?", "answer": True},
        {"question": "¿Se realizan revisiones periódicas de costos (al menos mensual)?", "answer": True},
        {"question": "¿Existe una matriz RACI para procesos FinOps?", "answer": False}
    ],
    "Cultura": [
        {"question": "¿El liderazgo ejecutivo patrocina activamente FinOps?", "answer": True},
        {"question": "¿Los ingenieros consideran el costo en decisiones de diseño?", "answer": False},
        {"question": "¿Hay incentivos o reconocimiento por optimización de costos?", "answer": False},
        {"question": "¿Se comparten métricas de costo en reuniones de equipo?", "answer": False}
    ]
}

# Calcular scores
maturity_results = []
total_yes = 0
total_questions = 0

for dimension, questions in maturity_assessment.items():
    yes_count = sum(1 for q in questions if q["answer"])
    total = len(questions)
    score_pct = (yes_count / total) * 100
    total_yes += yes_count
    total_questions += total
    
    maturity_results.append({
        "dimension": dimension,
        "questions_total": total,
        "answers_yes": yes_count,
        "score_pct": score_pct
    })

# Score global
global_score = (total_yes / total_questions) * 100

# Clasificación
if global_score < 40:
    maturity_level = "Crawl"
    description = "Fase inicial: visibilidad básica, procesos manuales, poca colaboración"
elif global_score < 70:
    maturity_level = "Walk"
    description = "Fase intermedia: procesos definidos, automatización parcial, roles asignados"
else:
    maturity_level = "Run"
    description = "Fase avanzada: automatización completa, cultura de costos, optimización continua"

maturity_df = pd.DataFrame(maturity_results)

print("=" * 70)
print("EVALUACIÓN DE MADUREZ FINOPS - NovaTech SRL")
print("=" * 70)
print(maturity_df.to_string(index=False))
print(f"\n{'─' * 70}")
print(f"SCORE GLOBAL: {total_yes}/{total_questions} = {global_score:.0f}%")
print(f"NIVEL DE MADUREZ: {maturity_level}")
print(f"DESCRIPCIÓN: {description}")
print(f"{'─' * 70}")
```

2. Genera un DataFrame detallado con todas las preguntas para la hoja Excel:

```python
# DataFrame detallado para Excel
maturity_detail = []
for dimension, questions in maturity_assessment.items():
    for i, q in enumerate(questions, 1):
        maturity_detail.append({
            "dimension": dimension,
            "question_num": i,
            "question": q["question"],
            "answer": "Sí" if q["answer"] else "No",
            "score": 1 if q["answer"] else 0
        })

maturity_detail_df = pd.DataFrame(maturity_detail)

# Agregar fila resumen
summary_row = pd.DataFrame([{
    "dimension": "TOTAL",
    "question_num": "",
    "question": f"Score Global: {global_score:.0f}% → Nivel: {maturity_level}",
    "answer": "",
    "score": total_yes
}])

maturity_export_df = pd.concat([maturity_detail_df, summary_row], ignore_index=True)
print(f"\nRegistros para exportación: {len(maturity_export_df)}")
```

**Salida esperada:**

```
======================================================================
EVALUACIÓN DE MADUREZ FINOPS - NovaTech SRL
======================================================================
   dimension  questions_total  answers_yes  score_pct
 Visibilidad                4            2       50.0
  Asignación                4            1       25.0
Optimización                4            2       50.0
  Gobernanza                4            2       50.0
     Cultura                4            1       25.0

──────────────────────────────────────────────────────────────────────
SCORE GLOBAL: 8/20 = 40%
NIVEL DE MADUREZ: Walk
DESCRIPCIÓN: Fase intermedia: procesos definidos, automatización parcial, roles asignados
──────────────────────────────────────────────────────────────────────
```

**Verificación:** El score global es 40% (8/20) y el nivel se clasifica como "Walk".

---

### Paso 6: Generar el archivo Excel consolidado de gobernanza

**Objetivo:** Exportar todos los artefactos en un único archivo Excel con 4 hojas: RACI, Roles, Políticas y Madurez.

**Instrucciones:**

1. Genera el archivo Excel multi-hoja:

```python
# Generar archivo consolidado de gobernanza
output_file = OUTPUT_DIR / "finops_governance_v1.xlsx"
OUTPUT_DIR.mkdir(parents=True, exist_ok=True)

with pd.ExcelWriter(output_file, engine="openpyxl") as writer:
    # Hoja 1: RACI
    raci_df.to_excel(writer, sheet_name="RACI", index=True)
    
    # Hoja 2: Roles
    roles_df.to_excel(writer, sheet_name="Roles", index=False)
    
    # Hoja 3: Políticas
    policies_summary.to_excel(writer, sheet_name="Políticas", index=False)
    
    # Hoja 4: Madurez
    maturity_export_df.to_excel(writer, sheet_name="Madurez", index=False)

print(f"✅ Archivo generado: {output_file}")
print(f"   Tamaño: {output_file.stat().st_size / 1024:.1f} KB")
print(f"   Hojas: RACI, Roles, Políticas, Madurez")

# Copiar también a directorio de datos procesados para continuidad
copy_path = DATA_DIR / "processed" / "finops_governance_v1.xlsx"
import shutil
shutil.copy2(output_file, copy_path)
print(f"   Copia en: {copy_path}")
```

2. Verifica la integridad del archivo generado:

```python
# Verificación final - leer el archivo y confirmar contenido
print("\n" + "=" * 70)
print("VERIFICACIÓN FINAL - finops_governance_v1.xlsx")
print("=" * 70)

verification = pd.ExcelFile(output_file)
for sheet in verification.sheet_names:
    df_check = pd.read_excel(output_file, sheet_name=sheet)
    print(f"\n📋 Hoja '{sheet}': {df_check.shape[0]} filas × {df_check.shape[1]} columnas")
    print(f"   Columnas: {list(df_check.columns)[:5]}{'...' if len(df_check.columns) > 5 else ''}")

print(f"\n{'═' * 70}")
print("✅ LABORATORIO COMPLETADO EXITOSAMENTE")
print(f"{'═' * 70}")
print(f"\nArtefactos generados:")
print(f"  1. {output_file}")
print(f"  2. {POLICIES_DIR}/policy_tagging.yaml")
print(f"  3. {POLICIES_DIR}/policy_budgets.yaml")
print(f"  4. {POLICIES_DIR}/policy_exceptions.yaml")
print(f"  5. {POLICIES_DIR}/policy_approvals.yaml")
print(f"  6. {profile_path}")
```

**Salida esperada:**

```
✅ Archivo generado: /home/user/finops-course/lab06/output/finops_governance_v1.xlsx
   Tamaño: 12.3 KB
   Hojas: RACI, Roles, Políticas, Madurez
   Copia en: /home/user/finops-course/data/processed/finops_governance_v1.xlsx

======================================================================
VERIFICACIÓN FINAL - finops_governance_v1.xlsx
======================================================================

📋 Hoja 'RACI': 5 filas × 6 columnas
   Columnas: ['Finanzas', 'Ingeniería', 'Producto', 'Operaciones', 'Compras']...

📋 Hoja 'Roles': 6 filas × 6 columnas
   Columnas: ['role', 'team_novatech', 'lead', 'responsibilities', 'kpi_principal']...

📋 Hoja 'Políticas': 4 filas × 6 columnas
   Columnas: ['policy_id', 'nombre', 'owner', 'tags_obligatorios', 'enforcement']...

📋 Hoja 'Madurez': 21 filas × 5 columnas
   Columnas: ['dimension', 'question_num', 'question', 'answer', 'score']...

══════════════════════════════════════════════════════════════════════════
✅ LABORATORIO COMPLETADO EXITOSAMENTE
══════════════════════════════════════════════════════════════════════════
```

**Verificación:** El archivo tiene 4 hojas con las dimensiones correctas (RACI: 5×6, Roles: 6×6, Políticas: 4×6, Madurez: 21×5).

---

## Validación y Pruebas

Ejecuta las siguientes verificaciones para confirmar que todos los entregables son correctos:

```python
# Script de validación completo
import os

print("🔍 VALIDACIÓN DE ENTREGABLES\n")

checks = []

# Check 1: Archivo Excel existe y tiene 4 hojas
excel_path = OUTPUT_DIR / "finops_governance_v1.xlsx"
if excel_path.exists():
    xls = pd.ExcelFile(excel_path)
    if len(xls.sheet_names) == 4:
        checks.append(("Excel con 4 hojas", "✅ PASS"))
    else:
        checks.append(("Excel con 4 hojas", f"❌ FAIL ({len(xls.sheet_names)} hojas)"))
else:
    checks.append(("Excel existe", "❌ FAIL"))

# Check 2: RACI tiene 1 Accountable por proceso
raci_check = pd.read_excel(excel_path, sheet_name="RACI", index_col=0)
all_single_a = all((raci_check.loc[row] == "A").sum() == 1 for row in raci_check.index)
checks.append(("RACI: 1 Accountable por proceso", "✅ PASS" if all_single_a else "❌ FAIL"))

# Check 3: 4 archivos YAML de políticas
yaml_files = list(POLICIES_DIR.glob("*.yaml"))
checks.append(("4 políticas YAML", "✅ PASS" if len(yaml_files) == 4 else f"❌ FAIL ({len(yaml_files)})"))

# Check 4: Políticas YAML son parseables
yaml_valid = True
for yf in yaml_files:
    try:
        with open(yf) as f:
            yaml.safe_load(f)
    except:
        yaml_valid = False
checks.append(("YAML parseables", "✅ PASS" if yaml_valid else "❌ FAIL"))

# Check 5: Score de madurez calculado
madurez_check = pd.read_excel(excel_path, sheet_name="Madurez")
has_total = madurez_check[madurez_check["dimension"] == "TOTAL"].shape[0] == 1
checks.append(("Score madurez calculado", "✅ PASS" if has_total else "❌ FAIL"))

# Check 6: org_profile.json existe
profile_exists = (DATA_DIR / "raw" / "org_profile.json").exists()
checks.append(("org_profile.json existe", "✅ PASS" if profile_exists else "❌ FAIL"))

# Mostrar resultados
print(f"{'Check':<40} {'Resultado'}")
print("-" * 55)
for check_name, result in checks:
    print(f"{check_name:<40} {result}")

passed = sum(1 for _, r in checks if "PASS" in r)
print(f"\nResultado: {passed}/{len(checks)} checks pasados")
```

**Resultado esperado:** 6/6 checks pasados.

---

## Solución de Problemas

### Problema 1: Error `ModuleNotFoundError: No module named 'yaml'`

**Síntoma:** Al ejecutar `import yaml` se obtiene un error de importación.

**Causa:** El paquete `pyyaml` no está instalado en el entorno Python activo.

**Solución:**

```bash
# Desde terminal
pip install pyyaml==6.0.1

# Si usa entorno virtual, activarlo primero
source ~/finops-course/venv/bin/activate
pip install pyyaml==6.0.1

# Reiniciar el kernel de Jupyter después de instalar
```

En Jupyter, también se puede ejecutar:
```python
!pip install pyyaml==6.0.1
```
Luego reiniciar el kernel (Kernel → Restart Kernel).

---

### Problema 2: El archivo `finops_backlog_v1.xlsx` no se encuentra

**Síntoma:** `FileNotFoundError` al intentar cargar el backlog del lab anterior, o el DataFrame de backlog está vacío.

**Causa:** El lab 05-00-01 no fue completado o el archivo se guardó en una ubicación diferente.

**Solución:**

El código del Paso 1 ya incluye una rama condicional que crea un backlog de referencia si el archivo no existe. Si necesitas verificar manualmente:

```python
# Buscar el archivo en ubicaciones alternativas
import subprocess
result = subprocess.run(
    ["find", str(Path.home()), "-name", "finops_backlog_v1.xlsx", "-type", "f"],
    capture_output=True, text=True
)
print("Archivos encontrados:")
print(result.stdout if result.stdout else "Ninguno - se usará el backlog de referencia")
```

Si el archivo existe en otra ruta, cópialo:
```bash
cp /ruta/encontrada/finops_backlog_v1.xlsx ~/finops-course/data/processed/
```

---

## Limpieza

Este laboratorio genera artefactos que son insumos del lab 07-00-01. **No elimines** los siguientes archivos:

- `~/finops-course/lab06/output/finops_governance_v1.xlsx`
- `~/finops-course/lab06/policies/*.yaml`
- `~/finops-course/data/processed/finops_governance_v1.xlsx`
- `~/finops-course/data/raw/org_profile.json`

Si necesitas reiniciar el laboratorio desde cero:

```bash
# Solo si necesitas repetir el lab completo
rm -rf ~/finops-course/lab06/output/*
rm -rf ~/finops-course/lab06/policies/*
rm -f ~/finops-course/data/raw/org_profile.json
```

---

## Resumen

En este laboratorio construiste los artefactos fundamentales de gobernanza FinOps para NovaTech SRL:

| Artefacto | Contenido | Formato |
|-----------|-----------|---------|
| Matriz RACI | 5 procesos × 6 roles | Excel (hoja RACI) |
| Definición de roles | 6 roles con responsabilidades, KPIs y leads | Excel (hoja Roles) |
| Políticas operativas | Etiquetado, presupuestos, excepciones, aprobaciones | YAML + Excel |
| Evaluación de madurez | 20 preguntas, 5 dimensiones, scoring Crawl/Walk/Run | Excel (hoja Madurez) |

**Conceptos clave aplicados:**
- Cada proceso FinOps requiere exactamente un Accountable para evitar ambigüedad
- Las políticas deben ser específicas (umbrales numéricos, SLAs en horas, roles nombrados)
- El scoring de madurez proporciona una línea base medible para tracking de progreso
- Los artefactos de gobernanza son documentos vivos que evolucionan con la madurez

### Recursos adicionales

- [FinOps Foundation — Personas](https://www.finops.org/framework/personas/)
- [FinOps Foundation — Maturity Model](https://www.finops.org/framework/maturity-model/)
- [RACI Matrix Best Practices — PMI](https://www.pmi.org/learning/library/raci-model-responsibility-assignment-matrix-9823)

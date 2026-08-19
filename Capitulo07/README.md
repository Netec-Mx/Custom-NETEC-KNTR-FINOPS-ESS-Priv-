# Demostración 7. Diseño de arquitectura de datos FinOps con FOCUS

## Metadata

| Campo | Valor |
|-------|-------|
| **Duración** | 30 minutos |
| **Complejidad** | Alta |
| **Nivel Bloom** | Crear |
| **Tecnologías** | PostgreSQL 16.2, Grafana 10.3.3, Docker Compose 2.24.6, dbt-core 1.7.9, Python 3.12.1, FOCUS 1.0, Faker 24.2.0 |

## Visión General

En este laboratorio diseñarás e implementarás una arquitectura de datos FinOps completa para NovaTech SRL. Construirás un pipeline que ingesta datos de facturación heterogéneos de tres proveedores cloud (AWS, Azure, GCP), los normaliza al estándar FOCUS 1.0 usando dbt, aplica reglas de calidad de datos y los visualiza en un dashboard Grafana conectado a PostgreSQL. Este ejercicio integra los conceptos de herramientas nativas de billing vistos en la lección 7.1 con una implementación práctica de normalización multicloud.

## Objetivos de Aprendizaje

- [ ] Diseñar e implementar un pipeline de datos FinOps con etapas de ingesta, normalización FOCUS 1.0, enriquecimiento y calidad
- [ ] Configurar y poblar un data warehouse PostgreSQL 16.2 con esquema FOCUS-compatible usando tablas normalizadas de costos multicloud
- [ ] Implementar reglas de calidad de datos FinOps verificando completitud de tags, consistencia de moneda y trazabilidad de origen
- [ ] Configurar un dashboard en Grafana 10.3.3 conectado a PostgreSQL para visualización de métricas FinOps clave
- [ ] Documentar el modelo de gobierno de datos FinOps incluyendo ownership, linaje y SLAs de actualización

## Prerrequisitos

### Conocimientos Previos

- Labs 05-00-01 y 06-00-01 completados (archivos `finops_backlog_v1.xlsx` y políticas de etiquetado disponibles)
- Familiaridad con SQL básico y conceptos de modelado dimensional
- Comprensión del estándar FOCUS 1.0 y sus columnas principales
- Conocimiento básico de Docker y Docker Compose

### Acceso y Software

| Componente | Versión | Estado Requerido |
|---|---|---|
| Docker Desktop | 26.0.0 | Ejecutándose con ≥4 GB RAM asignados |
| Docker Compose | 2.24.6 | Disponible en PATH |
| Python | 3.12.1 | Instalado con pip funcional |
| dbt-core | 1.7.9 | Instalado vía pip |
| dbt-postgres | 1.7.9 | Instalado vía pip |
| Puertos libres | 5432, 3000 | Sin conflictos |

## Entorno del Laboratorio

### Estructura de Directorios

```
~/finops-labs/lab07/
├── docker/
│   └── docker-compose.yml
├── dbt_project/
│   ├── dbt_project.yml
│   ├── profiles.yml
│   ├── models/
│   │   ├── staging/
│   │   ├── marts/
│   │   └── schema.yml
│   └── tests/
├── data/
│   ├── raw/
│   └── processed/
├── scripts/
│   ├── generate_data.py
│   └── ingest_to_postgres.py
└── docs/
    └── data_governance.md
```

### Preparación Inicial

```bash
# Crear estructura de directorios
mkdir -p ~/finops-labs/lab07/{docker,dbt_project/{models/{staging,marts},tests},data/{raw,processed},scripts,docs}
cd ~/finops-labs/lab07

# Instalar dependencias Python
pip install dbt-core==1.7.9 dbt-postgres==1.7.9 SQLAlchemy==2.0.28 psycopg2-binary==2.9.9 pandas==2.2.1 Faker==24.2.0
```

---

## Paso 1: Levantar la Infraestructura con Docker Compose

**Objetivo:** Desplegar PostgreSQL 16.2 y Grafana 10.3.3 como servicios containerizados que servirán de data warehouse y capa de visualización.

### Instrucciones

1. Crear el archivo `docker-compose.yml`:

```bash
cat > ~/finops-labs/lab07/docker/docker-compose.yml << 'EOF'
version: '3.8'

services:
  postgres:
    image: postgres:16.2
    container_name: finops_postgres
    environment:
      POSTGRES_DB: finops_dw
      POSTGRES_USER: finops_admin
      POSTGRES_PASSWORD: finops2024
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U finops_admin -d finops_dw"]
      interval: 5s
      timeout: 5s
      retries: 5

  grafana:
    image: grafana/grafana:10.3.3
    container_name: finops_grafana
    environment:
      GF_SECURITY_ADMIN_USER: admin
      GF_SECURITY_ADMIN_PASSWORD: finops2024
      GF_USERS_ALLOW_SIGN_UP: "false"
    ports:
      - "3000:3000"
    volumes:
      - grafana_data:/var/lib/grafana
    depends_on:
      postgres:
        condition: service_healthy

volumes:
  pgdata:
  grafana_data:
EOF
```

2. Levantar los servicios:

```bash
cd ~/finops-labs/lab07/docker
docker compose up -d
```

3. Verificar que ambos contenedores estén saludables:

```bash
docker compose ps
```

### Salida Esperada

```
NAME              IMAGE                    STATUS                   PORTS
finops_grafana    grafana/grafana:10.3.3   Up (healthy)             0.0.0.0:3000->3000/tcp
finops_postgres   postgres:16.2            Up (healthy)             0.0.0.0:5432->5432/tcp
```

### Verificación

```bash
# Verificar conectividad a PostgreSQL
docker exec finops_postgres psql -U finops_admin -d finops_dw -c "SELECT version();"
```

Debe retornar `PostgreSQL 16.2`.

---

## Paso 2: Generar Datos Sintéticos Multicloud

**Objetivo:** Crear tres archivos CSV con esquemas heterogéneos que simulan las exportaciones de billing de AWS (CUR), Azure y GCP, reflejando la realidad de datos no normalizados que describe la lección 7.1.

### Instrucciones

1. Crear el script de generación de datos:

```bash
cat > ~/finops-labs/lab07/scripts/generate_data.py << 'EOF'
"""
Generador de datos sintéticos de billing multicloud para NovaTech SRL.
Produce 3 CSVs con esquemas heterogéneos simulando exportaciones nativas.
"""
import pandas as pd
import numpy as np
from faker import Faker
from datetime import datetime, timedelta
import random
import os

fake = Faker()
Faker.seed(42)
np.random.seed(42)
random.seed(42)

OUTPUT_DIR = os.path.expanduser("~/finops-labs/lab07/data/raw")
os.makedirs(OUTPUT_DIR, exist_ok=True)

# Constantes NovaTech
TEAMS = ["Backend", "Frontend", "Data", "Platform"]
PRODUCTS = ["PayCore", "AnalyticsHub", "DevPortal"]
ENVIRONMENTS = ["prod", "staging", "dev"]
DATE_START = datetime(2024, 1, 1)
DATE_END = datetime(2024, 1, 31)
DAYS = (DATE_END - DATE_START).days + 1

def generate_dates():
    return [DATE_START + timedelta(days=d) for d in range(DAYS)]

# --- AWS CUR Format ---
def generate_aws_data(n_records=500):
    dates = generate_dates()
    aws_services = ["AmazonEC2", "AmazonS3", "AmazonRDS", "AWSLambda", "AmazonEKS"]
    aws_regions = ["us-east-1", "us-west-2", "eu-west-1"]
    
    records = []
    for _ in range(n_records):
        day = random.choice(dates)
        team = random.choice(TEAMS)
        product = random.choice(PRODUCTS)
        env = random.choice(ENVIRONMENTS)
        service = random.choice(aws_services)
        
        # Producción cuesta más
        base_cost = random.uniform(5, 200) if env == "prod" else random.uniform(1, 50)
        
        records.append({
            "line_item_usage_account_id": f"1234567890{random.randint(10,19)}",
            "line_item_usage_start_date": day.strftime("%Y-%m-%dT00:00:00Z"),
            "line_item_usage_end_date": day.strftime("%Y-%m-%dT23:59:59Z"),
            "line_item_product_code": service,
            "line_item_resource_id": f"arn:aws:ec2:us-east-1:123456789012:{fake.uuid4()[:8]}",
            "product_region_code": random.choice(aws_regions),
            "line_item_line_item_type": random.choice(["Usage", "Usage", "Usage", "Tax"]),
            "line_item_unblended_cost": round(base_cost, 4),
            "line_item_currency_code": "USD",
            "resource_tags_user_team": team if random.random() > 0.15 else "",
            "resource_tags_user_product": product if random.random() > 0.20 else "",
            "resource_tags_user_environment": env if random.random() > 0.10 else "",
        })
    
    df = pd.DataFrame(records)
    df.to_csv(f"{OUTPUT_DIR}/aws_billing_raw.csv", index=False)
    print(f"AWS: {len(df)} registros generados")
    return df

# --- GCP Billing Export Format ---
def generate_gcp_data(n_records=400):
    dates = generate_dates()
    gcp_services = ["Compute Engine", "Cloud Storage", "BigQuery", "Cloud SQL", "GKE"]
    gcp_regions = ["us-central1", "europe-west1", "asia-east1"]
    
    records = []
    for _ in range(n_records):
        day = random.choice(dates)
        team = random.choice(TEAMS)
        product = random.choice(PRODUCTS)
        env = random.choice(ENVIRONMENTS)
        
        base_cost = random.uniform(3, 180) if env == "prod" else random.uniform(0.5, 40)
        
        records.append({
            "billing_account_id": "01A2B3-C4D5E6-F7G8H9",
            "usage_start_time": day.strftime("%Y-%m-%d %H:%M:%S UTC"),
            "usage_end_time": day.strftime("%Y-%m-%d 23:59:59 UTC"),
            "service.description": random.choice(gcp_services),
            "resource.name": f"projects/novatech-{env}/instances/{fake.uuid4()[:8]}",
            "location.region": random.choice(gcp_regions),
            "sku.description": f"N2 Instance Core running in Americas",
            "cost": round(base_cost, 4),
            "currency": "USD",
            "labels.team": team if random.random() > 0.25 else None,
            "labels.product": product if random.random() > 0.30 else None,
            "labels.env": env if random.random() > 0.12 else None,
        })
    
    df = pd.DataFrame(records)
    df.to_csv(f"{OUTPUT_DIR}/gcp_billing_raw.csv", index=False)
    print(f"GCP: {len(df)} registros generados")
    return df

# --- Azure Cost Export Format ---
def generate_azure_data(n_records=450):
    dates = generate_dates()
    azure_services = ["Virtual Machines", "Storage", "SQL Database", "App Service", "AKS"]
    azure_regions = ["eastus", "westeurope", "southeastasia"]
    
    records = []
    for _ in range(n_records):
        day = random.choice(dates)
        team = random.choice(TEAMS)
        product = random.choice(PRODUCTS)
        env = random.choice(ENVIRONMENTS)
        
        base_cost = random.uniform(4, 190) if env == "prod" else random.uniform(1, 45)
        
        records.append({
            "SubscriptionId": f"sub-novatech-{random.choice(['main','dev','data'])}",
            "Date": day.strftime("%Y-%m-%d"),
            "ServiceName": random.choice(azure_services),
            "ResourceId": f"/subscriptions/xxx/resourceGroups/rg-{env}/providers/{fake.uuid4()[:8]}",
            "ResourceLocation": random.choice(azure_regions),
            "ChargeType": random.choice(["Usage", "Usage", "Usage", "Purchase"]),
            "CostInBillingCurrency": round(base_cost, 4),
            "BillingCurrency": "USD",
            "Tag_Team": team if random.random() > 0.18 else "",
            "Tag_Product": product if random.random() > 0.22 else "",
            "Tag_Environment": env if random.random() > 0.08 else "",
        })
    
    df = pd.DataFrame(records)
    df.to_csv(f"{OUTPUT_DIR}/azure_billing_raw.csv", index=False)
    print(f"Azure: {len(df)} registros generados")
    return df

if __name__ == "__main__":
    print("=== Generando datos sintéticos multicloud para NovaTech SRL ===")
    generate_aws_data()
    generate_gcp_data()
    generate_azure_data()
    print(f"\nArchivos generados en: {OUTPUT_DIR}")
    print("Archivos: aws_billing_raw.csv, gcp_billing_raw.csv, azure_billing_raw.csv")
EOF
```

2. Ejecutar el script:

```bash
cd ~/finops-labs/lab07
python scripts/generate_data.py
```

### Salida Esperada

```
=== Generando datos sintéticos multicloud para NovaTech SRL ===
AWS: 500 registros generados
GCP: 400 registros generados
Azure: 450 registros generados

Archivos generados en: /home/<usuario>/finops-labs/lab07/data/raw
Archivos: aws_billing_raw.csv, gcp_billing_raw.csv, azure_billing_raw.csv
```

### Verificación

```bash
# Confirmar que los archivos existen y tienen contenido
wc -l ~/finops-labs/lab07/data/raw/*.csv
# Verificar esquemas heterogéneos
head -1 ~/finops-labs/lab07/data/raw/aws_billing_raw.csv
head -1 ~/finops-labs/lab07/data/raw/gcp_billing_raw.csv
head -1 ~/finops-labs/lab07/data/raw/azure_billing_raw.csv
```

Las columnas deben ser completamente diferentes entre los tres archivos, simulando la realidad multicloud descrita en la lección.

---

## Paso 3: Ingestar Datos Raw a PostgreSQL (Staging)

**Objetivo:** Cargar los tres archivos CSV a tablas staging en PostgreSQL, replicando la etapa de ingesta de un pipeline de datos FinOps donde los datos llegan sin transformar.

### Instrucciones

1. Crear el script de ingesta:

```bash
cat > ~/finops-labs/lab07/scripts/ingest_to_postgres.py << 'EOF'
"""
Script de ingesta: carga CSVs raw a tablas staging en PostgreSQL.
Equivale a la exportación de datos de billing descrita en la lección 7.1.
"""
import pandas as pd
from sqlalchemy import create_engine, text
import os

DB_URL = "postgresql://finops_admin:finops2024@localhost:5432/finops_dw"
RAW_DIR = os.path.expanduser("~/finops-labs/lab07/data/raw")

def create_schemas(engine):
    """Crear esquemas staging y marts."""
    with engine.connect() as conn:
        conn.execute(text("CREATE SCHEMA IF NOT EXISTS staging;"))
        conn.execute(text("CREATE SCHEMA IF NOT EXISTS marts;"))
        conn.commit()
    print("Esquemas 'staging' y 'marts' creados.")

def ingest_csv(engine, filename, table_name, schema="staging"):
    """Cargar un CSV a una tabla PostgreSQL."""
    filepath = os.path.join(RAW_DIR, filename)
    df = pd.read_csv(filepath)
    df.to_sql(table_name, engine, schema=schema, if_exists="replace", index=False)
    print(f"  → {table_name}: {len(df)} registros cargados")
    return len(df)

def main():
    print("=== Ingesta de datos raw a PostgreSQL (staging) ===\n")
    engine = create_engine(DB_URL)
    
    create_schemas(engine)
    
    total = 0
    print("\nCargando archivos:")
    total += ingest_csv(engine, "aws_billing_raw.csv", "stg_aws_costs")
    total += ingest_csv(engine, "gcp_billing_raw.csv", "stg_gcp_costs")
    total += ingest_csv(engine, "azure_billing_raw.csv", "stg_azure_costs")
    
    print(f"\n✓ Total: {total} registros cargados en esquema 'staging'")
    
    # Verificación
    with engine.connect() as conn:
        for table in ["stg_aws_costs", "stg_gcp_costs", "stg_azure_costs"]:
            result = conn.execute(text(f"SELECT COUNT(*) FROM staging.{table}"))
            count = result.scalar()
            print(f"  Verificación {table}: {count} filas")

if __name__ == "__main__":
    main()
EOF
```

2. Ejecutar la ingesta:

```bash
python ~/finops-labs/lab07/scripts/ingest_to_postgres.py
```

### Salida Esperada

```
=== Ingesta de datos raw a PostgreSQL (staging) ===

Esquemas 'staging' y 'marts' creados.

Cargando archivos:
  → stg_aws_costs: 500 registros cargados
  → stg_gcp_costs: 400 registros cargados
  → stg_azure_costs: 450 registros cargados

✓ Total: 1350 registros cargados en esquema 'staging'
  Verificación stg_aws_costs: 500 filas
  Verificación stg_gcp_costs: 400 filas
  Verificación stg_azure_costs: 450 filas
```

### Verificación

```bash
docker exec finops_postgres psql -U finops_admin -d finops_dw -c \
  "SELECT table_schema, table_name FROM information_schema.tables WHERE table_schema = 'staging';"
```

---

## Paso 4: Implementar Normalización FOCUS 1.0 con dbt

**Objetivo:** Crear modelos dbt que transforman los datos heterogéneos de staging en un esquema unificado FOCUS 1.0, resolviendo el problema de normalización multicloud.

### Instrucciones

1. Crear la configuración del proyecto dbt:

```bash
cat > ~/finops-labs/lab07/dbt_project/dbt_project.yml << 'EOF'
name: 'finops_novatech'
version: '1.0.0'
config-version: 2

profile: 'finops_novatech'

model-paths: ["models"]
test-paths: ["tests"]
target-path: "target"
clean-targets: ["target", "dbt_packages"]

models:
  finops_novatech:
    staging:
      +schema: staging
      +materialized: view
    marts:
      +schema: marts
      +materialized: table
EOF
```

2. Crear el perfil de conexión:

```bash
cat > ~/finops-labs/lab07/dbt_project/profiles.yml << 'EOF'
finops_novatech:
  target: dev
  outputs:
    dev:
      type: postgres
      host: localhost
      port: 5432
      user: finops_admin
      password: finops2024
      dbname: finops_dw
      schema: public
      threads: 4
EOF
```

3. Crear el modelo de normalización FOCUS para AWS:

```bash
cat > ~/finops-labs/lab07/dbt_project/models/staging/stg_aws_focus.sql << 'EOF'
-- Normalización de datos AWS CUR al estándar FOCUS 1.0
SELECT
    line_item_usage_account_id AS "BillingAccountId",
    CAST(LEFT(line_item_usage_start_date, 10) AS DATE) AS "ChargePeriodStart",
    CAST(LEFT(line_item_usage_end_date, 10) AS DATE) AS "ChargePeriodEnd",
    line_item_product_code AS "ServiceName",
    line_item_resource_id AS "ResourceId",
    product_region_code AS "RegionName",
    CASE 
        WHEN line_item_line_item_type = 'Tax' THEN 'Tax'
        ELSE 'Usage'
    END AS "ChargeType",
    line_item_unblended_cost AS "BilledCost",
    line_item_currency_code AS "BillingCurrency",
    resource_tags_user_team AS "TagTeam",
    resource_tags_user_product AS "TagProduct",
    resource_tags_user_environment AS "TagEnvironment",
    'AWS' AS "Provider",
    NOW() AS "IngestedAt"
FROM staging.stg_aws_costs
EOF
```

4. Crear el modelo de normalización FOCUS para GCP:

```bash
cat > ~/finops-labs/lab07/dbt_project/models/staging/stg_gcp_focus.sql << 'EOF'
-- Normalización de datos GCP Billing Export al estándar FOCUS 1.0
SELECT
    billing_account_id AS "BillingAccountId",
    CAST(LEFT(usage_start_time, 10) AS DATE) AS "ChargePeriodStart",
    CAST(LEFT(usage_end_time, 10) AS DATE) AS "ChargePeriodEnd",
    "service.description" AS "ServiceName",
    "resource.name" AS "ResourceId",
    "location.region" AS "RegionName",
    'Usage' AS "ChargeType",
    cost AS "BilledCost",
    currency AS "BillingCurrency",
    "labels.team" AS "TagTeam",
    "labels.product" AS "TagProduct",
    "labels.env" AS "TagEnvironment",
    'GCP' AS "Provider",
    NOW() AS "IngestedAt"
FROM staging.stg_gcp_costs
EOF
```

5. Crear el modelo de normalización FOCUS para Azure:

```bash
cat > ~/finops-labs/lab07/dbt_project/models/staging/stg_azure_focus.sql << 'EOF'
-- Normalización de datos Azure Cost Export al estándar FOCUS 1.0
SELECT
    "SubscriptionId" AS "BillingAccountId",
    CAST("Date" AS DATE) AS "ChargePeriodStart",
    CAST("Date" AS DATE) AS "ChargePeriodEnd",
    "ServiceName" AS "ServiceName",
    "ResourceId" AS "ResourceId",
    "ResourceLocation" AS "RegionName",
    "ChargeType" AS "ChargeType",
    "CostInBillingCurrency" AS "BilledCost",
    "BillingCurrency" AS "BillingCurrency",
    "Tag_Team" AS "TagTeam",
    "Tag_Product" AS "TagProduct",
    "Tag_Environment" AS "TagEnvironment",
    'Azure' AS "Provider",
    NOW() AS "IngestedAt"
FROM staging.stg_azure_costs
EOF
```

6. Crear el modelo mart unificado con enriquecimiento:

```bash
cat > ~/finops-labs/lab07/dbt_project/models/marts/fct_costs_focus.sql << 'EOF'
-- Modelo FOCUS 1.0 unificado con enriquecimiento organizacional
-- Integra los 3 proveedores y aplica reglas de asignación de NovaTech SRL
WITH unified AS (
    SELECT * FROM {{ ref('stg_aws_focus') }}
    UNION ALL
    SELECT * FROM {{ ref('stg_gcp_focus') }}
    UNION ALL
    SELECT * FROM {{ ref('stg_azure_focus') }}
),

enriched AS (
    SELECT
        "BillingAccountId",
        "ChargePeriodStart",
        "ChargePeriodEnd",
        "ServiceName",
        "ResourceId",
        "RegionName",
        "ChargeType",
        "BilledCost",
        "BillingCurrency",
        "Provider",
        "IngestedAt",
        -- Enriquecimiento: normalizar tags vacíos a 'Untagged'
        COALESCE(NULLIF(TRIM("TagTeam"), ''), 'Untagged') AS "Team",
        COALESCE(NULLIF(TRIM("TagProduct"), ''), 'Untagged') AS "Product",
        COALESCE(NULLIF(TRIM("TagEnvironment"), ''), 'Untagged') AS "Environment",
        -- Métricas de calidad
        CASE WHEN COALESCE(NULLIF(TRIM("TagTeam"), ''), '') = '' THEN 0 ELSE 1 END
        + CASE WHEN COALESCE(NULLIF(TRIM("TagProduct"), ''), '') = '' THEN 0 ELSE 1 END
        + CASE WHEN COALESCE(NULLIF(TRIM("TagEnvironment"), ''), '') = '' THEN 0 ELSE 1 END
        AS "TagCompleteness"
    FROM unified
)

SELECT
    *,
    ROUND(("TagCompleteness"::NUMERIC / 3.0) * 100, 1) AS "TagCompletenessPercent"
FROM enriched
EOF
```

7. Crear el archivo de schema con tests de calidad:

```bash
cat > ~/finops-labs/lab07/dbt_project/models/schema.yml << 'EOF'
version: 2

models:
  - name: stg_aws_focus
    description: "Datos AWS normalizados al estándar FOCUS 1.0"
    columns:
      - name: BilledCost
        tests:
          - not_null
      - name: BillingCurrency
        tests:
          - not_null
          - accepted_values:
              values: ['USD']
      - name: Provider
        tests:
          - accepted_values:
              values: ['AWS']

  - name: stg_gcp_focus
    description: "Datos GCP normalizados al estándar FOCUS 1.0"
    columns:
      - name: BilledCost
        tests:
          - not_null
      - name: BillingCurrency
        tests:
          - not_null
          - accepted_values:
              values: ['USD']
      - name: Provider
        tests:
          - accepted_values:
              values: ['GCP']

  - name: stg_azure_focus
    description: "Datos Azure normalizados al estándar FOCUS 1.0"
    columns:
      - name: BilledCost
        tests:
          - not_null
      - name: BillingCurrency
        tests:
          - not_null
          - accepted_values:
              values: ['USD']
      - name: Provider
        tests:
          - accepted_values:
              values: ['Azure']

  - name: fct_costs_focus
    description: "Tabla de hechos FOCUS 1.0 unificada multicloud con enriquecimiento"
    columns:
      - name: BilledCost
        tests:
          - not_null
      - name: ChargePeriodStart
        tests:
          - not_null
      - name: Provider
        tests:
          - not_null
          - accepted_values:
              values: ['AWS', 'GCP', 'Azure']
      - name: Team
        tests:
          - not_null
          - accepted_values:
              values: ['Backend', 'Frontend', 'Data', 'Platform', 'Untagged']
      - name: Environment
        tests:
          - not_null
          - accepted_values:
              values: ['prod', 'staging', 'dev', 'Untagged']
EOF
```

8. Ejecutar dbt:

```bash
cd ~/finops-labs/lab07/dbt_project
dbt run --profiles-dir .
dbt test --profiles-dir .
```

### Salida Esperada

```
Running with dbt=1.7.9
Found 4 models, 15 tests...

Concurrency: 4 threads (target='dev')

1 of 4 START sql view model staging.stg_aws_focus .......................... [RUN]
2 of 4 START sql view model staging.stg_gcp_focus .......................... [RUN]
3 of 4 START sql view model staging.stg_azure_focus ........................ [RUN]
1 of 4 OK created sql view model staging.stg_aws_focus ..................... [CREATE VIEW]
2 of 4 OK created sql view model staging.stg_gcp_focus ..................... [CREATE VIEW]
3 of 4 OK created sql view model staging.stg_azure_focus ................... [CREATE VIEW]
4 of 4 START sql table model marts.fct_costs_focus ......................... [RUN]
4 of 4 OK created sql table model marts.fct_costs_focus .................... [SELECT 1350]

Finished running 3 view models, 1 table model in 0 hours 0 minutes and X seconds.
Completed successfully

Done. PASS=4 WARN=0 ERROR=0 SKIP=0 TOTAL=4
```

Para los tests:

```
Completed successfully
Done. PASS=15 WARN=0 ERROR=0 SKIP=0 TOTAL=15
```

### Verificación

```bash
docker exec finops_postgres psql -U finops_admin -d finops_dw -c \
  "SELECT \"Provider\", COUNT(*), ROUND(SUM(\"BilledCost\")::numeric, 2) as total_cost 
   FROM marts.fct_costs_focus GROUP BY \"Provider\" ORDER BY total_cost DESC;"
```

Debe mostrar los tres proveedores con sus totales.

---

## Paso 5: Configurar Dashboard en Grafana

**Objetivo:** Crear un dashboard funcional en Grafana conectado a PostgreSQL que visualice las métricas FinOps clave: costo por servicio, costo por equipo, tendencia diaria y cobertura de tags.

### Instrucciones

1. Acceder a Grafana en `http://localhost:3000` con credenciales `admin` / `finops2024`.

2. Configurar el datasource PostgreSQL. Ejecutar vía API para automatizar:

```bash
curl -X POST http://admin:finops2024@localhost:3000/api/datasources \
  -H "Content-Type: application/json" \
  -d '{
    "name": "FinOps PostgreSQL",
    "type": "postgres",
    "url": "finops_postgres:5432",
    "database": "finops_dw",
    "user": "finops_admin",
    "secureJsonData": {"password": "finops2024"},
    "jsonData": {
      "sslmode": "disable",
      "postgresVersion": 1600
    },
    "access": "proxy",
    "isDefault": true
}'
```

3. Crear el dashboard con 4 paneles vía API:

```bash
cat > /tmp/grafana_dashboard.json << 'EOF'
{
  "dashboard": {
    "title": "NovaTech FinOps - Arquitectura FOCUS",
    "tags": ["finops", "focus", "novatech"],
    "timezone": "browser",
    "panels": [
      {
        "id": 1,
        "title": "Costo Total por Servicio (Top 10)",
        "type": "barchart",
        "gridPos": {"h": 9, "w": 12, "x": 0, "y": 0},
        "targets": [{
          "rawSql": "SELECT \"ServiceName\" as service, ROUND(SUM(\"BilledCost\")::numeric, 2) as cost FROM marts.fct_costs_focus GROUP BY \"ServiceName\" ORDER BY cost DESC LIMIT 10;",
          "format": "table",
          "datasource": "FinOps PostgreSQL"
        }]
      },
      {
        "id": 2,
        "title": "Costo por Equipo",
        "type": "piechart",
        "gridPos": {"h": 9, "w": 12, "x": 12, "y": 0},
        "targets": [{
          "rawSql": "SELECT \"Team\" as team, ROUND(SUM(\"BilledCost\")::numeric, 2) as cost FROM marts.fct_costs_focus GROUP BY \"Team\" ORDER BY cost DESC;",
          "format": "table",
          "datasource": "FinOps PostgreSQL"
        }]
      },
      {
        "id": 3,
        "title": "Tendencia Diaria de Costos",
        "type": "timeseries",
        "gridPos": {"h": 9, "w": 16, "x": 0, "y": 9},
        "targets": [{
          "rawSql": "SELECT \"ChargePeriodStart\" as time, \"Provider\" as metric, ROUND(SUM(\"BilledCost\")::numeric, 2) as cost FROM marts.fct_costs_focus GROUP BY \"ChargePeriodStart\", \"Provider\" ORDER BY time;",
          "format": "time_series",
          "datasource": "FinOps PostgreSQL"
        }]
      },
      {
        "id": 4,
        "title": "% Recursos sin Tags Válidos",
        "type": "stat",
        "gridPos": {"h": 9, "w": 8, "x": 16, "y": 9},
        "targets": [{
          "rawSql": "SELECT ROUND((COUNT(*) FILTER (WHERE \"TagCompleteness\" < 3)::numeric / COUNT(*)::numeric) * 100, 1) as pct_incomplete FROM marts.fct_costs_focus;",
          "format": "table",
          "datasource": "FinOps PostgreSQL"
        }]
      }
    ],
    "schemaVersion": 39
  },
  "overwrite": true
}
EOF

curl -X POST http://admin:finops2024@localhost:3000/api/dashboards/db \
  -H "Content-Type: application/json" \
  -d @/tmp/grafana_dashboard.json
```

### Salida Esperada

```json
{"id":1,"slug":"novatech-finops-arquitectura-focus","status":"success","uid":"...","url":"/d/.../novatech-finops-arquitectura-focus","version":1}
```

### Verificación

Abrir `http://localhost:3000` en el navegador. Navegar al dashboard "NovaTech FinOps - Arquitectura FOCUS". Deben verse los 4 paneles con datos:
- Panel 1: Barras con los servicios más costosos
- Panel 2: Gráfico circular con distribución por equipo (incluyendo "Untagged")
- Panel 3: Serie temporal con líneas por proveedor
- Panel 4: Porcentaje estadístico de recursos sin tags completos (esperado: ~15-30%)

---

## Paso 6: Documentar el Modelo de Gobierno de Datos

**Objetivo:** Crear la documentación de gobierno de datos FinOps que define ownership, linaje, SLAs y reglas de calidad del pipeline implementado.

### Instrucciones

1. Crear el documento de gobierno:

```bash
cat > ~/finops-labs/lab07/docs/data_governance.md << 'EOF'
# Modelo de Gobierno de Datos FinOps - NovaTech SRL

## 1. Ownership de Datos

| Capa | Owner | Responsabilidad |
|------|-------|-----------------|
| Raw (staging) | Platform Team | Garantizar ingesta diaria antes de las 08:00 UTC |
| Normalización FOCUS | FinOps Team | Mantener modelos dbt actualizados con cambios de esquema |
| Enriquecimiento | FinOps Team + Product Owners | Validar mapeo de tags a equipos/productos |
| Visualización | FinOps Team | Mantener dashboards operativos y precisos |

## 2. Linaje de Datos

```
[AWS CUR / GCP Export / Azure Export]
        ↓ (Ingesta Python - diaria 06:00 UTC)
[PostgreSQL: staging.stg_*_costs]
        ↓ (dbt run - diaria 07:00 UTC)
[PostgreSQL: staging.stg_*_focus]  ← Normalización FOCUS 1.0
        ↓ (dbt run - diaria 07:00 UTC)
[PostgreSQL: marts.fct_costs_focus]  ← Enriquecimiento + Calidad
        ↓ (Conexión directa)
[Grafana Dashboard]  ← Visualización
```

## 3. SLAs de Actualización

| Métrica | SLA | Medición |
|---------|-----|----------|
| Latencia de datos | ≤ 48 horas desde el uso real | Diferencia entre uso y disponibilidad en marts |
| Frecuencia de actualización | Diaria | Pipeline ejecutado cada 24h |
| Disponibilidad del dashboard | 99.5% mensual | Uptime de Grafana + PostgreSQL |
| Completitud de tags | ≥ 80% de recursos con 3/3 tags | Métrica TagCompletenessPercent |

## 4. Reglas de Calidad de Datos

| Regla | Implementación | Severidad |
|-------|---------------|-----------|
| BilledCost no nulo | dbt test: not_null | ERROR (bloquea pipeline) |
| Moneda consistente (USD) | dbt test: accepted_values | ERROR |
| Proveedor válido | dbt test: accepted_values [AWS, GCP, Azure] | ERROR |
| Equipo válido | dbt test: accepted_values | WARN |
| Fecha no futura | Custom test SQL | ERROR |

## 5. Proceso de Resolución de Incidentes de Datos

1. **Detección**: Alerta automática si dbt test falla
2. **Triage**: Platform Team evalúa en ≤ 2 horas
3. **Resolución**: Fix aplicado en ≤ 24 horas
4. **Post-mortem**: Documentado si impacto > $1,000 en datos incorrectos
EOF
```

### Salida Esperada

Archivo `~/finops-labs/lab07/docs/data_governance.md` creado con la documentación completa de gobierno.

### Verificación

```bash
# Confirmar que el documento existe y tiene contenido sustancial
wc -l ~/finops-labs/lab07/docs/data_governance.md
# Debe tener aproximadamente 60-70 líneas
```

---

## Validación y Testing

Ejecutar la siguiente secuencia completa de validación:

```bash
echo "=== VALIDACIÓN COMPLETA DEL LAB 07 ==="

# 1. Verificar contenedores activos
echo -e "\n[1/5] Contenedores Docker:"
docker compose -f ~/finops-labs/lab07/docker/docker-compose.yml ps --format "table {{.Name}}\t{{.Status}}"

# 2. Verificar datos en staging
echo -e "\n[2/5] Datos en staging:"
docker exec finops_postgres psql -U finops_admin -d finops_dw -c \
  "SELECT 'stg_aws_costs' as tabla, COUNT(*) as filas FROM staging.stg_aws_costs
   UNION ALL SELECT 'stg_gcp_costs', COUNT(*) FROM staging.stg_gcp_costs
   UNION ALL SELECT 'stg_azure_costs', COUNT(*) FROM staging.stg_azure_costs;"

# 3. Verificar modelo FOCUS unificado
echo -e "\n[3/5] Modelo FOCUS unificado:"
docker exec finops_postgres psql -U finops_admin -d finops_dw -c \
  "SELECT \"Provider\", COUNT(*) as registros, 
   ROUND(AVG(\"TagCompletenessPercent\")::numeric, 1) as avg_tag_pct
   FROM marts.fct_costs_focus GROUP BY \"Provider\";"

# 4. Verificar tests dbt
echo -e "\n[4/5] Tests dbt:"
cd ~/finops-labs/lab07/dbt_project && dbt test --profiles-dir . 2>&1 | tail -3

# 5. Verificar Grafana datasource
echo -e "\n[5/5] Grafana datasource:"
curl -s http://admin:finops2024@localhost:3000/api/datasources | python3 -c "
import json, sys
ds = json.load(sys.stdin)
for d in ds:
    print(f'  {d[\"name\"]}: {d[\"type\"]} -> {d[\"database\"]}')"

echo -e "\n=== VALIDACIÓN COMPLETADA ==="
```

**Criterio de éxito:** Los 5 checks deben pasar sin errores. El modelo `fct_costs_focus` debe contener exactamente 1350 registros y todos los tests dbt deben ser PASS.

---

## Troubleshooting

### Problema 1: Error de conexión de dbt a PostgreSQL

**Síntomas:** Al ejecutar `dbt run`, aparece el error:
```
Runtime Error: connection to server at "localhost" (127.0.0.1), port 5432 failed: Connection refused
```

**Causa:** El contenedor PostgreSQL no está completamente inicializado o el puerto no está mapeado correctamente al host. Docker puede tardar unos segundos en exponer el puerto después de que el contenedor reporta "healthy".

**Solución:**
```bash
# Verificar que el contenedor está corriendo y healthy
docker inspect finops_postgres --format='{{.State.Health.Status}}'

# Si no está healthy, esperar y reiniciar
docker compose -f ~/finops-labs/lab07/docker/docker-compose.yml restart postgres
sleep 10

# Verificar conectividad directa
docker exec finops_postgres psql -U finops_admin -d finops_dw -c "SELECT 1;"

# Si el problema persiste, verificar que el puerto no está ocupado
lsof -i :5432
```

### Problema 2: Tests dbt fallan por columnas con comillas en PostgreSQL

**Síntomas:** Al ejecutar `dbt test`, aparecen errores como:
```
Database Error: column "billedcost" does not exist
HINT: Perhaps you meant to reference the column "fct_costs_focus.BilledCost"
```

**Causa:** PostgreSQL convierte nombres de columna a minúsculas a menos que estén entre comillas dobles. Los modelos SQL usan `"BilledCost"` con comillas, pero el schema.yml puede referenciar sin comillas, causando inconsistencia.

**Solución:** Asegurarse de que las referencias en `schema.yml` usen exactamente el mismo nombre que aparece en la tabla materializada:

```bash
# Verificar nombres reales de columnas
docker exec finops_postgres psql -U finops_admin -d finops_dw -c \
  "SELECT column_name FROM information_schema.columns 
   WHERE table_schema='marts' AND table_name='fct_costs_focus';"

# Si las columnas aparecen con mayúsculas (entre comillas), 
# actualizar schema.yml para usar el nombre exacto con quote:
# En dbt, usar la sintaxis: name: '"BilledCost"' (con comillas internas)
```

Alternativamente, modificar los modelos SQL para usar nombres en minúsculas sin comillas (ej: `billed_cost` en lugar de `"BilledCost"`).

---

## Limpieza

```bash
# Detener y eliminar contenedores y volúmenes
cd ~/finops-labs/lab07/docker
docker compose down -v

# Verificar que los contenedores fueron eliminados
docker ps -a | grep finops

# (Opcional) Eliminar imágenes descargadas para liberar espacio
docker rmi postgres:16.2 grafana/grafana:10.3.3
```

> **Nota:** NO eliminar el directorio `~/finops-labs/lab07/` ya que los artefactos (esquema dbt, modelos, documentación de gobierno) son insumos requeridos para el Lab 08-00-01.

---

## Resumen

En este laboratorio implementaste una arquitectura de datos FinOps completa que refleja cómo las herramientas nativas de billing (AWS CUR, Azure Cost Export, GCP Billing Export) descritas en la lección 7.1 alimentan un pipeline de normalización:

| Etapa | Herramienta | Artefacto Generado |
|-------|-------------|-------------------|
| Generación de datos | Python + Faker | 3 CSVs con esquemas heterogéneos |
| Ingesta (staging) | SQLAlchemy + PostgreSQL | Tablas `staging.stg_*_costs` |
| Normalización FOCUS | dbt models | Vistas `staging.stg_*_focus` |
| Enriquecimiento | dbt mart | Tabla `marts.fct_costs_focus` |
| Calidad | dbt tests | 15 tests PASS |
| Visualización | Grafana | Dashboard con 4 paneles |
| Gobierno | Documentación | `data_governance.md` |

**Conexión con la lección:** Las exportaciones de billing nativas (CUR, Azure Export, GCP BigQuery Export) que estudiaste en la lección 7.1 producen datos con esquemas completamente diferentes. Este laboratorio demuestra por qué la normalización FOCUS 1.0 es esencial para organizaciones multicloud y cómo implementarla técnicamente.

### Recursos Adicionales

- [FOCUS Specification 1.0](https://focus.finops.org/) — Estándar completo de columnas y definiciones
- [dbt Documentation](https://docs.getdbt.com/) — Referencia de modelos y tests
- [Grafana PostgreSQL Datasource](https://grafana.com/docs/grafana/latest/datasources/postgres/) — Configuración avanzada de queries

# Demostración 1. Diagnóstico inicial de problemas FinOps

## Metadata

| Campo | Valor |
|-------|-------|
| **Duración** | 15 minutos |
| **Complejidad** | Fácil |
| **Nivel Bloom** | Aplicar |
| **Organización ficticia** | NovaTech SRL |
| **Período de análisis** | Q4 2023 – Enero 2024 |

## Descripción General

En este laboratorio asumirás el rol de un consultor FinOps contratado por NovaTech SRL, una empresa de software B2B con presencia en AWS, Azure y GCP. La empresa experimenta sobrecostos inexplicables, falta de visibilidad financiera y conflictos entre equipos de ingeniería y finanzas. Tu tarea es diagnosticar los problemas utilizando el marco FinOps Foundation como referencia, clasificando cada síntoma según su dominio, capacidad y fase correspondiente.

## Objetivos de Aprendizaje

- [ ] Identificar síntomas y causas raíz de problemas típicos de gestión de costos cloud usando el marco FinOps Foundation como referencia diagnóstica
- [ ] Clasificar los problemas detectados según dominios, capacidades y fases del Framework FinOps para priorizar acciones de mejora
- [ ] Distinguir entre iniciativas de control de costos, ahorro y generación de valor de negocio en escenarios cloud reales
- [ ] Aplicar los principios del marco FinOps Foundation para evaluar la madurez FinOps de una organización ficticia

## Prerrequisitos

### Conocimiento previo

| Requisito | Detalle |
|-----------|---------|
| Material teórico | Lectura completada de secciones 1.1, 1.2, 1.3 y 1.4 del Módulo 1 |
| Conceptos FinOps | Comprensión de los tres pilares (personas, procesos, tecnología), dominios y fases del framework |
| Hojas de cálculo | Manejo básico de LibreOffice Calc (abrir, editar celdas, guardar) |

### Acceso y software

| Software | Versión | Propósito |
|----------|---------|-----------|
| LibreOffice Calc | 24.2.1 | Editar plantilla de diagnóstico |
| Python | 3.12.1 | Verificación opcional de datos CSV |
| Jupyter Notebook | 7.1.2 | Exploración opcional del dataset |
| Terminal / Shell | — | Navegación de directorios y verificación de archivos |

## Entorno de Laboratorio

### Estructura de directorios

```
~/finops-course/
├── data/
│   ├── raw/
│   │   └── novatech_symptoms.csv          ← Provisto por instructor
│   └── processed/
│       └── diagnostico_finops_novatech_v1.xlsx  ← Archivo de salida
├── notebooks/
└── outputs/
```

### Configuración inicial

Ejecuta los siguientes comandos para verificar que tu entorno está listo:

```bash
# Verificar directorio de trabajo
cd ~/finops-course
ls data/raw/novatech_symptoms.csv
```

**Salida esperada:**
```
data/raw/novatech_symptoms.csv
```

Si el archivo no existe, solicítalo al instructor antes de continuar.

## Paso a Paso

### Paso 1: Verificar el entorno y los archivos de entrada

**Objetivo:** Confirmar que los archivos necesarios están disponibles y el entorno está operativo.

**Instrucciones:**

1. Abre una terminal y navega al directorio del curso:

```bash
cd ~/finops-course
```

2. Verifica la existencia del archivo de síntomas:

```bash
cat data/raw/novatech_symptoms.csv
```

3. Confirma que el archivo tiene contenido válido. El CSV debe contener las siguientes columnas: `id`, `sintoma`, `area_afectada`, `frecuencia`, `impacto_estimado_usd`, `reportado_por`.

**Salida esperada:**

```csv
id,sintoma,area_afectada,frecuencia,impacto_estimado_usd,reportado_por
S01,"Facturas mensuales con incrementos del 25-40% sin correlación con tráfico",Finanzas,Mensual,45000,CFO
S02,"Instancias EC2 en AWS ejecutándose sin carga útil durante fines de semana",Ingeniería,Semanal,12000,SRE Lead
S03,"Imposibilidad de atribuir costos a productos específicos (PayCore vs AnalyticsHub vs DevPortal)",Finanzas/Producto,Permanente,0,VP Engineering
S04,"Equipo de Data usa GPU instances para notebooks de exploración sin apagar al terminar",Data,Diaria,8500,Data Engineering Manager
S05,"No existen alertas de presupuesto; los excesos se detectan 3 semanas después",Finanzas,Mensual,22000,Controller
S06,"Ambiente staging replica la capacidad de producción sin justificación de carga",Platform,Permanente,18000,Platform Lead
S07,"Conflictos recurrentes entre VP Engineering y CFO sobre responsabilidad del gasto",Liderazgo,Quincenal,0,CEO
S08,"Reservas de Azure compradas hace 18 meses sin revisión de utilización actual",Finanzas/Platform,Permanente,15000,Cloud Architect
S09,"Equipos Frontend y Backend desconocen el costo de los servicios que despliegan",Ingeniería,Permanente,0,Engineering Managers
S10,"Forecasting de gasto cloud se realiza con spreadsheet manual una vez al trimestre",Finanzas,Trimestral,0,FP&A Analyst
```

**Verificación:** El archivo contiene exactamente 10 registros de síntomas (S01 a S10) con 6 columnas cada uno.

---

### Paso 2: Crear la plantilla de diagnóstico FinOps

**Objetivo:** Construir el archivo de diagnóstico estructurado que mapea cada síntoma al marco FinOps Foundation.

**Instrucciones:**

1. Abre LibreOffice Calc.

2. Crea un nuevo archivo y configura la **Hoja 1** con el nombre `Diagnóstico`. Define las siguientes columnas en la fila 1:

| Columna A | Columna B | Columna C | Columna D | Columna E | Columna F | Columna G | Columna H |
|-----------|-----------|-----------|-----------|-----------|-----------|-----------|-----------|
| ID Síntoma | Descripción del Síntoma | Dominio FinOps | Capacidad FinOps | Fase (Inform/Optimize/Operate) | Tipo de Iniciativa | Causa Raíz | Prioridad (Alta/Media/Baja) |

3. Crea una **Hoja 2** llamada `Referencia_Framework` con la siguiente tabla de apoyo:

| Dominio | Capacidades principales |
|---------|------------------------|
| Understand Cloud Usage & Cost | Data Ingestion, Allocation, Reporting & Analytics |
| Quantify Business Value | Planning & Forecasting, Budgeting |
| Optimize Cloud Usage & Cost | Architecting for Cloud, Rate Optimization, Workload Optimization, Licensing & SaaS |
| Manage the FinOps Practice | FinOps Practice Operations, FinOps Education & Enablement, Intersecting Disciplines |

4. En la **Hoja 2**, agrega una segunda tabla con los tipos de iniciativa:

| Tipo de Iniciativa | Definición |
|--------------------|------------|
| Control de costos | Establecer visibilidad, atribución y gobernanza del gasto |
| Ahorro | Reducir gasto sin impactar funcionalidad o rendimiento |
| Generación de valor | Invertir gasto cloud para maximizar retorno de negocio |

5. Guarda el archivo como:

```
~/finops-course/data/processed/diagnostico_finops_novatech_v1.xlsx
```

**Verificación:** El archivo existe con dos hojas nombradas correctamente y las cabeceras definidas.

---

### Paso 3: Mapear los síntomas al marco FinOps Foundation

**Objetivo:** Clasificar cada síntoma de NovaTech SRL según dominio, capacidad, fase y tipo de iniciativa del framework FinOps.

**Instrucciones:**

1. Regresa a la **Hoja 1 (Diagnóstico)** del archivo creado en el paso anterior.

2. Completa las filas 2 a 11 con el mapeo de cada síntoma. Utiliza la tabla de referencia de la Hoja 2 y el contenido de la lección 1.1 como guía. A continuación se muestra el mapeo completo esperado:

| ID | Descripción | Dominio | Capacidad | Fase | Tipo Iniciativa | Causa Raíz | Prioridad |
|----|-------------|---------|-----------|------|-----------------|-------------|-----------|
| S01 | Facturas con incrementos 25-40% sin correlación con tráfico | Understand Cloud Usage & Cost | Reporting & Analytics | Inform | Control de costos | Ausencia de monitoreo continuo y alertas de anomalías | Alta |
| S02 | Instancias EC2 sin carga en fines de semana | Optimize Cloud Usage & Cost | Workload Optimization | Optimize | Ahorro | Falta de políticas de apagado automático fuera de horario | Media |
| S03 | Imposibilidad de atribuir costos a productos | Understand Cloud Usage & Cost | Allocation | Inform | Control de costos | Sin estrategia de etiquetado ni jerarquía de cuentas | Alta |
| S04 | GPU instances de Data sin apagar | Optimize Cloud Usage & Cost | Workload Optimization | Optimize | Ahorro | Ausencia de gobernanza sobre recursos efímeros de alto costo | Media |
| S05 | Sin alertas de presupuesto | Quantify Business Value | Budgeting | Inform | Control de costos | No se han configurado umbrales ni notificaciones | Alta |
| S06 | Staging replica capacidad de producción | Optimize Cloud Usage & Cost | Architecting for Cloud | Optimize | Ahorro | Falta de right-sizing por ambiente | Media |
| S07 | Conflictos entre VP Engineering y CFO | Manage the FinOps Practice | FinOps Practice Operations | Operate | Generación de valor | Ausencia de modelo de responsabilidad compartida (accountability) | Alta |
| S08 | Reservas sin revisión de utilización | Optimize Cloud Usage & Cost | Rate Optimization | Optimize | Ahorro | Sin proceso de revisión periódica de compromisos | Media |
| S09 | Equipos desconocen costo de sus servicios | Manage the FinOps Practice | FinOps Education & Enablement | Inform | Control de costos | Falta de cultura de visibilidad de costos para ingeniería | Alta |
| S10 | Forecasting manual trimestral | Quantify Business Value | Planning & Forecasting | Inform | Generación de valor | Sin herramientas ni procesos de proyección automatizada | Media |

3. Ingresa cada fila en LibreOffice Calc siguiendo exactamente la estructura anterior.

4. Guarda el archivo (`Ctrl+S`).

**Salida esperada:** La Hoja 1 contiene 10 filas de datos completas (filas 2-11) más la fila de encabezado.

**Verificación:** Cada síntoma tiene asignado exactamente un dominio, una capacidad, una fase, un tipo de iniciativa, una causa raíz y una prioridad. No hay celdas vacías en las filas 2-11.

---

### Paso 4: Generar el resumen ejecutivo de madurez

**Objetivo:** Sintetizar los hallazgos en un resumen que evalúe la madurez FinOps actual de NovaTech SRL.

**Instrucciones:**

1. Crea una **Hoja 3** en el mismo archivo llamada `Resumen_Madurez`.

2. En la celda A1 escribe el título: `Evaluación de Madurez FinOps - NovaTech SRL`

3. Completa la siguiente tabla resumen a partir de la fila 3:

| Métrica | Valor |
|---------|-------|
| Total de síntomas identificados | 10 |
| Síntomas en fase Inform | 6 |
| Síntomas en fase Optimize | 4 |
| Síntomas en fase Operate | 0 |
| Prioridad Alta | 5 |
| Prioridad Media | 5 |
| Impacto financiero estimado mensual (USD) | $120,500 |
| Nivel de madurez estimado | Crawl (Gatear) |

4. En la celda A13, escribe el encabezado `Conclusión Diagnóstica:` y en A14 ingresa el siguiente texto:

```
NovaTech SRL se encuentra en nivel de madurez "Crawl" (Gatear). La mayoría de los
síntomas (60%) corresponden a la fase Inform, lo que indica que la organización
carece de visibilidad básica sobre su gasto cloud. No existen procesos operativos
maduros (0 síntomas en fase Operate). La causa raíz transversal es la ausencia de
una práctica FinOps formalizada con responsabilidades claras, herramientas de
visibilidad y cultura de accountability distribuida.
```

5. Guarda el archivo final:

```
~/finops-course/data/processed/diagnostico_finops_novatech_v1.xlsx
```

**Verificación:** El archivo contiene tres hojas (Diagnóstico, Referencia_Framework, Resumen_Madurez) y la conclusión diagnóstica está presente.

---

### Paso 5: Validar la integridad del archivo de salida

**Objetivo:** Confirmar que el archivo generado es válido y puede ser utilizado en laboratorios posteriores.

**Instrucciones:**

1. Cierra LibreOffice Calc.

2. Desde la terminal, verifica que el archivo existe y tiene un tamaño razonable:

```bash
ls -lh ~/finops-course/data/processed/diagnostico_finops_novatech_v1.xlsx
```

**Salida esperada:**

```
-rw-r--r-- 1 usuario usuario 12K [fecha] diagnostico_finops_novatech_v1.xlsx
```

El tamaño debe estar entre 8 KB y 25 KB aproximadamente.

3. (Opcional) Verifica la estructura del archivo con Python:

```bash
cd ~/finops-course
python3 -c "
import openpyxl
wb = openpyxl.load_workbook('data/processed/diagnostico_finops_novatech_v1.xlsx')
print('Hojas encontradas:', wb.sheetnames)
ws = wb['Diagnóstico']
print('Filas con datos:', ws.max_row)
print('Columnas:', ws.max_column)
assert ws.max_row >= 11, 'ERROR: Faltan filas de datos'
assert ws.max_column == 8, 'ERROR: Faltan columnas'
print('✓ Validación exitosa: archivo estructuralmente correcto')
"
```

**Salida esperada:**

```
Hojas encontradas: ['Diagnóstico', 'Referencia_Framework', 'Resumen_Madurez']
Filas con datos: 11
Columnas: 8
✓ Validación exitosa: archivo estructuralmente correcto
```

**Verificación:** El script Python confirma 3 hojas, 11 filas (1 encabezado + 10 datos) y 8 columnas.

## Validación y Pruebas

Para confirmar que el laboratorio se completó exitosamente, verifica los siguientes criterios:

| # | Criterio de validación | Estado |
|---|------------------------|--------|
| 1 | El archivo `diagnostico_finops_novatech_v1.xlsx` existe en `~/finops-course/data/processed/` | ☐ |
| 2 | La Hoja "Diagnóstico" contiene 10 síntomas mapeados con todas las columnas completas | ☐ |
| 3 | Cada síntoma tiene un dominio válido del framework FinOps (uno de los 4 dominios) | ☐ |
| 4 | Cada síntoma tiene una fase asignada (Inform, Optimize u Operate) | ☐ |
| 5 | Cada síntoma está clasificado por tipo de iniciativa (Control de costos, Ahorro, Generación de valor) | ☐ |
| 6 | La Hoja "Resumen_Madurez" contiene la evaluación con nivel "Crawl" | ☐ |
| 7 | El script de validación Python ejecuta sin errores (si se realizó el paso opcional) | ☐ |

## Solución de Problemas

### Problema 1: El archivo `novatech_symptoms.csv` no se encuentra

**Síntomas:** Al ejecutar `cat data/raw/novatech_symptoms.csv` se obtiene el error `No such file or directory`.

**Causa:** El instructor no ha distribuido los archivos de entrada o el estudiante no los ha colocado en la ruta correcta.

**Solución:**

1. Solicita el archivo al instructor.
2. Si tienes el archivo en otra ubicación (por ejemplo, `~/Downloads/`), cópialo:

```bash
cp ~/Downloads/novatech_symptoms.csv ~/finops-course/data/raw/
```

3. Si necesitas crear el archivo manualmente para continuar, usa el contenido CSV proporcionado en el Paso 1 de este laboratorio:

```bash
cat > ~/finops-course/data/raw/novatech_symptoms.csv << 'EOF'
id,sintoma,area_afectada,frecuencia,impacto_estimado_usd,reportado_por
S01,"Facturas mensuales con incrementos del 25-40% sin correlación con tráfico",Finanzas,Mensual,45000,CFO
S02,"Instancias EC2 en AWS ejecutándose sin carga útil durante fines de semana",Ingeniería,Semanal,12000,SRE Lead
S03,"Imposibilidad de atribuir costos a productos específicos (PayCore vs AnalyticsHub vs DevPortal)",Finanzas/Producto,Permanente,0,VP Engineering
S04,"Equipo de Data usa GPU instances para notebooks de exploración sin apagar al terminar",Data,Diaria,8500,Data Engineering Manager
S05,"No existen alertas de presupuesto; los excesos se detectan 3 semanas después",Finanzas,Mensual,22000,Controller
S06,"Ambiente staging replica la capacidad de producción sin justificación de carga",Platform,Permanente,18000,Platform Lead
S07,"Conflictos recurrentes entre VP Engineering y CFO sobre responsabilidad del gasto",Liderazgo,Quincenal,0,CEO
S08,"Reservas de Azure compradas hace 18 meses sin revisión de utilización actual",Finanzas/Platform,Permanente,15000,Cloud Architect
S09,"Equipos Frontend y Backend desconocen el costo de los servicios que despliegan",Ingeniería,Permanente,0,Engineering Managers
S10,"Forecasting de gasto cloud se realiza con spreadsheet manual una vez al trimestre",Finanzas,Trimestral,0,FP&A Analyst
EOF
```

---

### Problema 2: Error al abrir el archivo `.xlsx` con openpyxl en la validación

**Síntomas:** Al ejecutar el script de validación Python se obtiene `ModuleNotFoundError: No module named 'openpyxl'` o `InvalidFileException`.

**Causa:** El paquete `openpyxl` no está instalado, o el archivo fue guardado en formato `.ods` en lugar de `.xlsx`.

**Solución:**

1. Instala la dependencia si falta:

```bash
pip install openpyxl==3.1.2
```

2. Si el error es `InvalidFileException`, verifica el formato de guardado en LibreOffice Calc:
   - Abre el archivo en LibreOffice Calc
   - Ve a **Archivo → Guardar como...**
   - En el campo "Tipo de archivo" selecciona **Microsoft Excel 2007-365 (.xlsx)**
   - Confirma la ruta: `~/finops-course/data/processed/diagnostico_finops_novatech_v1.xlsx`
   - Haz clic en **Guardar** y acepta el formato xlsx si aparece un diálogo de confirmación

## Limpieza

Este laboratorio genera un archivo de salida que es **requerido por laboratorios posteriores**. **No elimines** el archivo generado:

```
~/finops-course/data/processed/diagnostico_finops_novatech_v1.xlsx
```

No se requiere limpieza adicional. No se crearon servicios, contenedores ni procesos en segundo plano.

## Resumen

En este laboratorio completaste las siguientes actividades:

| Actividad | Resultado |
|-----------|-----------|
| Revisión del caso NovaTech SRL | Identificaste 10 síntomas de problemas FinOps |
| Mapeo al framework FinOps | Clasificaste cada síntoma por dominio, capacidad y fase |
| Clasificación por tipo de iniciativa | Distinguiste entre control de costos, ahorro y generación de valor |
| Evaluación de madurez | Determinaste que NovaTech está en nivel "Crawl" con 60% de síntomas en fase Inform |

**Hallazgos clave de NovaTech SRL:**

- **5 síntomas de prioridad alta** requieren atención inmediata, todos relacionados con visibilidad y gobernanza básica.
- **$120,500 USD mensuales** de impacto financiero estimado por los problemas identificados.
- La organización necesita establecer primero los fundamentos (Inform) antes de optimizar (Optimize) o automatizar (Operate).
- El conflicto cultural entre ingeniería y finanzas (S07) es un indicador claro de la ausencia de práctica FinOps formalizada.

### Conexión con laboratorios siguientes

El archivo `diagnostico_finops_novatech_v1.xlsx` será utilizado como contexto organizacional en:

- **Lab 02-00-01:** Análisis de reportes de consumo cloud (requiere `novatech_billing_raw.csv`)
- **Lab 02-00-02:** Normalización y resumen de datos de facturación
- **Lab 03-00-01:** Estrategia de asignación de costos
- **Lab 04-00-01:** Caso diagnóstico integrado con roadmap 30-60-90 días

### Recursos adicionales

- [FinOps Foundation — Framework Overview](https://www.finops.org/framework/)
- [FinOps Foundation — Capabilities](https://www.finops.org/framework/capabilities/)
- [FinOps Foundation — Maturity Model](https://www.finops.org/framework/maturity-model/)
- Libro: *Cloud FinOps* — J.R. Storment & Mike Fuller (O'Reilly, 2nd Edition)

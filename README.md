# Fundamentos de FinOps

Este curso proporciona una base sólida y práctica para comprender, implementar y operar una práctica FinOps dentro de una organización. A partir del marco de referencia de la FinOps Foundation, los participantes aprenderán a analizar costos en la nube, asignar gasto por equipos o productos, construir reportes, definir KPIs, presupuestos y forecast, identificar anomalías, aplicar estrategias de optimización y establecer un modelo operativo colaborativo entre Finanzas, Ingeniería, Producto y Operaciones. El curso incorpora conceptos actuales como FOCUS, showback, chargeback, unit economics, compromisos de uso, costos compartidos, herramientas FinOps, automatización, madurez organizacional y consideraciones para SaaS, Kubernetes, multicloud e IA.

## Estructura

- `CapituloXX/README.md`: guía de laboratorio por capítulo.

## Lista de laboratorios

### Capítulo 1

- [Demostración 1. Diagnóstico inicial de problemas FinOps](Capitulo01/README.md#demostración-1-diagnóstico-inicial-de-problemas-finops)
  - Descripción: Ejercicio guiado para diagnosticar problemas FinOps en una organización ficticia, relacionando los hallazgos con los fundamentos, principios, dominios, capacidades, fases y scopes del Framework.
  - Duración estimada: 15 min

### Capítulo 2

- [Demostración 1. Lectura de un reporte de consumo cloud](Capitulo02/README.md#demostración-1-lectura-de-un-reporte-de-consumo-cloud)
  - Descripción: Interpretación guiada de un dataset de costos para reconocer consumo, gasto, uso, costo amortizado, costo neto, reservas y los principales componentes de facturación cloud.
  - Duración estimada: 25 min
- [Demostración 2. Análisis de tendencias de consumo por servicio](Capitulo02/README.md#demostración-2-análisis-de-tendencias-de-consumo-por-servicio)
  - Descripción: Ejercicio en hoja de cálculo o dataset provisto para analizar tendencias de consumo por servicio e identificar cómo los drivers de compute, storage, network y servicios gestionados afectan el gasto.
  - Duración estimada: 25 min

### Capítulo 3

- [Demostración 3. Asignar costos por producto, equipo y ambiente](Capitulo03/README.md#demostración-3-asignar-costos-por-producto-equipo-y-ambiente)
  - Descripción: Ejercicio de clasificación y distribución de costos por producto, equipo y ambiente, aplicando etiquetas, centros de costo y reglas para costos directos, compartidos, no asignados y no etiquetados.
  - Duración estimada: 30 min

### Capítulo 4

- [Demostración 4. Construir un tablero FinOps básico](Capitulo04/README.md#demostración-4-construir-un-tablero-finops-básico)
  - Descripción: Construcción de un dashboard FinOps básico que integre gasto, variación, tendencias y responsables para facilitar el seguimiento de KPIs, presupuesto, forecast y anomalías.
  - Duración estimada: 60 min

### Capítulo 5

- [Demostración 5. Backlog de optimización y priorización por impacto](Capitulo05/README.md#demostración-5-backlog-de-optimización-y-priorización-por-impacto)
  - Descripción: Elaboración de un backlog de optimización y priorización de oportunidades según ahorro, riesgo, esfuerzo y responsable, considerando uso, tarifas, arquitectura y licenciamiento.
  - Duración estimada: 30 min

### Capítulo 6

- [Demostración 6. Definición de roles, RACI y políticas mínimas](Capitulo06/README.md#demostración-6-definición-de-roles-raci-y-políticas-mínimas)
  - Descripción: Taller organizacional para definir roles, una matriz RACI y políticas FinOps mínimas de etiquetado, presupuestos, excepciones y aprobaciones, alineadas con el nivel de madurez.
  - Duración estimada: 25 min

### Capítulo 7

- [Demostración 7. Diseño de arquitectura de datos FinOps con FOCUS](Capitulo07/README.md#demostración-7-diseño-de-arquitectura-de-datos-finops-con-focus)
  - Descripción: Diseño de una arquitectura de referencia para datos FinOps con FOCUS, contemplando ingesta, normalización, enriquecimiento, calidad, trazabilidad, alertas y responsabilidades de gobierno.
  - Duración estimada: 30 min

### Capítulo 8

- [Demostración 8. Identificar drivers de costo en una solución moderna](Capitulo08/README.md#demostración-8-identificar-drivers-de-costo-en-una-solución-moderna)
  - Descripción: Caso aplicado para identificar drivers de costo en una solución moderna de SaaS, Kubernetes o IA, relacionando consumo, asignación, normalización y valor por caso de uso.
  - Duración estimada: 15 min

### Capítulo 9

- [Demostración 9. Caso integrador: diagnóstico, asignación, reporte y plan de optimización](Capitulo09/README.md#demostración-9-caso-integrador-diagnóstico-asignación-reporte-y-plan-de-optimización)
  - Descripción: Caso final de aplicación completa que integra diagnóstico, asignación de costos, reporte y elaboración de un plan de optimización, incorporando KPIs, responsables y siguientes pasos.
  - Duración estimada: 45 min

## Flujo de colaboración

- Trabajar en `changes_course`.
- Crear Pull Request hacia `main`.
- Merge por `Squash and merge`.

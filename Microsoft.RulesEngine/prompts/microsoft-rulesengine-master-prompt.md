# Microsoft.RulesEngine — PROMPT MAESTRO: Guía Técnica Definitiva

---

## Rol y Contexto Obligatorio

Asume el rol de **Principal Software Architect** con más de 18 años de experiencia en:

- Desarrollo backend avanzado en .NET (desde .NET Framework 4.x hasta .NET 8+)
- Diseño e implementación de motores de reglas en entornos empresariales críticos
- Sistemas financieros de alta disponibilidad: core bancario, scoring crediticio, prevención de fraude, cumplimiento regulatorio (AML/KYC/PLD)
- Arquitectura de microservicios, DDD, Clean Architecture, CQRS y Event Sourcing
- Optimización de performance en sistemas de alto throughput (millones de evaluaciones/día)
- Diseño de plataformas multi-tenant con reglas versionadas por país y producto

Tu audiencia es un **desarrollador backend senior** que necesita dominar Microsoft.RulesEngine a nivel productivo. No expliques como tutorial introductorio. Explica como mentor técnico senior escribiendo documentación interna para un equipo de arquitectura. Cada afirmación debe estar respaldada por análisis técnico, trade-offs reales y experiencia en producción.

---

## Estándar de Calidad Inquebrantable

Antes de redactar cada sección, valida internamente:

- ¿Un arquitecto enterprise encontraría esto suficientemente profundo?
- ¿Estoy explicando las implicancias reales en producción, no solo la teoría?
- ¿Estoy analizando trade-offs con criterio?
- ¿Estoy cubriendo edge cases que aparecen en sistemas reales?
- ¿Estoy considerando performance, concurrencia y escalabilidad?
- ¿Estoy considerando riesgos organizacionales y de mantenibilidad?
- ¿El código que incluyo es completo, compilable y realista?
- ¿Estoy aportando valor que no se encuentra fácilmente en la documentación oficial?

Si la respuesta a cualquiera es no, profundiza más antes de continuar.

**Prohibiciones absolutas:**

- No ser superficial.
- No resumir donde se requiere profundidad.
- No omitir detalles técnicos relevantes.
- No simplificar excesivamente.
- No escribir código incompleto o placeholder.
- No usar frases genéricas como "depende del caso" sin explicar de qué depende concretamente.

---

## Estructura Obligatoria del Documento

Genera un documento Markdown profesional extenso, organizado en los siguientes **14 módulos numerados**. Cada módulo debe desarrollarse con profundidad extrema.

---

### MÓDULO 1 — Fundamentos y Filosofía Arquitectónica

Desarrollar en profundidad:

- Qué es Microsoft.RulesEngine: definición técnica precisa, no marketing.
- Qué problema arquitectónico resuelve concretamente y por qué existe.
- Filosofía de diseño: motor basado en expresiones (expression-based) vs motor basado en inferencia (inference-based). Explicar las implicancias profundas de esta decisión.
- Cuándo un motor de reglas aporta valor real vs cuándo es sobreingeniería.
- Historia y evolución del proyecto. Estado actual del repositorio, actividad de mantenimiento, versión estable.
- Modelo mental correcto para pensar en reglas externalizadas.
- Relación entre reglas de negocio, lógica de dominio y políticas operativas.
- Análisis crítico: fortalezas genuinas y debilidades reales del motor.

Incluir:

- Ejemplo introductorio completo en C# que demuestre el flujo básico end-to-end.
- JSON de workflow correspondiente.
- Explicación línea por línea del flujo de ejecución.

---

### MÓDULO 2 — Arquitectura Interna en Profundidad

Desarrollar cada componente interno con detalle técnico:

- **Workflow**: estructura, propósito, relación con conjuntos de reglas, estrategia de agrupación.
- **Rule**: todos los campos disponibles (RuleName, SuccessEvent, ErrorMessage, ErrorType, RuleExpressionType, Expression, Actions, Enabled, etc.). Explicar cada campo, su comportamiento por defecto y sus implicancias.
- **RuleParameter**: cómo se mapean inputs al motor, tipado dinámico, convenciones de nombres, navegación de propiedades anidadas.
- **RuleResultTree**: estructura completa del resultado, cómo navegar resultados anidados, cómo extraer información útil.
- **RuleExpressionType**: diferencias profundas entre `LambdaExpression` y `RuleExpressionType` (AND/OR para reglas anidadas). Cuándo usar cada uno, implicancias en composición.
- **Expression Trees internos**: cómo el motor compila strings a Expression Trees de .NET, costos de compilación, caché de compilaciones.
- **Flujo de ejecución interno paso a paso**: desde `ExecuteAllRulesAsync()` hasta la generación del `RuleResultTree`. Incluir diagrama conceptual en texto.
- **Manejo de null**: cómo propaga nulls en expresiones, null-conditional en expresiones dinámicas, riesgos de NullReferenceException.
- **Manejo de excepciones internas**: qué ocurre cuando una expresión lanza excepción, cómo se captura, cómo afecta al resultado.
- **Caché interno de reglas compiladas**: mecanismo, ciclo de vida, invalidación, impacto en memoria.

---

### MÓDULO 3 — Configuración Avanzada y Customización

Desarrollar con código completo:

- Instalación vía NuGet. Versiones relevantes. Dependencias transitivas.
- **ReSettings** (configuración del motor): cada propiedad disponible, valores por defecto, impacto de cada configuración.
  - `CustomTypes`: cómo registrar tipos personalizados accesibles desde expresiones. Ejemplos con clases de utilidad, enums, helpers estáticos.
  - `CustomActions`: registro de acciones personalizadas ejecutables como resultado de reglas.
  - `NestedRuleExecutionMode`: modos disponibles, diferencias y cuándo usar cada uno.
  - `EnableExceptionAsErrorMessage`: comportamiento, riesgos.
- **Custom Functions (métodos personalizados)**: registro de funciones C# invocables desde expresiones JSON. Patrón completo de registro, restricciones de firma, manejo de sobrecarga, ejemplos complejos.
- **Custom Types**: registro de tipos personalizados para uso en expresiones. Clases estáticas con métodos de utilidad. Patrones de extensión.
- **Integración con ASP.NET Core**:
  - Registro en `IServiceCollection`.
  - Ciclo de vida recomendado (Singleton vs Scoped vs Transient). Justificar.
  - Patrón de inyección en servicios de dominio.
- **Inyección de Dependencias**: patrones para inyectar el motor en capas de aplicación, integración con Mediator pattern.
- **Carga de reglas desde JSON**: lectura desde archivos, desde configuración, desde base de datos (EF Core, Dapper), desde servicios remotos.
- **Recarga dinámica en runtime**: estrategias de hot-reload sin reiniciar la aplicación. Patrón con FileSystemWatcher, polling, event-driven.
- **Versionado de reglas**: esquemas de versionado, almacenamiento multiversión, rollback, auditoría de cambios.
- **Logging y observabilidad**: integración con ILogger, logging estructurado, métricas de evaluación, trazabilidad con OpenTelemetry/Application Insights.

---

### MÓDULO 4 — Diseño de Reglas: Casos Básicos a Intermedios

Desarrollar con ejemplos JSON completos y código C# correspondiente:

- **Estructura completa de un Workflow JSON**: anatomía campo por campo con JSON anotado.
- **Reglas simples con LambdaExpression**: comparaciones, operadores lógicos, operadores aritméticos, acceso a propiedades anidadas.
- **Múltiples inputs**: cómo pasar varios objetos como RuleParameter, convenciones de nombrado, acceso cruzado entre inputs.
- **LocalParams (ScopedParams)**: definición, propósito, sintaxis, evaluación secuencial, dependencia entre params. Ejemplos de cálculos intermedios reutilizables.
- **GlobalParams**: diferencias con LocalParams, alcance, cuándo usar cada uno.
- **Reglas con Actions**: OnSuccess y OnFailure. Tipos de acciones: OutputExpression, EvaluateRule. Cómo extraer valores calculados del resultado.
- **Reglas habilitadas/deshabilitadas**: campo Enabled, estrategias de feature flags con reglas.
- **Mensajes de error y eventos de éxito**: SuccessEvent, ErrorMessage, ErrorType. Cómo diseñarlos para consumo en APIs.

Casos prácticos completos:

- Validación de solicitud de registro con múltiples campos.
- Motor de descuentos por tipo de cliente y monto de compra.
- Evaluación de elegibilidad para un producto financiero.
- Cálculo de impuestos dinámicos por jurisdicción.

---

### MÓDULO 5 — Diseño de Reglas: Casos Avanzados

Desarrollar con profundidad extrema:

- **Reglas anidadas (Nested Rules)**: composición AND/OR, profundidad de anidamiento, estrategias de organización, legibilidad del JSON, impacto en depuración.
- **ScopedParams avanzados**: encadenamiento de cálculos, acceso a resultados intermedios, patrones de transformación de datos dentro de la evaluación.
- **Evaluación sobre colecciones**: cómo evaluar reglas contra listas de elementos, patrones con LINQ en expresiones, `Any()`, `All()`, `Count()`, `Where()` dentro de expresiones dinámicas. Limitaciones.
- **Acceso a métodos de extensión**: cómo hacer visibles métodos de extensión en expresiones, registro de tipos que los contienen.
- **Expresiones complejas**: operaciones con fechas, strings, regex, conversiones de tipo, operaciones matemáticas, operador ternario en expresiones.
- **Dependencia entre reglas**: cómo orquestar reglas donde la salida de una alimenta la entrada de otra. Patrones de encadenamiento multi-etapa.
- **Composición de workflows**: estrategias para dividir lógica compleja en múltiples workflows ejecutados secuencialmente, con paso de contexto entre ellos.
- **Reglas dinámicas generadas en runtime**: construcción programática de objetos Workflow/Rule sin JSON.

---

### MÓDULO 6 — Casos Empresariales del Sector Financiero

Para cada caso, desarrollar: contexto de negocio, modelo de datos, JSON de reglas completo, código C# de ejecución, manejo de resultados, consideraciones de producción.

**Caso 1 — Motor de Aprobación Crediticia:**
- Evaluación de ingreso mínimo, historial crediticio, relación deuda/ingreso, antigüedad laboral, edad.
- Reglas con múltiples niveles de aprobación (automática, revisión manual, rechazo).
- ScopedParams para cálculos intermedios (ratio DTI, score ponderado).

**Caso 2 — Scoring Dinámico (Credit Scoring):**
- Cálculo de puntaje basado en múltiples variables ponderadas.
- Rangos de score con diferentes acciones (aprobado, aprobado condicional, rechazado, blacklist).
- Versionado de modelo de scoring por producto y país.

**Caso 3 — Evaluación de Riesgo (Risk Assessment):**
- Análisis multi-dimensional: riesgo de mercado, riesgo operacional, riesgo crediticio.
- Reglas compuestas con umbrales dinámicos por segmento de cliente.
- Orquestación de evaluación secuencial con corte temprano (early exit).

**Caso 4 — Prevención de Fraude (Fraud Detection):**
- Detección de patrones anómalos: monto inusual, frecuencia atípica, geolocalización sospechosa, dispositivo desconocido.
- Reglas con ventanas de tiempo (transacciones en últimas 24h).
- Niveles de alerta: bajo, medio, alto, bloqueo automático.
- Integración con flujos de revisión manual.

**Caso 5 — Validaciones AML/KYC (Anti Money Laundering / Know Your Customer):**
- Reglas de umbral de transacción (CTR - Currency Transaction Reports).
- Detección de structuring (operaciones fraccionadas).
- Validación contra listas de sanciones (PEP, OFAC).
- Reglas regulatorias versionadas por jurisdicción (país, estado).
- Auditoría completa de evaluación para compliance.

**Caso 6 — Motor de Comisiones Bancarias:**
- Cálculo dinámico de comisiones por tipo de transacción, canal, producto, segmento.
- Reglas de exención y descuento por programa de fidelidad.
- Escalas progresivas de comisión.

**Caso 7 — Cálculo de Intereses Dinámicos:**
- Determinación de tasa de interés por perfil de riesgo, plazo, monto, producto.
- Aplicación de spreads regulatorios.
- Ajustes por condiciones de mercado (parámetros externos).

**Caso 8 — Evaluación de Límites de Crédito:**
- Asignación de límite basada en ingreso, historial, productos existentes, exposición total.
- Reglas de incremento y decremento automático.
- Restricciones regulatorias por tipo de producto.

**Caso 9 — Orquestación de Reglas Multi-Etapa:**
- Pipeline completo: pre-validación → scoring → pricing → aprobación → condiciones.
- Paso de contexto entre etapas.
- Manejo de corte temprano y bypass condicional.
- Logging de cada etapa para auditoría.

---

### MÓDULO 7 — Edge Cases, Problemas Reales y Escenarios Complejos

Analizar cada escenario con código que lo reproduzca y solución:

- **Null propagation**: qué ocurre cuando una propiedad intermedia es null en una cadena de navegación. Cómo protegerse. Diferencias con null-conditional de C#.
- **Excepciones dentro de expresiones**: división por cero, parsing fallido, cast inválido. Cómo el motor las maneja. Cómo controlar el comportamiento.
- **Problemas de concurrencia**: qué ocurre con evaluaciones simultáneas. Thread safety del motor. Compartir instancia entre threads. Riesgos de mutación de estado.
- **Problemas de serialización**: deserialización de JSON con caracteres especiales, escape de comillas en expresiones, Unicode, expresiones multilínea. Problemas con System.Text.Json vs Newtonsoft.
- **Expresiones dinámicas malformadas**: qué pasa cuando el JSON contiene expresiones con errores de sintaxis. Detección temprana vs fallo en runtime.
- **Reglas inconsistentes o contradictorias**: dos reglas que se contradicen, orden de evaluación, determinismo del resultado.
- **Cambios dinámicos en runtime**: riesgos de modificar reglas mientras se están evaluando. Race conditions. Estrategias de inmutabilidad.
- **Reglas con efectos colaterales**: expresiones que intentan mutar estado. Por qué es peligroso y cómo prevenirlo.
- **Evaluación sobre colecciones vacías**: comportamiento de Any(), All(), Count() sobre colecciones null o vacías.
- **Límites del parser de expresiones**: expresiones demasiado complejas, anidamiento excesivo, longitud máxima práctica.
- **Problemas de mantenibilidad a largo plazo**: crecimiento descontrolado de reglas, pérdida de trazabilidad, reglas huérfanas.
- **Casos donde NO conviene usar RulesEngine**: lógica que requiere inferencia, backward chaining, reglas con estado acumulativo, procesamiento en streaming, lógica que necesita acceso a I/O.

---

### MÓDULO 8 — Performance, Escalabilidad y Optimización

Analizar con nivel de arquitecto de performance:

- **Costo de compilación vs costo de evaluación**: desglose de dónde se gasta el tiempo. Primera ejecución vs ejecuciones subsecuentes.
- **Caché de reglas compiladas**: mecanismo interno, cuándo se invalida, cómo forzar recompilación, impacto en memoria con miles de workflows.
- **Comparación de rendimiento**:
  - Microsoft.RulesEngine vs código imperativo nativo.
  - Microsoft.RulesEngine vs NRules.
  - Overhead real medido en microsegundos para diferentes complejidades de reglas.
- **Impacto en CPU**: profiling de evaluación, hotspots típicos, expresiones costosas.
- **Impacto en memoria**: allocations por evaluación, presión en GC, objetos de resultado, retención de Expression Trees compilados.
- **Estrategias de optimización**:
  - Reducir complejidad de expresiones.
  - Pre-calcular valores en ScopedParams.
  - Evaluar workflows selectivamente.
  - Implementar early exit.
  - Pool de objetos de resultado.
- **Escenarios de alto throughput**: evaluación masiva batch, paralelización, particionamiento de reglas.
- **Benchmarks conceptuales**: tabla de referencia con tiempos esperados para diferentes cargas (10, 100, 1000, 10000 reglas).
- **Cuándo el motor se convierte en bottleneck**: señales de alerta, métricas a monitorear, umbrales de decisión.

---

### MÓDULO 9 — Testing Unitario e Integración

Desarrollar con código completo en xUnit:

- **Estrategia de testing para reglas**: qué testear, a qué nivel, con qué granularidad.
- **Tests unitarios de reglas individuales**: fixture, setup de motor de pruebas, assertions sobre RuleResultTree.
- **Tests de reglas con ScopedParams**: verificar cálculos intermedios.
- **Tests de reglas anidadas**: validar composición AND/OR.
- **Tests de custom functions**: mockear dependencias de funciones personalizadas.
- **Tests de integración de workflows completos**: escenarios end-to-end con múltiples reglas.
- **Testing con datos parametrizados**: Theory/InlineData para cubrir múltiples escenarios con una sola definición.
- **Tests de regresión**: estrategia para detectar cambios inesperados en comportamiento al modificar reglas.
- **Testing masivo automatizado**: generación de datos de prueba, evaluación masiva, detección de anomalías.
- **Testing de performance**: benchmarks con BenchmarkDotNet para reglas críticas.
- **Mocking de dependencias externas**: cuando las custom functions acceden a servicios externos.
- **Tests de contrato de JSON de reglas**: validar estructura de JSON antes de deployment.
- **Pipeline de CI/CD**: integración de tests de reglas en pipelines de build. Estrategia de gates.

---

### MÓDULO 10 — Seguridad

Desarrollar con enfoque enterprise:

- **Riesgos de inyección de código**: el motor evalúa expresiones string como código. Implicancias de seguridad si el JSON es editable por usuarios externos o APIs públicas.
- **Sandbox de expresiones**: qué puede y qué no puede hacer una expresión. Acceso a System, IO, Reflection, Process desde expresiones.
- **Restricción de tipos accesibles**: cómo limitar los tipos disponibles en expresiones. Whitelist vs blacklist de tipos.
- **Validación de reglas antes de ejecución**: parseo y validación previa de expresiones para detectar intenciones maliciosas.
- **Seguridad en almacenamiento de reglas**: cifrado en reposo, control de acceso, auditoría de modificaciones.
- **Seguridad en transporte de reglas**: APIs de gestión de reglas, autenticación, autorización por rol.
- **Principio de mínimo privilegio**: el motor solo debe tener acceso a los datos estrictamente necesarios.
- **Auditoría de evaluación**: registro inmutable de qué reglas se evaluaron, con qué datos, qué resultado produjeron, cuándo y por quién.
- **Compliance**: cómo cumplir requisitos de SOX, PCI-DSS, GDPR al usar reglas externalizadas.
- **Protección contra DoS**: expresiones diseñadas para consumir recursos (loops infinitos conceptuales, expresiones recursivas, regex catastrophic backtracking).

---

### MÓDULO 11 — Comparaciones Técnicas

Desarrollar tabla comparativa y análisis narrativo profundo:

- **Microsoft.RulesEngine vs NRules**: paradigma, capacidades, performance, curva de aprendizaje, madurez, comunidad, soporte para forward/backward chaining, persistencia, extensibilidad.
- **Microsoft.RulesEngine vs Drools (Java)**: comparación cross-platform, funcionalidades equivalentes, gaps.
- **Microsoft.RulesEngine vs código imperativo (if/else, Strategy Pattern, Specification Pattern)**: cuándo cada enfoque es superior, análisis de mantenibilidad con 10, 50, 200, 1000 reglas.
- **Microsoft.RulesEngine vs FluentValidation**: diferencias de propósito, cuándo se complementan, cuándo compiten.
- **Microsoft.RulesEngine vs Azure Logic Apps / AWS Step Functions**: reglas locales vs orquestación cloud, trade-offs.
- **Matriz de decisión**: tabla con criterios (performance, flexibilidad, mantenibilidad, curva, ecosistema, costo) y puntaje para cada motor.
- **Guía de selección**: diagrama de decisión conceptual para elegir el motor correcto según el escenario.

---

### MÓDULO 12 — Integración Arquitectónica (Clean Architecture, DDD, CQRS, Microservicios)

Desarrollar con diagramas conceptuales y código:

- **Ubicación del motor en Clean Architecture**: en qué capa vive, cómo se abstrae, interfaces de puerto/adaptador. Diagrama de capas con RulesEngine.
- **Integración con DDD**: el motor como Domain Service, relación con Aggregates, cómo las reglas externalizadas se relacionan con invariantes de dominio. Cuándo las reglas pertenecen al dominio vs cuándo son políticas de aplicación.
- **Integración con CQRS**: reglas en el command side vs query side. Reglas como parte de la validación de comandos. Reglas como parte de la proyección de read models.
- **Integración con Mediator (MediatR)**: reglas en pipeline behaviors, reglas en handlers, patrón de ejecución pre/post comando.
- **Integración con microservicios**:
  - Servicio centralizado de reglas vs reglas embebidas por servicio.
  - API de gestión de reglas (CRUD + versionado).
  - Distribución de reglas vía eventos (Event-Driven Architecture).
  - Caché distribuida de reglas (Redis).
  - Consistencia eventual de reglas entre servicios.
- **Integración con APIs REST/gRPC**: endpoint de evaluación de reglas, modelo de request/response, versionado de API.
- **Patrón de gobernanza**: quién crea reglas, quién las aprueba, quién las despliega. Flujo de promoción (dev → staging → prod).

---

### MÓDULO 13 — Buenas Prácticas y Anti-Patrones

**Buenas prácticas (desarrollar cada una con justificación):**

- Nombrado consistente de workflows y reglas.
- Documentación de reglas dentro del JSON (campos descriptivos).
- Versionado semántico de conjuntos de reglas.
- Separación de reglas por dominio/bounded context.
- Uso de ScopedParams para cálculos intermedios en vez de expresiones monolíticas.
- Testing exhaustivo de cada regla en aislamiento.
- Validación del JSON de reglas en CI/CD.
- Monitoreo de performance de evaluación en producción.
- Feature flags para reglas nuevas.
- Rollback instantáneo de reglas.
- Inmutabilidad de reglas en evaluación.
- Logging estructurado de cada evaluación.

**Anti-patrones (desarrollar cada uno con ejemplo de lo que NO hacer):**

- Poner lógica de negocio crítica que necesita transaccionalidad dentro de reglas.
- Crear expresiones excesivamente largas e ilegibles.
- No testear reglas modificadas por negocio.
- Exponer el JSON de reglas a usuarios sin validación.
- Usar el motor para lógica que cambia raramente (sobreingeniería).
- Acoplar las reglas a la estructura interna de entidades de dominio.
- Ignorar el impacto de performance de reglas complejas.
- No versionar reglas.
- Mezclar lógica de orquestación con lógica de evaluación en las mismas reglas.
- Asumir que el orden de evaluación es determinista sin verificarlo.
- No manejar resultados parciales o errores de evaluación.
- Hardcodear valores que deberían ser parámetros.

---

### MÓDULO 14 — Checklist Final de Adopción y Producción

Generar un checklist detallado y accionable organizado en fases:

**Fase 1 — Evaluación:**
- [ ] ¿Las reglas de negocio cambian con frecuencia suficiente para justificar un motor?
- [ ] ¿El equipo tiene la madurez para gestionar reglas externalizadas?
- [ ] ¿Se ha evaluado alternativas (NRules, código imperativo, Specification Pattern)?
- [ ] ¿Se ha hecho un POC con un caso real?

**Fase 2 — Diseño:**
- [ ] ¿Se ha definido la arquitectura de almacenamiento de reglas?
- [ ] ¿Se ha definido el esquema de versionado?
- [ ] ¿Se ha definido el modelo de gobernanza?
- [ ] ¿Se ha definido la estrategia de testing?
- [ ] ¿Se han registrado todos los Custom Types y Custom Functions necesarios?
- [ ] ¿Se ha definido la integración con el stack existente (DI, logging, observabilidad)?

**Fase 3 — Implementación:**
- [ ] ¿Las reglas están testeadas unitariamente?
- [ ] ¿Existe validación de JSON en CI/CD?
- [ ] ¿Se ha verificado thread safety?
- [ ] ¿Se ha verificado performance bajo carga esperada?
- [ ] ¿Se han cubierto edge cases (nulls, colecciones vacías, datos inválidos)?

**Fase 4 — Producción:**
- [ ] ¿Existe monitoreo de tiempos de evaluación?
- [ ] ¿Existe alerting ante fallos de evaluación?
- [ ] ¿Existe mecanismo de rollback de reglas?
- [ ] ¿Existe auditoría de cambios en reglas?
- [ ] ¿Existe proceso de revisión antes de desplegar reglas nuevas?
- [ ] ¿Se ha documentado el runbook operativo?

**Fase 5 — Evolución:**
- [ ] ¿Existe proceso de limpieza de reglas obsoletas?
- [ ] ¿Se revisa periódicamente la performance?
- [ ] ¿Se actualiza la librería con criterio (changelog, breaking changes)?
- [ ] ¿Se mide el ROI de mantener el motor vs simplificar?

Incluir recomendaciones estratégicas finales: perfil ideal de proyecto, casos donde es mala elección, roadmap de adopción empresarial sugerido con timeline.

---

## Reglas Obligatorias de Generación

1. El documento debe ser **extenso y completo**. No hay límite de longitud. Si necesitas continuar, hazlo automáticamente.
2. Formato **Markdown profesional** con jerarquía clara de headers.
3. Todo el **código C# debe ser completo, compilable y realista**. No usar pseudocódigo ni fragmentos incompletos.
4. Todo el **JSON debe ser completo y válido**, listo para ser usado.
5. Las explicaciones deben tener **profundidad de libro técnico especializado**.
6. Incluir **diagramas conceptuales en texto** (usando ASCII art o descripciones de flujo) donde aporten claridad.
7. Cada decisión arquitectónica debe estar **justificada con trade-offs**.
8. Los **ejemplos financieros deben ser realistas**, no triviales ni genéricos.
9. No incluir disclaimers, meta-comentarios ni explicaciones sobre lo que estás generando.
10. No incluir secciones de "introducción al documento" ni "sobre esta guía".
11. Ir directamente al contenido técnico desde el primer párrafo.
12. El tono debe ser **técnico, directo y autoritativo**. Sin exceso de formalidades.
13. Asumir siempre que el lector tiene experiencia sólida en C# y .NET.
14. Priorizar **conocimiento que no se encuentra fácilmente** en la documentación oficial o en tutoriales superficiales.

---

## Instrucción Final

Genera ahora el documento técnico completo en Markdown dentro de la carpeta docs, con el nombre Microsoft-NRules-Documentation.

No generes explicación adicional.
No hagas resumen previo.
No hagas índice separado.
No hagas meta-comentarios.
Solo genera el contenido técnico completo, módulo por módulo, con la profundidad exigida.

Comienza directamente con el MÓDULO 1.

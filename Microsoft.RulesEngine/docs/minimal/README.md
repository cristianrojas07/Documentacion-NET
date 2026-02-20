# Microsoft.RulesEngine — Guía Maestra Técnica Enterprise

> **Nivel**: Arquitecto Principal / Staff Engineer  
> **Audiencia**: Equipos de arquitectura, leads técnicos, ingenieros senior  
> **Versión del motor analizada**: Microsoft.RulesEngine 5.x (NuGet)  
> **Plataforma**: .NET 8+

---

## Tabla de Contenidos

1. [Fundamentos y Filosofía Arquitectónica](#1-fundamentos-y-filosofía-arquitectónica)
2. [Arquitectura Interna en Profundidad](#2-arquitectura-interna-en-profundidad)
3. [Configuración Avanzada y Customización](#3-configuración-avanzada-y-customización)
4. [Diseño Avanzado de Reglas Empresariales](#4-diseño-avanzado-de-reglas-empresariales)
5. [Performance y Escalabilidad Real](#5-performance-y-escalabilidad-real)
6. [Patrones de Diseño y Gobernanza](#6-patrones-de-diseño-y-gobernanza)
7. [Edge Cases Reales y Riesgos](#7-edge-cases-reales-y-riesgos)
8. [Conclusión Estratégica](#8-conclusión-estratégica)

---

# 1. Fundamentos y Filosofía Arquitectónica

## 1.1 Qué es Microsoft.RulesEngine

Microsoft.RulesEngine es un motor de evaluación de reglas de negocio basado en **expresiones** (expression-based), diseñado para externalizar lógica condicional desde código compilado hacia definiciones declarativas en formato JSON. No es un sistema de inferencia, no mantiene estado entre evaluaciones, y no implementa algoritmos de encadenamiento como Rete.

Su propósito arquitectónico central es **desacoplar la lógica de decisión del código de aplicación**, permitiendo que reglas de negocio sean modificadas, versionadas y desplegadas independientemente del ciclo de vida del software.

En términos concretos, el motor toma:
- Una **definición de workflow** (conjunto de reglas en JSON)
- **Inputs tipados** (objetos C#)
- Evalúa cada regla compilando expresiones C# dinámicas mediante **System.Linq.Expressions**
- Retorna un **árbol de resultados** (`RuleResultTree`) con el estado de cada regla

### Modelo Mental Correcto

No pensar en RulesEngine como un "if/else externalizado". Pensarlo como un **evaluador declarativo de predicados con capacidad de composición**, donde cada predicado es una expresión Lambda compilada en runtime que opera sobre inputs tipados.

```csharp
// Lo que parece ser un simple "if"
// "Expression": "input1.Age > 18 && input1.Country == \"AR\""

// Internamente se compila a:
Expression<Func<RuleInput, bool>> compiled = (input1) => input1.Age > 18 && input1.Country == "AR";
```

## 1.2 El Problema Arquitectónico que Resuelve

En sistemas enterprise, la lógica de negocio muta con frecuencia desproporcionada respecto al código estructural. Un sistema de pricing puede cambiar sus reglas semanalmente. Un motor de compliance debe adaptarse a regulaciones trimestrales. Un sistema de underwriting ajusta sus criterios según el apetito de riesgo del mercado.

El problema no es técnico en esencia — es **organizacional y operacional**:

| Aspecto | Código Imperativo | RulesEngine |
|---|---|---|
| Cambio de regla | PR → Review → CI/CD → Deploy | Cambio JSON → Recarga (sin deploy) |
| Responsable del cambio | Desarrollador | Analista de negocio + validación |
| Riesgo de regresión | Alto (todo recompila) | Aislado (solo regla afectada) |
| Auditoría | Diff en Git | Versionado explícito + metadata |
| Time-to-market | Días/semanas | Minutos/horas |
| Testabilidad aislada | Requiere mocking complejo | Input → Regla → Resultado determinístico |

### Cuándo el Problema Justifica el Motor

RulesEngine se justifica cuando confluyen al menos 3 de estos factores:

1. **Alta frecuencia de cambio** en lógica condicional (>1 cambio/mes)
2. **Múltiples stakeholders no técnicos** que definen o validan reglas
3. **Necesidad de auditoría** sobre qué regla se evaluó y cuándo
4. **Versionado independiente** de reglas respecto al código
5. **Ejecución determinística** de predicados sobre datos estructurados

## 1.3 Expression-Based vs Inference-Based: Diferencias Fundamentales

Esta distinción es **crítica** para decisiones arquitectónicas correctas.

### Expression-Based Engine (Microsoft.RulesEngine)

- Evalúa cada regla **independientemente** contra un conjunto de inputs
- No mantiene estado entre evaluaciones (*stateless*)
- No infiere hechos nuevos a partir de resultados
- El orden de evaluación es **declarativo** (definido por el workflow)
- Cada evaluación es una función pura: `f(inputs) → results`
- Complejidad temporal: **O(n)** donde n = número de reglas (lineal)

### Inference-Based Engine (NRules, Drools)

- Utiliza **working memory** (memoria de trabajo) con hechos
- Aplica algoritmo **Rete** para optimizar matching de patrones
- Los resultados de una regla pueden **activar otras reglas** (forward chaining)
- Mantiene estado entre ciclos de inferencia
- El orden de ejecución es **emergente** (determinado por el motor)
- Complejidad temporal: variable, puede ser exponencial en grafos complejos

### Tabla Comparativa Profunda

| Característica | Microsoft.RulesEngine | NRules | Drools |
|---|---|---|---|
| Paradigma | Expression-based | Inference (Rete) | Inference (Rete/PHREAK) |
| Lenguaje de reglas | C# Expressions (string) | C# fluent API | DRL (propio) |
| Estado entre evals | No | Sí (Working Memory) | Sí (Working Memory) |
| Forward Chaining | No nativo | Sí | Sí |
| Backward Chaining | No | No | Sí |
| Runtime | .NET (System.Linq.Expressions) | .NET | JVM |
| Definición dinámica | JSON en runtime | Código compilado | DRL en runtime |
| Curva de aprendizaje | Baja-Media | Media-Alta | Alta |
| Overhead por regla | ~0.01-0.1ms (compilada) | ~0.1-1ms | Variable |
| Escalabilidad horizontal | Trivial (stateless) | Compleja (state sync) | Compleja |
| Ideal para | Evaluación de condiciones | Sistemas expertos | Sistemas expertos complejos |

### Cuándo Elegir Cada Uno

**Microsoft.RulesEngine**: Validaciones de negocio, eligibilidad, pricing simple a medio, compliance con reglas independientes, cualquier escenario donde las reglas no generan hechos nuevos.

**NRules**: Sistemas donde reglas interdependientes deben inferir conclusiones, diagnóstico médico, configuradores de producto complejos, sistemas expertos puros en .NET.

**Drools**: Ecosistema JVM, reglas de altísima complejidad con backward chaining, integración con plataformas Red Hat/IBM.

## 1.4 Decisiones de Diseño Internas

El equipo de Microsoft tomó decisiones deliberadas que definen las capacidades y limitaciones:

### Decisión 1: Expresiones como Strings C#

Las reglas se definen como strings que contienen expresiones C# válidas. Esto permite familiaridad para desarrolladores .NET pero introduce riesgos de seguridad (code injection) y limita validación estática.

```json
{
  "RuleName": "CheckEligibility",
  "Expression": "input1.CreditScore >= 700 && input1.DebtToIncomeRatio < 0.43m"
}
```

**Trade-off**: Flexibilidad máxima vs. seguridad. No hay sandboxing nativo.

### Decisión 2: Compilación Lazy con Cache

Las expresiones se compilan a delegates la primera vez que se ejecutan y se cachean. Esto amortiza el costo de compilación pero introduce un cold-start penalty.

**Trade-off**: Performance en estado estable vs. latencia en primera ejecución.

### Decisión 3: Evaluación Independiente por Regla

Cada regla se evalúa independientemente. No hay propagación de resultados entre reglas del mismo workflow (salvo mediante LocalParams).

**Trade-off**: Simplicidad y predictibilidad vs. poder expresivo. Se sacrifica inferencia por determinismo.

### Decisión 4: Resultado como Árbol

El resultado no es un booleano simple sino un `RuleResultTree` que contiene información de evaluación por cada regla y sub-regla.

**Trade-off**: Riqueza de información vs. overhead de memoria y complejidad de consumo.

## 1.5 Flujo Interno Detallado de Ejecución

```
┌─────────────────────────────────────────────────────────────────┐
│                    FLUJO DE EJECUCIÓN INTERNO                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. ExecuteAllRulesAsync(workflowName, inputs[])                │
│     │                                                           │
│  2. Buscar Workflow por nombre en diccionario interno            │
│     │                                                           │
│  3. Para cada Rule en Workflow.Rules:                            │
│     │                                                           │
│     ├─ 3a. Resolver RuleExpressionType                          │
│     │       ├─ LambdaExpression → compilar/cachear expresión     │
│     │       └─ RuleFunc → usar delegate pre-registrado           │
│     │                                                           │
│     ├─ 3b. Evaluar LocalParams (si existen)                     │
│     │       └─ Compilar y ejecutar cada LocalParam               │
│     │           como variable intermedia                         │
│     │                                                           │
│     ├─ 3c. Compilar Expression a Expression<Func<..., bool>>    │
│     │       └─ System.Linq.Expressions.Expression.Lambda()       │
│     │       └─ .Compile() → Func<..., bool> cacheado            │
│     │                                                           │
│     ├─ 3d. Invocar delegate compilado con inputs                │
│     │       └─ Resultado: true/false                             │
│     │                                                           │
│     ├─ 3e. Si tiene ChildRules → evaluar recursivamente         │
│     │                                                           │
│     ├─ 3f. Ejecutar OnSuccess/OnFailure Actions                 │
│     │                                                           │
│     └─ 3g. Construir RuleResultTree node                        │
│                                                                 │
│  4. Agregar todos los RuleResultTree en List<RuleResultTree>    │
│     │                                                           │
│  5. Retornar resultado completo                                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## 1.6 Expression Trees: Construcción y Compilación

El núcleo del motor es la compilación dinámica de expresiones C#. Internamente:

```csharp
// Paso 1: El string de expresión se parsea
string expression = "input1.CreditScore >= 700";

// Paso 2: Se construye un ParameterExpression por cada input
var param = Expression.Parameter(typeof(CreditApplication), "input1");

// Paso 3: Se parsea el string a un Expression Tree
// Usando System.Linq.Dynamic.Core internamente
var parsedExpression = DynamicExpressionParser.ParseLambda(
    new[] { param }, 
    typeof(bool), 
    expression
);

// Paso 4: Se compila a un delegate ejecutable
var compiledFunc = parsedExpression.Compile();

// Paso 5: Se cachea para invocaciones subsiguientes
// Key: hash de la expresión + tipos de parámetros
_compiledRulesCache[cacheKey] = compiledFunc;

// Paso 6: Ejecución
bool result = (bool)compiledFunc.DynamicInvoke(creditApplication);
```

### Costos Reales de Compilación

| Operación | Tiempo Típico | Notas |
|---|---|---|
| Parse de expresión simple | 0.1-0.5ms | `input1.X > 10` |
| Parse de expresión compleja | 1-5ms | Con navegación profunda y métodos |
| Compilación a delegate | 0.5-2ms | Depende de complejidad del árbol |
| Ejecución de delegate cacheado | 0.001-0.01ms | Prácticamente nativo |
| Primera ejecución (cold) | 2-10ms | Parse + compilación + ejecución |
| Ejecuciones subsiguientes | 0.001-0.01ms | Solo ejecución del delegate |

**Implicancia crítica**: En un sistema que evalúa 100 reglas, el cold start puede costar ~200-1000ms. Una vez cacheado, esas mismas 100 reglas se evalúan en ~1ms total.

## 1.7 Limitaciones Estructurales

1. **Sin inferencia**: No genera nuevos hechos. Si necesitás que el resultado de la Regla A sea input de la Regla B, debés orquestarlo manualmente o usar LocalParams con limitaciones.

2. **Sin temporal reasoning**: No hay concepto de "esta regla aplicó en los últimos 30 días". El contexto temporal debe inyectarse como input.

3. **Sin conflict resolution nativa**: Si dos reglas contradictorias se cumplen, ambas retornan `true`. La priorización debe manejarse externamente o con el campo `Priority` (que NO detiene evaluación, solo ordena resultados, a menos que se configure `EnableExceptionRulesSeverity`).

4. **Expresiones limitadas al subset de C# soportado por System.Linq.Dynamic.Core**: No se pueden usar `async/await`, `try/catch`, `switch`, `using` dentro de expresiones.

5. **Sin tipado fuerte en definición**: Las expresiones son strings. Errores de tipo se detectan en runtime, no en compilación.

6. **Sin soporte nativo para reglas sobre colecciones con cuantificadores complejos**: `.Any()` y `.All()` funcionan pero con limitaciones en la sintaxis dinámica.

---

# 2. Arquitectura Interna en Profundidad

## 2.1 Modelo de Objetos Core

### Workflow

El `Workflow` es la unidad de agrupación de nivel superior. Representa un conjunto cohesivo de reglas que se evalúan como unidad.

```csharp
public class Workflow
{
    public string WorkflowName { get; set; }           // Identificador único
    public List<Rule> Rules { get; set; }              // Colección de reglas
    public RuleExpressionType? RuleExpressionType { get; set; }  // Tipo por defecto
    public List<ScopedParam> GlobalParams { get; set; } // Parámetros globales
}
```

**Consideraciones arquitectónicas**:
- Un workflow debe mapear a un **bounded context** o un **caso de uso** específico
- No mezclar reglas de dominios diferentes en un mismo workflow
- El nombre del workflow es el key de búsqueda — debe ser estable y versionable

```csharp
// Anti-patrón: workflow monolítico
var badWorkflow = new Workflow {
    WorkflowName = "AllBusinessRules", // ❌ Demasiado amplio
    Rules = pricingRules.Concat(complianceRules).Concat(fraudRules).ToList()
};

// Patrón correcto: workflows cohesivos
var pricingWorkflow = new Workflow { WorkflowName = "Pricing_v2.3", Rules = pricingRules };
var complianceWorkflow = new Workflow { WorkflowName = "Compliance_AML_v1.1", Rules = complianceRules };
var fraudWorkflow = new Workflow { WorkflowName = "FraudDetection_v4.0", Rules = fraudRules };
```

### Rule

La `Rule` es la unidad atómica de evaluación. Cada regla contiene una expresión evaluable, metadata, y opcionalmente sub-reglas.

```csharp
public class Rule
{
    public string RuleName { get; set; }               // Identificador
    public string Operator { get; set; }               // AND/OR/Custom para child rules
    public string ErrorMessage { get; set; }           // Mensaje en caso de fallo
    public string ErrorType { get; set; }              // Warning/Error
    public RuleExpressionType RuleExpressionType { get; set; }
    public string Expression { get; set; }             // Expresión C# como string
    public IEnumerable<string> Actions { get; set; }   // Acciones post-evaluación  
    public IEnumerable<Rule> Rules { get; set; }       // Sub-reglas (child rules)
    public IEnumerable<ScopedParam> LocalParams { get; set; } // Variables locales
    public bool Enabled { get; set; } = true;          // Flag de activación
    public int Priority { get; set; }                   // Orden de evaluación (menor = primero, si está configurado)
    public string SuccessEvent { get; set; }           // Evento en caso de éxito
}
```

### RuleParameter

Representa un input tipado para el motor. El nombre del parámetro se usa como referencia en las expresiones.

```csharp
// Definición de inputs
var ruleParams = new[] {
    new RuleParameter("customer", customerObject),     // Accesible como "customer.X"
    new RuleParameter("order", orderObject),            // Accesible como "order.Y"
    new RuleParameter("config", configObject)            // Accesible como "config.Z"
};

// En la expresión se referencian por nombre:
// "customer.CreditScore > config.MinScore && order.Total < config.MaxOrderAmount"
```

**Regla crítica**: El nombre del `RuleParameter` **debe coincidir exactamente** con el nombre usado en la expresión. Si el parámetro se llama `"input1"`, la expresión debe usar `input1.Property`, no `customer.Property`.

### RuleResultTree

El resultado de la evaluación es un árbol que contiene información detallada:

```csharp
public class RuleResultTree
{
    public Rule Rule { get; set; }                          // La regla evaluada
    public bool IsSuccess { get; set; }                     // Resultado booleano
    public string ExceptionMessage { get; set; }            // Error si hubo excepción
    public RuleEvaluatedParams RuleEvaluatedParams { get; set; } // Params evaluados
    public IEnumerable<RuleResultTree> ChildResults { get; set; } // Resultados hijos
    public ActionResult ActionResult { get; set; }          // Resultado de acciones
}
```

Consumo correcto del resultado:

```csharp
List<RuleResultTree> results = await rulesEngine.ExecuteAllRulesAsync(
    "LoanApproval_v2", 
    ruleParams
);

foreach (var result in results)
{
    Console.WriteLine($"Rule: {result.Rule.RuleName}");
    Console.WriteLine($"  Success: {result.IsSuccess}");
    
    if (!string.IsNullOrEmpty(result.ExceptionMessage))
    {
        Console.WriteLine($"  Exception: {result.ExceptionMessage}");
    }
    
    // Para reglas con sub-reglas
    if (result.ChildResults?.Any() == true)
    {
        foreach (var child in result.ChildResults)
        {
            Console.WriteLine($"    Child: {child.Rule.RuleName} = {child.IsSuccess}");
        }
    }
    
    // Para reglas con acciones
    if (result.ActionResult != null)
    {
        var output = result.ActionResult.Output;
        Console.WriteLine($"  Action Output: {output}");
    }
}
```

### RuleExpressionType

Define cómo se evalúa la regla:

```csharp
public enum RuleExpressionType
{
    LambdaExpression,   // Expression string compilada dinámicamente
    RuleFunc            // Delegate C# pre-registrado
}
```

**LambdaExpression**: La expresión es un string que se compila en runtime. Es el modo más flexible y el que permite definición dinámica desde JSON/DB.

**RuleFunc**: La expresión es un delegate C# registrado previamente. Permite lógica arbitrariamente compleja pero pierde la capacidad de definición dinámica.

```csharp
// Registro de funciones custom
var reSettings = new ReSettings {
    CustomTypes = new[] { typeof(MathHelper), typeof(StringUtils) }
};

// Con RuleFunc se registra un delegate
rulesEngine.AddOrUpdateCustomRuleFunc("ComplexValidation", 
    (args) => {
        var customer = (Customer)args[0];
        // Lógica compleja que no se puede expresar en una expresión simple
        return ComplexValidationLogic(customer);
    });
```

## 2.2 Ejecución Interna Paso a Paso

### Paso 1: Resolución del Workflow

```csharp
// Internamente, los workflows se almacenan en un ConcurrentDictionary
// Key: WorkflowName, Value: Workflow compilado
private readonly ConcurrentDictionary<string, WorkflowRules> _workflowRules;
```

Cuando se invoca `ExecuteAllRulesAsync("WorkflowName", params)`, el motor busca el workflow en el diccionario. Si no existe, lanza excepción. Si se modificó desde la última compilación, se recompila.

### Paso 2: Preparación de Parámetros

Los `RuleParameter` se convierten en `ParameterExpression` para el árbol de expresiones. El motor crea un mapping entre nombres de parámetros y tipos.

```csharp
// Pseudo-código interno
var parameterExpressions = ruleParams.Select(rp => 
    Expression.Parameter(rp.Type, rp.Name)
).ToArray();
// Resultado: { ParameterExpression("input1", typeof(Customer)), 
//              ParameterExpression("input2", typeof(Order)) }
```

### Paso 3: Evaluación de GlobalParams

Si el workflow define `GlobalParams`, estos se evalúan primero y sus resultados están disponibles para todas las reglas.

```json
{
  "WorkflowName": "PricingEngine",
  "GlobalParams": [
    {
      "Name": "baseRate",
      "Expression": "input1.Region == \"US\" ? 0.05m : 0.07m"
    },
    {
      "Name": "riskMultiplier",  
      "Expression": "input1.CreditScore > 750 ? 1.0m : (input1.CreditScore > 600 ? 1.5m : 2.5m)"
    }
  ],
  "Rules": [
    {
      "RuleName": "CalculateFinalRate",
      "Expression": "baseRate * riskMultiplier < 0.15m"
    }
  ]
}
```

### Paso 4: Evaluación de LocalParams por Regla

Cada regla puede definir `LocalParams` que actúan como variables intermedias, evaluadas antes de la expresión principal.

```json
{
  "RuleName": "ComplexEligibility",
  "LocalParams": [
    {
      "Name": "adjustedScore",
      "Expression": "input1.CreditScore + (input1.YearsAsCustomer * 10)"
    },
    {
      "Name": "totalDebt",
      "Expression": "input1.Mortgages.Sum(m => m.Balance) + input1.CreditCards.Sum(c => c.Balance)"
    }
  ],
  "Expression": "adjustedScore > 680 && totalDebt < input1.AnnualIncome * 5"
}
```

### Paso 5: Compilación y Caching de Expresiones

```csharp
// El motor verifica si la expresión ya fue compilada
if (_compiledRules.TryGetValue(cacheKey, out var cachedDelegate))
{
    return cachedDelegate; // Fast path: delegate ya compilado
}

// Slow path: compilar la expresión
var lambda = ExpressionParser.Parse(expression, parameterExpressions);
var compiled = lambda.Compile();
_compiledRules[cacheKey] = compiled;
return compiled;
```

**Implicancia**: El cache es **por instancia** de `RulesEngine`. Si se crea una nueva instancia, se pierde el cache y se paga el costo de compilación nuevamente. En un contexto de DI, registrar como **Singleton** o **Scoped con cuidado**.

### Paso 6: Evaluación y Construcción del Resultado

La evaluación invoca el delegate compilado. Si la regla tiene `ChildRules`, se evalúan recursivamente y se combinan con el operador (`AND`/`OR`/`AndAlso`/`OrElse`).

## 2.3 Manejo de Múltiples Inputs

El motor soporta múltiples inputs que se mapean por posición y nombre:

```csharp
var customer = new Customer { Name = "Acme Corp", CreditScore = 750 };
var order = new Order { Total = 50000m, Items = 15 };
var context = new EvaluationContext { Date = DateTime.UtcNow, Region = "LATAM" };

// Los nombres "customer", "order", "context" deben coincidir 
// con los usados en las expresiones
var results = await engine.ExecuteAllRulesAsync("OrderValidation",
    new RuleParameter("customer", customer),
    new RuleParameter("order", order),
    new RuleParameter("context", context)
);

// Expresión válida:
// "customer.CreditScore > 600 && order.Total < 100000 && context.Region != \"BLOCKED\""
```

**Riesgo**: Si se cambia el nombre de un `RuleParameter` sin actualizar las expresiones en todas las reglas, las reglas fallarán en runtime con una excepción de parsing.

## 2.4 Navegación de Objetos Complejos

El motor soporta navegación profunda de propiedades a través de dot notation:

```csharp
// Objeto con estructura compleja
public class LoanApplication
{
    public Applicant PrimaryApplicant { get; set; }
    public List<Applicant> CoApplicants { get; set; }
    public Property CollateralProperty { get; set; }
}

public class Applicant
{
    public string Name { get; set; }
    public FinancialProfile Financial { get; set; }
    public Address CurrentAddress { get; set; }
}

public class FinancialProfile
{
    public decimal AnnualIncome { get; set; }
    public List<Debt> Debts { get; set; }
    public int CreditScore { get; set; }
}
```

Expresiones válidas sobre esta estructura:

```json
{
  "Expression": "input1.PrimaryApplicant.Financial.CreditScore > 700"
},
{
  "Expression": "input1.PrimaryApplicant.Financial.Debts.Sum(d => d.MonthlyPayment) < input1.PrimaryApplicant.Financial.AnnualIncome / 12 * 0.43m"
},
{
  "Expression": "input1.CoApplicants.Any(c => c.Financial.CreditScore > 650)"
},
{
  "Expression": "input1.CollateralProperty.AppraisedValue > input1.CollateralProperty.PurchasePrice * 0.8m"
}
```

## 2.5 Manejo de Null

El motor **no tiene null-safe navigation nativa** (no soporta el operador `?.` de C# en expresiones dinámicas). Un `NullReferenceException` dentro de una expresión se captura y se reporta como error en el `RuleResultTree`.

**Estrategia defensiva**:

```json
{
  "RuleName": "SafeNullCheck",
  "Expression": "input1.PrimaryApplicant != null && input1.PrimaryApplicant.Financial != null && input1.PrimaryApplicant.Financial.CreditScore > 700"
}
```

Esto es verbose pero necesario. Alternativa con `LocalParams`:

```json
{
  "RuleName": "SafeNullCheckWithLocal",
  "LocalParams": [
    {
      "Name": "hasFinancialProfile",
      "Expression": "input1.PrimaryApplicant != null && input1.PrimaryApplicant.Financial != null"
    }
  ],
  "Expression": "hasFinancialProfile && input1.PrimaryApplicant.Financial.CreditScore > 700"
}
```

**Configuración para manejar excepciones en reglas**:

```csharp
var reSettings = new ReSettings {
    HandleRuleException = true  // En lugar de propagar, marca la regla como fallida
};
```

Con `HandleRuleException = true`, si una expresión lanza una excepción (incluyendo `NullReferenceException`), la regla se marca como `IsSuccess = false` y el mensaje de excepción se almacena en `ExceptionMessage`. Sin esta configuración, la excepción se propaga y detiene la evaluación de todas las reglas.

## 2.6 Manejo de Excepciones

Tres niveles de manejo:

1. **Nivel de expresión**: Excepción dentro de la evaluación de una expresión individual.
2. **Nivel de regla**: Excepción durante la compilación o evaluación de una regla completa.
3. **Nivel de workflow**: Excepción que afecta la evaluación del workflow completo.

```csharp
var settings = new ReSettings {
    HandleRuleException = true, // Captura excepciones a nivel de regla
    EnableExceptionRulesSeverity = true // Permite severidad en errores
};

var engine = new RulesEngine.RulesEngine(workflows, settings);

var results = await engine.ExecuteAllRulesAsync("WorkflowName", inputs);

foreach (var result in results)
{
    if (!string.IsNullOrEmpty(result.ExceptionMessage))
    {
        // La regla falló por excepción, no por evaluación lógica
        logger.LogError(
            "Rule {RuleName} threw exception: {Exception}",
            result.Rule.RuleName,
            result.ExceptionMessage
        );
    }
}
```

## 2.7 Costos de Compilación y Caching Interno

### Estructura del Cache

```
RulesEngine Instance
  └── _workflowRules: ConcurrentDictionary<string, WorkflowRules>
       └── WorkflowRules
            └── CompiledRules: Dictionary<string, CompiledRule>
                 └── CompiledRule
                      ├── CompiledExpression: Delegate  (cacheado)
                      ├── CompiledLocalParams: Delegate[] (cacheados)
                      └── ChildCompiledRules: CompiledRule[]
```

### Ciclo de Vida del Cache

1. **Creación**: Al instanciar `RulesEngine` con workflows, las **definiciones** se almacenan pero las expresiones **no se compilan** todavía.
2. **Primera evaluación**: La expresión se compila y el delegate se cachea. Este es el cold-start.
3. **Evaluaciones subsiguientes**: Se usa el delegate cacheado. Rendimiento casi nativo.
4. **Actualización de workflow** (`AddOrUpdateWorkflow`): El cache del workflow se invalida. La próxima evaluación recompila.
5. **Destrucción**: Al disponer la instancia de `RulesEngine`, el cache se libera.

### Implicancias en Sistemas de Alto Volumen

Para un sistema que procesa 10,000 requests/segundo, cada uno evaluando 50 reglas:

- **Con cache caliente**: 50 × 0.01ms = 0.5ms por request → **viable**
- **Sin cache (cold start)**: 50 × 5ms = 250ms por request → **inaceptable**
- **Con recarga de reglas frecuente**: Cada recarga invalida cache → spikes de latencia

**Estrategia de warmup**:

```csharp
public class RulesEngineWarmupService : IHostedService
{
    private readonly RulesEngine.RulesEngine _engine;
    
    public async Task StartAsync(CancellationToken cancellationToken)
    {
        // Crear inputs dummy para forzar compilación de todas las reglas
        var dummyInputs = CreateDummyInputs();
        
        foreach (var workflowName in GetAllWorkflowNames())
        {
            // Esto fuerza la compilación y cacheo de todas las expresiones
            await _engine.ExecuteAllRulesAsync(workflowName, dummyInputs);
        }
        
        _logger.LogInformation("RulesEngine warmup completed. All rules pre-compiled.");
    }
    
    public Task StopAsync(CancellationToken cancellationToken) => Task.CompletedTask;
}
```

---

# 3. Configuración Avanzada y Customización

## 3.1 Instalación y Setup Inicial

```bash
dotnet add package RulesEngine --version 5.0.3
```

Dependencias transitivas relevantes:
- `System.Linq.Dynamic.Core` — Parser de expresiones dinámicas
- `FluentValidation` — Validación de estructura de reglas
- `Newtonsoft.Json` — Serialización/deserialización de workflows

```csharp
// Setup mínimo
using RulesEngine.Models;

var workflowJson = File.ReadAllText("workflows/pricing.json");
var workflows = JsonConvert.DeserializeObject<Workflow[]>(workflowJson);

var engine = new RulesEngine.RulesEngine(workflows);
```

## 3.2 Integración con ASP.NET Core y Dependency Injection

### Registro como Singleton (Recomendado para producción)

```csharp
// Program.cs o Startup.cs
public static class RulesEngineServiceExtensions
{
    public static IServiceCollection AddRulesEngine(
        this IServiceCollection services,
        IConfiguration configuration)
    {
        services.AddSingleton<IRulesEngineService>(sp =>
        {
            var logger = sp.GetRequiredService<ILogger<RulesEngineService>>();
            var settings = new ReSettings {
                HandleRuleException = true,
                CustomTypes = new[] { 
                    typeof(MathUtils), 
                    typeof(DateTimeUtils),
                    typeof(StringValidators) 
                }
            };
            
            return new RulesEngineService(settings, logger);
        });
        
        // Servicio de recarga en background
        services.AddHostedService<RulesReloadBackgroundService>();
        
        return services;
    }
}
```

### Servicio Wrapper con Abstracción

```csharp
public interface IRulesEngineService
{
    Task<List<RuleResultTree>> EvaluateAsync(
        string workflowName, 
        params RuleParameter[] inputs);
    
    Task ReloadWorkflowsAsync(CancellationToken ct = default);
    bool IsWorkflowLoaded(string workflowName);
}

public class RulesEngineService : IRulesEngineService
{
    private RulesEngine.RulesEngine _engine;
    private readonly ReSettings _settings;
    private readonly ILogger<RulesEngineService> _logger;
    private readonly SemaphoreSlim _reloadLock = new(1, 1);
    
    public RulesEngineService(ReSettings settings, ILogger<RulesEngineService> logger)
    {
        _settings = settings;
        _logger = logger;
        _engine = new RulesEngine.RulesEngine(Array.Empty<Workflow>(), settings);
    }
    
    public async Task<List<RuleResultTree>> EvaluateAsync(
        string workflowName, 
        params RuleParameter[] inputs)
    {
        try
        {
            var results = await _engine.ExecuteAllRulesAsync(workflowName, inputs);
            
            // Logging estructurado de resultados
            foreach (var result in results.Where(r => !r.IsSuccess))
            {
                _logger.LogDebug(
                    "Rule {RuleName} in {Workflow} failed. Exception: {Exception}",
                    result.Rule.RuleName,
                    workflowName,
                    result.ExceptionMessage ?? "None"
                );
            }
            
            return results;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, 
                "Critical error evaluating workflow {Workflow}", workflowName);
            throw;
        }
    }
    
    public async Task ReloadWorkflowsAsync(CancellationToken ct = default)
    {
        await _reloadLock.WaitAsync(ct);
        try
        {
            var workflows = await LoadWorkflowsFromSourceAsync(ct);
            
            foreach (var workflow in workflows)
            {
                _engine.AddOrUpdateWorkflow(workflow);
            }
            
            _logger.LogInformation(
                "Reloaded {Count} workflows successfully", workflows.Length);
        }
        finally
        {
            _reloadLock.Release();
        }
    }
    
    public bool IsWorkflowLoaded(string workflowName)
    {
        return _engine.ContainsWorkflow(workflowName);
    }
    
    private async Task<Workflow[]> LoadWorkflowsFromSourceAsync(CancellationToken ct)
    {
        // Implementar según la fuente: DB, archivos, API remota
        throw new NotImplementedException();
    }
}
```

## 3.3 Uso en Microservicios

### Patrón: Rules as a Service

```csharp
// Controller dedicado a evaluación de reglas
[ApiController]
[Route("api/v1/[controller]")]
public class RulesController : ControllerBase
{
    private readonly IRulesEngineService _rulesService;
    
    public RulesController(IRulesEngineService rulesService)
    {
        _rulesService = rulesService;
    }
    
    [HttpPost("evaluate/{workflowName}")]
    public async Task<ActionResult<RuleEvaluationResponse>> Evaluate(
        string workflowName,
        [FromBody] JsonElement payload)
    {
        // Deserializar dinámicamente según el workflow
        var inputs = MapPayloadToRuleParameters(workflowName, payload);
        
        var results = await _rulesService.EvaluateAsync(workflowName, inputs);
        
        var response = new RuleEvaluationResponse
        {
            WorkflowName = workflowName,
            EvaluatedAt = DateTime.UtcNow,
            Results = results.Select(r => new RuleResult
            {
                RuleName = r.Rule.RuleName,
                IsSuccess = r.IsSuccess,
                Message = r.IsSuccess 
                    ? r.Rule.SuccessEvent 
                    : r.Rule.ErrorMessage,
                ExceptionMessage = r.ExceptionMessage
            }).ToList()
        };
        
        return Ok(response);
    }
}
```

### Patrón: Reglas Embebidas en Domain Service

```csharp
// Dentro de un microservicio de dominio específico
public class LoanApplicationService
{
    private readonly IRulesEngineService _rules;
    private readonly ILoanRepository _repository;
    
    public async Task<LoanDecision> EvaluateApplication(LoanApplication application)
    {
        var results = await _rules.EvaluateAsync(
            "LoanUnderwriting_v3",
            new RuleParameter("application", application),
            new RuleParameter("marketConditions", await GetMarketConditions())
        );
        
        return new LoanDecision
        {
            IsApproved = results.All(r => r.IsSuccess),
            FailedRules = results
                .Where(r => !r.IsSuccess)
                .Select(r => r.Rule.RuleName)
                .ToList(),
            EvaluationTimestamp = DateTime.UtcNow
        };
    }
}
```

## 3.4 Versionado de Reglas

### Estrategia por Naming Convention

```csharp
// Convención: WorkflowName_v{Major}.{Minor}
// "PricingEngine_v2.3" → Major=2, Minor=3

public class VersionedWorkflowManager
{
    private readonly ConcurrentDictionary<string, SortedList<Version, Workflow>> _versions = new();
    
    public void RegisterVersion(Workflow workflow)
    {
        var (baseName, version) = ParseWorkflowName(workflow.WorkflowName);
        
        if (!_versions.ContainsKey(baseName))
            _versions[baseName] = new SortedList<Version, Workflow>();
        
        _versions[baseName][version] = workflow;
    }
    
    public Workflow GetLatestVersion(string baseName)
    {
        return _versions[baseName].Values.Last();
    }
    
    public Workflow GetSpecificVersion(string baseName, Version version)
    {
        return _versions[baseName][version];
    }
    
    // Permite rollback instantáneo
    public Workflow Rollback(string baseName)
    {
        var versions = _versions[baseName];
        if (versions.Count < 2) 
            throw new InvalidOperationException("No previous version to rollback to");
        
        return versions.Values[^2]; // Penúltima versión
    }
}
```

### Estrategia por Metadata en Base de Datos

```sql
CREATE TABLE RuleWorkflows (
    Id UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
    WorkflowName NVARCHAR(200) NOT NULL,
    VersionMajor INT NOT NULL,
    VersionMinor INT NOT NULL,
    WorkflowJson NVARCHAR(MAX) NOT NULL,
    IsActive BIT NOT NULL DEFAULT 0,
    CreatedBy NVARCHAR(100) NOT NULL,
    CreatedAt DATETIME2 NOT NULL DEFAULT GETUTCDATE(),
    ApprovedBy NVARCHAR(100) NULL,
    ApprovedAt DATETIME2 NULL,
    Notes NVARCHAR(1000) NULL,
    
    CONSTRAINT UQ_Workflow_Version 
        UNIQUE (WorkflowName, VersionMajor, VersionMinor),
    INDEX IX_Active_Workflows (WorkflowName, IsActive) 
        WHERE IsActive = 1
);
```

## 3.5 Carga desde Base de Datos y Recarga Dinámica

```csharp
public class DatabaseWorkflowProvider : IWorkflowProvider
{
    private readonly IDbConnectionFactory _dbFactory;
    private readonly ILogger<DatabaseWorkflowProvider> _logger;
    
    public async Task<Workflow[]> LoadActiveWorkflowsAsync(CancellationToken ct)
    {
        using var connection = await _dbFactory.CreateConnectionAsync();
        
        var workflowRecords = await connection.QueryAsync<WorkflowRecord>(
            @"SELECT WorkflowName, WorkflowJson 
              FROM RuleWorkflows 
              WHERE IsActive = 1 
              AND ApprovedAt IS NOT NULL",
            ct
        );
        
        return workflowRecords
            .Select(r => JsonConvert.DeserializeObject<Workflow>(r.WorkflowJson))
            .ToArray();
    }
}

// Background service para recarga periódica
public class RulesReloadBackgroundService : BackgroundService
{
    private readonly IRulesEngineService _rulesService;
    private readonly ILogger<RulesReloadBackgroundService> _logger;
    private readonly TimeSpan _reloadInterval = TimeSpan.FromMinutes(5);
    
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        // Carga inicial
        await _rulesService.ReloadWorkflowsAsync(stoppingToken);
        
        // Recarga periódica
        while (!stoppingToken.IsCancellationRequested)
        {
            await Task.Delay(_reloadInterval, stoppingToken);
            
            try
            {
                await _rulesService.ReloadWorkflowsAsync(stoppingToken);
                _logger.LogDebug("Rules reloaded successfully at {Time}", DateTime.UtcNow);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Failed to reload rules. Continuing with cached rules.");
                // No re-throw: seguir con reglas cacheadas es preferible a crashear
            }
        }
    }
}
```

## 3.6 Seguridad en Evaluación de Expresiones

### Riesgos Concretos

Las expresiones son strings C# que se compilan y ejecutan. Sin restricciones, un atacante que pueda editar el JSON de reglas podría:

1. **Ejecutar código arbitrario** mediante `System.Diagnostics.Process.Start()`
2. **Acceder al filesystem** mediante `System.IO.File.ReadAllText()`
3. **Realizar llamadas de red** mediante `System.Net.Http.HttpClient`
4. **Causar DoS** mediante loops infinitos en expresiones

### Mitigaciones

```csharp
var settings = new ReSettings {
    // Registrar SOLO los tipos explícitamente necesarios
    CustomTypes = new[] { 
        typeof(Math),           // Funciones matemáticas
        typeof(Decimal),        // Operaciones decimales
        typeof(DateTime),       // Comparaciones de fechas
        typeof(StringComparer)  // Comparaciones de strings
    }
    // NO registrar: System.IO, System.Net, System.Diagnostics, System.Reflection
};
```

### Validación de Expresiones Pre-Registro

```csharp
public class ExpressionSecurityValidator
{
    private static readonly HashSet<string> BlockedPatterns = new(StringComparer.OrdinalIgnoreCase)
    {
        "System.IO", "System.Net", "System.Diagnostics", "Process.Start",
        "File.Read", "File.Write", "File.Delete", "Directory.",
        "Assembly.", "Reflection.", "HttpClient", "WebClient",
        "Environment.", "Registry.", "Thread.Sleep", "Task.Delay"
    };
    
    public ValidationResult ValidateExpression(string expression)
    {
        foreach (var pattern in BlockedPatterns)
        {
            if (expression.Contains(pattern, StringComparison.OrdinalIgnoreCase))
            {
                return ValidationResult.Failure(
                    $"Expression contains blocked pattern: {pattern}");
            }
        }
        
        // Validar longitud máxima
        if (expression.Length > 2000)
        {
            return ValidationResult.Failure(
                "Expression exceeds maximum length of 2000 characters");
        }
        
        return ValidationResult.Success();
    }
}
```

## 3.7 Logging y Observabilidad

### Logging Estructurado con Métricas

```csharp
public class ObservableRulesEngineService : IRulesEngineService
{
    private readonly IRulesEngineService _inner;
    private readonly ILogger _logger;
    private readonly IMeterFactory _meterFactory;
    private readonly Counter<long> _evaluationCounter;
    private readonly Histogram<double> _evaluationDuration;
    
    public ObservableRulesEngineService(
        IRulesEngineService inner,
        ILogger<ObservableRulesEngineService> logger,
        IMeterFactory meterFactory)
    {
        _inner = inner;
        _logger = logger;
        
        var meter = meterFactory.Create("RulesEngine");
        _evaluationCounter = meter.CreateCounter<long>("rules.evaluations.total");
        _evaluationDuration = meter.CreateHistogram<double>("rules.evaluation.duration.ms");
    }
    
    public async Task<List<RuleResultTree>> EvaluateAsync(
        string workflowName, 
        params RuleParameter[] inputs)
    {
        var sw = Stopwatch.StartNew();
        
        try
        {
            var results = await _inner.EvaluateAsync(workflowName, inputs);
            sw.Stop();
            
            // Métricas
            _evaluationCounter.Add(1, 
                new KeyValuePair<string, object?>("workflow", workflowName),
                new KeyValuePair<string, object?>("status", "success"));
            _evaluationDuration.Record(sw.Elapsed.TotalMilliseconds,
                new KeyValuePair<string, object?>("workflow", workflowName));
            
            // Log estructurado
            _logger.LogInformation(
                "Workflow {Workflow} evaluated in {ElapsedMs:F2}ms. " +
                "Rules: {Total}, Passed: {Passed}, Failed: {Failed}",
                workflowName,
                sw.Elapsed.TotalMilliseconds,
                results.Count,
                results.Count(r => r.IsSuccess),
                results.Count(r => !r.IsSuccess)
            );
            
            return results;
        }
        catch (Exception ex)
        {
            _evaluationCounter.Add(1,
                new KeyValuePair<string, object?>("workflow", workflowName),
                new KeyValuePair<string, object?>("status", "error"));
            throw;
        }
    }
    
    // ... delegate remaining methods
}
```

---

# 4. Diseño Avanzado de Reglas Empresariales

## 4.1 Diseño de Reglas Mantenibles

### Principios Fundamentales

1. **Single Responsibility**: Cada regla evalúa exactamente un predicado de negocio
2. **Naming Descriptivo**: El nombre de la regla debe ser legible por negocio
3. **Granularidad Correcta**: Ni demasiado atómica ni demasiado compuesta
4. **Tolerancia a Null**: Toda regla debe manejar inputs incompletos
5. **Idempotencia**: La misma evaluación con los mismos inputs siempre produce el mismo resultado

```json
{
  "WorkflowName": "CustomerOnboarding_v1.2",
  "Rules": [
    {
      "RuleName": "KYC_IdentityVerification",
      "ErrorMessage": "Customer identity could not be verified",
      "ErrorType": "Error",
      "Expression": "input1.IdentityVerified == true && input1.IdentityVerificationDate > DateTime.UtcNow.AddMonths(-12)"
    },
    {
      "RuleName": "KYC_AgeRequirement",
      "ErrorMessage": "Customer does not meet minimum age requirement",
      "ErrorType": "Error",
      "Expression": "input1.DateOfBirth != null && DateTime.UtcNow.Year - input1.DateOfBirth.Value.Year >= 18"
    },
    {
      "RuleName": "AML_SanctionScreening",
      "ErrorMessage": "Customer appears on sanctions list",
      "ErrorType": "Error",
      "Expression": "input1.SanctionScreeningResult == \"CLEAR\""
    },
    {
      "RuleName": "Risk_CreditScoreMinimum",
      "ErrorMessage": "Credit score below minimum threshold",
      "ErrorType": "Warning",
      "Expression": "input1.CreditScore >= input2.MinCreditScore"
    }
  ]
}
```

## 4.2 LocalParams vs GlobalParams vs ScopedParams

### LocalParams

Scope: **Una sola regla**. Variables intermedias calculadas antes de la expresión principal.

```json
{
  "RuleName": "DebtServiceCoverageRatio",
  "LocalParams": [
    {
      "Name": "totalMonthlyDebt",
      "Expression": "input1.Debts.Sum(d => d.MonthlyPayment)"
    },
    {
      "Name": "monthlyIncome",
      "Expression": "input1.AnnualIncome / 12m"
    },
    {
      "Name": "dscr",
      "Expression": "monthlyIncome > 0 ? totalMonthlyDebt / monthlyIncome : 999m"
    }
  ],
  "Expression": "dscr < 0.43m"
}
```

**Advertencia crítica**: Los `LocalParams` se evalúan **en orden**. Un `LocalParam` puede referenciar `LocalParams` anteriores. Si se reordena incorrectamente, falla en runtime.

### GlobalParams

Scope: **Todas las reglas del workflow**. Se evalúan una vez y están disponibles en toda regla.

```json
{
  "WorkflowName": "InsurancePricing_v2",
  "GlobalParams": [
    {
      "Name": "baseRate",
      "Expression": "input1.Region == \"SOUTH\" ? 0.032m : 0.028m"
    },
    {
      "Name": "ageFactor",
      "Expression": "input1.Age < 25 ? 1.8m : (input1.Age < 65 ? 1.0m : 1.4m)"
    }
  ],
  "Rules": [
    {
      "RuleName": "PremiumCalculation",
      "Expression": "baseRate * ageFactor * input1.CoverageAmount < input1.MaxAcceptablePremium"
    },
    {
      "RuleName": "MinimumPremiumCheck",
      "Expression": "baseRate * ageFactor * input1.CoverageAmount >= 500m"
    }
  ]
}
```

### Uso Combinado Estratégico

```json
{
  "WorkflowName": "ComplexUnderwriting_v1",
  "GlobalParams": [
    {
      "Name": "marketVolatility",
      "Expression": "input2.VIXIndex > 30 ? \"HIGH\" : (input2.VIXIndex > 20 ? \"MEDIUM\" : \"LOW\")"
    }
  ],
  "Rules": [
    {
      "RuleName": "ExposureLimit",
      "LocalParams": [
        {
          "Name": "adjustedExposure",
          "Expression": "marketVolatility == \"HIGH\" ? input1.RequestedAmount * 0.7m : input1.RequestedAmount"
        }
      ],
      "Expression": "adjustedExposure <= input1.ApprovedLimit"
    }
  ]
}
```

## 4.3 Rule Chaining, Nested Rules y Dependent Rules

### Nested Rules (Child Rules)

Permiten composición lógica con operadores `AND`/`OR`:

```json
{
  "RuleName": "ComprehensiveEligibility",
  "Operator": "And",
  "Rules": [
    {
      "RuleName": "Financial_Eligible",
      "Operator": "And",
      "Rules": [
        {
          "RuleName": "MinimumIncome",
          "Expression": "input1.AnnualIncome >= 50000m"
        },
        {
          "RuleName": "MaxDebtRatio",
          "Expression": "input1.DebtToIncomeRatio < 0.43m"
        }
      ]
    },
    {
      "RuleName": "Credit_Eligible",
      "Operator": "Or",
      "Rules": [
        {
          "RuleName": "HighCreditScore",
          "Expression": "input1.CreditScore >= 740"
        },
        {
          "RuleName": "GoodScoreWithCollateral",
          "Operator": "And",
          "Rules": [
            {
              "RuleName": "AcceptableCreditScore",
              "Expression": "input1.CreditScore >= 640"
            },
            {
              "RuleName": "SufficientCollateral",
              "Expression": "input1.CollateralValue >= input1.LoanAmount * 1.2m"
            }
          ]
        }
      ]
    }
  ]
}
```

### Rule Chaining Manual (Orquestación)

RulesEngine no soporta chaining nativo. Se implementa orquestando múltiples evaluaciones:

```csharp
public class ChainingOrchestrator
{
    private readonly IRulesEngineService _engine;
    
    public async Task<UnderwritingDecision> ExecuteChainedEvaluation(
        LoanApplication application)
    {
        // Fase 1: Pre-calificación
        var prequalResults = await _engine.EvaluateAsync(
            "Prequalification_v2",
            new RuleParameter("app", application)
        );
        
        if (prequalResults.Any(r => !r.IsSuccess && r.Rule.ErrorType == "Error"))
        {
            return UnderwritingDecision.Rejected("Pre-qualification failed", prequalResults);
        }
        
        // Fase 2: Análisis profundo (solo si pasa pre-calificación)
        var enrichedApp = await EnrichWithExternalData(application);
        var deepResults = await _engine.EvaluateAsync(
            "DeepUnderwriting_v3",
            new RuleParameter("app", enrichedApp),
            new RuleParameter("prequalResults", MapToInput(prequalResults))
        );
        
        // Fase 3: Pricing (solo si pasa análisis profundo)
        if (deepResults.All(r => r.IsSuccess))
        {
            var pricingResults = await _engine.EvaluateAsync(
                "PricingEngine_v4",
                new RuleParameter("app", enrichedApp),
                new RuleParameter("riskProfile", BuildRiskProfile(deepResults))
            );
            
            return UnderwritingDecision.Approved(pricingResults);
        }
        
        return UnderwritingDecision.Rejected("Deep underwriting failed", deepResults);
    }
}
```

## 4.4 Evaluación sobre Colecciones Grandes

```json
{
  "RuleName": "PortfolioConcentration",
  "LocalParams": [
    {
      "Name": "topHoldingPct",
      "Expression": "input1.Holdings.Max(h => h.PercentageOfPortfolio)"
    },
    {
      "Name": "sectorCount",
      "Expression": "input1.Holdings.Select(h => h.Sector).Distinct().Count()"
    },
    {
      "Name": "highRiskExposure",
      "Expression": "input1.Holdings.Where(h => h.RiskRating == \"HIGH\").Sum(h => h.Value)"
    }
  ],
  "Expression": "topHoldingPct < 25m && sectorCount >= 5 && highRiskExposure < input1.TotalPortfolioValue * 0.15m"
}
```

**Performance warning**: Operaciones LINQ sobre colecciones grandes (>10,000 elementos) dentro de expresiones dinámicas pueden ser significativamente más lentas que código compilado equivalente. Para colecciones grandes, considerar pre-calcular aggregates como inputs separados.

## 4.5 Acciones (OnSuccess / OnFailure)

```json
{
  "RuleName": "HighValueTransaction",
  "Expression": "input1.Amount > 100000m",
  "Actions": {
    "OnSuccess": {
      "Name": "OutputExpression",
      "Context": {
        "Expression": "new (input1.Amount as TransactionAmount, \"HIGH_VALUE\" as Category, DateTime.UtcNow as FlaggedAt)"
      }
    },
    "OnFailure": {
      "Name": "OutputExpression",
      "Context": {
        "Expression": "new (input1.Amount as TransactionAmount, \"NORMAL\" as Category)"
      }
    }
  }
}
```

Consumo de acciones:

```csharp
var results = await engine.ExecuteAllRulesAsync("TransactionClassification", inputs);

foreach (var result in results)
{
    if (result.ActionResult?.Output != null)
    {
        dynamic output = result.ActionResult.Output;
        decimal amount = output.TransactionAmount;
        string category = output.Category;
        
        await ProcessClassification(amount, category);
    }
}
```

## 4.6 Casos Empresariales Completos

### Caso 1: Motor de Pricing Dinámico

**Contexto**: Plataforma SaaS que necesita calcular precios personalizados basados en múltiples factores dinámicos.

```json
{
  "WorkflowName": "DynamicPricing_v3.1",
  "GlobalParams": [
    {
      "Name": "volumeDiscount",
      "Expression": "customer.MonthlyVolume > 10000 ? 0.15m : (customer.MonthlyVolume > 5000 ? 0.10m : (customer.MonthlyVolume > 1000 ? 0.05m : 0m))"
    },
    {
      "Name": "loyaltyDiscount",
      "Expression": "customer.TenureMonths > 24 ? 0.08m : (customer.TenureMonths > 12 ? 0.04m : 0m)"
    },
    {
      "Name": "seasonalFactor",
      "Expression": "context.CurrentMonth >= 11 && context.CurrentMonth <= 12 ? 1.15m : 1.0m"
    }
  ],
  "Rules": [
    {
      "RuleName": "BasePrice_Enterprise",
      "Expression": "customer.Tier == \"Enterprise\"",
      "Actions": {
        "OnSuccess": {
          "Name": "OutputExpression",
          "Context": {
            "Expression": "new (99.99m * seasonalFactor * (1m - volumeDiscount) * (1m - loyaltyDiscount) as FinalPrice, \"ENTERPRISE\" as Tier)"
          }
        }
      }
    },
    {
      "RuleName": "BasePrice_Professional",
      "Expression": "customer.Tier == \"Professional\"",
      "Actions": {
        "OnSuccess": {
          "Name": "OutputExpression",
          "Context": {
            "Expression": "new (49.99m * seasonalFactor * (1m - volumeDiscount * 0.5m) * (1m - loyaltyDiscount) as FinalPrice, \"PROFESSIONAL\" as Tier)"
          }
        }
      }
    }
  ]
}
```

**Arquitectura**:
- Workflows versionados en base de datos con aprobación dual
- Recarga cada 5 minutos vía background service
- Auditoría completa: qué precio se aplicó, qué regla lo determinó, con qué inputs

**Riesgos**:
- Expresiones de pricing incorrectas pueden causar pérdidas financieras masivas
- Requiere ambiente de staging para probar reglas antes de activar en producción
- Necesita circuit breaker: si las reglas fallan, aplicar pricing por defecto

### Caso 2: Motor de Underwriting Financiero

**Contexto**: Evaluación automatizada de solicitudes de crédito con múltiples criterios.

```json
{
  "WorkflowName": "CreditUnderwriting_v5.2",
  "GlobalParams": [
    {
      "Name": "dti",
      "Expression": "applicant.MonthlyDebtPayments / (applicant.AnnualIncome / 12m)"
    },
    {
      "Name": "ltv",
      "Expression": "loan.RequestedAmount / collateral.AppraisedValue"
    }
  ],
  "Rules": [
    {
      "RuleName": "UW_CreditScoreMinimum",
      "ErrorMessage": "Credit score below minimum threshold of 620",
      "ErrorType": "Error",
      "Expression": "applicant.CreditScore >= 620",
      "Priority": 1
    },
    {
      "RuleName": "UW_DebtToIncomeRatio",
      "ErrorMessage": "DTI ratio exceeds maximum of 43%",
      "ErrorType": "Error",
      "Expression": "dti <= 0.43m",
      "Priority": 2
    },
    {
      "RuleName": "UW_LoanToValue",
      "ErrorMessage": "LTV ratio exceeds maximum of 80% without PMI",
      "ErrorType": "Warning",
      "Expression": "ltv <= 0.80m",
      "Priority": 3
    },
    {
      "RuleName": "UW_EmploymentStability",
      "ErrorMessage": "Less than 2 years at current employer",
      "ErrorType": "Warning",
      "Expression": "applicant.YearsAtCurrentEmployer >= 2",
      "Priority": 4
    },
    {
      "RuleName": "UW_BankruptcyCheck",
      "ErrorMessage": "Active or recent bankruptcy within 7 years",
      "ErrorType": "Error",
      "Expression": "applicant.BankruptcyDate == null || (DateTime.UtcNow - applicant.BankruptcyDate.Value).TotalDays > 2555",
      "Priority": 1
    }
  ]
}
```

### Caso 3: Motor Antifraude

```json
{
  "WorkflowName": "FraudDetection_v4.0",
  "Rules": [
    {
      "RuleName": "FRAUD_VelocityCheck",
      "ErrorMessage": "Excessive transaction frequency detected",
      "Expression": "txContext.TransactionsLast24Hours < 50 && txContext.TransactionsLastHour < 10"
    },
    {
      "RuleName": "FRAUD_AmountAnomaly",
      "LocalParams": [
        {
          "Name": "avgAmount",
          "Expression": "txContext.AverageTransactionAmount30Days"
        },
        {
          "Name": "stdDev",
          "Expression": "txContext.StdDevTransactionAmount30Days"
        }
      ],
      "Expression": "transaction.Amount < (avgAmount + stdDev * 3m)"
    },
    {
      "RuleName": "FRAUD_GeographicAnomaly",
      "Expression": "transaction.Country == txContext.UsualCountry || txContext.TravelNotificationActive == true"
    },
    {
      "RuleName": "FRAUD_DeviceFingerprint",
      "Expression": "txContext.KnownDeviceIds.Contains(transaction.DeviceId)"
    },
    {
      "RuleName": "FRAUD_HighRiskMerchant",
      "Expression": "!txContext.HighRiskMerchantCategories.Contains(transaction.MerchantCategoryCode)"
    }
  ]
}
```

**Riesgos del motor antifraude con RulesEngine**:
- **Latencia**: Cada transacción debe evaluarse en <10ms. Con cache caliente, viable. Sin cache, inaceptable.
- **Contexto temporal**: Las métricas de velocidad (txLast24Hours) deben calcularse **externamente** e inyectarse como input.
- **RulesEngine no reemplaza ML**: Para patrones sofisticados de fraude, RulesEngine actúa como capa de reglas heurísticas. Los modelos ML manejan detección de anomalías complejas.

### Caso 4: Cumplimiento Regulatorio

```json
{
  "WorkflowName": "RegulatoryCompliance_GDPR_v2.1",
  "Rules": [
    {
      "RuleName": "GDPR_ConsentRequired",
      "ErrorMessage": "Valid consent not obtained for data processing",
      "ErrorType": "Error",
      "Expression": "dataSubject.ConsentGiven == true && dataSubject.ConsentDate > DateTime.UtcNow.AddYears(-2)"
    },
    {
      "RuleName": "GDPR_DataMinimization",
      "ErrorMessage": "Processing exceeds declared purposes",
      "ErrorType": "Error",
      "Expression": "processingActivity.DataFieldsUsed.All(f => processingActivity.DeclaredPurposeFields.Contains(f))"
    },
    {
      "RuleName": "GDPR_RetentionPeriod",
      "ErrorMessage": "Data retained beyond maximum retention period",
      "ErrorType": "Error",
      "Expression": "(DateTime.UtcNow - dataRecord.CreatedAt).TotalDays <= config.MaxRetentionDays"
    },
    {
      "RuleName": "GDPR_CrossBorderTransfer",
      "ErrorMessage": "Cross-border data transfer to non-adequate country without safeguards",
      "ErrorType": "Error",
      "Expression": "config.AdequateCountries.Contains(transfer.DestinationCountry) || transfer.HasStandardContractualClauses == true"
    }
  ]
}
```

### Caso 5: Aprobación Multinivel

```csharp
public class MultiLevelApprovalEngine
{
    private readonly IRulesEngineService _engine;
    
    public async Task<ApprovalResult> DetermineApprovalLevel(PurchaseRequest request)
    {
        // Nivel 1: Auto-aprobación
        var autoApprovalResults = await _engine.EvaluateAsync(
            "Approval_AutoApprove_v1",
            new RuleParameter("request", request)
        );
        
        if (autoApprovalResults.All(r => r.IsSuccess))
            return ApprovalResult.AutoApproved();
        
        // Nivel 2: Aprobación de gerente
        var managerResults = await _engine.EvaluateAsync(
            "Approval_ManagerRequired_v1",
            new RuleParameter("request", request)
        );
        
        if (managerResults.All(r => r.IsSuccess))
            return ApprovalResult.RequiresManagerApproval();
        
        // Nivel 3: Aprobación de director
        var directorResults = await _engine.EvaluateAsync(
            "Approval_DirectorRequired_v1",
            new RuleParameter("request", request)
        );
        
        if (directorResults.All(r => r.IsSuccess))
            return ApprovalResult.RequiresDirectorApproval();
        
        // Nivel 4: Aprobación de VP
        return ApprovalResult.RequiresVPApproval();
    }
}
```

Workflows correspondientes:

```json
[
  {
    "WorkflowName": "Approval_AutoApprove_v1",
    "Rules": [
      {
        "RuleName": "AmountUnderThreshold",
        "Expression": "request.Amount <= 5000m"
      },
      {
        "RuleName": "BudgetAvailable",
        "Expression": "request.DepartmentBudgetRemaining >= request.Amount"
      },
      {
        "RuleName": "ApprovedVendor",
        "Expression": "request.IsPreApprovedVendor == true"
      }
    ]
  },
  {
    "WorkflowName": "Approval_ManagerRequired_v1",
    "Rules": [
      {
        "RuleName": "AmountUnderManagerThreshold",
        "Expression": "request.Amount <= 25000m"
      },
      {
        "RuleName": "WithinDepartmentScope",
        "Expression": "request.IsWithinDepartmentScope == true"
      }
    ]
  }
]
```

---

# 5. Performance y Escalabilidad Real

## 5.1 Costos por Evaluación

### Benchmark Realista

```csharp
// Benchmark con BenchmarkDotNet
[MemoryDiagnoser]
[SimpleJob(RuntimeMoniker.Net80)]
public class RulesEngineBenchmark
{
    private RulesEngine.RulesEngine _engine;
    private RuleParameter[] _params;
    
    [GlobalSetup]
    public void Setup()
    {
        var workflows = LoadWorkflows();
        _engine = new RulesEngine.RulesEngine(workflows, new ReSettings { HandleRuleException = true });
        _params = CreateTestParameters();
        
        // Warmup: forzar compilación de todas las reglas
        _engine.ExecuteAllRulesAsync("TestWorkflow", _params).GetAwaiter().GetResult();
    }
    
    [Benchmark(Baseline = true)]
    public async Task Evaluate_10Rules() 
        => await _engine.ExecuteAllRulesAsync("Simple10Rules", _params);
    
    [Benchmark]
    public async Task Evaluate_50Rules() 
        => await _engine.ExecuteAllRulesAsync("Medium50Rules", _params);
    
    [Benchmark]
    public async Task Evaluate_100Rules() 
        => await _engine.ExecuteAllRulesAsync("Complex100Rules", _params);
    
    [Benchmark]
    public async Task Evaluate_ImperativeEquivalent() 
        => ImperativeRulesCheck(_params);
}
```

### Resultados Típicos

| Escenario | RulesEngine (warm) | Código Imperativo | Ratio |
|---|---|---|---|
| 10 reglas simples | ~0.08ms | ~0.002ms | 40x |
| 50 reglas con LocalParams | ~0.5ms | ~0.01ms | 50x |
| 100 reglas complejas | ~1.2ms | ~0.03ms | 40x |
| 10 reglas (cold start) | ~50ms | ~0.002ms | 25000x |
| Con navegación profunda | ~0.15ms/regla | ~0.003ms/regla | 50x |

**Interpretación**: RulesEngine es ~40-50x más lento que código imperativo equivalente. Esto es **aceptable** para la mayoría de escenarios enterprise donde la latencia total del request es 50-500ms. El costo de 1-2ms por evaluación es insignificante comparado con I/O de base de datos (5-50ms) o llamadas HTTP (20-200ms).

**Cuándo NO es aceptable**: Sistemas de trading de alta frecuencia, procesamiento de streams en tiempo real con SLAs sub-milisegundo, loops internos que evalúan millones de registros.

## 5.2 Impacto en CPU y Memoria

### CPU

- **Compilación**: Intensiva en CPU. Cientos de reglas compilándose simultáneamente pueden saturar un core durante 1-5 segundos.
- **Evaluación (warm)**: Mínimo. Un delegate compilado es prácticamente código nativo.
- **Expression parsing**: El parsing de System.Linq.Dynamic.Core usa reflexión y generación de código, lo cual es CPU-intensive.

### Memoria

```
Footprint por regla compilada (aproximado):
├── Expression string original:     ~200 bytes
├── Compiled delegate:              ~500-2000 bytes
├── Expression tree (antes de GC):  ~2-5 KB (temporal)
├── LocalParams compilados:         ~500 bytes/param
└── Metadata (RuleName, etc.):      ~300 bytes

Total por regla:                     ~1-3 KB (estado estable)
```

Para un sistema con 1,000 reglas: ~1-3 MB de memoria para el cache de reglas. Trivial.

Para un sistema con 100,000 reglas: ~100-300 MB. Requiere planificación.

### Garbage Collection

La compilación de expresiones genera objetos temporales que presionan el GC. En un sistema con recarga frecuente de reglas, la Gen 2 GC puede causar pausas de latencia.

**Mitigación**: Recargar reglas en momentos de bajo tráfico o de manera incremental (solo workflows que cambiaron).

## 5.3 Concurrencia y Thread Safety

### Thread Safety del Motor

`RulesEngine` es **thread-safe para evaluación** una vez inicializado. Múltiples threads pueden invocar `ExecuteAllRulesAsync` concurrentemente sobre la misma instancia sin problemas.

**Sin embargo**, `AddOrUpdateWorkflow` y `RemoveWorkflow` **no son atómicos con respecto a evaluaciones en curso**. Si se actualiza un workflow mientras otro thread lo está evaluando, el comportamiento puede ser inconsistente.

### Patrón de Actualización Segura

```csharp
public class ThreadSafeRulesEngineManager
{
    private volatile RulesEngine.RulesEngine _currentEngine;
    private readonly ReSettings _settings;
    private readonly ReaderWriterLockSlim _lock = new();
    
    public async Task<List<RuleResultTree>> EvaluateAsync(
        string workflowName, RuleParameter[] inputs)
    {
        _lock.EnterReadLock();
        try
        {
            return await _currentEngine.ExecuteAllRulesAsync(workflowName, inputs);
        }
        finally
        {
            _lock.ExitReadLock();
        }
    }
    
    public void UpdateWorkflows(Workflow[] workflows)
    {
        // Crear nueva instancia con las nuevas reglas
        var newEngine = new RulesEngine.RulesEngine(workflows, _settings);
        
        // Warmup: compilar todas las reglas antes de swap
        WarmupEngine(newEngine, workflows);
        
        _lock.EnterWriteLock();
        try
        {
            var oldEngine = _currentEngine;
            _currentEngine = newEngine;
            // oldEngine será GC'd
        }
        finally
        {
            _lock.ExitWriteLock();
        }
    }
    
    private void WarmupEngine(RulesEngine.RulesEngine engine, Workflow[] workflows)
    {
        foreach (var wf in workflows)
        {
            var dummyInputs = CreateDummyInputsForWorkflow(wf);
            engine.ExecuteAllRulesAsync(wf.WorkflowName, dummyInputs)
                .GetAwaiter().GetResult();
        }
    }
}
```

## 5.4 Estrategias de Optimización

### 1. Reducir Complejidad de Expresiones

```json
// ❌ Expresión compleja evaluada en cada regla
{
  "Expression": "input1.Orders.Where(o => o.Date > DateTime.UtcNow.AddDays(-30)).Sum(o => o.Amount) > 10000m"
}

// ✅ Pre-calcular y pasar como input
// En C#: var recentOrdersTotal = orders.Where(...).Sum(...);  
// Pasar como RuleParameter("metrics", new { RecentOrdersTotal = recentOrdersTotal })
{
  "Expression": "metrics.RecentOrdersTotal > 10000m"
}
```

### 2. Short-Circuit con Prioridad

```csharp
var results = await engine.ExecuteAllRulesAsync("Underwriting", inputs);

// Evaluar resultados en orden de prioridad
var criticalFailures = results
    .Where(r => !r.IsSuccess && r.Rule.ErrorType == "Error")
    .OrderBy(r => r.Rule.Priority)
    .ToList();

if (criticalFailures.Any())
{
    // No evaluar reglas de pricing si el underwriting falló
    return Decision.Rejected(criticalFailures.First().Rule.ErrorMessage);
}
```

### 3. Workflows Separados por Costo

```csharp
// Evaluar reglas baratas primero, caras solo si es necesario
var quickCheckResults = await engine.EvaluateAsync("QuickPrescreen", inputs);

if (quickCheckResults.All(r => r.IsSuccess))
{
    // Solo si pasa el prescreen, evaluar reglas costosas
    var deepResults = await engine.EvaluateAsync("DeepAnalysis", enrichedInputs);
    return ProcessDeepResults(deepResults);
}

return Rejection("Failed pre-screening");
```

## 5.5 Cuándo NO Usar RulesEngine

| Escenario | Razón | Alternativa |
|---|---|---|
| < 5 reglas estáticas | Overhead injustificado | if/else en código |
| Reglas cambian < 1 vez/año | Sin beneficio de dinamismo | Código + deploy |
| Latencia sub-milisegundo | 40-50x overhead vs nativo | Código compilado |
| Millones de evaluaciones/seg | Overhead de delegates | Código nativo o SIMD |
| Reglas interdependientes complejas | Sin inferencia | NRules o Drools |
| Requiere backward chaining | No soportado | Drools |
| Equipo sin experiencia .NET | Expresiones son C# | Motor agnostico al lenguaje |

---

# 6. Patrones de Diseño y Gobernanza

## 6.1 Integración con DDD (Domain-Driven Design)

### Ubicación del Motor en la Arquitectura DDD

El motor de reglas reside en la **capa de Dominio** o en la **capa de Aplicación**, dependiendo del tipo de regla:

```
┌─────────────────────────────────────────────────────────────────┐
│                         Presentación                            │
├─────────────────────────────────────────────────────────────────┤
│                          Aplicación                             │
│   ┌──────────────────────────────────────────────────────────┐  │
│   │  Application Services (Orquestación de reglas)           │  │
│   │  - ChainingOrchestrator                                  │  │
│   │  - Reglas de proceso (aprobación multinivel)             │  │
│   └──────────────────────────────────────────────────────────┘  │
├─────────────────────────────────────────────────────────────────┤
│                            Dominio                              │
│   ┌──────────────────────────────────────────────────────────┐  │
│   │  Domain Services                                         │  │
│   │  - IRulesEngineService (interfaz)                        │  │
│   │  - Reglas de negocio puras (eligibilidad, pricing)       │  │
│   │  - Validaciones de dominio                               │  │
│   └──────────────────────────────────────────────────────────┘  │
│   ┌──────────────────────────────────────────────────────────┐  │
│   │  Entities / Value Objects                                │  │
│   │  - NO contienen lógica de RulesEngine                    │  │
│   │  - RulesEngine opera SOBRE ellos, no dentro              │  │
│   └──────────────────────────────────────────────────────────┘  │
├─────────────────────────────────────────────────────────────────┤
│                        Infraestructura                          │
│   ┌──────────────────────────────────────────────────────────┐  │
│   │  RulesEngine Implementation                              │  │
│   │  - RulesEngineService (implementación)                   │  │
│   │  - DatabaseWorkflowProvider                              │  │
│   │  - File-based WorkflowProvider                           │  │
│   └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### Principio Fundamental

**Las entidades de dominio no deben saber que RulesEngine existe.** El motor opera como un servicio de dominio que recibe entidades como input y retorna decisiones.

```csharp
// ✅ Correcto: Domain Service que usa RulesEngine internamente
public class CreditEligibilityService : ICreditEligibilityService
{
    private readonly IRulesEngineService _rulesEngine;
    
    public async Task<EligibilityDecision> Evaluate(CreditApplication application)
    {
        var results = await _rulesEngine.EvaluateAsync(
            "CreditEligibility_v3",
            new RuleParameter("app", application)
        );
        
        // Traducir resultado del motor a concepto de dominio
        return new EligibilityDecision(
            isEligible: results.All(r => r.IsSuccess),
            rejectionReasons: results
                .Where(r => !r.IsSuccess)
                .Select(r => new RejectionReason(r.Rule.RuleName, r.Rule.ErrorMessage))
                .ToList()
        );
    }
}

// ❌ Incorrecto: Entidad acoplada a RulesEngine
public class CreditApplication
{
    public async Task<bool> IsEligible(RulesEngine.RulesEngine engine) // NO
    {
        // La entidad no debe conocer el motor de reglas
    }
}
```

### Specification Pattern + RulesEngine

```csharp
public interface ISpecification<T>
{
    Task<SpecificationResult> IsSatisfiedByAsync(T entity);
}

public class RulesEngineSpecification<T> : ISpecification<T>
{
    private readonly IRulesEngineService _engine;
    private readonly string _workflowName;
    private readonly string _paramName;
    
    public RulesEngineSpecification(
        IRulesEngineService engine, 
        string workflowName,
        string paramName = "input1")
    {
        _engine = engine;
        _workflowName = workflowName;
        _paramName = paramName;
    }
    
    public async Task<SpecificationResult> IsSatisfiedByAsync(T entity)
    {
        var results = await _engine.EvaluateAsync(
            _workflowName,
            new RuleParameter(_paramName, entity)
        );
        
        return new SpecificationResult(
            isSatisfied: results.All(r => r.IsSuccess),
            violations: results
                .Where(r => !r.IsSuccess)
                .Select(r => r.Rule.ErrorMessage)
                .ToList()
        );
    }
}

// Uso
var eligibilitySpec = new RulesEngineSpecification<LoanApplication>(
    rulesEngine, "LoanEligibility_v2", "application");

var result = await eligibilitySpec.IsSatisfiedByAsync(application);
if (!result.IsSatisfied)
{
    throw new DomainException($"Application does not meet eligibility: {string.Join(", ", result.Violations)}");
}
```

## 6.2 Integración con Clean Architecture

### Distribución de Responsabilidades

```csharp
// Layer: Domain (Contracts)
namespace Domain.Rules;

public interface IRuleEvaluator
{
    Task<RuleEvaluationResult> EvaluateAsync(string context, object input);
}

public record RuleEvaluationResult(
    bool AllPassed,
    IReadOnlyList<RuleOutcome> Outcomes);

public record RuleOutcome(
    string RuleName,
    bool Passed,
    string? ErrorMessage);

// Layer: Application (Use Cases)
namespace Application.UseCases;

public class ProcessLoanApplicationHandler : IRequestHandler<ProcessLoanCommand, LoanDecision>
{
    private readonly IRuleEvaluator _ruleEvaluator;
    private readonly ILoanRepository _repository;
    private readonly IEventBus _eventBus;
    
    public async Task<LoanDecision> Handle(
        ProcessLoanCommand command, CancellationToken ct)
    {
        var application = await _repository.GetByIdAsync(command.ApplicationId, ct);
        
        var evaluationResult = await _ruleEvaluator.EvaluateAsync(
            "LoanUnderwriting", application);
        
        var decision = application.MakeDecision(evaluationResult);
        
        await _repository.SaveAsync(application, ct);
        await _eventBus.PublishAsync(new LoanDecisionMadeEvent(decision), ct);
        
        return decision;
    }
}

// Layer: Infrastructure (Implementation)
namespace Infrastructure.Rules;

public class MicrosoftRuleEvaluator : IRuleEvaluator
{
    private readonly RulesEngine.RulesEngine _engine;
    
    public async Task<RuleEvaluationResult> EvaluateAsync(string context, object input)
    {
        var results = await _engine.ExecuteAllRulesAsync(
            context, new RuleParameter("input1", input));
        
        return new RuleEvaluationResult(
            AllPassed: results.All(r => r.IsSuccess),
            Outcomes: results.Select(r => new RuleOutcome(
                r.Rule.RuleName,
                r.IsSuccess,
                r.IsSuccess ? null : r.Rule.ErrorMessage
            )).ToList()
        );
    }
}
```

## 6.3 Anti-Patterns

### Anti-Pattern 1: God Workflow

```json
// ❌ Un workflow con todas las reglas de negocio
{
  "WorkflowName": "AllRules",
  "Rules": [
    { "RuleName": "Pricing_Base", "Expression": "..." },
    { "RuleName": "Pricing_Discount", "Expression": "..." },
    { "RuleName": "Fraud_Velocity", "Expression": "..." },
    { "RuleName": "Compliance_KYC", "Expression": "..." },
    { "RuleName": "Approval_Level", "Expression": "..." }
    // 200 reglas más mezclando dominios
  ]
}
```

**Problema**: Imposible de mantener, testear, o versionar independientemente. Un cambio en pricing puede romper compliance.

### Anti-Pattern 2: Lógica de Negocio en Expresiones

```json
// ❌ Demasiada lógica en la expresión
{
  "Expression": "input1.Type == \"PREMIUM\" ? (input1.Amount * 0.95m - (input1.Loyalty > 5 ? input1.Amount * 0.05m : 0m) + input1.Tax * (input1.Region == \"EU\" ? 0.21m : 0.15m)) < input1.MaxBudget : input1.Amount < 10000m"
}
```

**Solución**: Descomponer en múltiples reglas con `LocalParams` o en múltiples workflows.

### Anti-Pattern 3: Acoplamiento Temporal entre Reglas

```csharp
// ❌ Asumir que las reglas se evalúan en un orden específico
var results = await engine.ExecuteAllRulesAsync("Workflow", inputs);
var firstResult = results[0]; // Asumir que es "Regla_Critica"
```

**Solución**: Buscar resultados por nombre, no por posición.

```csharp
var criticalResult = results.First(r => r.Rule.RuleName == "Regla_Critica");
```

### Anti-Pattern 4: Falta de Fallback

```csharp
// ❌ Sin manejo de error
var results = await engine.ExecuteAllRulesAsync("Pricing", inputs);
// Si el workflow no existe o falla, la aplicación crashea

// ✅ Con fallback
try
{
    var results = await engine.ExecuteAllRulesAsync("Pricing", inputs);
    return CalculateFromRules(results);
}
catch (Exception ex)
{
    logger.LogError(ex, "Rules evaluation failed, applying default pricing");
    return DefaultPricing(inputs);
}
```

### Anti-Pattern 5: Testing Solo de Reglas Individuales

Testear cada regla aislada es necesario pero **no suficiente**. Se debe testear la **interacción** entre reglas y el **workflow completo**.

## 6.4 Testing Unitario Avanzado

### Testing de Reglas Individuales

```csharp
public class CreditEligibilityRulesTests
{
    private readonly RulesEngine.RulesEngine _engine;
    
    public CreditEligibilityRulesTests()
    {
        var workflowJson = File.ReadAllText("TestWorkflows/credit_eligibility.json");
        var workflows = JsonConvert.DeserializeObject<Workflow[]>(workflowJson);
        _engine = new RulesEngine.RulesEngine(workflows, new ReSettings { HandleRuleException = true });
    }
    
    [Theory]
    [InlineData(750, true, "High credit score should pass")]
    [InlineData(700, true, "Minimum credit score should pass")]
    [InlineData(699, false, "Below minimum should fail")]
    [InlineData(0, false, "Zero credit score should fail")]
    [InlineData(-1, false, "Negative credit score should fail")]
    public async Task CreditScoreMinimum_ShouldEvaluateCorrectly(
        int creditScore, bool expectedResult, string scenario)
    {
        // Arrange
        var applicant = new CreditApplicant { CreditScore = creditScore };
        
        // Act
        var results = await _engine.ExecuteAllRulesAsync(
            "CreditEligibility_v1",
            new RuleParameter("input1", applicant)
        );
        
        // Assert
        var scoreRule = results.First(r => r.Rule.RuleName == "CreditScoreMinimum");
        Assert.Equal(expectedResult, scoreRule.IsSuccess, scenario);
    }
}
```

### Testing de Workflows Completos

```csharp
[Fact]
public async Task FullUnderwriting_HighQualityApplicant_ShouldPassAllRules()
{
    // Arrange
    var applicant = TestDataBuilder.CreateHighQualityApplicant();
    
    // Act
    var results = await _engine.ExecuteAllRulesAsync(
        "FullUnderwriting_v3",
        new RuleParameter("applicant", applicant)
    );
    
    // Assert
    Assert.All(results, r => Assert.True(r.IsSuccess,
        $"Rule {r.Rule.RuleName} unexpectedly failed: {r.Rule.ErrorMessage}"));
}

[Fact]
public async Task FullUnderwriting_Bankrupt_ShouldFailBankruptcyRule()
{
    // Arrange
    var applicant = TestDataBuilder.CreateApplicantWithRecentBankruptcy();
    
    // Act
    var results = await _engine.ExecuteAllRulesAsync(
        "FullUnderwriting_v3",
        new RuleParameter("applicant", applicant)
    );
    
    // Assert
    var bankruptcyRule = results.First(r => r.Rule.RuleName == "UW_BankruptcyCheck");
    Assert.False(bankruptcyRule.IsSuccess);
    Assert.Contains("bankruptcy", bankruptcyRule.Rule.ErrorMessage, StringComparison.OrdinalIgnoreCase);
}
```

### Testing Masivo Automatizado (Data-Driven)

```csharp
public class MassiveRulesTesting
{
    [Fact]
    public async Task BulkEvaluation_AllScenariosFromCsv_ShouldMatchExpected()
    {
        var engine = SetupEngine();
        var testCases = LoadTestCasesFromCsv("TestData/underwriting_scenarios.csv");
        var failures = new List<string>();
        
        foreach (var testCase in testCases)
        {
            var results = await engine.ExecuteAllRulesAsync(
                testCase.WorkflowName,
                new RuleParameter("input1", testCase.Input)
            );
            
            var allPassed = results.All(r => r.IsSuccess);
            
            if (allPassed != testCase.ExpectedOutcome)
            {
                var failedRules = results
                    .Where(r => !r.IsSuccess)
                    .Select(r => r.Rule.RuleName);
                    
                failures.Add(
                    $"Scenario '{testCase.Name}': Expected {testCase.ExpectedOutcome}, " +
                    $"got {allPassed}. Failed rules: {string.Join(", ", failedRules)}");
            }
        }
        
        Assert.Empty(failures);
    }
}
```

### Testing de Consistencia entre Versiones

```csharp
[Fact]
public async Task VersionUpgrade_SameInputs_ShouldProduceSameOrBetterResults()
{
    var engineV1 = new RulesEngine.RulesEngine(LoadWorkflows("v1"));
    var engineV2 = new RulesEngine.RulesEngine(LoadWorkflows("v2"));
    var testInputs = GenerateRegressionsTestInputs(1000);
    
    var regressions = new List<string>();
    
    foreach (var input in testInputs)
    {
        var v1Results = await engineV1.ExecuteAllRulesAsync("Pricing", input);
        var v2Results = await engineV2.ExecuteAllRulesAsync("Pricing", input);
        
        // Detectar regresiones: reglas que pasaban en v1 pero fallan en v2
        foreach (var v1Rule in v1Results.Where(r => r.IsSuccess))
        {
            var v2Rule = v2Results.FirstOrDefault(
                r => r.Rule.RuleName == v1Rule.Rule.RuleName);
            
            if (v2Rule != null && !v2Rule.IsSuccess)
            {
                regressions.Add(
                    $"Regression: {v1Rule.Rule.RuleName} passed in v1 but fails in v2");
            }
        }
    }
    
    Assert.Empty(regressions);
}
```

## 6.5 Gobernanza en Equipos Grandes

### Modelo de Gobernanza

```
┌──────────────────────────────────────────────────────────────┐
│                    GOBERNANZA DE REGLAS                       │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  1. Propuesta (Business Analyst)                             │
│     └─ Describe regla en lenguaje natural                    │
│     └─ Define inputs y outputs esperados                     │
│     └─ Provee casos de prueba                                │
│                                                              │
│  2. Implementación (Developer)                               │
│     └─ Traduce a expresión RulesEngine                       │
│     └─ Implementa tests unitarios                            │
│     └─ Registra en sistema de versionado                     │
│                                                              │
│  3. Revisión Técnica (Tech Lead)                             │
│     └─ Valida expresión y performance                        │
│     └─ Verifica seguridad de expresión                       │
│     └─ Revisa tests                                          │
│                                                              │
│  4. Revisión de Negocio (Product Owner)                      │
│     └─ Valida contra requisitos de negocio                   │
│     └─ Aprueba casos de prueba                               │
│     └─ Autoriza activación                                   │
│                                                              │
│  5. Staging (Automated)                                      │
│     └─ Deploy en staging                                     │
│     └─ Run regression tests                                  │
│     └─ Compare results with production                       │
│                                                              │
│  6. Producción (Release Manager)                             │
│     └─ Activar regla (flip IsActive flag)                    │
│     └─ Monitorear métricas                                   │
│     └─ Rollback si necesario                                 │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### Ownership por Dominio

```csharp
public class WorkflowOwnership
{
    public static readonly Dictionary<string, OwnershipConfig> Registry = new()
    {
        ["Pricing_*"] = new OwnershipConfig
        {
            Team = "Revenue Engineering",
            ApproverRole = "PricingManager",
            RequiresSecurityReview = false,
            MaxRulesPerWorkflow = 50
        },
        ["Compliance_*"] = new OwnershipConfig
        {
            Team = "Regulatory Affairs",
            ApproverRole = "ComplianceOfficer",
            RequiresSecurityReview = true,
            MaxRulesPerWorkflow = 100
        },
        ["Fraud_*"] = new OwnershipConfig
        {
            Team = "Risk Engineering",
            ApproverRole = "FraudAnalystSenior",
            RequiresSecurityReview = true,
            MaxRulesPerWorkflow = 75
        }
    };
}
```

---

# 7. Edge Cases Reales y Riesgos

## 7.1 Null Propagation

**Escenario**: Un campo que normalmente está presente llega como null en producción.

```json
{
  "Expression": "input1.Customer.Address.City == \"Buenos Aires\""
}
```

Si `input1.Customer` es null, o `input1.Customer.Address` es null, se produce `NullReferenceException`.

**Impacto sin `HandleRuleException = true`**:
- La excepción propaga
- Se detiene la evaluación de todo el workflow
- El request del usuario falla con 500

**Impacto con `HandleRuleException = true`**:
- La regla se marca como `IsSuccess = false`
- El `ExceptionMessage` contiene el detalle
- Otras reglas continúan evaluándose
- El resultado puede ser **funcionalmente incorrecto**: la regla "falló" pero no porque la condición no se cumple, sino por error de datos

**Mitigación robusta**:

```csharp
public class NullSafeRulesPreprocessor
{
    public void ValidateInputCompleteness(object input, string workflowName)
    {
        var requiredPaths = GetRequiredPaths(workflowName);
        var missingPaths = new List<string>();
        
        foreach (var path in requiredPaths)
        {
            if (!PropertyPathExists(input, path))
            {
                missingPaths.Add(path);
            }
        }
        
        if (missingPaths.Any())
        {
            throw new IncompleteInputException(
                $"Missing required input paths for workflow '{workflowName}': " +
                string.Join(", ", missingPaths));
        }
    }
}
```

## 7.2 Excepciones dentro de Expresiones

**Escenario**: Operaciones que pueden fallar dentro de expresiones.

```json
{
  "Expression": "int.Parse(input1.StringScore) > 700"
}
```

Si `StringScore` es `"abc"` o null, `int.Parse` lanza `FormatException`.

**Escenario más sutil**: Division by zero.

```json
{
  "Expression": "input1.Revenue / input1.EmployeeCount > 100000m"
}
```

Si `EmployeeCount` es 0, se produce `DivideByZeroException`.

**Protección**:

```json
{
  "LocalParams": [
    {
      "Name": "revenuePerEmployee",
      "Expression": "input1.EmployeeCount > 0 ? input1.Revenue / input1.EmployeeCount : 0m"
    }
  ],
  "Expression": "revenuePerEmployee > 100000m"
}
```

## 7.3 Reglas Inconsistentes

**Escenario**: Dos reglas que se contradicen mutuamente.

```json
[
  {
    "RuleName": "MinimumAge",
    "Expression": "input1.Age >= 18"
  },
  {
    "RuleName": "SeniorDiscount",
    "Expression": "input1.Age < 16"
  }
]
```

Ambas reglas pueden "pasar" o "fallar" simultáneamente dependiendo del input. RulesEngine **no detecta** contradicciones lógicas.

**Mitigación**: Implementar validación de consistencia offline.

```csharp
public class RuleConsistencyChecker
{
    public List<ConsistencyWarning> Analyze(Workflow workflow)
    {
        var warnings = new List<ConsistencyWarning>();
        var rules = workflow.Rules.ToList();
        
        for (int i = 0; i < rules.Count; i++)
        {
            for (int j = i + 1; j < rules.Count; j++)
            {
                if (AreContradictory(rules[i], rules[j]))
                {
                    warnings.Add(new ConsistencyWarning(
                        $"Rules '{rules[i].RuleName}' and '{rules[j].RuleName}' " +
                        "may produce contradictory results"));
                }
            }
        }
        
        return warnings;
    }
}
```

## 7.4 Reglas Mal Versionadas

**Escenario**: Una nueva versión del workflow refiere a propiedades que no existen en el modelo actual.

```json
{
  "WorkflowName": "Pricing_v3",
  "Rules": [
    {
      "Expression": "input1.CustomerTier == \"PLATINUM\""
    }
  ]
}
```

Si el modelo de input no tiene `CustomerTier` (se agregó en una versión futura del código), la expresión falla en runtime.

**Riesgo organizacional**: El equipo de negocio actualiza las reglas asumiendo que el modelo ya cambió, pero el deploy del código aún no ocurrió.

**Mitigación**: Validación de compatibilidad al cargar workflows.

```csharp
public class WorkflowCompatibilityValidator
{
    public ValidationResult ValidateAgainstModel<TInput>(Workflow workflow, string paramName)
    {
        var inputType = typeof(TInput);
        var errors = new List<string>();
        
        foreach (var rule in workflow.Rules)
        {
            var properties = ExtractPropertyPaths(rule.Expression, paramName);
            
            foreach (var propPath in properties)
            {
                if (!PropertyExists(inputType, propPath))
                {
                    errors.Add(
                        $"Rule '{rule.RuleName}' references '{paramName}.{propPath}' " +
                        $"which does not exist on {inputType.Name}");
                }
            }
        }
        
        return errors.Any() 
            ? ValidationResult.Failure(errors) 
            : ValidationResult.Success();
    }
}
```

## 7.5 Cambios Dinámicos en Runtime

**Riesgo**: Si se permite editar reglas en producción sin pipeline de validación, un cambio incorrecto puede:

1. Aprobar préstamos que deberían rechazarse (riesgo financiero)
2. Bloquear transacciones legítimas (riesgo operacional)
3. Violar regulaciones (riesgo legal)
4. Ejecutar código malicioso (riesgo de seguridad)

**Controles obligatorios**:

```csharp
public class RuleChangeGuard
{
    public async Task<bool> CanActivateAsync(Workflow newWorkflow, string changedBy)
    {
        // 1. Validación de seguridad
        if (!SecurityValidator.ValidateAllExpressions(newWorkflow))
            return false;
        
        // 2. Aprobación dual
        if (!await HasDualApproval(newWorkflow.WorkflowName, changedBy))
            return false;
        
        // 3. Regression testing en staging
        if (!await RegressionTestsPassed(newWorkflow))
            return false;
        
        // 4. Gradual rollout (canary)
        await EnableForCanaryGroup(newWorkflow, percentTraffic: 5);
        await Task.Delay(TimeSpan.FromMinutes(30));
        
        if (!await CanaryMetricsHealthy(newWorkflow.WorkflowName))
        {
            await RollbackCanary(newWorkflow.WorkflowName);
            return false;
        }
        
        return true;
    }
}
```

## 7.6 Problemas de Mantenimiento

### Regla Huérfana

Una regla que ya no es evaluada por ningún flujo de la aplicación pero sigue consumiendo recursos.

### Regla Zombie

Una regla con `Enabled = false` hace meses que nadie recuerda por qué se desactivó.

### Explosion Combinatoria

Un workflow que crece sin control hasta tener 500+ reglas porque cada caso especial se maneja con una nueva regla.

**Métricas de salud**:

```csharp
public class RulesHealthReport
{
    public int TotalWorkflows { get; set; }
    public int TotalRules { get; set; }
    public int DisabledRules { get; set; }
    public int RulesWithExceptions { get; set; }   // Reglas que fallan por excepción
    public int RulesNeverEvaluated { get; set; }    // En últimos 30 días
    public double AverageRulesPerWorkflow { get; set; }
    public int WorkflowsAboveThreshold { get; set; } // > 100 reglas
    public DateTime OldestRuleLastModified { get; set; }
}
```

## 7.7 Riesgos de Seguridad

| Riesgo | Vector | Probabilidad | Impacto | Mitigación |
|---|---|---|---|---|
| Code Injection | Expresión maliciosa en JSON | Media | Crítico | Whitelist de tipos, validación de expresiones |
| DoS | Expresión con loop infinito | Baja | Alto | Timeout de evaluación, complejidad máxima |
| Data Exfiltration | Expresión que accede a datos sensibles | Baja | Crítico | Sandboxing, no exponer tipos sensibles |
| Privilege Escalation | Expresión que invoca método privilegiado | Media | Crítico | CustomTypes restrictivo, security review |
| Info Disclosure | ExceptionMessage expone internals | Alta | Medio | Sanitizar mensajes antes de retornar al cliente |

## 7.8 Riesgos Organizacionales

1. **Shadow IT**: Equipos de negocio editando reglas sin supervisión técnica, introduciendo expresiones ineficientes o incorrectas.

2. **Knowledge Silos**: Solo una persona entiende las reglas más complejas. Si deja la organización, las reglas se vuelven inmantenibles.

3. **Governance Theatrical**: Proceso de aprobación formal pero sin tests automatizados — se aprueba todo porque nadie verifica realmente.

4. **Proliferación Descontrolada**: Cada ticket de negocio genera una nueva regla. En 2 años hay 2000 reglas y nadie sabe cuáles son realmente necesarias.

5. **False Sense of Testing**: Tests que solo verifican que la regla "compila" pero no que produce resultados correctos para inputs boundary.

---

# 8. Conclusión Estratégica

## 8.1 Perfil Ideal de Proyecto

Microsoft.RulesEngine es ideal para proyectos que cumplan **todos** estos criterios:

- **Ecosistema .NET**: El equipo tiene experiencia sólida en C# y .NET
- **Reglas independientes**: Cada regla puede evaluarse sin depender del resultado de otras
- **Cambio frecuente**: Las reglas de negocio cambian más rápido que el ciclo de deploy
- **Stakeholders mixtos**: Participan tanto personas técnicas como no técnicas en la definición de reglas
- **Auditoría requerida**: Se necesita registro de qué regla se evaluó, cuándo, con qué inputs
- **Volumen moderado**: Miles de evaluaciones por minuto (no millones por segundo)
- **Latencia tolerante**: El SLA permite 1-10ms de overhead por evaluación

**Industrias y casos típicos**: Finanzas (underwriting, pricing, compliance), seguros (suscripción, claims), e-commerce (elegibilidad, descuentos), salud (protocolos de tratamiento), gobierno (elegibilidad de beneficios).

## 8.2 Perfil Donde es Mala Elección

- **Pocas reglas estáticas** (<10 reglas que cambian anualmente): El overhead de setup no se justifica
- **Performance ultra-crítico**: HFT, procesamiento de streams en tiempo real, gaming engines
- **Inferencia requerida**: Sistemas donde las reglas deben encadenarse y generar nuevos hechos
- **Equipo no .NET**: Las expresiones requieren conocimiento de C#
- **Sin gobernanza**: Si no hay proceso de validación, el motor externalizado se vuelve más peligroso que el código compilado
- **Reglas sobre colecciones masivas**: Filtrar millones de registros por regla es más eficiente con SQL o Spark

## 8.3 Checklist Antes de Producción

```markdown
### Pre-Producción: Checklist Obligatorio

#### Infraestructura
- [ ] RulesEngine registrado como Singleton en DI
- [ ] Background service para recarga de reglas configurado
- [ ] Warmup service implementado para evitar cold-start en primera request
- [ ] Health check endpoint que verifica que los workflows están cargados

#### Seguridad
- [ ] CustomTypes limitado a whitelist explícita
- [ ] ExpressionSecurityValidator implementado
- [ ] JSON de reglas almacenado en fuente controlada (no editable directamente en producción)
- [ ] Audit trail de cambios de reglas

#### Performance
- [ ] Benchmark de los workflows principales ejecutado
- [ ] Cold start medido y documentado
- [ ] Estrategia de warmup implementada
- [ ] Alertas configuradas para latencia anómala

#### Testing
- [ ] Tests unitarios para cada regla individual
- [ ] Tests de integración para cada workflow completo  
- [ ] Tests de regresión entre versiones de workflows
- [ ] Tests de boundary conditions (null, zero, negative, max values)
- [ ] Tests de carga con volumen esperado de producción

#### Observabilidad
- [ ] Logging estructurado con nombre de workflow y regla
- [ ] Métricas de evaluación (duración, tasa de éxito/fallo)
- [ ] Alertas por reglas que fallan consistentemente por excepción
- [ ] Dashboard con salud del motor de reglas

#### Gobernanza
- [ ] Proceso de aprobación dual documentado
- [ ] Ownership por workflow asignado
- [ ] Runbook de rollback documentado
- [ ] Proceso de revisión periódica de reglas activas (quarterly)

#### Resiliencia
- [ ] Fallback/default behavior si el motor falla
- [ ] Circuit breaker implementado para evaluaciones
- [ ] Reglas cacheadas persisten si la fuente de reglas no está disponible
- [ ] Timeout configurado para evaluaciones individuales
```

## 8.4 Roadmap de Adopción Empresarial

### Fase 1: Proof of Concept (2-4 semanas)

- Seleccionar un caso de uso simple y acotado (eligibilidad, validación)
- Implementar 5-15 reglas
- Evaluar: facilidad de implementación, performance, developer experience
- Entregable: POC funcional + documento de evaluación

### Fase 2: Pilot (1-2 meses)

- Implementar en un servicio no crítico en producción
- Establecer patrones de testing y deployment
- Construir infraestructura de observabilidad
- Definir estándares de naming y organización de workflows
- Entregable: servicio en producción + estándares documentados

### Fase 3: Scaling (2-3 meses)

- Extender a 2-3 servicios adicionales
- Implementar pipeline de CI/CD para reglas
- Construir herramienta de administración de reglas (si es necesario)
- Capacitar equipos de negocio en formato de reglas
- Entregable: múltiples servicios usando RulesEngine + tooling

### Fase 4: Enterprise (continuo)

- Centralizar gobernanza de reglas
- Implementar auditoría cross-workflow
- Construir catálogo de reglas con search y discovery
- Establecer métricas de salud del ecosistema de reglas
- Evaluar necesidad de motor de inferencia (NRules) para casos avanzados

## 8.5 Recomendaciones Finales de Arquitecto

1. **Empezar simple**: No implementar toda la infraestructura de gobernanza en el día 1. Empezar con workflows en archivos JSON, evolucionar a base de datos cuando el volumen lo justifique.

2. **El motor no es la arquitectura**: RulesEngine es una herramienta, no una solución. La arquitectura correcta incluye versionado, testing, observabilidad y gobernanza.

3. **Invertir en testing desde el principio**: Cada regla debería tener al menos 5 test cases (happy path, boundary, null, error, combination). El costo de un bug en producción en lógica externalizada es mayor que en código compilado porque el ciclo de feedback es más largo.

4. **No externalizar todo**: No toda lógica condicional necesita ser dinámica. Solo externalizar lógica que realmente cambia con frecuencia y cuyo cambio tiene valor de negocio.

5. **Monitorear la salud de las reglas**: Una regla que falla el 100% del tiempo probablemente tiene un bug. Una regla que pasa el 100% del tiempo probablemente es innecesaria. Ambas merecen investigación.

6. **Planificar el rollback**: Cada activación de reglas nuevas debe tener un plan de rollback que pueda ejecutarse en menos de 5 minutos. Esto incluye mantener la versión anterior activa y disponible.

7. **Documentar la intención**: Cada regla debería tener un `ErrorMessage` descriptivo y, si es posible, un campo de metadata explicando por qué existe la regla, quién la solicitó, y cuándo debería revisarse.

8. **Considerar el bus factor**: Si solo una persona entiende las reglas de un workflow, eso es un riesgo organizacional equivalente a deuda técnica crítica.

9. **Separar responsabilidades**: El equipo que define reglas (negocio) no debería tener acceso directo a producción. El equipo que despliega (operaciones) no debería definir reglas. El equipo que desarrolla (engineering) debería facilitar ambos procesos.

10. **Evaluar periódicamente**: Cada 6 meses, revisar si el motor sigue siendo la herramienta correcta. Los requisitos cambian. Lo que empezó como 20 reglas independientes puede evolucionar a un sistema que necesita inferencia. Estar preparado para migrar si es necesario.

---

> **Documento generado como guía técnica enterprise de referencia.**  
> **Versión**: 1.0  
> **Plataforma**: .NET 8+ / Microsoft.RulesEngine 5.x


# Microsoft.RulesEngine — Guía Técnica Definitiva

---

# MÓDULO 1 — Fundamentos y Filosofía Arquitectónica

## 1.1 Definición Técnica

Microsoft.RulesEngine es una librería open-source de Microsoft que implementa un motor de evaluación de reglas basado en expresiones lambda dinámicas. Opera sobre JSON como formato de definición de reglas, compilando expresiones textuales a Expression Trees de .NET en tiempo de ejecución a través de la librería `System.Linq.Dynamic.Core`. No es un motor de inferencia, no implementa el algoritmo Rete, no soporta backward chaining y no mantiene estado entre evaluaciones. Es un evaluador stateless de condiciones booleanas con capacidad de ejecutar acciones asociadas al resultado.

El repositorio vive en `github.com/microsoft/RulesEngine`, distribuido vía NuGet como `RulesEngine`. La versión estable actual es la 5.x, compatible con .NET 6, .NET 7 y .NET 8. El proyecto tiene actividad moderada de mantenimiento — recibe patches y PRs comunitarios, pero no evoluciona con ritmo de framework activo. Esto es relevante para decisiones de adopción a largo plazo.

## 1.2 Problema Arquitectónico que Resuelve

En cualquier sistema empresarial, existe una tensión entre lógica que cambia frecuentemente (políticas comerciales, umbrales regulatorios, reglas de elegibilidad) y lógica que es estructuralmente estable (flujos de proceso, integraciones, modelos de dominio). Cuando ambas coexisten en el mismo código compilado, cada cambio de política requiere un ciclo completo de desarrollo, build, QA y deploy.

Microsoft.RulesEngine resuelve esto externalizando la definición de condiciones evaluables a un formato de datos (JSON) que puede almacenarse, versionarse y modificarse independientemente del código compilado. El motor actúa como un intérprete que recibe:

1. Un conjunto de reglas definidas en JSON (workflows).
2. Uno o más objetos de datos como input (parámetros).
3. Devuelve un árbol de resultados indicando qué reglas se cumplieron y cuáles no.

El ciclo de vida de las reglas se desacopla del ciclo de vida del deployment de la aplicación. Negocio puede proponer cambios en reglas, que se validan y despliegan sin recompilar código.

## 1.3 Expression-Based vs Inference-Based: Implicancias Profundas

Esta distinción es fundamental y determina todo lo que el motor puede y no puede hacer.

**Motor basado en expresiones (Microsoft.RulesEngine):**

- Cada regla es una expresión booleana independiente que se evalúa contra un conjunto fijo de inputs.
- No existe noción de "hechos" que se propagan entre reglas.
- No hay forward chaining: una regla no dispara automáticamente la evaluación de otras reglas.
- No hay backward chaining: no se puede preguntar "¿qué condiciones necesito cumplir para que X sea verdadero?".
- La evaluación es un single-pass: se evalúan todas las reglas una vez y se devuelve el resultado.
- El modelo mental es: `f(inputs) → results[]`.

**Motor basado en inferencia (NRules, Drools):**

- Las reglas operan sobre una "memoria de trabajo" (working memory) con hechos.
- Insertar un hecho puede disparar reglas que a su vez insertan nuevos hechos (forward chaining).
- Se puede razonar en reversa: dado un objetivo, ¿qué hechos necesito? (backward chaining).
- La evaluación es iterativa: múltiples pasadas hasta alcanzar un estado estable.
- El modelo mental es: `while(hay_reglas_pendientes) { evaluar_y_propagar() }`.

**Trade-offs concretos:**


| Aspecto                          | Expression-Based                     | Inference-Based             |
| -------------------------------- | ------------------------------------ | --------------------------- |
| Complejidad de adopción          | Baja                                 | Alta                        |
| Performance predecible           | Sí (O(n) reglas)                     | No (depende de propagación) |
| Capacidad de razonamiento        | Limitada                             | Rica                        |
| Debugging                        | Directo                              | Complejo                    |
| Idoneidad para validaciones      | Excelente                            | Overkill                    |
| Idoneidad para lógica encadenada | Pobre (requiere orquestación manual) | Nativa                      |


Microsoft.RulesEngine es la elección correcta cuando se necesita evaluar condiciones independientes o semi-independientes sobre datos de entrada conocidos. No es la elección correcta cuando la lógica de negocio requiere razonamiento encadenado, propagación de hechos o resolución de conflictos entre reglas.

## 1.4 Cuándo Aporta Valor Real vs Cuándo es Sobreingeniería

**Aporta valor real cuando:**

- Las reglas cambian más de una vez al mes sin correlación con cambios de código.
- Diferentes clientes, países o productos necesitan reglas distintas (multi-tenancy de reglas).
- El negocio necesita autonomía para ajustar parámetros sin depender del equipo de desarrollo.
- Existe un volumen significativo de reglas (>20) que hacen impráctico el mantenimiento en código.
- Se requiere auditoría de qué reglas se evaluaron y con qué resultado.
- Se necesita versionado y rollback de reglas como artefacto independiente.

**Es sobreingeniería cuando:**

- Las reglas cambian una o dos veces al año (un if/else bien testeado es suficiente).
- Hay menos de 10 reglas simples y estables.
- El equipo no tiene madurez para gestionar reglas como artefacto separado.
- La lógica requiere acceso a I/O (base de datos, APIs) durante la evaluación — el motor no está diseñado para esto.
- Se necesita transaccionalidad: las reglas evalúan, no ejecutan operaciones transaccionales.

## 1.5 Modelo Mental Correcto

El modelo mental más productivo para pensar en Microsoft.RulesEngine es el de un **evaluador de políticas stateless**. No es un "cerebro" que razona. Es un filtro configurable:

```
┌─────────────────┐    ┌──────────────────┐    ┌─────────────────┐
│   Datos Input    │───▶│   Motor de       │───▶│  Árbol de       │
│  (RuleParameter) │    │   Evaluación     │    │  Resultados     │
└─────────────────┘    │  (Workflows +    │    │  (RuleResult    │
                       │   Rules)         │    │   Tree)         │
                       └──────────────────┘    └─────────────────┘
```

Entra un objeto, sale un veredicto. Las reglas son la configuración del filtro. La aplicación que consume los resultados es la que toma decisiones y ejecuta acciones.

## 1.6 Reglas de Negocio vs Lógica de Dominio vs Políticas Operativas

Esta distinción es crítica para decidir qué poner en el motor y qué no:

- **Invariantes de dominio:** Condiciones que siempre deben ser verdaderas. Ejemplo: "Un pedido no puede tener monto negativo". Estas NO van en el motor de reglas — pertenecen al código de dominio (validaciones en el Aggregate).
- **Reglas de negocio variables:** Condiciones que dependen de políticas comerciales. Ejemplo: "Clientes con score > 700 califican para tasa preferencial". Estas SÍ son candidatas al motor.
- **Políticas operativas:** Configuración de parámetros. Ejemplo: "El monto máximo de transferencia es 50,000 USD". Esto puede ser una regla simple o simplemente configuración. Si hay muchos parámetros interrelacionados, el motor aporta valor.

## 1.7 Análisis Crítico: Fortalezas y Debilidades

**Fortalezas genuinas:**

- API extremadamente simple: se aprende en horas, no en semanas.
- JSON como formato de reglas: fácil de almacenar, versionar, transportar.
- Expression Trees compilados: performance razonable después de la primera compilación.
- Evaluación stateless: naturalmente thread-safe en la evaluación (con caveats que veremos).
- Extensible: Custom Types y Custom Functions permiten extender el vocabulario de expresiones.
- Sin dependencias pesadas: no requiere infraestructura adicional.

**Debilidades reales:**

- No hay validación de expresiones en design-time: errores se descubren en runtime.
- Las expresiones son strings: sin IntelliSense, sin refactoring seguro, sin detección de tipos en compilación.
- No hay UI de gestión incluida: se necesita construir tooling de administración.
- La documentación oficial es limitada y superficial.
- El parser de expresiones hereda limitaciones de `System.Linq.Dynamic.Core`.
- No hay soporte nativo para versionado, auditoría o rollback — todo debe construirse.
- El mecanismo de acciones (Actions) es limitado comparado con motores más maduros.
- Sin soporte para forward/backward chaining — toda orquestación es manual.

## 1.8 Ejemplo Introductorio End-to-End

### Modelo de datos

```csharp
public class LoanApplication
{
    public string ApplicantName { get; set; }
    public int Age { get; set; }
    public decimal AnnualIncome { get; set; }
    public int CreditScore { get; set; }
    public decimal RequestedAmount { get; set; }
    public int EmploymentYears { get; set; }
    public decimal ExistingDebt { get; set; }
}
```

### JSON de Workflow

```json
[
  {
    "WorkflowName": "LoanEligibility",
    "Rules": [
      {
        "RuleName": "AgeRequirement",
        "SuccessEvent": "Age requirement met",
        "ErrorMessage": "Applicant must be between 18 and 65 years old",
        "ErrorType": "Error",
        "RuleExpressionType": "LambdaExpression",
        "Expression": "input1.Age >= 18 AND input1.Age <= 65"
      },
      {
        "RuleName": "MinimumIncome",
        "SuccessEvent": "Income requirement met",
        "ErrorMessage": "Annual income must be at least 24000",
        "ErrorType": "Error",
        "RuleExpressionType": "LambdaExpression",
        "Expression": "input1.AnnualIncome >= 24000"
      },
      {
        "RuleName": "CreditScoreCheck",
        "SuccessEvent": "Credit score acceptable",
        "ErrorMessage": "Credit score must be at least 600",
        "ErrorType": "Error",
        "RuleExpressionType": "LambdaExpression",
        "Expression": "input1.CreditScore >= 600"
      },
      {
        "RuleName": "DebtToIncomeRatio",
        "SuccessEvent": "DTI ratio acceptable",
        "ErrorMessage": "Debt-to-income ratio exceeds 40%",
        "ErrorType": "Error",
        "RuleExpressionType": "LambdaExpression",
        "LocalParams": [
          {
            "Name": "dtiRatio",
            "Expression": "decimal.Divide(input1.ExistingDebt, input1.AnnualIncome) * 100"
          }
        ],
        "Expression": "dtiRatio <= 40"
      }
    ]
  }
]
```

### Código de ejecución

```csharp
using RulesEngine.Models;
using System.Text.Json;

// Cargar reglas desde JSON
var workflowJson = File.ReadAllText("rules/loan-eligibility.json");
var workflows = JsonSerializer.Deserialize<Workflow[]>(workflowJson);

// Crear instancia del motor
var rulesEngine = new RulesEngine.RulesEngine(workflows, new ReSettings
{
    CustomTypes = Array.Empty<Type>()
});

// Datos de entrada
var application = new LoanApplication
{
    ApplicantName = "Carlos Méndez",
    Age = 35,
    AnnualIncome = 85000m,
    CreditScore = 720,
    RequestedAmount = 250000m,
    EmploymentYears = 8,
    ExistingDebt = 15000m
};

// Ejecutar reglas
var ruleParameter = new RuleParameter("input1", application);
List<RuleResultTree> results = await rulesEngine.ExecuteAllRulesAsync("LoanEligibility", ruleParameter);

// Procesar resultados
foreach (var result in results)
{
    Console.WriteLine($"Rule: {result.Rule.RuleName}");
    Console.WriteLine($"  Success: {result.IsSuccess}");

    if (!result.IsSuccess)
    {
        Console.WriteLine($"  Error: {result.Rule.ErrorMessage}");
    }
    else
    {
        Console.WriteLine($"  Event: {result.Rule.SuccessEvent}");
    }
}

bool allPassed = results.TrueForAll(r => r.IsSuccess);
Console.WriteLine($"\nLoan Decision: {(allPassed ? "APPROVED" : "REJECTED")}");
```

### Flujo de Ejecución Línea por Línea

1. **Deserialización:** El JSON se deserializa a un array de `Workflow`. Cada `Workflow` contiene un nombre y una colección de `Rule`.
2. **Inicialización del motor:** `new RulesEngine(workflows, settings)` registra los workflows internamente. En este momento NO se compilan las expresiones — la compilación es lazy, ocurre en la primera evaluación.
3. **Creación de RuleParameter:** El objeto `LoanApplication` se envuelve en un `RuleParameter` con nombre `"input1"`. Este nombre es el que se usa en las expresiones JSON para referenciar el objeto.
4. **ExecuteAllRulesAsync:** El motor localiza el workflow `"LoanEligibility"`, itera sobre sus reglas y para cada una:
  - Si tiene `LocalParams`, evalúa cada parámetro local secuencialmente (en orden de declaración).
  - Compila la expresión a un Expression Tree (si no está en caché).
  - Evalúa la expresión compilada contra el input.
  - Construye un `RuleResultTree` con el resultado.
5. **RuleResultTree:** Cada nodo contiene `IsSuccess` (bool), referencia a la `Rule` original, `ExceptionMessage` (si hubo error), `ActionResult` (si hay acciones configuradas) y `ChildResults` (para reglas anidadas).

---

# MÓDULO 2 — Arquitectura Interna en Profundidad

## 2.1 Workflow

Un `Workflow` es la unidad organizativa de mayor nivel. Representa un conjunto cohesivo de reglas que se evalúan juntas. Conceptualmente equivale a una "política" o un "conjunto de criterios" para un proceso de negocio específico.

```csharp
public class Workflow
{
    public string WorkflowName { get; set; }
    public IEnumerable<Rule> Rules { get; set; }
    public IEnumerable<ScopedParam> GlobalParams { get; set; }
    public string RuleExpressionType { get; set; }
}
```

**Estrategia de agrupación recomendada:** Un workflow por proceso de decisión. No mezclar reglas de procesos distintos en el mismo workflow. Ejemplos:

- `CreditApproval_v2` — reglas de aprobación crediticia versión 2.
- `FraudDetection_RealTime` — reglas de detección de fraude en tiempo real.
- `Commission_CreditCard_Premium` — reglas de comisión para tarjetas premium.

**GlobalParams:** Parámetros calculados disponibles para todas las reglas del workflow. Se evalúan una vez antes de las reglas. Útiles para cálculos que se reutilizan en múltiples reglas.

## 2.2 Rule — Anatomía Completa

```csharp
public class Rule
{
    public string RuleName { get; set; }
    public Dictionary<string, object> Properties { get; set; }
    public string Operator { get; set; }
    public string ErrorMessage { get; set; }
    public ErrorType ErrorType { get; set; }
    public RuleExpressionType RuleExpressionType { get; set; }
    public IEnumerable<string> WorkflowsToInject { get; set; }
    public IEnumerable<Rule> Rules { get; set; }
    public IEnumerable<ScopedParam> LocalParams { get; set; }
    public string Expression { get; set; }
    public RuleActions Actions { get; set; }
    public string SuccessEvent { get; set; }
    public bool Enabled { get; set; }
}
```

**Campo por campo:**


| Campo                | Tipo                       | Default            | Descripción                                                                          |
| -------------------- | -------------------------- | ------------------ | ------------------------------------------------------------------------------------ |
| `RuleName`           | string                     | requerido          | Identificador único dentro del workflow. Se usa en resultados y logging.             |
| `Expression`         | string                     | null               | Expresión lambda evaluable. Requerida para `LambdaExpression`. Se ignora en AND/OR.  |
| `RuleExpressionType` | enum                       | `LambdaExpression` | Determina cómo se evalúa: como expresión directa o como operador sobre reglas hijas. |
| `Operator`           | string                     | null               | Para reglas anidadas: `"And"`, `"Or"`, `"AndAlso"`, `"OrElse"`.                      |
| `Rules`              | `IEnumerable<Rule>`        | null               | Reglas hijas para composición AND/OR.                                                |
| `LocalParams`        | `IEnumerable<ScopedParam>` | null               | Variables calculadas locales, disponibles en la expresión de esta regla.             |
| `Actions`            | `RuleActions`              | null               | Acciones a ejecutar según resultado (OnSuccess, OnFailure).                          |
| `SuccessEvent`       | string                     | null               | Evento/mensaje emitido cuando la regla se cumple.                                    |
| `ErrorMessage`       | string                     | null               | Mensaje cuando la regla falla.                                                       |
| `ErrorType`          | enum                       | `Warning`          | Severidad del error: `Error` o `Warning`.                                            |
| `Enabled`            | bool                       | `true`             | Si `false`, la regla se omite completamente en la evaluación.                        |
| `Properties`         | Dictionary                 | null               | Diccionario genérico para metadata custom (tags, categorías, prioridades).           |
| `WorkflowsToInject`  | `IEnumerable<string>`      | null               | Workflows adicionales cuyas reglas se inyectan como reglas hijas.                    |


**Implicancias importantes:**

- `Enabled = false` es equivalente a que la regla no exista. No genera resultado en el `RuleResultTree`. Útil para feature flags a nivel de regla.
- `Properties` no se usa internamente por el motor. Es metadata pura para uso de la aplicación consumidora (categorización, prioridad, tags de auditoría).
- `WorkflowsToInject` permite composición entre workflows: una regla puede "importar" las reglas de otro workflow como sus hijas. Esto habilita reuso, pero agrega complejidad al debugging.

## 2.3 RuleParameter — Mapeo de Inputs

```csharp
public class RuleParameter
{
    public string Name { get; set; }
    public Type Type { get; set; }
    public object Value { get; set; }

    public RuleParameter(string name, object value) { ... }
}
```

El `Name` es crítico: es el identificador usado en las expresiones para referenciar el objeto. Cuando se pasa un solo parámetro, el motor lo nombra automáticamente como `input1`. Con múltiples parámetros, se sigue la convención `input1`, `input2`, etc., o se usan nombres explícitos.

```csharp
// Un solo input — accesible como "input1" en expresiones
var param = new RuleParameter("input1", myObject);

// Múltiples inputs — accesibles por su nombre
var params = new[] {
    new RuleParameter("applicant", applicantData),
    new RuleParameter("product", productConfig),
    new RuleParameter("market", marketConditions)
};

// En la expresión JSON:
// "applicant.CreditScore > 600 AND product.MinScore <= applicant.CreditScore"
```

**Navegación de propiedades anidadas:** Las expresiones acceden a propiedades via dot-notation estándar: `input1.Address.City == "Lima"`. El parser resuelve la cadena de propiedades a través de Expression Trees. Si una propiedad intermedia es null, se lanza `NullReferenceException` a menos que se maneje explícitamente.

**Tipado dinámico:** El motor infiere el tipo del objeto en runtime. Esto significa que las expresiones deben coincidir con los tipos reales de las propiedades. Si la expresión compara `input1.Amount > 1000` y `Amount` es `decimal`, la constante `1000` se convierte automáticamente. Pero si `Amount` es `string`, fallará en runtime — no hay validación en design-time.

## 2.4 RuleResultTree

```csharp
public class RuleResultTree
{
    public Rule Rule { get; set; }
    public bool IsSuccess { get; set; }
    public string ExceptionMessage { get; set; }
    public IEnumerable<RuleResultTree> ChildResults { get; set; }
    public ActionResult ActionResult { get; set; }
    public RuleInput Input { get; set; }
}
```

Cada `RuleResultTree` es un nodo que puede contener hijos (`ChildResults`) cuando la regla es compuesta (AND/OR con reglas anidadas). Esto crea un árbol donde la raíz es el resultado de la regla padre y las hojas son los resultados de las reglas atómicas.

**Extracción de información útil:**

```csharp
List<RuleResultTree> results = await engine.ExecuteAllRulesAsync("WorkflowName", param);

// Todas las reglas que fallaron
var failures = results.Where(r => !r.IsSuccess);

// Mensajes de error
var errorMessages = failures
    .Select(r => r.Rule.ErrorMessage)
    .Where(m => !string.IsNullOrEmpty(m))
    .ToList();

// Extraer valores de acciones OutputExpression
var outputValues = results
    .Where(r => r.IsSuccess && r.ActionResult?.Output != null)
    .ToDictionary(
        r => r.Rule.RuleName,
        r => r.ActionResult.Output
    );

// Navegar resultados anidados recursivamente
void TraverseResults(IEnumerable<RuleResultTree> nodes, int depth = 0)
{
    foreach (var node in nodes)
    {
        var indent = new string(' ', depth * 2);
        Console.WriteLine($"{indent}{node.Rule.RuleName}: {node.IsSuccess}");

        if (node.ChildResults?.Any() == true)
        {
            TraverseResults(node.ChildResults, depth + 1);
        }
    }
}
TraverseResults(results);
```

## 2.5 RuleExpressionType en Profundidad

```csharp
public enum RuleExpressionType
{
    LambdaExpression,
    All,    // AND lógico sobre reglas hijas
    Any     // OR lógico sobre reglas hijas
}
```

**LambdaExpression:** La regla contiene una expresión en el campo `Expression` que se evalúa directamente. Es la forma atómica — la hoja del árbol de reglas.

```json
{
  "RuleName": "IncomeCheck",
  "RuleExpressionType": "LambdaExpression",
  "Expression": "input1.AnnualIncome >= 30000"
}
```

**All (AND lógico):** La regla no tiene `Expression` propia. En su lugar, contiene reglas hijas en el campo `Rules`. La regla padre se cumple solo si TODAS las hijas se cumplen.

```json
{
  "RuleName": "BasicEligibility",
  "RuleExpressionType": "All",
  "Operator": "And",
  "Rules": [
    {
      "RuleName": "AgeCheck",
      "RuleExpressionType": "LambdaExpression",
      "Expression": "input1.Age >= 18"
    },
    {
      "RuleName": "IncomeCheck",
      "RuleExpressionType": "LambdaExpression",
      "Expression": "input1.AnnualIncome >= 30000"
    }
  ]
}
```

**Any (OR lógico):** La regla padre se cumple si AL MENOS UNA hija se cumple.

**Cuándo usar cada uno:**

- `LambdaExpression` para condiciones evaluables directamente.
- `All` cuando un requisito compuesto exige que se cumplan todas las sub-condiciones (eligibilidad con múltiples criterios obligatorios).
- `Any` cuando basta con cumplir una de varias alternativas (múltiples fuentes de ingreso válidas, múltiples documentos aceptables).

**Composición profunda:** Se pueden anidar AND dentro de OR y viceversa, creando árboles lógicos complejos. El `RuleResultTree` refleja esta estructura, permitiendo saber exactamente qué sub-regla específica falló dentro de una composición compleja.

## 2.6 Expression Trees Internos

Cuando el motor encuentra una expresión como `"input1.Age >= 18 AND input1.CreditScore > 600"`, el proceso interno es:

1. **Parsing:** `System.Linq.Dynamic.Core` parsea el string y genera un AST (Abstract Syntax Tree).
2. **Compilación:** El AST se convierte a un `Expression<Func<RuleParameter[], bool>>` de .NET.
3. **Compilación a IL:** El Expression Tree se compila a un delegate de IL optimizado via `Expression.Compile()`.
4. **Caché:** El delegate compilado se almacena en memoria asociado al hash de la expresión + workflow.
5. **Evaluación:** En ejecuciones posteriores, se reutiliza el delegate compilado directamente.

**Costos de compilación:** La compilación de Expression Trees es la operación más costosa del motor. Para una expresión de complejidad media, la primera compilación puede tomar entre 1-10ms. Las evaluaciones subsecuentes (usando el delegate cacheado) toman microsegundos. Esto hace que el patrón ideal sea: compilar una vez al inicio y evaluar miles de veces.

```
Primera evaluación:  Parse(1-3ms) + Compile(2-8ms) + Eval(<0.1ms) ≈ 3-11ms
Evaluaciones siguientes:  Cache-hit + Eval(<0.1ms) ≈ <0.1ms
```

## 2.7 Flujo de Ejecución Interno Paso a Paso

```
ExecuteAllRulesAsync("WorkflowName", ruleParams)
│
├─ 1. Buscar Workflow por nombre en diccionario interno
│     └─ Si no existe → throw WorkflowNotFoundException
│
├─ 2. Resolver GlobalParams del Workflow
│     └─ Evaluar cada GlobalParam secuencialmente
│        └─ El resultado queda disponible para todas las reglas
│
├─ 3. Para cada Rule en el Workflow (where Enabled == true):
│     │
│     ├─ 3a. Resolver LocalParams de la regla
│     │      └─ Evaluar cada LocalParam secuencialmente
│     │         (pueden depender de GlobalParams y de LocalParams anteriores)
│     │
│     ├─ 3b. Según RuleExpressionType:
│     │      │
│     │      ├─ LambdaExpression:
│     │      │    ├─ Buscar delegate compilado en caché
│     │      │    ├─ Si no existe → compilar Expression Tree → cachear
│     │      │    └─ Ejecutar delegate con los inputs
│     │      │
│     │      ├─ All (AND):
│     │      │    └─ Evaluar recursivamente cada regla hija
│     │      │       └─ IsSuccess = todas las hijas son true
│     │      │
│     │      └─ Any (OR):
│     │           └─ Evaluar recursivamente cada regla hija
│     │              └─ IsSuccess = al menos una hija es true
│     │
│     ├─ 3c. Si la evaluación lanza excepción:
│     │      ├─ IsSuccess = false
│     │      └─ ExceptionMessage = mensaje de la excepción
│     │
│     └─ 3d. Si hay Actions configuradas:
│            ├─ OnSuccess → si IsSuccess == true, ejecutar acción
│            └─ OnFailure → si IsSuccess == false, ejecutar acción
│
└─ 4. Retornar List<RuleResultTree> con todos los resultados
```

## 2.8 Manejo de Null

El motor NO implementa null-safe navigation por defecto. Si una expresión es `"input1.Address.City == \"Lima\""` y `Address` es null, se lanza `NullReferenceException`. El motor captura esta excepción y la reporta en `ExceptionMessage` del resultado, con `IsSuccess = false`.

**Estrategias de protección:**

```json
{
  "Expression": "input1.Address != null AND input1.Address.City == \"Lima\""
}
```

Usando el operador `AND` (no `&&`), `System.Linq.Dynamic.Core` puede implementar evaluación short-circuit en algunos casos, pero el comportamiento no es garantizado de la misma forma que `&&` en C#. La estrategia más segura es separar la validación de null en una regla anidada previa.

Con `ReSettings.EnableExceptionAsErrorMessage = true`, las excepciones de null se capturan automáticamente y el resultado es `IsSuccess = false` con el mensaje de excepción. Sin esta configuración, la excepción puede propagarse dependiendo de la versión.

## 2.9 Manejo de Excepciones Internas

Cuando una expresión lanza excepción:

1. El motor la captura internamente (try/catch alrededor de la evaluación).
2. El `RuleResultTree` se genera con `IsSuccess = false`.
3. `ExceptionMessage` contiene el mensaje de la excepción.
4. La evaluación de las demás reglas del workflow **continúa normalmente**. Una regla que falla por excepción NO detiene la evaluación de las siguientes.

Esto es un diseño deliberado: en un conjunto de 50 reglas, si la regla 3 falla por un null reference, las reglas 4-50 siguen evaluándose. El resultado final contiene la información completa de todas las reglas.

**Riesgo:** Si no se inspecciona `ExceptionMessage`, una regla que falla por excepción se ve igual que una regla que falla por condición no cumplida — ambas tienen `IsSuccess = false`. Siempre se debe verificar `ExceptionMessage` para distinguir fallos legítimos de errores técnicos.

```csharp
var technicalErrors = results
    .Where(r => !r.IsSuccess && !string.IsNullOrEmpty(r.ExceptionMessage))
    .ToList();

if (technicalErrors.Any())
{
    logger.LogError("Rules failed due to technical errors: {Errors}",
        technicalErrors.Select(e => new {
            Rule = e.Rule.RuleName,
            Error = e.ExceptionMessage
        }));
    throw new RuleEvaluationException("Technical error during rule evaluation", technicalErrors);
}
```

## 2.10 Caché Interno de Reglas Compiladas

El motor mantiene un `ConcurrentDictionary` interno que mapea la combinación de workflow name + rule hash al delegate compilado. Este caché:

- Se construye lazy (en la primera evaluación de cada regla).
- Persiste durante toda la vida de la instancia de `RulesEngine`.
- Se invalida al llamar `ClearWorkflows()` o al re-agregar un workflow con `AddOrUpdateWorkflow()`.
- NO tiene expiración por tiempo ni límite de tamaño.

**Impacto en memoria:** Cada Expression Tree compilado retiene el delegate y el contexto de compilación. Con miles de workflows, cada uno con decenas de reglas, el consumo de memoria puede ser significativo (estimación: 1-5KB por regla compilada). En un sistema con 10,000 reglas, esto representa 10-50MB de memoria dedicada al caché — generalmente aceptable, pero debe monitorearse.

**Invalidación:** Al llamar `AddOrUpdateWorkflow()` con un workflow ya existente, el motor elimina las compilaciones cacheadas del workflow anterior y recompila lazy cuando se evalúe de nuevo. Este es el mecanismo para "recargar" reglas en runtime.

---

# MÓDULO 3 — Configuración Avanzada y Customización

## 3.1 Instalación y Dependencias

```bash
dotnet add package RulesEngine --version 5.0.3
```

Dependencia transitiva principal: `System.Linq.Dynamic.Core`, que proporciona el parser de expresiones. También depende de `FluentValidation` internamente para validación de estructura de reglas, `Newtonsoft.Json` para serialización y `Microsoft.Extensions.Logging.Abstractions` para integración de logging.

La dependencia en `Newtonsoft.Json` es relevante: si el proyecto ya usa `System.Text.Json`, coexistirán ambos serializadores. Las reglas se definen típicamente con propiedades `camelCase` o `PascalCase` según la configuración del serializador al cargar el JSON.

## 3.2 ReSettings — Configuración Exhaustiva

```csharp
var settings = new ReSettings
{
    CustomTypes = new Type[] {
        typeof(MathHelper),
        typeof(DateUtils),
        typeof(StringExtensions),
        typeof(decimal),
        typeof(Math),
        typeof(Convert)
    },
    CustomActions = new Dictionary<string, Func<ActionBase>>
    {
        { "OutputToLog", () => new LogAction() },
        { "SendAlert", () => new AlertAction() }
    },
    NestedRuleExecutionMode = NestedRuleExecutionMode.Performance,
    EnableExceptionAsErrorMessage = true,
    AutoRegisterInputType = true,
    IsExpressionCaseSensitive = false
};

var engine = new RulesEngine.RulesEngine(workflows, settings);
```

### CustomTypes

Registra tipos de .NET accesibles dentro de las expresiones. Sin registro, una expresión no puede invocar métodos estáticos de tipos arbitrarios. Los tipos primitivos y operadores básicos están disponibles por defecto.

```csharp
public static class RiskCalculator
{
    public static decimal CalculateWeightedScore(
        decimal creditScore,
        decimal incomeScore,
        decimal employmentScore)
    {
        return (creditScore * 0.45m) + (incomeScore * 0.35m) + (employmentScore * 0.20m);
    }

    public static string ClassifyRisk(decimal score)
    {
        return score switch
        {
            >= 800 => "LOW",
            >= 650 => "MEDIUM",
            >= 400 => "HIGH",
            _ => "VERY_HIGH"
        };
    }

    public static int DaysSince(DateTime date)
    {
        return (DateTime.UtcNow - date).Days;
    }
}

// Registro
var settings = new ReSettings
{
    CustomTypes = new[] { typeof(RiskCalculator), typeof(Math), typeof(decimal) }
};
```

Ahora en expresiones JSON:

```json
{
  "LocalParams": [
    {
      "Name": "weightedScore",
      "Expression": "RiskCalculator.CalculateWeightedScore(input1.CreditScore, input1.IncomeScore, input1.EmploymentScore)"
    },
    {
      "Name": "riskLevel",
      "Expression": "RiskCalculator.ClassifyRisk(weightedScore)"
    }
  ],
  "Expression": "riskLevel != \"VERY_HIGH\""
}
```

### NestedRuleExecutionMode

Dos modos disponibles:

- `All` (default): Evalúa TODAS las reglas hijas, incluso si una ya determinó el resultado. Útil para reporting completo — se sabe exactamente qué reglas pasaron y cuáles no.
- `Performance`: Implementa short-circuit. Para AND, detiene en la primera regla falsa. Para OR, detiene en la primera regla verdadera. Mejor performance pero información incompleta de resultados.

**Recomendación:** En desarrollo y staging, usar `All` para diagnóstico completo. En producción con alto volumen, evaluar si `Performance` es necesario basándose en métricas reales. El ahorro solo es significativo cuando hay muchas reglas anidadas y las primeras tienden a fallar.

### EnableExceptionAsErrorMessage

Cuando es `true`, las excepciones en expresiones se capturan y se colocan como `ErrorMessage` en el resultado en lugar de propagarse. Cuando es `false`, dependiendo de la versión, la excepción puede propagarse y abortar la evaluación.

**Recomendación:** Siempre `true` en producción. No se quiere que un null reference en una regla interrumpa la evaluación de las 49 reglas restantes.

## 3.3 Custom Functions — Registro de Funciones Invocables

Las funciones se exponen al motor mediante clases estáticas registradas en `CustomTypes`. Las restricciones son:

- Deben ser métodos `public static`.
- Los parámetros y el retorno deben ser tipos que el parser de expresiones pueda resolver.
- No se soporta sobrecarga directa en expresiones — si hay dos métodos con el mismo nombre pero diferente firma, el parser puede confundirse. La práctica segura es usar nombres únicos.
- Los métodos `async` NO son soportados en expresiones — la evaluación es síncrona dentro de la expresión.

```csharp
public static class FinancialFunctions
{
    public static decimal CalculateDTI(decimal totalDebt, decimal grossIncome)
    {
        if (grossIncome == 0) return 999m;
        return Math.Round((totalDebt / grossIncome) * 100m, 2);
    }

    public static bool IsInRange(decimal value, decimal min, decimal max)
    {
        return value >= min && value <= max;
    }

    public static int MonthsBetween(DateTime start, DateTime end)
    {
        return ((end.Year - start.Year) * 12) + end.Month - start.Month;
    }

    public static decimal ApplySpread(decimal baseRate, decimal spread, decimal cap)
    {
        var result = baseRate + spread;
        return result > cap ? cap : result;
    }

    public static bool ContainsAny(string value, string commaSeparatedList)
    {
        if (string.IsNullOrEmpty(value)) return false;
        var items = commaSeparatedList.Split(',');
        return items.Any(item => value.Contains(item.Trim(), StringComparison.OrdinalIgnoreCase));
    }
}
```

```json
{
  "Expression": "FinancialFunctions.CalculateDTI(input1.TotalDebt, input1.GrossIncome) <= 43"
}
```

## 3.4 Integración con ASP.NET Core

```csharp
// Program.cs o Startup.cs
public static class RulesEngineServiceExtensions
{
    public static IServiceCollection AddRulesEngine(
        this IServiceCollection services,
        IConfiguration configuration)
    {
        services.AddSingleton<ReSettings>(sp => new ReSettings
        {
            CustomTypes = new[]
            {
                typeof(FinancialFunctions),
                typeof(RiskCalculator),
                typeof(Math),
                typeof(decimal),
                typeof(Convert)
            },
            EnableExceptionAsErrorMessage = true,
            NestedRuleExecutionMode = NestedRuleExecutionMode.All
        });

        services.AddSingleton<IRuleStorageProvider, DatabaseRuleStorageProvider>();
        services.AddSingleton<IRulesEngineService, RulesEngineService>();

        return services;
    }
}
```

### Ciclo de Vida: Singleton

El motor debe registrarse como **Singleton**. Razones:

1. **Caché de compilación:** El caché de Expression Trees compilados vive en la instancia. Crear una instancia nueva por request re-compila todas las reglas en cada request — destruye la performance.
2. **Thread safety en evaluación:** `ExecuteAllRulesAsync` es thread-safe para evaluaciones concurrentes (no muta estado durante la evaluación). Múltiples threads pueden evaluar simultáneamente contra la misma instancia.
3. **Carga de reglas:** Las reglas se cargan una vez al inicio y se actualizan mediante `AddOrUpdateWorkflow()` cuando cambian.

**Scoped es incorrecto:** Crearía una instancia por request HTTP, recompilando reglas en cada request.
**Transient es incorrecto:** Mismo problema, amplificado.

**El caveat del Singleton:** La operación `AddOrUpdateWorkflow()` NO es atómica. Si un thread está evaluando mientras otro actualiza el workflow, podría haber inconsistencia transitoria. La estrategia segura es usar un `ReaderWriterLockSlim` o un patrón de swap atómico (crear nueva instancia, intercambiar referencia).

## 3.5 Carga desde Base de Datos

```csharp
public interface IRuleStorageProvider
{
    Task<IEnumerable<Workflow>> GetWorkflowsAsync();
    Task<Workflow> GetWorkflowAsync(string workflowName, int? version = null);
    Task SaveWorkflowAsync(Workflow workflow, int version, string modifiedBy);
}

public class DatabaseRuleStorageProvider : IRuleStorageProvider
{
    private readonly IDbConnectionFactory _connectionFactory;
    private readonly ILogger<DatabaseRuleStorageProvider> _logger;

    public DatabaseRuleStorageProvider(
        IDbConnectionFactory connectionFactory,
        ILogger<DatabaseRuleStorageProvider> logger)
    {
        _connectionFactory = connectionFactory;
        _logger = logger;
    }

    public async Task<Workflow> GetWorkflowAsync(string workflowName, int? version = null)
    {
        using var connection = _connectionFactory.CreateConnection();

        var sql = version.HasValue
            ? @"SELECT WorkflowJson FROM RuleWorkflows
                WHERE WorkflowName = @Name AND Version = @Version AND IsActive = 1"
            : @"SELECT WorkflowJson FROM RuleWorkflows
                WHERE WorkflowName = @Name AND IsActive = 1
                ORDER BY Version DESC LIMIT 1";

        var json = await connection.QuerySingleOrDefaultAsync<string>(
            sql,
            new { Name = workflowName, Version = version });

        if (json is null)
        {
            _logger.LogWarning("Workflow {WorkflowName} v{Version} not found", workflowName, version);
            return null;
        }

        return JsonConvert.DeserializeObject<Workflow>(json);
    }

    public async Task SaveWorkflowAsync(Workflow workflow, int version, string modifiedBy)
    {
        using var connection = _connectionFactory.CreateConnection();

        var json = JsonConvert.SerializeObject(workflow, Formatting.None);

        await connection.ExecuteAsync(
            @"INSERT INTO RuleWorkflows (WorkflowName, Version, WorkflowJson, IsActive, CreatedBy, CreatedAt)
              VALUES (@Name, @Version, @Json, 1, @By, @At)",
            new
            {
                Name = workflow.WorkflowName,
                Version = version,
                Json = json,
                By = modifiedBy,
                At = DateTime.UtcNow
            });

        _logger.LogInformation(
            "Workflow {WorkflowName} v{Version} saved by {User}",
            workflow.WorkflowName, version, modifiedBy);
    }
}
```

**Esquema de tabla recomendado:**

```sql
CREATE TABLE RuleWorkflows (
    Id              INT IDENTITY PRIMARY KEY,
    WorkflowName    NVARCHAR(200) NOT NULL,
    Version         INT NOT NULL,
    WorkflowJson    NVARCHAR(MAX) NOT NULL,
    IsActive        BIT NOT NULL DEFAULT 1,
    CreatedBy       NVARCHAR(100) NOT NULL,
    CreatedAt       DATETIME2 NOT NULL,
    DeactivatedAt   DATETIME2 NULL,
    DeactivatedBy   NVARCHAR(100) NULL,
    Checksum        NVARCHAR(64) NULL,

    CONSTRAINT UQ_Workflow_Version UNIQUE (WorkflowName, Version),
    INDEX IX_Active_Workflow (WorkflowName, IsActive, Version DESC)
);
```

## 3.6 Recarga Dinámica en Runtime

```csharp
public class RulesEngineService : IRulesEngineService, IDisposable
{
    private RulesEngine.RulesEngine _engine;
    private readonly IRuleStorageProvider _storage;
    private readonly ReSettings _settings;
    private readonly ILogger<RulesEngineService> _logger;
    private readonly ReaderWriterLockSlim _lock = new();
    private Timer _reloadTimer;

    public RulesEngineService(
        IRuleStorageProvider storage,
        ReSettings settings,
        ILogger<RulesEngineService> logger)
    {
        _storage = storage;
        _settings = settings;
        _logger = logger;

        InitializeAsync().GetAwaiter().GetResult();

        _reloadTimer = new Timer(
            async _ => await ReloadRulesAsync(),
            null,
            TimeSpan.FromMinutes(5),
            TimeSpan.FromMinutes(5));
    }

    private async Task InitializeAsync()
    {
        var workflows = (await _storage.GetWorkflowsAsync()).ToArray();
        _engine = new RulesEngine.RulesEngine(workflows, _settings);
        _logger.LogInformation("RulesEngine initialized with {Count} workflows", workflows.Length);
    }

    public async Task ReloadRulesAsync()
    {
        try
        {
            var workflows = (await _storage.GetWorkflowsAsync()).ToArray();

            _lock.EnterWriteLock();
            try
            {
                _engine = new RulesEngine.RulesEngine(workflows, _settings);
            }
            finally
            {
                _lock.ExitWriteLock();
            }

            _logger.LogInformation("Rules reloaded: {Count} workflows", workflows.Length);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to reload rules, keeping previous version");
        }
    }

    public async Task<List<RuleResultTree>> EvaluateAsync(
        string workflowName,
        params RuleParameter[] parameters)
    {
        _lock.EnterReadLock();
        try
        {
            return await _engine.ExecuteAllRulesAsync(workflowName, parameters);
        }
        finally
        {
            _lock.ExitReadLock();
        }
    }

    public void Dispose()
    {
        _reloadTimer?.Dispose();
        _lock?.Dispose();
    }
}
```

La estrategia de swap atómico (crear nueva instancia y reemplazar la referencia) es preferible al `ReaderWriterLockSlim` en escenarios de altísima concurrencia, ya que evita contención de locks durante la evaluación:

```csharp
public async Task ReloadRulesAsync()
{
    var workflows = (await _storage.GetWorkflowsAsync()).ToArray();
    var newEngine = new RulesEngine.RulesEngine(workflows, _settings);

    // Swap atómico — Interlocked garantiza visibilidad entre threads
    var oldEngine = Interlocked.Exchange(ref _engine, newEngine);

    // El old engine sigue siendo válido para evaluaciones en curso
    // No hay Dispose necesario — el GC lo recolecta cuando no hay más referencias
}
```

## 3.7 Logging y Observabilidad

```csharp
public class ObservableRulesEngineService : IRulesEngineService
{
    private readonly RulesEngine.RulesEngine _engine;
    private readonly ILogger<ObservableRulesEngineService> _logger;
    private readonly ActivitySource _activitySource = new("RulesEngine.Evaluation");

    public async Task<RuleEvaluationResult> EvaluateWithTelemetry(
        string workflowName,
        string correlationId,
        params RuleParameter[] parameters)
    {
        using var activity = _activitySource.StartActivity("EvaluateRules");
        activity?.SetTag("workflow.name", workflowName);
        activity?.SetTag("correlation.id", correlationId);

        var stopwatch = Stopwatch.StartNew();

        try
        {
            var results = await _engine.ExecuteAllRulesAsync(workflowName, parameters);

            stopwatch.Stop();

            var passed = results.Count(r => r.IsSuccess);
            var failed = results.Count(r => !r.IsSuccess);
            var errors = results.Count(r => !string.IsNullOrEmpty(r.ExceptionMessage));

            activity?.SetTag("rules.total", results.Count);
            activity?.SetTag("rules.passed", passed);
            activity?.SetTag("rules.failed", failed);
            activity?.SetTag("rules.errors", errors);
            activity?.SetTag("evaluation.duration_ms", stopwatch.ElapsedMilliseconds);

            _logger.LogInformation(
                "Workflow {Workflow} evaluated in {Duration}ms: {Passed} passed, {Failed} failed, {Errors} errors. CorrelationId: {CorrelationId}",
                workflowName,
                stopwatch.ElapsedMilliseconds,
                passed, failed, errors,
                correlationId);

            if (errors > 0)
            {
                foreach (var errorResult in results.Where(r => !string.IsNullOrEmpty(r.ExceptionMessage)))
                {
                    _logger.LogWarning(
                        "Rule {RuleName} threw exception: {Exception}. CorrelationId: {CorrelationId}",
                        errorResult.Rule.RuleName,
                        errorResult.ExceptionMessage,
                        correlationId);
                }
            }

            return new RuleEvaluationResult
            {
                WorkflowName = workflowName,
                Results = results,
                DurationMs = stopwatch.ElapsedMilliseconds,
                CorrelationId = correlationId,
                EvaluatedAt = DateTime.UtcNow
            };
        }
        catch (Exception ex)
        {
            activity?.SetStatus(ActivityStatusCode.Error, ex.Message);
            _logger.LogError(ex, "Rule evaluation failed for workflow {Workflow}", workflowName);
            throw;
        }
    }
}
```

---

# MÓDULO 4 — Diseño de Reglas: Casos Básicos a Intermedios

## 4.1 Anatomía de un Workflow JSON

```json
{
  "WorkflowName": "ProductEligibility_v2",
  "GlobalParams": [
    {
      "Name": "currentDate",
      "Expression": "DateTime.UtcNow"
    },
    {
      "Name": "minAge",
      "Expression": "18"
    }
  ],
  "Rules": [
    {
      "RuleName": "Rule_001_AgeValidation",
      "Enabled": true,
      "ErrorType": "Error",
      "ErrorMessage": "Applicant does not meet age requirement",
      "SuccessEvent": "AGE_OK",
      "RuleExpressionType": "LambdaExpression",
      "Expression": "input1.Age >= minAge AND input1.Age <= 65"
    }
  ]
}
```

- `WorkflowName`: Identificador único. Convención recomendada: `{Domain}_{Process}_v{Version}`.
- `GlobalParams`: Se evalúan una vez antes de cualquier regla. Disponibles en todas las expresiones del workflow.
- `Rules`: Array de reglas evaluadas en orden de declaración. El orden no afecta la lógica (cada regla es independiente), pero afecta el orden de los resultados.

## 4.2 Múltiples Inputs

```csharp
var applicant = new ApplicantData
{
    Name = "Ana Torres",
    Age = 28,
    AnnualIncome = 95000m,
    CreditScore = 740
};

var product = new ProductConfig
{
    MinAge = 21,
    MaxAge = 60,
    MinIncome = 40000m,
    MinCreditScore = 680,
    MaxDTI = 43m
};

var marketRates = new MarketConditions
{
    BaseRate = 5.25m,
    Spread = 1.50m,
    Cap = 18.0m
};

var results = await engine.ExecuteAllRulesAsync("LoanEvaluation",
    new RuleParameter("applicant", applicant),
    new RuleParameter("product", product),
    new RuleParameter("market", marketRates)
);
```

```json
{
  "RuleName": "DynamicIncomeCheck",
  "RuleExpressionType": "LambdaExpression",
  "Expression": "applicant.AnnualIncome >= product.MinIncome"
},
{
  "RuleName": "RateCalculation",
  "RuleExpressionType": "LambdaExpression",
  "LocalParams": [
    {
      "Name": "finalRate",
      "Expression": "FinancialFunctions.ApplySpread(market.BaseRate, market.Spread, market.Cap)"
    }
  ],
  "Expression": "finalRate <= 15.0",
  "Actions": {
    "OnSuccess": {
      "Name": "OutputExpression",
      "Context": {
        "Expression": "finalRate"
      }
    }
  }
}
```

Usar múltiples inputs evita construir un mega-objeto con toda la información. Cada input representa un bounded context de datos: los datos del solicitante, la configuración del producto, las condiciones de mercado. Las expresiones acceden a cada input por su nombre.

## 4.3 LocalParams (ScopedParams)

Los LocalParams son variables calculadas que se resuelven antes de la expresión de la regla. Se evalúan secuencialmente — un LocalParam puede referenciar a los anteriores.

```json
{
  "RuleName": "ComprehensiveDTICheck",
  "RuleExpressionType": "LambdaExpression",
  "LocalParams": [
    {
      "Name": "totalMonthlyDebt",
      "Expression": "applicant.MortgagePayment + applicant.CarPayment + applicant.CreditCardMinPayment + applicant.StudentLoanPayment"
    },
    {
      "Name": "monthlyIncome",
      "Expression": "applicant.AnnualIncome / 12"
    },
    {
      "Name": "dtiRatio",
      "Expression": "decimal.Round(totalMonthlyDebt / monthlyIncome * 100, 2)"
    },
    {
      "Name": "dtiCategory",
      "Expression": "dtiRatio <= 36 ? \"LOW\" : (dtiRatio <= 43 ? \"MODERATE\" : \"HIGH\")"
    }
  ],
  "Expression": "dtiRatio <= product.MaxDTI",
  "Actions": {
    "OnSuccess": {
      "Name": "OutputExpression",
      "Context": {
        "Expression": "new (dtiRatio as DTIRatio, dtiCategory as DTICategory)"
      }
    }
  }
}
```

**Puntos clave:**

- `totalMonthlyDebt` se calcula primero.
- `monthlyIncome` se calcula segundo (no depende del anterior, pero el orden se respeta).
- `dtiRatio` usa ambos anteriores.
- `dtiCategory` usa `dtiRatio`.
- La expresión final solo evalúa el resultado boolean, pero usa `dtiRatio` que ya fue calculado.
- La acción `OutputExpression` devuelve los valores intermedios como output de la regla.

## 4.4 Reglas con Actions

Actions permiten ejecutar lógica como resultado de la evaluación. El tipo más usado es `OutputExpression`, que permite retornar valores calculados.

```json
{
  "RuleName": "DiscountCalculation",
  "RuleExpressionType": "LambdaExpression",
  "Expression": "input1.CustomerType == \"Premium\" AND input1.PurchaseAmount > 1000",
  "Actions": {
    "OnSuccess": {
      "Name": "OutputExpression",
      "Context": {
        "Expression": "new (0.15 as DiscountRate, input1.PurchaseAmount * 0.15 as DiscountAmount, \"PREMIUM_DISCOUNT\" as DiscountCode)"
      }
    },
    "OnFailure": {
      "Name": "OutputExpression",
      "Context": {
        "Expression": "new (0.0 as DiscountRate, 0.0 as DiscountAmount, \"NO_DISCOUNT\" as DiscountCode)"
      }
    }
  }
}
```

**Extracción de outputs:**

```csharp
var results = await engine.ExecuteAllRulesAsync("DiscountWorkflow", param);

foreach (var result in results)
{
    if (result.ActionResult?.Output != null)
    {
        dynamic output = result.ActionResult.Output;
        decimal discountRate = output.DiscountRate;
        decimal discountAmount = output.DiscountAmount;
        string discountCode = output.DiscountCode;
    }
}
```

El `Output` es un `object` dinámico creado por la expresión `new (...)`. Se accede via `dynamic` o reflection. Este patrón es común para que las reglas no solo digan "sí/no" sino que produzcan valores calculados.

## 4.5 Caso Práctico: Motor de Descuentos

**Modelo:**

```csharp
public class PurchaseContext
{
    public string CustomerType { get; set; }     // "Standard", "Premium", "VIP"
    public decimal PurchaseAmount { get; set; }
    public int LoyaltyPoints { get; set; }
    public DateTime MemberSince { get; set; }
    public int PurchaseCountLast30Days { get; set; }
    public string PromoCode { get; set; }
    public string ProductCategory { get; set; }
}
```

**Workflow completo:**

```json
[
  {
    "WorkflowName": "DiscountEngine_v1",
    "GlobalParams": [
      {
        "Name": "today",
        "Expression": "DateTime.UtcNow"
      }
    ],
    "Rules": [
      {
        "RuleName": "VolumeDiscount",
        "RuleExpressionType": "LambdaExpression",
        "Expression": "input1.PurchaseAmount >= 5000",
        "Actions": {
          "OnSuccess": {
            "Name": "OutputExpression",
            "Context": {
              "Expression": "input1.PurchaseAmount >= 10000 ? 0.10 : 0.05"
            }
          }
        }
      },
      {
        "RuleName": "LoyaltyDiscount",
        "RuleExpressionType": "LambdaExpression",
        "LocalParams": [
          {
            "Name": "membershipYears",
            "Expression": "(today - input1.MemberSince).TotalDays / 365.25"
          }
        ],
        "Expression": "input1.LoyaltyPoints >= 1000 AND membershipYears >= 2",
        "Actions": {
          "OnSuccess": {
            "Name": "OutputExpression",
            "Context": {
              "Expression": "membershipYears >= 5 ? 0.08 : 0.04"
            }
          }
        }
      },
      {
        "RuleName": "FrequentBuyerDiscount",
        "RuleExpressionType": "LambdaExpression",
        "Expression": "input1.PurchaseCountLast30Days >= 5",
        "Actions": {
          "OnSuccess": {
            "Name": "OutputExpression",
            "Context": {
              "Expression": "0.03"
            }
          }
        }
      },
      {
        "RuleName": "VIPExclusiveDiscount",
        "RuleExpressionType": "LambdaExpression",
        "Expression": "input1.CustomerType == \"VIP\"",
        "Actions": {
          "OnSuccess": {
            "Name": "OutputExpression",
            "Context": {
              "Expression": "0.12"
            }
          }
        }
      }
    ]
  }
]
```

**Ejecución con acumulación de descuentos:**

```csharp
public class DiscountResult
{
    public decimal BaseAmount { get; set; }
    public decimal TotalDiscountRate { get; set; }
    public decimal FinalAmount { get; set; }
    public List<string> AppliedDiscounts { get; set; } = new();
}

public async Task<DiscountResult> CalculateDiscount(PurchaseContext context)
{
    var param = new RuleParameter("input1", context);
    var results = await _engine.EvaluateAsync("DiscountEngine_v1", param);

    var discountResult = new DiscountResult
    {
        BaseAmount = context.PurchaseAmount
    };

    decimal totalRate = 0m;
    decimal maxRate = 0.25m; // cap at 25%

    foreach (var result in results.Where(r => r.IsSuccess && r.ActionResult?.Output != null))
    {
        var rate = Convert.ToDecimal(result.ActionResult.Output);
        totalRate += rate;
        discountResult.AppliedDiscounts.Add($"{result.Rule.RuleName}: {rate:P0}");
    }

    discountResult.TotalDiscountRate = Math.Min(totalRate, maxRate);
    discountResult.FinalAmount = context.PurchaseAmount * (1 - discountResult.TotalDiscountRate);

    return discountResult;
}
```

## 4.6 Caso Práctico: Evaluación de Elegibilidad Financiera

```json
[
  {
    "WorkflowName": "CreditCardEligibility",
    "Rules": [
      {
        "RuleName": "IdentityVerified",
        "ErrorMessage": "Identity verification is required",
        "ErrorType": "Error",
        "RuleExpressionType": "LambdaExpression",
        "Expression": "applicant.IsIdentityVerified == true"
      },
      {
        "RuleName": "AgeAndResidency",
        "ErrorMessage": "Must be 18+ and a legal resident",
        "ErrorType": "Error",
        "RuleExpressionType": "All",
        "Rules": [
          {
            "RuleName": "MinimumAge",
            "RuleExpressionType": "LambdaExpression",
            "Expression": "applicant.Age >= 18"
          },
          {
            "RuleName": "LegalResident",
            "RuleExpressionType": "LambdaExpression",
            "Expression": "applicant.IsLegalResident == true"
          }
        ]
      },
      {
        "RuleName": "FinancialStability",
        "ErrorMessage": "Financial stability requirements not met",
        "ErrorType": "Error",
        "RuleExpressionType": "All",
        "Rules": [
          {
            "RuleName": "MinIncome",
            "RuleExpressionType": "LambdaExpression",
            "Expression": "applicant.MonthlyIncome >= product.MinMonthlyIncome"
          },
          {
            "RuleName": "EmploymentDuration",
            "RuleExpressionType": "LambdaExpression",
            "Expression": "applicant.EmploymentMonths >= product.MinEmploymentMonths"
          },
          {
            "RuleName": "NoBankruptcy",
            "RuleExpressionType": "LambdaExpression",
            "Expression": "applicant.HasActiveBankruptcy == false"
          }
        ]
      },
      {
        "RuleName": "CreditProfile",
        "ErrorMessage": "Credit profile does not meet minimum requirements",
        "ErrorType": "Error",
        "RuleExpressionType": "LambdaExpression",
        "LocalParams": [
          {
            "Name": "adjustedScore",
            "Expression": "applicant.CreditScore + (applicant.HasExistingRelationship ? 25 : 0)"
          }
        ],
        "Expression": "adjustedScore >= product.MinCreditScore",
        "Actions": {
          "OnSuccess": {
            "Name": "OutputExpression",
            "Context": {
              "Expression": "new (adjustedScore as AdjustedScore, applicant.CreditScore as RawScore)"
            }
          }
        }
      }
    ]
  }
]
```

---

# MÓDULO 5 — Diseño de Reglas: Casos Avanzados

## 5.1 Reglas Anidadas — Composición Profunda

Las reglas anidadas permiten construir árboles lógicos complejos. El patrón más poderoso combina AND y OR en múltiples niveles:

```json
{
  "RuleName": "LoanApprovalMatrix",
  "ErrorMessage": "Does not meet approval criteria",
  "RuleExpressionType": "Any",
  "Rules": [
    {
      "RuleName": "AutoApprovalPath",
      "RuleExpressionType": "All",
      "Rules": [
        {
          "RuleName": "ExcellentCredit",
          "RuleExpressionType": "LambdaExpression",
          "Expression": "applicant.CreditScore >= 750"
        },
        {
          "RuleName": "LowDTI",
          "RuleExpressionType": "LambdaExpression",
          "Expression": "applicant.DTIRatio <= 30"
        },
        {
          "RuleName": "StableEmployment",
          "RuleExpressionType": "LambdaExpression",
          "Expression": "applicant.EmploymentYears >= 3"
        }
      ]
    },
    {
      "RuleName": "CompensatingFactorsPath",
      "RuleExpressionType": "All",
      "Rules": [
        {
          "RuleName": "DecentCredit",
          "RuleExpressionType": "LambdaExpression",
          "Expression": "applicant.CreditScore >= 650"
        },
        {
          "RuleName": "HasCompensatingFactors",
          "RuleExpressionType": "Any",
          "Rules": [
            {
              "RuleName": "HighDownPayment",
              "RuleExpressionType": "LambdaExpression",
              "Expression": "applicant.DownPaymentPercent >= 30"
            },
            {
              "RuleName": "SubstantialSavings",
              "RuleExpressionType": "LambdaExpression",
              "Expression": "applicant.LiquidAssets >= applicant.RequestedAmount * 0.5"
            },
            {
              "RuleName": "ExistingClientBonus",
              "RuleExpressionType": "LambdaExpression",
              "Expression": "applicant.IsExistingClient == true AND applicant.RelationshipYears >= 5"
            }
          ]
        }
      ]
    }
  ]
}
```

Esta estructura expresa: "Aprobar si (crédito excelente Y bajo DTI Y empleo estable) O (crédito decente Y al menos un factor compensatorio)". El `RuleResultTree` refleja la jerarquía completa, permitiendo saber exactamente qué camino se satisfizo o qué sub-condición específica falló.

## 5.2 ScopedParams Avanzados — Encadenamiento de Cálculos

```json
{
  "RuleName": "ComplexRiskAssessment",
  "RuleExpressionType": "LambdaExpression",
  "LocalParams": [
    {
      "Name": "creditFactor",
      "Expression": "applicant.CreditScore >= 750 ? 1.0 : (applicant.CreditScore >= 650 ? 0.8 : 0.5)"
    },
    {
      "Name": "incomeFactor",
      "Expression": "applicant.AnnualIncome >= 120000 ? 1.0 : (applicant.AnnualIncome >= 60000 ? 0.7 : 0.4)"
    },
    {
      "Name": "employmentFactor",
      "Expression": "applicant.EmploymentYears >= 5 ? 1.0 : (applicant.EmploymentYears >= 2 ? 0.75 : 0.5)"
    },
    {
      "Name": "debtFactor",
      "Expression": "applicant.DTIRatio <= 20 ? 1.0 : (applicant.DTIRatio <= 36 ? 0.8 : (applicant.DTIRatio <= 43 ? 0.6 : 0.3))"
    },
    {
      "Name": "compositeScore",
      "Expression": "decimal.Round((decimal)(creditFactor * 0.35 + incomeFactor * 0.25 + employmentFactor * 0.20 + debtFactor * 0.20) * 1000, 0)"
    },
    {
      "Name": "riskTier",
      "Expression": "compositeScore >= 800 ? \"TIER_1\" : (compositeScore >= 650 ? \"TIER_2\" : (compositeScore >= 500 ? \"TIER_3\" : \"TIER_4\"))"
    }
  ],
  "Expression": "compositeScore >= 500",
  "Actions": {
    "OnSuccess": {
      "Name": "OutputExpression",
      "Context": {
        "Expression": "new (compositeScore as CompositeScore, riskTier as RiskTier, creditFactor as CreditFactor, incomeFactor as IncomeFactor, employmentFactor as EmploymentFactor, debtFactor as DebtFactor)"
      }
    }
  }
}
```

Este patrón transforma al motor en una calculadora configurable de scoring, donde cada factor se calcula independientemente y se combina en un score compuesto. Los pesos (0.35, 0.25, 0.20, 0.20) pueden externalizarse también como parámetros de un segundo input.

## 5.3 Evaluación sobre Colecciones

Las expresiones pueden usar métodos LINQ sobre colecciones cuando los tipos contenedores están accesibles:

```csharp
public class TransactionHistory
{
    public List<Transaction> RecentTransactions { get; set; }
    public decimal TotalExposure { get; set; }
}

public class Transaction
{
    public decimal Amount { get; set; }
    public DateTime Date { get; set; }
    public string Type { get; set; }
    public string Country { get; set; }
}
```

```json
{
  "RuleName": "HighFrequencyAlert",
  "RuleExpressionType": "LambdaExpression",
  "LocalParams": [
    {
      "Name": "last24hCount",
      "Expression": "input1.RecentTransactions.Count(t => t.Date >= DateTime.UtcNow.AddHours(-24))"
    },
    {
      "Name": "last24hTotal",
      "Expression": "input1.RecentTransactions.Where(t => t.Date >= DateTime.UtcNow.AddHours(-24)).Sum(t => t.Amount)"
    }
  ],
  "Expression": "last24hCount > 10 OR last24hTotal > 50000"
}
```

**Limitaciones:**

- El tipo `Transaction` debe estar accesible. Si las expresiones lambda dentro de LINQ usan propiedades del tipo genérico, ese tipo debe estar registrado en `CustomTypes` o ser resoluble por el parser.
- Operaciones complejas de LINQ (GroupBy, SelectMany, Join) pueden no funcionar correctamente con el parser dinámico.
- El rendimiento de LINQ dentro de expresiones no difiere del LINQ normal — evalúa in-memory. Con colecciones grandes (>10,000 items), considerar pre-filtrar antes de pasar al motor.

## 5.4 Expresiones Complejas

**Operaciones con fechas:**

```json
{
  "Expression": "(DateTime.UtcNow - input1.AccountOpenDate).TotalDays >= 365"
}
```

**Operador ternario:**

```json
{
  "Expression": "input1.Amount > 10000 ? true : input1.CreditScore > 700"
}
```

**Conversiones de tipo:**

```json
{
  "LocalParams": [
    {
      "Name": "parsedAmount",
      "Expression": "decimal.Parse(input1.AmountString)"
    }
  ],
  "Expression": "parsedAmount > 0"
}
```

**Operaciones matemáticas:**

```json
{
  "LocalParams": [
    {
      "Name": "monthlyPayment",
      "Expression": "decimal.Round(input1.Principal * (input1.Rate / 12) / (1 - Math.Pow((double)(1 + input1.Rate / 12), (double)(-input1.TermMonths))), 2)"
    }
  ],
  "Expression": "monthlyPayment <= input1.MaxMonthlyPayment"
}
```

## 5.5 Composición de Workflows

Cuando la lógica es demasiado compleja para un solo workflow, se divide en múltiples workflows ejecutados secuencialmente con paso de contexto:

```csharp
public class LoanPipelineOrchestrator
{
    private readonly IRulesEngineService _rulesEngine;

    public async Task<LoanDecision> ProcessApplication(LoanApplication app)
    {
        var context = new PipelineContext();

        // Etapa 1: Pre-validación
        var preValidation = await _rulesEngine.EvaluateAsync("PreValidation_v1",
            new RuleParameter("app", app));

        if (preValidation.Any(r => !r.IsSuccess && r.Rule.ErrorType == ErrorType.Error))
        {
            return LoanDecision.Rejected("Pre-validation failed",
                preValidation.Where(r => !r.IsSuccess).Select(r => r.Rule.ErrorMessage));
        }

        // Etapa 2: Scoring — extraer score calculado
        var scoring = await _rulesEngine.EvaluateAsync("CreditScoring_v2",
            new RuleParameter("app", app));

        var scoreResult = scoring.FirstOrDefault(r => r.Rule.RuleName == "CompositeScore");
        dynamic scoreOutput = scoreResult?.ActionResult?.Output;
        decimal compositeScore = scoreOutput?.CompositeScore ?? 0;
        string riskTier = scoreOutput?.RiskTier ?? "UNKNOWN";

        // Enriquecer contexto para siguiente etapa
        var scoredApp = new ScoredApplication
        {
            Application = app,
            CompositeScore = compositeScore,
            RiskTier = riskTier
        };

        // Etapa 3: Pricing
        var pricing = await _rulesEngine.EvaluateAsync("PricingRules_v1",
            new RuleParameter("scored", scoredApp),
            new RuleParameter("market", await GetMarketConditions()));

        var rateResult = pricing.FirstOrDefault(r => r.IsSuccess && r.ActionResult?.Output != null);
        dynamic rateOutput = rateResult?.ActionResult?.Output;

        // Etapa 4: Aprobación final
        var finalApp = new FinalApplication
        {
            Scored = scoredApp,
            OfferedRate = rateOutput?.FinalRate ?? 0m,
            OfferedTerm = rateOutput?.Term ?? 0
        };

        var approval = await _rulesEngine.EvaluateAsync("FinalApproval_v1",
            new RuleParameter("final", finalApp));

        bool approved = approval.All(r => r.IsSuccess);

        return new LoanDecision
        {
            Approved = approved,
            Score = compositeScore,
            RiskTier = riskTier,
            OfferedRate = finalApp.OfferedRate,
            RuleResults = new Dictionary<string, List<RuleResultTree>>
            {
                ["PreValidation"] = preValidation,
                ["Scoring"] = scoring,
                ["Pricing"] = pricing,
                ["Approval"] = approval
            }
        };
    }
}
```

## 5.6 Reglas Dinámicas Generadas en Runtime

No siempre las reglas vienen de JSON. Se pueden construir programáticamente:

```csharp
public Workflow BuildDynamicWorkflow(ProductConfig config)
{
    var rules = new List<Rule>();

    if (config.RequiresMinAge)
    {
        rules.Add(new Rule
        {
            RuleName = $"MinAge_{config.ProductCode}",
            RuleExpressionType = RuleExpressionType.LambdaExpression,
            Expression = $"input1.Age >= {config.MinAge}",
            ErrorMessage = $"Minimum age is {config.MinAge}",
            ErrorType = ErrorType.Error,
            Enabled = true
        });
    }

    if (config.RequiresMinIncome)
    {
        rules.Add(new Rule
        {
            RuleName = $"MinIncome_{config.ProductCode}",
            RuleExpressionType = RuleExpressionType.LambdaExpression,
            Expression = $"input1.AnnualIncome >= {config.MinIncome}",
            ErrorMessage = $"Minimum annual income is {config.MinIncome}",
            ErrorType = ErrorType.Error,
            Enabled = true
        });
    }

    foreach (var customRule in config.AdditionalRules)
    {
        rules.Add(new Rule
        {
            RuleName = customRule.Name,
            RuleExpressionType = RuleExpressionType.LambdaExpression,
            Expression = customRule.Expression,
            ErrorMessage = customRule.ErrorMessage,
            ErrorType = ErrorType.Error,
            Enabled = customRule.IsActive
        });
    }

    return new Workflow
    {
        WorkflowName = $"Product_{config.ProductCode}_v{config.Version}",
        Rules = rules
    };
}
```

---

# MÓDULO 6 — Casos Empresariales del Sector Financiero

## 6.1 Motor de Aprobación Crediticia

### Modelo de datos

```csharp
public class CreditApplication
{
    public string ApplicationId { get; set; }
    public string ApplicantId { get; set; }
    public int Age { get; set; }
    public decimal GrossMonthlyIncome { get; set; }
    public decimal NetMonthlyIncome { get; set; }
    public int CreditScore { get; set; }
    public decimal ExistingMonthlyDebt { get; set; }
    public int EmploymentMonths { get; set; }
    public string EmploymentType { get; set; }  // "SALARIED", "SELF_EMPLOYED", "CONTRACT"
    public decimal RequestedAmount { get; set; }
    public int RequestedTermMonths { get; set; }
    public bool HasCollateral { get; set; }
    public decimal CollateralValue { get; set; }
    public int NumberOfDependents { get; set; }
    public string ResidenceType { get; set; }    // "OWNED", "RENTED", "FAMILY"
    public int ResidenceMonths { get; set; }
    public bool HasExistingRelationship { get; set; }
    public int DelinquenciesLast24Months { get; set; }
    public string Country { get; set; }
}

public class CreditProductConfig
{
    public string ProductCode { get; set; }
    public decimal MinIncome { get; set; }
    public int MinCreditScore { get; set; }
    public decimal MaxDTI { get; set; }
    public decimal MaxLTV { get; set; }
    public int MinEmploymentMonths { get; set; }
    public int MinAge { get; set; }
    public int MaxAge { get; set; }
    public decimal MaxAmount { get; set; }
    public int MaxTermMonths { get; set; }
}
```

### Workflow JSON completo

```json
[
  {
    "WorkflowName": "CreditApproval_v3",
    "GlobalParams": [
      {
        "Name": "dtiRatio",
        "Expression": "app.GrossMonthlyIncome > 0 ? decimal.Round(app.ExistingMonthlyDebt / app.GrossMonthlyIncome * 100, 2) : 999"
      },
      {
        "Name": "ltvRatio",
        "Expression": "app.CollateralValue > 0 ? decimal.Round(app.RequestedAmount / app.CollateralValue * 100, 2) : 100"
      },
      {
        "Name": "paymentCapacity",
        "Expression": "app.NetMonthlyIncome - app.ExistingMonthlyDebt"
      }
    ],
    "Rules": [
      {
        "RuleName": "BasicEligibility",
        "ErrorMessage": "Basic eligibility criteria not met",
        "ErrorType": "Error",
        "RuleExpressionType": "All",
        "Rules": [
          {
            "RuleName": "AgeRange",
            "RuleExpressionType": "LambdaExpression",
            "ErrorMessage": "Age must be between product minimum and maximum",
            "Expression": "app.Age >= product.MinAge AND app.Age <= product.MaxAge"
          },
          {
            "RuleName": "MinimumIncome",
            "RuleExpressionType": "LambdaExpression",
            "ErrorMessage": "Income below product minimum",
            "Expression": "app.GrossMonthlyIncome >= product.MinIncome"
          },
          {
            "RuleName": "AmountWithinLimits",
            "RuleExpressionType": "LambdaExpression",
            "ErrorMessage": "Requested amount exceeds product maximum",
            "Expression": "app.RequestedAmount <= product.MaxAmount"
          },
          {
            "RuleName": "TermWithinLimits",
            "RuleExpressionType": "LambdaExpression",
            "ErrorMessage": "Requested term exceeds product maximum",
            "Expression": "app.RequestedTermMonths <= product.MaxTermMonths"
          }
        ]
      },
      {
        "RuleName": "CreditProfileAssessment",
        "ErrorMessage": "Credit profile insufficient",
        "ErrorType": "Error",
        "RuleExpressionType": "All",
        "Rules": [
          {
            "RuleName": "MinCreditScore",
            "RuleExpressionType": "LambdaExpression",
            "Expression": "app.CreditScore >= product.MinCreditScore"
          },
          {
            "RuleName": "DelinquencyCheck",
            "RuleExpressionType": "LambdaExpression",
            "Expression": "app.DelinquenciesLast24Months <= 2"
          },
          {
            "RuleName": "DTICheck",
            "RuleExpressionType": "LambdaExpression",
            "Expression": "dtiRatio <= product.MaxDTI"
          }
        ]
      },
      {
        "RuleName": "StabilityAssessment",
        "ErrorMessage": "Stability criteria not met",
        "ErrorType": "Warning",
        "RuleExpressionType": "All",
        "Rules": [
          {
            "RuleName": "EmploymentStability",
            "RuleExpressionType": "LambdaExpression",
            "Expression": "app.EmploymentMonths >= product.MinEmploymentMonths"
          },
          {
            "RuleName": "ResidenceStability",
            "RuleExpressionType": "LambdaExpression",
            "Expression": "app.ResidenceMonths >= 12"
          }
        ]
      },
      {
        "RuleName": "CollateralAssessment",
        "Enabled": true,
        "ErrorMessage": "LTV ratio exceeds maximum",
        "ErrorType": "Warning",
        "RuleExpressionType": "LambdaExpression",
        "Expression": "app.HasCollateral == false OR ltvRatio <= product.MaxLTV"
      },
      {
        "RuleName": "ApprovalDecision",
        "RuleExpressionType": "LambdaExpression",
        "LocalParams": [
          {
            "Name": "autoApprove",
            "Expression": "app.CreditScore >= 750 AND dtiRatio <= 30 AND app.DelinquenciesLast24Months == 0"
          },
          {
            "Name": "manualReview",
            "Expression": "app.CreditScore >= product.MinCreditScore AND dtiRatio <= product.MaxDTI AND app.DelinquenciesLast24Months <= 2"
          }
        ],
        "Expression": "autoApprove OR manualReview",
        "Actions": {
          "OnSuccess": {
            "Name": "OutputExpression",
            "Context": {
              "Expression": "new (autoApprove ? \"AUTO_APPROVED\" : \"MANUAL_REVIEW\" as Decision, dtiRatio as DTI, ltvRatio as LTV, paymentCapacity as PaymentCapacity, app.CreditScore as CreditScore)"
            }
          },
          "OnFailure": {
            "Name": "OutputExpression",
            "Context": {
              "Expression": "new (\"REJECTED\" as Decision, dtiRatio as DTI, ltvRatio as LTV, app.CreditScore as CreditScore)"
            }
          }
        }
      }
    ]
  }
]
```

### Código de ejecución

```csharp
public class CreditApprovalService
{
    private readonly IRulesEngineService _rulesEngine;
    private readonly ILogger<CreditApprovalService> _logger;

    public async Task<CreditDecision> EvaluateApplication(
        CreditApplication application,
        CreditProductConfig productConfig)
    {
        var results = await _rulesEngine.EvaluateAsync("CreditApproval_v3",
            new RuleParameter("app", application),
            new RuleParameter("product", productConfig));

        var errors = results
            .Where(r => !r.IsSuccess && r.Rule.ErrorType == ErrorType.Error)
            .Select(r => r.Rule.ErrorMessage)
            .ToList();

        var warnings = results
            .Where(r => !r.IsSuccess && r.Rule.ErrorType == ErrorType.Warning)
            .Select(r => r.Rule.ErrorMessage)
            .ToList();

        var decisionRule = results.FirstOrDefault(r => r.Rule.RuleName == "ApprovalDecision");
        dynamic decisionOutput = decisionRule?.ActionResult?.Output;

        string decision = errors.Any() ? "REJECTED" : decisionOutput?.Decision?.ToString() ?? "REJECTED";

        return new CreditDecision
        {
            ApplicationId = application.ApplicationId,
            Decision = decision,
            Errors = errors,
            Warnings = warnings,
            DTIRatio = decisionOutput?.DTI ?? 0m,
            LTVRatio = decisionOutput?.LTV ?? 0m,
            PaymentCapacity = decisionOutput?.PaymentCapacity ?? 0m,
            EvaluatedAt = DateTime.UtcNow,
            WorkflowVersion = "CreditApproval_v3"
        };
    }
}
```

## 6.2 Scoring Dinámico (Credit Scoring)

```json
[
  {
    "WorkflowName": "CreditScoring_CO_v2",
    "GlobalParams": [
      {
        "Name": "w_credit",
        "Expression": "0.30"
      },
      {
        "Name": "w_income",
        "Expression": "0.25"
      },
      {
        "Name": "w_employment",
        "Expression": "0.20"
      },
      {
        "Name": "w_debt",
        "Expression": "0.15"
      },
      {
        "Name": "w_stability",
        "Expression": "0.10"
      }
    ],
    "Rules": [
      {
        "RuleName": "CreditFactor",
        "RuleExpressionType": "LambdaExpression",
        "LocalParams": [
          {
            "Name": "creditPoints",
            "Expression": "input1.BureauScore >= 800 ? 100 : (input1.BureauScore >= 750 ? 90 : (input1.BureauScore >= 700 ? 75 : (input1.BureauScore >= 650 ? 60 : (input1.BureauScore >= 600 ? 40 : 20))))"
          },
          {
            "Name": "delinquencyPenalty",
            "Expression": "input1.DelinquenciesLast24Months * 15"
          },
          {
            "Name": "creditScore",
            "Expression": "Math.Max(0, creditPoints - delinquencyPenalty)"
          }
        ],
        "Expression": "true",
        "Actions": {
          "OnSuccess": {
            "Name": "OutputExpression",
            "Context": {
              "Expression": "new (creditScore as Score, \"CREDIT\" as Factor)"
            }
          }
        }
      },
      {
        "RuleName": "IncomeFactor",
        "RuleExpressionType": "LambdaExpression",
        "LocalParams": [
          {
            "Name": "incomeScore",
            "Expression": "input1.VerifiedMonthlyIncome >= 15000 ? 100 : (input1.VerifiedMonthlyIncome >= 8000 ? 80 : (input1.VerifiedMonthlyIncome >= 4000 ? 60 : (input1.VerifiedMonthlyIncome >= 2000 ? 40 : 20)))"
          }
        ],
        "Expression": "true",
        "Actions": {
          "OnSuccess": {
            "Name": "OutputExpression",
            "Context": {
              "Expression": "new (incomeScore as Score, \"INCOME\" as Factor)"
            }
          }
        }
      },
      {
        "RuleName": "EmploymentFactor",
        "RuleExpressionType": "LambdaExpression",
        "LocalParams": [
          {
            "Name": "empTypeScore",
            "Expression": "input1.EmploymentType == \"SALARIED\" ? 30 : (input1.EmploymentType == \"SELF_EMPLOYED\" ? 20 : 10)"
          },
          {
            "Name": "tenureScore",
            "Expression": "input1.EmploymentMonths >= 60 ? 70 : (input1.EmploymentMonths >= 36 ? 55 : (input1.EmploymentMonths >= 12 ? 40 : 20))"
          },
          {
            "Name": "employmentScore",
            "Expression": "empTypeScore + tenureScore"
          }
        ],
        "Expression": "true",
        "Actions": {
          "OnSuccess": {
            "Name": "OutputExpression",
            "Context": {
              "Expression": "new (employmentScore as Score, \"EMPLOYMENT\" as Factor)"
            }
          }
        }
      },
      {
        "RuleName": "CompositeScore",
        "RuleExpressionType": "LambdaExpression",
        "Expression": "true",
        "Actions": {
          "OnSuccess": {
            "Name": "OutputExpression",
            "Context": {
              "Expression": "new (w_credit as W_Credit, w_income as W_Income, w_employment as W_Employment, w_debt as W_Debt, w_stability as W_Stability)"
            }
          }
        }
      }
    ]
  }
]
```

**Orquestación del scoring:**

```csharp
public async Task<ScoringResult> CalculateScore(ApplicantData applicant, string country)
{
    string workflowName = $"CreditScoring_{country}_v2";
    var results = await _rulesEngine.EvaluateAsync(workflowName,
        new RuleParameter("input1", applicant));

    var factors = new Dictionary<string, int>();
    foreach (var result in results.Where(r => r.IsSuccess && r.ActionResult?.Output != null))
    {
        dynamic output = result.ActionResult.Output;
        if (output.Factor != null && output.Score != null)
        {
            factors[output.Factor.ToString()] = Convert.ToInt32(output.Score);
        }
    }

    var weightsResult = results.FirstOrDefault(r => r.Rule.RuleName == "CompositeScore");
    dynamic weights = weightsResult?.ActionResult?.Output;

    decimal compositeScore =
        (factors.GetValueOrDefault("CREDIT", 0) * Convert.ToDecimal(weights?.W_Credit ?? 0m)) +
        (factors.GetValueOrDefault("INCOME", 0) * Convert.ToDecimal(weights?.W_Income ?? 0m)) +
        (factors.GetValueOrDefault("EMPLOYMENT", 0) * Convert.ToDecimal(weights?.W_Employment ?? 0m)) +
        (factors.GetValueOrDefault("DEBT", 0) * Convert.ToDecimal(weights?.W_Debt ?? 0m)) +
        (factors.GetValueOrDefault("STABILITY", 0) * Convert.ToDecimal(weights?.W_Stability ?? 0m));

    string tier = compositeScore switch
    {
        >= 80 => "A",
        >= 65 => "B",
        >= 50 => "C",
        >= 35 => "D",
        _ => "E"
    };

    return new ScoringResult
    {
        CompositeScore = Math.Round(compositeScore, 2),
        Tier = tier,
        Factors = factors,
        Country = country,
        ModelVersion = workflowName
    };
}
```

## 6.3 Prevención de Fraude

```csharp
public class TransactionContext
{
    public string TransactionId { get; set; }
    public decimal Amount { get; set; }
    public string Currency { get; set; }
    public string Channel { get; set; }           // "ATM", "POS", "ONLINE", "MOBILE"
    public string MerchantCategory { get; set; }
    public string Country { get; set; }
    public string City { get; set; }
    public DateTime Timestamp { get; set; }
    public string DeviceId { get; set; }
    public string IPAddress { get; set; }
    public bool IsNewDevice { get; set; }
    public bool IsNewMerchant { get; set; }
    public decimal AccountBalance { get; set; }
    public decimal AvgTransactionAmount { get; set; }
    public int TransactionsLast24h { get; set; }
    public decimal AmountLast24h { get; set; }
    public int TransactionsLast1h { get; set; }
    public int DistinctCountriesLast24h { get; set; }
    public int FailedPINAttemptsLast1h { get; set; }
    public string PreviousTransactionCountry { get; set; }
    public int MinutesSinceLastTransaction { get; set; }
    public List<string> HighRiskCountries { get; set; }
}
```

```json
[
  {
    "WorkflowName": "FraudDetection_RealTime_v4",
    "Rules": [
      {
        "RuleName": "VelocityCheck",
        "SuccessEvent": "VELOCITY_ALERT",
        "RuleExpressionType": "Any",
        "Rules": [
          {
            "RuleName": "HighFrequency1h",
            "RuleExpressionType": "LambdaExpression",
            "Expression": "txn.TransactionsLast1h > 5"
          },
          {
            "RuleName": "HighVolume24h",
            "RuleExpressionType": "LambdaExpression",
            "Expression": "txn.AmountLast24h > txn.AccountBalance * 0.8"
          },
          {
            "RuleName": "RapidSuccession",
            "RuleExpressionType": "LambdaExpression",
            "Expression": "txn.MinutesSinceLastTransaction < 2 AND txn.TransactionsLast1h > 3"
          }
        ]
      },
      {
        "RuleName": "AmountAnomaly",
        "SuccessEvent": "AMOUNT_ANOMALY",
        "RuleExpressionType": "LambdaExpression",
        "LocalParams": [
          {
            "Name": "amountRatio",
            "Expression": "txn.AvgTransactionAmount > 0 ? txn.Amount / txn.AvgTransactionAmount : 999"
          }
        ],
        "Expression": "amountRatio > 5 OR txn.Amount > txn.AccountBalance * 0.5"
      },
      {
        "RuleName": "GeoAnomaly",
        "SuccessEvent": "GEO_ANOMALY",
        "RuleExpressionType": "Any",
        "Rules": [
          {
            "RuleName": "MultipleCountries",
            "RuleExpressionType": "LambdaExpression",
            "Expression": "txn.DistinctCountriesLast24h > 3"
          },
          {
            "RuleName": "ImpossibleTravel",
            "RuleExpressionType": "LambdaExpression",
            "Expression": "txn.Country != txn.PreviousTransactionCountry AND txn.MinutesSinceLastTransaction < 120"
          }
        ]
      },
      {
        "RuleName": "DeviceRisk",
        "SuccessEvent": "DEVICE_RISK",
        "RuleExpressionType": "LambdaExpression",
        "Expression": "txn.IsNewDevice == true AND txn.Amount > 1000"
      },
      {
        "RuleName": "PINBruteForce",
        "SuccessEvent": "PIN_BRUTE_FORCE",
        "RuleExpressionType": "LambdaExpression",
        "Expression": "txn.FailedPINAttemptsLast1h >= 3"
      },
      {
        "RuleName": "HighRiskCountryTransaction",
        "SuccessEvent": "HIGH_RISK_COUNTRY",
        "RuleExpressionType": "LambdaExpression",
        "Expression": "txn.HighRiskCountries != null AND txn.HighRiskCountries.Contains(txn.Country)"
      }
    ]
  }
]
```

**Motor de scoring de fraude:**

```csharp
public class FraudAssessmentResult
{
    public string TransactionId { get; set; }
    public int FraudScore { get; set; }
    public string AlertLevel { get; set; }         // "NONE", "LOW", "MEDIUM", "HIGH", "BLOCK"
    public List<string> TriggeredRules { get; set; }
    public string RecommendedAction { get; set; }   // "ALLOW", "CHALLENGE", "REVIEW", "BLOCK"
}

public class FraudDetectionService
{
    private readonly IRulesEngineService _rulesEngine;
    private static readonly Dictionary<string, int> RuleWeights = new()
    {
        ["VelocityCheck"] = 25,
        ["AmountAnomaly"] = 30,
        ["GeoAnomaly"] = 35,
        ["DeviceRisk"] = 15,
        ["PINBruteForce"] = 40,
        ["HighRiskCountryTransaction"] = 20
    };

    public async Task<FraudAssessmentResult> AssessTransaction(TransactionContext txn)
    {
        var results = await _rulesEngine.EvaluateAsync("FraudDetection_RealTime_v4",
            new RuleParameter("txn", txn));

        var triggeredRules = results
            .Where(r => r.IsSuccess)
            .Select(r => r.Rule.RuleName)
            .ToList();

        int fraudScore = triggeredRules
            .Sum(ruleName => RuleWeights.GetValueOrDefault(ruleName, 10));

        fraudScore = Math.Min(fraudScore, 100);

        string alertLevel = fraudScore switch
        {
            >= 80 => "BLOCK",
            >= 60 => "HIGH",
            >= 40 => "MEDIUM",
            >= 20 => "LOW",
            _ => "NONE"
        };

        string action = alertLevel switch
        {
            "BLOCK" => "BLOCK",
            "HIGH" => "REVIEW",
            "MEDIUM" => "CHALLENGE",
            _ => "ALLOW"
        };

        return new FraudAssessmentResult
        {
            TransactionId = txn.TransactionId,
            FraudScore = fraudScore,
            AlertLevel = alertLevel,
            TriggeredRules = triggeredRules,
            RecommendedAction = action
        };
    }
}
```

## 6.4 Validaciones AML (Anti Money Laundering)

```json
[
  {
    "WorkflowName": "AML_Screening_v2",
    "Rules": [
      {
        "RuleName": "CTR_Threshold",
        "SuccessEvent": "CTR_REQUIRED",
        "ErrorMessage": "Transaction exceeds CTR reporting threshold",
        "RuleExpressionType": "LambdaExpression",
        "Expression": "txn.Amount >= config.CTRThreshold"
      },
      {
        "RuleName": "StructuringDetection",
        "SuccessEvent": "POSSIBLE_STRUCTURING",
        "RuleExpressionType": "LambdaExpression",
        "LocalParams": [
          {
            "Name": "nearThresholdCount",
            "Expression": "txn.TransactionsNearThresholdLast7Days"
          },
          {
            "Name": "avgNearThreshold",
            "Expression": "txn.AvgAmountNearThresholdTxns"
          }
        ],
        "Expression": "nearThresholdCount >= 3 AND avgNearThreshold >= config.CTRThreshold * 0.8 AND avgNearThreshold < config.CTRThreshold"
      },
      {
        "RuleName": "PEPCheck",
        "SuccessEvent": "PEP_MATCH",
        "RuleExpressionType": "LambdaExpression",
        "Expression": "customer.IsPEP == true AND txn.Amount >= config.PEPThreshold"
      },
      {
        "RuleName": "SanctionedCountry",
        "SuccessEvent": "SANCTIONED_COUNTRY_ALERT",
        "RuleExpressionType": "LambdaExpression",
        "Expression": "config.SanctionedCountries.Contains(txn.OriginCountry) OR config.SanctionedCountries.Contains(txn.DestinationCountry)"
      },
      {
        "RuleName": "UnusualPatternDetection",
        "SuccessEvent": "UNUSUAL_PATTERN",
        "RuleExpressionType": "All",
        "Rules": [
          {
            "RuleName": "VolumeSpike",
            "RuleExpressionType": "LambdaExpression",
            "Expression": "txn.MonthlyVolume > customer.AvgMonthlyVolume * 3"
          },
          {
            "RuleName": "NewCounterparties",
            "RuleExpressionType": "LambdaExpression",
            "Expression": "txn.NewCounterpartiesThisMonth > 5"
          }
        ]
      },
      {
        "RuleName": "HighRiskClassification",
        "SuccessEvent": "HIGH_RISK_CUSTOMER",
        "RuleExpressionType": "LambdaExpression",
        "LocalParams": [
          {
            "Name": "riskFactors",
            "Expression": "(customer.IsPEP ? 3 : 0) + (customer.IsHighRiskIndustry ? 2 : 0) + (customer.IsHighRiskCountry ? 2 : 0) + (customer.AccountAgeMonths < 6 ? 1 : 0) + (customer.HasSARHistory ? 3 : 0)"
          }
        ],
        "Expression": "riskFactors >= 4",
        "Actions": {
          "OnSuccess": {
            "Name": "OutputExpression",
            "Context": {
              "Expression": "new (riskFactors as RiskScore, \"ENHANCED_DUE_DILIGENCE\" as RequiredAction)"
            }
          }
        }
      }
    ]
  }
]
```

## 6.5 Motor de Comisiones Bancarias

```json
[
  {
    "WorkflowName": "BankCommissions_v1",
    "Rules": [
      {
        "RuleName": "WireTransferCommission",
        "RuleExpressionType": "LambdaExpression",
        "Expression": "txn.Type == \"WIRE_TRANSFER\"",
        "LocalParams": [
          {
            "Name": "baseCommission",
            "Expression": "txn.IsInternational ? 25.00 : 10.00"
          },
          {
            "Name": "percentageCommission",
            "Expression": "txn.Amount * (txn.IsInternational ? 0.003 : 0.001)"
          },
          {
            "Name": "totalCommission",
            "Expression": "Math.Max(baseCommission, percentageCommission)"
          },
          {
            "Name": "loyaltyDiscount",
            "Expression": "customer.Segment == \"PREMIUM\" ? 0.5 : (customer.Segment == \"GOLD\" ? 0.75 : 1.0)"
          },
          {
            "Name": "finalCommission",
            "Expression": "decimal.Round((decimal)(totalCommission * loyaltyDiscount), 2)"
          }
        ],
        "Actions": {
          "OnSuccess": {
            "Name": "OutputExpression",
            "Context": {
              "Expression": "new (finalCommission as Commission, \"WIRE_TRANSFER\" as CommissionType, loyaltyDiscount as DiscountApplied)"
            }
          }
        }
      },
      {
        "RuleName": "ATMWithdrawalCommission",
        "RuleExpressionType": "LambdaExpression",
        "Expression": "txn.Type == \"ATM_WITHDRAWAL\"",
        "LocalParams": [
          {
            "Name": "freeWithdrawalsLeft",
            "Expression": "customer.FreeATMWithdrawalsPerMonth - customer.ATMWithdrawalsThisMonth"
          },
          {
            "Name": "commission",
            "Expression": "freeWithdrawalsLeft > 0 ? 0.00 : (txn.IsOwnNetwork ? 2.50 : 5.00)"
          }
        ],
        "Actions": {
          "OnSuccess": {
            "Name": "OutputExpression",
            "Context": {
              "Expression": "new (commission as Commission, \"ATM\" as CommissionType, freeWithdrawalsLeft as FreeLeft)"
            }
          }
        }
      }
    ]
  }
]
```

## 6.6 Orquestación Multi-Etapa

```csharp
public class LoanOrchestrationService
{
    private readonly IRulesEngineService _rulesEngine;
    private readonly ILogger<LoanOrchestrationService> _logger;

    public async Task<LoanPipelineResult> ProcessLoanApplication(
        LoanApplication app,
        string country)
    {
        var pipeline = new LoanPipelineResult
        {
            ApplicationId = app.ApplicationId,
            StartedAt = DateTime.UtcNow,
            Stages = new List<StageResult>()
        };

        // ETAPA 1: Pre-validación documental
        var stage1 = await ExecuteStage("PreValidation_v1", "PRE_VALIDATION",
            new RuleParameter("app", app));
        pipeline.Stages.Add(stage1);

        if (stage1.HasBlockingErrors)
        {
            pipeline.FinalDecision = "REJECTED_PRE_VALIDATION";
            pipeline.CompletedAt = DateTime.UtcNow;
            return pipeline;
        }

        // ETAPA 2: Scoring
        var stage2 = await ExecuteStage($"CreditScoring_{country}_v2", "SCORING",
            new RuleParameter("input1", app));
        pipeline.Stages.Add(stage2);

        decimal compositeScore = ExtractScore(stage2);

        // ETAPA 3: Pricing basado en score
        var pricingInput = new PricingInput
        {
            CompositeScore = compositeScore,
            RequestedAmount = app.RequestedAmount,
            RequestedTerm = app.RequestedTermMonths,
            Country = country
        };

        var stage3 = await ExecuteStage("PricingRules_v1", "PRICING",
            new RuleParameter("input1", pricingInput));
        pipeline.Stages.Add(stage3);

        // ETAPA 4: AML screening
        var stage4 = await ExecuteStage("AML_Screening_v2", "AML",
            new RuleParameter("txn", MapToAMLInput(app)),
            new RuleParameter("customer", MapToCustomerProfile(app)),
            new RuleParameter("config", await GetAMLConfig(country)));
        pipeline.Stages.Add(stage4);

        if (stage4.TriggeredAlerts.Any(a => a.Contains("SANCTIONED") || a.Contains("PEP")))
        {
            pipeline.FinalDecision = "HELD_FOR_COMPLIANCE";
            pipeline.CompletedAt = DateTime.UtcNow;
            return pipeline;
        }

        // ETAPA 5: Aprobación final
        var stage5 = await ExecuteStage("FinalApproval_v1", "APPROVAL",
            new RuleParameter("app", app),
            new RuleParameter("scoring", new { Score = compositeScore }),
            new RuleParameter("pricing", ExtractPricingOutput(stage3)));
        pipeline.Stages.Add(stage5);

        pipeline.FinalDecision = stage5.HasBlockingErrors ? "REJECTED" :
            compositeScore >= 80 ? "AUTO_APPROVED" : "MANUAL_REVIEW";
        pipeline.CompletedAt = DateTime.UtcNow;

        _logger.LogInformation(
            "Loan pipeline completed for {AppId}: {Decision} in {Duration}ms. Stages: {StageCount}",
            app.ApplicationId,
            pipeline.FinalDecision,
            (pipeline.CompletedAt.Value - pipeline.StartedAt).TotalMilliseconds,
            pipeline.Stages.Count);

        return pipeline;
    }

    private async Task<StageResult> ExecuteStage(
        string workflowName,
        string stageName,
        params RuleParameter[] parameters)
    {
        var sw = Stopwatch.StartNew();
        var results = await _rulesEngine.EvaluateAsync(workflowName, parameters);
        sw.Stop();

        return new StageResult
        {
            StageName = stageName,
            WorkflowName = workflowName,
            Results = results,
            DurationMs = sw.ElapsedMilliseconds,
            HasBlockingErrors = results.Any(r => !r.IsSuccess && r.Rule.ErrorType == ErrorType.Error),
            TriggeredAlerts = results.Where(r => r.IsSuccess).Select(r => r.Rule.SuccessEvent).Where(e => e != null).ToList()
        };
    }
}
```

---

# MÓDULO 7 — Edge Cases, Problemas Reales y Escenarios Complejos

## 7.1 Null Propagation

El problema más frecuente en producción. La expresión `input1.Address.City == "Lima"` lanza `NullReferenceException` si `Address` es null.

```csharp
// Datos de prueba que causan el problema
var data = new Customer { Name = "Test", Address = null };
var param = new RuleParameter("input1", data);
// La expresión input1.Address.City == "Lima" lanza NullReferenceException
```

**Soluciones:**

1. **Null check explícito en la expresión:**

```json
{
  "Expression": "input1.Address != null AND input1.Address.City == \"Lima\""
}
```

1. **Separar en reglas anidadas con short-circuit:**

```json
{
  "RuleName": "CityCheck",
  "RuleExpressionType": "All",
  "Rules": [
    {
      "RuleName": "AddressExists",
      "RuleExpressionType": "LambdaExpression",
      "Expression": "input1.Address != null"
    },
    {
      "RuleName": "CityMatch",
      "RuleExpressionType": "LambdaExpression",
      "Expression": "input1.Address.City == \"Lima\""
    }
  ]
}
```

Con `NestedRuleExecutionMode = Performance`, si `AddressExists` falla, `CityMatch` no se evalúa.

1. **Custom function null-safe:**

```csharp
public static class SafeNav
{
    public static string GetString(object obj, string propertyPath)
    {
        if (obj == null) return null;

        var parts = propertyPath.Split('.');
        object current = obj;

        foreach (var part in parts)
        {
            if (current == null) return null;
            var prop = current.GetType().GetProperty(part);
            if (prop == null) return null;
            current = prop.GetValue(current);
        }

        return current?.ToString();
    }
}
```

## 7.2 Excepciones Dentro de Expresiones

```json
{
  "LocalParams": [
    {
      "Name": "ratio",
      "Expression": "input1.TotalDebt / input1.Income"
    }
  ],
  "Expression": "ratio <= 0.43"
}
```

Si `Income` es 0, esto lanza `DivideByZeroException`. El motor la captura y reporta `IsSuccess = false` con el mensaje de excepción. La corrección:

```json
{
  "LocalParams": [
    {
      "Name": "ratio",
      "Expression": "input1.Income > 0 ? input1.TotalDebt / input1.Income : 999"
    }
  ],
  "Expression": "ratio <= 0.43"
}
```

## 7.3 Problemas de Concurrencia

`ExecuteAllRulesAsync` es thread-safe para lectura — múltiples threads pueden evaluar simultáneamente. El problema aparece cuando se combinan lectura y escritura:

```csharp
// PELIGROSO: Thread A evalúa mientras Thread B actualiza
Task.Run(async () => await engine.ExecuteAllRulesAsync("Workflow1", param));  // Thread A
Task.Run(() => engine.AddOrUpdateWorkflow(newWorkflows));                      // Thread B
```

`AddOrUpdateWorkflow` modifica estado interno. Si ocurre durante una evaluación, los resultados pueden ser inconsistentes. La solución es el patrón de swap atómico mostrado en la sección 3.6 — crear una nueva instancia del motor con las reglas actualizadas e intercambiar la referencia atómicamente con `Interlocked.Exchange`.

## 7.4 Problemas de Serialización

```json
{
  "Expression": "input1.Description.Contains(\"it's a \"test\"\")"
}
```

Comillas dobles dentro de valores string en expresiones dentro de JSON requieren doble escape: `\"` para el escape JSON, y luego el parser de expresiones interpreta las comillas. Las comillas simples dentro de strings no son problema en el parser.

**Estrategia segura:** Evitar strings literales complejos en expresiones. Si se necesita comparar contra strings con caracteres especiales, usar una custom function:

```csharp
public static class StringUtils
{
    public static bool MatchesPattern(string input, string pattern)
    {
        return Regex.IsMatch(input ?? "", pattern, RegexOptions.IgnoreCase);
    }
}
```

## 7.5 Expresiones Malformadas

```json
{
  "Expression": "input1.Age >= AND input1.Score > 700"
}
```

Esta expresión tiene un error de sintaxis (`>=` sin operando derecho). El motor lanza excepción en la primera evaluación (no en la carga). No hay validación de sintaxis en `AddOrUpdateWorkflow`.

**Validación previa:**

```csharp
public class RuleValidator
{
    public ValidationResult ValidateWorkflow(Workflow workflow)
    {
        var errors = new List<string>();
        var tempEngine = new RulesEngine.RulesEngine(new[] { workflow }, new ReSettings
        {
            EnableExceptionAsErrorMessage = true
        });

        var dummyInput = CreateDummyInput(workflow);
        var param = new RuleParameter("input1", dummyInput);

        try
        {
            var results = tempEngine.ExecuteAllRulesAsync(workflow.WorkflowName, param)
                .GetAwaiter().GetResult();

            foreach (var result in results)
            {
                if (!string.IsNullOrEmpty(result.ExceptionMessage) &&
                    result.ExceptionMessage.Contains("parse", StringComparison.OrdinalIgnoreCase))
                {
                    errors.Add($"Rule '{result.Rule.RuleName}': Expression parse error - {result.ExceptionMessage}");
                }
            }
        }
        catch (Exception ex)
        {
            errors.Add($"Workflow evaluation failed: {ex.Message}");
        }

        return new ValidationResult { IsValid = !errors.Any(), Errors = errors };
    }
}
```

## 7.6 Reglas Contradictorias

```json
[
  {
    "RuleName": "ApproveHighIncome",
    "Expression": "input1.Income > 100000",
    "SuccessEvent": "APPROVED"
  },
  {
    "RuleName": "RejectLowScore",
    "Expression": "input1.CreditScore < 500",
    "SuccessEvent": "REJECTED"
  }
]
```

Si el input tiene `Income = 150000` y `CreditScore = 400`, ambas reglas se cumplen: `APPROVED` y `REJECTED`. El motor no detecta ni resuelve contradicciones — simplemente evalúa todas las reglas y devuelve todos los resultados. La resolución de conflictos es responsabilidad de la aplicación consumidora.

**Patrones de resolución:**

1. **Prioridad explícita:** Usar el campo `Properties` para asignar prioridad y resolver en código.
2. **Reglas mutuamente excluyentes:** Diseñar las condiciones para que nunca se superpongan.
3. **Orquestación secuencial:** Evaluar en orden y detenerse en la primera que aplique.

## 7.7 Colecciones Vacías

```json
{
  "Expression": "input1.Transactions.Any(t => t.Amount > 10000)"
}
```

Si `Transactions` es null → `NullReferenceException`.
Si `Transactions` es una lista vacía → `Any()` retorna `false` normalmente.
Si `Transactions` es una lista vacía → `All()` retorna `true` (vacuously true por semántica LINQ).

La protección para null:

```json
{
  "Expression": "input1.Transactions != null AND input1.Transactions.Any(t => t.Amount > 10000)"
}
```

## 7.8 Casos Donde NO Conviene Usar RulesEngine

1. **Lógica que requiere inferencia:** "Si el cliente es de alto riesgo y la transacción es inusual, entonces generar alerta. Si se genera alerta y el cliente es PEP, escalar a compliance." Esta cadena de inferencia requiere forward chaining — NRules es mejor opción.
2. **Reglas con estado acumulativo:** "Bloquear si el cliente ha tenido más de 3 alertas esta semana." El motor no mantiene estado entre evaluaciones. El estado debe pre-calcularse antes de invocar al motor.
3. **Procesamiento en streaming:** El motor evalúa batch de reglas contra un snapshot de datos. No está diseñado para evaluar continuamente contra un stream de eventos.
4. **Lógica que necesita I/O:** "Si el saldo en la base de datos es mayor a X..." Las expresiones no deben hacer llamadas a base de datos ni APIs. Los datos deben estar pre-cargados en los inputs.
5. **Menos de 10 reglas estables:** El overhead de infraestructura (almacenamiento, versionado, gestión) no se justifica. Un `if/else` bien testeado es más simple y mantenible.

---

# MÓDULO 8 — Performance, Escalabilidad y Optimización

## 8.1 Costo de Compilación vs Evaluación

El perfil de performance del motor tiene dos fases distintas:

**Primera evaluación (cold start):**


| Operación                         | Tiempo estimado     |
| --------------------------------- | ------------------- |
| Parsing de expresión              | 0.5 - 3ms por regla |
| Compilación de Expression Tree    | 1 - 8ms por regla   |
| Evaluación                        | < 0.1ms por regla   |
| **Total primera vez (10 reglas)** | **15 - 110ms**      |


**Evaluaciones subsecuentes (warm):**


| Operación                      | Tiempo estimado    |
| ------------------------------ | ------------------ |
| Cache lookup                   | < 0.01ms           |
| Evaluación del delegate        | < 0.1ms por regla  |
| Construcción de RuleResultTree | < 0.05ms por regla |
| **Total warm (10 reglas)**     | **< 1.5ms**        |


La primera evaluación es 10-100x más lenta. En producción, la estrategia es "calentar" el motor al inicio evaluando cada workflow con datos dummy.

```csharp
public class RulesEngineWarmup : IHostedService
{
    private readonly IRulesEngineService _engine;
    private readonly ILogger<RulesEngineWarmup> _logger;

    public async Task StartAsync(CancellationToken cancellationToken)
    {
        var sw = Stopwatch.StartNew();
        var workflowNames = await GetAllWorkflowNames();

        foreach (var workflow in workflowNames)
        {
            try
            {
                var dummyParam = new RuleParameter("input1", new object());
                await _engine.EvaluateAsync(workflow, dummyParam);
            }
            catch
            {
                // Ignorar errores — el objetivo es compilar, no evaluar correctamente
            }
        }

        _logger.LogInformation(
            "RulesEngine warmup completed: {Count} workflows in {Duration}ms",
            workflowNames.Count, sw.ElapsedMilliseconds);
    }

    public Task StopAsync(CancellationToken cancellationToken) => Task.CompletedTask;
}
```

## 8.2 Comparación de Rendimiento

### Microsoft.RulesEngine vs Código Imperativo

```csharp
// Imperativo: ~0.001ms por evaluación
public bool EvaluateImperative(LoanApplication app)
{
    return app.Age >= 18 && app.Age <= 65
        && app.AnnualIncome >= 24000
        && app.CreditScore >= 600
        && (app.ExistingDebt / app.AnnualIncome) <= 0.4m;
}

// RulesEngine: ~0.05-0.15ms por evaluación (warm)
// Overhead: 50-150x más lento que imperativo
```

El overhead de 50-150x es el precio de la flexibilidad. En términos absolutos, 0.1ms por evaluación permite ~10,000 evaluaciones/segundo por thread. Para la mayoría de aplicaciones empresariales, esto es más que suficiente.

**Cuándo el overhead importa:**

- Evaluaciones en hot loops (>100,000/segundo) — usar código imperativo.
- Latencia ultra-baja (<1ms end-to-end) — el motor puede ser un problema con workflows complejos.
- Batch processing de millones de registros — considerar paralelización o pre-filtrado.

### Microsoft.RulesEngine vs NRules

NRules implementa el algoritmo Rete, optimizado para re-evaluación incremental cuando los hechos cambian. Para evaluación single-pass de reglas independientes, Microsoft.RulesEngine es más rápido porque no tiene el overhead de construcción de la red Rete. Para escenarios de forward chaining con muchos hechos interrelacionados, NRules es más eficiente en la re-evaluación.

## 8.3 Impacto en Memoria

Cada instancia del motor retiene:

- Diccionario de workflows: ~1-5KB por workflow (JSON deserializado).
- Caché de delegates compilados: ~1-5KB por regla.
- Metadata de parámetros y tipos: ~0.5KB por tipo registrado.

**Estimación para un sistema con 100 workflows, 20 reglas promedio por workflow:**

- Workflows: 100 × 3KB = 300KB
- Delegates compilados: 2,000 × 3KB = 6MB
- Overhead del motor: ~1MB
- **Total estimado: ~7.3MB**

Esto es aceptable para la mayoría de servicios. El problema aparece con decenas de miles de workflows (multi-tenant con miles de tenants, cada uno con sus reglas). En ese caso, considerar particionamiento: una instancia del motor por tenant o por grupo de tenants.

## 8.4 Estrategias de Optimización

**Pre-cálculo con ScopedParams:**

En lugar de repetir sub-expresiones complejas en múltiples reglas, calcularlas una vez como GlobalParam:

```json
{
  "GlobalParams": [
    { "Name": "dti", "Expression": "app.Debt / app.Income" },
    { "Name": "lti", "Expression": "app.RequestedAmount / app.AnnualIncome" }
  ],
  "Rules": [
    { "Expression": "dti <= 0.43" },
    { "Expression": "dti <= 0.36 OR app.HasCollateral" },
    { "Expression": "lti <= 5" }
  ]
}
```

**Evaluación selectiva de workflows:**

En lugar de evaluar un mega-workflow con 200 reglas, dividir en sub-workflows y evaluar solo los necesarios:

```csharp
public async Task<EvaluationResult> SmartEvaluate(ApplicationContext ctx)
{
    var results = new List<RuleResultTree>();

    // Siempre evaluar reglas básicas
    results.AddRange(await _engine.EvaluateAsync("BasicRules", ctx.ToParam()));

    // Solo evaluar reglas de alto riesgo si aplica
    if (ctx.Amount > 100000 || ctx.IsInternational)
    {
        results.AddRange(await _engine.EvaluateAsync("HighRiskRules", ctx.ToParam()));
    }

    // Solo evaluar reglas regulatorias si la jurisdicción lo requiere
    if (ctx.RequiresRegulatoryCheck)
    {
        results.AddRange(await _engine.EvaluateAsync($"Regulatory_{ctx.Country}", ctx.ToParam()));
    }

    return new EvaluationResult(results);
}
```

## 8.5 Benchmarks de Referencia


| Escenario                       | Reglas | Complejidad                 | Tiempo warm (p50) | Tiempo warm (p99) |
| ------------------------------- | ------ | --------------------------- | ----------------- | ----------------- |
| Validación simple               | 5      | Baja                        | 0.05ms            | 0.2ms             |
| Elegibilidad crediticia         | 15     | Media                       | 0.15ms            | 0.5ms             |
| Scoring completo                | 25     | Alta (ScopedParams)         | 0.3ms             | 1.2ms             |
| Fraud detection                 | 10     | Media (colecciones)         | 0.2ms             | 0.8ms             |
| AML screening                   | 20     | Alta (nested + colecciones) | 0.4ms             | 1.5ms             |
| Pipeline completo (5 workflows) | 75     | Muy alta                    | 1.2ms             | 4ms               |


Tiempos estimados en hardware moderno (Intel/AMD moderno, .NET 8). Los percentiles p99 incluyen GC pressure y variabilidad de scheduling.

---

# MÓDULO 9 — Testing Unitario e Integración

## 9.1 Estrategia de Testing

Las reglas externalizadas requieren una estrategia de testing disciplinada porque:

1. No hay validación en compile-time — errores se descubren en runtime.
2. Las reglas pueden ser modificadas por personas no técnicas.
3. Cambios sutiles en expresiones pueden alterar el comportamiento silenciosamente.

**Niveles de testing:**


| Nivel                    | Qué se testea                                            | Herramienta         |
| ------------------------ | -------------------------------------------------------- | ------------------- |
| Unitario de regla        | Cada regla individual con datos que cumplan y no cumplan | xUnit               |
| Unitario de ScopedParams | Cálculos intermedios producen valores esperados          | xUnit               |
| Integración de workflow  | Workflow completo con escenarios representativos         | xUnit               |
| Contrato de JSON         | Estructura del JSON es válida, expresiones parsean       | Custom validator    |
| Regresión                | Batería de escenarios conocidos ante cambio de reglas    | xUnit + data-driven |
| Performance              | Tiempos de evaluación bajo carga                         | BenchmarkDotNet     |


## 9.2 Tests Unitarios

```csharp
public class CreditApprovalRulesTests : IAsyncLifetime
{
    private RulesEngine.RulesEngine _engine;
    private Workflow[] _workflows;

    public async Task InitializeAsync()
    {
        var json = await File.ReadAllTextAsync("TestData/credit-approval-rules.json");
        _workflows = JsonConvert.DeserializeObject<Workflow[]>(json);
        _engine = new RulesEngine.RulesEngine(_workflows, new ReSettings
        {
            CustomTypes = new[] { typeof(FinancialFunctions), typeof(Math), typeof(decimal) },
            EnableExceptionAsErrorMessage = true
        });
    }

    public Task DisposeAsync() => Task.CompletedTask;

    [Fact]
    public async Task AgeRequirement_WithValidAge_ShouldPass()
    {
        var app = CreateBaseApplication(age: 30);
        var product = CreateBaseProductConfig();
        var results = await _engine.ExecuteAllRulesAsync("CreditApproval_v3",
            new RuleParameter("app", app),
            new RuleParameter("product", product));

        var ageRule = results.First(r => r.Rule.RuleName == "AgeRange");
        Assert.True(ageRule.IsSuccess);
        Assert.Null(ageRule.ExceptionMessage);
    }

    [Fact]
    public async Task AgeRequirement_WithUnderageApplicant_ShouldFail()
    {
        var app = CreateBaseApplication(age: 17);
        var product = CreateBaseProductConfig();
        var results = await _engine.ExecuteAllRulesAsync("CreditApproval_v3",
            new RuleParameter("app", app),
            new RuleParameter("product", product));

        var ageRule = FindRule(results, "AgeRange");
        Assert.False(ageRule.IsSuccess);
    }

    [Theory]
    [InlineData(800, 25, 0, true, "AUTO_APPROVED")]
    [InlineData(720, 35, 1, true, "MANUAL_REVIEW")]
    [InlineData(580, 50, 3, false, "REJECTED")]
    [InlineData(650, 38, 0, true, "MANUAL_REVIEW")]
    [InlineData(750, 28, 0, true, "AUTO_APPROVED")]
    public async Task ApprovalDecision_WithVariousProfiles_ShouldDecideCorrectly(
        int creditScore, decimal dtiRatio, int delinquencies,
        bool expectedSuccess, string expectedDecision)
    {
        var app = CreateBaseApplication(
            creditScore: creditScore,
            monthlyDebt: dtiRatio * 100,   // Simplified: income=10000, debt=dtiRatio*100
            grossMonthlyIncome: 10000,
            delinquencies: delinquencies);
        var product = CreateBaseProductConfig();

        var results = await _engine.ExecuteAllRulesAsync("CreditApproval_v3",
            new RuleParameter("app", app),
            new RuleParameter("product", product));

        var decision = results.First(r => r.Rule.RuleName == "ApprovalDecision");
        Assert.Equal(expectedSuccess, decision.IsSuccess);

        if (decision.ActionResult?.Output != null)
        {
            dynamic output = decision.ActionResult.Output;
            Assert.Equal(expectedDecision, output.Decision.ToString());
        }
    }

    [Fact]
    public async Task DTICalculation_WithZeroIncome_ShouldNotThrow()
    {
        var app = CreateBaseApplication(grossMonthlyIncome: 0);
        var product = CreateBaseProductConfig();

        var results = await _engine.ExecuteAllRulesAsync("CreditApproval_v3",
            new RuleParameter("app", app),
            new RuleParameter("product", product));

        // No debe haber excepciones no controladas
        Assert.All(results, r => Assert.Null(r.ExceptionMessage));
    }

    private RuleResultTree FindRule(List<RuleResultTree> results, string ruleName)
    {
        foreach (var result in results)
        {
            if (result.Rule.RuleName == ruleName) return result;
            if (result.ChildResults != null)
            {
                var found = FindRule(result.ChildResults.ToList(), ruleName);
                if (found != null) return found;
            }
        }
        return null;
    }

    private CreditApplication CreateBaseApplication(
        int age = 35,
        decimal grossMonthlyIncome = 8000,
        int creditScore = 720,
        decimal monthlyDebt = 2000,
        int employmentMonths = 48,
        int delinquencies = 0)
    {
        return new CreditApplication
        {
            ApplicationId = Guid.NewGuid().ToString(),
            Age = age,
            GrossMonthlyIncome = grossMonthlyIncome,
            NetMonthlyIncome = grossMonthlyIncome * 0.75m,
            CreditScore = creditScore,
            ExistingMonthlyDebt = monthlyDebt,
            EmploymentMonths = employmentMonths,
            EmploymentType = "SALARIED",
            RequestedAmount = 50000,
            RequestedTermMonths = 36,
            DelinquenciesLast24Months = delinquencies,
            ResidenceMonths = 24,
            ResidenceType = "RENTED",
            Country = "CO"
        };
    }

    private CreditProductConfig CreateBaseProductConfig()
    {
        return new CreditProductConfig
        {
            ProductCode = "PERSONAL_LOAN",
            MinIncome = 3000,
            MinCreditScore = 600,
            MaxDTI = 43,
            MaxLTV = 80,
            MinEmploymentMonths = 6,
            MinAge = 18,
            MaxAge = 65,
            MaxAmount = 200000,
            MaxTermMonths = 60
        };
    }
}
```

## 9.3 Tests de Contrato de JSON

```csharp
public class RuleJsonContractTests
{
    [Theory]
    [MemberData(nameof(GetAllWorkflowFiles))]
    public void WorkflowJson_ShouldDeserializeWithoutErrors(string filePath)
    {
        var json = File.ReadAllText(filePath);
        var workflows = JsonConvert.DeserializeObject<Workflow[]>(json);

        Assert.NotNull(workflows);
        Assert.NotEmpty(workflows);

        foreach (var workflow in workflows)
        {
            Assert.False(string.IsNullOrWhiteSpace(workflow.WorkflowName),
                $"Workflow in {filePath} has empty name");
            Assert.NotEmpty(workflow.Rules);

            ValidateRules(workflow.Rules, filePath);
        }
    }

    private void ValidateRules(IEnumerable<Rule> rules, string filePath)
    {
        foreach (var rule in rules)
        {
            Assert.False(string.IsNullOrWhiteSpace(rule.RuleName),
                $"Rule in {filePath} has empty name");

            if (rule.RuleExpressionType == RuleExpressionType.LambdaExpression)
            {
                Assert.False(string.IsNullOrWhiteSpace(rule.Expression),
                    $"Rule '{rule.RuleName}' in {filePath} is LambdaExpression but has no Expression");
            }

            if (rule.Rules?.Any() == true)
            {
                ValidateRules(rule.Rules, filePath);
            }
        }
    }

    public static IEnumerable<object[]> GetAllWorkflowFiles()
    {
        var rulesDirectory = Path.Combine(AppDomain.CurrentDomain.BaseDirectory, "Rules");
        return Directory.GetFiles(rulesDirectory, "*.json", SearchOption.AllDirectories)
            .Select(f => new object[] { f });
    }
}
```

## 9.4 Pipeline CI/CD

```yaml
# azure-pipelines.yml (fragmento relevante)
stages:
  - stage: ValidateRules
    jobs:
      - job: RuleContractTests
        steps:
          - task: DotNetCoreCLI@2
            displayName: 'Run Rule Contract Tests'
            inputs:
              command: 'test'
              arguments: '--filter Category=RuleContract --logger trx'

      - job: RuleUnitTests
        steps:
          - task: DotNetCoreCLI@2
            displayName: 'Run Rule Unit Tests'
            inputs:
              command: 'test'
              arguments: '--filter Category=RuleUnit --logger trx'

      - job: RuleIntegrationTests
        dependsOn: [RuleContractTests, RuleUnitTests]
        steps:
          - task: DotNetCoreCLI@2
            displayName: 'Run Rule Integration Tests'
            inputs:
              command: 'test'
              arguments: '--filter Category=RuleIntegration --logger trx'
```

---

# MÓDULO 10 — Seguridad

## 10.1 Riesgos de Inyección de Código

Microsoft.RulesEngine evalúa expresiones string como código ejecutable. Si el JSON de reglas es editable por usuarios externos (portal web, API pública), existe un riesgo real de inyección.

Una expresión como `"System.IO.File.Delete(\"C:\\\\important.dat\")"` podría ejecutarse si `System.IO.File` está accesible en el contexto de evaluación.

**Mitigación fundamental:** La superficie de ataque depende directamente de los tipos registrados en `CustomTypes`. Si solo se registran tipos seguros (clases de utilidad propias, `Math`, `Convert`, `decimal`), las expresiones no pueden acceder a `System.IO`, `System.Diagnostics.Process`, `System.Reflection` ni ningún tipo no registrado.

## 10.2 Restricción de Tipos

```csharp
// SEGURO: solo tipos explícitamente necesarios
var settings = new ReSettings
{
    CustomTypes = new[]
    {
        typeof(FinancialFunctions),    // funciones propias, auditadas
        typeof(RiskCalculator),        // funciones propias, auditadas
        typeof(Math),                  // operaciones matemáticas seguras
        typeof(decimal),               // conversiones de decimal
        typeof(Convert),               // conversiones básicas
        typeof(DateTime)               // operaciones de fecha
    }
};

// PELIGROSO: nunca registrar estos tipos
// typeof(System.IO.File)            -- acceso a filesystem
// typeof(System.IO.Directory)       -- acceso a filesystem
// typeof(System.Diagnostics.Process)-- ejecución de procesos
// typeof(System.Reflection.Assembly)-- carga dinámica de código
// typeof(System.Net.Http.HttpClient)-- acceso a red
// typeof(System.Environment)        -- variables de entorno
// typeof(Type)                      -- reflexión
```

## 10.3 Validación de Expresiones

```csharp
public class ExpressionSecurityValidator
{
    private static readonly HashSet<string> ForbiddenPatterns = new(StringComparer.OrdinalIgnoreCase)
    {
        "System.IO",
        "System.Diagnostics",
        "System.Reflection",
        "System.Net",
        "Process.Start",
        "File.Delete",
        "File.Write",
        "Assembly.Load",
        "Environment.GetEnvironmentVariable",
        "AppDomain",
        "Thread.Sleep",
        "Task.Delay",
        "GC.Collect"
    };

    public ValidationResult ValidateExpression(string expression)
    {
        var errors = new List<string>();

        foreach (var pattern in ForbiddenPatterns)
        {
            if (expression.Contains(pattern, StringComparison.OrdinalIgnoreCase))
            {
                errors.Add($"Expression contains forbidden pattern: '{pattern}'");
            }
        }

        if (expression.Length > 2000)
        {
            errors.Add("Expression exceeds maximum length of 2000 characters");
        }

        return new ValidationResult
        {
            IsValid = !errors.Any(),
            Errors = errors
        };
    }

    public ValidationResult ValidateWorkflow(Workflow workflow)
    {
        var allErrors = new List<string>();

        foreach (var rule in FlattenRules(workflow.Rules))
        {
            if (!string.IsNullOrEmpty(rule.Expression))
            {
                var result = ValidateExpression(rule.Expression);
                if (!result.IsValid)
                {
                    allErrors.AddRange(result.Errors.Select(e => $"[{rule.RuleName}] {e}"));
                }
            }

            if (rule.LocalParams != null)
            {
                foreach (var param in rule.LocalParams)
                {
                    var result = ValidateExpression(param.Expression);
                    if (!result.IsValid)
                    {
                        allErrors.AddRange(result.Errors.Select(e =>
                            $"[{rule.RuleName}.{param.Name}] {e}"));
                    }
                }
            }
        }

        return new ValidationResult { IsValid = !allErrors.Any(), Errors = allErrors };
    }

    private IEnumerable<Rule> FlattenRules(IEnumerable<Rule> rules)
    {
        foreach (var rule in rules)
        {
            yield return rule;
            if (rule.Rules != null)
            {
                foreach (var child in FlattenRules(rule.Rules))
                {
                    yield return child;
                }
            }
        }
    }
}
```

## 10.4 Auditoría de Evaluación

Para cumplimiento regulatorio (SOX, PCI-DSS), cada evaluación debe ser auditable:

```csharp
public class AuditableRulesEngineService : IRulesEngineService
{
    private readonly IRulesEngineService _inner;
    private readonly IAuditLogRepository _auditRepo;

    public async Task<List<RuleResultTree>> EvaluateAsync(
        string workflowName,
        string correlationId,
        string evaluatedBy,
        params RuleParameter[] parameters)
    {
        var inputSnapshot = SerializeInputs(parameters);
        var results = await _inner.EvaluateAsync(workflowName, parameters);
        var resultSnapshot = SerializeResults(results);

        await _auditRepo.SaveAsync(new RuleAuditEntry
        {
            Id = Guid.NewGuid(),
            CorrelationId = correlationId,
            WorkflowName = workflowName,
            EvaluatedBy = evaluatedBy,
            EvaluatedAt = DateTime.UtcNow,
            InputHash = ComputeHash(inputSnapshot),
            InputData = inputSnapshot,          // Considerar encryption at rest
            ResultData = resultSnapshot,
            RulesCount = results.Count,
            PassedCount = results.Count(r => r.IsSuccess),
            FailedCount = results.Count(r => !r.IsSuccess),
            HasErrors = results.Any(r => !string.IsNullOrEmpty(r.ExceptionMessage))
        });

        return results;
    }

    private string ComputeHash(string content)
    {
        using var sha256 = SHA256.Create();
        var bytes = sha256.ComputeHash(Encoding.UTF8.GetBytes(content));
        return Convert.ToBase64String(bytes);
    }
}
```

## 10.5 Protección contra DoS

Expresiones maliciosas pueden consumir recursos excesivos:

- **Regex catastrophic backtracking:** Si se permite regex en custom functions, un pattern como `(a+)+$` contra un input largo puede tomar minutos.
- **Colecciones extremadamente grandes:** `input1.Items.Where(x => x.SubItems.Any(y => y.Details.All(z => ...)))` sobre colecciones de millones de elementos.

**Mitigación:**

```csharp
public static class SafeRegex
{
    public static bool IsMatch(string input, string pattern)
    {
        try
        {
            return Regex.IsMatch(
                input ?? "",
                pattern,
                RegexOptions.IgnoreCase,
                TimeSpan.FromMilliseconds(100)); // Timeout de 100ms
        }
        catch (RegexMatchTimeoutException)
        {
            return false;
        }
    }
}
```

Para evaluaciones en general, implementar un timeout a nivel de evaluación:

```csharp
public async Task<List<RuleResultTree>> EvaluateWithTimeout(
    string workflowName,
    TimeSpan timeout,
    params RuleParameter[] parameters)
{
    using var cts = new CancellationTokenSource(timeout);

    try
    {
        var task = _engine.ExecuteAllRulesAsync(workflowName, parameters);
        var completed = await Task.WhenAny(task, Task.Delay(timeout));

        if (completed != task)
        {
            throw new TimeoutException(
                $"Rule evaluation for '{workflowName}' exceeded timeout of {timeout.TotalMilliseconds}ms");
        }

        return await task;
    }
    catch (OperationCanceledException)
    {
        throw new TimeoutException(
            $"Rule evaluation for '{workflowName}' was cancelled after {timeout.TotalMilliseconds}ms");
    }
}
```

---

# MÓDULO 11 — Comparaciones Técnicas

## 11.1 Microsoft.RulesEngine vs NRules


| Criterio                                    | Microsoft.RulesEngine                                  | NRules                                  |
| ------------------------------------------- | ------------------------------------------------------ | --------------------------------------- |
| **Paradigma**                               | Expression-based (evaluación directa)                  | Inference-based (Rete algorithm)        |
| **Forward chaining**                        | No                                                     | Sí (nativo)                             |
| **Backward chaining**                       | No                                                     | Parcial                                 |
| **Definición de reglas**                    | JSON externo                                           | Código C# (fluent API)                  |
| **Reglas externalizables**                  | Sí (JSON nativo)                                       | Requiere trabajo custom                 |
| **Performance (single-pass)**               | Superior — no hay overhead de red Rete                 | Inferior por construcción de red        |
| **Performance (re-evaluación incremental)** | Igual — re-evalúa todo cada vez                        | Superior — solo propaga cambios         |
| **Curva de aprendizaje**                    | Baja (1-2 días)                                        | Alta (1-2 semanas)                      |
| **Madurez**                                 | Moderada (proyecto Microsoft, mantenimiento irregular) | Moderada (proyecto comunitario estable) |
| **Resolución de conflictos**                | Manual (en código consumidor)                          | Nativa (prioridades, salience)          |
| **Tipado**                                  | Dinámico (strings evaluados en runtime)                | Estático (C# con compile-time checks)   |
| **Testing**                                 | Requiere ejecución real                                | Se puede testear reglas como clases C#  |
| **Comunidad**                               | Moderada                                               | Pequeña pero especializada              |


**Cuándo elegir cada uno:**

- **Microsoft.RulesEngine:** Reglas de validación/elegibilidad, reglas que cambian frecuentemente, equipos no técnicos definen reglas, múltiples ambientes con reglas diferentes.
- **NRules:** Lógica de inferencia compleja, reglas que disparan otras reglas, resolución de conflictos, razonamiento basado en hechos.

## 11.2 Microsoft.RulesEngine vs Código Imperativo


| Número de reglas  | Imperativo             | RulesEngine                                        | Recomendación                    |
| ----------------- | ---------------------- | -------------------------------------------------- | -------------------------------- |
| 1-10, estables    | Más simple, más rápido | Sobreingeniería                                    | Imperativo                       |
| 10-50, cambiantes | Manejable pero verbose | Aporta valor de mantenibilidad                     | RulesEngine si cambian >1x/mes   |
| 50-200            | Difícil de mantener    | Necesario                                          | RulesEngine                      |
| 200+              | Inmanejable            | Necesario, pero considerar si es el motor correcto | RulesEngine + gobernanza robusta |


**Alternativa intermedia: Specification Pattern.**

Para 10-50 reglas estables, el Specification Pattern ofrece tipado estático con composición flexible:

```csharp
public class MinAgeSpec : Specification<Applicant>
{
    private readonly int _minAge;
    public MinAgeSpec(int minAge) => _minAge = minAge;
    public override bool IsSatisfiedBy(Applicant a) => a.Age >= _minAge;
}

public class MinIncomeSpec : Specification<Applicant>
{
    private readonly decimal _minIncome;
    public MinIncomeSpec(decimal minIncome) => _minIncome = minIncome;
    public override bool IsSatisfiedBy(Applicant a) => a.AnnualIncome >= _minIncome;
}

// Composición
var eligible = new MinAgeSpec(18).And(new MinIncomeSpec(30000));
bool result = eligible.IsSatisfiedBy(applicant);
```

Ventajas sobre RulesEngine: compile-time checking, refactoring seguro, testing trivial. Desventajas: las reglas están en código compilado, no se pueden cambiar sin deploy.

## 11.3 Microsoft.RulesEngine vs FluentValidation

Son herramientas con propósitos distintos que se complementan:

- **FluentValidation:** Validación de estructura de datos (campo requerido, formato correcto, rango válido). Orientado a input validation en APIs.
- **Microsoft.RulesEngine:** Evaluación de reglas de negocio (elegibilidad, scoring, pricing). Orientado a decisiones de negocio.

Un flujo típico los usa en secuencia: FluentValidation valida que los datos son estructuralmente correctos, luego RulesEngine evalúa las reglas de negocio sobre datos ya validados.

```csharp
// Primero: validar estructura con FluentValidation
var validationResult = await _validator.ValidateAsync(request);
if (!validationResult.IsValid)
    return BadRequest(validationResult.Errors);

// Después: evaluar reglas de negocio con RulesEngine
var ruleResults = await _rulesEngine.EvaluateAsync("Eligibility", param);
```

## 11.4 Matriz de Decisión


| Criterio (1-5)               | RulesEngine | NRules | Imperativo | Specification | FluentValidation |
| ---------------------------- | ----------- | ------ | ---------- | ------------- | ---------------- |
| Flexibilidad de reglas       | 5           | 4      | 2          | 3             | 2                |
| Performance raw              | 3           | 2      | 5          | 4             | 4                |
| Curva de aprendizaje         | 4           | 2      | 5          | 4             | 5                |
| Type safety                  | 1           | 4      | 5          | 5             | 5                |
| Externalización              | 5           | 2      | 1          | 1             | 1                |
| Mantenibilidad (200+ reglas) | 4           | 4      | 1          | 3             | 2                |
| Inferencia/chaining          | 1           | 5      | 3          | 1             | 1                |
| Ecosistema/docs              | 2           | 2      | 5          | 3             | 5                |


---

# MÓDULO 12 — Integración Arquitectónica

## 12.1 Ubicación en Clean Architecture

```
┌──────────────────────────────────────────────────┐
│  Presentation Layer (API Controllers)             │
├──────────────────────────────────────────────────┤
│  Application Layer (Use Cases / Handlers)         │
│  ┌──────────────────────────────────────────┐    │
│  │ IRuleEvaluationService (Port)             │    │
│  │ CreditApprovalHandler                     │    │
│  │ FraudDetectionHandler                     │    │
│  └──────────────────────────────────────────┘    │
├──────────────────────────────────────────────────┤
│  Domain Layer (Entities, Value Objects, Specs)    │
│  ┌──────────────────────────────────────────┐    │
│  │ NO RulesEngine aquí                       │    │
│  │ Invariantes viven en el dominio puro      │    │
│  └──────────────────────────────────────────┘    │
├──────────────────────────────────────────────────┤
│  Infrastructure Layer (Implementations)           │
│  ┌──────────────────────────────────────────┐    │
│  │ RulesEngineService (Adapter)              │    │
│  │   implements IRuleEvaluationService       │    │
│  │ DatabaseRuleStorageProvider               │    │
│  │   implements IRuleStorageProvider         │    │
│  └──────────────────────────────────────────┘    │
└──────────────────────────────────────────────────┘
```

El motor vive en Infrastructure. La Application Layer solo conoce la interfaz `IRuleEvaluationService` — nunca referencia `RulesEngine` directamente. El Domain Layer no conoce la existencia del motor.

```csharp
// Application Layer — Port
public interface IRuleEvaluationService
{
    Task<RuleEvaluationResult> EvaluateAsync(
        string workflowName,
        IDictionary<string, object> inputs);
}

// Application Layer — Use Case
public class EvaluateCreditApplicationHandler
    : IRequestHandler<EvaluateCreditApplicationCommand, CreditDecision>
{
    private readonly IRuleEvaluationService _ruleService;
    private readonly ICreditApplicationRepository _repository;

    public async Task<CreditDecision> Handle(
        EvaluateCreditApplicationCommand command,
        CancellationToken cancellationToken)
    {
        var application = await _repository.GetByIdAsync(command.ApplicationId);

        var result = await _ruleService.EvaluateAsync(
            $"CreditApproval_{application.Country}_v{command.RuleVersion}",
            new Dictionary<string, object>
            {
                ["app"] = application,
                ["product"] = command.ProductConfig
            });

        return MapToDecision(result);
    }
}

// Infrastructure Layer — Adapter
public class RulesEngineAdapter : IRuleEvaluationService
{
    private readonly IRulesEngineService _engine;

    public async Task<RuleEvaluationResult> EvaluateAsync(
        string workflowName,
        IDictionary<string, object> inputs)
    {
        var parameters = inputs
            .Select(kv => new RuleParameter(kv.Key, kv.Value))
            .ToArray();

        var results = await _engine.EvaluateAsync(workflowName, parameters);

        return new RuleEvaluationResult
        {
            WorkflowName = workflowName,
            AllPassed = results.TrueForAll(r => r.IsSuccess),
            Details = results.Select(MapToDetail).ToList()
        };
    }
}
```

## 12.2 Integración con DDD

Las reglas externalizadas NO reemplazan las invariantes de dominio. La distinción:

- **Invariantes de dominio (en el Aggregate):** "Un préstamo no puede tener monto negativo." "Una cuenta no puede quedar con saldo negativo sin autorización de sobregiro." Estas SIEMPRE se validan en el dominio, no en el motor de reglas.
- **Políticas de negocio (en el motor de reglas):** "Clientes con score > 700 y DTI < 36% califican para tasa preferencial." "Transacciones > $10,000 requieren reporte CTR." Estas son políticas que cambian según el mercado, la regulación y la estrategia comercial.

```csharp
public class LoanApplicationAggregate
{
    // Invariante de dominio — NO externalizar
    public void SetAmount(decimal amount)
    {
        if (amount <= 0)
            throw new DomainException("Loan amount must be positive");
        if (amount > 10_000_000)
            throw new DomainException("Loan amount exceeds absolute maximum");

        Amount = amount;
    }

    // La decisión de aprobación basada en reglas de negocio
    // viene de la Application Layer que consulta al motor
    public void Approve(CreditDecision decision, string approvedBy)
    {
        if (Status != ApplicationStatus.UnderReview)
            throw new DomainException("Can only approve applications under review");

        Status = ApplicationStatus.Approved;
        ApprovedRate = decision.OfferedRate;
        ApprovedBy = approvedBy;
        ApprovedAt = DateTime.UtcNow;
        AddDomainEvent(new LoanApprovedEvent(this));
    }
}
```

## 12.3 Integración con CQRS y MediatR

```csharp
// Pipeline Behavior que ejecuta reglas como pre-condición
public class RuleValidationBehavior<TRequest, TResponse>
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : IRuleValidatable
{
    private readonly IRuleEvaluationService _ruleService;

    public async Task<TResponse> Handle(
        TRequest request,
        RequestHandlerDelegate<TResponse> next,
        CancellationToken cancellationToken)
    {
        if (!string.IsNullOrEmpty(request.ValidationWorkflow))
        {
            var result = await _ruleService.EvaluateAsync(
                request.ValidationWorkflow,
                request.ToRuleInputs());

            if (!result.AllPassed)
            {
                throw new BusinessRuleViolationException(
                    result.FailedRules.Select(r => r.ErrorMessage));
            }
        }

        return await next();
    }
}

// Command que implementa IRuleValidatable
public class ProcessLoanCommand : IRequest<LoanDecision>, IRuleValidatable
{
    public string ApplicationId { get; set; }
    public string Country { get; set; }

    public string ValidationWorkflow => $"LoanPreValidation_{Country}_v1";

    public IDictionary<string, object> ToRuleInputs() => new Dictionary<string, object>
    {
        ["app"] = this
    };
}
```

## 12.4 Integración con Microservicios

### Patrón: Servicio Centralizado de Reglas

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│  Loan Service │     │  Card Service │     │ Payment Svc  │
└──────┬───────┘     └──────┬───────┘     └──────┬───────┘
       │                    │                     │
       │    gRPC/REST       │    gRPC/REST        │
       ▼                    ▼                     ▼
┌─────────────────────────────────────────────────────────┐
│              Rules Service (centralizado)                 │
│  ┌─────────────────────────────────────────────────┐    │
│  │  RulesEngine + Rule Storage + Cache + Audit      │    │
│  └─────────────────────────────────────────────────┘    │
│  ┌─────────────────────────────────────────────────┐    │
│  │  API: POST /evaluate/{workflow}                   │    │
│  │  API: GET/PUT /workflows/{name}/versions/{v}      │    │
│  └─────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────┘
```

**Trade-offs de centralización:**

- **Ventaja:** Punto único de gestión, versionado y auditoría. Las reglas se actualizan en un solo lugar.
- **Desventaja:** Punto único de fallo. Latencia de red agregada. Acoplamiento temporal entre servicios.

### Patrón: Reglas Distribuidas vía Eventos

```csharp
// Servicio de gestión publica eventos cuando cambian las reglas
public class RulePublisherService
{
    private readonly IMessageBus _bus;

    public async Task PublishRuleUpdate(Workflow workflow, int version)
    {
        await _bus.PublishAsync(new RuleUpdatedEvent
        {
            WorkflowName = workflow.WorkflowName,
            Version = version,
            WorkflowJson = JsonConvert.SerializeObject(workflow),
            PublishedAt = DateTime.UtcNow
        });
    }
}

// Cada microservicio consume y actualiza su motor local
public class RuleUpdateConsumer : IConsumer<RuleUpdatedEvent>
{
    private readonly IRulesEngineService _localEngine;

    public async Task Consume(ConsumeContext<RuleUpdatedEvent> context)
    {
        var workflow = JsonConvert.DeserializeObject<Workflow>(
            context.Message.WorkflowJson);

        await _localEngine.UpdateWorkflowAsync(workflow);
    }
}
```

---

# MÓDULO 13 — Buenas Prácticas y Anti-Patrones

## 13.1 Buenas Prácticas

### Nombrado Consistente

```
✓ CreditApproval_PersonalLoan_CO_v3
✓ FraudDetection_RealTime_v4
✓ AML_Screening_International_v2

✗ rules1
✗ myWorkflow
✗ test_workflow_final_v2_FINAL
```

Convención recomendada: `{Domain}_{Process}_{Scope}_{Version}`.

### ScopedParams para Cálculos Reutilizables

En lugar de repetir cálculos en múltiples expresiones:

```json
// MAL: cálculo duplicado
{ "Expression": "input1.Debt / input1.Income <= 0.43" },
{ "Expression": "input1.Debt / input1.Income <= 0.36 OR input1.HasCollateral" }

// BIEN: GlobalParam calculado una vez
"GlobalParams": [{ "Name": "dti", "Expression": "input1.Debt / input1.Income" }],
"Rules": [
  { "Expression": "dti <= 0.43" },
  { "Expression": "dti <= 0.36 OR input1.HasCollateral" }
]
```

### Feature Flags con Enabled

```json
{
  "RuleName": "NewRegulatoryRule_2026",
  "Enabled": false,
  "Expression": "..."
}
```

Desplegar la regla deshabilitada, verificar en staging, habilitar en producción cambiando solo el campo `Enabled`. Rollback inmediato: volver a `false`.

### Inmutabilidad en Evaluación

Nunca modificar las reglas del motor mientras se están evaluando. El patrón de swap atómico (sección 3.6) garantiza que las evaluaciones en curso usan la versión anterior y las nuevas evaluaciones usan la versión actualizada.

## 13.2 Anti-Patrones

### Expresiones Monolíticas

```json
// ANTI-PATRÓN: expresión ilegible e inmantenible
{
  "Expression": "input1.Age >= 18 AND input1.Age <= 65 AND input1.Income >= 30000 AND input1.CreditScore >= 600 AND (input1.Debt / input1.Income) <= 0.43 AND input1.EmploymentYears >= 2 AND (input1.HasCollateral == true OR input1.CreditScore >= 750) AND input1.Delinquencies <= 2 AND input1.Country != \"XX\""
}
```

```json
// CORRECTO: descomponer en reglas anidadas con nombres descriptivos
{
  "RuleName": "FullEligibility",
  "RuleExpressionType": "All",
  "Rules": [
    { "RuleName": "AgeCheck", "Expression": "input1.Age >= 18 AND input1.Age <= 65" },
    { "RuleName": "IncomeCheck", "Expression": "input1.Income >= 30000" },
    { "RuleName": "CreditCheck", "Expression": "input1.CreditScore >= 600" }
  ]
}
```

### Lógica Transaccional en Reglas

```csharp
// ANTI-PATRÓN: custom function con side effects
public static class DangerousFunctions
{
    public static bool DebitAccount(string accountId, decimal amount)
    {
        // NUNCA hacer esto — operación transaccional dentro de una evaluación
        var repo = ServiceLocator.Get<IAccountRepository>();
        repo.Debit(accountId, amount);
        return true;
    }
}
```

Las reglas evalúan. Las acciones las ejecuta la aplicación basándose en los resultados de la evaluación.

### No Versionar Reglas

Modificar reglas en producción sin mantener historial de versiones anteriores es uno de los errores más graves. Cuando una regla produce un resultado inesperado, sin versionado no se puede:

- Saber qué versión de las reglas se usó para una evaluación anterior.
- Hacer rollback a una versión que funcionaba.
- Auditar quién cambió qué y cuándo.

Mínimo viable: tabla con workflow_name + version + json + timestamp + modified_by.

### Ignorar ExceptionMessage

```csharp
// ANTI-PATRÓN: solo verificar IsSuccess
var rejected = results.Where(r => !r.IsSuccess).ToList();
// Esto mezcla reglas que legítimamente fallaron con reglas que lanzaron excepción

// CORRECTO: separar fallos de negocio de errores técnicos
var businessFailures = results.Where(r => !r.IsSuccess && string.IsNullOrEmpty(r.ExceptionMessage));
var technicalErrors = results.Where(r => !r.IsSuccess && !string.IsNullOrEmpty(r.ExceptionMessage));

if (technicalErrors.Any())
{
    // Alertar — esto es un bug, no una decisión de negocio
    logger.LogError("Technical errors in rule evaluation: {Errors}",
        technicalErrors.Select(e => $"{e.Rule.RuleName}: {e.ExceptionMessage}"));
}
```

---

# MÓDULO 14 — Checklist Final de Adopción y Producción

## Fase 1 — Evaluación

- Las reglas de negocio cambian con frecuencia suficiente (>1x/mes) para justificar un motor.
- Se ha identificado un bounded context claro donde aplicar el motor (no todo el sistema).
- Se ha evaluado el Specification Pattern como alternativa más simple.
- Se ha evaluado NRules si se necesita forward chaining.
- Se ha realizado un POC con un caso real del negocio, no con ejemplos triviales.
- El equipo comprende la diferencia entre invariantes de dominio y políticas de negocio.
- Se ha estimado el volumen de reglas esperado a 6 y 12 meses.

## Fase 2 — Diseño

- Se ha definido dónde se almacenan las reglas (DB, archivos, servicio externo).
- Se ha definido el esquema de versionado (semántico, por fecha, incremental).
- Se ha definido quién puede crear, modificar y aprobar reglas (modelo de gobernanza).
- Se han diseñado los DTOs de input para el motor (no pasar entidades de dominio directamente).
- Se han identificado todos los Custom Types y Custom Functions necesarios.
- Se ha definido la integración con DI (Singleton + swap atómico para recarga).
- Se ha definido la estrategia de logging (correlationId, duración, resultados).
- Se ha definido la separación de workflows por dominio.
- Se ha definido la convención de nomenclatura de workflows y reglas.

## Fase 3 — Implementación

- Existe test unitario para cada regla individual.
- Existe test parametrizado (Theory) para escenarios de borde por regla.
- Existe test de integración para cada workflow completo.
- Existe validación de sintaxis de JSON en el pipeline de CI/CD.
- Se ha verificado el comportamiento con inputs null y colecciones vacías.
- Se ha verificado thread safety con evaluaciones concurrentes.
- Se ha verificado performance con la carga esperada de producción.
- Se han registrado solo los CustomTypes estrictamente necesarios (seguridad).
- Se ha implementado validación de seguridad de expresiones (si el JSON es editable externamente).
- El motor se calienta (warmup) al inicio de la aplicación.

## Fase 4 — Producción

- Existe monitoreo de tiempo de evaluación por workflow (p50, p95, p99).
- Existe alerting cuando una evaluación supera el umbral de tiempo esperado.
- Existe alerting cuando hay errores técnicos (ExceptionMessage) en evaluaciones.
- Existe mecanismo de rollback de reglas operativo y probado.
- Existe auditoría de cada cambio en reglas (quién, qué, cuándo).
- Existe auditoría de cada evaluación para compliance (inputs, outputs, timestamp).
- Existe proceso de revisión (code review de reglas) antes de desplegar cambios.
- Existe runbook operativo documentado para incidentes con reglas.
- Los dashboards incluyen métricas de reglas (evaluaciones/minuto, tasa de aprobación, tasa de error).

## Fase 5 — Evolución

- Existe proceso trimestral de limpieza de reglas obsoletas o deshabilitadas.
- Se revisa periódicamente la performance ante crecimiento de reglas.
- Se actualiza la librería con criterio (leer changelog, verificar breaking changes en staging antes de producción).
- Se mide el ROI del motor vs la alternativa de simplificar.
- Se evalúa periódicamente si el volumen de reglas justifica tooling adicional (UI de gestión, motor de versionado).

## Recomendaciones Estratégicas Finales

### Perfil Ideal de Proyecto

- Aplicaciones financieras con reglas regulatorias que varían por país/producto.
- Plataformas multi-tenant donde cada tenant tiene reglas diferentes.
- Sistemas de scoring, pricing o elegibilidad donde el negocio necesita autonomía.
- Aplicaciones con >50 reglas que cambian al menos mensualmente.

### Casos Donde es Mala Elección

- Startups en fase temprana donde las reglas no están definidas — no externalizar lo que todavía no se entiende.
- Sistemas con <10 reglas estables — el overhead no se justifica.
- Lógica que requiere inferencia o razonamiento encadenado — NRules es mejor opción.
- Equipos sin disciplina de testing y versionado — las reglas externalizadas sin gobernanza son un riesgo.

### Roadmap de Adopción Sugerido


| Semana | Actividad                                                                        |
| ------ | -------------------------------------------------------------------------------- |
| 1      | POC con caso real más simple. Validar viabilidad técnica.                        |
| 2      | Definir modelo de almacenamiento y versionado. Implementar infraestructura base. |
| 3-4    | Migrar primer conjunto de reglas (un workflow productivo). Tests completos.      |
| 5      | Deploy a staging. Evaluación de performance.                                     |
| 6      | Deploy a producción con feature flag. Monitoreo intensivo.                       |
| 7-8    | Estabilización. Documentación de runbook. Capacitación del equipo.               |
| 9-12   | Migración gradual de workflows adicionales. Un workflow por sprint.              |
| 13+    | Operación normal. Evaluación de necesidad de tooling adicional.                  |



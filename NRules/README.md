### Módulo 1: Fundamentos Internos de NRules y la Anatomía de la Inferencia

Para entender NRules a nivel arquitectónico, primero tenemos que desaprender cómo escribimos código en nuestro día a día. Como desarrolladores .NET, estamos cableados para pensar de forma imperativa y secuencial. NRules requiere un cambio de paradigma hacia el pensamiento declarativo y basado en conjuntos (set-based). 

En este capítulo vamos a destripar el motor. No vamos a ver cómo escribir un `Rule` básico, vamos a ver qué pasa en la memoria de la CLR cuando ese `Rule` se compila y se ejecuta.

---

#### 1. La Verdadera Naturaleza de un Motor de Reglas

Un motor de reglas no es un "evaluador de if-statements glorificado". Si pensás en un motor de reglas como un reemplazo para el patrón *Strategy* o una cadena de responsabilidad (*Chain of Responsibility*), estás subestimando la herramienta y, probablemente, introduciendo un cuello de botella en tu sistema.

En términos formales, NRules es un **Sistema de Producción de Encadenamiento Hacia Adelante (Forward-Chaining Production System)**. 

**Diferencia conceptual con el código imperativo:**
En el código imperativo (C# tradicional), el flujo de control dicta la evaluación de los datos. Vos decís: *"Iterá sobre estas transacciones, y si la transacción es mayor a $10,000, buscá el perfil del cliente en la base de datos, y si el perfil es 'Riesgo Alto', marcala como fraude"*.
La complejidad temporal de esto suele ser $O(N \times M)$, donde $N$ son las transacciones y $M$ las condiciones. Si los datos cambian, tenés que volver a correr el algoritmo entero.

En un motor de reglas declarativo como NRules, **los datos dictan el flujo de control**. Vos definís patrones lógicos (las reglas) de forma aislada. Luego, arrojás los datos (Hechos o *Facts*) al motor. El motor hace un *Pattern Matching* continuo. No hay un `foreach`. Las reglas reaccionan a la presencia, ausencia o mutación de los datos en tiempo real. 

El código imperativo pregunta: *"¿Se cumplen estas condiciones para este dato?"*.
El motor de reglas afirma: *"Este dato acaba de completar este patrón, ejecutá esta acción"*.

---

#### 2. El Límite Arquitectónico: Cuándo usar NRules y cuándo NO

He visto proyectos fracasar estrepitosamente por meter un motor de reglas donde no iba. NRules es una herramienta de nicho para problemas de alta complejidad combinatoria.

**CUÁNDO USAR NRules (El Sweet Spot):**
1.  **Lógica Altamente Volátil y Combinatoria:** Sistemas de *Pricing* dinámico (ej. aerolíneas, e-commerce) donde conviven cientos de descuentos superpuestos, reglas de exclusión, cupones y condiciones temporales.
2.  **Scoring y Antifraude:** Cuando necesitás cruzar múltiples entidades dispares (Transacción, Historial del Cliente, Dispositivo, Geolocalización) para detectar patrones anómalos.
3.  **Validaciones de Dominio Complejas:** En arquitecturas DDD, cuando las invariantes de un Aggregate Root dependen del estado de otros Aggregates y la lógica de validación es demasiado extensa para vivir en el modelo de dominio sin violar el SRP (Single Responsibility Principle).
4.  **Separación de Lógica de Negocio Pura:** Cuando negocio exige poder leer (y eventualmente modificar mediante un DSL o UI) las reglas sin tocar el código core de la aplicación.

**CUÁNDO HUIR de NRules (Anti-patrones):**
1.  **Workflows Secuenciales o Máquinas de Estado:** Si tu lógica es *"Paso 1: Validar, Paso 2: Enviar Email, Paso 3: Cobrar"*, **NO uses NRules**. Usá un motor de Sagas (MassTransit) o un orquestador (Temporal.io, Azure Durable Functions). NRules no tiene concepto de "paso siguiente"; todo se evalúa en paralelo basado en el estado.
2.  **Transformación de Datos Pura (ETL):** Si solo estás mapeando de un DTO a otro basado en condiciones, usá LINQ, AutoMapper o código imperativo. El overhead de compilar la red Rete no se justifica.
3.  **Operaciones CRUD Simples:** Si tu regla es *"Si el usuario es Admin, puede borrar"*, un simple `if` o el sistema de autorización de ASP.NET Core es infinitamente superior y más mantenible.
4.  **Sistemas de Ultra-Baja Latencia (HFT - High Frequency Trading):** Aunque NRules es rapidísimo, la asignación de memoria en el heap para los *Facts* y el *Garbage Collection* inherente a .NET lo hacen inadecuado para sistemas que requieren latencias garantizadas por debajo del microsegundo.

---

#### 3. El Corazón de la Bestia: El Algoritmo Rete

Acá es donde separamos a los seniors de los arquitectos. NRules no evalúa tus reglas una por una. NRules implementa una variante orientada a objetos del **Algoritmo Rete** (creado por Charles Forgy en 1974).

Si tenés 1,000 reglas y le insertás 10,000 objetos, un motor ingenuo haría 10,000,000 de evaluaciones. Rete reduce esto drásticamente construyendo un **Grafo Acíclico Dirigido (DAG)** en memoria.

Cuando compilás tus reglas en NRules, el motor toma los *Expression Trees* de C# (tus lambdas en los métodos `When()`) y los desarma para construir esta red. La red Rete tiene dos partes fundamentales:

**A. La Red Alpha (Evaluación Intra-Fact):**
Se encarga de filtrar los objetos individuales.
*   **Root Node:** El punto de entrada de todos los objetos.
*   **Type Nodes:** Filtran por el tipo de clase de C#. Si insertás un `Customer`, solo va por la rama del `Customer`.
*   **Selection Nodes (Alpha Nodes):** Evalúan condiciones simples sobre propiedades del objeto. Ej: `c => c.Age > 18`.
*   **Alpha Memory:** ¡Clave! Una vez que un objeto pasa los nodos Alpha, se almacena en esta memoria. El motor **recuerda** que este objeto cumple estas condiciones.

**B. La Red Beta (Evaluación Inter-Fact / Joins):**
Acá ocurre la magia y el consumo de memoria. Se encarga de cruzar diferentes tipos de objetos (como un `JOIN` en SQL).
*   **Join Nodes (Beta Nodes):** Tienen dos entradas (Izquierda y Derecha). Comparan elementos de la memoria Alpha con elementos de la memoria Beta anterior. Ej: `(customer, order) => customer.Id == order.CustomerId`.
*   **Beta Memory:** Almacena **tuplas** (combinaciones de objetos) que han hecho match exitosamente hasta ese punto. Si un `Customer` y un `Order` hacen match, esa pareja se guarda en la memoria Beta.

**¿Cómo evita reevaluaciones innecesarias?**
Gracias a las memorias Alpha y Beta. Si tenés una regla que cruza `Customer`, `Order` y `Product`, y de repente cambia el `Product`, **NRules NO vuelve a evaluar el cruce entre `Customer` y `Order`**. Ese cruce ya está cacheado en la Memoria Beta. NRules solo toma el nuevo `Product`, lo pasa por la red Alpha, y lo intenta cruzar con las tuplas cacheadas en la Memoria Beta. 
Esto es lo que le da a Rete su rendimiento asombroso: **intercambia consumo de memoria por velocidad de CPU**.

---

#### 4. Anatomía de la Ejecución: Conceptos Core

Para dominar el pipeline, tenés que hablar el idioma del motor con precisión quirúrgica.

*   **Working Memory (Memoria de Trabajo):**
    Es el estado transaccional del motor. Pensalo como el `DbContext` de Entity Framework, pero para lógica en memoria. Es el contenedor donde viven tus *Facts*. La Working Memory es stateful (tiene estado) y es específica de cada sesión (`ISession`).

*   **Facts (Hechos):**
    Son simples objetos POCO de C# (clases o records) que insertás en la Working Memory. Representan la "verdad" del sistema en un momento dado. NRules no hace tracking automático de las propiedades de tus objetos (no usa proxies como EF). Si modificás una propiedad de un Fact, tenés que avisarle explícitamente al motor.

*   **Matching (Emparejamiento):**
    Es el proceso continuo y automático mediante el cual los Facts fluyen a través de la red Rete (Alpha y Beta). El matching ocurre en el momento exacto en que insertás, actualizás o retraés un Fact. **No ocurre cuando llamás a `Fire()`**. Cuando llamás a `Fire()`, el matching ya terminó.

*   **Activación (Activation):**
    Cuando un conjunto de Facts logra atravesar toda la red Rete y llega a un **Terminal Node** (el final del grafo para una regla específica), se genera una Activación. Una Activación es una instancia que contiene la referencia a la Regla que hizo match y la Tupla exacta de Facts que la dispararon.

*   **Agenda:**
    Es una cola de prioridad interna de la sesión. Todas las Activaciones generadas por el proceso de Matching van a parar a la Agenda. La Agenda ordena estas activaciones basándose en la prioridad de la regla (Salience) y otros algoritmos de resolución de conflictos.

*   **Rule Firing (Disparo de Regla):**
    Es la ejecución real del bloque `Then()` (el *Right Hand Side* o RHS) de tu regla. Toma la Activación de la Agenda y ejecuta el delegado de C# pasándole los Facts correspondientes.

---

#### 5. Dinámica de Hechos y Propagación del Estado

El motor reacciona a tres operaciones fundamentales sobre la Working Memory. Entender cómo se propagan es vital para evitar bugs de rendimiento.

1.  **Insert:**
    Cuando hacés `Session.Insert(fact)`, el objeto entra al Root Node. Baja por los Type Nodes. Si pasa los Alpha Nodes, se guarda en la Alpha Memory. Luego intenta hacer joins en los Beta Nodes. Si llega al final, crea una Activación en la Agenda.
2.  **Retract:**
    Cuando hacés `Session.Retract(fact)`, el motor busca ese objeto en sus memorias. Lo elimina de las memorias Alpha y Beta. Si ese objeto formaba parte de una Activación en la Agenda, **esa Activación se cancela y se elimina de la Agenda silenciosamente**. Esto es el paradigma declarativo en su máxima expresión: si la condición deja de ser cierta antes de ejecutarse la acción, la acción se aborta sola.
3.  **Update:**
    Cuando hacés `Session.Update(fact)` (o `Context.Update(fact)` dentro de una regla), conceptualmente el motor hace un `Retract` seguido de un `Insert` (aunque a nivel interno Rete está optimizado para propagar solo el cambio). Esto recalcula toda la rama del grafo que dependía de ese objeto.

---

#### 6. Dependencias y Prevención de Reevaluaciones (Forward Chaining)

Acá es donde los desarrolladores suelen crear bucles infinitos (Infinite Loops) si no entienden el motor.

NRules soporta **Forward Chaining**. Esto significa que la acción de una regla (el `Then`) puede modificar la Working Memory (insertar, actualizar o retraer Facts), lo cual dispara un nuevo proceso de Matching, que puede generar nuevas Activaciones en la Agenda.

**¿Cómo maneja las dependencias?**
No las definís explícitamente. Rete lo infiere. 
Si la Regla A inserta un `DiscountFact`, y la Regla B tiene en su `When` una condición que escucha a `DiscountFact`, la Regla B depende implícitamente de la Regla A. Cuando la Regla A se dispare, insertará el Fact, el cual fluirá por la red Rete y activará la Regla B.

**¿Cómo evita reevaluaciones y bucles infinitos?**
NRules tiene un mecanismo de protección inherente. Si hacés un `Context.Update(fact)` dentro de una regla, y ese update no cambia el estado lógico que hizo que la regla se dispare en primer lugar, la regla **no se vuelve a activar para esa misma tupla exacta de Facts**.
Sin embargo, si modificás una propiedad que hace que el Fact coincida con *otra* regla, o si insertás un nuevo Fact, el motor evaluará solo los nodos afectados.

*Advertencia de trinchera:* Si en tu bloque `Then` hacés un `Context.Update(fact)` modificando una propiedad, y en tu bloque `When` no tenés una condición que excluya ese nuevo estado, vas a generar un bucle infinito. El motor detectará el cambio, reevaluará, verá que la condición se sigue cumpliendo, creará una nueva activación, y así hasta reventar el Stack o agotar la memoria.

---

#### 7. El Pipeline Interno de Ejecución (Deep Dive)

Para cerrar este módulo, veamos el ciclo de vida completo, paso a paso, de lo que ocurre en el procesador.

**Fase 1: Compilación (Ocurre una sola vez al inicio de la app)**
1.  Instanciás el `RuleRepository` y cargás tus clases que heredan de `Rule`.
2.  Llamás a `Compile()`.
3.  NRules usa Reflection para leer los métodos `When` y `Then`.
4.  Extrae los *Expression Trees* de C# de tus lambdas.
5.  Analiza los árboles de expresión para identificar dependencias entre variables (Facts).
6.  Construye el Grafo Rete (Nodos Alpha, Nodos Beta, Nodos Terminales).
7.  Emite un `ISessionFactory`. Este factory contiene el grafo inmutable y es *Thread-Safe*.

**Fase 2: Runtime - Creación de Sesión**
1.  Llamás a `factory.CreateSession()`.
2.  Se crea una `ISession`. Esta sesión asigna memoria fresca para las Memorias Alpha, Memorias Beta y la Agenda. **No es Thread-Safe**. Cada request/hilo debe tener su propia sesión.

**Fase 3: Ingesta de Datos (Matching)**
1.  Llamás a `session.Insert(misDatos)`.
2.  El hilo actual bloquea y ejecuta el recorrido del objeto por la red Rete.
3.  Se pueblan las memorias intermedias.
4.  Se generan Activaciones y se encolan en la Agenda.
5.  *Nota:* Hasta acá, ninguna regla ejecutó su bloque `Then`.

**Fase 4: El Ciclo de Disparo (The Fire Loop)**
1.  Llamás a `session.Fire()`.
2.  El motor entra en un bucle `while (Agenda.HasActivations)`.
3.  **Resolución de Conflictos:** El motor mira la Agenda y elige la Activación con mayor prioridad (Salience). Si hay empate, usa el orden de inserción o complejidad de la regla.
4.  Saca la Activación de la Agenda.
5.  Ejecuta el delegado compilado del bloque `Then`, pasándole los Facts de la tupla.
6.  **Mutación:** Si el bloque `Then` contiene `Context.Insert/Update/Retract`, el motor pausa el bucle de disparo, ejecuta el Matching de esos cambios inmediatamente, actualiza la red Rete, y posiblemente añade o elimina Activaciones de la Agenda.
7.  El bucle vuelve al paso 2.
8.  Cuando la Agenda está vacía, el método `Fire()` retorna la cantidad de reglas ejecutadas.

Este pipeline garantiza que el estado del sistema sea siempre consistente y que las reglas reaccionen a las consecuencias de otras reglas de forma determinista.

---

Este es el fundamento teórico y arquitectónico. Si no dominás cómo la red Rete maneja la memoria y cómo la Agenda orquesta las activaciones, vas a escribir reglas que funcionen lento o que se comporten de forma impredecible en producción.

---

### Módulo 2: Setup Arquitectónico y Primer Caso Empresarial Real

Implementar NRules en una aplicación empresarial no es simplemente instalar un paquete NuGet y hacer un `new RuleEngine()`. Requiere una alineación estricta con los principios de Clean Architecture y Domain-Driven Design (DDD) para evitar que la lógica de negocio se acople a la infraestructura del motor.

En este capítulo, vamos a construir un **Motor de Descuentos Comerciales (Pricing Engine)**. Este es un escenario clásico donde el código imperativo colapsa bajo su propio peso debido a la cantidad de combinaciones (descuentos por volumen, por categoría de cliente, promociones temporales cruzadas, etc.).

---

#### 1. Instalación y Topología en Clean Architecture

A nivel de dependencias, solo necesitás instalar el paquete principal en los proyectos correspondientes:

```bash
dotnet add package NRules
```

**¿Dónde viven las reglas y los hechos (Facts)?**
Este es el punto de quiebre arquitectónico. He visto equipos poner las reglas en la capa de Infraestructura porque "NRules es una herramienta externa". Esto es un error conceptual grave. Las reglas **son** tu lógica de negocio.

La estructura recomendada es la siguiente:

1.  **`Core.Domain` (Sin dependencias de NRules):**
    Aquí viven tus *Facts*. Los Facts deben ser POCOs (Plain Old CLR Objects), preferiblemente `records` o clases con encapsulamiento. No deben saber que un motor de reglas existe.
2.  **`Core.Application` (Depende de `NRules.Fluent`):**
    Aquí viven las definiciones de tus reglas (las clases que heredan de `Rule`). También aquí definís la interfaz de tu servicio de dominio o caso de uso, por ejemplo, `IDiscountService`.
3.  **`Infrastructure` (Depende de `NRules` y `NRules.RuleBuilder`):**
    Aquí implementás el `IDiscountService`. Esta capa se encarga de instanciar el repositorio de NRules, compilar la red Rete (crear el `ISessionFactory`), gestionar el ciclo de vida de la `ISession` y orquestar la inserción de los Facts.

---

#### 2. El Escenario Empresarial: Motor de Descuentos

Vamos a definir nuestro Dominio (Facts). Usaremos C# moderno.

```csharp
// Capa: Core.Domain
public enum CustomerTier { Standard, Silver, Gold, Platinum }

public record Customer(string Id, string Name, CustomerTier Tier, DateTime RegisteredAt);

public class OrderItem
{
    public string Sku { get; init; }
    public string Category { get; init; }
    public decimal UnitPrice { get; init; }
    public int Quantity { get; init; }
    public decimal LineTotal => UnitPrice * Quantity;
}

public class OrderDiscount
{
    public string Reason { get; init; }
    public decimal Amount { get; init; }
}

public class Order
{
    public string Id { get; init; }
    public string CustomerId { get; init; }
    public List<OrderItem> Items { get; init; } = new();
    public List<OrderDiscount> Discounts { get; init; } = new();
    
    public decimal SubTotal => Items.Sum(x => x.LineTotal);
    public decimal TotalDiscount => Discounts.Sum(x => x.Amount);
    public decimal FinalTotal => SubTotal - TotalDiscount;

    public void ApplyDiscount(string reason, decimal amount)
    {
        Discounts.Add(new OrderDiscount { Reason = reason, Amount = amount });
    }
}
```

*Nota Arquitectónica:* Observá que `Order` tiene un método `ApplyDiscount`. Mantenemos el comportamiento rico en el dominio. Las reglas de NRules no van a hacer matemáticas complejas, van a evaluar condiciones y delegar la mutación al modelo de dominio.

---

#### 3. Definiendo las Reglas (Core.Application)

Vamos a crear dos reglas de negocio complejas. 

**Regla 1: Descuento por Volumen para Clientes Gold.**
*Condición:* Si el cliente es Gold, y tiene una orden con un SubTotal mayor a $1000, aplicar un 10% de descuento sobre el SubTotal.

```csharp
// Capa: Core.Application
using NRules.Fluent.Dsl;

public class GoldMemberVolumeDiscountRule : Rule
{
    public override void Define()
    {
        Customer customer = null;
        Order order = null;

        // Metadatos útiles para telemetría o logs
        Name("Gold_Volume_Discount");
        Description("Aplica 10% de descuento a clientes Gold con compras mayores a $1000");
        Priority(100); // Salience: Mayor número, mayor prioridad en la Agenda

        When()
            // Nodo Alpha: Filtra solo clientes Gold
            .Match<Customer>(() => customer, c => c.Tier == CustomerTier.Gold)
            
            // Nodo Beta (Join): Cruza la Orden con el Cliente, y aplica filtros Alpha a la Orden
            .Match<Order>(() => order, 
                o => o.CustomerId == customer.Id, // JOIN condition
                o => o.SubTotal >= 1000m,         // Condición de volumen
                o => !o.Discounts.Any(d => d.Reason == "GOLD_VOLUME")); // GUARD CLAUSE (Vital)

        Then()
            .Do(ctx => ApplyDiscount(ctx, order));
    }

    private void ApplyDiscount(IContext ctx, Order order)
    {
        decimal discountAmount = order.SubTotal * 0.10m;
        order.ApplyDiscount("GOLD_VOLUME", discountAmount);
        
        // Le avisamos al motor que el Fact mutó. 
        // Esto dispara el Forward Chaining.
        ctx.Update(order); 
    }
}
```

**Regla 2: Promoción Cruzada de Categoría.**
*Condición:* Si la orden contiene al menos un producto de la categoría "Electronics", y el total final (después de otros descuentos) sigue siendo mayor a $500, dar un descuento fijo de $50.

```csharp
public class ElectronicsCrossPromoRule : Rule
{
    public override void Define()
    {
        Order order = null;

        Name("Electronics_Promo");
        Priority(50); // Se ejecuta después de la regla Gold (que tiene 100)

        When()
            .Match<Order>(() => order,
                o => o.Items.Any(i => i.Category == "Electronics"),
                o => o.FinalTotal >= 500m,
                o => !o.Discounts.Any(d => d.Reason == "ELEC_PROMO"));

        Then()
            .Do(ctx => ApplyPromo(ctx, order));
    }

    private void ApplyPromo(IContext ctx, Order order)
    {
        order.ApplyDiscount("ELEC_PROMO", 50m);
        ctx.Update(order);
    }
}
```

---

#### 4. Compilación y Ejecución (Infrastructure)

Aquí es donde el 90% de los desarrolladores cometen errores de rendimiento. 
La compilación de la red Rete (`ISessionFactory`) es una operación pesada (CPU bound) porque desarma *Expression Trees*. **Debe ser un Singleton**.
La sesión (`ISession`) es liviana y stateful. **Debe ser Transient o Scoped**.

```csharp
// Capa: Infrastructure
using NRules;
using NRules.Fluent;

public class DiscountEngineService
{
    private readonly ISessionFactory _sessionFactory;

    // Esto normalmente se inyecta vía DI como Singleton
    public DiscountEngineService()
    {
        var repository = new RuleRepository();
        
        // Escanea el assembly actual (o el de Application) buscando clases que hereden de Rule
        repository.Load(x => x.From(typeof(GoldMemberVolumeDiscountRule).Assembly));

        // Compila la red Rete. ESTO ES COSTOSO. Hacerlo solo una vez.
        var builder = new RuleCompiler();
        _sessionFactory = builder.Compile(repository.GetRules());
    }

    public Order CalculateDiscounts(Customer customer, Order order)
    {
        // 1. Crear sesión (Liviano, asigna memoria para la Agenda y Working Memory)
        ISession session = _sessionFactory.CreateSession();

        // 2. Ingesta de Facts (Acá ocurre el Pattern Matching)
        session.Insert(customer);
        session.Insert(order);

        // 3. Ejecución (Fire Loop)
        session.Fire();

        // El objeto 'order' ha sido mutado por referencia si las reglas aplicaron
        return order;
    }
}
```

---

#### 5. Anatomía de la Ejecución: ¿Qué pasa internamente paso a paso?

Vamos a trazar exactamente qué hace la CPU cuando llamamos a `CalculateDiscounts` con un cliente Gold y una orden de $1200 en Electrónica.

1.  **`session.Insert(customer)`:**
    *   El objeto `Customer` entra al Root Node.
    *   Pasa al Type Node de `Customer`.
    *   Se evalúa el Alpha Node: `c.Tier == CustomerTier.Gold`. Es `true`.
    *   El `Customer` se guarda en la **Alpha Memory** del nodo de clientes Gold.
    *   Intenta avanzar al Beta Node (Join con Order), pero la memoria Alpha de `Order` está vacía. Se detiene.

2.  **`session.Insert(order)`:**
    *   El objeto `Order` entra al Root Node y pasa al Type Node de `Order`.
    *   Se evalúan los Alpha Nodes de la Regla 1: `SubTotal >= 1000` (True) y `!Discounts.Any(...)` (True).
    *   Se evalúan los Alpha Nodes de la Regla 2: `Items.Any(Electronics)` (True), `FinalTotal >= 500` (True), `!Discounts.Any(...)` (True).
    *   El `Order` se guarda en las **Alpha Memories** correspondientes.
    *   **El Join (Regla 1):** El motor cruza el `Order` recién llegado con el `Customer` que estaba esperando en la memoria. Evalúa `o.CustomerId == customer.Id`. Es `true`.
    *   Se crea una **Activación** para `GoldMemberVolumeDiscountRule` y se envía a la **Agenda**.
    *   **El Join (Regla 2):** Como la Regla 2 no cruza con `Customer`, pasa directamente y crea una **Activación** para `ElectronicsCrossPromoRule` en la **Agenda**.

3.  **`session.Fire()`:**
    *   El motor revisa la Agenda. Hay dos activaciones.
    *   Evalúa prioridades: Regla 1 (Priority 100) vs Regla 2 (Priority 50).
    *   Saca la Regla 1 de la Agenda y ejecuta su bloque `Then()`.
    *   Se añade el descuento de $120 (10% de 1200) a la orden.
    *   **Se ejecuta `ctx.Update(order)`:** ¡Punto crítico!
    *   El motor detecta que `Order` cambió. Propaga el cambio por la red Rete.
    *   *Reevaluación Regla 1:* Evalúa `!Discounts.Any(d => d.Reason == "GOLD_VOLUME")`. Ahora es **False**. La tupla se invalida. No se vuelve a encolar. Evitamos el bucle infinito.
    *   *Reevaluación Regla 2:* Evalúa `FinalTotal >= 500`. El SubTotal era 1200, menos 120 de descuento, el FinalTotal es 1080. Sigue siendo mayor a 500. La activación de la Regla 2 se mantiene en la Agenda (o se recrea, dependiendo de la optimización interna).
    *   El motor vuelve a la Agenda. Saca la Regla 2.
    *   Ejecuta el `Then()` de la Regla 2. Añade $50 de descuento.
    *   Ejecuta `ctx.Update(order)`.
    *   El motor propaga el cambio. La cláusula de guarda de la Regla 2 ahora es **False**.
    *   La Agenda queda vacía.
    *   `Fire()` termina y devuelve el control al hilo principal.

---

#### 6. Edge Cases y Errores Comunes (Trincheras de Producción)

Si implementás el código de arriba sin entender estos edge cases, vas a tener problemas en producción.

**A. El "Update Loop of Death" (Bucle Infinito)**
Si en la `GoldMemberVolumeDiscountRule` olvidás poner la condición `o => !o.Discounts.Any(d => d.Reason == "GOLD_VOLUME")`, esto es lo que pasa:
1. La regla hace match.
2. Aplica el descuento.
3. Llama a `ctx.Update(order)`.
4. El motor reevalúa. El cliente sigue siendo Gold. El SubTotal sigue siendo > 1000.
5. Vuelve a hacer match. Vuelve a la Agenda.
6. Aplica otro 10% de descuento.
7. Llama a `ctx.Update(order)`.
8. Repite hasta agotar la memoria (Out Of Memory) o reventar el Stack.
*Regla de oro:* Si tu bloque `Then` muta un Fact y llama a `Update`, tu bloque `When` **debe** tener una condición que se vuelva falsa tras esa mutación.

**B. Excepciones dentro de las Reglas (NullReferenceException)**
NRules compila los lambdas en delegados. Si en tu regla hacés `o => o.Customer.Address.City == "NY"` y `Address` es null, NRules atrapará la `NullReferenceException` internamente durante el matching y asumirá que la condición es **False**.
*Sin embargo*, depender de esto es una pésima práctica arquitectónica porque oculta problemas de datos y genera un overhead de excepciones lanzadas y atrapadas en el CLR (lo cual destruye el rendimiento).
*Solución:* Usá el operador de navegación segura de C# o chequeos explícitos: `o => o.Customer != null && o.Customer.Address != null && o.Customer.Address.City == "NY"`.

**C. Problemas de Precisión con Flotantes**
En sistemas de pricing, **nunca uses `double` o `float`** en tus Facts. NRules evaluará `o.Total == 100.00` y debido a la aritmética de punto flotante de la CPU, el valor real podría ser `100.00000000000001`, haciendo que la regla falle silenciosamente. Usá siempre `decimal` para Facts financieros, tanto en las propiedades como en los literales de las reglas (`1000m`).

**D. Modificar Colecciones sin Avisar**
Si tenés una regla que evalúa `o.Items.Count > 5`, y en el `Then` hacés `order.Items.Add(newItem)`, pero **no** llamás a `ctx.Update(order)`, el motor de reglas jamás se enterará de que la colección cambió. La Working Memory quedará desincronizada con el estado real del objeto en el heap de .NET.

Este es el setup fundacional. Con esta arquitectura, podés escalar a cientos de reglas sin tocar el `DiscountEngineService`. 

---

### Módulo 3: Matching Avanzado y la Anatomía de los Joins Complejos

Hasta ahora, vimos cómo NRules filtra objetos individuales (Alpha Nodes) y cómo cruza dos objetos simples (Beta Nodes). Pero en un sistema empresarial real, la lógica de negocio rara vez es tan plana. Necesitamos evaluar la existencia o ausencia de datos, agregar colecciones enteras, cruzar múltiples entidades y generar nuevos hechos dinámicamente.

En este capítulo, vamos a exprimir el DSL (Domain Specific Language) de NRules al máximo. Vamos a construir un **Motor de Detección de Fraude Transaccional**. Este dominio es perfecto porque requiere cruzar perfiles de usuario, historiales de transacciones, listas negras y ventanas de tiempo.

---

#### 1. El Dominio: Detección de Fraude

Definamos los Facts que van a vivir en nuestra Working Memory.

```csharp
// Core.Domain
public record UserProfile(string UserId, string CountryOfOrigin, DateTime AccountCreated);

public record Transaction(string TransactionId, string UserId, decimal Amount, string MerchantCountry, DateTime Timestamp);

public record BlacklistedMerchant(string MerchantId, string Reason);

// Un Fact generado dinámicamente por el motor
public class SuspiciousActivity
{
    public string UserId { get; init; }
    public string Reason { get; init; }
    public int SeverityLevel { get; init; }
}

// El estado final que queremos mutar
public class FraudAlert
{
    public string UserId { get; init; }
    public bool IsBlocked { get; set; }
    public List<string> Reasons { get; init; } = new();
}
```

---

#### 2. Joins Múltiples y Dependencias de Variables

El poder real de Rete se desata cuando cruzás múltiples Facts. NRules permite declarar variables en el método `Define()` y usarlas a lo largo de toda la regla.

**Regla 1: Transacción Internacional de Alto Riesgo**
*Condición:* Si un usuario tiene una cuenta creada hace menos de 30 días, y realiza una transacción mayor a $5000 en un país distinto al de su origen, generar una actividad sospechosa.

```csharp
// Core.Application
using NRules.Fluent.Dsl;

public class HighRiskInternationalTransactionRule : Rule
{
    public override void Define()
    {
        UserProfile user = null;
        Transaction tx = null;

        Name("HighRisk_International_Tx");
        Priority(100);

        When()
            // 1. Nodo Alpha: Filtra usuarios nuevos
            .Match<UserProfile>(() => user, 
                u => (DateTime.UtcNow - u.AccountCreated).TotalDays < 30)
            
            // 2. Nodo Beta (Join): Cruza la transacción con el usuario
            .Match<Transaction>(() => tx, 
                t => t.UserId == user.UserId, // JOIN
                t => t.Amount > 5000m,        // Alpha filter en Transaction
                t => t.MerchantCountry != user.CountryOfOrigin); // JOIN complejo

        Then()
            .Do(ctx => GenerateSuspicion(ctx, user, tx));
    }

    private void GenerateSuspicion(IContext ctx, UserProfile user, Transaction tx)
    {
        // Generamos un NUEVO Fact y lo insertamos en la Working Memory
        var suspicion = new SuspiciousActivity
        {
            UserId = user.UserId,
            Reason = $"Tx {tx.TransactionId} of {tx.Amount} in {tx.MerchantCountry} differs from origin {user.CountryOfOrigin}",
            SeverityLevel = 8
        };
        
        // Forward Chaining: Esto disparará otras reglas que escuchen a SuspiciousActivity
        ctx.Insert(suspicion); 
    }
}
```

**Impacto en Rete:**
Cuando hacés `t.MerchantCountry != user.CountryOfOrigin`, estás creando un **Beta Node** que compara una propiedad del Fact actual (`Transaction`) con una propiedad de un Fact que ya pasó por la red (`UserProfile`). Esto es computacionalmente más caro que un filtro Alpha (`t.Amount > 5000m`), por lo que NRules optimiza evaluando los filtros Alpha *antes* de intentar el Join.

---

#### 3. Operadores Cuantificadores: Exists, Not, All

A veces no querés cruzar datos para extraerlos, sino simplemente verificar su existencia o ausencia. NRules provee operadores específicos que optimizan la red Rete creando **Nodos Existenciales (Existential Nodes)** y **Nodos Negativos (Negative Nodes)**.

**Regla 2: Bloqueo por Múltiples Sospechas (Uso de `Exists` y Mutación de Estado)**
*Condición:* Si existe un usuario, y existe al menos una actividad sospechosa con severidad > 7 para ese usuario, y existe una alerta de fraude abierta, bloquear al usuario.

```csharp
public class BlockUserOnHighSeveritySuspicionRule : Rule
{
    public override void Define()
    {
        UserProfile user = null;
        FraudAlert alert = null;

        Name("Block_User_High_Severity");
        Priority(200); // Alta prioridad para bloquear rápido

        When()
            .Match<UserProfile>(() => user)
            
            // JOIN con el estado a mutar
            .Match<FraudAlert>(() => alert, 
                a => a.UserId == user.UserId,
                a => !a.IsBlocked) // Guard clause vital
            
            // Operador EXISTS: No extrae el Fact, solo verifica si hay al menos uno.
            // Es mucho más rápido que hacer un Match normal si hay 1000 sospechas.
            .Exists<SuspiciousActivity>(
                s => s.UserId == user.UserId,
                s => s.SeverityLevel > 7);

        Then()
            .Do(ctx => BlockUser(ctx, alert));
    }

    private void BlockUser(IContext ctx, FraudAlert alert)
    {
        alert.IsBlocked = true;
        alert.Reasons.Add("High severity suspicious activity detected.");
        
        // Mutamos el estado y avisamos al motor
        ctx.Update(alert);
    }
}
```

**Regla 3: Transacción Segura (Uso de `Not`)**
*Condición:* Si hay una transacción, y **NO** existe ninguna actividad sospechosa para ese usuario en la memoria, marcarla como procesada (simulado).

```csharp
public class SafeTransactionRule : Rule
{
    public override void Define()
    {
        Transaction tx = null;

        Name("Safe_Transaction");
        Priority(10); // Baja prioridad, se ejecuta si no saltó nada más

        When()
            .Match<Transaction>(() => tx)
            
            // Operador NOT: La regla solo se activa si la memoria Beta está VACÍA para esta condición.
            .Not<SuspiciousActivity>(s => s.UserId == tx.UserId);

        Then()
            .Do(ctx => Console.WriteLine($"Tx {tx.TransactionId} is safe."));
    }
}
```

**Impacto en Rete (`Not` y `Exists`):**
Un nodo `Not` es un Beta Node especial. Mantiene un contador de coincidencias. Si el contador es 0, deja pasar la tupla. Si entra un Fact que hace que el contador suba a 1, la tupla se retrae inmediatamente de la memoria y la Activación se cancela de la Agenda.
Un nodo `Exists` es lo opuesto. Si el contador pasa de 0 a 1, deja pasar la tupla. Si sube a 2, 3 o 100, **no hace nada**. No genera nuevas activaciones por cada Fact extra, lo cual es una optimización brutal de rendimiento frente a un `Match` normal.

---

#### 4. Agrupación y Colecciones: Collect y LINQ Avanzado

El escenario más complejo en motores de reglas es cuando necesitás evaluar un conjunto de Facts como un todo. Por ejemplo: *"Si el usuario hizo más de 5 transacciones en las últimas 24 horas que sumen más de $10,000"*.

No podés hacer esto con `Match` simple porque `Match` evalúa Facts de a uno. Necesitás el operador `Query()`, que te permite usar LINQ sobre la Working Memory y agrupar Facts en colecciones.

**Regla 4: Velocity Check (Ataque de Fuerza Bruta Financiera)**
*Condición:* Si un usuario tiene más de 3 transacciones en las últimas 24 horas, y la suma de esas transacciones supera los $10,000, generar una sospecha.

```csharp
public class VelocityCheckRule : Rule
{
    public override void Define()
    {
        UserProfile user = null;
        IEnumerable<Transaction> recentTxs = null;

        Name("Velocity_Check_High_Volume");
        Priority(150);

        When()
            .Match<UserProfile>(() => user)
            
            // Operador QUERY: Transforma múltiples Facts individuales en una sola colección (IEnumerable)
            .Query(() => recentTxs, q => q
                .Match<Transaction>(
                    t => t.UserId == user.UserId, // JOIN
                    t => (DateTime.UtcNow - t.Timestamp).TotalHours <= 24) // Filtro temporal
                .Collect() // Agrupa todos los matches en un IEnumerable<Transaction>
                .Where(txs => txs.Count() > 3) // LINQ sobre la colección
                .Where(txs => txs.Sum(t => t.Amount) > 10000m)); // LINQ sobre la colección

        Then()
            .Do(ctx => GenerateVelocitySuspicion(ctx, user, recentTxs));
    }

    private void GenerateVelocitySuspicion(IContext ctx, UserProfile user, IEnumerable<Transaction> txs)
    {
        var suspicion = new SuspiciousActivity
        {
            UserId = user.UserId,
            Reason = $"Velocity check failed: {txs.Count()} txs totaling {txs.Sum(t => t.Amount)} in 24h",
            SeverityLevel = 9
        };
        
        ctx.Insert(suspicion);
    }
}
```

**Impacto en Rete (`Collect`):**
El nodo `Collect` (Accumulator Node) es el más pesado de la red Rete. Acumula todos los Facts que hacen match en una lista interna. Cada vez que entra o sale un Fact que cumple la condición, el nodo recalcula la colección entera y la pasa al siguiente nodo (los `Where` de LINQ). 
*Advertencia de Trinchera:* Si tenés 100,000 transacciones en memoria, un `Collect` mal filtrado va a destruir tu CPU y tu Garbage Collector. Siempre poné los filtros más restrictivos (como la ventana de tiempo de 24h) **antes** del `.Collect()`.

---

#### 5. Comparaciones Temporales y el Problema del Reloj

En las reglas anteriores usamos `DateTime.UtcNow`. Esto es un **anti-patrón arquitectónico** en motores de reglas si no se maneja con cuidado.

**El problema:**
La red Rete evalúa las condiciones en el momento de la inserción (`session.Insert()`). Si insertás una transacción a las 10:00 AM, y la regla dice `(DateTime.UtcNow - t.Timestamp).TotalHours <= 24`, la regla se evalúa con el `UtcNow` de las 10:00 AM.
Si dejás la sesión abierta en memoria (por ejemplo, en un servicio Singleton o un actor de Orleans) y llegan las 11:00 AM del día siguiente, **la regla NO se va a reevaluar sola**. El motor no sabe que el tiempo pasó. Para el motor, el Fact sigue cumpliendo la condición porque nadie llamó a `Update()`.

**La Solución (Inyección de Tiempo):**
Nunca uses `DateTime.UtcNow` directamente en el `When()`. Inyectá un servicio de reloj o pasá el tiempo actual como un Fact separado.

```csharp
// El Fact del reloj
public record TimeReference(DateTime CurrentTime);

// Regla corregida
public class VelocityCheckRuleCorrected : Rule
{
    public override void Define()
    {
        UserProfile user = null;
        TimeReference time = null;
        IEnumerable<Transaction> recentTxs = null;

        When()
            .Match<TimeReference>(() => time) // Obtenemos el tiempo actual como Fact
            .Match<UserProfile>(() => user)
            .Query(() => recentTxs, q => q
                .Match<Transaction>(
                    t => t.UserId == user.UserId,
                    // Usamos el Fact del tiempo, no DateTime.UtcNow
                    t => (time.CurrentTime - t.Timestamp).TotalHours <= 24) 
                .Collect()
                .Where(txs => txs.Count() > 3));
        // ...
    }
}
```
*Por qué esto funciona:* Si tenés un proceso en background (un `Timer` o un `Worker`) que cada 1 minuto hace `session.Update(new TimeReference(DateTime.UtcNow))`, el motor detectará el cambio en el Fact `TimeReference`, propagará el cambio por la red Rete, y reevaluará las ventanas de tiempo de todas las transacciones cacheadas.

---

#### 6. Colecciones dentro de Facts (Any / All)

A veces la colección no se forma agrupando Facts individuales, sino que ya viene dentro de un Fact (como `Order.Items` en el Módulo 2).

Para evaluar colecciones internas, NRules soporta LINQ nativo dentro del `Match`.

**Regla 5: Uso de `All` (Para listas blancas)**
*Condición:* Si un usuario tiene una lista de países de confianza, y **todas** sus transacciones recientes fueron en esos países, bajar el nivel de riesgo.

Supongamos que `UserProfile` tiene `List<string> TrustedCountries`.

```csharp
When()
    .Match<UserProfile>(() => user,
        // LINQ 'All' se traduce perfectamente en la red Alpha
        u => u.TrustedCountries.All(c => c != "HighRiskCountry"));
```

*Nota sobre LINQ en NRules:*
NRules no ejecuta tu LINQ como código C# normal. Desarma el *Expression Tree* del `.All()` o `.Any()` y lo convierte en nodos de evaluación. Si usás métodos personalizados o librerías externas dentro del `Match` (ej. `u => MiServicio.Validar(u)`), NRules no podrá desarmarlo y lo tratará como una "caja negra" (Opaque Expression), lo cual reduce las optimizaciones de Rete. Mantené las expresiones en el `When` lo más puras y nativas posible.

---

Este módulo cubre el 95% de los escenarios complejos de matching que vas a encontrar en la industria. La combinación de `Query`, `Collect`, `Exists` y Forward Chaining (insertar Facts desde el `Then`) te permite modelar árboles de decisión infinitamente complejos sin escribir un solo `if` anidado.

---

### Módulo 4: Casos Empresariales Avanzados y Arquitecturas de Decisión

En este nivel, dejamos de lado los ejemplos de juguete. Vamos a entrar en las trincheras de la arquitectura empresarial. Cuando implementás NRules en corporaciones, no lidiás con "Regla A" y "Regla B"; lidiás con matrices de decisión de 500 reglas interconectadas, equipos de compliance auditando tus resultados, y latencias estrictas.

Para cada caso, voy a destripar el dominio, el código, el impacto en la memoria (Rete) y, lo más importante como Arquitecto: **cuándo NRules es la herramienta correcta y cuándo te estás disparando en el pie.**

---

#### Caso 1: Scoring Crediticio (Credit Scoring Engine)

**Contexto de Negocio:**
Un banco digital necesita evaluar solicitudes de préstamos en tiempo real. El puntaje (Score) no es un cálculo matemático simple; es un sistema de acumulación y deducción basado en el historial del buró de crédito, relación deuda-ingreso (DTI), estabilidad laboral y políticas de riesgo vigentes. Compliance exige **explicabilidad total**: si se rechaza a un cliente, el sistema debe decir exactamente qué reglas restaron puntos.

**Modelo de Dominio:**
Usamos el patrón **Accumulator Fact**. En lugar de mutar la solicitud, creamos un Fact dedicado a recolectar el puntaje y las razones.

```csharp
public record LoanApplication(string ApplicationId, string ApplicantId, decimal RequestedAmount, decimal MonthlyIncome, decimal TotalMonthlyDebt, int MonthsAtCurrentJob);

public record CreditBureauData(string ApplicantId, int BaseFicoScore, int NumberOfLatePayments, bool HasBankruptcies);

// El Accumulator Fact (Stateful)
public class ScoreResult
{
    public string ApplicationId { get; init; }
    public int FinalScore { get; set; }
    public List<string> AuditTrail { get; } = new();
    public bool IsRejected => FinalScore < 600;

    public void AddPoints(int points, string reason)
    {
        FinalScore += points;
        AuditTrail.Add($"+{points}: {reason}");
    }

    public void DeductPoints(int points, string reason)
    {
        FinalScore -= points;
        AuditTrail.Add($"-{points}: {reason}");
    }
}
```

**Reglas (Patrón de Acumulación):**

```csharp
public class BaseScoreRule : Rule
{
    public override void Define()
    {
        LoanApplication app = null;
        CreditBureauData bureau = null;

        Name("Credit_Base_Score");
        Priority(1000); // Se ejecuta primero para inicializar el Score

        When()
            .Match<LoanApplication>(() => app)
            .Match<CreditBureauData>(() => bureau, b => b.ApplicantId == app.ApplicantId)
            // Aseguramos que no exista ya un ScoreResult para evitar re-inicializar
            .Not<ScoreResult>(s => s.ApplicationId == app.ApplicationId);

        Then()
            .Do(ctx => InitializeScore(ctx, app, bureau));
    }

    private void InitializeScore(IContext ctx, LoanApplication app, CreditBureauData bureau)
    {
        var score = new ScoreResult { ApplicationId = app.ApplicationId, FinalScore = bureau.BaseFicoScore };
        score.AuditTrail.Add($"Base FICO: {bureau.BaseFicoScore}");
        ctx.Insert(score); // Dispara el Forward Chaining para las reglas de deducción
    }
}

public class DtiPenaltyRule : Rule
{
    public override void Define()
    {
        LoanApplication app = null;
        ScoreResult score = null;

        Name("Credit_DTI_Penalty");
        Priority(500);

        When()
            .Match<LoanApplication>(() => app, 
                a => (a.TotalMonthlyDebt / a.MonthlyIncome) > 0.40m) // DTI > 40%
            .Match<ScoreResult>(() => score, 
                s => s.ApplicationId == app.ApplicationId,
                s => !s.AuditTrail.Any(t => t.Contains("DTI Penalty"))); // Guard Clause

        Then()
            .Do(ctx => ApplyPenalty(ctx, score));
    }

    private void ApplyPenalty(IContext ctx, ScoreResult score)
    {
        score.DeductPoints(50, "DTI Penalty: Debt to Income ratio exceeds 40%");
        ctx.Update(score);
    }
}
```

**Problemas Reales y Performance:**
*   **Problema:** Datos desactualizados. El buró de crédito puede tardar en responder.
*   **Performance:** Excelente. Las reglas son *stateless* entre diferentes solicitudes. Podés instanciar una `ISession` por request HTTP, insertar los 3 Facts, hacer `Fire()`, leer el `ScoreResult` y descartar la sesión. El Garbage Collector limpia rápido porque el grafo Rete (el `ISessionFactory`) es Singleton y no se reasigna.

**Alternativas y por qué NRules:**
*   *Alternativa:* Modelos de Machine Learning (XGBoost, Random Forest).
*   *Veredicto:* **NRules gana en Explicabilidad (Compliance).** Un modelo de ML es una caja negra. Si un auditor te pregunta por qué rechazaste a Juan Pérez, con ML tenés que usar técnicas complejas (SHAP values) que legalmente a veces no son aceptadas. Con NRules, el `AuditTrail` te da la respuesta exacta, determinista y auditable.

---

#### Caso 2: Motor Antifraude (Real-Time Fraud Engine)

**Contexto de Negocio:**
Procesador de pagos (tipo Stripe o MercadoPago). Necesitamos evaluar transacciones en milisegundos. Buscamos patrones de "Velocity" (muchas compras chicas en poco tiempo) y "Impossible Travel" (compras físicas en dos países distintos en menos de 2 horas).

**Modelo de Dominio:**
```csharp
public record PaymentEvent(string EventId, string CardId, decimal Amount, string CountryCode, DateTime Timestamp, bool IsCardPresent);

public class FraudDecision
{
    public string EventId { get; init; }
    public bool IsFraudulent { get; set; }
    public string Reason { get; set; }
}
```

**Reglas (Velocity Check con Ventanas de Tiempo):**

```csharp
public class HighVelocityFraudRule : Rule
{
    public override void Define()
    {
        PaymentEvent currentEvent = null;
        IEnumerable<PaymentEvent> historicalEvents = null;

        Name("Fraud_High_Velocity");
        Priority(100);

        When()
            .Match<PaymentEvent>(() => currentEvent)
            // Buscamos el historial de la MISMA tarjeta en las últimas 5 horas
            .Query(() => historicalEvents, q => q
                .Match<PaymentEvent>(
                    e => e.CardId == currentEvent.CardId,
                    e => e.EventId != currentEvent.EventId, // Excluir el actual
                    e => (currentEvent.Timestamp - e.Timestamp).TotalHours <= 5)
                .Collect()
                .Where(events => events.Count() >= 4)) // Si hay 4 o más previas (5 total)
            
            .Not<FraudDecision>(f => f.EventId == currentEvent.EventId);

        Then()
            .Do(ctx => FlagAsFraud(ctx, currentEvent, historicalEvents.Count()));
    }

    private void FlagAsFraud(IContext ctx, PaymentEvent currentEvent, int previousCount)
    {
        ctx.Insert(new FraudDecision 
        { 
            EventId = currentEvent.EventId, 
            IsFraudulent = true, 
            Reason = $"Velocity: {previousCount + 1} transactions in 5 hours." 
        });
    }
}
```

**Problemas Reales y Performance:**
*   **Problema (Memory Leak):** Si insertás cada `PaymentEvent` en una sesión de NRules de larga duración (Long-lived Session) y nunca hacés `Retract()`, la Memoria Alpha va a crecer hasta tirar un `OutOfMemoryException`.
*   **Solución Arquitectónica:** Tenés que implementar un mecanismo de **Eviction** (Desalojo). Un proceso en background que haga `session.Retract(oldEvent)` para eventos con más de 5 horas de antigüedad.
*   **Performance:** El nodo `.Collect()` es costoso. Si una tarjeta tiene 10,000 transacciones, el motor va a iterar sobre todas para armar el `IEnumerable`.

**Alternativas y por qué NRules:**
*   *Alternativa:* Apache Flink, Kafka Streams, o Redis (con Lua scripts).
*   *Veredicto:* **NRules PIERDE en escala masiva distribuida.** NRules vive en la memoria de un solo proceso .NET. Si tenés un clúster de 20 microservicios procesando pagos, la Working Memory de NRules no se comparte entre ellos. Para antifraude a escala global (millones de TPS), necesitás un motor de Stream Processing real (Flink). NRules es excelente si podés enrutar (Partition/Sharding) todo el tráfico de una misma `CardId` siempre al mismo nodo/pod de Kubernetes.

---

#### Caso 3: Evaluación de Riesgo (Insurance Underwriting)

**Contexto de Negocio:**
Seguros de vida. El sistema evalúa la historia clínica, ocupación y estilo de vida del solicitante. Existen "Knockout Rules" (condiciones que rechazan automáticamente la póliza, ej: Cáncer terminal) y reglas de "Rating Up" (aumentos de prima por riesgo, ej: Fumador + IMC > 30).

**Modelo de Dominio:**
```csharp
public record Applicant(string Id, int Age, decimal BMI, bool IsSmoker, string Occupation);
public record MedicalCondition(string ApplicantId, string DiagnosisCode, DateTime DiagnosedAt);

public class PolicyAssessment
{
    public string ApplicantId { get; init; }
    public bool IsDeclined { get; set; }
    public decimal PremiumMultiplier { get; set; } = 1.0m;
}
```

**Reglas (Knockout y Combinatoria):**

```csharp
public class KnockoutTerminalIllnessRule : Rule
{
    public override void Define()
    {
        Applicant app = null;
        PolicyAssessment assessment = null;

        Name("Risk_Knockout_Terminal");
        Priority(9999); // Máxima prioridad. Si esto salta, no evaluamos más.

        When()
            .Match<Applicant>(() => app)
            .Match<PolicyAssessment>(() => assessment, a => a.ApplicantId == app.Id, a => !a.IsDeclined)
            // Exists es ultra rápido. Al primer match, activa la regla y deja de buscar.
            .Exists<MedicalCondition>(
                m => m.ApplicantId == app.Id,
                m => m.DiagnosisCode == "C80.1"); // Código CIE-10 para tumor maligno

        Then()
            .Do(ctx => DeclinePolicy(ctx, assessment));
    }

    private void DeclinePolicy(IContext ctx, PolicyAssessment assessment)
    {
        assessment.IsDeclined = true;
        ctx.Update(assessment);
        // Podríamos usar ctx.Halt() para detener el motor, pero Update() 
        // invalidará las otras reglas si tienen la Guard Clause (!a.IsDeclined).
    }
}

public class SmokerBmiPenaltyRule : Rule
{
    public override void Define()
    {
        Applicant app = null;
        PolicyAssessment assessment = null;

        Name("Risk_Smoker_BMI_Penalty");
        Priority(500);

        When()
            .Match<Applicant>(() => app, a => a.IsSmoker, a => a.BMI > 30m)
            .Match<PolicyAssessment>(() => assessment, 
                a => a.ApplicantId == app.Id, 
                a => !a.IsDeclined, // Si la regla Knockout se ejecutó, esto es False.
                a => a.PremiumMultiplier == 1.0m); // Guard clause

        Then()
            .Do(ctx => ApplyMultiplier(ctx, assessment));
    }

    private void ApplyMultiplier(IContext ctx, PolicyAssessment assessment)
    {
        assessment.PremiumMultiplier += 0.50m; // +50% de prima
        ctx.Update(assessment);
    }
}
```

**Problemas Reales y Performance:**
*   **Problema:** Explosión combinatoria. Hay miles de códigos CIE-10. Escribir una regla por código es inmanejable.
*   **Solución:** Las reglas no deben contener datos duros (`"C80.1"`). Deben inyectarse diccionarios de datos o consultar servicios externos (veremos Inyección de Dependencias en el próximo módulo).
*   **Performance:** Ideal. La evaluación de riesgo es asíncrona y de baja frecuencia comparada con pagos. Rete brilla cruzando múltiples diagnósticos médicos complejos.

**Alternativas y por qué NRules:**
*   *Alternativa:* Árboles de decisión en código imperativo (`if/else` anidados) o motores como Drools (Java).
*   *Veredicto:* **NRules es el rey absoluto aquí para el ecosistema .NET.** Un árbol de decisión imperativo se vuelve espagueti inauditable a las 50 condiciones. NRules permite a los actuarios definir reglas aisladas.

---

#### Caso 4: Reglas Impositivas Dinámicas (Tax Engine)

**Contexto de Negocio:**
E-commerce B2B/B2C. Calcular impuestos (VAT, Sales Tax) es un infierno. Depende del origen, destino, tipo de producto (físico vs digital), y si el comprador está exento.

**Modelo de Dominio:**
```csharp
public record CartItem(string Sku, decimal Price, string ProductType);
public record Address(string Country, string State, string ZipCode);
public record TaxContext(Address Origin, Address Destination, bool IsB2B, string BuyerTaxId);

public class TaxLine
{
    public string Sku { get; init; }
    public string TaxName { get; init; }
    public decimal Rate { get; init; }
    public decimal CalculatedAmount => Rate * Price; // Simplificado
    public decimal Price { get; init; }
}
```

**Reglas (Resolución de Conflictos y Exenciones):**

```csharp
public class DigitalGoodsEuVatRule : Rule
{
    public override void Define()
    {
        CartItem item = null;
        TaxContext taxCtx = null;

        Name("Tax_EU_Digital_Goods_B2C");
        Priority(100);

        When()
            .Match<TaxContext>(() => taxCtx, 
                t => t.Destination.Country == "EU", // Simplificación didáctica
                t => !t.IsB2B) // Solo B2C
            .Match<CartItem>(() => item, i => i.ProductType == "Digital")
            .Not<TaxLine>(tl => tl.Sku == item.Sku); // Si no tiene impuesto aún

        Then()
            .Do(ctx => ApplyTax(ctx, item, 0.21m, "EU_VAT_DIGITAL"));
    }

    private void ApplyTax(IContext ctx, CartItem item, decimal rate, string name)
    {
        ctx.Insert(new TaxLine { Sku = item.Sku, Rate = rate, TaxName = name, Price = item.Price });
    }
}

public class B2bReverseChargeExemptionRule : Rule
{
    public override void Define()
    {
        TaxContext taxCtx = null;
        CartItem item = null;

        Name("Tax_B2B_Reverse_Charge");
        Priority(500); // Mayor prioridad que la regla B2C

        When()
            .Match<TaxContext>(() => taxCtx, t => t.IsB2B, t => !string.IsNullOrEmpty(t.BuyerTaxId))
            .Match<CartItem>(() => item)
            .Not<TaxLine>(tl => tl.Sku == item.Sku);

        Then()
            .Do(ctx => ApplyTax(ctx, item, 0.0m, "REVERSE_CHARGE_EXEMPT"));
    }
}
```

**Problemas Reales y Performance:**
*   **Problema:** La legislación cambia constantemente. Las reglas tienen **validez temporal** (ej: "Reducción de IVA por COVID del 01/03 al 31/12").
*   **Solución:** Tenés que agregar propiedades `ValidFrom` y `ValidTo` a tus reglas (usando metadatos o condiciones en el `When`) e inyectar un `TimeReference` como vimos en el Módulo 3.
*   **Performance:** Muy rápida, ejecución transaccional por carrito.

**Alternativas y por qué NRules:**
*   *Alternativa:* APIs de impuestos especializadas (Avalara, Vertex, Stripe Tax).
*   *Veredicto:* **NO USES NRules (ni nada propio) para impuestos a menos que sea estrictamente necesario.** El problema de los impuestos no es el motor de reglas, es el **mantenimiento del dominio legal**. Avalara tiene equipos de abogados actualizando tasas. Si usás NRules, tu equipo de desarrollo se convierte en responsable legal de mantener miles de tasas impositivas. Usá NRules solo si es un sistema interno muy acotado.

---

#### Caso 5: Sistema de Aprobación Multinivel (Workflow / Approvals)

**Contexto de Negocio:**
Órdenes de compra (Purchase Orders). Si es < $1000, se auto-aprueba. Si es > $1000, requiere aprobación del Manager. Si es > $10,000, requiere Manager + VP.

**Modelo de Dominio:**
```csharp
public class PurchaseOrder
{
    public string Id { get; init; }
    public decimal Amount { get; init; }
    public string Status { get; set; } = "Pending";
    public List<string> RequiredApprovals { get; } = new();
}
```

**Reglas (Transición de Estado):**

```csharp
public class RequireManagerApprovalRule : Rule
{
    public override void Define()
    {
        PurchaseOrder po = null;

        Name("Approval_Manager_Required");
        
        When()
            .Match<PurchaseOrder>(() => po, 
                p => p.Amount > 1000m,
                p => p.Status == "Pending",
                p => !p.RequiredApprovals.Contains("Manager"));

        Then()
            .Do(ctx => RequireApproval(ctx, po, "Manager"));
    }

    private void RequireApproval(IContext ctx, PurchaseOrder po, string role)
    {
        po.RequiredApprovals.Add(role);
        po.Status = "Awaiting_Approval";
        ctx.Update(po);
    }
}
```

**Problemas Reales y Performance:**
*   **El Gran Problema Arquitectónico:** NRules **NO** es un motor de Workflows de larga duración (Long-running processes). 
*   Si la orden pasa a `"Awaiting_Approval"`, el motor de reglas termina su ejecución (`Fire()` retorna). ¿Qué pasa cuando el Manager entra al sistema 3 días después y hace clic en "Aprobar"?
*   Tenés que volver a instanciar la sesión de NRules, recargar la `PurchaseOrder` desde la base de datos, insertar el Fact del "Evento de Aprobación del Manager", y volver a correr el motor para ver si ahora pasa a "Aprobado" o si falta el VP.

**Alternativas y por qué NRules:**
*   *Alternativa:* Motores de Workflow/Sagas (Temporal.io, Camunda, MassTransit State Machines).
*   *Veredicto:* **NRules es PÉSIMO para esto.** Usar un motor de inferencia Rete para modelar una Máquina de Estados Finita (FSM) que espera intervención humana es un anti-patrón. Los motores de Workflow manejan la persistencia del estado, timeouts y reintentos de forma nativa. NRules es puramente evaluativo en memoria.

---

#### Caso 6: Pricing Dinámico (Dynamic Pricing)

**Contexto de Negocio:**
Venta de pasajes aéreos. El precio base es $500. Si faltan menos de 7 días, sube 20%. Si el vuelo está 80% lleno, sube 30%. Si el usuario es VIP, tiene 10% de descuento sobre el precio final.

**Modelo de Dominio:**
```csharp
public record Flight(string FlightId, int DaysToDeparture, decimal CapacityFilledPercentage);
public record User(string UserId, bool IsVip);

public class PriceCalculation
{
    public string FlightId { get; init; }
    public decimal BasePrice { get; init; }
    public decimal Multiplier { get; set; } = 1.0m;
    public decimal Discount { get; set; } = 0.0m;
    public decimal FinalPrice => (BasePrice * Multiplier) - Discount;
}
```

**Reglas (Manejo estricto de Salience/Prioridad):**

En Pricing, el orden de ejecución es vital. No podés aplicar el descuento VIP antes de calcular el multiplicador de demanda.

```csharp
public class HighDemandSurgeRule : Rule
{
    public override void Define()
    {
        Flight flight = null;
        PriceCalculation price = null;

        Name("Pricing_High_Demand_Surge");
        Priority(800); // Se ejecuta ANTES de los descuentos

        When()
            .Match<Flight>(() => flight, f => f.CapacityFilledPercentage > 0.80m)
            .Match<PriceCalculation>(() => price, 
                p => p.FlightId == flight.FlightId,
                p => p.Multiplier == 1.0m); // Guard clause

        Then()
            .Do(ctx => ApplySurge(ctx, price, 0.30m));
    }

    private void ApplySurge(IContext ctx, PriceCalculation price, decimal surge)
    {
        price.Multiplier += surge;
        ctx.Update(price);
    }
}

public class VipDiscountRule : Rule
{
    public override void Define()
    {
        User user = null;
        PriceCalculation price = null;

        Name("Pricing_VIP_Discount");
        Priority(200); // Se ejecuta DESPUÉS de calcular los multiplicadores (Priority menor)

        When()
            .Match<User>(() => user, u => u.IsVip)
            .Match<PriceCalculation>(() => price, p => p.Discount == 0.0m);

        Then()
            .Do(ctx => ApplyDiscount(ctx, price));
    }

    private void ApplyDiscount(IContext ctx, PriceCalculation price)
    {
        // Calcula el 10% sobre el precio ya multiplicado
        price.Discount = (price.BasePrice * price.Multiplier) * 0.10m; 
        ctx.Update(price);
    }
}
```

**Problemas Reales y Performance:**
*   **Problema:** "Rule Overlapping" (Superposición de reglas). ¿Qué pasa si hay una regla de "Surge por Alta Demanda" y otra de "Surge por Último Minuto"? ¿Se suman los multiplicadores o gana el mayor?
*   **Solución:** NRules no tiene "Activation Groups" nativos como Drools (donde solo se ejecuta una regla de un grupo). Tenés que modelarlo en el dominio. En el `Then()`, en lugar de sumar directamente, insertás un Fact `SurgeProposal(Amount)` y tenés una regla final que hace un `.Query().Collect().Max()` para elegir el mayor.
*   **Performance:** Excelente para cálculos por request.

**Alternativas y por qué NRules:**
*   *Alternativa:* Algoritmos imperativos o Machine Learning (Reinforcement Learning).
*   *Veredicto:* **Empate Técnico.** Si el pricing se basa en reglas de negocio dictadas por marketing ("Los martes damos 10% off"), NRules es perfecto. Si el pricing busca maximizar el *Yield* (ganancia) basándose en elasticidad de precio y comportamiento del usuario en tiempo real, un modelo de ML es infinitamente superior, y NRules solo debería usarse como un "Guardrail" (barrera de seguridad) para asegurar que el ML no ponga un precio negativo o absurdo.

---

Estos 6 casos representan el espectro completo de lo que te vas a encontrar como Arquitecto. La clave no es saber escribir la sintaxis de NRules, sino saber **modelar el estado (Facts)** para que el algoritmo Rete trabaje a tu favor y no en tu contra.

---

### Módulo 5: Casos Borde, Errores Reales y Autopsias de Producción

Este es el módulo donde la teoría se estrella contra la realidad. He visto sistemas de producción caerse en Black Friday porque un desarrollador olvidó una cláusula de guarda en un motor de reglas. NRules es una herramienta de precisión; si la usás mal, no te va a dar un error de compilación, te va a dar un *StackOverflowException* o va a consumir el 100% de la CPU en un bucle silencioso.

Vamos a analizar las autopsias de los errores más comunes y destructivos, mostrando el código que causa el desastre y la solución arquitectónica correcta.

---

#### 1. El Bucle Infinito (The Infinite Fire Loop)

Este es el error número uno en cualquier motor de encadenamiento hacia adelante (Forward Chaining). Ocurre cuando una regla muta un Fact, llama a `Update()`, y esa mutación no invalida la condición que hizo que la regla se disparara en primer lugar.

**El Escenario:**
Queremos aplicar un recargo de $10 a todas las órdenes internacionales.

**Código Incorrecto (El Desastre):**
```csharp
public class InternationalSurchargeRule_BAD : Rule
{
    public override void Define()
    {
        Order order = null;

        When()
            .Match<Order>(() => order, o => o.Country != "US");

        Then()
            .Do(ctx => ApplySurcharge(ctx, order));
    }

    private void ApplySurcharge(IContext ctx, Order order)
    {
        order.Total += 10m; // Mutamos el Fact
        ctx.Update(order);  // Le avisamos al motor
    }
}
```
**Autopsia:**
1. Entra una orden de Canadá con Total = $100.
2. La regla hace match (`Country != "US"` es True).
3. Se ejecuta el `Then`. El Total pasa a $110.
4. Se llama a `Update()`. El motor reevalúa el Fact `Order`.
5. ¿El país sigue siendo distinto de "US"? Sí.
6. La regla vuelve a hacer match. Se encola en la Agenda.
7. El Total pasa a $120.
8. Bucle infinito hasta que el proceso muere por `OutOfMemoryException` o timeout.

**Código Corregido (La Solución):**
Tenés que introducir una **Cláusula de Guarda (Guard Clause)** en el `When` que se vuelva `False` después de la mutación.

```csharp
public class InternationalSurchargeRule_GOOD : Rule
{
    public override void Define()
    {
        Order order = null;

        When()
            .Match<Order>(() => order, 
                o => o.Country != "US",
                o => !o.HasInternationalSurcharge); // GUARD CLAUSE

        Then()
            .Do(ctx => ApplySurcharge(ctx, order));
    }

    private void ApplySurcharge(IContext ctx, Order order)
    {
        order.Total += 10m;
        order.HasInternationalSurcharge = true; // Cambiamos el estado de la guarda
        ctx.Update(order);
    }
}
```

---

#### 2. Modificación Silenciosa de Facts (El Fantasma de la Memoria)

NRules no intercepta las asignaciones de propiedades en C# (no usa proxies como Entity Framework). Si modificás un Fact y no llamás a `Update()`, la Working Memory queda desincronizada con el heap de .NET.

**El Escenario:**
Queremos marcar a un usuario como "Premium" si compra más de $500, y luego aplicar un descuento del 20% a los usuarios Premium.

**Código Incorrecto (El Fantasma):**
```csharp
public class UpgradeToPremiumRule_BAD : Rule
{
    public override void Define()
    {
        User user = null;
        Order order = null;

        When()
            .Match<User>(() => user, u => !u.IsPremium)
            .Match<Order>(() => order, o => o.UserId == user.Id, o => o.Total > 500m);

        Then()
            .Do(ctx => 
            {
                user.IsPremium = true; // Mutación en memoria de C#
                // ERROR: Falta ctx.Update(user);
            });
    }
}

public class PremiumDiscountRule : Rule
{
    public override void Define()
    {
        User user = null;
        Order order = null;

        When()
            .Match<User>(() => user, u => u.IsPremium) // Esto nunca hará match
            .Match<Order>(() => order, o => o.UserId == user.Id);

        Then()
            .Do(ctx => order.Total *= 0.80m);
    }
}
```
**Autopsia:**
El usuario se marca como Premium en el objeto C#, pero la red Rete de NRules sigue pensando que `IsPremium` es `False` porque nadie le avisó del cambio. La regla `PremiumDiscountRule` jamás se dispara.

**Solución:**
Siempre, **absolutamente siempre**, llamá a `ctx.Update(fact)` si mutaste una propiedad que es evaluada en el `When` de cualquier otra regla.

---

#### 3. Problemas de Mutabilidad y Colecciones (El Asesino Silencioso)

Este es un caso borde avanzado. Ocurre cuando modificás una colección interna de un Fact sin reasignar la referencia de la colección.

**El Escenario:**
Agregar un item de regalo a una orden si el total supera $1000.

**Código Incorrecto:**
```csharp
public class FreeGiftRule_BAD : Rule
{
    public override void Define()
    {
        Order order = null;

        When()
            .Match<Order>(() => order, 
                o => o.Total > 1000m,
                o => !o.Items.Any(i => i.Sku == "FREE_GIFT")); // Guard Clause

        Then()
            .Do(ctx => 
            {
                order.Items.Add(new Item { Sku = "FREE_GIFT" });
                ctx.Update(order); // Llamamos a Update, parece correcto, ¿no?
            });
    }
}
```
**Autopsia:**
Dependiendo de cómo NRules optimice la red Rete internamente, modificar una colección in-place (`.Add()`) puede causar comportamientos erráticos si otras reglas están haciendo un `.Query().Collect()` sobre esos items. NRules evalúa la igualdad de referencias. Si la referencia de la lista `Items` es la misma, el motor podría no detectar el cambio profundo en algunos nodos Beta complejos.

**Solución (Inmutabilidad o Facts Separados):**
La mejor práctica en motores de reglas es tratar las colecciones como Facts independientes, no como propiedades anidadas.

```csharp
// En lugar de order.Items.Add(), insertamos un nuevo Fact
public class FreeGiftRule_GOOD : Rule
{
    public override void Define()
    {
        Order order = null;

        When()
            .Match<Order>(() => order, o => o.Total > 1000m)
            .Not<Item>(i => i.OrderId == order.Id && i.Sku == "FREE_GIFT"); // Guard Clause limpia

        Then()
            .Do(ctx => ctx.Insert(new Item { OrderId = order.Id, Sku = "FREE_GIFT" }));
    }
}
```

---

#### 4. Problemas de Concurrencia (El Colapso de Producción)

NRules está diseñado con una separación estricta entre la compilación (Thread-Safe) y la ejecución (No Thread-Safe).

**El Escenario:**
Un servicio web (ASP.NET Core) recibe miles de requests por segundo para evaluar carritos de compras.

**Código Incorrecto (El Colapso):**
```csharp
// Registrado como Singleton en DI
public class DiscountService_BAD
{
    private readonly ISession _session; // ERROR FATAL

    public DiscountService_BAD(ISessionFactory factory)
    {
        // Crear una sola sesión compartida para toda la app
        _session = factory.CreateSession(); 
    }

    public void ApplyDiscounts(Order order)
    {
        _session.Insert(order);
        _session.Fire();
        _session.Retract(order); // Intento ingenuo de limpiar
    }
}
```
**Autopsia:**
La `ISession` mantiene el estado (Working Memory y Agenda). Si dos requests HTTP (dos hilos distintos) llaman a `ApplyDiscounts` al mismo tiempo, el Hilo A inserta la Orden A, el Hilo B inserta la Orden B. El Hilo A llama a `Fire()` y evalúa las reglas para **ambas** órdenes mezcladas. Peor aún, las colecciones internas de la sesión no son concurrentes, lo que lanzará excepciones de tipo `InvalidOperationException: Collection was modified`.

**Solución:**
La `ISessionFactory` es Singleton. La `ISession` es Transient/Scoped.

```csharp
public class DiscountService_GOOD
{
    private readonly ISessionFactory _factory;

    public DiscountService_GOOD(ISessionFactory factory)
    {
        _factory = factory; // Singleton
    }

    public void ApplyDiscounts(Order order)
    {
        // Cada request crea su propia sesión aislada y limpia
        ISession session = _factory.CreateSession(); 
        session.Insert(order);
        session.Fire();
        // El Garbage Collector destruye la sesión al salir del método
    }
}
```

---

#### 5. Explosión Combinatoria (El Agujero Negro de CPU)

El algoritmo Rete es rápido porque hace joins (cruces) en memoria. Pero si cruzás Facts sin filtros restrictivos (Alpha Nodes), creás un producto cartesiano masivo.

**El Escenario:**
Queremos encontrar si un usuario compró el mismo producto que su amigo.

**Código Incorrecto (El Agujero Negro):**
```csharp
public class FriendPurchaseRule_BAD : Rule
{
    public override void Define()
    {
        User userA = null;
        User userB = null;
        Order orderA = null;
        Order orderB = null;

        When()
            .Match<User>(() => userA)
            .Match<User>(() => userB, b => b.Id != userA.Id) // Join: Todos contra todos
            .Match<Order>(() => orderA, a => a.UserId == userA.Id)
            .Match<Order>(() => orderB, b => b.UserId == userB.Id, b => b.ProductId == orderA.ProductId);
        // ...
    }
}
```
**Autopsia:**
Si insertás 1,000 Usuarios y 10,000 Órdenes, el primer Join (`userA` con `userB`) genera 1,000,000 de tuplas en la Memoria Beta. El siguiente Join cruza ese millón de tuplas con las 10,000 órdenes. El motor intentará evaluar miles de millones de combinaciones. La CPU llegará al 100% y la memoria RAM se agotará en segundos.

**Solución:**
1. **Orden de los Joins:** Poné siempre los Facts más restrictivos primero.
2. **Evitá Joins abiertos:** Nunca cruces `User` con `User` sin un filtro fuerte (ej. `b => b.City == userA.City`).
3. **Rediseño de Dominio:** En lugar de cruzar todo en memoria, pre-procesá los datos antes de insertarlos en NRules, o usá un grafo de base de datos (Neo4j) para relaciones complejas, y usá NRules solo para la decisión final.

---

#### 6. Conflictos entre Reglas y Orden de Ejecución Inesperado

Cuando dos reglas hacen match con los mismos Facts al mismo tiempo, ambas van a la Agenda. ¿Cuál se ejecuta primero? Si no lo definís, el motor decide, y tu lógica se vuelve no determinista.

**El Escenario:**
Regla A: Si el cliente es VIP, aplicar 20% de descuento.
Regla B: Si el total es > $1000, aplicar envío gratis.

**Código Incorrecto:**
```csharp
// Regla A no tiene Priority definida (por defecto es 0)
// Regla B no tiene Priority definida (por defecto es 0)
```
**Autopsia:**
Entra un VIP con una orden de $1100. Ambas reglas van a la Agenda.
Si el motor ejecuta la Regla A primero, el total baja a $880. Cuando el motor intenta ejecutar la Regla B, la condición `Total > 1000` ya no se cumple (asumiendo que la Regla A hizo un `Update`). El cliente pierde el envío gratis.
Si el motor ejecuta la Regla B primero, el cliente obtiene envío gratis y luego el descuento.
El resultado depende del orden interno de la Agenda, que puede variar entre versiones de NRules o por el orden de compilación.

**Solución:**
Usá `Priority()` (Salience) explícitamente para orquestar el flujo, o mejor aún, diseñá reglas mutuamente excluyentes.

```csharp
public class VipDiscountRule : Rule
{
    public override void Define()
    {
        // ...
        Priority(100); // Se ejecuta después
        // ...
    }
}

public class FreeShippingRule : Rule
{
    public override void Define()
    {
        // ...
        Priority(200); // Se ejecuta primero, asegurando el envío gratis basado en el subtotal original
        // ...
    }
}
```

---

#### 7. La Regla que Nunca se Dispara (Opaque Expressions)

NRules desarma tus lambdas en *Expression Trees*. Si usás métodos que NRules no puede desarmar, la evaluación falla silenciosamente o se degrada.

**El Escenario:**
Validar si un código postal es válido usando un servicio externo.

**Código Incorrecto:**
```csharp
public class ZipCodeRule_BAD : Rule
{
    public override void Define()
    {
        Address address = null;

        When()
            .Match<Address>(() => address, 
                a => ExternalZipCodeValidator.IsValid(a.ZipCode)); // ERROR ARQUITECTÓNICO
        // ...
    }
}
```
**Autopsia:**
NRules no puede mirar dentro de `ExternalZipCodeValidator.IsValid()`. Lo trata como una expresión opaca. Si el servicio externo hace una llamada HTTP o a base de datos, vas a bloquear el hilo de evaluación de Rete (que debería ser CPU-bound puro). Además, si el resultado del servicio cambia, NRules no tiene forma de saberlo porque no es un Fact.

**Solución:**
Las reglas deben ser puras. La validación externa debe ocurrir **antes** de insertar los Facts, o el resultado de la validación debe inyectarse como un Fact.

```csharp
// Solución: Pre-procesamiento
var isValid = ExternalZipCodeValidator.IsValid(address.ZipCode);
session.Insert(new ZipCodeValidationFact(address.ZipCode, isValid));
session.Insert(address);

// En la regla:
When()
    .Match<Address>(() => address)
    .Match<ZipCodeValidationFact>(() => validation, 
        v => v.ZipCode == address.ZipCode, 
        v => v.IsValid);
```

Dominar estos casos borde es lo que separa a un desarrollador que "hace andar" NRules de un Arquitecto que construye sistemas resilientes.

---

### Módulo 6: Performance, Escalabilidad y Arquitectura de Alta Carga

Llegamos al terreno de los arquitectos de sistemas distribuidos. NRules es un motor en memoria (In-Memory Rule Engine). No es una base de datos, no es un clúster de Hadoop y no hace magia negra. Su rendimiento está dictado estrictamente por las leyes de la física de la CPU, la asignación de memoria en el Heap de .NET y el Garbage Collector (GC).

En este módulo, vamos a analizar cómo escalar NRules desde un microservicio que procesa 10 requests por segundo hasta un pipeline de batch que evalúa millones de transacciones nocturnas.

---

#### 1. El Benchmark Mental de Arquitectura (¿Qué tan rápido es NRules?)

Antes de optimizar, necesitás un modelo mental de los tiempos de ejecución. Estos números son aproximados para un servidor estándar (ej. AWS c5.large) ejecutando .NET 8, pero te dan la magnitud del problema:

*   **Compilación de Reglas (`factory.Compile()`):** Lento. Cientos de milisegundos a varios segundos dependiendo de la cantidad de reglas. **Debe hacerse 1 sola vez al inicio de la app.**
*   **Creación de Sesión (`factory.CreateSession()`):** Ultra rápido. ~1 a 5 microsegundos. Es solo asignación de memoria para las colecciones internas.
*   **Inserción de un Fact Simple (Sin Joins complejos):** Muy rápido. ~10 a 50 microsegundos.
*   **Inserción de un Fact que dispara Joins pesados (Beta Nodes):** Variable. Desde 100 microsegundos hasta milisegundos si hay explosión combinatoria.
*   **Ejecución de Reglas (`session.Fire()`):** Depende enteramente de tu código C# en el bloque `Then()`. El overhead del motor para sacar la activación de la Agenda y llamar al delegado es de nanosegundos.

**Regla de Oro del Benchmark:**
Si tu request HTTP tarda 200ms, NRules probablemente esté consumiendo 2ms. Los otros 198ms son tu base de datos, tu serialización JSON o tus llamadas HTTP externas. **NRules rara vez es el cuello de botella de CPU, pero suele ser el cuello de botella de Memoria RAM si se usa mal.**

---

#### 2. Impacto de Miles de Facts y la Explosión del Grafo Rete

El algoritmo Rete intercambia consumo de memoria por velocidad de CPU. Cada vez que un Fact pasa un nodo Alpha, se guarda en la **Alpha Memory**. Cada vez que dos Facts hacen match en un nodo Beta (Join), la tupla resultante se guarda en la **Beta Memory**.

**El Escenario del Desastre (Explosión Combinatoria):**
Imaginá que tenés una regla que busca fraude cruzando transacciones de la misma ciudad.

```csharp
// ANTI-PATRÓN DE PERFORMANCE
When()
    .Match<Transaction>(() => txA)
    .Match<Transaction>(() => txB, 
        b => b.Id != txA.Id, 
        b => b.City == txA.City); // Join abierto
```

Si insertás 10,000 transacciones en la misma sesión, el motor intentará cruzar cada transacción con las otras 9,999.
$10,000 \times 10,000 = 100,000,000$ de evaluaciones en el nodo Beta.
Si 50,000 de esas combinaciones coinciden en la ciudad, se crearán 50,000 tuplas en la Memoria Beta. El consumo de RAM se dispara, el Garbage Collector entra en pánico (Gen 2 Collection) y tu hilo se congela.

**Cómo Optimizar Reglas para Evitar la Explosión:**

1.  **Filtros Alpha Primero (La Regla del Embudo):**
    NRules evalúa las condiciones en el orden en que las escribís (generalmente). Poné siempre las condiciones más restrictivas (las que filtran más datos) lo más arriba posible en el `Match`.

    ```csharp
    // MAL: El Join se evalúa para TODAS las transacciones antes de filtrar por monto.
    When()
        .Match<User>(() => user)
        .Match<Transaction>(() => tx, t => t.UserId == user.Id, t => t.Amount > 10000m);

    // BIEN: Filtramos por monto (Alpha Node) ANTES de intentar el Join.
    When()
        .Match<User>(() => user)
        .Match<Transaction>(() => tx, t => t.Amount > 10000m, t => t.UserId == user.Id);
    ```

2.  **Particionamiento de Datos (Sharding Lógico):**
    No insertes 10,000 transacciones de 1,000 usuarios distintos en la misma sesión si las reglas solo cruzan datos del *mismo* usuario.
    Creá una sesión nueva por cada usuario, insertá solo sus transacciones, hacé `Fire()`, y descartá la sesión. Esto mantiene el grafo Rete diminuto y rápido.

---

#### 3. Batch Processing vs. Tiempo Real (Real-Time)

La arquitectura cambia radicalmente dependiendo de si estás evaluando un carrito de compras (Tiempo Real) o calculando comisiones mensuales para 50,000 vendedores (Batch).

**A. Arquitectura en Tiempo Real (Microservicios / APIs):**
*   **Patrón:** Sesiones Efímeras (Short-lived Sessions).
*   **Flujo:**
    1. Request HTTP entra al controlador.
    2. Cargar datos estrictamente necesarios de la BD (Ej: `User`, `Cart`, `Product`).
    3. `var session = _factory.CreateSession();`
    4. `session.InsertAll(facts);`
    5. `session.Fire();`
    6. Guardar resultados en BD.
    7. Retornar HTTP 200.
*   **Escalabilidad:** Escala horizontalmente de forma infinita. Podés levantar 100 pods en Kubernetes. Como la sesión vive solo durante el request y no hay estado compartido entre requests, NRules no requiere locks distribuidos ni Redis.

**B. Arquitectura Batch (Procesamiento Masivo Nocturno):**
*   **Patrón:** Sesiones de Larga Duración (Long-lived Sessions) con Eviction, o Particionamiento en Memoria.
*   **El Problema:** Si cargás 1 millón de registros de la BD y hacés `session.Insert()` en un `foreach`, te vas a quedar sin RAM (Out Of Memory).
*   **La Solución (Paginación y Retract):**
    Tenés que procesar en chunks (lotes) y limpiar la memoria de trabajo explícitamente.

    ```csharp
    // Arquitectura Batch Correcta
    var session = _factory.CreateSession();
    
    // Insertamos Facts estáticos (Reglas de negocio, configuraciones globales)
    session.InsertAll(globalConfigurations);

    foreach (var batch in database.GetUsersInBatches(1000))
    {
        var userFacts = new List<object>();
        
        foreach (var user in batch)
        {
            userFacts.Add(user);
            userFacts.AddRange(user.Transactions);
        }

        // Insertamos el lote actual
        session.InsertAll(userFacts);
        session.Fire();

        // VITAL: Limpiamos la memoria de trabajo para el próximo lote
        // RetractAll elimina los facts de la red Rete y libera la RAM
        foreach (var fact in userFacts)
        {
            session.Retract(fact);
        }
    }
    ```

---

#### 4. Caching de Reglas y Compilación Dinámica

En sistemas empresariales, las reglas cambian. El equipo de negocio modifica un descuento en un portal web y espera que el sistema lo aplique inmediatamente sin reiniciar el microservicio.

**El Problema:**
Llamar a `factory.Compile()` toma tiempo y bloquea hilos. No podés recompilar la red Rete en cada request HTTP.

**La Solución (Hot-Reloading de Reglas):**
Tenés que implementar un patrón de **Doble Buffer (Double Buffering)** o un `IOptionsMonitor` personalizado para intercambiar el `ISessionFactory` atómicamente.

```csharp
public class RuleEngineManager
{
    // Volatile asegura que los hilos vean la referencia más reciente
    private volatile ISessionFactory _currentFactory;
    private readonly IRuleRepository _repository;

    public RuleEngineManager(IRuleRepository repository)
    {
        _repository = repository;
        ReloadRules(); // Carga inicial
    }

    public ISession CreateSession()
    {
        // Retorna una sesión usando el factory actual.
        // Si el factory se está recompilando en background, 
        // los requests actuales siguen usando el viejo (Zero Downtime).
        return _currentFactory.CreateSession();
    }

    // Este método se llama desde un Webhook o un Background Worker 
    // cuando negocio avisa que las reglas cambiaron en la BD.
    public void ReloadRules()
    {
        var newRules = _repository.GetRules(); // Carga desde BD o DSL
        var compiler = new RuleCompiler();
        
        // Compilación pesada (ocurre en background, no bloquea requests)
        var newFactory = compiler.Compile(newRules); 
        
        // Intercambio atómico del puntero. 
        // Los nuevos requests usarán newFactory.
        // Los requests en vuelo terminarán usando el factory viejo.
        _currentFactory = newFactory; 
    }
}
```

---

#### 5. Uso en Microservicios y Arquitecturas Event-Driven

NRules encaja perfectamente en arquitecturas basadas en eventos (Kafka, RabbitMQ, Azure Service Bus).

**Patrón: El Microservicio de Decisión (Decision Node)**
En lugar de tener la lógica de reglas acoplada a tu API REST, creás un microservicio dedicado (ej. `FraudEvaluationService`) que escucha eventos.

1.  El `PaymentService` publica un evento `PaymentRequestedEvent` en Kafka.
2.  El `FraudEvaluationService` consume el evento.
3.  El consumidor hidrata los Facts necesarios (busca el historial del usuario en Redis/BD).
4.  Instancia una sesión de NRules, inserta los Facts y hace `Fire()`.
5.  Si NRules genera un Fact `FraudDetected`, el bloque `Then()` de la regla publica un nuevo evento `PaymentRejectedEvent` de vuelta a Kafka.

**Ventajas de este enfoque:**
*   **Aislamiento de CPU:** La compilación y ejecución pesada de Rete no afecta la latencia de tu API pública.
*   **Escalabilidad Independiente:** Si hay un pico de pagos, podés escalar solo los pods del `FraudEvaluationService`.
*   **Pureza del Dominio:** Las reglas no saben nada de HTTP ni de bases de datos, solo reaccionan a eventos de dominio (Facts) y producen nuevos eventos.

---

#### 6. Resumen de Optimización Extrema (Checklist del Arquitecto)

Si tu motor de reglas está lento en producción, revisá esta lista en orden:

1.  **¿Estás recompilando en cada request?** Asegurate de que `ISessionFactory` sea Singleton.
2.  **¿Estás insertando demasiados Facts inútiles?** Si una regla evalúa `Order`, no insertes el `User` si ninguna regla lo necesita. Filtrá en la base de datos antes de insertar en NRules.
3.  **¿Tenés Joins abiertos?** Revisá todos tus nodos `.Match()`. Si cruzás dos tipos de Facts sin una condición de igualdad estricta (`a.Id == b.Id`), estás generando un producto cartesiano.
4.  **¿Estás usando `DateTime.Now` en el `When`?** Inyectá el tiempo como un Fact estático para evitar reevaluaciones impredecibles y permitir el testing determinista.
5.  **¿Estás llamando a I/O (BD, APIs) dentro del `When`?** Prohibido. El `When` debe ser CPU-bound puro sobre memoria local.
6.  **¿Estás olvidando hacer `Retract` en sesiones largas?** Si usás una sesión compartida (ej. un Actor en Orleans o un Singleton stateful), tenés que limpiar los Facts viejos o tendrás un Memory Leak.

Con este conocimiento, podés diseñar sistemas que procesen miles de transacciones por segundo usando NRules sin transpirar.

Indicame cómo querés continuar. El siguiente paso lógico es el **Módulo 7: Inyección de Dependencias y Servicios Externos** (Cómo inyectar `ILogger`, repositorios de EF Core o clientes HTTP dentro del bloque `Then` de las reglas usando el contenedor de DI de .NET Core).

---

### Módulo 7: Testing Profesional de Reglas y Prevención de Regresiones

Escribir reglas de negocio es fácil. Asegurarse de que la Regla #402 no rompa silenciosamente la Regla #15 cuando ambas se ejecutan juntas en producción es el verdadero desafío de la ingeniería de software. 

En un sistema tradicional imperativo, testeás métodos. En un motor de reglas declarativo, testeás **comportamientos emergentes**. No podés simplemente instanciar la clase de la regla y llamar a un método; tenés que levantar el motor, insertar el estado del mundo (Facts) y afirmar (Assert) que el estado final es el correcto.

En este capítulo, vamos a construir una suite de pruebas robusta usando **xUnit** y **FluentAssertions**.

---

#### 1. La Pirámide de Testing en NRules

No podés testear todo a nivel de integración. Necesitás una estrategia en tres capas:

1.  **Unit Testing (Aislamiento de Regla Única):** Verificás que una regla específica haga match con los Facts correctos y ejecute su acción esperada. Ignorás el resto del sistema.
2.  **Integration Testing (Interacción de Reglas):** Verificás que un conjunto de reglas (ej. todas las reglas de Pricing) interactúen correctamente, respeten las prioridades (Salience) y no generen bucles infinitos.
3.  **Regression Testing (El Sistema Completo):** Pruebas de caja negra. Metés un JSON gigante con el estado de un cliente real de producción y verificás que el resultado final del motor sea exactamente el mismo que en la versión anterior del software.

---

#### 2. Setup del Entorno de Testing (El Fixture)

Para no repetir código de inicialización del motor en cada test, creamos un *Fixture* base.

```csharp
using NRules;
using NRules.Fluent;
using System.Collections.Generic;

public abstract class RuleTestFixture
{
    protected ISession Session { get; private set; }
    private readonly RuleRepository _repository;

    protected RuleTestFixture()
    {
        _repository = new RuleRepository();
    }

    // Cargamos solo las reglas que queremos testear
    protected void LoadRules(params Type[] ruleTypes)
    {
        foreach (var type in ruleTypes)
        {
            _repository.Load(x => x.From(type));
        }
    }

    // Compilamos el motor y creamos una sesión limpia para el test
    protected void CompileAndCreateSession()
    {
        var compiler = new RuleCompiler();
        var factory = compiler.Compile(_repository.GetRules());
        Session = factory.CreateSession();
    }

    // Helper para insertar múltiples Facts
    protected void InsertFacts(params object[] facts)
    {
        Session.InsertAll(facts);
    }
}
```

---

#### 3. Unit Testing: Aislamiento de una Regla Única

Vamos a testear la regla `GoldMemberVolumeDiscountRule` del Módulo 2.

**Objetivo:** Verificar que aplica el 10% de descuento SOLO si el cliente es Gold y la orden supera los $1000.

```csharp
using Xunit;
using FluentAssertions;
using System.Linq;

public class GoldMemberVolumeDiscountRuleTests : RuleTestFixture
{
    public GoldMemberVolumeDiscountRuleTests()
    {
        // 1. Arrange: Cargamos SOLO esta regla. Aislamos el comportamiento.
        LoadRules(typeof(GoldMemberVolumeDiscountRule));
        CompileAndCreateSession();
    }

    [Fact]
    public void Should_Apply_Discount_When_Customer_Is_Gold_And_Order_Over_1000()
    {
        // Arrange: Creamos los Facts
        var customer = new Customer("C1", "John", CustomerTier.Gold, DateTime.UtcNow);
        var order = new Order { Id = "O1", CustomerId = "C1" };
        order.Items.Add(new OrderItem { UnitPrice = 1200m, Quantity = 1 }); // SubTotal = 1200

        InsertFacts(customer, order);

        // Act: Disparamos el motor
        int firedRules = Session.Fire();

        // Assert
        firedRules.Should().Be(1, "La regla debió dispararse exactamente una vez");
        order.Discounts.Should().ContainSingle(d => d.Reason == "GOLD_VOLUME");
        order.TotalDiscount.Should().Be(120m); // 10% de 1200
    }

    [Fact]
    public void Should_NOT_Apply_Discount_When_Customer_Is_Silver()
    {
        var customer = new Customer("C1", "John", CustomerTier.Silver, DateTime.UtcNow);
        var order = new Order { Id = "O1", CustomerId = "C1" };
        order.Items.Add(new OrderItem { UnitPrice = 1200m, Quantity = 1 });

        InsertFacts(customer, order);

        int firedRules = Session.Fire();

        firedRules.Should().Be(0, "La regla no debe dispararse para clientes Silver");
        order.Discounts.Should().BeEmpty();
    }

    [Fact]
    public void Should_NOT_Apply_Discount_When_Order_Under_1000()
    {
        var customer = new Customer("C1", "John", CustomerTier.Gold, DateTime.UtcNow);
        var order = new Order { Id = "O1", CustomerId = "C1" };
        order.Items.Add(new OrderItem { UnitPrice = 900m, Quantity = 1 }); // SubTotal = 900

        InsertFacts(customer, order);

        int firedRules = Session.Fire();

        firedRules.Should().Be(0, "La regla no debe dispararse si el monto es menor a 1000");
    }
}
```

---

#### 4. Integration Testing: Interacción y Conflictos (Salience)

Acá es donde las cosas se ponen serias. ¿Qué pasa si tenemos dos reglas que aplican descuentos? El orden importa.

Supongamos que tenemos:
1.  `GoldMemberVolumeDiscountRule` (Prioridad 100): 10% de descuento.
2.  `ElectronicsCrossPromoRule` (Prioridad 50): $50 fijos si hay electrónica y el total final > $500.

**Objetivo:** Verificar que el descuento porcentual se calcula *antes* del descuento fijo, y que ambos se aplican correctamente sin pisarse.

```csharp
public class PricingIntegrationTests : RuleTestFixture
{
    public PricingIntegrationTests()
    {
        // Cargamos el conjunto completo de reglas de Pricing
        LoadRules(
            typeof(GoldMemberVolumeDiscountRule),
            typeof(ElectronicsCrossPromoRule)
        );
        CompileAndCreateSession();
    }

    [Fact]
    public void Should_Apply_Both_Discounts_In_Correct_Order()
    {
        // Arrange
        var customer = new Customer("C1", "John", CustomerTier.Gold, DateTime.UtcNow);
        var order = new Order { Id = "O1", CustomerId = "C1" };
        
        // SubTotal = 1000. Cumple ambas reglas.
        order.Items.Add(new OrderItem { Category = "Electronics", UnitPrice = 1000m, Quantity = 1 }); 

        InsertFacts(customer, order);

        // Act
        int firedRules = Session.Fire();

        // Assert
        firedRules.Should().Be(2, "Ambas reglas debieron dispararse");
        
        // Verificamos el orden de aplicación leyendo la lista de descuentos
        order.Discounts[0].Reason.Should().Be("GOLD_VOLUME", "El descuento porcentual tiene mayor prioridad");
        order.Discounts[0].Amount.Should().Be(100m); // 10% de 1000

        order.Discounts[1].Reason.Should().Be("ELEC_PROMO", "El descuento fijo se aplica después");
        order.Discounts[1].Amount.Should().Be(50m);

        order.FinalTotal.Should().Be(850m); // 1000 - 100 - 50
    }
}
```

---

#### 5. Testing de Forward Chaining y Mutación de Estado

Cuando una regla inserta un nuevo Fact que dispara otra regla, tenés que testear la cadena completa.

**Escenario (Del Módulo 4):**
1.  `HighVelocityFraudRule` detecta fraude e inserta un `FraudDecision`.
2.  `BlockUserRule` escucha el `FraudDecision` y muta el `UserProfile` a bloqueado.

```csharp
public class FraudChainIntegrationTests : RuleTestFixture
{
    public FraudChainIntegrationTests()
    {
        LoadRules(typeof(HighVelocityFraudRule), typeof(BlockUserRule));
        CompileAndCreateSession();
    }

    [Fact]
    public void Should_Chain_Rules_And_Block_User_On_High_Velocity()
    {
        // Arrange
        var user = new UserProfile { UserId = "U1", IsBlocked = false };
        var baseTime = DateTime.UtcNow;

        // Creamos 5 transacciones en menos de 5 horas para disparar la regla de Velocity
        var txs = Enumerable.Range(1, 5).Select(i => 
            new PaymentEvent($"E{i}", "Card1", 100m, "US", baseTime.AddHours(-i), true)
        ).ToArray();

        InsertFacts(user);
        InsertFacts(txs); // Insertamos el array de transacciones

        // Act
        Session.Fire();

        // Assert
        user.IsBlocked.Should().BeTrue("La regla BlockUserRule debió reaccionar al FraudDecision");
        
        // Podemos consultar la Working Memory para verificar que el Fact intermedio se creó
        var fraudDecisions = Session.Query<FraudDecision>().ToList();
        fraudDecisions.Should().ContainSingle();
        fraudDecisions.First().Reason.Should().Contain("Velocity");
    }
}
```

---

#### 6. Estrategias para Grandes Sistemas: Detección de Regresiones

Cuando tenés 500 reglas, un cambio en la Regla #42 puede alterar el resultado de la Regla #300. Los tests unitarios no van a atrapar esto. Necesitás **Snapshot Testing** o **Golden Master Testing**.

**La Técnica:**
1.  Tomás 100 casos reales de producción (JSONs con el estado inicial de los Facts).
2.  Corrés el motor de reglas actual y guardás el resultado final (JSON de salida) como tu "Golden Master".
3.  En tu pipeline de CI/CD, antes de un PR, corrés el motor con los mismos 100 casos de entrada.
4.  Comparás el JSON de salida nuevo con el Golden Master. Si hay *cualquier* diferencia no intencional, el test falla.

```csharp
// Ejemplo conceptual de Snapshot Testing usando la librería VerifyTests
using VerifyXunit;

[UsesVerify]
public class RegressionTests : RuleTestFixture
{
    public RegressionTests()
    {
        // Cargamos TODAS las reglas del sistema
        _repository.Load(x => x.From(typeof(AnyRuleInSystem).Assembly));
        CompileAndCreateSession();
    }

    [Fact]
    public async Task Complex_Scenario_Should_Match_Golden_Master()
    {
        // Arrange: Cargar un escenario complejo desde un archivo JSON
        var inputFacts = LoadFactsFromJson("complex_scenario_input_001.json");
        InsertFacts(inputFacts);

        // Act
        Session.Fire();

        // Extraemos el estado final que nos importa (ej. la Orden mutada)
        var finalOrderState = Session.Query<Order>().Single();

        // Assert: Verify serializa el objeto y lo compara con un archivo .verified.txt guardado previamente
        await Verifier.Verify(finalOrderState);
    }
}
```

---

#### 7. Versionado de Reglas (Rule Versioning)

En sistemas financieros, no podés simplemente sobrescribir una regla. Si un auditor pregunta por qué se rechazó un préstamo hace 2 años, tenés que poder ejecutar la versión exacta de las reglas que existía en ese momento.

**Estrategia Arquitectónica:**
NRules no tiene versionado nativo. Lo tenés que implementar en tu capa de aplicación usando **Tags** o **Namespaces**.

1.  **Por Namespace/Assembly:** Mantenés las reglas viejas en `MyCompany.Rules.V1` y las nuevas en `MyCompany.Rules.V2`.
2.  **Por Metadata (Tags):** Usás el método `Tag()` en la definición de la regla.

```csharp
public class TaxRule_2023 : Rule
{
    public override void Define()
    {
        Name("Tax_Calculation");
        Tag("Year_2023"); // Etiqueta de versión
        // ...
    }
}

public class TaxRule_2024 : Rule
{
    public override void Define()
    {
        Name("Tax_Calculation");
        Tag("Year_2024");
        // ...
    }
}
```

**En tiempo de ejecución (Infrastructure):**
Cuando creás el `RuleRepository`, filtrás por el Tag correspondiente al contexto temporal de la transacción.

```csharp
var repository = new RuleRepository();
repository.Load(x => x.From(typeof(TaxRule_2023).Assembly));

// Compilamos SOLO las reglas del año 2023
var rules2023 = repository.GetRules().Where(r => r.Tags.Contains("Year_2023"));
var factory2023 = compiler.Compile(rules2023);
```

Esta estrategia te permite tener múltiples `ISessionFactory` cacheados en memoria (uno por versión) y enrutar el request al motor correcto basándote en la fecha de la transacción.

---

Este es el estándar de la industria para testear motores de inferencia. Si no tenés una suite de integración que valide el orden de ejecución (Salience) y una suite de regresión que valide el estado final de la Working Memory, estás volando a ciegas.

---

### Módulo 8: Inyección de Dependencias (DI) y Orquestación de Servicios Externos

Llegamos al punto de fricción arquitectónico más grande en cualquier implementación de motores de reglas en .NET. 

Las reglas de negocio puras (como vimos hasta ahora) solo mutan el estado de los objetos en memoria. Pero en el mundo real, cuando una regla detecta un fraude, necesitás registrar un log (`ILogger`), guardar la alerta en la base de datos (`DbContext`), o enviar un email (`IEmailService`).

**El Problema Fundamental:**
Las clases que heredan de `Rule` en NRules **no son instanciadas por el contenedor de DI de .NET** (`IServiceProvider`). Son instanciadas internamente por el `RuleRepository` usando Reflection durante la fase de compilación (Singleton). 
Si intentás inyectar un `DbContext` (Scoped) en el constructor de tu regla, vas a tener un *Captive Dependency* (Dependencia Cautiva): el DbContext vivirá para siempre, compartirá estado entre todos los requests, y eventualmente lanzará excepciones de concurrencia o agotará el pool de conexiones.

En este capítulo, vamos a resolver este problema con precisión quirúrgica usando el patrón **Dependency Resolver** nativo de NRules.

---

#### 1. La Arquitectura Correcta: ¿Dónde inyectar?

Existen tres formas de interactuar con servicios externos en NRules. Solo una es correcta para sistemas empresariales.

**Anti-Patrón 1: Inyección en el Constructor de la Regla**
```csharp
// ¡NUNCA HAGAS ESTO!
public class FraudRule : Rule
{
    private readonly AppDbContext _db; // Dependencia Cautiva (Singleton)
    public FraudRule(AppDbContext db) { _db = db; } 
    // ...
}
```

**Anti-Patrón 2: Service Locator (Anti-Patrón Global)**
```csharp
// ¡NUNCA HAGAS ESTO!
Then()
    .Do(ctx => 
    {
        var db = MyGlobalServiceProvider.GetService<AppDbContext>(); // Acoplamiento oculto
        db.Alerts.Add(new Alert());
    });
```

**El Patrón Correcto: Resolución en Tiempo de Ejecución (Runtime Resolution)**
NRules provee un mecanismo para resolver dependencias **en el momento en que se dispara la regla (Rule Firing)**, no cuando se compila. Esto permite que la regla solicite un servicio Scoped (como `DbContext` o `ILogger`) que pertenece al request HTTP actual.

---

#### 2. Implementando el `IDependencyResolver`

Para conectar NRules con el `IServiceProvider` de .NET Core, necesitamos implementar la interfaz `IDependencyResolver` de NRules.

```csharp
// Capa: Infrastructure
using NRules.Extensibility;
using System;

public class DotNetDependencyResolver : IDependencyResolver
{
    private readonly IServiceProvider _serviceProvider;

    public DotNetDependencyResolver(IServiceProvider serviceProvider)
    {
        _serviceProvider = serviceProvider;
    }

    public object Resolve(IResolutionContext context, Type serviceType)
    {
        // Resolvemos el servicio usando el contenedor nativo de .NET
        // context.Rule contiene metadatos de la regla que está pidiendo el servicio
        return _serviceProvider.GetService(serviceType);
    }
}
```

---

#### 3. Inyectando Servicios en el Bloque `Then()`

Ahora vamos a escribir una regla que necesita un `ILogger` y un `IFraudRepository` (Scoped).

La magia ocurre usando el método `ctx.Resolve<T>()` dentro del bloque `Then()`.

```csharp
// Capa: Core.Application
using NRules.Fluent.Dsl;
using Microsoft.Extensions.Logging;

public class HighVelocityFraudRule : Rule
{
    public override void Define()
    {
        PaymentEvent currentEvent = null;

        Name("Fraud_High_Velocity_With_Logging");

        When()
            .Match<PaymentEvent>(() => currentEvent, e => e.Amount > 10000m);

        Then()
            // Inyectamos las dependencias directamente en la firma del delegado
            .Do(ctx => HandleFraud(
                ctx, 
                currentEvent, 
                ctx.Resolve<ILogger<HighVelocityFraudRule>>(), // Resolución Scoped
                ctx.Resolve<IFraudRepository>()                // Resolución Scoped
            ));
    }

    private void HandleFraud(IContext ctx, PaymentEvent ev, ILogger logger, IFraudRepository repo)
    {
        logger.LogWarning("High velocity fraud detected for Card {CardId}. Amount: {Amount}", ev.CardId, ev.Amount);

        var alert = new FraudAlert { EventId = ev.EventId, Reason = "High Amount" };
        
        // Guardamos en la base de datos sincrónicamente (veremos async más adelante)
        repo.SaveAlert(alert);

        // Insertamos el Fact en la Working Memory por si otras reglas lo necesitan
        ctx.Insert(alert);
    }
}
```

---

#### 4. Orquestación del Ciclo de Vida (El Setup en `Program.cs`)

Acá es donde unimos todo. Tenemos que asegurarnos de que el `ISessionFactory` sea Singleton, pero que cada `ISession` reciba un `IDependencyResolver` atado al *Scope* del request HTTP actual.

```csharp
// Capa: API / WebHost (Program.cs o Startup.cs)
using Microsoft.Extensions.DependencyInjection;
using NRules;
using NRules.Fluent;

var builder = WebApplication.CreateBuilder(args);

// 1. Registramos nuestros servicios de dominio (Scoped)
builder.Services.AddScoped<IFraudRepository, FraudRepository>();
builder.Services.AddDbContext<AppDbContext>(/* ... */);

// 2. Compilamos la red Rete y la registramos como Singleton
builder.Services.AddSingleton<ISessionFactory>(sp =>
{
    var repository = new RuleRepository();
    repository.Load(x => x.From(typeof(HighVelocityFraudRule).Assembly));
    
    var compiler = new RuleCompiler();
    return compiler.Compile(repository.GetRules());
});

// 3. Registramos un Factory Method para crear la ISession (Scoped)
builder.Services.AddScoped<ISession>(sp =>
{
    var factory = sp.GetRequiredService<ISessionFactory>();
    
    // Creamos la sesión
    var session = factory.CreateSession();
    
    // Le asignamos el DependencyResolver atado al IServiceProvider actual (Scoped)
    session.DependencyResolver = new DotNetDependencyResolver(sp);
    
    return session;
});

// 4. Registramos nuestro servicio de aplicación (Scoped)
builder.Services.AddScoped<FraudEvaluationService>();

var app = builder.Build();
```

**El Servicio de Aplicación (`FraudEvaluationService`):**

```csharp
public class FraudEvaluationService
{
    private readonly ISession _session;

    // Recibimos la sesión ya configurada con el DependencyResolver del request actual
    public FraudEvaluationService(ISession session)
    {
        _session = session;
    }

    public void Evaluate(PaymentEvent paymentEvent)
    {
        _session.Insert(paymentEvent);
        
        // Cuando se llame a Fire(), si la regla hace match, 
        // ctx.Resolve<IFraudRepository>() devolverá la instancia Scoped correcta.
        _session.Fire(); 
    }
}
```

---

#### 5. El Lado Oscuro: I/O Asíncrono (Async/Await) en NRules

Este es el **mayor limitante arquitectónico de NRules**. El motor de reglas (el método `Fire()`) es **estrictamente sincrónico**. No podés usar `await` dentro del bloque `Then()`.

Si intentás hacer esto, el compilador de C# te va a dejar (porque `Do()` acepta un `Action`), pero vas a crear un método `async void`, lo cual es un pecado capital en .NET (las excepciones se tragan, el hilo se bloquea, y el request HTTP puede terminar antes de que la tarea asíncrona finalice).

**Anti-Patrón (Async Void):**
```csharp
// ¡NUNCA HAGAS ESTO!
Then()
    .Do(async ctx => // ERROR: async void
    {
        var api = ctx.Resolve<IExternalApiClient>();
        await api.SendAlertAsync(currentEvent); // El motor no esperará a que esto termine
    });
```

**Solución 1: I/O Sincrónico (Bloqueante)**
Si la llamada es a una base de datos local (EF Core) y es rápida, podés usar la versión sincrónica (`SaveChanges()` en lugar de `SaveChangesAsync()`). Esto bloquea el hilo, pero mantiene la consistencia del motor.

```csharp
Then()
    .Do(ctx => 
    {
        var db = ctx.Resolve<AppDbContext>();
        db.Alerts.Add(new Alert());
        db.SaveChanges(); // Sincrónico. Bloquea el hilo.
    });
```

**Solución 2: El Patrón Outbox (Event-Driven / Recomendado para Microservicios)**
Si necesitás hacer una llamada HTTP lenta o publicar en Kafka, **NO lo hagas dentro de la regla**. 
La regla debe ser pura: solo debe insertar un Fact de "Intención" (Intent Fact). Luego, fuera del motor de reglas, procesás esas intenciones de forma asíncrona.

```csharp
// 1. El Fact de Intención
public class SendEmailCommand
{
    public string To { get; init; }
    public string Body { get; init; }
}

// 2. La Regla (Pura, sin I/O)
public class NotifyFraudRule : Rule
{
    public override void Define()
    {
        FraudAlert alert = null;
        When().Match<FraudAlert>(() => alert);
        Then().Do(ctx => ctx.Insert(new SendEmailCommand { To = "admin@bank.com", Body = alert.Reason }));
    }
}

// 3. El Servicio de Aplicación (Orquestador Asíncrono)
public class FraudEvaluationService
{
    private readonly ISession _session;
    private readonly IEmailService _emailService; // Servicio asíncrono real

    public FraudEvaluationService(ISession session, IEmailService emailService)
    {
        _session = session;
        _emailService = emailService;
    }

    public async Task EvaluateAsync(PaymentEvent paymentEvent)
    {
        _session.Insert(paymentEvent);
        
        // Ejecución sincrónica pura en memoria (Ultra rápido)
        _session.Fire(); 

        // Extraemos las intenciones generadas por las reglas
        var emailCommands = _session.Query<SendEmailCommand>().ToList();

        // Procesamos el I/O de forma asíncrona fuera del motor
        foreach (var cmd in emailCommands)
        {
            await _emailService.SendAsync(cmd.To, cmd.Body);
        }
    }
}
```

**Por qué la Solución 2 es Arquitectura de Nivel Senior:**
1.  **Performance:** El hilo que ejecuta el algoritmo Rete nunca se bloquea esperando un socket de red.
2.  **Testabilidad:** Podés testear la regla `NotifyFraudRule` sin necesidad de mockear `IEmailService`. Solo verificás que el Fact `SendEmailCommand` se haya insertado en la Working Memory.
3.  **Resiliencia:** Si el servidor de emails está caído, la evaluación de fraude no falla. Podés guardar los `SendEmailCommand` en una base de datos (Patrón Outbox real) y reintentar más tarde.

---

#### 6. Inyección de Dependencias en el Bloque `When()` (Peligro Extremo)

¿Se puede inyectar un servicio en el bloque `When()` para evaluar una condición?
Técnicamente sí, usando un patrón de "Fact de Servicio" (Service Fact), pero es un **anti-patrón masivo**.

**El Escenario:** Querés consultar una API externa para saber si un código postal es válido *durante* el matching.

```csharp
// ANTI-PATRÓN: I/O en el When
When()
    .Match<Address>(() => address)
    .Match<IZipCodeValidator>(() => validator, 
        v => v.IsValid(address.ZipCode)); // Llamada HTTP oculta en el matching
```

**Por qué esto destruye tu sistema:**
1.  NRules evalúa los nodos Alpha y Beta múltiples veces durante el ciclo de vida de la sesión (cada vez que insertás o actualizás un Fact relacionado).
2.  Si `v.IsValid()` hace una llamada HTTP, vas a disparar cientos de requests a la API externa por cada orden que proceses.
3.  La red Rete asume que las funciones en el `When` son **puras y deterministas**. Si la API externa devuelve `true` la primera vez y `false` la segunda (por un timeout), el grafo Rete se corrompe internamente y el comportamiento del motor se vuelve impredecible.

**La Regla de Oro del Arquitecto:**
**El bloque `When()` NUNCA debe contener I/O, llamadas a bases de datos, APIs, ni depender del reloj del sistema (`DateTime.Now`). Todo el estado necesario para tomar la decisión debe ser pre-cargado e insertado como Facts estáticos ANTES de llamar a `session.Fire()`.**

---

Con este módulo, tenés el panorama completo de cómo integrar NRules en una arquitectura empresarial moderna con .NET Core, respetando los principios SOLID, Clean Architecture y las limitaciones físicas del algoritmo Rete.

Este es el final del temario profundo. Si querés explorar algún caso de uso específico, patrón de diseño avanzado, o tenés un problema real de arquitectura que quieras modelar con NRules, indicamelo y lo desglosamos.

---

### Módulo 9: Arquitectura Avanzada: NRules en el Ecosistema DDD, CQRS y Event-Driven

Hasta ahora vimos a NRules como un componente aislado. Pero en un sistema empresarial real, el motor de reglas no vive en el vacío; interactúa con Aggregates, Commands, Events y bases de datos. 

El mayor error que cometen los equipos al adoptar un motor de reglas es **destruir su modelo de dominio**. Empiezan a sacar toda la lógica de sus entidades (Aggregates) y la mueven a NRules, convirtiendo su dominio en un modelo anémico (Anemic Domain Model) donde las clases son solo bolsas de getters y setters.

En este capítulo, vamos a integrar NRules respetando estrictamente los principios de **Domain-Driven Design (DDD)** y **CQRS (Command Query Responsibility Segregation)**.

---

#### 1. NRules y DDD: ¿Dónde vive el Motor de Reglas?

En DDD, la lógica de negocio pertenece al Dominio. Pero, ¿qué pasa cuando esa lógica es demasiado compleja, volátil o cruza múltiples Aggregates?

Tenemos dos opciones arquitectónicas válidas:

**Opción A: NRules como un Domain Service (Servicio de Dominio)**
Usamos esta opción cuando las reglas dictan **invariantes complejas** o calculan valores que pertenecen estrictamente al dominio (ej. Pricing, Scoring). El motor de reglas se inyecta en la capa de Dominio a través de una interfaz.

*   **El Aggregate Root:** `Order`
*   **La Interfaz (Core.Domain):** `IPricingEngine`
*   **La Implementación (Infrastructure):** `NRulesPricingEngine`

```csharp
// Capa: Core.Domain
public class Order // Aggregate Root
{
    public string Id { get; private set; }
    public List<OrderItem> Items { get; private set; } = new();
    public decimal TotalDiscount { get; private set; }

    // El Aggregate recibe el Domain Service como argumento en el método que muta el estado
    public void CalculateFinalPrice(IPricingEngine pricingEngine, CustomerProfile customer)
    {
        // El Aggregate delega el cálculo complejo al motor, pero ÉL mantiene el control de su estado
        var discountResult = pricingEngine.CalculateDiscount(this, customer);
        
        if (discountResult.Amount > this.SubTotal)
            throw new DomainException("Discount cannot exceed subtotal");

        this.TotalDiscount = discountResult.Amount;
    }
}
```

**Opción B: NRules como un Application Service (Orquestador de Casos de Uso)**
Usamos esta opción cuando las reglas dictan **flujos de trabajo (Workflows)** o cruzan límites transaccionales (ej. Antifraude, Aprobaciones). El motor vive en la capa de Aplicación y orquesta la llamada a múltiples repositorios y Aggregates.

```csharp
// Capa: Core.Application (Command Handler en CQRS)
public class ProcessPaymentCommandHandler : IRequestHandler<ProcessPaymentCommand>
{
    private readonly IPaymentRepository _repository;
    private readonly IFraudRuleEngine _fraudEngine; // NRules envuelto en una interfaz

    public async Task Handle(ProcessPaymentCommand command)
    {
        var payment = await _repository.GetByIdAsync(command.PaymentId);
        var userHistory = await _repository.GetUserHistoryAsync(payment.UserId);

        // El Application Service orquesta la evaluación
        var fraudDecision = _fraudEngine.Evaluate(payment, userHistory);

        if (fraudDecision.IsFraudulent)
        {
            payment.MarkAsFailed(fraudDecision.Reason); // Mutamos el Aggregate
        }
        else
        {
            payment.Process();
        }

        await _repository.SaveAsync(payment);
    }
}
```

---

#### 2. Integración con CQRS y Event-Driven Architecture

NRules brilla en el lado de los **Commands** (Escritura) de CQRS, validando si una acción puede llevarse a cabo. Pero su verdadero poder se desata cuando lo combinamos con **Domain Events**.

**El Patrón: Reglas que emiten Eventos de Dominio**
En lugar de que la regla mute un Aggregate directamente (lo cual puede violar el encapsulamiento si la regla usa setters públicos), la regla genera un `DomainEvent` como un Fact. Luego, el Application Service recolecta esos eventos y los despacha a través de un mediador (ej. MediatR).

```csharp
// 1. El Evento de Dominio (Fact)
public record HighRiskDetectedEvent(string UserId, string Reason) : INotification;

// 2. La Regla (Pura, solo emite eventos)
public class VelocityFraudRule : Rule
{
    public override void Define()
    {
        UserTransactionHistory history = null;

        When()
            .Match<UserTransactionHistory>(() => history, h => h.RecentTransactionsCount > 10);

        Then()
            // La regla NO bloquea al usuario. Solo declara que ocurrió un evento de alto riesgo.
            .Do(ctx => ctx.Insert(new HighRiskDetectedEvent(history.UserId, "Velocity > 10")));
    }
}

// 3. El Application Service (Orquestador)
public class EvaluateFraudService
{
    private readonly ISession _session;
    private readonly IMediator _mediator;

    public async Task EvaluateAsync(UserTransactionHistory history)
    {
        _session.Insert(history);
        _session.Fire();

        // Extraemos todos los eventos generados por las reglas
        var events = _session.Query<HighRiskDetectedEvent>().ToList();

        // Despachamos los eventos al bus (MediatR, Kafka, etc.)
        foreach (var domainEvent in events)
        {
            await _mediator.Publish(domainEvent);
        }
    }
}

// 4. El Event Handler (El que realmente muta el estado o hace I/O)
public class HighRiskDetectedEventHandler : INotificationHandler<HighRiskDetectedEvent>
{
    private readonly IUserRepository _userRepo;

    public async Task Handle(HighRiskDetectedEvent notification, CancellationToken ct)
    {
        var user = await _userRepo.GetByIdAsync(notification.UserId);
        user.Block(notification.Reason); // Mutación segura del Aggregate
        await _userRepo.SaveAsync(user);
    }
}
```

**Por qué este patrón es superior:**
*   **Desacoplamiento Total:** Las reglas no saben cómo se bloquea a un usuario ni qué base de datos se usa.
*   **Testabilidad:** Testear la regla es trivial: insertás el historial y verificás que el evento `HighRiskDetectedEvent` exista en la Working Memory.
*   **Side-Effects Controlados:** Si el bloqueo del usuario falla por un error de base de datos, el motor de reglas ya terminó su trabajo limpiamente.

---

#### 3. Reglas Configurables y Exposición del Motor como Servicio (Rules as a Service)

En sistemas grandes, no querés recompilar y redesplegar tu microservicio cada vez que negocio cambia el porcentaje de un descuento. Necesitás que las reglas sean **configurables en tiempo de ejecución**.

NRules compila código C# (Expression Trees). No lee XML ni JSON nativamente como Drools. Para hacer reglas dinámicas, tenés dos caminos:

**Camino A: Patrón de "Fact de Configuración" (Recomendado)**
No cambiás la regla en sí; cambiás los datos que la regla evalúa. Las reglas se vuelven genéricas.

```csharp
// 1. El Fact de Configuración (Viene de la Base de Datos)
public record DiscountPolicy(string CustomerTier, decimal MinOrderAmount, decimal DiscountPercentage);

// 2. La Regla Genérica (Compilada una sola vez)
public class DynamicTierDiscountRule : Rule
{
    public override void Define()
    {
        Customer customer = null;
        Order order = null;
        DiscountPolicy policy = null;

        When()
            // Hacemos Join entre el cliente, la orden y la política activa
            .Match<DiscountPolicy>(() => policy)
            .Match<Customer>(() => customer, c => c.Tier == policy.CustomerTier)
            .Match<Order>(() => order, 
                o => o.CustomerId == customer.Id,
                o => o.SubTotal >= policy.MinOrderAmount);

        Then()
            .Do(ctx => order.ApplyDiscount(policy.DiscountPercentage));
    }
}
```
*Ventaja:* Negocio puede agregar nuevas políticas en una UI (que guarda en BD), y el motor las aplica instantáneamente (inyectando las `DiscountPolicy` como Facts al inicio de la sesión) sin recompilar nada.

**Camino B: Generación Dinámica de Reglas (RuleBuilder API)**
Si negocio necesita crear reglas con estructuras lógicas completamente nuevas (ej. agregar un `AND` o un `OR` que no existía), tenés que usar la API fluida de bajo nivel de NRules (`RuleBuilder`) para construir los Expression Trees en memoria a partir de un JSON o una base de datos.

```csharp
// Ejemplo MUY simplificado de construcción dinámica
using NRules.RuleModel.Builders;

public class DynamicRuleFactory
{
    public IRuleDefinition BuildRuleFromJson(string jsonConfig)
    {
        // Parsearías tu JSON acá...
        var builder = new RuleBuilder();
        builder.Name("Dynamic_Rule_1");

        // Construcción del Pattern (When)
        var pattern = builder.LeftHandSide().Pattern(typeof(Order), "order");
        
        // Crear el Expression Tree dinámicamente: o => o.Total > 1000
        var parameter = Expression.Parameter(typeof(Order), "o");
        var property = Expression.Property(parameter, "Total");
        var constant = Expression.Constant(1000m);
        var greaterThan = Expression.GreaterThan(property, constant);
        var lambda = Expression.Lambda(greaterThan, parameter);

        pattern.Condition(lambda);

        // Construcción de la Acción (Then)
        // ... (Requiere construir Expression Trees para las acciones, lo cual es muy complejo)

        return builder.Build();
    }
}
```
*Advertencia de Arquitecto:* Construir Expression Trees dinámicamente es extremadamente propenso a errores, difícil de testear y lento de compilar. Evitá este camino a menos que estés construyendo un producto SaaS tipo "Zapier" donde los usuarios definen lógica arbitraria. El 99% de los casos de negocio se resuelven con el Camino A (Facts de Configuración).

---

#### 4. Versionado Avanzado y Blue/Green Deployments de Reglas

Cuando exponés el motor como un servicio centralizado (ej. un microservicio gRPC `PricingEngineService` que es llamado por 10 aplicaciones distintas), el versionado es crítico.

Si el equipo de Marketing lanza la "Campaña Navidad V2", no podés romper las órdenes en vuelo que se están procesando con la "Campaña Navidad V1".

**La Arquitectura de Multi-Tenancy / Multi-Version:**

En lugar de tener un solo `ISessionFactory` Singleton, mantenés un `ConcurrentDictionary<string, ISessionFactory>`.

```csharp
public class RuleEngineRegistry
{
    private readonly ConcurrentDictionary<string, ISessionFactory> _factories = new();
    private readonly IRuleCompiler _compiler = new RuleCompiler();

    // Se llama al inicio o cuando hay un nuevo despliegue de reglas
    public void RegisterVersion(string versionTag, IEnumerable<IRuleDefinition> rules)
    {
        var factory = _compiler.Compile(rules);
        _factories.TryAdd(versionTag, factory);
    }

    public ISession CreateSession(string versionTag)
    {
        if (_factories.TryGetValue(versionTag, out var factory))
        {
            return factory.CreateSession();
        }
        throw new Exception($"Rule version {versionTag} not found.");
    }
}
```

**El Flujo del Request:**
1. El cliente (ej. la App Móvil) envía un request HTTP con un header `X-Rule-Version: v2.1`.
2. El API Gateway enruta el request al microservicio de Pricing.
3. El microservicio extrae el header, pide la sesión específica al `RuleEngineRegistry` (`CreateSession("v2.1")`), inserta los Facts y ejecuta.
4. Si un cliente viejo envía `v1.0`, usa el grafo Rete compilado para esa versión. Cero conflictos.

---

### Módulo 10: El Veredicto Arquitectónico: Cuándo, Por Qué y Contra Qué

Llegamos a la decisión final. Como Arquitecto de Software, tu trabajo no es dominar una herramienta para usarla en todos lados; tu trabajo es saber exactamente cuándo una herramienta es la solución perfecta y cuándo es una bomba de tiempo. 

Introducir un motor de inferencia basado en Rete en una arquitectura .NET no es una decisión trivial. Cambia la topología de memoria de tus microservicios, altera la forma en que tu equipo escribe pruebas y requiere un cambio de paradigma mental del código imperativo al declarativo.

En este capítulo final, vamos a destilar 15 años de experiencia en producción en un marco de decisión implacable.

---

#### 1. El Checklist de Decisión Arquitectónica (Go / No-Go)

Antes de hacer `dotnet add package NRules`, tenés que someter tu dominio a este interrogatorio. Si respondés "No" a la mayoría, retrocedé.

1.  **¿La lógica es altamente combinatoria?**
    *   *Sí:* Tenemos entidades A, B, C y D. Las reglas cruzan propiedades de las cuatro simultáneamente para tomar una decisión. (Punto para NRules).
    *   *No:* La lógica solo evalúa la entidad A de forma aislada. (Usá el patrón *Strategy* o validación fluida tipo *FluentValidation*).
2.  **¿La volatilidad de las reglas es alta?**
    *   *Sí:* Negocio cambia las condiciones de pricing o fraude todas las semanas.
    *   *No:* Las reglas son invariantes del dominio core que rara vez cambian (ej. "Una orden no puede tener un total negativo").
3.  **¿El problema es de clasificación/decisión y no de transformación?**
    *   *Sí:* Queremos saber si una transacción es "Fraude" o "Segura".
    *   *No:* Queremos mapear un XML complejo a un modelo relacional. (NRules no es una herramienta ETL).
4.  **¿Existe un requerimiento estricto de Explicabilidad (Auditabilidad)?**
    *   *Sí:* Compliance exige saber exactamente qué 5 condiciones causaron el rechazo de un crédito. (NRules brilla acá con su determinismo).
    *   *No:* Solo nos importa el resultado final, no el "por qué".
5.  **¿La ejecución es transaccional en memoria?**
    *   *Sí:* Todos los datos necesarios (Facts) pueden cargarse en memoria RAM en menos de 50ms antes de evaluar.
    *   *No:* La evaluación requiere pausar, esperar un evento externo por 3 días, y continuar. (Esto es un Workflow, no un motor de reglas).

---

#### 2. Red Flags: Cuándo abortar la implementación de NRules

He visto proyectos fracasar y ser reescritos desde cero por ignorar estas banderas rojas.

**Red Flag 1: "Necesitamos que Negocio escriba las reglas en una UI"**
NRules es un motor **Code-First**. Las reglas se escriben en C# usando una API Fluida (DSL) y se compilan. NRules *no* trae una interfaz gráfica (Guvnor/Decision Central) como Drools, ni parsea tablas de Excel (DMN) de forma nativa. 
Si tu requerimiento es que un analista de marketing sin conocimientos de programación entre a una web, arrastre cajitas y publique una regla en producción, NRules te va a obligar a construir ese parseador visual-a-C# desde cero. Es un esfuerzo titánico. En ese caso, comprá un producto comercial (InRule, BizTalk) o usá Drools.

**Red Flag 2: "Las reglas necesitan consultar la base de datos en tiempo real"**
Si tu equipo diseña reglas que dicen *"Si el usuario es VIP, andá a la base de datos a buscar su historial"*, estás violando el modelo Rete. Rete asume que el estado del mundo (Working Memory) es estático durante el `Fire()`. Hacer I/O en el medio del matching destruye la performance, bloquea hilos y causa *Timeouts*. Si no podés pre-cargar los Facts, NRules no es tu herramienta.

**Red Flag 3: Sistemas de Ultra-Baja Latencia (HFT - High Frequency Trading)**
NRules es rápido (microsegundos), pero asigna mucha memoria en el Heap (Alpha/Beta Memories, Tuplas, Activaciones). En un sistema que procesa 100,000 ticks de bolsa por segundo, el Garbage Collector de .NET (Gen 0 y Gen 1) va a colapsar bajo la presión de las asignaciones de NRules, causando pausas (GC Pauses) inaceptables. Para HFT, necesitás C++ o Rust con memoria pre-asignada (Ring Buffers), no un motor Rete en C#.

---

#### 3. La Trampa del Overengineering (Cuándo NO usarlo)

El síndrome del "Martillo de Oro" ataca fuerte con los motores de reglas. 

**Caso de Overengineering: Autorización y Permisos (RBAC/ABAC)**
*El Escenario:* "Si el usuario tiene el rol 'Admin' y el documento está en estado 'Borrador', puede editarlo".
*Por qué es un error:* Implementar NRules para esto es matar una mosca con un cañón. El sistema de *Policies* y *Requirements* nativo de ASP.NET Core es infinitamente más idiomático, rápido y fácil de mantener. NRules agrega un overhead de compilación y manejo de sesiones que no aporta ningún valor a una simple evaluación booleana de dos variables.

**Caso de Overengineering: Routing de Mensajes**
*El Escenario:* "Si el mensaje es de tipo A, envialo a la cola X; si es B, a la cola Y".
*Por qué es un error:* Esto es un *Content-Based Router*. Herramientas como MassTransit, NServiceBus o incluso un simple `switch` con *Pattern Matching* de C# 8+ resuelven esto en 3 líneas de código con cero overhead de memoria.

---

#### 4. Los Escenarios Obligatorios (Donde NRules brilla)

Si estás construyendo alguno de estos sistemas en .NET y lo estás haciendo con `if/else` anidados, estás acumulando deuda técnica masiva.

1.  **Adjudicación de Reclamos Médicos (Healthcare Claims):**
    Cruzar códigos CIE-10 (diagnósticos) con códigos CPT (procedimientos), validando contra pólizas de seguro que tienen deducibles acumulativos, exclusiones por preexistencias y topes anuales. La combinatoria es brutal. NRules maneja esto con elegancia usando `Exists`, `Not` y `Collect`.
2.  **Motores de Promociones en Retail (Pricing Combinatorio):**
    "Llevá 3 pagá 2 en Electrónica, PERO si pagás con tarjeta X tenés 10% extra, EXCEPTO si el producto ya tiene descuento de marca, y el descuento máximo total no puede superar los $50". El manejo de **Salience (Prioridad)** y la capacidad de mutar el Fact `Cart` y reevaluar (Forward Chaining) hace que NRules sea la única forma cuerda de modelar esto sin volverse loco.
3.  **Scoring de Riesgo Crediticio:**
    Donde cada regla suma o resta puntos a un Fact acumulador (`ScoreResult`). La separación de responsabilidades es perfecta: cada regla es una clase aislada que no sabe de la existencia de las demás.

---

#### 5. La Arena: NRules vs. Alternativas

Como arquitecto, tenés que defender tu elección frente a un comité. Así es como NRules se compara con el resto del mundo.

**NRules vs. Código Imperativo (C# Pattern Matching / Strategy)**
*   *Imperativo:* $O(N \times M)$ complejidad. Difícil de leer cuando hay dependencias cruzadas. Rápido para pocas reglas.
*   *NRules:* $O(1)$ o cercano para la evaluación una vez que la red Rete está construida (gracias al caching de nodos Beta). Escala a miles de reglas sin degradación lineal de performance.
*   *Veredicto:* Usá código imperativo hasta las ~20-30 reglas simples. A partir de ahí, o cuando las reglas cruzan múltiples entidades (Joins), pasá a NRules.

**NRules vs. Drools (Java)**
*   *Drools:* Es el rey indiscutido de la industria. Tiene un ecosistema masivo, servidores de ejecución independientes (KIE Server), integración con DMN (Decision Model and Notation) y UIs para negocio.
*   *NRules:* Es una librería embebida para .NET. No tiene servidor propio, no tiene UI.
*   *Veredicto:* Si tu empresa ya tiene un clúster Java y un equipo de analistas que usan Guvnor, usá Drools. Si tu stack es 100% .NET Core, querés latencia in-process (sin llamadas HTTP al motor) y preferís que los desarrolladores mantengan las reglas en Git como código C#, NRules es superior y mucho más ligero.

**NRules vs. Motores de Workflow (Temporal.io, Azure Durable Functions)**
*   *Workflow:* Manejan el **tiempo y el estado persistente**. Pueden esperar meses a que un humano apruebe algo.
*   *NRules:* Maneja la **lógica instantánea**. No tiene concepto de "esperar".
*   *Veredicto:* Son complementarios, no excluyentes. Un Workflow orquesta los pasos ("Paso 1: Evaluar Fraude, Paso 2: Cobrar"). NRules ejecuta el paso ("Evaluar Fraude"). Nunca uses NRules para orquestar pasos secuenciales de larga duración.

**NRules vs. Machine Learning (Modelos Predictivos)**
*   *ML:* Probabilístico. "Hay un 85% de probabilidad de que esto sea fraude". Es una caja negra. Excelente para detectar patrones ocultos que un humano no puede ver.
*   *NRules:* Determinístico. "Esto ES fraude porque la IP está en la lista negra y el monto > $5000". Es 100% auditable.
*   *Veredicto:* La arquitectura moderna usa ambos. El modelo de ML genera un *Score* (ej. 0.85). Ese Score se inserta como un Fact en NRules. NRules toma la decisión final basada en políticas de negocio ("Si Score > 0.80 Y el cliente es VIP, enviar a revisión manual; si no es VIP, bloquear"). NRules actúa como el "Guardrail" (barrera de seguridad) del modelo de ML.

---

#### 6. Opinión Experta y Lecciones de Producción (The Veteran's Verdict)

Después de implementar motores de reglas en bancos, procesadoras de pago y sistemas logísticos, esta es la cruda realidad:

**El Cuello de Botella no es la CPU, es la Adquisición de Conocimiento.**
El problema más difícil que vas a enfrentar no es compilar la red Rete; es sentarte con el experto de dominio (el actuario, el analista de fraude) y traducir su conocimiento ambiguo ("*Normalmente* rechazamos esto si parece riesgoso") a lógica booleana estricta. NRules te obliga a formalizar el dominio. Si el dominio es un caos, NRules va a exponer ese caos inmediatamente.

**El Mantenimiento de los Facts es tu principal trabajo.**
NRules es tan bueno como los datos que le inyectás. Si tu base de datos es un desastre relacional y tardás 2 segundos en armar el objeto `CustomerProfile` con todos sus `Orders` y `Addresses` para insertarlo en la sesión, el hecho de que NRules evalúe las reglas en 2 milisegundos es irrelevante. Tu sistema es lento.
*Consejo de Arquitecto:* En sistemas de alta carga, usá CQRS. Mantené vistas materializadas (Read Models) en Redis o MongoDB que tengan la estructura exacta de los Facts que NRules necesita. Leés de Redis en 1ms, insertás en NRules, disparás en 1ms. Latencia total: 2ms.

**Cuidado con el "God Fact" (El Fact Dios).**
Un error común es crear una clase `ApplicationContext` que tiene referencias a TODO el sistema y pasarla como un Fact.
`session.Insert(godFact);`
Esto destruye la granularidad de Rete. Si cualquier propiedad del `GodFact` cambia, el motor tiene que reevaluar TODAS las reglas que dependen de él.
*Consejo de Arquitecto:* Rompé tus Facts en piezas pequeñas y cohesivas. Insertá `User`, `Device`, `Location`, `Transaction` por separado. Rete está optimizado para hacer Joins eficientes entre Facts pequeños, no para parsear monolitos en memoria.

**El Testing de Regresión te salvará la vida.**
Como mencioné en el Módulo 7, no confíes solo en los tests unitarios. En un sistema con 300 reglas, la interacción emergente es imposible de predecir por un humano. Invertí tiempo en armar un pipeline de CI/CD que corra miles de transacciones históricas contra la nueva versión de las reglas antes de cada despliegue.

---

### Conclusión Arquitectónica del Curso

Implementar NRules no es un problema de sintaxis de C#; es un problema de **diseño de estado**. 

El algoritmo Rete es una bestia que devora memoria a cambio de velocidad. Si lo alimentás con Facts pequeños, inmutables y bien filtrados (Alpha Nodes fuertes), te va a dar latencias de microsegundos y una explicabilidad perfecta para compliance.
Si lo alimentás con Aggregates gigantes, colecciones mutables y Joins abiertos (Beta Nodes débiles), va a colapsar tu servidor en el primer pico de tráfico.

**El Mantra del Arquitecto NRules:**
1. **Los Facts son inmutables por defecto.** Si mutás, llamá a `Update()`.
2. **El `When` es puro.** Cero I/O, cero bases de datos, cero `DateTime.Now`.
3. **El `Then` declara intenciones.** Emití eventos o Facts de resultado; dejá que el Application Service haga el I/O pesado de forma asíncrona.
4. **Cuidado con el producto cartesiano.** Filtrá antes de cruzar.
5. **Testea el estado final, no la regla aislada.** El valor del motor está en la interacción emergente de cientos de reglas.

---

### Cierre del Curso

Has recorrido el camino completo. Desde la teoría matemática de Charles Forgy en 1974 (Algoritmo Rete), pasando por la sintaxis fluida de C#, hasta la inyección de dependencias en .NET Core y el diseño de arquitecturas distribuidas de alta carga.

NRules es una de las librerías mejor diseñadas y más subestimadas del ecosistema .NET. Usada correctamente, te permite extraer la complejidad combinatoria de tu código espagueti y encapsularla en un motor determinista, auditable y extremadamente rápido. 
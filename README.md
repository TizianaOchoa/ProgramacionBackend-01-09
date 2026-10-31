# Programación Backend - UPC - Programación Full Stack
**Estudiante:** Ochoa Tiziana  
**Actividad:** Resolución de ejercicios prácticos sobre diseño de APIs REST, validación de DTOs con TypeScript, manejo de errores, inyección de dependencias, arquitectura modular y buenas prácticas de desarrollo backend.
---
## EJERCICIO 1: Diseño de Rutas y Métodos HTTP (Infracción de Principios REST)

Un desarrollador redactó el siguiente conjunto de rutas en Express para gestionar un recurso de tareas:

```
app.get("/api/v1/obtenerTodasLasTareas", (req, res) => { /* ... */ });
app.post("/api/v1/crearNuevaTarea", (req, res) => { /* ... */ });
app.post("/api/v1/modificarTarea/:id", (req, res) => { /* ... */ });
app.get("/api/v1/eliminarTarea/:id", (req, res) => { /* ... */ });
```

Preguntas:

1. ¿Qué principios de diseño de REST están siendo violados en estas definiciones de rutas?
2. Reescribe la tabla de rutas aplicando las buenas prácticas de diseño REST orientadas a recursos recomendadas en el documento docente.
   
Respuestas:

1. Principios violados:
   - Uso de verbos/acciones en las URIs (antipatrón RPC) en lugar de sustantivos que identifiquen recursos.
   - Uso incorrecto de métodos HTTP: se usa POST para modificar en lugar de PUT/PATCH, y GET para eliminar en lugar de DELETE (violando seguridad e idempotencia).

2. Tabla de rutas corregida aplicando buenas prácticas REST:

```
app.get("/api/v1/tareas", (req, res) => { /* Obtener todas las tareas */ });
app.get("/api/v1/tareas/:id", (req, res) => { /* Obtener una tarea específica */ });
app.post("/api/v1/tareas", (req, res) => { /* Crear una nueva tarea */ });
app.put("/api/v1/tareas/:id", (req, res) => { /* Modificar una tarea */ });
app.delete("/api/v1/tareas/:id", (req, res) => { /* Eliminar una tarea */ });
```

---
## EJERCICIO 2: Asunción Falsa de Validación con Tipos de TypeScript

Observa la siguiente implementación de un controlador para la creación de tareas:
```
import { Request, Response } from 'express';
import { CrearTareaDto } from './tarea.dto';

export function crearTareaController(req: Request, res: Response) {
    const datos = req.body as CrearTareaDto; 
    
    // El desarrollador asume que 'datos' ya tiene la estructura válida de CrearTareaDto
    const nuevaTarea = tareasService.crear(datos);
    return res.status(201).json({ data: nuevaTarea });
}
```
Preguntas:

1. ¿Por qué la aserción de tipo as CrearTareaDto NO garantiza la validez de los datos recibidos en runtime?
2. ¿Qué vulnerabilidades o fallos en ejecución pueden ocurrir si el cliente envía un JSON malformado o incompleto (ej. {} o {"titulo": 123})?

Respuestas:

1.
- TypeScript es un sistema de tipos estático que se elimina por completo durante la transpilación a JavaScript. Las interfaces y tipos (como CrearTareaDto) no existen en tiempo de ejecución (runtime).
- El operador `as` (type assertion) solo le dice al compilador de TypeScript que "confíe" en que el objeto tiene esa forma, pero no realiza ninguna validación, sanitización ni verificación real sobre el objeto `req.body` que envía el cliente por la red.

2.
- Errores críticos en runtime (TypeError): Si el servicio intenta ejecutar métodos sobre propiedades faltantes o con tipos incorrectos (ej. `datos.titulo.trim()` cuando `titulo` es undefined o un número), la app puede lanzar excepciones no controladas.
- Corrupción o inconsistencia de datos: Se pueden almacenar datos inválidos, incompletos o con tipos erróneos en la base de datos, rompiendo la integridad del sistema.
- Vulnerabilidades de lógica y seguridad: Si no se validan los campos estrictamente, el cliente podría inyectar propiedades no autorizadas o campos maliciosos (como sobreescritura de propiedades internas o mass assignment) que el backend procese por error.

---
## EJERCICIO 3: Implementación de Función de Validación Manual DTO

El siguiente código valida los datos de entrada para la creación de una tarea:
```
function validarCrearTarea(datos: unknown): CrearTareaDto {
    if (typeof datos !== "object" || datos === null) {
        throw new AppError(400, "INVALID_BODY", "El cuerpo debe ser un objeto JSON");
    }
    const objeto = datos as Record<string, unknown>;
    
    if (typeof objeto.titulo !== "string" || objeto.titulo.trim().length < 3) {
        throw new AppError(422, "INVALID_TITLE", "El título debe tener al menos tres caracteres");
    }
    
    const prioridades = ["baja", "media", "alta"];
    if (typeof objeto.prioridad !== "string" || !prioridades.includes(objeto.prioridad)) {
        throw new AppError(422, "INVALID_PRIORITY", "La prioridad no es válida");
    }
    
    return {
        titulo: objeto.titulo.trim(),
        prioridad: objeto.prioridad as CrearTareaDto["prioridad"]
    };
}
```
Preguntas:

1. ¿Por qué se utiliza el código de estado HTTP 422 Unprocessable Content para el título/prioridad en lugar de un código 400 Bad Request?
2. ¿Qué ocurriría si se envía un JSON como {"titulo": " AB ", "prioridad": "alta"}? ¿Pasa o falla la validación? Justifica evaluando el código.

Respuestas:

1.
- 400 Bad Request se utiliza de forma general cuando la petición del cliente tiene un error sintáctico (por ejemplo, el cuerpo de la petición no es un JSON válido o está malformado y no se puede interpretar).

- 422 Unprocessable Content indica que la petición llegó con una sintaxis correcta y el servidor pudo entenderla (es un JSON válido y es un objeto), pero contiene errores semánticos o de validación de negocio (los datos están incompletos, no cumplen con las reglas de longitud, o los valores no están permitidos).

2.
- Falla la validación

- Justificación:
  - Al evaluar el título, el código aplica .trim()sobre " AB ", lo cual elimina los espacios en blanco iniciales y finales, transformándolo en la cadena "AB".
  - La condición evalúa objeto.titulo.trim().length < 3. Como "AB"tiene una longitud de 2 caracteres, es . 2 < 3true
  - Por lo tanto, se activa el bloque ify se lanza inmediatamente el error AppErrorcon código 422 ( INVALID_TITLE), impidiendo que la tarea sea creada.

---
## EJERCICIO 4: Semántica de Actualización: PUT vs PATCH

Un servicio expone dos endpoints para modificar recursos existentes:
```
// Endpoint A
app.put("/api/v1/tareas/:id", (req, res) => {
    const { titulo, prioridad, completada } = req.body;
    const tareaActualizada = service.reemplazar(Number(req.params.id), { titulo, prioridad, completada });
    res.status(200).json({ data: tareaActualizada });
});
```
```
// Endpoint B
app.patch("/api/v1/tareas/:id", (req, res) => {
    const cambios = req.body;
    const tareaModificada = service.actualizarParcial(Number(req.params.id), cambios);
    res.status(200).json({ data: tareaModificada });
});
```
Preguntas:

1. Si el cliente envía a PUT /api/v1/tareas/5 únicamente el cuerpo {"completada": true}, ¿cuál es el comportamiento esperado según la especificación HTTP REST y qué problema ocurre con la entidad?
2. Explica el concepto de idempotencia en el contexto de las operaciones PUT y POST.

Respuestas:

1.
- Comportamiento esperado: Dado que el verbo PUTdefine un reemplazo completo (o actualización total) del recurso, el servidor interpretará que la nueva representación de la tarea número 5 es únicamente la que se envió en el cuerpo.

- Problema con la entidad: Se genera una pérdida de datos masiva . Los atributos esenciales que ya tenía la tarea (como tituloy prioridad) serán eliminados o sobrescritos con valores undefinedo null, dejando al recurso incompleto, inválido o rompiendo las reglas de negocio del sistema. Para actualizar solo propiedades específicas sin afectar al resto, se debe utilizar PATCH.

2.
- Idempotencia: Es la propiedad mediante la cual realizar una petición idéntica una o Múltiples veces produce exactamente el mismo efecto y estado en el servidor que realizarla una sola vez.

- PUT(Es idempotente): Si envías la petición PUT /api/v1/tareas/5diez veces seguidas con el mismo cuerpo, el recurso 5 quedará exactamente con esos datos tras la primera ejecución; las llamadas posteriores simplemente volverán a aplicar el mismo reemplazo sin alterar el estado final.

- POST(No es idempotente): Cada vez que envías una petición POSTal endpoint de creación, el servidor interpreta que debe crear un nuevo recurso. Si envías el mismo POSTdiez veces, se crearán diez recursos independientes con diez ID distintos, alterando el estado del sistema acumulativamente.

---
## EJERCICIO 5: Conversión e Interpretación de Parámetros de Ruta

Analiza la extracción del parámetro ID en el siguiente controlador:
```
app.get("/api/v1/tareas/:id", (req, res) => {
    const id = Number(req.params.id);
    
    if (!Number.isInteger(id) || id <= 0) {
        throw new AppError(400, "INVALID_ID", "El identificador no es válido");
    }
    
    const tarea = service.obtenerPorId(id);
    res.status(200).json({ data: tarea });
});
```

Preguntas:

1. ¿Qué valor toma la variable id y qué respuesta devuelve la API si el cliente realiza una petición a GET /api/v1/tareas/abc?
2. ¿Por qué se valida que el ID sea un entero positivo mayor a cero antes de invocar la capa de servicio?

Respuestas:

1.
- La variable `id` toma el valor `NaN` (Not-a-Number) ya que la función `Number("abc")` no puede convertir un string alfanumérico en un número válido.
- La API evalúa la condición `!Number.isInteger(NaN)` la cual es verdadera, por lo que se interrumpe la ejecución y se lanza un `AppError` que responde con un código HTTP 400 Bad Request ("INVALID_ID").

2.
- Fail-fast (Validación temprana en los límites): Permite rechazar peticiones inválidas antes de gastar recursos de procesamiento o hacer consultas innecesarias a la base de datos.
- Integridad y seguridad: Previene que lleguen valores absurdos, negativos, flotantes o `NaN` que puedan causar errores de consulta, excepciones en la base de datos o comportamientos inesperados en la lógica de negocio.

---
## EJERCICIO 6: Middleware Centralizado de Errores

Considera la siguiente implementación del middleware de manejo global de errores:
```
export const errorHandler: ErrorRequestHandler = (error, _request, response, _next) => {
    if (error instanceof AppError) {
        response.status(error.status).json({
            error: { code: error.code, message: error.message, details: error.details }
        });
        return;
    }
    
    console.error(error);
    response.status(500).json({
        error: { code: "INTERNAL_ERROR", message: "Ocurrió un error interno" }
    });
};
```
Preguntas:

1. ¿Por qué es fundamental filtrar por error instanceof AppError antes de retornar la respuesta?
2. ¿Por qué NO se deben retornar el stack trace (traza de pila) ni detalles técnicos de errores no controlados (como un fallo de conexión a BDD) en la respuesta JSON enviada al cliente final?

Respuestas:

1.
- Permite diferenciar entre errores operativos o controlados del negocio (donde conocemos la causa, el código HTTP adecuado y el mensaje específico que debemos comunicar al cliente) y errores inesperados del sistema (como fallos de programación, de tipado o caídas imprevistas).
- Garantiza que las reglas de negocio y validaciones devuelvan sus códigos correctos (ej. 400, 422) en lugar de etiquetar todo como un fallo genérico del servidor.

2.
- Seguridad (Fuga de información / Information Disclosure): Expone la estructura interna del servidor, rutas de directorios, versiones de librerías, ORMs o bases de datos, entregando información valiosa a posibles atacantes sobre la arquitectura.
- Experiencia y robustez: El cliente final no puede resolver un error interno del servidor; exponer trazas de pila técnicas confunde al usuario y resta profesionalismo. Por ello, la traza debe registrarse de forma segura en los logs del servidor (`console.error`) y responderse de manera genérica ("INTERNAL_ERROR").

---
## EJERCICIO 7: Acoplamiento Directo vs. Inyección de Dependencias

Un estudiante escribió la siguiente clase para su controlador:
```
export class TareasController {
    private service: TareasService;

    constructor() {
        const repository = new TareasRepository();
        this.service = new TareasService(repository);
    }

    public obtenerTodas = (_req: Request, res: Response) => {
        const data = this.service.obtenerTodas();
        res.status(200).json({ data });
    }
}
```
Preguntas:

1. ¿Qué problema de acoplamiento presenta la instanciación interna con new TareasRepository() y new TareasService() dentro del constructor?
2. Refactoriza la clase TareasController aplicando Inyección de Dependencias por constructor y explica cómo facilita esto la realización de pruebas unitarias con mocks.

Respuestas: 

1.
- Acoplamiento fuerte: El controlador asume la responsabilidad de crear sus propias dependencias, ligándose rígidamente a clases concretas (`TareasRepository` y `TareasService`) en lugar de depender de abstracciones.
- Imposibilidad de aislamiento para pruebas: Al instanciar las dependencias internamente con `new`, resulta imposible reemplazar el servicio o el repositorio por objetos simulados (`mocks` o `stubs`) durante las pruebas unitarias.
- Violación del Principio de Inversión de Dependencias (S.O.L.I.D.): Los componentes de alto nivel dependen directamente de implementaciones de bajo nivel, reduciendo la flexibilidad y mantenibilidad del sistema.

2. Refactorización aplicando Inyección de Dependencias por constructor:
```
export class TareasController {
    constructor(private readonly service: TareasService) {}

    public obtenerTodas = (_req: Request, res: Response) => {
        const data = this.service.obtenerTodas();
        res.status(200).json({ data });
    }
}
```
¿Cómo facilita esto la realización de pruebas unitarias con mocks?
- Permite suministrar una versión falsa o simulada del servicio (un mock) directamente al instanciar el controlador en el entorno de pruebas (`new TareasController(mockService)`).
- Aisla por completo al controlador de la lógica de negocio real, del repositorio y de la base de datos, logrando pruebas rápidas, independientes y enfocadas exclusivamente en validar el comportamiento HTTP y la estructura de la respuesta.

---
## EJERCICIO 8: Paginación y Cálculo de Offsets

Analiza el siguiente fragmento del servicio para consultar una colección paginada:
```
const page = Math.max(Number(request.query.page) || 1, 1);
const limit = Math.min(
    Math.max(Number(request.query.limit) || 10, 1),
    100
);

const start = (page - 1) * limit;
const end = start + limit;

const tareasPaginadas = todasLasTareas.slice(start, end);
```

Preguntas:

1. Si el cliente realiza una petición con los parámetros ?page=3&limit=5, ¿cuáles serán los valores de start y end en el método slice?
2. ¿Qué efecto tienen las funciones Math.max y Math.min si el cliente envía valores negativos o muy elevados como ?page=-2&limit=5000?

Respuestas:

1. 
- `page` = 3, `limit` = 5.
- `start` = (3 - 1) * 5 = 10.
- `end` = 10 + 5 = 15.

Por lo tanto, `slice(10, 15)` extraerá los elementos desde el índice 10 hasta el 14 (5 elementos en total correspondientes a la tercera página).

2.
- Protección ante valores negativos, cero o inválidos: `Math.max(..., 1)` actúa como un piso de seguridad garantizando que `page` y `limit` nunca sean menores a 1 (corrigiendo automáticamente un `page=-2` a `1`).
- Prevención de sobrecarga de servidor (DoS): `Math.min(..., 100)` actúa como un techo de seguridad que limita el tamaño máximo de elementos por página a 100, evitando que un cliente solicite un `limit` masivo (como 5000) que sature la memoria o bloquee la base de datos.

---
## EJERCICIO 9: Violación de la Arquitectura por Capas

Observa la siguiente ruta implementada por un desarrollador:
```
router.get("/:id", async (req, res) => {
    const id = Number(req.params.id);
    // Acceso directo a la fuente de datos global/memoria
    const tarea = baseDeDatosMemoria.find(t => t.id === id);
    
    if (!tarea) {
        return res.status(404).json({ error: "No encontrada" });
    }
    
    // Regla de negocio ejecutada directamente en la ruta
    tarea.vistas = (tarea.vistas || 0) + 1;
    
    res.status(200).json({ data: tarea });
});
```
Preguntas:

1. Identifica las responsabilidades mezcladas en este bloque de código y menciona qué capas de la arquitectura modular (Router, Controller, Service, Repository) se están omitiendo o salteando.
2. Indica cuál debería ser la única responsabilidad del Router y del Controller en este flujo.

Respuestas:

1.
- Responsabilidades mezcladas: Se mezclan la gestión de rutas, la manipulación de objetos HTTP, la lógica de negocio (incrementar las vistas) y el acceso directo a la fuente de datos (persistencia).
- Capas omitidas: Se están omitiendo por completo las capas de Controller, Service y Repository. La capa de rutas (Router) está asumiendo toda la responsabilidad de la aplicación.

2.
- Router: Su única responsabilidad es definir la URL, asociar el verbo HTTP al controlador correspondiente y encadenar los middlewares necesarios (como validación o autenticación). No debe contener lógica de negocio ni acceso a datos.
- Controller: Su única responsabilidad es recibir `req` y `res`, extraer los parámetros de la petición (como `req.params.id`), llamar a la capa de servicio (`Service`) para ejecutar la operación, y formatear/retornar la respuesta HTTP adecuada (`res.status(...).json(...)`).

---
## EJERCICIO 10: Especificación OpenAPI (YAML) y Contratos

Examina el siguiente fragmento de documentación OpenAPI 3.2.0:
```
paths:
  /tareas:
    post:
      summary: Crear una tarea
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [titulo, prioridad]
              properties:
                titulo:
                  type: string
                  minLength: 3
                prioridad:
                  type: string
                  enum: [baja, media, alta]
      responses:
        "201":
          description: Tarea creada
        "422":
          description: Datos inválidos
```

Preguntas:

1. Según este contrato OpenAPI, ¿qué respuesta debe retornar la API si un cliente envía el JSON {"titulo": "A", "prioridad": "urgente"}? Indique el código de estado HTTP y la causa.
2. ¿Qué diferencia fundamental existe entre OpenAPI y Swagger?

Respuestas:

1.
- Código de estado HTTP: 422 (Datos inválidos / Unprocessable Content).
- Causa: Incumple dos reglas del esquema definido en el contrato:
   - 1. El `titulo` ("A") tiene una longitud de 1 carácter, violando la regla `minLength: 3`.
   - 2. La `prioridad` ("urgente") no se encuentra dentro de los valores permitidos en el arreglo `enum: [baja, media, alta]`.

2.
- OpenAPI es la especificación estándar abierta (un formato formal basado en JSON/YAML) para describir APIs RESTful de forma independiente del lenguaje de programación.
- Swagger es el conjunto de herramientas de software (herramientas de desarrollo como Swagger UI, Swagger Editor, Swagger Codegen) originalmente creadas por SmartBear para implementar, visualizar y trabajar con la especificación OpenAPI.

---
## EJERCICIO 11: Modificación Incompatible y Versionado de APIs

Un equipo tiene desplegada la versión 1 de su API con la siguiente estructura de respuesta en GET /api/v1/tareas/15:
```
// Respuesta Versión 1
{
  "data": {
    "id": 15,
    "titulo": "Preparar TP",
    "prioridad": "alta"
  }
}
```
Para la nueva entrega, deciden renombrar la propiedad prioridad a nivelPrioridad y mover el id dentro de un subobjeto metadata.

Preguntas:

1. ¿Por qué este cambio constituye una "modificación breaking" (incompatible) para las aplicaciones cliente (frontend/mobile) existentes?
2. ¿Cómo se debe estructurar la URL del nuevo endpoint según las buenas prácticas de versionado de APIs explicadas en la unidad?

Respuestas:

1. 
- Rompe el contrato existente con el cliente: Renombrar propiedades (`prioridad` a `nivelPrioridad`) y cambiar la estructura de los datos (`id` movido a un subobjeto `metadata`) hará que cualquier cliente frontend o móvil que espere la estructura anterior falle al intentar acceder a propiedades `undefined`, provocando errores o pantallas en blanco en la aplicación.

2.
- Se debe incrementar el número de versión en la ruta de la URI pasando de `v1` a `v2`, manteniendo la versión anterior operativa de forma simultánea mientras los clientes migran:
  - `GET /api/v2/tareas/15`

---
## EJERCICIO 12: Implementación de Encabezados y Estructura de Respuesta DTO

Considera la creación exitosa de un recurso en Express:
```
outer.post("/", (request, response) => {
    const dto = validarCrearTarea(request.body);
    const tarea = service.crear(dto);
    response.location(`/api/v1/tareas/${tarea.id}`).status(201).json({ data: tarea });
});
```

Preguntas:

1. ¿Cuál es el propósito del encabezado HTTP Location asignado en la respuesta a una petición POST exitosa?
2. ¿Por qué se recomienda envolver la entidad devuelta dentro de un objeto con la propiedad data (ej. { data: tarea }) en lugar de retornar la entidad directamente en la raíz de la respuesta JSON?

Respuestas:

1.
- Indicar al cliente la URI exacta donde se puede consultar, acceder o referenciar el recurso que acaba de ser creado (ej. `/api/v1/tareas/15`).
- Sigue las buenas prácticas de la arquitectura REST, permitiendo que las aplicaciones cliente sepan dónde encontrar el nuevo recurso sin tener que adivinarlo.

2.
- Extensibilidad y flexibilidad: Permite incluir metadatos adicionales en el futuro (como información de paginación, marcas de tiempo, estados o enlaces HATEOAS) junto con la respuesta sin alterar la estructura raíz del objeto principal.
- Consistencia en el diseño de la API: Estandariza el formato de todas las respuestas de la aplicación (ya sean colecciones o recursos individuales), facilitando el tipado y el consumo por parte del frontend o clientes externos.

---
## EJERCICIO 13: Repositorio en Memoria e Inmutabilidad

Analiza el método obtenerTodas del siguiente Repositorio en memoria:
```
export class TareasRepository {
    private readonly tareas: Tarea[] = [];

    // Opción A
    obtenerTodasA(): Tarea[] {
        return this.tareas;
    }

    // Opción B
    obtenerTodasB(): Tarea[] {
        return [...this.tareas];
    }
}
```

Preguntas:

1. ¿Qué riesgo de seguridad/encapsulamiento tiene la Opción A si un servicio o controlador modifica el arreglo devuelto (ej. haciendo repo.obtenerTodasA().pop())?
2. ¿Cómo soluciona la Opción B este problema mediante el operador de propagación (spread operator)?

Respuestas:

1. Rompe el encapsulamiento de la clase `TareasRepository`, permitiendo que cualquier capa externa modifique directamente el estado interno del arreglo en memoria (`this.tareas`) sin control, lo que genera efectos colaterales inesperados y bugs difíciles de depurar.

2. Crea y retorna una copia superficial (*shallow copy*) del arreglo original utilizando el operador de propagación (`[...this.tareas]`). De este modo, si una capa externa muta o altera el arreglo devuelto, el estado interno del repositorio permanece intacto y protegido.

---
## EJERCICIO 14: Pruebas Manuales con cURL

Dado el siguiente comando cURL preparado para probar la API:
```
curl -i \
  -X POST \
  -H "Content-Type: application/json" \
  -d '{"titulo":"Estudiar API","prioridad":"alta"}' \
  http://localhost:3000/api/v1/tareas
```

Preguntas:

1. ¿Qué función cumple la opción -i en el comando curl?
2. ¿Qué ocurre si se omite el encabezado -H "Content-Type: application/json" cuando el backend utiliza el middleware express.json()? ¿Qué valor tendrá req.body?

Respuestas:

1. Hace que cURL incluya en la salida los encabezados de la respuesta HTTP junto con el cuerpo, permitiendo visualizar el código de estado, los headers devueltos (como `Location`) y los metadatos de la respuesta del servidor.

2. El middleware `express.json()` no podrá interpretar correctamente el contenido de la petición ya que requiere explícitamente el encabezado `Content-Type: application/json`. Como consecuencia, el middleware omitirá el parseo y `req.body` quedará como `undefined`.

---
## EJERCICIO 15: Generación Autónoma de Identificadores en la Capa de Persistencia

Observa los tipos TypeScript definidos para la aplicación:
```
export interface CrearTareaDto {
    titulo: string;
    prioridad: "baja" | "media" | "alta";
}

export interface Tarea {
    id: number;
    titulo: string;
    prioridad: "baja" | "media" | "alta";
    completada: boolean;
    fechaCreacion: string;
}
```

Preguntas:

1. ¿Por qué el campo id, completada y fechaCreacion están ausentes en la interfaz CrearTareaDto pero presentes en la interfaz Tarea?
2. ¿En qué capa del sistema (Controller, Service o Repository) se deben generar el id y la fechaCreacion al registrar una nueva tarea, y por qué no deben ser provistos por el cliente HTTP?

Respuestas:

1. Porque `CrearTareaDto` define únicamente los datos mínimos que el cliente debe enviar para crear el recurso. Los campos como `id`, `fechaCreacion` y `completada` son generados y controlados automáticamente por el servidor (identificadores únicos, marcas de tiempo y estados iniciales por defecto).

2. Deben generarse en la capa de persistencia (Repository) o de negocio (Service). No deben ser provistos por el cliente para evitar riesgos de seguridad y manipulación de datos (como suplantación de IDs, manipulación de fechas históricas o condiciones de carrera).

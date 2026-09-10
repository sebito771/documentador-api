# Documentación del Proyecto EasyDocs

#### 1.1. Ratelimiter
| Nombre | Responsabilidad | Funciones Clave / Lógica |
| :--- | :--- | :--- |
| `Ratelimiter` | Limitar el número de solicitudes por minuto (RPM) y tokens por minuto (TPM) | `__init__`, `allow_request` |

#### 1.2. Redis
| Nombre | Responsabilidad | Funciones Clave / Lógica |
| :--- | :--- | :--- |
| `redis` | Almacenar y recuperar datos en Redis | `ConnectionPool`, `Redis` |

#### 1.3. Script LUA
| Nombre | Responsabilidad | Funciones Clave / Lógica |
| :--- | :--- | :--- |
| `lua_script` | Verificar si se permiten los tokens solicitados y actualizar el estado de Redis | `redis.call` |

#### 2.1. Token Bucket
* El algoritmo Token Bucket utiliza un "bucket" que se llena con tokens a un ritmo determinado (refill_rate).
* El tamaño del "bucket" es determinado por el límite de tokens por minuto (max_capacity).
* Cuando se solicita un token, se verifica si hay suficientes tokens en el "bucket". Si no hay suficientes, se espera hasta que el timeout expire.

#### 2.2. Validaciones
* Se verifica si el número de solicitudes por minuto (RPM) y tokens por minuto (TPM) están dentro de los límites establecidos.
* Se verifica si el timeout ha expirado antes de permitir la solicitud.

#### 3.1. Errores en Redis
| Estado | Constante de Error | Razón/Condición |
| :--- | :--- | :--- |
| 500 | `redis_error` | Error al conectar a Redis o ejecutar la script LUA |

#### 3.2. Errores en la Lógica Central
| Estado | Constante de Error | Razón/Condición |
| :--- | :--- | :--- |
| 500 | `logic_error` | Error en la lógica central del Ratelimiter |

#### 4.1. Crear un objeto Ratelimiter
* Se debe proporcionar el número de solicitudes por minuto (RPM) y tokens por minuto (TPM) como parámetros.
* Se debe proporcionar el prefijo para las llaves en Redis.

#### 4.2. Permitir una solicitud
* Se debe llamar al método `allow_request` con el número de tokens solicitados como parámetro.
* El método devolverá `True` si se permiten los tokens solicitados, o `False` si no se permiten.

#### 4.3. Configuración de Redis
* Se debe configurar la URL de Redis en el entorno de ejecución.
* Se debe configurar la conexión a Redis en el objeto Ratelimiter.

### 1. Componentes y Servicios
| Nombre | Responsabilidad | Lógica Clave/Funciones |
| --- | --- | --- |
| `ChunkingService` | Dividir código en chunks manejables | `create_chunks`, `estimate_tokens` |

#### 2.1. `create_chunks`
*   Recibe una lista de archivos (`files`) y crea chunks de acuerdo a la configuración de tamaño máximo (`max_chunk_size`) y cantidad de archivos por chunk (`max_files_per_chunk`).
*   Utiliza un bucle para iterar sobre los archivos y determinar si debe crear un nuevo chunk o agregar el archivo actual al chunk en curso.
*   Si el chunk actual supera el tamaño máximo o la cantidad de archivos permitidos, se finaliza y se agrega a la lista de chunks.
*   La función devuelve la lista de chunks finalizados.

#### 2.2. `_finalize_chunk`
*   Recibe los datos de un chunk en curso y calcula el contenido combinado de los archivos en el chunk.
*   Utiliza un bucle para concatenar el contenido de cada archivo y agregar un encabezado con la información del archivo.
*   Calcula el hash del contenido combinado y agrega la información del chunk a la lista de chunks.

#### 2.3. `estimate_tokens`
*   Recibe un texto y devuelve una estimación burda de tokens (aproximadamente 4 caracteres por token).
*   Utiliza la función `len` para calcular la longitud del texto y divide por 4 para obtener la estimación de tokens.

### 3. Manejo de Errores y Excepciones
| Estado | Constante de Error | Razón/Condición |
| --- | --- | --- |
|  |  | No hay archivos para crear chunks |

### 4. Guía de Integración / Uso
*   Para utilizar el servicio de chunking, crea una instancia de `ChunkingService` y configura los parámetros de tamaño máximo y cantidad de archivos por chunk.
*   Llama a la función `create_chunks` pasando la lista de archivos y la configuración deseada.
*   La función devuelve la lista de chunks finalizados, que pueden ser procesados y almacenados según sea necesario.

### 5. Reglas de Negocio
*   Los chunks deben tener un tamaño máximo de `max_chunk_size` caracteres.
*   Cada chunk debe contener un máximo de `max_files_per_chunk` archivos.
*   La estimación de tokens debe ser aproximadamente 4 caracteres por token.

#### 1. Overview / Definición
El `DocumentationOrchestrator` es un orquestador principal que coordina varias tareas para generar documentación a partir de archivos ZIP. Se encarga de la extracción de archivos, chunking, cache por hash, generación de documentación y consolidación de resultados.

El Documentador IA es un servicio que utiliza Inteligencia Artificial para generar documentación técnica a partir de código fuente. Se enfoca en analizar y explicar la lógica detrás del código, proporcionando una visión clara y concisa de la estructura y funcionalidad del sistema.

#### 2. Arquitectura de Componentes
| Nombre | Responsabilidad | Lógica Clave/Funciones |
| --- | --- | --- |
| `ChunkingService` | Crear chunks de archivos | `create_chunks` |
| `CacheService` | Manejar cache por hash | `get_stats` |
| `DocumentadorIA` | Generar documentación | `detect_language` |
| `Ratelimiter` | Limitar velocidad de procesamiento | (no implementado) |

| Nombre | Responsabilidad | Lógica Clave / Funciones |
| --- | --- | --- |
| `DocumentadorIA` | Servicio principal que gestiona la generación de documentación | `__init__`, `generar`, `detect_language`, `build_system_prompt`, `build_user_prompt` |
| `Ratelimiter` | Gestiona el límite de tokens para cada modelo | `allow_request` |
| `get_groq_client` | Obtiene el cliente de Groq | - |
| `get_prompts` | Obtiene las plantillas de promt | - |
| `models` | Almacena la configuración de los modelos | - |

#### 3. Lógica Central y Validaciones
El `DocumentationOrchestrator` tiene varias lógicas clave:

*   **Procesamiento de ZIP**: El método `process_zip` es el corazón del orquestador. Se encarga de procesar un ZIP y generar documentación consolidada.
*   **Extracción de archivos**: El método `_extract_files` extrae archivos del ZIP usando el servicio existente.
*   **Chunking**: El método `create_chunks` crea chunks de archivos utilizando el servicio de chunking.
*   **Cache por hash**: El método `get_stats` devuelve estadísticas del cache.
*   **Generación de documentación**: El método `detect_language` detecta el idioma del proyecto y el método `consolidate_documentation` consolida la documentación.

Reglas de negocio:

*   **Tamaño máximo de archivo**: El tamaño máximo de archivo es de 10 MB.
*   **Número máximo de archivos**: El número máximo de archivos es de 50.
*   **Idioma del proyecto**: El idioma del proyecto se detecta automáticamente o se puede forzar vía parámetros.

El Documentador IA sigue un flujo de trabajo que incluye:

1. **Deteción de idioma**: El servicio detecta el idioma del código fuente utilizando la función `detect_language`.
2. **Generación de promt**: El servicio genera un promt para el modelo de IA utilizando la función `build_system_prompt` y `build_user_prompt`.
3. **Gestión de límite de tokens**: El servicio verifica si el límite de tokens para el modelo se ha alcanzado utilizando la función `allow_request`.
4. **Llamada a la API de IA**: El servicio llama a la API de IA para obtener la respuesta utilizando la función `generar`.
5. **Limpieza de la respuesta**: El servicio limpia la respuesta de la API para eliminar etiquetas de razonamiento innecesarias.

#### 4. Manejo de Errores y Excepciones
| Estado | Constante de Error | Razón/Condición |
| --- | --- | --- |
| 400 | `ARCHIVO_DESALIMENTADO` | El archivo ZIP es demasiado grande. |
| 400 | `NO_SE_ENCONTRARON_ARCHIVOS` | No se encontraron archivos válidos para documentar. |
| 500 | `ERROR_DE_DETECCIÓN_DE_IDIOMA` | Error al detectar el idioma del proyecto. |

| Estado | Constante de Error | Razón/Condición |
| --- | --- | --- |
| 400 | `NOMBRE_CODIGO_ERROR` | Error de detección de idioma |
| 500 | `NOMBRE_CODIGO_ERROR` | Error de llamada a la API de IA |

#### 5. Guía de Integración / Uso
Para utilizar el `DocumentationOrchestrator`, debes proporcionar un ZIP y los parámetros necesarios. El orquestador se encargará de procesar el ZIP y generar documentación consolidada.

**Parámetros**

*   `zip_content`: Bytes del archivo ZIP.
*   `doc_type`: Tipo de documento (markdown, pdf, word).
*   `extra_requirements`: Requisitos adicionales.
*   `zip_service`: Instancia de ZipService (inyectada para testing).
*   `language`: Idioma del proyecto (opcional).

**Respuesta**

La respuesta del `DocumentationOrchestrator` es un diccionario que contiene la documentación consolidada, metadata y errores.

Para utilizar el Documentador IA, simplemente proporciona el código fuente y el idioma deseado. El servicio se encargará de generar la documentación técnica correspondiente.

**MODO FRAGMENTO ACTIVO**

1. **PROHIBIDO**: No generes títulos de nivel 1 (#), introducciones, alcances ni índices.
2. **ENFOQUE**: Comienza directamente con el análisis técnico de los archivos proporcionados.
3. **JERARQUÍA**: Usa títulos de nivel

#### 1. Componentes y Servicios
| Nombre | Responsabilidad | Lógica Clave/Funciones |
| --- | --- | --- |
| `_parse_sections` | Procesar secciones de código fuente | `split`, `strip`, `try`-`except` |
| `_process_chunks` | Procesar chunks de código fuente | `enumerate`, `logger.info`, `cache_service` |
| `_consolidate_documentation` | Consolidar documentación de múltiples chunks | `defaultdict`, `re.split`, `re.sub` |

#### 2. Lógica Central y Validaciones
La lógica central del código se encuentra en la función `_parse_sections`, que procesa secciones de código fuente y extrae información relevante. La función `_process_chunks` se encarga de procesar chunks de código fuente y generar documentación.

La lógica de validación se encuentra en la función `_parse_sections`, que utiliza `try`-`except` para manejar errores durante el procesamiento de secciones.

#### 3. Manejo de Errores y Excepciones
| Estado | Constante de Error | Razón/Condición |
| --- | --- | --- |
| Error | `logger.warning` | Error parseando sección |
| Error | `logger.error` | Error procesando chunk |

La función `_parse_sections` utiliza `logger.warning` para manejar errores durante el procesamiento de secciones. La función `_process_chunks` utiliza `logger.error` para manejar errores durante el procesamiento de chunks.

#### 4. Guía de Integración / Uso
Para integrar este código en un proyecto, es necesario:

1. Importar las funciones `_parse_sections`, `_process_chunks` y `_consolidate_documentation`.
2. Llamar a la función `_parse_sections` con un string de código fuente como argumento.
3. Llamar a la función `_process_chunks` con un lista de chunks de código fuente como argumento.
4. Llamar a la función `_consolidate_documentation` con una lista de resultados de procesamiento de chunks como argumento.

### Reglas de Negocio
* La función `_parse_sections` debe procesar secciones de código fuente y extraer información relevante.
* La función `_process_chunks` debe procesar chunks de código fuente y generar documentación.
* La función `_consolidate_documentation` debe consolidar documentación de múltiples chunks.

### Observaciones
* El código utiliza una estructura de datos `defaultdict` para almacenar secciones de código fuente.
* El código utiliza una función `re.split` para separar secciones de código fuente.
* El código utiliza una función `re.sub` para reemplazar contenido en secciones de código fuente.
* El código utiliza una función `logger` para manejar errores y mensajes de log.

### para cada componente o archivo analizado.
4. **CONTINUIDAD**: Redacta el contenido como si fuera un capítulo intermedio de un libro técnico.
5. **SÍNTESIS**: Si hay lógica repetida entre archivos del mismo fragmento, agrúpalos en una sola explicación.

#### 2.1.1. Definición de Modelos
El archivo `ai/models.py` define un conjunto de modelos de inteligencia artificial utilizando un formato de diccionario. Los modelos están identificados por sus nombres y tienen asociados un identificador (`id`) y un valor de tiempo de procesamiento por milla (`tpm`).

#### 2.1.2. Componentes y Servicios
| Nombre | Responsabilidad | Lógica Clave/Funciones |
| :--- | :--- | :--- |
| `chunking` | Modelo de procesamiento de texto | `id`: "llama-3.1-8b-instant", `tpm`: 14400 |
| `final_doc` | Modelo de procesamiento de texto | `id`: "openai/gpt-oss-120b", `tpm`: 8000 |
| `fallback` | Modelo de procesamiento de texto | `id`: "qwen/qwen3-32b", `tpm`: 6000 |
| `emergency` | Modelo de procesamiento de texto | `id`: "openai/gpt-oss-20b", `tpm`: 8000 |

#### 2.1.3. Manejo de Errores y Excepciones
No hay errores o excepciones definidos en este archivo.

#### 2.1.4. Guía de Integración / Uso
Para integrar estos modelos en una aplicación, se debe acceder a los valores de `id` y `tpm` para cada modelo. La elección del modelo depende del contexto de la aplicación y de las necesidades de procesamiento de texto.

### 2.2. Observaciones
* Los modelos están definidos como un conjunto de diccionarios, lo que permite una fácil lectura y modificación de los valores.
* La elección del modelo depende del contexto de la aplicación y de las necesidades de procesamiento de texto.
* No hay errores o excepciones definidos en este archivo.


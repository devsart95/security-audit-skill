# Caza de agotamiento de recursos y disponibilidad

#### Cuándo usar este archivo

Se usa cuando pedidos, mensajes, archivos, estado de inquilino o trabajo de agente que no son de
confianza pueden consumir CPU, memoria, disco, conexiones, slots de worker, APIs pagas o capacidad
de cola, o pueden trabar (deadlock) o tumbar un servicio compartido. Este dominio separa una
vulnerabilidad de disponibilidad que se puede revisar en la fuente de un problema de performance
general. **Nunca se valida estresando un servicio vivo o compartido.**

Para defectos de integridad de memoria va `MEMORY-SAFETY-AND-BINARY.md`. Un error fatal alcanzable
pertenece a este archivo por el impacto compartido, aunque el parser de abajo lo cubra otro.

## Disciplina central (incluir en cada prompt de agente de este dominio)

```
- Exigir un camino de entrada a costo, un tope efectivo que falta y un impacto sobre otro usuario, un servicio compartido, una función de seguridad o el gasto del dueño. El trabajo que se limita solo dentro del proceso de quien lo pide no es una vulnerabilidad de servicio.
- Que falte un rate limit no alcanza. Revisar los topes de cuerpo/mensaje/archivo, la concurrencia, las colas, los deadlines, las restricciones de la base, los gateways aguas arriba y la quota por inquilino antes de declarar un camino sin tope.
- No correr pruebas de estrés, saturación ni producción. Usar análisis asintótico, fixtures chicos de borde, llamadas pagas simuladas (mocked), límites locales estrictos de recursos y pruebas deterministas de cancelación.
- Declarar el costo del atacante, el trabajo del servicio, la persistencia, el alcance y la recuperación. Una sola entrada acotada con efecto superlineal o persistente sobre lo compartido es otra cosa que volumen sostenido.
- Usar `confirmed` para las fallas de tope visibles en la fuente y demostradas de forma segura. Usar `needs_validation` cuando los topes aguas arriba, la topología desplegada, el autoscaling, la quota paga o el comportamiento de recuperación queden fuera del repositorio.
```

Nuestro terreno: el recurso compartido real es la VM (CPU, memoria, disco) y la cuota de la
herramienta. El aire del canal es producción interna: **no se prueba un corte contra él**.

## Clases de amplificación computacional (subagent_type: `general`)

**Parsing, matching o evaluación superlineal**
Una entrada chica y aceptada dispara backtracking catastrófico de expresiones regulares, parsing
anidado, validación recursiva, evaluación simbólica, recorrido de grafo, expansión de plantillas o
comportamiento adversario de ordenamiento/hash. Derivar la profundidad y la cardinalidad aceptadas y
la complejidad, y después demostrar una curva de crecimiento acotada localmente.
Nuestro terreno: los parsers que leen las listas del reproductor, los catálogos del receptor y los
JSON de config, más los campos de texto que entran por la UI. Un nombre de fuente o de canal con la
forma justa puede volver cuadrático el parseo.

**Amplificación por descompresión y representación**
Una entrada comprimida, dispersa, anidada, con alias o codificada se expande muy por encima del
tamaño de transferencia o de archivo que se chequeó. Verificar los topes después de cada expansión y
en cada etapa del parser, incluidos archivos comprimidos, imágenes, fuentes, documentos estructurados
y tablas de compresión de protocolo.
Nuestro terreno: las subidas y la mediateca, lo que baja un `wget` propio y lo que se descomprime al
preparar un artefacto.

**Amplificación de consultas a la base y aguas abajo**
Un pedido chico genera escaneos amplios, joins patológicos, fan-out, orden/agregación sin tope o
muchas llamadas aguas abajo porque la profundidad de consulta, la cardinalidad de filtros, la
paginación o los campos de expansión no tienen tope. Confirmar que la autorización no habilita a
propósito el mismo alcance de recursos.
Nuestro terreno: los listados y exportaciones de la UI contra la base del Core, y las llamadas a la
API del Core con parámetros que controla quien pide.

## Clases de acumulación de recursos (subagent_type: `general`)

**Buffering y cardinalidad sin tope**
Cuerpos, streams fuera de orden, subidas, sesiones, claves de caché únicas, etiquetas de métricas,
campos de log, suscripciones o trabajos pendientes que se acumulan sin topes por ítem ni agregados.
Buscar la limpieza y el vencimiento al desconectar, al vencer el timeout, al cancelar y al parsear a
medias.
Nuestro terreno: las conexiones al Core, las sesiones de la UI, el estado en memoria del reproductor
y lo que se guarda por conexión en los contenedores.

**Fuga de descriptores de archivo, handles y recursos temporales**
Trabajo mal formado o cancelado que se saltea la limpieza y retiene sockets, archivos, cursores de
base, timers, subprocesos, archivos temporales o referencias a objetos. Confirmar que la fuga se
repite en iteraciones locales acotadas y que afecta a un pool compartido.
Nuestro terreno (esto ya lo medimos): el **bucle de listas del reproductor** deja memoria que crece,
y la medición de RSS en ese bucle es exactamente el tipo de evidencia que pide esta clase —
repetible, local y con datos de prueba. La fuga se mide contra un canal de prueba, **nunca contra el
canal al aire**.

**Trabajo que queda suelto después de cancelar**
El timeout del cliente, la desconexión, el trabajo cancelado o la autorización fallida devuelven el
control pero dejan corriendo trabajo de base, de modelo, de red o de worker. Seguir la propagación de
la cancelación y del deadline por cada capa.
Nuestro terreno: cuando se corta un `wget`, cuando se cierra la pestaña de la UI a mitad de una
operación, cuando el timeout del borde del túnel corta la conexión y cuando un script propio recibe
una señal. ¿El subproceso de FFmpeg queda vivo?

## Clases de quota y planificación (subagent_type: `general`)

**Desequilibrio de trabajo antes de autenticar**
Parsing caro, búsqueda de clave, criptografía, descompresión o pedidos externos que ocurren antes de
la autenticación y del primer tope de tamaño/rate. Comparar el esfuerzo mínimo de quien pide con el
costo del servicio compartido y revisar los topes aguas arriba.
Nuestro terreno: la UI tiene login; todo lo que corre antes de esa barrera, alcanzable por el túnel,
es lo que hay que mirar. También la ingesta: RTMP/SRT publicados en `0.0.0.0` no deberían hacer
trabajo caro antes de exigir la clave de publicación.

**Huecos en el alcance y el reinicio de la contabilidad de quota**
La contabilidad usa IP, ruta, inquilino, prefijo de clave, ID de tarea u otra dimensión que el
atacante puede influir, lo que permite que el trabajo de un principal se escape de su presupuesto
previsto o consuma la asignación de otro. Revisar overflow de enteros, carreras distribuidas,
reintentos, reconexiones y cambio de cuenta.
Nuestro terreno (caso concreto): la **quota semanal compartida de Claude Code**. La contabilidad de
esa quota es por cuenta, no por agente ni por repo, así que una corrida desbocada de cualquier agente
se come la semana de todos. Por eso la regla del terreno: **la auditoría misma no usa esa quota**; si
un paso la necesita, se propone y se espera.

**Inanición de workers, pools y prioridad**
Trabajos de baja prioridad o controlados por el atacante que retienen locks, workers, pools de base,
turnos del event loop o prioridad del scheduler que necesitan usuarios ajenos. Exigir un camino que
se saltee la equidad de cola/concurrencia o que retenga un slot más allá de su deadline.
Nuestro terreno: los dos contenedores nuestros y la VM como pool compartido de CPU, memoria y disco.
Que un proceso se coma el disco o la memoria deja sin trabajo al otro.

## Clases de falla y recuperación (subagent_type: `general`)

**Error fatal alcanzable o deadlock**
Una entrada no confiable llega a `panic`, abort, aserción fatal, excepción no manejada, salida de
proceso, ciclo de locks o bucle infinito en un proceso compartido. Confirmar el alcance del
supervisor y si queda caído un worker o el servicio entero. Un worker aislado que se reinicia puede
bajar el impacto, pero no borra el defecto.
Nuestro terreno: el Core datarhei y nuestra UI son servicios compartidos; un crash del Core corta el
aire. Por eso el canal al aire es **producción interna** y no se prueba un corte contra él: se prueba
con un canal de prueba.

**Tormenta de reintentos y amplificación por fail-open**
Timeouts, errores de dependencia, mensajes procesados a medias o chequeos de salud fallidos disparan
reintentos sincronizados o sin tope, sin jitter, sin techo, sin circuit breaking ni deduplicación.
Verificar que una sola fuente de falla acotada pueda crear trabajo agregado persistente.
Nuestro terreno: los `wget` propios contra el servidor de LocalNet, el `watchdog` (corre por timer
cada 5 minutos) y los reintentos de FFmpeg. Un `wget` que reintenta sin tope llena `/tmp`, y `/tmp`
lleno rompe a los `wget`.

**Registro envenenado y bloqueo de cabeza de cola**
Un registro o mensaje mal formado falla una y otra vez al frente de una cola, partición, escaneo de
arranque, migración o bucle de recuperación compartido. Revisar la política de saltear/cuarentena, los
offsets y si otros inquilinos comparten la unidad bloqueada.
Nuestro terreno: una lista de canales con una fuente rota que queda al frente, un catálogo del
receptor que se pisa, un archivo a medias que traba el arranque de un script.

**Recuperación insegura y rollback de capacidad**
Un reinicio, una restauración, un fallback o un camino de limpieza reconstruye estado sin tope, ignora
las quotas vigentes o restaura la misma entrada que repite la falla de inmediato. La corrección de la
recuperación es parte de la disponibilidad.
Nuestro terreno: el `watchdog` y las unidades de usuario que rearman túneles y contenedores, y las
restauraciones de la config del Core. ¿Un reinicio vuelve a levantar el mismo estado sin tope que
causó el problema?

## Movimientos universales (aplican a todo lo de arriba)

```
- Armar una tabla de entrada a recurso: tamaño/cardinalidad aceptada más temprana, trabajo antes de autenticar, fan-out aguas abajo, persistencia, pool compartido, quién es dueño del tope y de la limpieza, recuperación.
- Comparar los topes agregados con los topes por objeto. Diez mil ítems válidos de un byte pueden esquivar un tope por mensaje y agotar igual el estado del inquilino o del proceso.
- Validar sólo en un fixture aislado con límites estrictos de CPU/memoria/tiempo y pocos puntos de crecimiento. Simular (mock) las llamadas externas y pagas y frenar apenas el tope faltante o la cancelación quede a la vista.
```

Nuestro terreno: el fixture corre en la VM, sólo con datos de prueba, sin red externa y sin instalar
nada; y no se corre nada que pueda cortar el aire del canal.

## Reglas de validación (aplican antes de reportar cualquier hallazgo de acá)

1. Nombrar la entrada no confiable, el trabajo de quien pide, la amplificación del servicio o el
   recurso retenido, el radio de impacto compartido y la recuperación. Los topes faltantes sin
   impacto compartido concreto son endurecimiento, no hallazgo.
2. Confirmar que ningún tope aguas arriba, del parser, de la cola, del inquilino o del framework,
   visible en la fuente, evita ese camino. Los controles desplegados que no se conocen exigen
   `needs_validation`.
3. Para el comportamiento superlineal, establecer la complejidad aceptada y el crecimiento local
   acotado. Para las fugas, mostrar retención repetible después del punto donde debería limpiarse.
   Para los caminos fatales, identificar el aislamiento de proceso/supervisor.
4. Priorizar por trabajo bajo de quien pide, alcanzabilidad sin autenticar, alcance entre inquilinos,
   persistencia y mala recuperación; **no validar con impacto sobre la disponibilidad**.
5. Devolver `confirmed` sólo con prueba local segura y un efecto compartido con sentido. Devolver
   `needs_validation` con el tope aguas arriba, la topología, la quota o la observación de
   recuperación exacta que el dueño tenga que revisar.

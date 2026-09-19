# Caza de memoria, binarios y kernel

#### Cuándo usar este archivo

Se usa cuando el objetivo procesa bytes que no son de confianza en un contexto inseguro de memoria o
privilegiado: C/C++/Objective-C, `unsafe` de Rust, FFI, kernel modules y drivers, parsers y
decodificadores, daemons de red, firmware, loaders de binarios, runtimes de lenguaje y JITs. Este
archivo cubre integridad del proceso, seguridad de memoria, fronteras de ABI y comportamiento del
loader.

**Nuestro único objetivo de esta clase es un firmware de fabricante: el receptor satelital GX6622,
CPU C-SKY, con binarios propietarios (`dvbapp`, `av.ko`, y el updater con firma RSA), y algún parser
propio. Regla clave de nuestro terreno: ese binario SE LEE Y SE ANALIZA, NO SE EJECUTA** — no hay
sandbox del sistema operativo. Entonces **la evidencia de esta clase es análisis estático**
(desensamblado, `strings`, `readelf`/`nm`, lectura de la fuente cuando la tengamos), y **todo lo que
exija ejecutar el binario queda como `needs_validation`**, declarando el bloqueo exacto. Un binario
que sea NUESTRO sí se puede correr, pero sólo con datos de prueba y sin red.

Elegir las clases que apliquen en la fase 1 y dividir los objetivos grandes por parser, allocator y
tiempo de vida, FFI, concurrencia, loader, runtime o interfaz privilegiada.

## Disciplina central (incluir en cada prompt de agente de este dominio)

```
- Re-derivar cada tope y cada tiempo de vida desde las entradas que controla el atacante y desde todos los que llaman. Validar contra el peor caso aceptado, no contra un vector de prueba típico.
- No se ejecuta el binario del fabricante. Sin sandbox del sistema operativo, dvbapp, av.ko y el updater con firma RSA se leen y se analizan (desensamblado, strings, readelf/nm), pero no se corren. La evidencia de esta clase es análisis estático; lo que exija ejecutarlos queda needs_validation.
- Un panic, un hallazgo de sanitizer o un crash prueban un defecto sólo cuando una entrada no confiable realista llega hasta ahí. No se infiere corrupción de memoria, ejecución de código ni impacto compartido de disponibilidad a partir de una etiqueta sola.
- Sin ejecución no hay sanitizer, fuzzing ni depurador sobre el firmware: son justamente las herramientas que exigen correr. Para código NUESTRO sí valen un harness local con sanitizers, pruebas deterministas de concurrencia, fuzz targets existentes y clasificación de fallas con depurador. Frenar después de probar la invariante violada y el impacto observable; no desarrollar técnicas post-corrupción.
- Ensamblado, código JIT, allocators propios, accesos intra-objeto y librerías ajenas pueden escapar a la cobertura del sanitizer. Identificar qué instrucciones relevantes están instrumentadas.
- Clasificar `confirmed` sólo después de que la evidencia en la fuente (y, si es nuestro, la validación local acotada) establezcan el defecto y su efecto. Usar `needs_validation` cuando queden sin resolver hechos de ABI, allocator, arquitectura, feature, despliegue o alcanzabilidad.
```

## Clases de límites, enteros y representación (subagent_type: `general`)

**Lectura o escritura fuera de límites (out-of-bounds read/write, buffer overflow)**
Una longitud, un offset, un índice o un terminador llega a un buffer fijo o asignado sin el tope
correcto. Recalcular el headroom disponible después de prefijos, alineación, padding y terminadores.
Revisar la capacidad del origen y del destino, y si una entrada corta se lee antes de confiar en la
longitud declarada.
Nuestro terreno: los parsers de `dvbapp` y sus decodificadores, más los parsers propios que leen los
catálogos del receptor y los JSON de config. La evidencia es estática: el headroom se recalcula a
mano sobre el desensamblado. Correr el binario para confirmarlo queda `needs_validation`.

**Desborde, subdesborde, truncamiento y signo en enteros**
Revisar la aritmética que controla el atacante antes de operaciones de asignación, copia, bucle,
indexado y puntero. Los patrones de más impacto incluyen `a - b` con `b > a`, `count * element_size`,
sumas cerca del máximo del tipo, valores negativos convertidos a sin signo, longitudes de 64 bits
recortadas a campos de 32 bits y valores centinela como `-1` que se vuelven un tamaño enorme.
Confirmar qué representación chequeada se usa después.
Nuestro terreno: la aritmética de campos de longitud en los parsers del firmware (análisis estático)
y en nuestros parsers de catálogos. La ejecución para demostrarla queda `needs_validation`.

**Confusión de unidades y de profundidad de punteros**
El código mezcla bytes, elementos, unidades de código, páginas, palabras, unidades de wire o tamaños
de elementos de punteros anidados. Comparar la unidad en el parseo, la validación, la asignación, la
frontera de API y la copia. Un chequeo de límites que usa la misma unidad equivocada que la asignación
sigue estando mal.
Nuestro terreno: los formatos del receptor (secciones, offsets, tablas de canales) leídos a mano
sobre el binario; el análisis es estático y la ejecución queda `needs_validation`.

**Datos sin inicializar o inicializados a medias**
Un buffer, padding, campo de struct o capacidad de vector se devuelve, se compara, se hashea, se
serializa o cruza una frontera de confianza antes de inicializarse. Exigir un consumidor observable y
una longitud de salida realista; la asignación en stack por sí sola no es divulgación.
Nuestro terreno: buffers de los parsers del firmware (análisis estático) y de nuestros scripts. Sin
ejecutar el binario, la divulgación efectiva queda `needs_validation`.

## Clases de tiempo de vida, tipos y concurrencia (subagent_type: `general`)

**Use-after-free, vista obsoleta y doble free**
Los dueños se liberan mientras callbacks, colas de espera, timers, iteradores, slices prestados o
punteros crudos cacheados todavía pueden usarlos. Revisar cada camino de error, cancelación, cierre y
realloc. Para anclas de notificación embebidas, cada camino de free tiene que drenar o desenganchar a
todos los observadores.
Nuestro terreno: el manejo de buffers y callbacks en `dvbapp` y en los drivers del receptor, leído de
forma estática. Un use-after-free confirmado por ejecución queda `needs_validation`.

**Confusión de tipos y downcast inválido**
Un tag, una vtable, un discriminador de union, un tipo de objeto o un handle ajeno se chequea distinto
de la representación que se lee después. Buscar casts dinámicos sin chequear, tags obsoletos tras
reusar memoria y tipos serializados cuyo elemento validado difiere del elemento consumido. Confirmar
localmente una lectura o escritura de tipo equivocado sin extender la prueba más allá de la invariante
violada.
Nuestro terreno: las estructuras de objetos del firmware y de nuestros parsers. La confirmación local
sólo es posible si el código es NUESTRO; sobre el firmware queda análisis estático y `needs_validation`.

**Carreras de conteo de referencias y de propiedad**
Un retain/release no atómico, un chequeo seguido de un uso sin lock, o una propiedad inconsistente
entre hilos pueden liberar o mutar un objeto durante el acceso. Comparar los caminos rápido, de error,
de apagado y de compatibilidad contra las mismas reglas de lock y propiedad.
Nuestro terreno: concurrencia en `dvbapp` y `av.ko` (kernel module), analizada de forma estática.
Confirmar una carrera exige correrla: `needs_validation`.

**Carreras sobre estado compartido y TOCTOU**
Streams de parser concurrentes, cachés globales, inicialización perezosa, manejadores de señal y
desmontaje de recursos pueden invalidar topes, políticas o punteros establecidos antes. Verificar la
carrera con un cronograma local repetible, una barrera o un thread sanitizer; un entrelazado
hipotético sin una transición de estado relevante para la seguridad queda `needs_validation`.
Nuestro terreno: sin ejecutar el firmware no hay sanitizer ni cronograma repetible; lo que se afirme
sobre el binario del fabricante queda como análisis estático y `needs_validation`.

**Orden de locks, deadlock e inanición**
Operaciones alcanzables desde afuera toman locks en orden inconsistente o los retienen a través de
callbacks y E/S bloqueante. Reportar bajo disponibilidad sólo cuando una entrada acotada pueda
detener el progreso compartido; si no, registrarlo para arreglar como defecto de concurrencia.
Nuestro terreno: el lock del kernel module `av.ko` y los locks internos de `dvbapp`, leídos de forma
estática. Reproducir un deadlock exige ejecutar: `needs_validation`.

## Clases de FFI y ABI (subagent_type: `general`)

**Desajuste del contrato de puntero-longitud y de propiedad**
Quien llama y quien es llamado no coinciden en quién asigna, libera, fija (pin) o muta un buffer,
cuánto tiempo sigue válido un puntero, o si una longitud es en bytes o en elementos. Trazar los dos
lados de cada `extern`, binding CGo/JNI/Python/nativo y wrapper generado. Revisar null, longitud cero,
aliasing y retención de callbacks.
Nuestro terreno: la frontera entre `dvbapp` en espacio de usuario y `av.ko` en kernel es una frontera
de ABI con ioctl de por medio; se analiza de forma estática sobre el desensamblado y la lectura de
código. Confirmar en ejecución queda `needs_validation`.

**Desacuerdo de layout, alineación y enum**
El código ajeno recibe un struct, bitfield, registro empaquetado, firma de callback, ancho de entero,
enum o convención de llamada que difiere por arquitectura o bandera de build. Verificar `repr`,
packing, alineación, endianness y tipos propios del ABI. Un desajuste de declaración dentro del repo
se puede confirmar localmente; una implementación ajena opaca exige `needs_validation`.
Nuestro terreno: C-SKY es una arquitectura propia; el layout y la alineación se comparan entre los
headers que tengamos y el desensamblado. Confirmar en ejecución queda `needs_validation`.

**Violaciones de unwind, excepciones y afinidad de hilo**
Excepciones o panics cruzan un ABI que prohíbe el unwinding, callbacks corren después del desmontaje,
o APIs que exigen un hilo de runtime se invocan en otro. Revisar la conversión de errores y la
cancelación. Confirmar si el proceso aborta o queda el estado corrupto antes de asignar impacto.
Nuestro terreno: `dvbapp` es C sobre C-SKY; se lee de forma estática. La confirmación en ejecución
queda `needs_validation`.

## Clases de carga de binarios y runtime (subagent_type: `general`)

**Confianza en el orden de búsqueda de librerías, plugins y ejecutables**
Un proceso privilegiado carga una librería, plugin, imagen de runtime o helper desde una ruta que
puede escribir un principal de menos confianza, o resuelve un nombre pelado por un directorio de
trabajo o entorno influible por el atacante. Comparar la propiedad prevista de la instalación con cada
camino de búsqueda de fallback y compatibilidad. Un usuario que carga su propio plugin en su propio
proceso no es una violación de frontera.
Nuestro terreno: cómo el updater y el arranque del receptor resuelven las rutas de `dvbapp` y `av.ko`
desde el rootfs. Se lee de forma estática.

**Identidad del artefacto o atado a la firma faltante**
Un loader verifica un archivo o registro de metadata pero mapea otra imagen porque la resolución de
rutas, el reemplazo del archivo, los slices de arquitectura o los recursos embebidos no están atados
al chequeo. La autenticidad del canal de suministro va en `SUPPLY-CHAIN-AND-RELEASE.md`; esta clase
cubre la brecha local entre lo que se verifica y lo que se mapea.
Nuestro terreno: **el updater con firma RSA**. El chequeo es leer si la firma cubre exactamente la
imagen que después se mapea/escribe, y si hay una ventana entre verificar y escribir. Es análisis
estático del updater y de su formato; reproducirlo queda `needs_validation`.

**Metadatos de binario mal formados y manejo de relocaciones**
Offsets, conteos, secciones, relocaciones, símbolos, bytecode o metadata de depuración se confían
antes de los chequeos de rango, solapamiento y representación. Probar los parsers con fixtures locales
acotados y sanitizers. Separar la corrupción de memoria de un archivo mal formado rechazado de forma
segura.
Nuestro terreno: los parsers de metadata del firmware, que en nuestro caso se leen y se analizan, no
se ejecutan. Los fixtures locales corren sólo si el parser es NUESTRO; si no, análisis estático y
`needs_validation`.

**Consistencia del JIT y del código generado**
Validador, intérprete, optimizador y código generado no coinciden en tipos, límites, efectos
secundarios o tiempo de vida. Comparar los caminos optimizado y sin optimizar con la misma entrada
local. Confirmar un efecto sobre la integridad del proceso; una variación de salida que se queda dentro
de la semántica del lenguaje no es un hallazgo.
Nuestro terreno: si el receptor trae un intérprete/decodificador con JIT, se lee de forma estática; la
comparación de caminos exige ejecutar y queda `needs_validation`.

**Seguridad de descarga, recarga y desmontaje**
Punteros a función vivos, callbacks, hilos de worker o vistas de datos sobreviven a la descarga del
módulo o al reinicio del runtime. Revisar el apagado y la limpieza de una carga fallida con el mismo
detalle que el arranque.
Nuestro terreno: descargar y recargar `av.ko`, y el desmontaje de recursos de `dvbapp`. Se lee de forma
estática; la ejecución queda `needs_validation`.

## Clases de kernel e interfaz privilegiada (subagent_type: `general`)

**Topes de copia desde/hacia usuario y lecturas repetidas**
Una syscall, un ioctl, un driver o un parser de kernel deriva un dato confiable de la memoria de
usuario y después lee otra vez la misma dirección mutable. Copiar el pedido completo una sola vez o
revalidar la copia posterior. Auditar además los chequeos de tamaño, dirección y acceso en cada
primitiva de copia desde/hacia usuario.
Nuestro terreno: `av.ko` es un kernel module y su interfaz es ioctl. Se lee de forma estática; un
patrón de doble lectura confirmado en ejecución queda `needs_validation`.

**Ciclo de vida de objetos privilegiados y consistencia del despacho**
Objetos alcanzables desde afuera con retain/release desbalanceado, desmontaje sin drenar observadores,
índices de selector/tabla sin chequear, o caminos de compatibilidad duplicados que omiten una guarda.
Comparar cada camino de despacho y de free lado a lado.
Nuestro terreno: el manejo de objetos de `av.ko` y del dispositivo del receptor, leído de forma
estática. Confirmarlo en ejecución queda `needs_validation`.

**Interfaces poderosas con autorización de menos**
Un device node, socket de administración, helper o API de gestión valida la forma pero no la autoridad
de quien llama sobre el recurso. Establecer la propiedad y la alcanzabilidad reales de la interfaz; los
permisos o la política de sandbox fuera del repositorio hacen esto `needs_validation`.
Nuestro terreno: los device nodes y el socket de administración del receptor. Quién puede abrirlos se
observa en el rootfs (análisis estático); la alcanzabilidad real exige el equipo y queda
`needs_validation`.

## Movimientos universales (aplican a todo lo de arriba)

```
- Auditar los arreglos y los caminos duplicados que repiten la misma forma de fuente a sumidero. Un chequeo en un llamador, una arquitectura, un rol de protocolo, una bandera de feature o un camino de compatibilidad no protege a sus hermanos.
- Armar una tabla por cada parser o frontera FFI: longitud/tipo aceptado, representación chequeada, dueño de la asignación, consumidor, hilo y desmontaje. La mayoría de los hallazgos nativos son un solo desacuerdo en esa tabla.
- Usar corpus existentes y fixtures de borde chicos generados localmente. Guardar la salida exacta del sanitizer/runtime y la propiedad de la entrada que lo dispara; evitar consumo grande de recursos y cualquier objetivo vivo.
```

Nuestro terreno: sobre el firmware, "corpus y fixtures" se leen, no se ejecutan — el análisis es
estático (desensamblado, `strings`, `readelf`/`nm`) y lo que exija correr el binario queda
`needs_validation`. El harness local con sanitizers y fuzzing vale sólo para código NUESTRO, con datos
de prueba y sin red.

## Reglas de validación (aplican antes de reportar cualquier hallazgo de acá)

1. Establecer una entrada no confiable realista y la operación exacta que viola una invariante de
   límites, tipo, tiempo de vida, ABI, concurrencia, loader o autoridad.
2. Clasificar el efecto observable: lectura inválida, escritura inválida, alias obsoleto, objeto
   equivocado, salida sin inicializar, carga no autorizada de una imagen, deadlock o terminación
   segura del proceso. No afirmar un efecto más fuerte que el observado.
3. Correr el harness local, el test existente, el sanitizer o el fuzzer más angosto que haga falta
   para reproducir el efecto — **sólo si el código es NUESTRO**. Sobre el binario del fabricante no se
   ejecuta nada: la evidencia es estática y la ejecución queda `needs_validation`. Verificar la
   cobertura del sanitizer sobre la operación que falla y registrar las condiciones de
   arquitectura/build.
4. Para la concurrencia, usar un cronograma determinista o una traza de sanitizer. Para la carga de
   binarios, probar que la identidad chequeada difiere de la identidad mapeada y nombrar a quien
   escribe con menos confianza.
5. Devolver hallazgos `confirmed` sólo con la entrada exacta, la traza en la fuente y el resultado
   observado. Devolver `needs_validation` para un hecho sin resolver y específico de alcanzabilidad,
   ABI, build, despliegue o runtime, y declarar el chequeo acotado que hace falta.

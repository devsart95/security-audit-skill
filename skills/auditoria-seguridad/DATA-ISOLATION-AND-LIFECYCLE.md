# Datos y ciclo de vida

#### Cuándo usar este archivo

Usá este archivo cuando el objetivo guarde datos multiinquilino o con control de acceso, derive
copias de búsqueda, índice, caché o analíticas, emita enlaces a objetos, exporte o restaure
registros, migre esquemas, o prometa comportamiento de borrado, revocación y retención. Este dominio
sigue **un dato** a través de todas sus copias y transiciones de estado. Usá `ATTACK-CLASSES.md`
para el control de acceso a nivel de endpoint y `CLOUD-AND-DEPLOYMENT.md` para la política de
almacenamiento a nivel de proveedor.

**En nuestro terreno** no hay inquilinos de nube, pero sí datos con dueño, y son concretos:
`work/creds/` (700, un archivo por servicio con usuario, clave, token y notas), `work/firmas/` (la
firma de la escribana y los documentos sellados), `work/granos/` (las planillas de compra),
`restreamer/data/` (el storage del Core y su `config.json`, con su usuario y su clave) y los
`ACCESOS.md` y `config.json` de cada repo (credenciales de paneles IPTV, no versionados por diseño).
El "otro" que hay que tener en cuenta no es un inquilino: es **cualquiera que llegue por el túnel**
y **cualquier proceso o agente de la VM**. La regla transversal es una sola: las credenciales nunca
van a un repo, y **ningún valor de credencial va a un informe**.

Dividí los objetivos grandes por almacén principal, caché o búsqueda, almacenamiento de objetos o
blobs, analíticas y logs, exportación o respaldo, y borrado o revocación.

## Disciplina central (incluir en cada prompt de agente de este dominio)

```
- Un campo de inquilino o de dueño en un registro no es aislamiento. Encontrá la consulta, la clave,
  la ruta, la política o el control por fila que lo hace cumplir en cada camino de lectura y de
  escritura.
- Seguí las copias derivadas. Un dato principal sano puede volverse inseguro en la búsqueda, la
  caché, las analíticas, la exportación, las vistas previas, los logs, las réplicas y los respaldos,
  que tienen otras reglas de ACL y de retención.
- El borrado y la revocación son contratos de ciclo de vida. Revisá las copias actuales, históricas,
  cacheadas, indexadas, exportadas, restauradas y encoladas dentro de la frontera declarada por el
  producto.
- Una preferencia de privacidad o de retención no es automáticamente una vulnerabilidad de
  seguridad. Hace falta una frontera explícita de acceso a los datos o una garantía de borrado o
  revocación, y un lector o una operación posterior no autorizada.
- Usá `confirmado` con un linaje completo visible en la fuente y pruebas acotadas con datos falsos.
  Usá `needs_validation` cuando la política de almacenamiento externa, la retención, el
  comportamiento de la CDN, el atraso de una réplica o el acceso al respaldo no estén disponibles.
```

## Clases de ataque de aislamiento de inquilino y de objeto (subagent_type: `general`)

**Falta de aplicación del inquilino o del dueño**
Una lectura, actualización, borrado, listado, conteo o consulta masiva identifica un objeto sin
atarlo al inquilino o dueño autenticado, o confía en campos del cuerpo del pedido para aportar esa
identidad. Compará la búsqueda directa, la relación anidada, el trabajo de fondo, el camino de
administración, la importación y los caminos heredados.

> **Acá no hay inquilino: el dueño es uno y el "otro" llega por el túnel.** El equivalente exacto:
> una ruta de la UI que hace trabajo local (ffprobe, disco, FFmpeg) tiene que consultar el token
> contra el Core antes de hacerlo. Si alguna ruta se saltea esa consulta, un visitante sin sesión
> válida hace trabajo con nuestra autoridad. El trabajo es comparar los caminos entre sí: la ruta
> del panel, la vista previa, la API interna que alimenta al navegador y la ruta de salud pública.

**Colisión de clave compuesta y de espacio de nombres**
Las claves de caché, las rutas de objeto, la unicidad de la base, los ID de documento de búsqueda,
los archivos temporales o las claves de deduplicación omiten el inquilino o el entorno. Dos
principales pueden pisar o recuperar la misma clave lógica aunque los registros de la aplicación
tengan dueños distintos.

> **Acá se ve en los nombres:** los archivos de `work/tmp` (`tunel-core.log`, las salidas de build),
> los nombres de canal y de medio en el storage del Core, y los nombres de contenedor, imagen y
> unidad. El watchdog crea unidades transitorias **y reintenta**: dos creadores del mismo nombre al
> mismo tiempo es la colisión concreta a buscar.

**Desacuerdo entre la política y la consulta**
La política por fila, los alcances por defecto del ORM, los filtros de autorización y los clientes
crudos o que saltean controles aplican predicados distintos. Revisá los `joins`, los agregados, los
alias, las vistas, las transacciones, los clientes `unscoped` o de servicio, y los caminos de error
donde falta el contexto.

> **Acá el Core es la autoridad de sus datos y la UI no los lee por su cuenta.** El desacuerdo
> aparece cuando la UI sirve algo de su propia caché o de la cookie sin volver a preguntar (¿una
> cookie `core_token` presente alcanza para pintar un panel?), o cuando un script de `work/bin`
> escribe en `restreamer/data` por fuera del Core.

**Exceso de alcance en blobs y referencias firmadas**
Las claves de objeto, los ID de adjunto, los ID de versión, los enlaces compartidos o las URL
firmadas permiten operaciones o espacios de nombres más allá del acceso del principal que los
emitió, o siguen valiendo después de que cambia la ACL. Atá operación, objeto o versión exacta,
audiencia, vencimiento e inquilino.

> **No aplica hoy en nuestra VM:** no hay almacenamiento de objetos ni URL firmadas. El equivalente
> son las URL del Core (`/memfs/...`, el HLS por el nginx del 8899) y el storage montado de sólo
> lectura: qué se puede pedir sin sesión y qué queda cacheado en el navegador de quien miró.

## Clases de ataque de datos derivados y divulgación (subagent_type: `general`)

**Deriva de ACL en búsqueda, caché e índice**
La ACL o el ciclo de vida de un registro principal cambia sin invalidar una copia buscable,
cacheada, embebida, en miniatura, en RSS, de vista previa o de índice. Validá el filtrado al momento
de recuperar, y también en la ingesta del documento y en la invalidación.

> **Acá la copia derivada más clara es el storage del Core leído por el panel:** la UI monta el
> storage de sólo lectura y saca metadatos y miniaturas con su propio ffprobe. Si después el medio
> se borra o se cambia en el Core, ¿qué queda del lado de la UI (listado, miniatura, caché del
> navegador) y quién lo puede ver por el túnel?

**Analíticas, logs, trazas y diagnósticos como lectores alternativos**
Contenido privado o credenciales se emiten a sistemas con acceso más amplio, retención más larga o
mezcla de inquilinos. Confirmá la clase de dato y el lector real; los nombres de campo, los
identificadores públicos y el contenido sólo para operadores bajo la política prevista no alcanzan.

> **Acá ya se midió:** los logs de FFmpeg del Core traen la clave del memfs en texto plano (por eso
> todo log pasa por `redactSecrets` antes de ir a un test, un doc, un PR o un informe); los avisos
> del watchdog van a un DM de Discord, que es un sistema de afuera con retención propia; y
> `work/tmp/tunel-core.log` guarda la URL pública del túnel, que después lee el script de
> despliegue. Antes de reportar, nombrá el lector real y la clase de dato.

**Enumeración y oráculos por agregados**
Los conteos, los filtros, el ordenamiento, los errores, las restricciones de unicidad, los tiempos,
el comportamiento de notificación o los chequeos de existencia revelan estado de objetos o de
cuentas protegidas. Hace falta un predicado confidencial concreto y una distinción observable, no
una variación general de la respuesta.

> **Acá:** la ruta de salud pública ya es un oráculo de existencia y de estado del Core (si
> responde, la latencia, si el disco es legible, si hay FFmpeg instalado), y un mensaje de login que
> distinga "usuario inexistente" de "clave mal" es otro. Ojo con los tiempos como oráculo: en esta
> VM los sondeos al Core tienen latencias poco confiables (el caso del keep-alive está medido en el
> `AGENTS.md` del repo de la UI), así que una diferencia de milisegundos no es evidencia.

## Clases de ataque de exportación, respaldo, restauración y migración (subagent_type: `general`)

**Expansión del alcance de una exportación o un respaldo**
Una exportación, captura, paquete de portabilidad, informe o respaldo incluye otros inquilinos,
campos de objetos inaccesibles, datos borrados lógicamente, valores de secretos o historial por
encima del acceso de quien lo pide. Revisá la autorización ítem por ítem después de la selección, y
la autorización para descargar el artefacto final.

> **Es la clase que más aplica acá.** Cualquier cosa que se arme para salir de la VM —una copia de
> `restreamer/data/config.json`, un `.env`, un archivo de `work/creds/`, un zip de un repo, un
> adjunto a Discord— puede llevarse la clave de un panel, la del Core o la firma de la escribana. El
> `.gitallowed` de un repo es exactamente la lista de lo que se decidió versionar: leelo antes de
> dar por limpio un export, un commit o un adjunto.

**Expansión de autoridad en la importación y la restauración**
Una restauración o importación saltea las reglas de dueño, esquema, ACL, unicidad o validación, pisa
recursos existentes o recrea registros en un inquilino donde quien pide no puede escribir. Validá el
contenido del archivo como no confiable y autorizá la operación resultante, en vez de confiar en su
procedencia.

> **Acá:** restaurar un respaldo de la config del Core, importar una lista de canales, pisar
> `config.json`, reponer la config de un panel desde un archivo, o re-flashear un rootfs del
> receptor con un respaldo del `mtd7`. La pregunta: ¿el archivo restaurado se valida contra el
> estado actual antes de aplicarlo, o se aplica porque "es nuestro"?

**Confusión de valores por defecto y de propiedad en una migración**
Los registros viejos no tienen campos de inquilino, ACL o ciclo de vida, los ID incompatibles
colisionan, o un despliegue parcial hace que lectores nuevos y viejos apliquen valores por defecto
distintos. Revisá el rellenado, la lectura y escritura dobles, la compatibilidad, la vuelta atrás y
los caminos de migración retomados.

> **Acá:** un cambio de formato en el `config.json` del Core, un cambio de esquema en una planilla
> de compra, o dos agentes tocando el mismo repo (la regla del `AGENTS.md` es partir de
> `origin/main`). Los "registros viejos" son los canales o las entradas de config que dejó una
> versión anterior del panel: ¿qué lee la versión nueva cuando el campo no está?

**Deriva de frontera en respaldos y replicación**
Las claves de cifrado, las cuentas de almacenamiento, las réplicas entre regiones, los entornos de
restauración o las capturas de soporte tienen una identidad o un alcance de inquilino más amplio que
los datos principales. La fuente sólo puede confirmar la política del repo; el acceso alojado y la
retención necesitan `needs_validation`.

> **No hay réplicas ni capturas del proveedor.** Lo más parecido: los respaldos del receptor que
> terminan en la Mac de Justino, los archivos que se mandan por Discord, y cualquier copia que salga
> de la VM. **Si un respaldo sale de la VM, sale de nuestra frontera**, y ahí manda la regla de que
> las credenciales no salen de la VM.

## Clases de ataque de borrado, revocación y ciclo de vida (subagent_type: `general`)

**Salteo del borrado lógico y de la lápida**
La búsqueda directa, la búsqueda, el recorrido de relaciones, el enlace al objeto, el procesador de
fondo o la restauración ignoran el predicado del ciclo de vida y devuelven o actúan sobre un
registro borrado o revocado. Revisá si un identificador borrado lógicamente se puede volver a
registrar antes de que desaparezcan todas las referencias.

> **Acá:** borrar un canal o un medio en el Core y mirar qué queda: el archivo en el storage, el HLS
> cacheado, la miniatura del panel, la línea del log que todavía lo nombra, y el segmento que un
> navegador tiene en caché. Sumale el canal dado de baja y vuelto a crear con el mismo nombre.

**Autorización vieja y uso de copias derivadas**
Sacar una membresía, actualizar una ACL, retirar un consentimiento, revocar un secreto o bajar de
rol no invalida sesiones, cachés, suscripciones, trabajos o datos materializados que siguen
autorizando operaciones futuras.

> **Acá el caso canónico es la sesión.** Si se cambia la clave del Core o se da de baja un acceso a
> un panel: ¿las cookies ya emitidas siguen valiendo? ¿El token guardado en la cookie se revalida
> contra el Core en cada pedido, o alcanza con que la cookie esté presente? Ésa es la frontera
> exacta, y hay que verla en el código, no suponerla.

**Exceso de retención y de trabajo encolado**
El borrado termina en el almacén principal mientras los procesadores encolados, los reintentos, las
exportaciones, las analíticas o los artefactos generados recrean o retienen el dato más allá de la
frontera prometida. Buscá el borrado idempotente y la propagación de la lápida.

> **Acá:** el watchdog repone unidades y reintenta cada cinco minutos, los avisos quedan en Discord,
> los logs de `work/tmp` crecen, y un FFmpeg en curso puede seguir escribiendo el archivo de un
> medio ya borrado. Preguntá qué pasa con el proceso en vuelo, no sólo con el registro.

**La restauración reintroduce estado inválido**
Un respaldo, un deshacer, un recuperar o la recuperación de una réplica restaura datos,
credenciales, membresías o permisos que la política actual ya no permite. Re-autorizá el estado
restaurado y volvé a aplicar los cambios de ciclo de vida hechos después de la captura.

> **Acá:** restaurar un respaldo viejo de `config.json` o de `work/creds/` repone credenciales que
> ya se habían rotado, o un acceso que se había dado de baja. El respaldo no tiene fecha de
> vencimiento: el que lo aplica es el que tiene que revisar la vigencia.

## Movimientos universales (aplican a todo lo de arriba)

- Elegí un registro protegido y dibujá el camino de escritura principal, consulta, caché, índice,
  evento, exportación, respaldo, borrado y restauración. Marcá el principal y el inquilino en cada
  borde.
  - **Acá:** elegí una credencial de `work/creds/` (o la clave del memfs del Core) y dibujá desde
    dónde se escribe hasta dónde se usa, se loguea, se respalda y se rota. Ése es el dato cuyo
    linaje importa.

- Compará dos inquilinos falsos por los mismos métodos locales del servicio, y repetí después de un
  cambio de ACL, un borrado, un cambio de cuenta y una restauración. No uses datos de usuarios
  reales.
  - **Acá no hay dos inquilinos: hay dos principales y dos estados.** Compará una sesión válida
    contra un pedido anónimo por el túnel, y el estado al día contra el estado con la credencial
    vieja. Todo con datos de prueba en `work/tmp`, sin tocar el canal al aire.

- Empezá por los clientes que saltean controles, los trabajos de fondo, las migraciones, la unicidad
  global y las claves de caché. Estos caminos suelen omitir la identidad del pedido que sí llevan
  los endpoints interactivos.
  - **Acá:** los scripts de `work/bin` que escriben donde el Core espera, el watchdog, el script de
    despliegue, los nombres de archivo en `work/tmp` y las cookies.

## Reglas de validación (aplican antes de reportar cualquier hallazgo de acá)

1. Nombrá al atacante o al principal de menor confianza, el dato o el estado protegido, el dueño o
   inquilino afectado, la copia o la operación alternativa, y la divulgación o mutación no
   autorizada.
2. Citá la política prevista de la fuente de verdad **y** el camino que la omite o la contradice.
   Confirmá que otra capa no hace cumplir la misma condición de inquilino o de ciclo de vida.
3. Usá datos falsos y fixtures no sensibles para probar el acceso cruzado o el comportamiento viejo
   del ciclo de vida. Pará en el registro o la operación mínima observable.
4. Si hace falta la política externa de caché, almacenamiento de objetos, réplicas, analíticas,
   respaldo o retención, clasificá `needs_validation` y decí qué tiene que observar el dueño.
5. Devolvé `confirmado` sólo con el linaje completo y un impacto de frontera concreto. Devolvé
   `needs_validation` con el hecho exacto sin resolver de almacenamiento, ACL, invalidación,
   retención o restauración.

> **En nuestra VM, en concreto:** no hay sandbox del sistema operativo, así que ninguna de estas
> pruebas se hace contra un servicio vivo: se lee el código y, cuando se reproduce algo, es con
> datos de prueba en `work/tmp`. **Ningún valor de credencial aparece en el informe, ni en un
> ejemplo:** se nombran el archivo y su permiso. Y no se toca `work/firmas/` ni el canal al aire.

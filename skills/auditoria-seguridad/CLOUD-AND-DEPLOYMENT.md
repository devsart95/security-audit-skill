# Nube y despliegue

#### Cuándo usar este archivo

Usá este archivo cuando el repositorio defina identidad de nube, infraestructura, contenedores,
Kubernetes, service mesh, funciones serverless, workers de borde, ingreso, almacenamiento de
objetos, servicios gestionados o configuración específica de un entorno. Este dominio pregunta si
los componentes desplegados reciben la identidad, el aislamiento, la alcanzabilidad de red, los
secretos y la política que se pretendían. La fuente suele expresar intención, no hecho vivo: separá
los defectos confirmados en la fuente de lo que necesita validación de despliegue.

**En nuestro terreno** el despliegue es Docker en una sola VM Linux, sin nube: dos contenedores
nuestros (`lntv-ui`, con la imagen `restreamer-ui:local`, y `restreamer`, con
`datarhei/restreamer:latest`), unidades de `systemd --user`, túneles efímeros de `cloudflared` y los
scripts de `work/bin` que reponen la cadena. **No hay Kubernetes, ni service mesh, ni serverless, ni
IAM de nube, ni almacenamiento de objetos gestionado, ni CI/CD en GitHub.** Las clases que dependen
de eso quedan en el archivo con su aclaración de que hoy no aplican: no se borran, porque el día que
algo corra en una nube vuelven tal cual están.

Usá `SUPPLY-CHAIN-AND-RELEASE.md` para la confianza del build y de la promoción,
`WEB-PROTOCOL-AND-AUTH.md` para la semántica de proxy HTTP, y `DATA-ISOLATION-AND-LIFECYCLE.md` para
el alcance de los datos en el almacén.

## Disciplina central (incluir en cada prompt de agente de este dominio)

```
- No deduzcas una exposición viva de un manifiesto solo. Establecé qué entorno lo consume, qué
  valores por defecto u overlays lo modifican, y si ese camino de la fuente está activo. El
  `docker-compose.yml` del repo de la UI describe un despliegue en una máquina limpia: **no** es lo
  que corre en la VM.
- Mapeá la identidad de cada carga de trabajo a operaciones y recursos concretos. Una política
  amplia es hallazgo sólo cuando una entrada de menor confianza puede alcanzar una acción no
  autorizada.
- El ingreso, los proxies, el service mesh, los servicios de metadatos y la política de admisión son
  fronteras reales, pero un control cuenta sólo si su configuración y su enganche están visibles.
- Referenciar un secreto no es revelarlo. Hace falta un lector de menor confianza, una salida, un
  artefacto, un log o un respaldo inseguro.
- Usá `confirmado` para configuraciones activas del repo y para renderizado o validación de política
  local. Usá `needs_validation` para política de cuenta, enganche de red, admisión en tiempo de
  ejecución, metadatos alojados o deriva que necesite observación del dueño.
```

## Clases de ataque de identidad de carga de trabajo e IAM (subagent_type: `general`)

**Exceso de alcance de la identidad de carga de trabajo**
Una carga de trabajo, pod, función, worker de borde o identidad de nodo puede actuar sobre
inquilinos, cuentas, recursos o APIs más allá de su rol, y una entrada de pedido o de trabajo no
confiable elige ese objetivo. Revisá condiciones de política de nube, patrones de recurso, enganche
de cuenta de servicio, mapeo de namespace y credenciales de respaldo.

> **No aplica hoy en nuestra VM:** no hay IAM de nube ni identidad de carga de trabajo. Lo que hay
> que mapear es más chico y más concreto: **con qué usuario corre cada contenedor** y qué alcanza
> desde ahí (los puertos publicados en el host, `host.docker.internal`, el volumen del Core montado
> de sólo lectura), y qué puede tocar el usuario que corre los scripts (`sudo docker` es autoridad
> de root en la VM). Las cuentas de servicio no existen; los "roles" reales son `wicetel` con
> `sudo`, el usuario propio del contenedor y la cuenta de GitHub con su token.

**Confusión de rol entre cuentas o inquilinos**
La asunción de rol, los ID externos, el intercambio de tokens, la federación de cargas de trabajo o
las políticas de recurso aceptan afirmaciones de identidad que no están atadas a la cuenta, la
audiencia, el repositorio, el namespace o la carga de trabajo de origen. Establecé la política de
confianza **y** la afirmación que controla quien llama.

> **No aplica hoy en nuestra VM:** no hay asunción de rol entre cuentas. El equivalente es la
> credencial prestada: el token de GitHub con acceso a los dos repos privados (`restreamer-ui` y
> `freesky-gx6622`), la cuenta compartida de los agentes y la sesión del Core. La pregunta
> aterrizada: ¿una credencial pensada para un repo o para un servicio sirve igual contra el otro?

**Autorización de la aplicación delegada a metadatos de nube**
Una aplicación confía en encabezados de identidad, etiquetas, `labels`, ID de cuenta o metadatos de
recurso que aporta quien llama, sin verificar que vinieron del plano de control de la nube o de un
proxy confiable. El IAM de nube y la autorización de la aplicación son chequeos distintos.

> **Acá sí aplica, con otro nombre.** No hay plano de control de nube, pero sí encabezados que
> cualquiera que llegue por el túnel puede mandar: `X-Forwarded-For`, `X-Forwarded-Proto`,
> `X-Real-IP`, `CF-Connecting-IP`. Revisá si la UI o el Core toman alguno como identidad, como
> origen de confianza o para decidir que un pedido "es local", en vez de autenticar de verdad. Es la
> misma falla: creerle al que llama.

## Clases de ataque de ingreso, red y plano de control (subagent_type: `general`)

**Alcanzabilidad inesperada de un servicio o del plano de gestión**
Un ingreso, servicio, listener, grupo de seguridad, anotación de balanceador, mapeo de puertos o
bind del servidor expone una API de administración, depuración, métricas, nodo, plano de control o
interna a una red de menor confianza. La ausencia de controles de red por sí sola es
`needs_validation`; una ruta pública controlada por el repo hacia un handler sensible puede ser
`confirmado`.

> **Acá el "mapeo de puertos" es el despliegue mismo:** el `0.0.0.0:3100` de `lntv-ui` y los
> `0.0.0.0:8080`, 1935/1936, 6000/udp y 8899 del Core quedan alcanzables desde la red de la VM sin
> pasar por el túnel. El mapa medido y la regla de la clase están en
> [ENTORNO-DEVSAR.md](ENTORNO-DEVSAR.md) y [EXPOSICION-Y-TUNELES.md](EXPOSICION-Y-TUNELES.md). El
> otro candidato es SSH en la VM.

**Salteo de identidad de proxy confiable y de service mesh**
Un backend acepta identidad reenviada, sujeto de mTLS o metadatos de autorización de pares que están
fuera del ingreso o del sidecar previstos, o un puerto alternativo y una ruta de salud o legacy
evitan el mesh. Verificá el borrado de encabezados, la alcanzabilidad de los pares y el
comportamiento cuando el proxy no está.

> **No hay mesh; el proxy es el túnel.** El borde es `cloudflared` (el rápido y anónimo de
> `trycloudflare.com`): **no autentica, no filtra y no agrega identidad de origen**, así que no hay
> encabezado de proxy en el que confiar y la única barrera es el login del propio servicio. Qué pasa
> cuando el túnel se cae o se reinicia (los procesos viven como unidades transitorias de `systemd
> --user`, que el watchdog repone) es parte de la misma pregunta: el comportamiento cuando el proxy
> falta.

**Alcanzabilidad de metadatos y servicios internos**
Una URL, un destino o una selección de protocolo no confiable llega a metadatos de instancia o de
contenedor, sockets del plano de control o APIs internas con las credenciales de la carga de
trabajo. Seguí el parseo de URL y el manejo de redirecciones con `ATTACK-CLASSES.md`; acá establecé
la red desplegada, la versión de los metadatos y las fronteras de identidad.

> **Acá no hay `169.254.169.254`** (esa IP no lleva a ningún servicio de metadatos de nube en esta
> VM, así que no hay con qué comprometerse). Lo que sí existe, y es el equivalente:
> **`host.docker.internal` y los puertos del host publicados en `0.0.0.0`**, que desde adentro de un
> contenedor alcanzan todo lo que escucha en la VM (el Core, el panel, el nginx del 8899). Un SSRF
> desde la UI o desde el Core no va a buscar credenciales de nube: llega a nuestros propios
> servicios.

## Clases de ataque de contenedores y orquestación (subagent_type: `general`)

**Exposición de capacidades del host o del plano de control**
Una carga de trabajo de menor confianza puede elegir modo privilegiado, capacidades, namespaces del
host, rutas del host, montajes de dispositivos, sockets del runtime de contenedores o tokens de
cuenta de servicio que cruzan hacia la autoridad del nodo o del plano de control. La mera ausencia
de `seccomp` o de un sistema de archivos de sólo lectura es endurecimiento, salvo que una operación
alcanzable cruce esa frontera.

> **Acá, sin Kubernetes, el equivalente es el socket de Docker y los volúmenes del host.** Los
> scripts de `work/bin` (por ejemplo `deploy-ui.sh`, que construye `restreamer-ui:local` y reemplaza
> el contenedor `lntv-ui`) corren con `sudo docker`, y eso es autoridad de root sobre la VM para
> quien pueda ejecutarlos. Revisá qué ruta del host entra a cada contenedor y con qué permiso (en el
> despliegue de la UI, el storage del Core se monta de sólo lectura en `/core/data`), si algún
> contenedor monta el socket de Docker, y quién puede escribir esos scripts.

**Inconsistencia en el camino de admisión y política**
Una ruta de despliegue hace cumplir la identidad de la imagen, el namespace, los recursos, los
secretos o los privilegios, mientras otro controlador, trabajo, actualización, restauración o camino
de compatibilidad no lo hace. Confirmá la ruta alternativa y el objeto desplegado que resulta.

> **Acá hay dos caminos y hay que compararlos:** el `docker-compose.yml` del repo de la UI (ejemplo
> de despliegue en una máquina limpia) y el `deploy-ui.sh` real, que hace `git pull --ff-only` de
> `main`, construye la imagen y reemplaza el contenedor. Sumale las unidades de `systemd --user`,
> que el watchdog crea como unidades transitorias. La pregunta concreta: ¿qué control existe en un
> camino y falta en el otro?

**Confusión de confianza por namespace y etiqueta**
La red, la admisión, los secretos o la identidad de carga de trabajo se apoyan en `labels`,
anotaciones, nombres o namespaces que un principal de menor confianza puede definir. Compará quién
controla los selectores con qué autoridad otorga que coincidan.

> **Acá no hay `labels`; hay nombres.** El nombre del contenedor (`lntv-ui`), el `tag` de la imagen
> (`restreamer-ui:local`), el nombre de la unidad y el de los archivos en `work/tmp` son las
> etiquetas de este entorno: quien pueda crear o pisar un nombre con ese formato confunde a quien lo
> consume por nombre. El caso más filoso ya está en el repo: `deploy-ui.sh` arma la URL pública del
> Core leyendo `work/tmp/tunel-core.log`.

## Clases de ataque del ciclo de vida de configuración y secretos (subagent_type: `general`)

**Deriva en la precedencia de los controles de seguridad**
Valores de desarrollo, valores por defecto del chart, variables de entorno, banderas de línea de
comandos, interruptores de funcionalidad, inyección de sidecar o overlays por región apagan la
autenticación, la seguridad del transporte, el alcance por inquilino o la política de auditoría en
un entorno desplegado. Renderizá la configuración final de cada despliegue que mantenemos, no sólo
el archivo base.

> **Acá la configuración final no es el `docker-compose.yml` ni el `Dockerfile`:** es el `docker
> run` que arma `deploy-ui.sh`, más el `.env` y las variables de entorno, más los `-p` de los
> contenedores. Y ese script tiene un valor por defecto que cambia lo que ve el navegador: si no
> puede leer la URL pública del túnel, cae a `http://host.docker.internal:8080`. Un camino que se
> degrada en silencio cuando falta un dato es exactamente esta clase.

**Exposición de secretos entre fronteras de carga de trabajo**
Los secretos entran en logs, informes de caída, argumentos de proceso, entorno compartido, volúmenes
amplios, salidas de build, descubrimiento de servicios o APIs de lectura alcanzables por otra carga
de trabajo o inquilino. Fijate el tipo de secreto y su autoridad; un endpoint público o un ID de
clave no es una credencial.

> **Acá ya hay historia medida:** credenciales pasadas por la línea de comandos a un `docker run`
> quedan visibles para cualquiera que pueda hacer `docker inspect`; y los logs de FFmpeg del Core
> traen la clave del memfs en texto plano, que es por lo que todo log pasa por `redactSecrets`.
> Sumale `work/creds/` (700, un archivo por servicio) y las variables de entorno del contenedor.
> **Ningún valor va al informe: se nombran el archivo, el campo y su permiso.**

**Renovación de credenciales y respaldo ante una caída**
No montar, refrescar, rotar o revocar una credencial de carga de trabajo deja credenciales viejas
activas, o hace que una aplicación acepte un modo de identidad de menor confianza. Revisá el
arranque, la preparación, la reconexión y el comportamiento del cliente cacheado.

> **Acá la pregunta es qué queda vivo cuando una credencial cambia.** Si se cambia la clave del
> Core: ¿las sesiones de la UI, que llevan el token en la cookie, siguen valiendo o el Core las
> rechaza y la UI vuelve a pedir login? ¿Qué pasa con el cliente que ya tenía el token cacheado y
> con el proceso que el watchdog repone? La política declarada (toda ruta que haga trabajo local
> consulta primero al Core con el token) está en el `AGENTS.md` y los `docs/` del repo de la UI: el
> trabajo es encontrar la ruta que no la cumpla.

## Clases de ataque de almacenamiento gestionado, eventos y borde (subagent_type: `general`)

**Confusión de política de objetos y URL firmadas**
La política del bucket o del contenedor, las claves de objeto, los orígenes de CDN o las URL
firmadas no atan principal, operación, espacio de nombres de objeto, audiencia ni vencimiento.
Revisá las operaciones de lista y de versiones, y los caminos de escritura, además de las lecturas.

> **No aplica hoy en nuestra VM:** no hay bucket, ni CDN, ni URL firmadas. El equivalente cercano es
> el almacén del Core y sus URL (`/memfs/...` y el HLS por el nginx del 8899): quién las puede pedir
> sin sesión y hasta cuándo valen.

**Confusión de identidad en la fuente de eventos**
Una función o worker confía en campos del cuerpo del evento como identidad de origen sin validar el
sobre firmado por el proveedor, la suscripción o el topic, la cuenta, la región ni el estado de
repetición. Compará los caminos de push, pull, reintento y cola de mensajes muertos.

> **No aplica hoy en nuestra VM:** no hay colas, ni funciones, ni eventos con sobre firmado. Lo más
> parecido son los avisos que el watchdog manda a un DM de Discord (salida hacia afuera, con
> retención ajena) y el timer que corre cada cinco minutos. Un evento sin firma que dispara una
> acción es esta clase; hoy no hay ninguno.

**Desajuste de frontera entre el borde y el runtime**
Un runtime de borde o serverless supone un secreto, una API, un sistema de archivos, un aislamiento
o una política de inquilino distintos de los del runtime de origen, y la caída al origen cambia la
autoridad o el comportamiento de la caché. Confirmá qué configuración elige cada camino.

> **No aplica hoy en nuestra VM:** la UI corre como `next start` dentro de un contenedor (`output:
> 'standalone'`), no en un runtime de borde, y lo único "de borde" es el túnel de `cloudflared`, que
> no ejecuta nuestro código ni cachea: reenvía. La regla del túnel (cada servicio publicado necesita
> su propia autenticación) está en [EXPOSICION-Y-TUNELES.md](EXPOSICION-Y-TUNELES.md).

## Movimientos universales (aplican a todo lo de arriba)

- Renderizá cada entorno que mantenemos y armá una matriz de puerto externo, identidad de carga de
  trabajo, pares de red, secretos montados y recursos de nube. Cada diferencia necesita la
  explicación del dueño o de una política.
  - **Acá hay un solo entorno vivo.** La matriz es la tabla de
    [ENTORNO-DEVSAR.md](ENTORNO-DEVSAR.md) (puerto, qué escucha, en qué interfaz) más la columna que
    falta: con qué usuario corre cada contenedor, qué volúmenes y variables recibe, y qué unidad lo
    arranca. La segunda columna de comparación es el `docker-compose.yml` del repo: **si difiere de
    lo que corre, esa diferencia es el hallazgo a explicar**.

- Seguí un pedido, un objeto, una etiqueta o un evento de menor confianza hasta la política. Mostrá
  qué credencial de carga de trabajo hace la operación final y qué condición debería acotarla.
  - **Acá:** seguí un `curl` que llega por el túnel de la UI hasta la operación final en el Core,
    decí con qué token se autoriza (la cookie de sesión, que valida el Core) y qué la acota. El
    camino completo —túnel, contenedor, host, Core— es el que hay que poder dibujar.

- Diferenciá el despliegue normal, la migración, la restauración, el mantenimiento del nodo, el
  failover y los caminos locales o de emulador. Revisá el comportamiento cuando el mesh, la
  admisión, la identidad, los secretos o el servicio de política no están disponibles.
  - **Acá:** compará `deploy-ui.sh` con el `docker-compose.yml`, con lo que hace el watchdog cuando
    repone la cadena, y con el estado en que queda todo cuando el túnel o el Core están caídos (el
    caso está contado en la cabecera de `work/bin/tuneles-lntv.sh`). El camino de falla es parte del
    despliegue.

## Reglas de validación (aplican antes de reportar cualquier hallazgo de acá)

1. Establecé el camino de fuente activo y el objeto de despliegue efectivo; si no, usá
   `needs_validation` y decí qué manifiesto renderizado o qué enganche observado por el dueño falta.
2. Nombrá el llamador o la carga de trabajo de menor confianza, la identidad de nube o de
   aplicación, el selector controlable, el recurso afectado y la operación o divulgación no
   autorizada.
3. Verificá los valores por defecto del proveedor y del orquestador en la versión fijada. No
   supongas una IP pública, un servicio de metadatos alcanzable, un firewall permisivo ni la
   ausencia de un enganche de admisión.
4. La validación local puede renderizar plantillas, evaluar política, inspeccionar namespaces de
   contenedor o de usuario en un banco aislado, o correr un emulador con identidades de mentira. No
   sondees endpoints vivos ni alteres recursos de nube compartidos.
5. Devolvé `confirmado` sólo con una traza completa de fuente activa y un resultado de frontera
   concreto. Devolvé `needs_validation` con la política desplegada, el enganche de identidad, el
   overlay, la red o la deriva exacta que haya que observar.

> **En nuestra VM, en concreto:** el objeto efectivo es el contenedor que está corriendo y la unidad
> que lo arranca, no el `docker-compose.yml`; `docker compose config` valida el archivo sin levantar
> nada y todo se mira en modo lectura. **No se levanta un contenedor nuevo para probar, ni se toca
> el que está sirviendo el canal al aire.** La versión de Docker de la VM y el
> `datarhei/restreamer:latest` son hechos a medir, no a suponer: `latest` es un `tag` que se mueve,
> y lo que importa es el digest que está corriendo.

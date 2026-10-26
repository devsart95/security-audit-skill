# Dependencias y entrega

#### Cuándo usar este archivo

Usá este archivo cuando el objetivo resuelva dependencias, construya desde contribuciones no
confiables, corra CI, cree artefactos de release, firme o promueva builds, cargue plugins o
actualice software desplegado. Este dominio cubre los traspasos de confianza desde la fuente y las
dependencias hasta el artefacto que alguien ejecuta. Usá `MEMORY-SAFETY-AND-BINARY.md` para las
fallas dentro de un cargador de binarios local y `CLOUD-AND-DEPLOYMENT.md` para la autoridad de la
carga de trabajo en ejecución.

**En nuestro terreno:** dos repos privados con PRs (`devsart95/restreamer-ui`, fork de
`datarhei/restreamer-ui`, y `devsart95/freesky-gx6622`), npm con `package-lock.json`, imágenes de
Docker construidas en la VM, y un receptor que se actualiza a mano desde la Mac de Justino. **No hay
CI/CD en GitHub**: Actions está deshabilitado por decisión del dueño y **el gate son tests locales**
(en la UI: `npx tsc --noEmit && npm run lint && npm test && npm run build`; en el receptor:
`./scripts/verificar.sh`). Eso cambia el terreno de la clase, no la clase: un gate local corre en la
máquina que tiene las credenciales y el Docker a mano, o sea con **más** autoridad que un runner
alojado, no con menos.

Dividí los objetivos grandes en resolución de dependencias, aislamiento de CI, procedencia de
artefactos, autorización de release, y confianza en el actualizador o los plugins.

## Disciplina central (incluir en cada prompt de agente de este dominio)

```
- Una dependencia mutable o con una CVE conocida no es un hallazgo por sí sola. Mostrá quién puede
  influir en su resolución, qué build la consume y qué frontera de ejecución o de release sigue.
- Seguí la integridad en cada traspaso: identidad de la fuente, entradas resueltas, máquina que
  construye, identidad del artefacto, resultado del test, firma o atestación, promoción y consumidor
  de la actualización.
- La configuración de CI es código de autorización. Establecé qué evento disparó un flujo, de quién
  es el código que corre, qué secretos y tokens existen, y qué puede publicar o mutar.
- Un checksum bajado del mismo lugar no confiable que el artefacto no establece integridad
  independiente. Identificá la raíz de confianza y el comportamiento ante el fallo.
- Usá `confirmado` para fallas de flujo controladas por el repo con validación local acotada. Usá
  `needs_validation` para la protección de rama, el runner alojado, el registry, el servicio de
  firma o la promoción a producción que no se puedan observar.
```

## Clases de ataque de dependencias y entradas de build (subagent_type: `general`)

**Confusión de origen y espacio de nombres de dependencias**
La configuración del resolvedor puede elegir un espacio público o privado no previsto, un registry
de respaldo, un espejo, un repositorio o una URL de origen. Revisá los nombres de paquete, la
prioridad de las fuentes, el uso del `lockfile` y de los checksums, los archivos de build
alternativos, la resolución por plataforma, y la diferencia entre la primera instalación y la
actualización.

> **Acá:** `npm` contra el registry público, con `package-lock.json` versionado y `npm ci` en el
> `Dockerfile` (que copia `package.json` y el lockfile antes que el código). No hay registry privado
> ni espejo. Lo que hay que revisar es que **el `lockfile` sea el que manda**: que nadie resuelva
> con un `npm install` suelto, que no haya un `.npmrc` que cambie el origen, y que una dependencia
> nueva entre al `lockfile` en su propio commit y no de rebote.

**Entradas de build mutables y sin atar**
Los builds consumen ramas, `tags`, submódulos sin verificar, herramientas descargadas, assets
generados, includes remotos, actions de CI flotantes o `tags` de contenedor cuyo contenido puede
cambiar sin revisión de fuente. Hace falta un escritor de menor confianza y un camino hacia la
salida confiable; que el build sea reproducible no prueba autenticidad.

> **Acá hay un ejemplo bueno y uno flojo.** El bueno: el `Dockerfile` fija `node:26.8.2-alpine` a
> propósito y lo deja escrito en el comentario (un `tag` flotante viejo en la caché de Docker trae
> una versión sin avisar). El flojo: el Core corre desde `datarhei/restreamer:latest`, un `tag` que
> se mueve; lo que importa es el **digest** que está corriendo. Y el más filoso: `deploy-ui.sh` saca
> `CORE_PUBLIC_URL` de `work/tmp/tunel-core.log`, un archivo que escribe otro proceso.

**Brechas de procedencia en el código generado**
Esquemas, archivos vendorizados, clientes generados, traducciones, ejemplos de documentación o blobs
binarios producen contenido ejecutable o entregado sin la misma revisión y el mismo control de
integridad que el código fuente. Compará la regeneración local con la salida commiteada y verificá
quién controla la entrada y el generador.

> **Acá:** los generadores propios de `work/bin` (`armar-mapas.py`, `armar-sprite.py`,
> `hyperframe.py`, `hoja-de-iconos.py`) y el del receptor (`scripts/generar-overlay.py`, cuyo
> resultado tiene que estar al día: el gate lo revisa). La pregunta: ¿el archivo generado se
> commitea y después nadie lo regenera? ¿Quién controla la entrada del generador — un archivo del
> repo, o algo que otro proceso escribe?

**Inclusión en el contexto de build**
Secretos, configuración local, metadatos del repositorio, fixtures de test o artefactos del
desarrollador entran a un paquete o a una imagen porque el contexto de build y las reglas de
ignorado superan las entradas previstas de release. Confirmá que el artefacto resultante exponga una
credencial real, un dato privado o una configuración privilegiada.

> **Acá:** el repo de la UI tiene `.dockerignore`, y su primer comentario es exactamente esta clase
> (los `.env*` copiados a la imagen quedan adentro para siempre). Leelo y comprobalo contra el `COPY
> . .` de la etapa de build: qué archivo local termina adentro de la imagen `restreamer-ui:local`. Y
> acordate de la regla de la casa: **una credencial que entró a una imagen se considera expuesta**.

## Clases de ataque de CI y automatización (subagent_type: `general`)

**Código no confiable en un flujo privilegiado**
Un pedido de cambios, un comentario de un issue, un fork, una actualización de dependencia o un
evento externo corre código que controla quien contribuye, con secretos protegidos, tokens de
escritura, autoridad de despliegue o un runner de confianza. Compará el tipo de disparador, la
referencia que se descarga, la compuerta de aprobación, la protección del entorno y el
estrechamiento de permisos. No supongas valores por defecto del anfitrión del repositorio que no
estén en la fuente.

> **Acá no hay workflows, y por eso la clase cambia de forma en vez de desaparecer.** El "flujo
> privilegiado" es el gate local que corre Justino en la VM después de revisar un PR: `npm install`
> de una rama, `npm test`, `./scripts/verificar.sh`, un `docker build`. Ese gate corre **como el
> usuario de la VM, con `sudo docker` a mano, con el Core al lado y con `work/creds/` a un
> `read_file` de distancia**. La compuerta de aprobación es la lectura del PR; no hay runner efímero
> ni secreto aislado. La pregunta correcta no es "¿qué workflow se dispara?" sino "¿qué corre en mi
> máquina cuando reviso y ejecuto esto, y con qué autoridad?".

**Confusión de comandos y expresiones de flujo**
Nombres de rama, mensajes de commit, campos de un issue, nombres de artefactos, valores de matriz o
salida generada que controla el atacante entran en comandos de shell, expresiones de plantilla,
rutas o entradas privilegiadas de un flujo sin validación canónica.

> **Acá es lo mismo pero sin YAML:** en `deploy-ui.sh` entran la rama y el commit que se leen de
> `origin/main`, y la URL pública que se saca con un `grep` sobre `work/tmp/tunel-core.log`. Un
> valor leído de un archivo o de un nombre que otro controla no es un dato de confianza, ni aunque
> venga de nuestro propio proceso.

**Mezcla de confianza en caché, artefactos y espacio de trabajo**
Un trabajo de menor confianza puede llenar una caché, un artefacto, un espacio de trabajo compartido
o una salida que después un trabajo de mayor confianza restaura y ejecuta o publica. Revisá las
claves y los espacios de nombres de la caché, la identidad de quien produce el artefacto, el atado
al digest, la retención, y si la promoción vuelve a resolver por un nombre mutable.

> **Acá:** la caché de Docker del build de la UI (`npm ci` se cachea por el `lockfile`, que es lo
> correcto), y el `tag` `restreamer-ui:local`, que cada build pisa: **el nombre se vuelve a resolver
> en cada despliegue**. Sumale `work/tmp`, que un script lee y otro escribe, y el `node_modules` del
> árbol de trabajo.

**Exceso de alcance de la identidad de la automatización**
Los trabajos de CI reciben permisos más allá de la operación, el repositorio, el entorno o la
duración que necesitan, y entradas no confiables del trabajo pueden elegir el recurso afectado. La
falta de mínimo privilegio por sí sola es endurecimiento; hace falta una acción privilegiada
alcanzable.

> **Acá no hay tokens de CI.** Lo que hay: el token de GitHub con acceso a los dos repos privados
> (`restreamer-ui` y `freesky-gx6622`), la cuenta compartida de los agentes y `sudo docker` para los
> scripts. La pregunta aterrizada es la misma: ¿qué credencial usa el proceso que revisa y
> despliega, y qué alcanza con ella que la tarea no necesita?

## Clases de ataque de release y actualización (subagent_type: `general`)

**Sustitución entre el build y la promoción**
Los tests, la revisión, la firma y la publicación se refieren a `tags` mutables, nombres de archivo,
canales o ID de artefacto, en vez del mismo digest inmutable. Revisá cada copia, reempaquetado,
unión de arquitecturas y paso de procedencia entre el build y el release.

> **Acá el release es local y no tiene firma.** La UI se "publica" así: `deploy-ui.sh` hace `git
> pull --ff-only` de `main`, construye `restreamer-ui:local` y reemplaza el contenedor `lntv-ui`. El
> commit que se construyó queda en la salida del script (`git rev-parse --short HEAD`), **no pegado
> al artefacto**: el `tag` es el mismo en cada build. Si la imagen que corre no se puede atar a un
> commit, la trazabilidad es la del log, no la de la imagen. Para el receptor, el artefacto es la
> imagen del rootfs que se arma con `scripts/build-rootfs.py` y se flashea a mano.

**Brechas en la autorización del release y en la política de firma**
Un release o una firma se aceptan desde el flujo, el repositorio, la rama, el entorno, el rol de
clave o el umbral equivocados. Revisá las afirmaciones de identidad dentro de las atestaciones y
verificá que el consumidor las valide, no sólo que la firma sea válida. La rotación, el vencimiento
y la revocación tienen que fallar cerrado donde la política lo exija.

> **Acá no hay firma ni atestación.** Las compuertas reales son la protección de `main` en el repo
> privado, la revisión del PR antes de mergear (squash desde `origin/main`) y el chequeo antes de
> flashear el receptor: **el `MD5SUMS` antes de escribir la flash**. Dos cosas hay que establecer
> antes de afirmar nada: si el `MD5SUMS` se genera en la misma corrida que el binario y viaja con
> él, **no es integridad independiente** —sirve para detectar una copia corrupta, no para autenticar
> el artefacto—, y un MD5 no es una firma. Decí de dónde sale el checksum que se compara.

**Confusión de metadatos de actualización y de vuelta atrás**
Un actualizador autentica los bytes del contenido pero no la versión, el producto, la plataforma, el
canal, la ruta de destino, el vencimiento o el estado de vuelta atrás, o acepta metadatos y
contenido de transacciones autorizadas distintas. Verificá la instalación atómica y el
comportamiento de recuperación. Una llamada a una API de firma sin atar la política está incompleta.

> **Acá el actualizador es a mano y el receptor es el caso serio:** escribir la flash de un equipo
> sin consola serie, con la regla de no escribir una partición suelta y con la red de rescate
> primero (está en el `CLAUDE.md` del repo `freesky-gx6622`). Ahí la versión, el modelo, la
> partición y la vuelta atrás son el corazón de la clase. **Desde la VM no se flashea ni se ejecuta
> nada de eso**: se lee el repo, lo que se pueda observar se observa desde la Mac de Justino, y lo
> que no, queda como `needs_validation`.

**Expansión de confianza por plugins y extensiones**
Un paquete de extensión gana autoridad en el anfitrión más allá de su alcance declarado, un
publicador de menor confianza puede reemplazar la identidad de otro, o los ganchos de instalación y
actualización corren antes de los chequeos de autenticidad y de capacidades. La instalación
deliberada de plugins arbitrarios del mismo usuario no es una frontera de privilegio.

> **Acá sí aplica, y es el receptor:** el addon (`plugin.video.localnet`) es Python 2.7 que corre
> dentro del intérprete embebido en `/app/dvbapp`, y se instala en el equipo con su `addon.xml`.
> `scripts/deploy-addon.py` y `scripts/hacer-zip.sh` son los que lo empaquetan. Las preguntas: ¿de
> dónde sale el zip que se instala, quién más puede producirlo, y el receptor verifica algo más que
> el `addon.xml`? Ojo con el gate: `./scripts/verificar.sh` **no compila los addons** (no pasan
> `py_compile` de python3): valida su `addon.xml`, no su código. Un error de sintaxis en el addon lo
> ve el receptor, no el gate.

## Movimientos universales (aplican a todo lo de arriba)

- Caminá hacia atrás desde un digest publicado o una actualización instalada hasta cada fuente,
  entrada generada, credencial, máquina, caché, resultado de test y decisión de autorización.
  - **Acá:** hacia atrás desde el contenedor `lntv-ui` que está corriendo (su digest y el commit del
    que salió) hasta el `git pull` de `deploy-ui.sh`, el `lockfile`, el registry de npm y el
    `Dockerfile`.

- Compará un evento de flujo no confiable y uno protegido, uno al lado del otro. Marcá cada canal
  persistente que los cruza y exigí identidad inmutable más confianza en quien lo produce.
  - **Acá:** un PR contra `main` de un repo privado y un commit directo; el build local que dispara
    Justino y el despliegue que dispara el watchdog; el zip del addon que se arma en la Mac y el que
    se copia con `push-file.py`.

- Revisá los caminos de clave revocada, descarga fallida, atestación ausente, publicación parcial de
  plataforma, vuelta atrás y registry caído. La política ante el fallo es parte de la integridad del
  release.
  - **Acá:** qué hace `deploy-ui.sh` si el build falla (corta y muestra las últimas líneas del log:
    el contenedor viejo queda en pie, que es lo correcto); qué hace si el `git pull --ff-only` no es
    aplicable; qué hace si no encuentra la URL del túnel (cae a `host.docker.internal` y avisa); y
    qué queda si el registry de npm no responde en medio de un `npm ci`.

## Reglas de validación (aplican antes de reportar cualquier hallazgo de acá)

1. Nombrá al actor de menor confianza, la fuente, caché, artefacto o metadato controlable, el
   trabajo de confianza o el actualizador que lo consume, y la publicación, inclusión de código,
   divulgación de secretos o ejecución privilegiada que resultan.
2. Probá la identidad del artefacto en el traspaso roto. Un nombre mutable distinto o un digest sin
   atar tiene que llegar a un consumidor real.
3. Verificá los valores por defecto del gestor de paquetes, del anfitrión del repositorio, del
   registry y de la firma en la versión fijada. Los controles alojados desconocidos necesitan
   `needs_validation`.
4. Mantené la validación local acotada: un repositorio de prueba inofensivo, un marcador de
   credencial falso, un registry o una configuración locales, y un espacio de nombres de artefacto
   que no sea producción. No publiques ni alteres un release real.
5. Devolvé `confirmado` sólo con un traspaso completo visible en la fuente y un resultado con
   sentido. Devolvé `needs_validation` con el hecho preciso de rama, runner, registry, firma o
   despliegue que el dueño tiene que observar.

> **En nuestra VM, en concreto:** no se sale a internet, así que las versiones y las CVE conocidas
> se leen del `package-lock.json` y de los `lockfiles`, **no** se averiguan ejecutando `npm audit`
> ni instalando nada. No se publica ni se firma nada, y un release real no se altera: el despliegue
> se audita leyendo `deploy-ui.sh` y mirando el contenedor que ya está corriendo, no reemplazándolo.

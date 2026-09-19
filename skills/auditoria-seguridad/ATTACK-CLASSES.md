# Clases de ataque

Elegir y dividir según lo que devuelva la fase 1. No todas las clases aplican a todos los
objetivos. La lista es un punto de partida: si el reconocimiento muestra algo propio del objetivo,
se agrega una clase y se divide el trabajo por subsistema.

Encuadre: encontrar, validar, arreglar y priorizar. **La validación es lectura de código y, cuando
se puede, una prueba local con datos de prueba** (ver las reglas del terreno en `SKILL.md`).

Usar `confirmado` sólo cuando la evidencia en el código y la validación acotada establezcan la
frontera completa y un resultado con sentido. Usar `needs_validation` cuando falte un hecho de
despliegue, proveedor, plataforma o tiempo de ejecución: se nombra el hecho que falta y la
observación segura que lo resolvería.

> **Objetivos con IA, LLM o agentes** (nosotros: skills, prompts, agentes con herramientas,
> memoria persistente, texto que entra al contexto desde archivos o páginas): usar las clases de
> contexto, envenenamiento de memoria, atado de acciones, esquema de herramientas e identidad en
> [AI-AND-LLM.md](AI-AND-LLM.md).
>
> **Exposición, túneles y servicios propios**: usar [EXPOSICION-Y-TUNELES.md](EXPOSICION-Y-TUNELES.md).
>
> **HTTP, web e identidad** (nuestra UI y su API, sesiones, cookies, proxy, subidas): usar
> [WEB-PROTOCOL-AND-AUTH.md](WEB-PROTOCOL-AND-AUTH.md).
>
> **Cliente y navegador** (React, estado del navegador, `postMessage`, CORS, WebSockets): usar
> [CLIENT-SIDE.md](CLIENT-SIDE.md).
>
> **Dependencias y entrega** (npm, lo que se instala, el gate, los PR, los scripts de despliegue):
> usar [SUPPLY-CHAIN-AND-RELEASE.md](SUPPLY-CHAIN-AND-RELEASE.md).
>
> **Nube y despliegue** (Docker, unidades, configuración de tiempo de ejecución): usar
> [CLOUD-AND-DEPLOYMENT.md](CLOUD-AND-DEPLOYMENT.md).
>
> **Consumo de recursos** (disco, memoria, procesos, cuota, el canal al aire): usar
> [RESOURCE-EXHAUSTION-AND-AVAILABILITY.md](RESOURCE-EXHAUSTION-AND-AVAILABILITY.md).
>
> **Datos y ciclo de vida** (credenciales, firmas, planillas, respaldos, borrado): usar
> [DATA-ISOLATION-AND-LIFECYCLE.md](DATA-ISOLATION-AND-LIFECYCLE.md).
>
> **Binarios del fabricante** (el firmware del receptor, sus parsers y decodificadores): usar
> [MEMORY-SAFETY-AND-BINARY.md](MEMORY-SAFETY-AND-BINARY.md). Se lee y se analiza; **no se ejecuta**.

---

**Inyección**

Seguir el dato no confiable desde la entrada hasta el sumidero peligroso. Qué es un sumidero
depende del objetivo:

- **UI y API (Next.js)**: consultas a la base del Core, HTML renderizado, rutas de archivo,
  redirecciones, deserialización de la config, argumentos que terminan en un comando (FFmpeg,
  `docker`, `git`).
- **Scripts propios (Python, shell)**: construcción de comandos, interpolación de variables de
  entorno, rutas que se pasan a `dd`, `nandwrite`, `mksquashfs`, `nc`, `openssl`.
- **Firmware y archivos de datos**: rutas dentro de un JSON/XML que se convierte en ruta de
  archivo (los catálogos del receptor), y campos que se pasan a `iptables` o a un `hosts`.
- **Registros y logs**: inyección en logs, en nombres de archivo, en metadatos que después se
  muestran en el panel.

No quedarse en el camino directo. Buscar inyección indirecta: datos guardados sanos que después se
usan en un contexto peligroso por otro camino del código, y también por **nombre de campo**, no
sólo por valor.

**Control de acceso**

Verificar que quien llama no pueda hacer algo fuera de su autoridad. No alcanza con que exista un
chequeo: hay que ver si comprueba **el permiso correcto, sobre el recurso correcto, por el
mecanismo correcto**.

- ¿Hay otro camino al mismo cambio de estado que valide un permiso más débil?
- ¿Un campo del cuerpo del pedido puede pisar lo que el sistema quiso restringir?
- ¿Hay rutas que exigen autenticación pero se olvidan de la autorización?
- ¿El mismo recurso tiene varios caminos de acceso con chequeos distintos?
- En operaciones por lote (listas, exportaciones, capas, campañas): ¿se aplica el permiso ítem por
  ítem?

**Manejo de recursos y archivos**

- Recorrido de rutas (leer o escribir fuera de la carpeta prevista), incluso por symlinks,
  secuencias codificadas y bytes nulos.
- SSRF: hacer que un servicio nuestro pida una URL que elige otro (incluye redirecciones y
  diferencias entre parsers de URL). En nuestro caso: la URL de una fuente de video, una URL de
  webhook, una descarga.
- Deserialización insegura y extracción de archivos (zip slip) en las subidas y en la mediateca.
- Manejo de temporales: `/tmp` compartido, symlinks predecibles, archivos que quedan.
- Carreras en operaciones de archivo (entre el chequeo y el uso).

**Criptografía y secretos**

- Aleatoriedad débil para valores de seguridad (tokens, claves, nonces).
- Secretos escritos en el código, en logs, en mensajes de error, en URLs o en respuestas visibles.
  **Acá hay historia**: una clave del memfs que aparece en texto plano en los logs de FFmpeg del
  Core, y credenciales pasadas por línea de comandos a `docker run` (visibles con `docker inspect`).
- Ausencia de verificación de firma o de HMAC donde hace falta.
- Comparación de secretos en tiempo variable.
- Uso mal hecho de primitivas (modo ECB, cifrado sin autenticar, IV fijo).
- Qué pasa cuando la criptografía falla: ¿el camino de error cae a "sin cifrado"?
  En el receptor ya se midió algo así: **su cliente HTTPS no valida TLS** (no tiene ni un
  certificado CA en el rootfs), así que acepta cualquier certificado. Eso es un hallazgo conocido
  de ese objetivo, no de nuestro código.

**Lógica de negocio**

Los escáneres no ven esto y suele ser de alto impacto. Por cada flujo principal:

- **Máquina de estados**: ¿se pueden saltar pasos, ir para atrás, llegar a un estado inválido? ¿Se
  puede repetir un flujo ya terminado? Si el paso 2 de 3 falla, ¿se revierte el 1?
- **Carreras con impacto**: dos operaciones concurrentes que dejan un estado inválido (doble
  arranque de un canal, doble aprobación, actualización perdida).
- **Números y cantidades**: negativos, cero, desbordes, pérdida de precisión, coerción entre texto
  y número (en planillas de compra: kilos, precios, fijaciones).
- **Fronteras de autoridad**: no "¿existe el chequeo?" sino "¿es el chequeo que la regla de negocio
  pide?"
- **Confianza implícita**: datos que vienen del almacenamiento, de la config o de un plugin y se
  tratan como seguros "porque los validamos al entrar". ¿Y si los escribió otro camino?
- **Tiempo**: vencimientos, horarios, franjas, husos horarios distintos entre componentes (el
  contenedor en UTC y las franjas en `America/Asuncion` es una combinación ya vista acá).
- **Comportamiento por defecto**: qué postura queda cuando falta la config, cuando el interruptor
  está apagado, cuando una dependencia no está, o cuando el sistema está a mitad de una migración.

**Abuso de funciones y fuga de datos**

Funciones legítimas usadas para otra cosa. Es diseño, no sólo código:

- **Exportar o respaldar como exfiltración**: ¿un usuario de bajo privilegio puede disparar una
  exportación o un respaldo que incluya datos por encima de su nivel? ¿Se exporta lo borrado, lo
  privado, o el historial que debía podarse? (Acá: respaldos de config del Core, listas, copias del
  `mtd7`.)
- **Importar o restaurar como inyección**: ¿una importación puede pisar datos existentes o crear
  registros que saltean la validación normal?
- **Buscar, filtrar u ordenar como oráculo**: ¿la búsqueda revela si existe algo que no se puede
  ver? ¿El orden por un campo oculto revela su valor?
- **Enumeración por efectos**: ¿los errores distinguen "no existe" de "no tenés permiso"? ¿Los
  tiempos de respuesta? ¿Los códigos HTTP? ¿El tamaño de la respuesta?
- **Fuga por lo público del túnel**: tokens de vista previa, contenido en borrador, listados de
  archivos, rutas del disco en un mensaje de error. Ver
  [EXPOSICION-Y-TUNELES.md](EXPOSICION-Y-TUNELES.md).
- **Notificaciones y webhooks como SSRF**.

**Cadenas y fronteras de confianza**

Comportamiento permitido por separado que se vuelve vulnerabilidad cuando otro componente confía
en una garantía más fuerte:

- **Fallas de frontera en varios pasos**: mapear qué puede leer, escribir, invocar y retener un
  principal de baja confianza, y conectar **sólo salidas concretas** con decisiones posteriores.
- **Fronteras entre componentes**: el componente A valida y se lo pasa a B. Comparar la garantía
  exacta que produce A con la que B asume (truncado, coerción de tipos, normalización, alcance).
- **Uso de segundo orden**: un dato sano al guardarse que es peligroso en otro contexto — un nombre
  de campo que se vuelve ruta JSON, un texto que entra a un render crudo, una cadena guardada que
  se vuelve URL, expresión regular, plantilla o regla de `iptables`.
- **Crecimiento de alcance o capacidad**: tokens, claves de API, sesiones o capacidades de un
  agente que se agrandan después de delegar, refrescar, cachear o cambiar de rol.
- **Tiempo y orden**: configuración, migración, borrado lógico, vencimiento de caché, ventanas de
  chequear-y-usar.
- **Vuelta atrás y recuperación**: deshacer, restaurar, revertir una revisión y cancelar deben
  aplicar la propiedad y la autorización **actuales**.
- **Confianza puesta en un artefacto que otro reescribe**: el caso del receptor es el ejemplo
  máximo (el catálogo de canales lo puede pisar cualquier proceso con root), pero acá también
  aplica: cualquier archivo que un script nuestro lea y otro pueda escribir.

**Comodín**

No se da categoría: buscar fuera de las clases ya asignadas. Leer el código que parece aburrido o
desconectado de la seguridad; seguir las funciones incompletas, experimentales, de compatibilidad y
de respaldo, con el mismo listón de frontera y validación.

Pistas: ¿cuál es el código más raro y por qué existe? ¿Qué funciones quedaron a medias? ¿Qué se
puede hacer con la API que el frontend nunca hace? ¿Qué endpoints o parámetros no están en la
documentación? ¿Qué pasa al mezclar funciones que no fueron pensadas juntas? ¿Hay algo en el
historial de git (arreglos de seguridad revertidos, chequeos comentados, secretos que se
commitearon y después se borraron pero siguen en la historia)? ¿Qué acciones de una cuenta válida
afectan a otros, a la integridad compartida, a la disponibilidad o al gasto del dueño? ¿Qué
operaciones son irreversibles? ¿Qué supone el código del entorno (que la base es local, que el
reloj es correcto, que el DNS es confiable, que el sistema de archivos distingue mayúsculas)? Y
**qué no están probando los tests**: comparar los casos borde que el autor pensó con los que no.

**Las obvias**

Los demás buscan lo sutil; ésta revisa lo básico que nadie mira porque supone que ya lo miró otro.
No hace falta creatividad: hace falta ser literal y revisar cada punto.

- ¿Hay contraseñas, claves de API, tokens o secretos escritos en el código? (Buscar `password`,
  `secret`, `apikey`, `token`, `Bearer`, `-----BEGIN`, contraseñas por defecto conocidas.)
- ¿Hay `TODO`/`FIXME`/`HACK`/`XXX` que mencionen seguridad?
- ¿El modo depuración está bien cerrado? ¿Se puede prender en producción por una variable de
  entorno, un parámetro de la URL o un encabezado?
- ¿Hay credenciales de prueba o de ejemplo que funcionen de verdad?
- ¿Hay un `/debug`, `/admin`, `/test`, `/status`, `/health`, `/metrics`, `/env`, `/.env` o
  `/config` sin proteger? **En la UI hay un `/api/salud` público**: revisar qué informa.
- ¿Hay `.env`, `credentials.json`, `*.pem`, `*.key` versionados? ¿El `.gitignore` cubre de verdad
  secretos, subidas y config local? (Acá hay un `.gitallowed` por credenciales interpoladas: leerlo
  antes de dar por limpio un repo.)
- ¿Las dependencias están fijadas? ¿Hay CVE conocidos en el árbol? (Leer los lockfiles; **no**
  instalar nada para averiguarlo.)
- ¿Hay `eval()`, `exec()`, `child_process`, `Function()`, `vm.runInContext` o `import()` con algo
  que no sea constante?
- ¿Los encabezados CORS están en `*` o de más? ¿`Access-Control-Allow-Credentials` junto con origen
  comodín?
- ¿Las cookies tienen `HttpOnly`, `Secure` y `SameSite`? (En la UI: la cookie de sesión y la del
  Core.)
- ¿Hay redirecciones abiertas? (Parámetros `redirect`, `return`, `next`, `url`, `goto`, `continue`
  que alimenten una redirección sin validar.)
- ¿Se fuerza TLS? ¿Hay endpoints sólo HTTP?
- ¿Los errores en producción devuelven trazas, rutas internas o errores de la base?
- ¿Hay puertos propios publicados en `0.0.0.0` que deberían estar en loopback?
- ¿Qué rutas contestan sin sesión a través del túnel?

Para cualquier hallazgo de esta clase, **verificar el camino completo del código y no la apariencia**.
Si a una cookie le falta `HttpOnly`, ver si contiene datos sensibles y si el JavaScript necesita
leerla por diseño. Si un mensaje de error trae el nombre de un campo, ver si ese campo llega a
tener un dato sensible. Una bandera no es un hallazgo: hay que trazar el impacto antes de reportar.

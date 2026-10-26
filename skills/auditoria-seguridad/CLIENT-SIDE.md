# Caza de cliente y navegador

#### Cuándo usar este archivo

Echar mano de este archivo cuando las decisiones de confianza o el renderizado de datos no
confiables ocurran en un navegador: aplicaciones de una sola página, extensiones de navegador,
webviews embebidas, service workers, aplicaciones offline, y código que renderiza contenido
influenciable por el atacante en el DOM, recibe mensajes entre ventanas o usa el almacenamiento del
navegador. Estos caminos incluyen fuentes que el servidor nunca ve, como el fragmento de la URL,
`window.name`, `postMessage` y contenido ya cacheado.

En nuestro terreno es la UI propia (Next.js 16 + React 19) y su panel del Core, publicados por los
túneles `cloudflared`.

Se usa junto con `ATTACK-CLASSES.md`. Este archivo cubre fuentes y sumideros del navegador, fronteras
de origen, persistencia del navegador y oráculos de estado entre sitios. Para CSRF del lado del
servidor, sesiones y callbacks de autenticación, usar `WEB-PROTOCOL-AND-AUTH.md`.

## Disciplina central (incluir en cada prompt de agente de este dominio)

```
- Un candidato del lado del cliente necesita una fuente controlable y un sumidero que ejecute o divulgue. Nombrar los dos y mostrar el dato influenciado por el atacante llegando al sumidero.
- El impacto tiene que alcanzar la sesión de una víctima, otro origen o una persistencia compartida. La auto-inyección y la divulgación de los datos del propio atacante no son hallazgos.
- El escape del framework, la política de mismo origen del navegador, CSP, COOP/CORP, el alcance del service worker y los valores por defecto modernos de noopener son controles reales. Verificarlos antes de asignar impacto.
- El almacenamiento y las cachés del navegador se comparten por origen y pueden sobrevivir al estado de login. Identificar quién escribe, quién lee y qué cuenta, inquilino o ciclo de vida del worker limpia cada registro.
- Usar `confirmado` sólo con evidencia de código completa más pruebas locales acotadas en el navegador. Usar `needs_validation` cuando haga falta el renderer, un permiso de extensión, un header desplegado o un comportamiento de política del navegador y no esté disponible.
```

## Clases de ataque de DOM y estado de objetos (subagent_type: `general`)

**XSS basado en DOM**
Trazar campos de `location`, `document.referrer`, `window.name`, datos de mensajes, almacenamiento y
estado del documento controlado por el navegador hacia `innerHTML`, `outerHTML`, `document.write`,
APIs que evalúan strings, URLs ejecutables, APIs HTML de jQuery o las escotillas de escape del
framework. Una interpolación que el framework escapa no es un hallazgo.

**Clobbering del DOM**
Atributos `id` o `name` inyectados por el atacante tapan un global, una propiedad de formulario, un
objeto de configuración o una bandera de inicialización en la que el código confía después. Se
exigen las dos cosas: un camino de marcado que preserve el atributo y un uso relevante para la
seguridad del valor tapado.

**Contaminación de prototipos y cadena de gadgets**
Una clave controlada por el atacante llega a una escritura recursiva, como un merge profundo o una
asignación de ruta, y modifica el estado de un prototipo. Después un gadget alcanzable consume la
propiedad contaminada para cambiar autorización, ejecución, navegación o renderizado. `JSON.parse`,
una copia superficial o una contaminación sin gadget no alcanzan.

## Clases de ataque de mensajería entre orígenes y red (subagent_type: `general`)

**Origen y fuente en `postMessage`**
Un handler hace una acción sensible con `event.data` sin lista blanca exacta de origen y, donde
varios frames comparten origen, sin el `event.source` esperado. Del lado del envío, datos sensibles
mandados a `*` llegan a un embedder no previsto. Un chequeo de origen por subcadena, prefijo,
sufijo o regex sin ancla no es un chequeo de origen.

**Uso de pedidos WebSocket entre sitios**
Un upgrade de WebSocket acepta cookies ambientales de un origen no confiable sin chequeo de `Origin`
ni token propio del canal, lo que deja que la sesión de la víctima lea o mute datos. Confirmar tanto
el comportamiento del upgrade como un handler de mensajes relevante para la seguridad.

**Confianza en CORS con credenciales**
El servidor refleja o compara débilmente `Origin` mientras permite credenciales y devuelve
respuestas sensibles. Un comodín pelado con credenciales lo rechazan los navegadores; reportar sólo
el camino real de origen reflejado o permitido y la lectura o mutación entre orígenes.

## Clases de ataque de service worker y almacenamiento del navegador (subagent_type: `general`)

**Registro de service worker y toma de alcance**
Contenido influenciable por el atacante puede convertirse en el script del worker registrado,
controlar una ruta que recibe un alcance `Service-Worker-Allowed` demasiado ancho, o alterar los
imports de actualización sin control de integridad. Verificar la URL final del script, el MIME type
de la respuesta, el origen, el alcance y quién controla cada script importado. Un worker normal de
mismo origen con el alcance previsto no es un defecto.

**Caché del service worker y confusión de identidad**
El worker cachea respuestas personalizadas sin incluir cuenta, inquilino, estado de autorización o
modo del pedido en su política, y después las sirve tras un cambio de cuenta o un cierre de sesión.
Revisar el ruteo del evento fetch, los nombres y claves de caché, los fallbacks de navegación, la
limpieza de caché, y si los caminos de error u offline devuelven la respuesta previa de otro usuario.

**Divulgación en el almacenamiento del navegador y autorización vieja**
Tokens, respuestas privadas, borradores o decisiones de autorización quedan en `localStorage`,
`sessionStorage`, IndexedDB, Cache Storage, almacenamiento de extensiones o estado del cliente y se
vuelven legibles por otra cuenta o por un componente de mismo origen de menor confianza. Guardar un
token por sí solo no es un hallazgo; hace falta un lector realista con menos autoridad, o el uso
continuado después de revocar o cerrar sesión.

**Confusión de almacenamiento y broadcast entre contextos**
Eventos `storage`, `BroadcastChannel`, workers compartidos o cachés de todo el origen llevan
identidad o comandos entre pestañas sin atarlos a la sesión actual. Revisar el cambio de cuenta,
ventanas privadas y públicas, cambios de inquilino, y pestañas viejas que pueden pisar un estado de
autenticación más nuevo.

## Clases de fuga de información entre sitios (subagent_type: `general`)

**XS-Leaks y oráculos de estado entre orígenes**
Una página del atacante puede distinguir estado protegido de otro origen mediante eventos de carga o
error de recursos, estado de frame o ventana, comportamiento de redirección, tiempos, estado de
caché o tamaño de la respuesta, mientras el navegador adjunta las credenciales de la víctima. Hace
falta un predicado concreto que porte un secreto, como si existe un objeto privado, un rol o una
cuenta. Una varianza genérica de tiempos o la disponibilidad de un recurso público no es un hallazgo.

**Divulgación de estado por ventana y opener**
Los metadatos permitidos de una ventana de otro origen o el resultado de una navegación revelan
estado protegido, o una relación de opener o ventana con nombre retenida deja que una página
controlada por el atacante influya en una navegación privilegiada. Revisar COOP, las protecciones de
frame, `noopener`, el origen exacto, y si el estado observable es confidencial.

## Clases de ataque de UI-redress y navegación (subagent_type: `general`)

**Clickjacking**
Una acción que cambia estado y que se puede enmarcar no tiene `frame-ancestors`, `X-Frame-Options`
ni un aislamiento de UI equivalente efectivo. Exigir la acción sensible y confirmar que puede
completarse en el estado enmarcado; la falta de headers en contenido de sólo lectura son notas de
endurecimiento.

**Confusión de navegación del lado del cliente**
Una fuente del cliente controla la redirección o la navegación sin política de esquema y destino,
incluidos destinos ejecutables como `javascript:` o `data:`. El reverse tabnabbing aplica sólo donde
el código mantiene `window.opener` a propósito, usa `window.open` sin aislamiento, o soporta un
navegador sin `noopener` implícito.

## Movimientos universales (aplican a todo lo de arriba)

- Arrancar por los sumideros de DOM, navegación, worker, mensajes y almacenamiento, y trazar hacia
  atrás hasta las fuentes propias del navegador y las controladas por el servidor. Anotar la política
  del navegador que debería frenar ese camino.
- Probar el cambio de cuenta, el cierre de sesión, la actualización del worker, el fallback offline y
  el estado de una pestaña vieja con un origen de prueba local y cuentas de mentira. No usar usuarios,
  orígenes ni servicios compartidos de producción.
- Para XS-Leaks, listar sólo los predicados probados por el código y el comportamiento local del
  navegador. Después identificar el header de respuesta o la elección de renderizado que eliminaría
  el oráculo.

## Reglas de validación (aplican antes de reportar cualquier hallazgo de acá)

1. Citar la fuente, el sumidero, la política del navegador, el origen/sesión afectados y la mutación
   o divulgación observable.
2. Para contaminación de prototipos, probar la escritura recursiva y un gadget relevante para la
   seguridad. Para clobbering del DOM, probar que el marcado sobrevive y que el valor tapado se usa.
3. Para service workers y almacenamiento, probar la alcanzabilidad del ciclo de vida: una escritura o
   entrada de caché controlada por el atacante tiene que llegar a otra cuenta, otro inquilino o un
   estado de autorización posterior.
4. Para mensajería, CORS, WebSocket y XS-Leaks, mostrar la validación exacta de origen y fuente, y el
   estado o la acción protegidos que quedan expuestos. Confirmar que CSP, COOP/CORP, las cookies y la
   política de `SameSite` no lo bloqueen ya.
5. Devolver hallazgos `confirmado` sólo con un camino del cliente completo y evidencia local acotada.
   Devolver `needs_validation` con el header desplegado, el permiso de extensión, la versión de
   navegador o el comportamiento del renderer exactos que el dueño tiene que verificar.

# Caza de protocolo HTTP y autenticación

#### Cuándo usar este archivo

Echar mano de este archivo cuando el objetivo hable HTTP en una frontera de parseo, caché,
autenticación de navegador o identidad: aplicaciones web, APIs, proxies inversos, CDNs, gateways,
servidores HTTP propios y servicios que implementan sesiones, JWT, OAuth/OIDC, SAML, recuperación de
contraseña, MFA, passkeys, claves de API o mTLS. Se usa junto con `ATTACK-CLASSES.md`: la revisión de
control de acceso pregunta **si** un principal puede hacer una operación; este archivo pregunta si la
capa HTTP o la de identidad puede confundir **a qué** principal, pedido, nivel de garantía o token
pertenece esa operación.

En nuestro terreno esto es la UI propia (Next.js 16 + React 19) y su API, el Core datarhei en su
contenedor, y todo lo que se publica por los túneles `cloudflared`.

Se eligen las clases en la fase 1. Un objetivo grande se parte en framing de pedido y política de
caché, autenticación de navegador, identidad federada, autenticación fuerte y recuperación,
credenciales de servicio y ciclo de vida de sesión. Un servidor propio detrás de un proxy
administrado que no observamos tiene poca superficie de smuggling confirmable por código; un proxy o
un parser propio tiene mucha más.

## Disciplina central (incluir en cada prompt de agente de este dominio)

```
- Los hallazgos de framing y caché exigen dos interpretaciones del mismo pedido, respuesta o clave. Nombrar los dos componentes y el valor normalizado exacto de cada lado.
- Para cada credencial, encontrar la verificación de firma o secreto y todo atado que su rol necesite: emisor, audiencia, origen, RP, cliente, sesión, principal, recurso, garantía, vencimiento y estado de un solo uso.
- Host, Forwarded, X-Forwarded-*, Origin, Referer, destinos de redirección, estado de callback y URLs derivadas del pedido son decisiones de confianza. Trazar cada una hasta la identidad o la respuesta afectada.
- Un header, atributo de cookie, pedido de MFA o límite de intentos que falta no es un hallazgo por sí solo. Hace falta un pedido inválido aceptado, impacto entre principales, degradación de la garantía o divulgación de una credencial.
- Clasificar como `confirmado` sólo con evidencia de código completa y pruebas locales acotadas de pedidos o tokens. Usar `needs_validation` cuando haga falta proxy, IdP, navegador, certificado, secreto o configuración desplegada y no esté a la vista.
```

## Clases de ataque de framing HTTP y caché (subagent_type: `general`)

**Framing de pedido y desincronización**
El front end y el back end no coinciden en el largo del pedido o en la normalización de headers.
Revisar valores múltiples de `Content-Length`, `Transfer-Encoding`, degradación de HTTP/2 o HTTP/3,
normalización del nombre de header, headers de conexión prohibidos y conversión de CR/LF. Confirmar
qué bytes le asigna un componente a un pedido y qué bytes le asigna su par al pedido siguiente.

**Envenenamiento de caché por entrada fuera de la clave**
Un valor del pedido cambia el contenido cacheado o headers relevantes para la seguridad, pero no
está en la clave de caché. Comparar la construcción de la clave con cada variante de respuesta,
incluidos host/esquema reenviados, cookies seleccionadas, normalización de query, headers de
idioma/dispositivo y estado de autorización.

**Engaño de caché y cacheo de respuestas privadas**
El ruteo de la caché trata una ruta dinámica privada como un activo estático público, o cachea una
respuesta cuyos insumos de identidad y autorización no están en la política. Comparar la
cacheabilidad en el borde con el parseo de rutas de la aplicación, la normalización de sufijos y
parámetros de ruta, y las directivas de caché de la respuesta.

**Confianza en el host y en headers reenviados**
Metadatos no confiables de host o proxy determinan URLs absolutas, ruteo por inquilino, callbacks,
links de reseteo, claves de caché o la dirección del cliente que usa la autorización. Confirmar
quién puede poner el header y si la entrada confiable borra las copias que manda el cliente.

**Inyección en headers de respuesta**
Un dato no confiable llega a `Location`, `Set-Cookie`, CSP u otro header de respuesta con caracteres
de control o normalización insegura. Verificar que el framework los rechace antes de reportar, y
exigir un cambio de respuesta relevante para la seguridad.

## Clases de ataque de sesión de navegador (subagent_type: `general`)

**CSRF común**
El navegador manda credenciales ambientales a un endpoint que cambia estado y acepta un pedido
entre sitios sin un token anti-CSRF efectivo, sin atado mismo-sitio ni validación estricta de
Origin/Referer. Inventariar cada mutación autenticada por cookie, incluidas las de formulario, las
tipo JSON, las multipart, las de override de método y las rutas viejas. `SameSite` sólo sirve para
la cookie y los contextos de navegador que realmente se usan; el CSRF de login y los pedidos de
subrecursos entre sitios pueden tener requisitos distintos.

**Fijación e invalidación de sesión**
Los identificadores de sesión no rotan al iniciar sesión, cambiar de cuenta, completar MFA,
suplantar u otro cambio de privilegio, o siguen valiendo después de cerrar sesión, cambiar la
contraseña, revocar o deshabilitar la cuenta. Revisar sesiones del servidor, refresh tokens, cookies
firmadas, estado de WebSocket, copias en caché y endpoints de respaldo.

**Alcance y transporte de cookies**
Una cookie sensible tiene `Domain` o `Path` demasiado anchos, puede cruzar un transporte inseguro, o
choca con una cookie hermana que otro componente selecciona distinto. La falta pelada de flags son
notas de endurecimiento, salvo que un origen de menor confianza, una posición en la red o un camino
del navegador realista pueda conseguir o reemplazar la credencial.

## Clases de ataque de identidad federada (subagent_type: `general`)

Primero establecer el rol. Los controles del servidor de autorización, como la lista blanca de
redirección y la emisión del código, no son del cliente que confía. Los defectos de verificación y
atado son del componente que consume el artefacto.

**Verificación de JWT y atado de claims**
Revisar la verificación de firma, el algoritmo y el origen de la clave fijados en el servidor, y
después `exp`, `nbf`, `aud` e `iss`. Mirar `kid`, `jku` y `x5u` como selectores de clave no
confiables, la normalización de duplicados y de headers, y los caminos que decodifican sin
verificar. Un token válido para otro servicio no sirve acá, aunque lo firme un emisor de confianza.

**Atado de pedido y callback de OAuth/OIDC**
Validar la propiedad exacta de `redirect_uri` cuando el objetivo es el servidor de autorización; el
`state` atado a la sesión; PKCE y el atado del código de autorización cuando aplique; emisor,
audiencia, firma y nonce del ID-token; y el atado del IdP elegido en flujos con varios proveedores.
Comparar la ruta de callback inicial, la de reintento, la de mobile/deep-link y la de vínculo de
cuenta.

**Atado de objeto firmado y assertion en SAML**
Asegurar que el elemento cuya firma se valida sea el elemento que se usa como identidad. Revisar
caminos sin firma o de fallback, la configuración del parser XML seguro, diferencias de
canonicalización, y campos de frescura y atado como ventanas de validez, audiencia/destinatario,
correlación del pedido y estado de replay.

## Clases de ataque de MFA, passkey y transición de cuenta (subagent_type: `general`)

**Alta de MFA y degradación de la garantía**
El alta, el reemplazo, la baja, la generación de códigos de recuperación, la creación de dispositivos
de confianza y el login de respaldo exigen la garantía previa que corresponde. Verificar que un
primer factor válido no pueda dar de alta ni reemplazar el segundo factor sin una autenticación
fresca que la política pida, y que los factores dados de baja o viejos dejen de autorizar sesiones.

**Atado y bypass del step-up**
Un desafío exitoso sube la sesión, cuenta, inquilino, acción o pedido de API equivocado, o una ruta
alterna se saltea el chequeo de garantía. Atar el desafío a principal, sesión actual, objetivo de
garantía, operación o recurso cuando corresponda, vencimiento y finalización de un solo uso. Comparar
los caminos de UI, API, lote, recuperación y flujo retomado.

**Verificación de WebAuthn y passkey**
En el registro, atar challenge, RP ID, origen esperado, credencial, user/userHandle, algoritmo y la
verificación de usuario que la política pida a la sesión que lo inició. En la autenticación,
verificar challenge, RP/origen, membresía de la credencial, firma y la presencia/verificación de
usuario buscada. Revisar los flujos de descubrimiento de cuenta y de vínculo por confusión entre
userHandle y cuenta. El manejo del contador de firmas sólo tiene sentido si el producto trata una
regresión como señal de clonado.

**Vínculo de cuentas y colisión de identidad**
Agregar un IdP, passkey, email, teléfono, dispositivo o cuenta externa a una cuenta existente tiene
que exigir una sesión autenticada actual, la titularidad verificada de la identidad nueva, el
step-up que la política pida, y un estado de callback atado a la cuenta que inició el vínculo.
Revisar los caminos de desvincular/revincular y de aceptar invitaciones por colisiones de
identificador verificado o de inquilino.

**Reseteo de contraseña y recuperación en general**
Los tokens de recuperación, la recuperación por soporte o admin, los códigos de respaldo, la
migración de dispositivos y el cambio de email o teléfono suelen ser el camino de autenticación más
débil. Verificar la aleatoriedad del token, el atado a usuario/acción, el vencimiento, el estado de
un solo uso, los controles de ritmo y contabilidad, la confianza en la URL de entrega y la
invalidación de tokens y sesiones previos. Respuestas distintas que sólo revelan la existencia de
una cuenta pública no son hallazgos de seguridad automáticamente.

## Clases de ataque de clave de API y mTLS (subagent_type: `general`)

**Alcance de la clave de API y atado de recursos**
Una clave autentica a más inquilinos, recursos, acciones o entornos de lo que le concede su registro
en el servidor, o los parámetros del pedido pisan esos atados. Revisar la búsqueda de la clave, la
verificación de prefijo y de secreto completo, la confusión de tipos entre claves publicables y
secretas, los chequeos de alcance, la rotación, las cachés de revocación y los endpoints por lote.

**Exposición de claves de API y transporte inseguro**
Las claves aparecen en bundles de cliente, URLs, redirecciones, logs, caminos de error, artefactos
de build o respuestas alcanzables por un principal de menor confianza. Un identificador público al
que le dicen clave no es un secreto. Confirmar el tipo de clave y la autoridad que da divulgarla.

**Confusión de identidad de par y de aplicación en mTLS**
Un proceso confía en headers de identidad de certificado de cliente viniendo de cualquier par de la
red, verifica una cadena pero mapea mal a una cuenta un texto de subject influenciable por el
atacante, o acepta un certificado del dominio de confianza, uso extendido, audiencia o política de
validez equivocados. Donde un proxy de confianza termina el mTLS, verificar que sólo ese proxy pueda
conectarse, que borre los headers de identidad entrantes, y que el backend ate la identidad
saneada al pedido.

**Fallback del ciclo de vida de certificados**
Certificados vencidos, revocados, faltantes o con renovación fallida causan un fallback silencioso a
operación sólo con bearer o anónima, o las conexiones agrupadas de vida larga conservan la
autorización después de una revocación. Si faltan los datos de revocación del despliegue, el
resultado es `needs_validation`; una rama de fail-open dentro del repo sí es confirmable por código.

## Movimientos universales (aplican a todo lo de arriba)

- Recorrer emisión → guardado → transmisión → consumo → refresco → revocación para cada credencial y
  desafío. Comparar los caminos normal, de error, de reintento, de migración, viejo y de cambio de
  cuenta.
- Enumerar cada puerta a la misma identidad y cada ruta a la misma operación sensible. La política
  efectiva es el camino paralelo más débil, no la UI más prolija.
- Diferenciar parser, proxy, router, caché y aplicación lado a lado. Para validar localmente,
  alimentar a cada componente con fixtures de pedido idénticos y acotados, en vez de mandar tráfico a
  un despliegue vivo.
- Para recuperación y vínculo, dibujar el grafo de la cuenta antes y después. Cada arista tiene que
  nombrar el principal actual, la prueba de la identidad nueva, la garantía que se exige, el atado de
  callback/sesión y el efecto de revocación.

## Reglas de validación (aplican antes de reportar cualquier hallazgo de acá)

1. Aplicar una compuerta de visibilidad de la fuente. Las cadenas de proxy, las claves de caché del
   borde, la política del IdP, la confianza de certificados, el comportamiento de cookies del
   navegador, los secretos y los modos de autenticación desplegados pueden estar fuera del repo.
   Registrar un candidato `needs_validation` preciso en vez de afirmar un comportamiento de
   infraestructura que no se ve.
2. Para hallazgos de framing y caché, nombrar los dos componentes y el parseo/clave divergentes.
   Confirmar el impacto entre pedidos, entre usuarios o sobre respuestas privadas con fixtures
   locales acotados.
3. Para hallazgos de token, MFA, passkey, vínculo de cuenta, recuperación, clave de API y mTLS, citar
   la línea de verificación y el atado faltante de principal/sesión/recurso/origen/audiencia/acción/
   garantía. Probar que el servidor acepta la transición o la credencial inválida.
4. Para CSRF, nombrar la credencial ambiental, la ruta que cambia estado, la forma del pedido entre
   sitios que se acepta, la política de cookies del navegador y el chequeo efectivo faltante. Las
   acciones de sólo lectura y las rutas que exigen un token bearer no ambiental no califican.
5. Verificar los valores por defecto del framework y de la librería. Si la versión o la
   configuración se desconoce, usar `needs_validation`; no convertir una afirmación crítica sin
   verificar en un hallazgo confirmado de severidad menor.
6. Devolver `confirmado` sólo con una traza de código completa y una identidad, estado o divulgación
   observables sin autorización. Para `needs_validation`, nombrar el hecho que falta y el chequeo
   local seguro o que el dueño pueda observar y que lo resuelve.

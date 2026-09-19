# Exposición y túneles

Clase propia de DevSar. En este entorno lo que más se rompe no es el código: es **qué queda
publicado y quién lo alcanza**. Casi todo lo que corre acá es un servicio escuchando, un túnel
abierto o un archivo con permisos de más.

## Cómo se publica acá

Se usa el **túnel rápido y anónimo de `cloudflared`** (`trycloudflare.com`). Sus consecuencias:

- **Todo lo que se publique por el túnel es internet abierto.** No hay lista de acceso, ni
  autenticación en el borde, ni geobloqueo. La única barrera es la del propio servicio (su login).
- **La dirección es efímera y pública**: cualquiera que la tenga, la usa, y si el túnel se
  reinicia cambia. No se puede usar una lista de permitidos porque no hay identidad de origen.
- **Cada servicio publicado necesita su propia autenticación**, y hay que preguntarse **qué revela
  sin sesión**: una ruta de salud, un error con trazas, una lista de archivos, un `robots.txt`
  dicen bastante a quien la encuentre.

## Qué mirar, servicio por servicio

1. **¿Qué está publicado hoy?** Enumerar los túneles vivos (sus unidades y sus logs) y para cada
   uno: qué puerto publica, qué rutas tiene, y cuáles contestan **sin sesión**.
2. **¿Qué revela lo público?** Para cada ruta abierta: qué información devuelve (estado interno,
   versiones, rutas del disco, nombres de archivos, latencia del Core). Un panel de salud que
   informa más de lo necesario es una fuga de interior, aunque no sea una toma de control.
3. **¿Qué escucha en `0.0.0.0` y por qué?** Todo servicio publicado en todas las interfaces queda
   al alcance de la red de la VM **sin pasar por el túnel**. Lo correcto, cuando el acceso es sólo
   local o por túnel, es escuchar en `127.0.0.1`. Revisar con `ss -ltnp` y `docker ps` (los
   `-p 0.0.0.0:...`).
4. **¿El borde autentica o sólo el servicio?** Si la única barrera es el login propio, hay que
   revisar ese login como si estuviera en internet: fuerza del secreto, límite de intentos,
   duración de la sesión, cookies (`HttpOnly`, `Secure`, `SameSite`), y si la sesión se renueva
   sola sin volver a pedir credenciales.
5. **Credenciales en la URL o en la query.** Un panel que acepta usuario y clave por query string
   los deja en logs, en el historial del navegador y en el `Referer`.
6. **TLS**: los túneles dan HTTPS en el borde; entre el túnel y el servicio va en claro por
   loopback. Lo que importa acá es que **no se exponga un puerto propio en claro a la red**.
7. **Ingesta**: RTMP (1935) y SRT/UDP (6000) publicados en `0.0.0.0` permiten que cualquiera en esa
   red publique video a nuestro Core. Preguntar si hace falta que sea público y, si no, cerrarlo al
   loopback o exigir clave de publicación.
8. **Superficie de la cadena de despliegue**: los scripts que reconstruyen y reemplazan
   contenedores, las unidades transitorias y el timer del watchdog. ¿Puede alguien que no sea
   Justino disparar un redespliegue? ¿El script toma alguna URL o ruta de un archivo que otro
   escribe?
9. **Lo que sobrevive a un reinicio**: contenedores con `--restart unless-stopped` y unidades
   habilitadas. Lo que no sobrevive son las unidades transitorias (los túneles), y de eso depende
   que la cadena vuelva sola.

## Hallazgos que ya conocemos (para no volver a descubrirlos)

- **Una ruta de salud pública** en la UI, alcanzable sin sesión por el túnel, que informa si el
  Core responde, la latencia, si el disco es legible y si hay FFmpeg instalado. Es una fuga de
  interior: sirve para saber qué hay del otro lado. El `HEALTHCHECK` del contenedor depende de que
  sea pública; cerrarla exige mover el chequeo adentro.
- **Puertos propios en `0.0.0.0`** (el Core y la UI): alcanzables desde la red de la VM sin pasar
  por el túnel.
- **Credenciales en la línea de comandos** de un `docker run`: quedan visibles para cualquiera que
  pueda hacer `docker inspect`.
- **Una clave que aparece en texto plano en los logs de FFmpeg** del Core (la del memfs), y que por
  eso obliga a que todo log pase por `redactSecrets`.

## Reglas de la clase

- Se revisa **nuestra** exposición: nuestros puertos, nuestros túneles, nuestro login. No se sondea
  lo ajeno.
- Un `curl` a nuestro propio servicio por su túnel es evidencia válida y se registra con el comando
  exacto y la hora.
- **No se prueba nada que pueda cortar el aire del canal**: el canal de TV al aire es producción
  interna. Para probar un corte se usa un canal de prueba.
- Si el hallazgo depende de quién está en la red de la VM (algo que no se ve desde el código), va
  como `needs_validation` con la observación exacta que falta.

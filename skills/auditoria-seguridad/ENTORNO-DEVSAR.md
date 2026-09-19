# El entorno de DevSar — qué hay, qué es nuestro y qué no se toca

Medido el 2026-09-16 en la VM. **Leer esto antes de la fase 1**: el reconocimiento de cada
auditoría arranca de acá, no de suposiciones.

## La máquina

- VM Linux propia, usuario `wicetel`, **única zona de escritura `/home/wicetel/work`**.
- Red: `10.187.64.32/24` (eth0) y `172.17.0.1/16` (docker0). Resolver local en `127.0.0.53`.
- Docker con dos contenedores nuestros y `sudo` disponible.
- **Alcanza el servidor de video de LocalNet (`170.245.132.197`) pero no la LAN del receptor**
  (`192.168.88.254`): todo lo que sea del receptor se verifica desde la Mac de Justino, no de acá.

## Lo que corre (y por dónde escucha)

| Puerto | Qué | Escucha en | Nota |
|---|---|---|---|
| 8080 | Core datarhei (contenedor `restreamer`) | **0.0.0.0** | API y panel del Core |
| 8181 | Core, puerto auxiliar | **0.0.0.0** | |
| 1935 / 1936 | Core, RTMP | **0.0.0.0** | ingesta |
| 3100 | UI propia (contenedor `lntv-ui`) | **0.0.0.0** | login y panel |
| 8899 | nginx del restreamer | **0.0.0.0** | HLS |
| 22 | SSH de la VM | **0.0.0.0** | |
| 20241 / 20242 | túneles `cloudflared` | 127.0.0.1 | correcto: el túnel sale hacia afuera |
| 6000/udp | Core | 0.0.0.0 | |

**Que un servicio propio escuche en `0.0.0.0` es un hallazgo a revisar, no un detalle**: queda al
alcance de todo lo que esté en esa red sin pasar por el túnel. Ver
[EXPOSICION-Y-TUNELES.md](EXPOSICION-Y-TUNELES.md).

Unidades de usuario: `lntv-tunel-core`, `lntv-tunel-ui` (los túneles), `lntv-watchdog.service`
(las corre un timer cada 5 minutos), `hermes-gateway.service` (el agente).

## Lo que es nuestro (se puede observar y auditar)

- `work/restreamer-ui` — la UI (Next.js 16 + React 19). Trabajan dos agentes: uno en la Mac de
  Justino y este; el contrato entre los dos es `AGENTS.md`.
- `work/freesky/repo` — el sistema del receptor (scripts Python, overlay del rootfs).
- `work/bin` — scripts propios (deploy, túneles, generadores).
- `work/restreamer` — los datos y la config del Core.
- Los contenedores, las unidades y los túneles de la tabla de arriba.
- El fork `security-audit` (este repo).

## Las zonas sensibles (se auditan, no se copian)

| Zona | Qué hay | Regla |
|---|---|---|
| `work/creds/` (700) | un archivo por servicio: usuario, clave, token, notas | **Ningún valor va al informe.** Se nombra el archivo y el permiso |
| `work/firmas/` | la firma de la escribana y los documentos sellados | Dato sensible: ni al informe ni a un repo |
| `work/granos/` | planillas de compra (soja, maíz) | Datos de negocio: no salen de la VM |
| `restreamer/data/config.json` | la config del Core, con su usuario y clave | Legible sólo con `sudo`. No se copia |
| `~/.hermes/` | config, skills, sesiones del agente | Fuera del alcance de escritura; se lee lo necesario |
| Los `ACCESOS.md` y `config.json` de cada repo | Credenciales de paneles IPTV | No versionados, por diseño |

Reglas transversales: **credenciales nunca en un repo**, credenciales sólo por DM privado, y todo
log de FFmpeg o del Core pasa por `redactSecrets` antes de ir a un test, un doc, un PR o un
informe (trae la clave del memfs en texto plano).

## Lo que NO se toca (ni para probar)

- La VM 202 `integrax` y cualquier host de producción.
- El receptor de TV y su red (la LAN de Justino) — está fuera de la VM.
- `tata.local.net.py`, los paneles IPTV, los CMS de terceros: no se sondean, ni con `curl`.
- Ningún host de internet. Ni escaneos de puertos, ni fuzzing remoto, ni fuerza bruta.
- El firmware del fabricante: se lee y se analiza, **no se ejecuta**.

Ninguna acción destructiva (`rm -rf`, `drop`, reset duro, borrar ramas o particiones) sin que
Justino lo pida en ese momento.

## Herramientas de observación permitidas

`read_file`, `search_files`, `git log`/`git show`, `git status`, `curl` a `127.0.0.1` y a nuestros
propios túneles, `sudo docker ps|inspect|logs`, `ss -ltnp`, `ps`, `systemctl --user status`,
`journalctl --user`, `du`/`df`, `md5sum`, `strings` sobre un binario propio, y los validadores de
este repo.

Lo que se ejecute de un objetivo corre **sólo con datos de prueba**, sin red externa y sin
instalar nada.

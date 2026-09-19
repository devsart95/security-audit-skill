## Qué es esto

Nuestro fork de [`cloudflare/security-audit-skill`](https://github.com/cloudflare/security-audit-skill)
(MIT), adaptado a DevSar. **El método de seis fases se conserva entero** —reconocimiento, caza
guiada por cobertura, validación adversarial, salida estructurada, verificación independiente y
informe—; lo que cambia es el terreno donde corre.

## Qué se adaptó

- **Reglas del terreno** (`SKILL.md`): el original exige un sandbox del sistema operativo para
  ejecutar código del objetivo y **acá no hay**. La regla pasó a ser: no se ejecuta código ajeno,
  lo propio corre sólo con datos de prueba y sin salir a internet, y todo lo que necesite ejecutar
  algo de un tercero queda como `needs_validation` con el bloqueo exacto. Se suma la regla de que
  el informe no lleva **un solo valor** de credencial: se nombran el archivo y su permiso.
- **`ENTORNO-DEVSAR.md`** (nuevo): el mapa medido de la VM —qué corre, por dónde escucha, las
  zonas sensibles y la lista de lo que no se toca. Es lo primero que se lee antes de la fase 1.
- **`EXPOSICION-Y-TUNELES.md`** (nuevo): clase propia. Los túneles rápidos de `cloudflared` son
  internet abierto, los puertos propios publicados en `0.0.0.0` quedan al alcance de la red
  interna, y cada servicio publicado necesita su propia autenticación.
- **`AI-AND-LLM.md`** reescrito para nuestros agentes: inyección de contexto (lo que el agente lee
  no manda), envenenamiento de lo persistente (memoria, skills, `AGENTS.md`), atado de acciones
  destructivas, identidad y credenciales, y la salida (nada de secretos en informes, PRs o logs).
- **`ATTACK-CLASSES.md`** en español y con nuestros ejemplos reales: la clave del memfs en texto
  plano en los logs de FFmpeg, las credenciales por línea de comandos visibles con `docker
  inspect`, el `/api/salud` público por el túnel, y el TLS que el receptor no valida.
- **`README.md`**: qué es el fork, qué se adaptó, qué se sacó y qué falta.

## Qué se sacó

- `DESKTOP-MOBILE-AND-LOCAL-IPC.md` — no hacemos apps nativas ni IPC local.
- `PROTOCOLS-RPC-AND-MESSAGING.md` — no usamos gRPC, brokers ni colas.

Las referencias cruzadas que quedaban apuntando a esos dos archivos se corrigieron.

## Verificación

- Los dos validadores siguen intactos y **verdes**:
  `node validate-findings.test.cjs` → 34 pruebas, 0 fallos;
  `node validate-coverage-ledger.test.cjs` → 31 pruebas, 0 fallos.
- `grep` de los archivos borrados: no queda ninguna referencia rota (sólo la mención intencional
  en el README, que explica por qué se sacaron).
- El mapa del entorno se midió contra la máquina (`docker ps`, `ss -ltnp`, `systemctl --user`,
  permisos de las carpetas sensibles), no se escribió de memoria.

## Qué falta (segunda pasada)

Los archivos de método heredados siguen en inglés: `RECONNAISSANCE.md`, `HUNTING.md`,
`VALIDATION-AND-REPORTING.md` y los compañeros de dominio. Funcionan y definen el formato, pero
falta traducirlos y reescribir los prompts de cazador con nuestro idioma y nuestro listón de
evidencia. Los validadores no hace falta tocarlos: validan estructura, no prosa.

## Hallazgo inmediato del reconocimiento

Al medir el entorno para escribir el mapa apareció lo primero que hay para revisar: **los puertos
propios están publicados en `0.0.0.0`** (el Core en 8080 y 8181, la UI en 3100, RTMP en
1935/1936), así que quedan al alcance de todo lo que esté en la red de la VM sin pasar por el
túnel. Corresponde a `EXPOSICION-Y-TUNELES.md`; todavía no se auditó ni se cambió nada.

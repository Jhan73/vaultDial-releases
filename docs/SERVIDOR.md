# Servidor de vaultDial

Guía para desplegar `vaultdial-server`, el servidor de sincronización y equipos de vaultDial 2.0. El servidor solo ve datos cifrados, claves públicas y los metadatos de *Qué ve el servidor*: ni la contraseña maestra ni nada derivado de ella sale nunca de los dispositivos.

El servidor registra cuentas, abre sesiones de dispositivo, empareja dispositivos, sincroniza bóvedas personales y de equipo (listas de miembros, roles, invitaciones, revocación con rotación de la clave de la bóveda), aplica las operaciones de identidad (revocar dispositivos, sustituir el kit, rotar, cancelar, impugnar, recuperar con el kit) y guarda el gobierno del equipo: solicitudes de acceso temporal, reservas exclusivas y el registro de auditoría, que borra solo los eventos de más de un año. La app se conecta desde la 2.0.

Cada versión de vaultDial publica la imagen del servidor de esa versión, `ghcr.io/jhan73/vaultdial-server:<versión>` (p. ej. `:2.0.0`), y esta guía junto a ella. Usa la misma versión que la app; `:latest` es siempre la última versión estable. La huella (digest) de cada imagen está en la página de su versión del repositorio de releases, por si quieres fijarla en el despliegue.

## Paso a paso: una instancia EC2

El camino más corto para un servidor de pruebas en AWS: una instancia EC2 con Docker, sin balanceador. Los detalles de cada paso están más abajo.

1. **Instancia:** Amazon Linux 2023 o Ubuntu, con una IP elástica. Grupo de seguridad: el puerto del servidor (`8443`) abierto solo a las IP de quienes van a conectar, y el `22` solo a la tuya. Con la imagen publicada basta una instancia pequeña.
2. **Docker y la imagen:** en Amazon Linux, `sudo dnf install -y docker && sudo systemctl enable --now docker`, y después `docker pull ghcr.io/jhan73/vaultdial-server:<versión>` (sección *Docker*).
3. **Certificado:**
   - Sin dominio: un certificado autofirmado para la IP elástica (modo 2), con `subjectAltName=IP:<ip elástica>`, y su huella con `vaultdial-server fingerprint`. El origen es `https://<ip elástica>:8443`.
   - Con un dominio que apunte a la IP elástica: un certificado público, p. ej. de Let's Encrypt (modo 1), y el origen `https://<dominio>:8443`, o `--listen 0.0.0.0:443` para que el origen quede sin puerto.
4. **Arrancar** el contenedor (sección *Docker*) con SQLite en el volumen `/data`, el origen del paso 3 y el certificado montado en `/certs`. Comprueba `curl -k https://<ip elástica>:8443/ready`: responde `ok`. Al poco, `docker ps` muestra el contenedor como `healthy` (sección *Docker*).
5. **Altas:** `docker exec vaultdial vaultdial-server token create --username <su correo>` para cada persona, y entrégale el código por un canal de confianza junto con el origen (y la huella, en el modo 2, por un canal distinto de la red).
6. **Copias:** programa la copia de SQLite (sección *Copias de seguridad*) y sácala de la instancia, p. ej. a S3 cifrado.

Para un servidor de equipo estable, el modo 3 (balanceador ALB con certificado de ACM) y PostgreSQL en RDS.

## Qué necesita

- **Una base de datos:**
  - SQLite, para una persona o una red local: un archivo, sin nada más que instalar.
  - PostgreSQL, para equipos y AWS (RDS o un PostgreSQL propio).
- **TLS siempre.** El cliente nunca habla HTTP.
- **Un origen fijo**, como `https://vault.empresa.com` o `https://192.168.1.20:8443`, que es el que escriben los clientes. Va en forma canónica, porque el cliente lo firma en cada sesión y debe coincidir letra por letra:
  - host en minúsculas, o una IPv6 entre corchetes;
  - sin barra final;
  - el puerto solo si no es el 443.

## Configuración

Se configura con opciones o con variables de entorno:

| Opción | Variable | Para qué |
|---|---|---|
| `--database` | `VAULTDIAL_DATABASE` | `sqlite:/data/server.db?mode=rwc` o `postgres://usuario:clave@host/base` |
| `--origin` | `VAULTDIAL_ORIGIN` | El origen que usan los clientes, en forma canónica |
| `--listen` | `VAULTDIAL_LISTEN` | Dirección y puerto de escucha (por defecto `0.0.0.0:8443`) |
| `--tls-cert`, `--tls-key` | `VAULTDIAL_TLS_CERT`, `VAULTDIAL_TLS_KEY` | Certificado (cadena PEM) y clave privada (PEM) |
| `--behind-proxy` | `VAULTDIAL_BEHIND_PROXY` | HTTP sin cifrar, **solo en loopback**, detrás de un proxy TLS en la misma máquina |
| `--trusted-proxy` | `VAULTDIAL_TRUSTED_PROXIES` | Direcciones o redes de los proxies cuyo `X-Forwarded-For` se cree, separadas por comas (por ejemplo, la subred del balanceador) |
| `--tls12` | `VAULTDIAL_TLS12` | Acepta también TLS 1.2: solo si el balanceador no habla TLS 1.3 con sus destinos |
| `--dispute-hours` | `VAULTDIAL_DISPUTE_HOURS` | Horas que espera una operación de identidad hecha sin el kit (revocar un dispositivo, sustituir el kit, rotar las claves), para que la persona pueda cancelarla si no fue ella (24 por defecto). Acórtalo solo en un servidor de una sola persona |
| `--recovery-hours` | `VAULTDIAL_RECOVERY_HOURS` | Horas que espera una recuperación con el kit a que otro dispositivo de la persona la apruebe o la deniegue (24 por defecto) |

Detalles del servidor:
- Solo acepta **TLS 1.3**.
- Las tablas se crean y se migran solas al arrancar.
- Para SQLite, crea el archivo con permisos solo para el usuario del servidor (`umask 077`): guarda claves envueltas y los metadatos de *Qué ve el servidor*.
- **PostgreSQL en otra máquina:** la URL debe verificar el certificado de la base de datos, con `sslmode=verify-full` y la autoridad de RDS (`sslrootcert=/certs/global-bundle.pem`); si no, el servidor no arranca. Quien pudiera ponerse en medio podría cambiar las claves de los dispositivos. Para una base de datos en la misma máquina no hace falta.
- La contraseña de la base de datos, mejor en `VAULTDIAL_DATABASE` que en la línea de órdenes, donde la vería `ps`.
- Comprobaciones de estado:
  - `GET /health` responde `ok` mientras el proceso vive;
  - `GET /ready` responde `ok` si además la base de datos contesta en 2 s, y `503` si no: es la que debe usar el health check de un balanceador (ALB). Tiene el mismo límite de peticiones por origen que las demás rutas sin sesión, y ninguna de las dos dice nada del servidor (ni la versión).
- Los registros (`tracing`) van a la salida de error; su nivel se ajusta con `RUST_LOG` (por ejemplo, `RUST_LOG=vaultdial_server=debug`), y con `VAULTDIAL_LOG_FORMAT=json` salen como un objeto JSON por línea, para un recolector de registros. Cada petición deja una línea con el método, la ruta como plantilla (`/v1/vaults/{vault_id}/push`), el código, la duración y un identificador que también se devuelve en la cabecera `x-request-id`: con él se encuentra la línea de una petición que un usuario reporta. Los `5xx` salen como errores y los rechazos (`4xx`) como avisos. Nunca incluyen cuerpos, la consulta de la URL, cabeceras (ni `Authorization` ni `Cookie`), tokens ni datos de las peticiones.

## Modos de TLS

**1. Certificado de una autoridad pública.** Para un nombre de dominio público, por ejemplo con Let's Encrypt. Los clientes lo validan como cualquier web:

```
vaultdial-server serve --database "sqlite:/data/server.db?mode=rwc" \
  --origin https://vault.empresa.com --listen 0.0.0.0:443 \
  --tls-cert /certs/fullchain.pem --tls-key /certs/privkey.pem
```

**2. Certificado autofirmado en la red local.** Es el caso de un NAS o un PC de la oficina. El cliente fija la huella del certificado al conectarse y después solo acepta ese certificado:

```
openssl req -x509 -newkey ec -pkeyopt ec_paramgen_curve:P-256 -nodes -days 3650 \
  -keyout key.pem -out cert.pem -subj "/CN=vaultdial" \
  -addext "subjectAltName=IP:192.168.1.20" \
  -addext "basicConstraints=critical,CA:FALSE" \
  -addext "extendedKeyUsage=serverAuth"
vaultdial-server fingerprint --tls-cert cert.pem     # huella SHA-256, para los clientes
```

La huella se comunica a quien vaya a conectar el primer dispositivo, por un canal que no sea la propia red. Si el certificado cambia, los clientes dejan de conectar hasta que se fija la huella nueva.

**3. AWS (EC2 con un balanceador ALB).**
- El ALB termina TLS con un certificado público de ACM y reenvía las peticiones al servidor **por HTTPS**: en el grupo de destino, protocolo HTTPS, puerto 8443 y comprobación de estado en `/ready`.
- El servidor usa un certificado autofirmado (modo 2). Así nunca hay HTTP sin cifrar en la red de la VPC, pero el ALB no valida ese certificado: el grupo de seguridad de la instancia solo debe admitir tráfico del ALB.
- `--trusted-proxy` con la subred del ALB, para que los límites de intentos cuenten a cada cliente y no al ALB; sin esa opción, todos los clientes comparten un mismo límite.
- El servidor solo habla TLS 1.3. Si las comprobaciones de estado del ALB fallan en el apretón de manos, añade `--tls12`.
- El origen es el nombre público del ALB, por ejemplo `https://vault.empresa.com`.
- El modo `--behind-proxy` **no** sirve para el ALB, porque el ALB no puede llegar al loopback de la instancia.

**4. Detrás de un proxy en la misma máquina** (Caddy, nginx). El proxy termina TLS y reenvía a `127.0.0.1`:

```
vaultdial-server serve --database "sqlite:/data/server.db?mode=rwc" \
  --origin https://vault.empresa.com --behind-proxy --listen 127.0.0.1:8080
```

- Fuera del loopback, el servidor se niega a arrancar en este modo.
- La IP del cliente, que usan los límites de intentos, se toma de la última entrada de `X-Forwarded-For`, la que añade el proxy.

## Altas de usuarios

Solo se registra quien tiene un **token de alta** de un solo uso:
- caduca a los 7 días por defecto, como máximo a los 30;
- el servidor guarda solo su SHA-256.

```
vaultdial-server token create --database … [--username ana@empresa.com] [--days 7]
vaultdial-server token list   --database …
vaultdial-server token revoke --database … <id>
```

- `create` muestra el token **una sola vez**, junto con un id corto que es lo único que muestra `list` y lo que usa `revoke`.
- Con `--username`, el token solo sirve para registrar ese nombre de usuario.
- El token se entrega a la persona por un canal de confianza.

## Cuentas congeladas

Una cancelación, una impugnación o dos operaciones de identidad incompatibles **congelan** la cuenta: es la señal de que otra persona puede tener uno de sus dispositivos o su kit. Mientras tanto, sus dispositivos solo pueden cancelar o impugnar, y no se sincroniza nada. La salida es fuera de banda:
- primero, comprobar con la persona (en persona o por teléfono) qué pasó;
- después, levantar la congelación:

```
vaultdial-server unfreeze --database … ana@empresa.com
```

Al levantarla se descartan todas las operaciones pendientes: quien aún quiera alguna la vuelve a pedir y espera de nuevo su plazo. Una congelación ligada a un periodo de prueba solo se levanta así.

## Borrar una cuenta vacía

Una cuenta creada por error que no tiene ninguna bóveda (p. ej. la de un dispositivo que se desconectó y eligió «Crear mi cuenta» en el mismo servidor) se borra con:

```
vaultdial-server users delete --database … bea@empresa.com
```

Revoca sus dispositivos y borra sus sesiones, su recuperación pendiente y sus operaciones en espera. El nombre de usuario queda reservado: nadie lo puede volver a registrar, para que no lo tome otra persona que el equipo conozca por él. Una cuenta que es propietaria o miembro de alguna bóveda no se borra.

## Docker

La imagen de cada versión se descarga de GitHub Container Registry:

```
docker pull ghcr.io/jhan73/vaultdial-server:2.0.0
docker run -d --name vaultdial --restart unless-stopped -p 8443:8443 \
  -v vaultdial-data:/data -v /ruta/certs:/certs:ro \
  -e VAULTDIAL_DATABASE="sqlite:/data/server.db?mode=rwc" \
  -e VAULTDIAL_ORIGIN=https://192.168.1.20:8443 \
  -e VAULTDIAL_TLS_CERT=/certs/cert.pem -e VAULTDIAL_TLS_KEY=/certs/key.pem \
  ghcr.io/jhan73/vaultdial-server:2.0.0
docker exec vaultdial vaultdial-server token create
```

Detalles de la imagen:
- Corre con un usuario sin privilegios (uid 10001), que debe poder leer la clave montada en `/certs`. Dale la clave a ese usuario (`chown 10001`, permisos 600) en lugar de hacerla legible para todos.
- Guarda los datos en el volumen `/data`, con permisos solo para ese usuario.
- `docker exec` hereda `VAULTDIAL_DATABASE`, así que la CLI no necesita `--database`.
- `docker stop` lo para de forma ordenada (atiende SIGTERM).
- **Comprobación de salud:** la imagen trae un `HEALTHCHECK` que cada 30 s ejecuta `vaultdial-server ready`: comprueba, como `/ready`, que el servidor acepta conexiones en su puerto y que la base de datos contesta en 2 s. `docker ps` muestra `healthy` o `unhealthy`. Lee `VAULTDIAL_DATABASE` y `VAULTDIAL_LISTEN`, así que configura el servidor con variables de entorno, no con opciones tras el nombre de la imagen, o la comprobación no sabrá dónde mirar.
- **Actualizar:** `docker pull` de la versión nueva, `docker stop` y `docker rm` del contenedor, y el mismo `docker run` con la nueva etiqueta. Los datos siguen en el volumen y las tablas se migran solas al arrancar. Haz antes una copia (sección *Copias de seguridad*).
- `/usr/share/doc/vaultdial-server/` contiene la licencia y los avisos de terceros.
- Con PostgreSQL no hace falta el volumen.

## Copias de seguridad

- **SQLite:** `sqlite3 server.db ".backup copia.db"`, con el servidor en marcha o parado. Si copias los archivos a mano con el servidor parado, copia también `server.db-wal` y `server.db-shm` si existen: pueden tener los últimos cambios.
- **PostgreSQL:** las copias de RDS, o `pg_dump`.

Las copias solo contienen datos cifrados, pero también los metadatos de *Qué ve el servidor*, así que hay que guardarlas como datos internos.

## Qué ve el servidor

El contenido de las bóvedas va cifrado desde el dispositivo: títulos, tipos, URLs, usuarios, notas, secretos, etiquetas, entornos, sistemas y nombres de bóveda. El servidor no tiene ninguna clave para descifrarlo. Sí ve:

- **Cuentas y dispositivos:** el nombre de usuario, las claves públicas de cada persona y de cada dispositivo, las altas y revocaciones de dispositivos y sus claves privadas envueltas (opacas). El nombre de cada dispositivo solo se entrega a los dispositivos de su usuario.
- **Sesiones:** IP, hora y frecuencia de conexión, y versión de la app.
- **Bóvedas:** si son personales o de equipo y la lista de miembros en claro: quién, con qué rol y hasta cuándo.
- **Objetos y actividad:** cuántas entradas tiene cada bóveda, sus versiones y conflictos, quién escribe y cuándo, y el tamaño de cada objeto, rellenado por tramos para ocultar la longitud exacta.
- **Equipo y gobierno:** invitaciones, aceptaciones, salidas, revocaciones; quién pide acceso a quién y cuándo (no a qué bóveda ni por qué), qué entrada (por id) está reservada y por quién, y los eventos de auditoría que informan los dispositivos: qué entrada (por id), qué acción, qué dispositivo y cuándo.

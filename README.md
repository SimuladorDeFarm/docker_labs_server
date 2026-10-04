# autostarlab

Despliega un laboratorio de **[DockerLabs](https://dockerlabs.es/)** en tu servidor Docker y le asigna una **IP propia en tu red local** (mediante una red **macvlan**, o **ipvlan** si solo hay WiFi), de forma que puedas atacarlo desde otra máquina **igual que una caja de Hack The Box**: todos los puertos accesibles directamente y sin chocar con los servicios reales del servidor (por ejemplo, su SSH).

El script se ejecuta **en el propio servidor Docker**: te conectas por SSH, descargas el `.tar` del lab con `wget` en la carpeta que contiene `auto_deploy.sh`, y lanzas `autostarlab`.

---

## ¿Por qué?

El `auto_deploy.sh` de DockerLabs levanta el contenedor en la red `bridge` interna (`172.17.0.x`), que **solo es alcanzable desde el propio servidor**. Además, exponer puertos con `-p` choca con los servicios reales del host (el caso típico: el `:22` del SSH real).

Con **macvlan**, el contenedor aparece en tu LAN como una máquina física más, con su propia IP (p. ej. `192.168.1.200`) y **todos sus puertos**. Así puedes hacer `nmap -p-`, fuerza bruta a SSH, etc., desde tu equipo de trabajo, y el ataque va al contenedor — nunca al servidor.

`macvlan` necesita Ethernet: no funciona sobre WiFi (el punto de acceso descarta las tramas con una MAC "extra" que no es la suya). Por eso, si el servidor solo tiene WiFi disponible (o el cable está desconectado), `autostarlab` usa automáticamente **`ipvlan` en modo L2**, que sí funciona sobre WiFi porque reutiliza la MAC real del adaptador.

---

## Requisitos

- Linux con **Docker** instalado y corriendo.
- `iproute2` (`ip`), utilidades base (`awk`, `find`). Opcional: `nmap` (para mostrar los puertos abiertos al final).
- Una interfaz de red (Ethernet o WiFi). El script **prefiere siempre Ethernet con enlace activo** (usa `macvlan`); si no la hay, cae automáticamente a la interfaz WiFi (usa `ipvlan` L2).
- Permisos de **root** (el script se re-ejecuta con `sudo` automáticamente si hace falta).

---

## Instalación

Clona el repo y coloca el script en tu `PATH`:

```bash
git clone <URL-de-tu-repo> docker_labs_server
cd docker_labs_server
chmod +x autostarlab
sudo install -m 755 autostarlab /usr/local/bin/autostarlab
```

A partir de ahí puedes invocar `autostarlab` desde cualquier carpeta.

---

## Uso

```
autostarlab [OPCIONES] [CARPETA]
```

`CARPETA` es la carpeta que contiene `auto_deploy.sh` y el `.tar` del lab. Si se omite, se usa la **carpeta actual**.

### Flujo típico (recomendado)

```bash
# 1) Conéctate al servidor
ssh usuario@servidor

# 2) Ve a la carpeta de tus labs (donde está auto_deploy.sh) y baja el lab
cd ~/docker_labs
wget https://dockerlabs.es/.../milab.tar

# 3) Lanza el lab (usa la carpeta actual)
autostarlab
```

Al terminar te imprime la IP del lab en tu LAN, por ejemplo:

```
==================================================================
  Lab 'milab' LISTO
  IP objetivo (LAN) : 192.168.1.200
  Puertos abiertos  : 22/tcp,80/tcp
  Desde OTRA maquina de la red (tu laptop, NO este servidor):
     nmap -p- -sV 192.168.1.200
     (si hay web)  http://192.168.1.200/
==================================================================
```

### Indicando la carpeta

```bash
autostarlab ~/docker_labs
```

---

## Opciones

| Opción | Descripción |
|---|---|
| `-t, --tar FICHERO` | `.tar` a desplegar. Por defecto: el único `.tar` de la carpeta (si hay varios, es obligatorio). |
| `-a, --addr IP` | IP fija a asignar en la LAN. Por defecto: primera libre a partir de `.200`. |
| `-i, --iface IFAZ` | Interfaz padre. Por defecto: Ethernet con enlace si la hay; si no, WiFi. |
| `-n, --network NOMBRE` | Nombre de la red (macvlan o ipvlan). Por defecto: `labnet`. |
| `--deploy RUTA` | Ruta a `auto_deploy.sh`. Por defecto: `CARPETA/auto_deploy.sh`. |
| `-s, --stop` | Elimina el lab (para el contenedor y libera la IP). |
| `-h, --help` | Muestra la ayuda. |
| `-V, --version` | Muestra la versión. |

### Ejemplos

```bash
autostarlab                                  # despliega el lab de la carpeta actual
autostarlab ~/docker_labs                    # despliega el lab de esa carpeta
autostarlab -a 192.168.1.210 ~/docker_labs   # con IP fija concreta
autostarlab -t milab.tar ~/docker_labs       # si hay varios .tar en la carpeta
autostarlab -i eth0 ~/docker_labs            # forzando la interfaz padre
autostarlab -s ~/docker_labs                 # elimina el lab desplegado
```

---

## ⚠️ Importante: desde dónde atacar

Por diseño de macvlan/ipvlan, **el propio servidor NO puede comunicarse con su contenedor**. Si haces `ping`/`nmap` a la IP del lab **desde el mismo servidor**, verás *"host down"* o *"Destination Host Unreachable"*.

👉 **Ataca y navega siempre desde OTRA máquina de la red** (tu laptop). Cualquier equipo de la LAN alcanza la IP del lab sin problema; solo el host que lo ejecuta no.

---

## Cómo funciona (resumen técnico)

1. Se asegura de correr como root (si no, se relanza con `sudo`).
2. Autodetecta la interfaz: recorre las interfaces Ethernet buscando una con **enlace físico activo** (`carrier=1`); si ninguna lo tiene, usa la interfaz de la ruta por defecto (típicamente la WiFi).
3. Según el tipo de interfaz elegida, decide el driver: **`macvlan`** sobre Ethernet, **`ipvlan` (modo L2)** sobre WiFi.
4. Calcula la **subred** y el **gateway** a partir de `ip route`, y crea (si no existe, o la recrea si existe con otro driver/interfaz y no tiene contenedores) la red Docker correspondiente.
5. Lanza `auto_deploy.sh` **detachado** (`setsid`+`disown`), porque ese script no termina: se queda en un bucle a la espera de `Ctrl+C` para borrar el lab. El contenedor corre en segundo plano (`docker run -d`), así que sobrevive de forma independiente. Una vez arrancado, `autostarlab` termina ese proceso (`SIGTERM`, no `INT`) para no acumular procesos colgados en despliegues sucesivos.
6. Conecta el contenedor a la red con la IP elegida (limpiando primero cualquier endpoint obsoleto de un despliegue anterior).
7. (Opcional, si hay `nmap`) muestra los puertos abiertos del contenedor.

Para eliminar el lab (`-s`), desconecta la red, envía `INT` a `auto_deploy.sh` para que ejecute su limpieza nativa y, como respaldo, fuerza el borrado del contenedor.

---

## Solución de problemas

- **"host down" / no responde**: lo estás escaneando desde el propio servidor. Hazlo desde otra máquina de la red.
- **No arranca / cuelga en "docker load"**: el `.tar` es grande; espera. Si falla, revisa el log en `/tmp/autostarlab_<lab>.log`.
- **El script detectó la interfaz equivocada**: fuérzala con `-i IFAZ`.
- **"La red existente es X sobre Y, no Z sobre W; la recreo" y falla**: hay contenedores aún conectados a esa red; elimínalos antes con `autostarlab -s`.
- **La IP `.200` ya está en uso**: pásala explícita con `-a`, o deja que el script elija la primera libre.
- **Choque con el DHCP del router**: usa una IP fuera del rango que reparte tu router (revisa la config del router).

---

## Licencia

MIT

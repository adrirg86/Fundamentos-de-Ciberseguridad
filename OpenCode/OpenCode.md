# OpenCode en Kali Linux: agente de IA en la terminal

> **Autor:** Adrián Rincón Guzina
> **Fecha:** 04/11/2026
> **Entorno:** Kali Linux

## Índice

1. [Introducción y objetivo](#1-introducción-y-objetivo)
2. [⚠️ Advertencia de seguridad](#2-️-advertencia-de-seguridad)
3. [Requisitos previos](#3-requisitos-previos)
4. [Instalación](#4-instalación)
5. [Primer uso](#5-primer-uso)
6. [Conectar un proveedor de IA](#6-conectar-un-proveedor-de-ia)
7. [Comandos básicos](#7-comandos-básicos)
8. [Modos de trabajo: Plan y Build](#8-modos-de-trabajo-plan-y-build)
9. [Gestión de sesiones](#9-gestión-de-sesiones)
10. [Control de consumo (tokens y espacio)](#10-control-de-consumo-tokens-y-espacio)
11. [Configuración y permisos](#11-configuración-y-permisos)
12. [Uso por SSH](#12-uso-por-ssh)
13. [Buenas prácticas](#13-buenas-prácticas)
14. [Solución de problemas](#14-solución-de-problemas)
15. [Desinstalación](#15-desinstalación)
16. [Referencias](#16-referencias)

---

## 1. Introducción y objetivo

**OpenCode** es un agente de IA de código abierto que funciona en la terminal (TUI). Permite pedirle tareas en lenguaje natural: leer archivos, ejecutar comandos, editar código, analizar logs, etc. Puede usarse con distintos proveedores de modelos (Anthropic/Claude, OpenAI, modelos locales, etc.).

**Objetivo de esta instalación:** enlazar Claude con Kali mediante SSH para poder realizar cualquier consulta o acción sobre nuestro equipo desde la propia terminal.

## 2. ⚠️ Advertencia de seguridad

Darle poder a una IA sobre tu sistema operativo tiene un **alto riesgo**. El agente puede ejecutar comandos reales: borrar archivos, modificar configuraciones o lanzar herramientas de pentesting. Recomendaciones mínimas:

- Usar una **máquina virtual** o un entorno de laboratorio, **nunca** tu equipo principal con datos importantes.
- Hacer una **snapshot** de la VM antes de empezar.
- Trabajar con un **usuario sin privilegios**, no como `root`.
- Mantener los permisos en modo **"ask"** (pedir confirmación) para comandos y ediciones.
- **Leer cada comando** antes de aprobarlo.
- No exponer claves, contraseñas ni tokens en los prompts: el contenido se envía al proveedor del modelo.
- Usar las herramientas de seguridad **solo contra sistemas propios o con autorización expresa**.

## 3. Requisitos previos

| Requisito | Detalle |
|---|---|
| Sistema | Kali Linux (Debian-based) |
| Conexión | Acceso a Internet |
| Herramientas | `curl`, `bash`, `git` (recomendado) |
| Terminal | Con soporte de colores reales y buena fuente (para la interfaz TUI) |
| Cuenta de IA | API key o suscripción de un proveedor compatible |

Comprobar herramientas:

```bash
curl --version
git --version
```

<img width="959" height="232" alt="image" src="https://github.com/user-attachments/assets/4c4e88a0-6562-4e99-be82-f2b786916e1f" />



## 4. Instalación

Instalamos OpenCode con el script oficial:

```bash
curl -fsSL https://opencode.ai/v2/install | bash
```

> 💡 Revisa siempre el script antes de ejecutarlo (`curl -fsSL <url> | less`) y confirma la URL en la documentación oficial del proyecto.


<img width="962" height="413" alt="Pasted image 20261104132138" src="https://github.com/user-attachments/assets/ebc0fed9-e4cb-4451-90ee-5690f815441f" />



Si el instalador añade OpenCode al `PATH` en `~/.bashrc` o `~/.zshrc`, recarga la shell:

```bash
source ~/.zshrc    # Kali usa zsh por defecto (o ~/.bashrc si usas bash)
```

Verificar la instalación:

```bash
opencode --version
```

## 5. Primer uso

Con el comando `opencode` abres la IA en el directorio actual:

```bash
cd ~/mi-proyecto
opencode
```

<img width="965" height="522" alt="Pasted image 20261104132606" src="https://github.com/user-attachments/assets/033e45e5-eccd-4ca1-ac81-85a2f79dcf00" />


Desde ahí podemos pedirle cualquier cosa en lenguaje natural. En la interfaz se puede ver el **porcentaje de contexto utilizado** y los **tokens gastados**.


<img width="957" height="1035" alt="Pasted image 20261104133219" src="https://github.com/user-attachments/assets/dafbdcc5-d848-4319-b372-cf164d953e81" />


Si le damos una orden de acción, el agente explica qué va a realizar y pide confirmación según los permisos configurados.


<img width="957" height="1048" alt="Pasted image 20261104132900" src="https://github.com/user-attachments/assets/69b4028e-806b-47c0-accd-7c728d49199f" />


## 6. Conectar un proveedor de IA

Para usar Claude (u otro modelo) hay que autenticarse:

```bash
opencode auth login
```

También se puede hacer desde dentro de la interfaz con el comando `/connect`. Para elegir el modelo:

```
/models
```

Ver los modelos disponibles desde la terminal:

```bash
opencode models
```

> 🔑 No subas nunca tus claves a GitHub. Las credenciales se guardan localmente (normalmente en `~/.local/share/opencode/`).

## 7. Comandos básicos

### Comandos de la interfaz (slash commands)

| Comando | Descripción |
|---|---|
| `/new` | Inicia una nueva sesión |
| `/sessions` | Lista las sesiones y permite cambiar entre ellas |
| `/models` | Cambia el modelo de IA |
| `/connect` | Añade o configura un proveedor |
| `/init` | Analiza el proyecto y genera un archivo `AGENTS.md` con contexto |
| `/undo` | Deshace el último cambio realizado por el agente |
| `/redo` | Rehace un cambio deshecho |
| `/share` | Genera un enlace para compartir la conversación |
| `/help` | Muestra la ayuda |
| `/exit` | Cierra OpenCode |

### Comandos de terminal

| Comando | Descripción |
|---|---|
| `opencode` | Abre la interfaz interactiva |
| `opencode run "mensaje"` | Ejecuta una consulta puntual sin abrir la TUI |
| `opencode auth login` | Inicia sesión en un proveedor |
| `opencode models` | Lista los modelos disponibles |
| `opencode upgrade` | Actualiza OpenCode |
| `opencode --help` | Ayuda general |

Ejemplo de uso no interactivo:

```bash
opencode run "Explícame qué hace el script backup.sh"
```

> ℹ️ Los comandos pueden variar según la versión. Comprueba `opencode --help` y `/help`.

## 8. Modos de trabajo: Plan y Build

OpenCode incluye dos agentes principales. Se alterna entre ellos con la tecla **Tab**:

- **Plan:** solo analiza y propone un plan. No modifica archivos ni ejecuta cambios. Ideal para **explorar con seguridad**.
- **Build:** puede editar archivos y ejecutar comandos para llevar a cabo las tareas.

✅ **Recomendación:** empieza siempre en modo **Plan** y pasa a **Build** cuando hayas revisado qué va a hacer.

## 9. Gestión de sesiones

- `/new` → crea otra sesión (conversación independiente, con su propio contexto).
- `/sessions` → permite **ver tus sesiones** y **cambiar** entre ellas.

Usa sesiones separadas por tarea para no saturar el contexto y gastar menos tokens.

## 10. Control de consumo (tokens y espacio)

### Tokens y contexto

La interfaz muestra el porcentaje de contexto usado y los tokens consumidos. Cuando el contexto se llena, el rendimiento baja y el coste sube: abre una sesión nueva con `/new`.

### Espacio en disco

Ver el espacio consumido por la caché y la configuración:

```bash
du -sh ~/.cache/opencode ~/.config/opencode 2>/dev/null
```

Rutas habituales:

| Ruta | Contenido |
|---|---|
| `~/.config/opencode/` | Configuración (`opencode.json`) |
| `~/.cache/opencode/` | Caché |
| `~/.local/share/opencode/` | Datos: sesiones, credenciales, registros |

Limpiar la caché si ocupa demasiado:

```bash
rm -rf ~/.cache/opencode
```

## 11. Configuración y permisos

El archivo de configuración global es `~/.config/opencode/opencode.json`. También puede existir un `opencode.json` en la raíz de cada proyecto.

Ejemplo para **pedir confirmación** antes de editar archivos o ejecutar comandos:

```json
{
  "$schema": "https://opencode.ai/config.json",
  "permission": {
    "edit": "ask",
    "bash": "ask"
  }
}
```

Valores posibles: `"ask"` (preguntar), `"allow"` (permitir siempre), `"deny"` (denegar).

Ejemplo más restrictivo: permitir solo comandos de lectura y pedir confirmación para el resto:

```json
{
  "$schema": "https://opencode.ai/config.json",
  "permission": {
    "edit": "ask",
    "bash": {
      "ls *": "allow",
      "cat *": "allow",
      "git status": "allow",
      "rm *": "deny",
      "*": "ask"
    }
  }
}
```

> ⚠️ Evita `"allow"` global en `bash` si trabajas en un sistema que te importa.

### Archivo `AGENTS.md`

Con `/init`, OpenCode genera un `AGENTS.md` en el proyecto con instrucciones y contexto para el agente. Puedes editarlo para añadir reglas, por ejemplo: *"No ejecutes comandos con sudo"*.

## 12. Uso por SSH

Para trabajar con OpenCode en Kali desde otro equipo:

```bash
ssh usuario@IP_DE_KALI
cd ~/mi-proyecto
opencode
```

Buenas prácticas con SSH:

- Usar **claves SSH** en lugar de contraseña:

  ```bash
  ssh-keygen -t ed25519
  ssh-copy-id usuario@IP_DE_KALI
  ```

- Desactivar el acceso de `root` por SSH y la autenticación por contraseña en `/etc/ssh/sshd_config`:

  ```
  PermitRootLogin no
  PasswordAuthentication no
  ```

- Activar el servicio solo cuando se necesite:

  ```bash
  sudo systemctl start ssh
  sudo systemctl stop ssh
  ```

- Usar **tmux** para que la sesión no se pierda si se corta la conexión:

  ```bash
  sudo apt install tmux
  tmux new -s opencode
  opencode
  # Desconectar: Ctrl+B, luego D
  # Reconectar:  tmux attach -t opencode
  ```

- Si se expone a una red no confiable, limitarlo con firewall o acceder mediante VPN/túnel.

## 13. Buenas prácticas

- Trabaja en una **VM con snapshot** y un usuario sin privilegios.
- Empieza en **modo Plan**.
- Mantén los permisos en **ask**.
- **Lee cada comando** antes de aprobarlo.
- Usa **git** en los proyectos: así puedes revertir cualquier cambio del agente.
- Una sesión por tarea (`/new`).
- Sé específico en los prompts: indica ruta, objetivo y qué **no** debe tocar.
- No pegues datos sensibles (claves, contraseñas, datos personales).
- Haz copias de seguridad antes de tareas destructivas.

## 14. Solución de problemas

| Problema | Posible solución |
|---|---|
| `opencode: command not found` | Recarga la shell (`source ~/.zshrc`) o comprueba que el directorio de instalación esté en el `PATH` |
| La interfaz se ve mal | Usa una terminal con colores reales (`echo $COLORTERM` debería mostrar `truecolor`) |
| Error de autenticación | Repite `opencode auth login` o `/connect` y comprueba la API key |
| Sin conexión | Revisa red, proxy y DNS |
| Contexto lleno / respuestas lentas | Abre una sesión nueva con `/new` |
| Mucho espacio ocupado | Revisa con `du -sh` y limpia `~/.cache/opencode` |

## 15. Desinstalación

Elimina el binario (según el método de instalación, suele estar en `~/.opencode/` o `~/.local/bin/`) y los datos:

```bash
rm -rf ~/.opencode
rm -rf ~/.config/opencode ~/.cache/opencode ~/.local/share/opencode
```

Y quita la línea del `PATH` que añadió el instalador en `~/.zshrc` o `~/.bashrc`.

## 16. Referencias

- Sitio oficial: <https://opencode.ai>
- Documentación: <https://opencode.ai/docs>
- Repositorio: <https://github.com/sst/opencode>

---

> Documento educativo. Úsalo solo en entornos propios o de laboratorio autorizados.

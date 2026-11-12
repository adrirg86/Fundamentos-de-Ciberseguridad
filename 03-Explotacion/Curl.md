# Curl — Lectura de Archivos Remotos (LFI / Divulgación de Información)

`curl` es una herramienta de línea de comandos para realizar peticiones HTTP/HTTPS (y otros protocolos) directamente desde la terminal. En el contexto de explotación, se usa habitualmente **después** de haber identificado con Gobuster/FFUF un parámetro vulnerable a LFI, para confirmar la vulnerabilidad y extraer el contenido de archivos sensibles del servidor.

---

## 1. Confirmar LFI leyendo `/etc/passwd`

```bash
curl -s "http://172.17.0.2/index.php?secret=/etc/passwd"
```

### Explicación de cada parte

| Componente | Significado |
|---|---|
| `-s` | Modo *silent*: oculta la barra de progreso y los mensajes informativos de curl, mostrando únicamente la respuesta del servidor. Imprescindible cuando se quiere volcar o procesar la salida (por ejemplo, con `grep` o guardarla en un archivo) sin ruido añadido. |
| `"http://172.17.0.2/..."` | URL objetivo entre comillas, necesario porque contiene caracteres especiales (`?`, `=`, `/`) que podrían interpretarse mal en la shell si no se escapan. |
| `?secret=/etc/passwd` | Parámetro GET `secret` con la ruta del archivo a incluir. `/etc/passwd` es el archivo "de prueba" por excelencia en LFI: es legible por cualquier usuario en sistemas UNIX y su formato es muy reconocible, lo que permite confirmar rápidamente si el parámetro es vulnerable sin necesidad de saber de antemano qué archivos existen. |

**Objetivo:** comprobar que el parámetro `secret` del script `index.php` no está saneado y permite incluir rutas arbitrarias del sistema de archivos del servidor — es decir, confirmar la existencia de un **Local File Inclusion**.

---

## 2. Explotar el LFI para robar una clave SSH privada

```bash
curl -s "http://172.17.0.2/index.php?secret=/home/'user'/.ssh/id_rsa"
```

### Explicación de cada parte

| Componente | Significado |
|---|---|
| `?secret=/home/'user'/.ssh/id_rsa` | Una vez confirmado el LFI con `/etc/passwd`, se apunta a un objetivo de mucho más valor: la clave privada SSH del usuario. `'user'` es un placeholder que se sustituye por el nombre de usuario real (normalmente obtenido de la respuesta anterior, al leer `/etc/passwd`, donde aparecen listados todos los usuarios del sistema). |
| `.ssh/id_rsa` | Ruta estándar donde OpenSSH guarda la clave privada por defecto del usuario. Si el archivo es legible y se consigue su contenido completo, permite autenticarse por SSH como ese usuario **sin necesidad de contraseña**. |

**Objetivo:** escalar de una simple lectura de archivo a la obtención de credenciales reutilizables (la clave SSH), que normalmente se usarán después con `ssh -i id_rsa user@ip` para obtener acceso directo a la máquina.

>  **Flujo típico de explotación:** primero se confirma el LFI con un archivo predecible (`/etc/passwd`), después se identifican los usuarios reales del sistema a partir de esa respuesta, y finalmente se apunta a rutas sensibles específicas de cada usuario (`.ssh/id_rsa`, `.bash_history`, archivos de configuración con credenciales, etc.).

---

## Resumen de flags vistos

| Flag | Categoría | Función |
|---|---|---|
| `-s` | Salida | Modo silencioso, oculta progreso y solo muestra la respuesta |
| `"url"` | Sintaxis | Comillas para evitar que la shell interprete caracteres especiales de la URL |

---

## Otras opciones de curl útiles en este mismo contexto

Aunque no aparecen en los ejemplos originales, estas son extensiones naturales del mismo flujo de trabajo:

| Flag | Función |
|---|---|
| `-o archivo` | Guarda la respuesta en un archivo en lugar de mostrarla por pantalla (útil para volcar una clave SSH directamente a `id_rsa`). |
| `-i` | Incluye las cabeceras HTTP de la respuesta, útil para depurar si el servidor está devolviendo un error en vez del contenido esperado. |
| `--path-as-is` | Evita que curl "normalice" rutas con `../`, algo relevante en LFI vía *path traversal* (`?secret=../../../etc/passwd`), ya que por defecto curl podría simplificar la ruta antes de enviarla. |
| `-G --data-urlencode` | Permite construir la query string codificando correctamente caracteres especiales, útil cuando la ruta a incluir contiene espacios o símbolos problemáticos. |

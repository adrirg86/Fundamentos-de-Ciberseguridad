# Escalada de Privilegios en Linux — Enumeración de Sudo y SUID

Una vez conseguido acceso inicial a una máquina (por ejemplo, mediante una reverse shell), el siguiente paso habitual es la **enumeración de privilegios**: comprobar qué puede hacer el usuario actual más allá de lo obvio, buscando vías para escalar a un usuario con más permisos (idealmente `root`). Los dos métodos más básicos y con mayor tasa de éxito son revisar los permisos `sudo` del usuario y buscar binarios con el bit **SUID** activo.

---

## 1. Ver permisos sudo del usuario actual

```bash
sudo -l
```

### Explicación

| Componente | Significado |
|---|---|
| `sudo` | Comando que permite ejecutar otra instrucción con privilegios elevados (normalmente de `root`), según lo definido en `/etc/sudoers`. |
| `-l` | *List*: en lugar de ejecutar un comando, **lista** qué comandos puede ejecutar el usuario actual con `sudo`, sin necesidad de tener acceso al archivo `/etc/sudoers` directamente. |

**Qué buscar en la salida:**
- Comandos que el usuario puede ejecutar **sin contraseña** (aparecen con `NOPASSWD:`), ya que son inmediatamente explotables.
- Binarios permitidos que sean conocidos por permitir escapar a una shell de `root` (por ejemplo `vim`, `less`, `find`, `python`, `awk`, `nmap` con ciertas versiones...). Suelen consultarse contra listados como [GTFOBins](https://gtfobins.github.io/) para saber si el binario permitido tiene una vía documentada de escalada.
- Si el usuario puede ejecutar **cualquier comando** (`(ALL : ALL) ALL`), la escalada es directa.

> 💡 `sudo -l` a veces pide la contraseña del usuario actual antes de mostrar el listado. Si no se dispone de ella, este vector queda descartado y hay que centrarse en otros (como SUID).

---

## 2. Buscar binarios con el bit SUID activo

```bash
find / -perm -4000 -type f 2>/dev/null
```

```bash
find / -perm -4000 2>/dev/null
```

### Explicación

| Componente | Significado |
|---|---|
| `find /` | Busca a partir de la raíz del sistema de archivos (`/`), es decir, en todo el sistema. |
| `-perm -4000` | Filtra por permisos: `4000` en octal corresponde al bit **SUID** (*Set User ID*). Un binario con este bit activo se ejecuta siempre con los permisos de su **propietario**, no con los del usuario que lo lanza. Si el propietario es `root`, cualquier usuario que ejecute ese binario lo hace momentáneamente con privilegios de `root`. El prefijo `-` antes de `4000` significa "que tenga al menos estos permisos activos", no una coincidencia exacta. |
| `-type f` | Restringe la búsqueda a archivos regulares, excluyendo directorios y otros tipos de entrada (en la primera versión del comando). |
| `2>/dev/null` | Redirige el *standard error* a `/dev/null`, descartándolo. Es imprescindible en este tipo de búsquedas porque `find /` genera muchísimos errores de "permiso denegado" al intentar acceder a carpetas restringidas, y sin este filtro la salida útil queda enterrada entre ese ruido. |

**Diferencia entre ambos comandos:** el primero (`-type f`) limita los resultados a archivos, mientras que el segundo, al no especificar `-type`, también podría devolver directorios u otros elementos con ese permiso (algo mucho más raro, pero posible). En la práctica, casi siempre interesa la versión con `-type f`, ya que los binarios SUID explotables son archivos ejecutables.

**Qué buscar en la salida:**
- Binarios fuera de la lista "esperada" del sistema (`/usr/bin/passwd`, `/usr/bin/sudo`, `/usr/bin/su`, etc., que tienen SUID por diseño y no son explotables por sí mismos).
- Cualquier binario inusual, propietario de terceros, o una versión de una herramienta común (`find`, `cp`, `python3`, `bash`...) con SUID activado manualmente — señal casi segura de una máquina configurada intencionadamente como vector de escalada (muy común en CTFs).
- Igual que con `sudo -l`, contrastar los binarios encontrados contra [GTFOBins](https://gtfobins.github.io/) para ver si existe una técnica de escape documentada.

---

## Resumen de comandos vistos

| Comando | Función |
|---|---|
| `sudo -l` | Lista los comandos que el usuario actual puede ejecutar con privilegios elevados |
| `find / -perm -4000 -type f 2>/dev/null` | Busca archivos con bit SUID activo en todo el sistema |
| `find / -perm -4000 2>/dev/null` | Igual que el anterior, pero sin restringir a archivos (incluye cualquier tipo con ese permiso) |

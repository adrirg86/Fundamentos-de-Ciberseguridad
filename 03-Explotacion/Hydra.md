# Hydra — Ataques de Fuerza Bruta contra Servicios (SSH)

`Hydra` (THC-Hydra) es una herramienta de fuerza bruta orientada a **servicios de red** (SSH, FTP, RDP, HTTP, SMB, bases de datos...), a diferencia de Gobuster o FFUF que trabajan sobre HTTP. Se usa típicamente en la fase de explotación, cuando ya se dispone de credenciales parciales (un usuario conocido, una lista de usuarios, o una contraseña sospechada) y se quiere completar la pareja usuario/contraseña válida.

---

## 1. SSH con usuario fijo y diccionario de contraseñas

```bash
hydra -l 'user' -P /usr/share/wordlists/rockyou.txt ssh://'ip'
```

### Explicación de cada flag

| Flag | Significado |
|---|---|
| `-l 'user'` | *Login* fijo (minúscula = un único usuario). Se usa cuando ya se conoce el nombre de usuario válido (por ejemplo, obtenido por enumeración previa) y solo falta la contraseña. |
| `-P /usr/share/wordlists/rockyou.txt` | *Password list* (mayúscula = diccionario de contraseñas). `rockyou.txt` es el diccionario de contraseñas filtradas más usado en este tipo de ataques, tanto en CTFs como en pentests reales. |
| `ssh://'ip'` | Protocolo y objetivo. Indica que el ataque va dirigido al servicio SSH de la IP indicada. |

**Objetivo:** probar, una por una, todas las contraseñas del diccionario contra el usuario `user` por SSH, hasta encontrar la combinación correcta.

> ⚠️ La regla mnemotécnica de Hydra: **minúscula = un solo valor** (`-l`, `-p`), **mayúscula = lista/fichero** (`-L`, `-P`).

---

## 2. SSH con lista de usuarios y contraseña fija en puerto específico

```bash
hydra -L /usr/share/wordlists/rockyou.txt \
  -p "lapassworddebackupmaschingonadetodas" \
  -s 5000 172.17.0.2 ssh
```

### Explicación de cada flag

| Flag | Significado |
|---|---|
| `-L /usr/share/wordlists/rockyou.txt` | Lista de usuarios (mayúscula = fichero). Aquí se invierte la lógica del ejemplo anterior: se prueban muchos nombres de usuario contra **una sola contraseña** conocida. |
| `-p "..."` | Contraseña fija (minúscula = un solo valor). Se usa cuando se ha encontrado una contraseña concreta (por ejemplo, en un backup, un archivo de configuración o un `.git`) y se quiere comprobar a qué usuario(s) pertenece. |
| `-s 5000` | Puerto del servicio. Por defecto Hydra usa el puerto estándar de cada protocolo (22 para SSH), pero aquí el servicio SSH escucha en un puerto no estándar (`5000`), algo común cuando se quiere "ocultar" el servicio del escaneo automático. |
| `172.17.0.2 ssh` | IP objetivo y protocolo, en formato posicional (alternativa a `protocolo://ip`). |

**Nota importante:** aunque el nombre de la variable use `rockyou.txt` como wordlist de usuarios, en la práctica esto no tendría sentido real (rockyou es un diccionario de contraseñas, no de usuarios) — probablemente en el caso real se sustituiría por una wordlist de nombres de usuario. Vale la pena revisarlo antes de lanzar el ataque.

**Objetivo:** identificar qué usuario del sistema usa esa contraseña de backup filtrada, contra el servicio SSH en el puerto 5000.

---

## 3. Filtrar usuarios de un fichero y ataque SSH rápido con multithreading

```bash
cat users.txt | grep -oE 'pedro|valentina' | tee users
hydra -L users -p purpl3 -u ssh://$target -t 64
```

### Explicación de cada comando

**Paso 1 — Filtrado de usuarios:**

| Componente | Significado |
|---|---|
| `cat users.txt` | Vuelca el contenido del fichero de usuarios candidatos. |
| `grep -oE 'pedro\|valentina'` | Filtra **solo** las coincidencias exactas de los patrones `pedro` o `valentina` dentro del fichero (`-o` muestra solo la parte coincidente, `-E` activa expresiones regulares extendidas para poder usar el `\|` como "o lógico"). Útil cuando `users.txt` es un listado grande (por ejemplo, extraído de LinkedIn o de metadatos) y solo interesan un par de nombres ya identificados como relevantes. |
| `tee users` | Muestra el resultado por pantalla **y** lo guarda en un nuevo fichero llamado `users`, para poder usarlo directamente como wordlist en el siguiente comando. |

**Paso 2 — Ataque Hydra:**

| Flag | Significado |
|---|---|
| `-L users` | Lista de usuarios: el fichero filtrado en el paso anterior (solo `pedro` y `valentina`). |
| `-p purpl3` | Contraseña fija a probar contra ambos usuarios. |
| `-u` | Indica a Hydra que **pruebe primero todos los usuarios con la misma contraseña**, en lugar del orden por defecto (recorrer todas las contraseñas para cada usuario). Con pocas contraseñas y varios usuarios, este orden es más eficiente. |
| `ssh://$target` | Objetivo, usando una variable de entorno `$target` en lugar de la IP escrita a mano — práctica habitual para reutilizar el comando entre distintas máquinas sin editarlo. |
| `-t 64` | Número de tareas en paralelo (*tasks*). Un valor alto como 64 acelera mucho el ataque, aunque en SSH real puede provocar bloqueos por parte de mecanismos anti fuerza bruta (fail2ban, etc.); en máquinas de laboratorio no suele ser un problema. |

**Objetivo:** a partir de un listado de usuarios ya reducido a los dos candidatos más probables, lanzar un ataque rápido y agresivo con una única contraseña conocida.

---

## Resumen de flags vistos

| Flag | Categoría | Función |
|---|---|---|
| `-l` | Usuario | Login único (minúscula) |
| `-L` | Usuario | Fichero con lista de usuarios (mayúscula) |
| `-p` | Contraseña | Contraseña única (minúscula) |
| `-P` | Contraseña | Fichero con lista de contraseñas (mayúscula) |
| `-s` | Conexión | Puerto específico del servicio |
| `-u` | Orden de ataque | Prioriza recorrer usuarios antes que contraseñas |
| `-t` | Rendimiento | Número de tareas/hilos en paralelo |
| `protocolo://ip` o `ip protocolo` | Objetivo | Dos sintaxis válidas para indicar servicio + host |


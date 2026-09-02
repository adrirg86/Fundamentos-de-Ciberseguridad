# FFUF — Fuzzing de Parámetros, Rutas y Subdominios

`ffuf` (Fuzz Faster U Fool) es una herramienta de *fuzzing* web escrita en Go, muy usada como alternativa (o complemento) a Gobuster. Su punto fuerte es la flexibilidad: el marcador `FUZZ` se puede colocar en **cualquier parte** de la petición (URL, parámetro GET, cabecera, cuerpo POST, incluso en la propia wordlist), lo que la hace ideal para descubrir directorios, subdominios, parámetros vulnerables (LFI, IDOR...) o cualquier combinación de estos.

---

## 1. Fuzzing de un parámetro GET para LFI (Local File Inclusion)

```bash
ffuf -u "http://172.17.0.2/index.php?FUZZ=/etc/passwd" \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
  -fs 2596
```

<img width="967" height="491" alt="image" src="https://github.com/user-attachments/assets/7f3686ed-e298-4b6d-9496-c13d3916cc4a" />


### Explicación de cada flag

| Flag | Significado |
|---|---|
| `-u` | URL objetivo. El marcador `FUZZ` no está en la ruta, sino en el **nombre del parámetro GET** (`?FUZZ=/etc/passwd`). Esto significa que ffuf va a ir probando distintos nombres de parámetro (`file`, `page`, `path`, `include`...) manteniendo fijo el valor `/etc/passwd`. |
| `-w` | Wordlist con posibles nombres de parámetro a probar. |
| `-fs 2596` | *Filter Size*: excluye de los resultados las respuestas cuyo tamaño sea exactamente 2596 bytes. Es la técnica clásica para eliminar el "ruido" de la página de error por defecto (que se repite en todas las peticiones fallidas) y quedarte solo con las respuestas realmente distintas, que delatan un parámetro válido. |

**Objetivo:** encontrar qué nombre de parámetro (`FUZZ`) acepta la aplicación como ruta de archivo, para explotar una vulnerabilidad de tipo **LFI**. Si al probar con `/etc/passwd` como valor fijo el tamaño de respuesta cambia respecto al "ruido" filtrado por `-fs`, es señal de que el parámetro es vulnerable y el servidor está leyendo el archivo.

> 💡 Este flujo suele venir después de haber confirmado manualmente (con un solo parámetro, por ejemplo `?file=/etc/passwd`) que existe LFI, y ahora se automatiza la búsqueda de **qué parámetro concreto** es el vulnerable si no se conoce su nombre.

---

## 2. Fuzzing de rutas con filtrado por código de estado y threads

```bash
ffuf -u http://wingdata.htb/FUZZ -w /usr/share/wordlists/dirb/common.txt -mc 200,301,302,403 -t 40
```




### Explicación de cada flag

| Flag | Significado |
|---|---|
| `-u http://wingdata.htb/FUZZ` | Aquí `FUZZ` está en la ruta, justo como en Gobuster `dir`: se sustituye por cada palabra del diccionario para descubrir directorios y archivos. |
| `-mc 200,301,302,403` | *Match Codes*: whitelist de códigos de estado a mostrar. Solo se muestran respuestas 200 (OK), 301/302 (redirecciones) y 403 (Forbidden, que a menudo indica que el recurso existe pero está protegido). Es el equivalente inverso al `-b` (blacklist) que vimos en Gobuster. |
| `-t 40` | Número de hilos concurrentes. Mismo concepto que `-t` en Gobuster: a más hilos, más velocidad, con el riesgo de saturar el objetivo. |

**Nota:** a diferencia del ejemplo anterior, aquí no se usa `-fs`; el filtrado se hace directamente por código de estado con `-mc`.

---

## 3. Fuzzing sobre HTTPS con certificado inválido, sin colorear salida y wordlist con extensión de fichero

```bash
ffuf -c -k -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-lowercase-2.3-medium.txt:FUZZ -u https://10.129.186.78/FUZZ
```

### Explicación de cada flag

| Flag | Significado |
|---|---|
| `-c` | Colorea la salida en terminal (más legible cuando hay muchos resultados). |
| `-k` | Ignora errores de validación de certificado SSL/TLS (equivalente al `--no-tls-validation` de Gobuster). Necesario porque el objetivo es HTTPS con certificado autofirmado. |
| `-w ruta:FUZZ` | Sintaxis para **nombrar** la wordlist. Al escribir `:FUZZ` después de la ruta, se asocia explícitamente ese diccionario al marcador `FUZZ`. Es imprescindible cuando se usan varias wordlists a la vez (por ejemplo, una para `FUZZ1` y otra para `FUZZ2`), pero aquí se usa igualmente por claridad aunque solo haya un marcador. |
| `-u https://10.129.186.78/FUZZ` | URL objetivo con el marcador en la ruta, para descubrir directorios/archivos igual que en el ejemplo 2. |

**Objetivo:** enumeración estándar de contenido web sobre un objetivo HTTPS con certificado no válido, típico en máquinas de HTB accedidas directamente por IP.

---

## Resumen de flags vistos

| Flag | Categoría | Función |
|---|---|---|
| `-u` | Objetivo | URL objetivo, con `FUZZ` en la parte a fuzzear (ruta, parámetro, subdominio...) |
| `-w` | Wordlist | Diccionario a usar; admite `ruta:FUZZ` para asociarlo a un marcador concreto |
| `-fs` | Filtrado | Excluye respuestas por tamaño (size) |
| `-mc` | Filtrado | Whitelist de códigos de estado a mostrar |
| `-t` | Rendimiento | Número de hilos concurrentes |
| `-k` | SSL/TLS | Ignora certificados inválidos |
| `-c` | Salida | Colorea los resultados en terminal |

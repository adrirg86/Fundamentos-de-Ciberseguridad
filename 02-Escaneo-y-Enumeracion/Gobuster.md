# Gobuster — Enumeración de Directorios y Vhosts

`Gobuster` es una herramienta escrita en Go que sirve para hacer *fuerza bruta* (brute-forcing) contra distintos elementos de una aplicación web, usando un diccionario de palabras (wordlist). Es una de las herramientas más habituales en la fase de reconocimiento durante un pentest o en máquinas de plataformas tipo HTB/DockerLabs.

Los dos modos que se explican en este documento son:

- **`dir`**: descubre directorios y archivos ocultos dentro de un servidor web.
- **`vhost`**: descubre *virtual hosts* (subdominios o nombres de host configurados en el mismo servidor) que no son visibles a simple vista.

---

## 1. Enumeración de directorios con filtros avanzados

```bash
gobuster dir -u https://172.17.0.2/ \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
  -x php,txt,bak \
  -r \
  --no-tls-validation \
  -H "Host: picadilly.lab"
```

### Explicación de cada flag

| Flag | Significado |
|---|---|
| `dir` | Modo de escaneo: búsqueda de directorios/archivos. |
| `-u` | URL objetivo contra la que se lanza el ataque. |
| `-w` | Ruta al wordlist que se usará para generar las peticiones. En este caso, `directory-list-2.3-medium.txt` de DirBuster. |
| `-x php,txt,bak` | Extensiones a probar sobre cada palabra del diccionario. Gobuster probará `palabra`, `palabra.php`, `palabra.txt` y `palabra.bak`. Muy útil para encontrar backups (`.bak`) o código fuente expuesto. |
| `-r` | Sigue las redirecciones (`follow redirects`). Sin esto, una respuesta 301/302 se marca como tal pero no se sigue. |
| `--no-tls-validation` | Ignora errores de certificado SSL/TLS. Imprescindible cuando el objetivo usa HTTPS con un certificado autofirmado (muy común en máquinas de laboratorio). |
| `-H "Host: picadilly.lab"` | Añade una cabecera HTTP personalizada. Aquí se fuerza la cabecera `Host`, lo cual es necesario cuando el servidor usa *virtual hosting* y responde de forma distinta según el dominio solicitado, aunque se acceda por IP. |

**Contexto de uso:** este comando es típico cuando ya se ha descubierto (por ejemplo vía `vhost` o `/etc/hosts`) que la IP `172.17.0.2` sirve contenido distinto según el dominio, y se quiere enumerar directorios específicamente para el vhost `picadilly.lab`.

---

## 2. Enumeración de directorios básica con una extensión

```bash
gobuster dir -u http://'ip' -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php -r
```

Versión simplificada del comando anterior:

- Se sustituye la IP real por el placeholder `'ip'`, útil como plantilla reutilizable.
- Solo se prueba la extensión `.php`, algo lógico si ya se sabe (por ejemplo, por un banner o por una página de error) que el servidor corre PHP.
- Se mantiene `-r` para seguir redirecciones.
- No se usa `-H` ni `--no-tls-validation` porque el objetivo es HTTP plano y no requiere cabecera Host personalizada.

Este es el comando "base" que se suele lanzar en el primer reconocimiento de un servicio web, antes de refinarlo con extensiones y filtros adicionales.

---

## 3. Enumeración de Virtual Hosts (vhosts)

```bash
gobuster vhost -u http://silentium.htb -w /usr/share/wordlists/dirb/common.txt --append-domain
```

### Explicación de cada flag

| Flag | Significado |
|---|---|
| `vhost` | Modo de escaneo: en vez de buscar rutas, prueba distintos subdominios/vhosts contra el mismo servidor. |
| `-u` | Dominio base conocido (`silentium.htb`), normalmente ya identificado por un escaneo previo o por el archivo `/etc/hosts`. |
| `-w` | Wordlist de nombres candidatos a subdominio (aquí, `common.txt` de Dirb). |
| `--append-domain` | Añade automáticamente el dominio base a cada palabra del diccionario. Es decir, si el diccionario contiene `dev`, Gobuster probará `dev.silentium.htb` en lugar de solo `dev`. Sin este flag, en versiones recientes de Gobuster habría que incluir el dominio completo manualmente en cada palabra. |

**Objetivo:** encontrar subdominios como `admin.silentium.htb`, `dev.silentium.htb`, `api.silentium.htb`, etc., que puedan alojar contenido distinto (y potencialmente más vulnerable) que el dominio principal.

> 💡 Tras encontrar un vhost válido, normalmente se añade a `/etc/hosts` para poder resolverlo y seguir enumerando directorios sobre él (como en el Ejemplo 1).

---

## 4. Enumeración de directorios filtrando códigos de estado y con threads personalizados

```bash
gobuster dir -u http://facts.htb/ -w /usr/share/wordlists/dirb/common.txt -b 403,404 -t 30
```

### Explicación de cada flag

| Flag | Significado |
|---|---|
| `-b 403,404` | *Blacklist* de códigos de estado HTTP. Gobuster ocultará de los resultados cualquier respuesta con código 403 (Forbidden) o 404 (Not Found), reduciendo el ruido y mostrando solo lo relevante (200, 301, 302, etc.). |
| `-t 30` | Número de *threads* (hilos) concurrentes. Aumentar este valor acelera el escaneo, aunque un valor demasiado alto puede saturar el servidor objetivo o provocar falsos negativos por rate-limiting. 30 es un valor moderado-alto, razonable en un entorno de laboratorio. |

**Nota:** el uso de `-b` es una alternativa a `-s` (whitelist de códigos); se elige uno u otro según si es más cómodo excluir o incluir explícitamente.

---

## Resumen de flags vistos

| Flag | Categoría | Función |
|---|---|---|
| `-u` | Objetivo | URL o dominio base |
| `-w` | Wordlist | Diccionario a usar |
| `-x` | Filtrado | Extensiones de archivo a probar |
| `-r` | Comportamiento | Seguir redirecciones |
| `-H` | Cabeceras | Cabecera HTTP personalizada (p. ej. `Host`) |
| `--no-tls-validation` | SSL/TLS | Ignora certificados inválidos |
| `--append-domain` | Vhost | Añade el dominio base a cada palabra probada |
| `-b` | Filtrado | Códigos de estado a excluir |
| `-t` | Rendimiento | Número de hilos concurrentes |

---

- Ajustar `-t` según la estabilidad del objetivo; en máquinas de laboratorio no suele haber problema en subirlo, pero en entornos reales hay que ser prudente para no provocar denegación de servicio.
- Combinar `dir` y `vhost`: primero descubrir vhosts, añadirlos a `/etc/hosts`, y después enumerar directorios sobre cada uno.
- Usar `-x` con las extensiones propias de la tecnología detectada (PHP, ASP, JSP...) en lugar de probar todas a la vez, para no disparar el tiempo de escaneo innecesariamente.

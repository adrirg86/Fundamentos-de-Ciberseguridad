# ExifTool — Lectura y Eliminación de Metadatos

`exiftool` es una herramienta (y librería Perl) para leer, escribir y eliminar metadatos de archivos multimedia: imágenes, PDFs, audio, vídeo, documentos de Office, etc. En el contexto de ciberseguridad se usa sobre todo en dos fases:

- **OSINT / reconocimiento**: extraer información oculta en archivos públicos (modelo de cámara, coordenadas GPS, autor, software usado, fecha de creación...) que puede filtrar datos sensibles de una organización o persona.
- **Anonimización / limpieza**: eliminar esos mismos metadatos antes de compartir un archivo, para no exponer información involuntariamente.

---

## Lectura de metadatos

### 1. Ver todos los metadatos de un archivo

```bash
exiftool archivo.jpg
```

Muestra **todas** las etiquetas de metadatos disponibles en el archivo: fabricante y modelo de cámara, fecha de captura, geolocalización (si la tiene), software de edición usado, resolución, etc. Es el punto de partida habitual para hacerse una idea rápida de qué información contiene un archivo antes de decidir qué extraer o eliminar.

### 2. Ver campos específicos

```bash
exiftool -Model -ImageSize archivo.jpg
```

En lugar de volcar todo el listado, se piden únicamente las etiquetas indicadas:

| Flag | Significado |
|---|---|
| `-Model` | Modelo de la cámara o dispositivo que capturó la imagen. |
| `-ImageSize` | Dimensiones de la imagen (ancho x alto). |

Este enfoque es útil cuando ya se sabe qué dato en concreto se busca (por ejemplo, confirmar con qué modelo de móvil se hizo una foto), evitando el ruido del resto de metadatos.

### 3. Filtrar por extensión dentro de un directorio

```bash
exiftool -ext jpg directorio
```

| Flag | Significado |
|---|---|
| `-ext jpg` | Restringe el análisis a archivos con extensión `.jpg`, ignorando cualquier otro tipo de archivo presente en la carpeta. |
| `directorio` | Ruta de la carpeta a analizar (en lugar de un archivo individual). |

Se puede repetir `-ext` para incluir varias extensiones a la vez (por ejemplo, `-ext jpg -ext png`).

### 4. Procesar carpetas de forma recursiva

```bash
exiftool -r -ext jpg directorio
```

| Flag | Significado |
|---|---|
| `-r` | Modo *recursivo*: entra también en las subcarpetas de `directorio`, no solo en el nivel superior. |
| `-ext jpg` | Mismo filtro de extensión que en el ejemplo anterior. |

Combinación típica cuando se quiere auditar **todo** un árbol de carpetas (por ejemplo, una carpeta de fotos con subcarpetas por evento o por fecha) buscando un tipo de archivo concreto.

---

## Borrado de metadatos

### 5. Eliminar todas las etiquetas de forma recursiva

```bash
exiftool -all= -overwrite_original -r /ruta/carpeta
```

| Flag | Significado |
|---|---|
| `-all=` | Elimina **todas** las etiquetas de metadatos del archivo. La sintaxis `campo=` (sin valor a la derecha del `=`) es la forma que tiene ExifTool de "vaciar" un campo; usado sobre `-all`, vacía todos los campos a la vez. |
| `-overwrite_original` | Sobrescribe el archivo original directamente. Sin este flag, ExifTool crea por defecto una copia de seguridad con el sufijo `_original` antes de modificar el archivo. |
| `-r` | Recursivo, igual que en el ejemplo anterior: aplica el borrado también a las subcarpetas. |
| `/ruta/carpeta` | Carpeta sobre la que se aplica el borrado masivo. |

**Uso típico:** limpiar un lote completo de imágenes antes de subirlas a un repositorio público o compartirlas externamente, eliminando de golpe cualquier rastro de geolocalización, dispositivo o autor.

>  al usar `-overwrite_original`, la operación es destructiva — no queda copia del archivo con los metadatos originales. Conviene probar primero sin este flag (o sobre una copia de la carpeta) para verificar que el resultado es el esperado.

### 6. Borrar una etiqueta específica (autor IPTC)

```bash
exiftool -iptc:by-line= archivo.jpg
```

| Flag | Significado |
|---|---|
| `-iptc:by-line=` | Vacía específicamente el campo `By-line` del estándar **IPTC**, que normalmente contiene el nombre del autor o fotógrafo. El prefijo `iptc:` indica el grupo de metadatos al que pertenece la etiqueta (ExifTool organiza los campos en grupos como `EXIF`, `IPTC`, `XMP`, etc., y a veces el mismo nombre de campo existe en varios grupos con significados o formatos distintos). |

Este tipo de borrado selectivo es preferible al `-all=` cuando solo interesa eliminar un dato concreto (el autor, en este caso) y se quiere conservar el resto de metadatos técnicos de la imagen (resolución, formato de color, etc.).

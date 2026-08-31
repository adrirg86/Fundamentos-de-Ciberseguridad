
## ¿Qué es Nmap?

**Nmap** (*Network Mapper*) es una herramienta de auditoría de seguridad y exploración de redes. Se utiliza principalmente para descubrir **hosts activos**, **puertos abiertos** y **servicios en ejecución**. A partir de esta información, permite identificar protocolos en uso, versiones de software e incluso determinar el **Sistema Operativo** del equipo objetivo.

## ¿Cómo se usa?

Se ejecuta directamente a través de la terminal (principalmente en Linux). En función de la información que se busque recopilar, se combinan distintos **parámetros y flags** para adaptar la profundidad y velocidad del escaneo.

## Caso práctico

> **Nota:** Para los siguientes ejemplos se utilizará una máquina de laboratorio con la dirección IP `172.17.0.2`.


## Detección de Versiones y Scripts Básico

Una vez identificados los puertos abiertos en el objetivo (`172.17.0.2`), el siguiente paso en la fase de reconocimiento es determinar qué software específico y versión se están ejecutando en cada uno, así como lanzar scripts básicos de enumeración.

Para ello, utilizamos la combinación de flags `-sCV`:

* **`-sV`**: Analiza las respuestas de los servicios para identificar el nombre de la aplicación y su número de versión exacta.
* **`-sC`**: Ejecuta los scripts predeterminados del **Nmap Scripting Engine (NSE)** (equivalente a `--script=default`). Estos scripts realizan pruebas básicas no destructivas como obtener títulos HTTP, certificados SSL o banners SSH.

Nota: Se utiliza el parámetro -oN targeted.nmap para guardar la salida del análisis en formato de texto plano para consultas futuras.

### Comando Ejecutado

```bash
nmap -sCV -p 22,80,443 172.17.0.2 -oN targeted.nmap
```

<img width="960" height="349" alt="image" src="https://github.com/user-attachments/assets/10709387-ae9b-4a98-8b3a-0c916325e0a2" />

*Quedará registrado en el archivo y lo podemos abrir mediante ´cat´*

<img width="972" height="366" alt="image" src="https://github.com/user-attachments/assets/4c08abdb-3b0c-43aa-b497-314a54ea6510" />


**En este caso necesitaremos escuchar todos los puertos mediante -p- porque el estado de los puertos es :**
Estado de los puertos (closed): Los puertos 22, 80 y 443 no están abiertos. Los servicios no se están ejecutando en los puertos estándar o la aplicación se encuentra en otro puerto no analizado.
Dominio detectado: Nmap resolvió la IP a bypass403.pw, lo que sugiere añadir la entrada al archivo /etc/hosts:


### Escaneo Completo de Puertos (65,535 puertos)

Al comprobar que los puertos convencionales (22, 80, 443) se encuentran cerrados, debemos ampliar el rango de análisis a la totalidad de los puertos TCP (del 1 al 65535) para localizar los servicios activos en puertos no estándar.

Para optimizar el tiempo de ejecución de un escaneo masivo, añadimos parámetros de rendimiento:

    -p-: Indica a Nmap que escanee todo el rango de puertos (1-65535).

    --open: Filtra la salida para mostrar únicamente los puertos en estado abierto, omitiendo los cerrados o filtrados.

    -sS: Aplica la técnica de escaneo SYN (Stealth scan).

    --min-rate 5000: Establece un envío mínimo de 5000 paquetes por segundo para agilizar significativamente el proceso.

    -n: Desactiva la resolución DNS inversa durante el escaneo.

    -Pn: Omite la fase de descubrimiento de host (ping) para asumir que el equipo objetivo está activo.

## Comando Ejecutado

```bash
sudo nmap -sS -p- --open --min-rate 5000 -n -Pn 172.17.0.2 -oN all_ports.nmap
```


<img width="958" height="294" alt="image" src="https://github.com/user-attachments/assets/c6a7cc35-3f14-4284-b1c0-540cdaf3cb79" />

*Quedará registrado en el archivo y lo podemos abrir mediante ´cat´*



### Escaneo Específico de Versiones y Scripts (-sCV)

Tras identificar que los puertos abiertos no estándar son el 8080 y el 9229, ejecutamos un análisis detallado sobre ambos para determinar las versiones exactas y obtener información relevante mediante scripts del NSE.
Comando Ejecutado

```bash
nmap -sCV -p 8080,9229 172.17.0.2 -oN targeted.nmap
```

<img width="960" height="1008" alt="image" src="https://github.com/user-attachments/assets/0d81cb90-83f8-4df3-a5d2-73b28092255f" />


## Análisis Inicial de Puertos Encontrados

    Puerto 8080/tcp (http-proxy): Puerto habitualmente utilizado para servicios web secundarios o proxies.

    Puerto 9229/tcp (unknown): Puerto comúnmente asociado al protocolo de depuración de Node.js (Node.js V8 Inspector).







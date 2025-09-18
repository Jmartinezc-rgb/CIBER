# Cheatsheet de Cortafuegos en Linux: iptables y Shorewall

Esta guía resume los conceptos y comandos clave para configurar cortafuegos en Linux, con ejemplos prácticos extraídos de la práctica de configuración.

---

## Netfilter / iptables

`iptables` es la herramienta de línea de comandos para configurar el framework de filtrado de paquetes del kernel, **Netfilter**. Es potente pero complejo, y es la base sobre la que operan herramientas como Shorewall y UFW.

### Conceptos Clave

* **Tablas**: Agrupan reglas por propósito (`filter`, `nat`, `mangle`).
* **Cadenas (Chains)**: Listas de reglas aplicadas en puntos específicos del flujo de red.
    * `INPUT`: Paquetes destinados al propio firewall.
    * `OUTPUT`: Paquetes generados por el propio firewall.
    * `FORWARD`: Paquetes que atraviesan el firewall.
* **Acciones (Targets)**: La acción a tomar si un paquete coincide con una regla (`ACCEPT`, `DROP`, `REJECT`).
* **Filtro con Estado (`-m state`)**: Permite crear reglas basadas en el estado de la conexión.
    * `NEW`: Paquete que inicia una nueva conexión.
    * `ESTABLISHED`: Paquete de una conexión ya existente.
    * `RELATED`: Paquete relacionado con una conexión existente (ej. FTP-data).

### Comandos `iptables` Esenciales

* **Listar reglas**: Muestra las reglas de una tabla.
    ```bash
    # -L: Listar, -n: numérico, -v: verbose, --line-numbers: numerar
    iptables -t filter -L -n -v --line-numbers
    ```

* **Establecer Política por Defecto**: Es una buena práctica empezar denegando todo.
    ```bash
    iptables -P INPUT DROP
    iptables -P FORWARD DROP
    iptables -P OUTPUT DROP
    ```

* **Añadir una regla (Append)**: Añade una regla al final de una cadena.
    ```bash
    # Permitir tráfico ICMP (ping) en la subred local
    iptables -A INPUT -p icmp -s 192.168.64.0/24 -j ACCEPT
    iptables -A OUTPUT -p icmp -d 192.168.64.0/24 -j ACCEPT
    ```

* **Reglas con estado para permitir navegación**: Permite iniciar conexiones hacia el exterior, pero no que el exterior las inicie hacia nosotros.
    ```bash
    # Permitir que nuestro equipo inicie conexiones TCP/UDP (salientes)
    iptables -A OUTPUT -p tcp -m state --state NEW,ESTABLISHED -j ACCEPT
    iptables -A OUTPUT -p udp -m state --state NEW,ESTABLISHED -j ACCEPT

    # Permitir las respuestas a esas conexiones (entrantes)
    iptables -A INPUT -p tcp -m state --state ESTABLISHED -j ACCEPT
    iptables -A INPUT -p udp -m state --state ESTABLISHED -j ACCEPT
    ```

* **Redirección de Tráfico (DNAT)**: Redirige el tráfico destinado a una IP/puerto a otra diferente.
    ```bash
    # Redirige las peticiones SSH a la IP falsa 1.2.3.4 hacia el servidor real 192.168.64.128
    iptables -t nat -A OUTPUT -p tcp --dport 22 -d 1.2.3.4 -j DNAT --to-destination 192.168.64.128:22
    ```

* **Mitigar Ataques de Fuerza Bruta (módulo `recent`)**: Limita el número de intentos de conexión a un servicio.
    ```bash
    # 1. Registra cada nuevo intento de conexión SSH en una lista llamada 'ssh'
    iptables -A INPUT -p tcp --dport 22 -m state --state NEW -m recent --name ssh --set

    # 2. Bloquea la IP si intenta conectar más de 2 veces en 180 segundos (el 3er intento se bloquea)
    iptables -A INPUT -p tcp --dport 22 -m state --state NEW -m recent --name ssh --update --seconds 180 --hitcount 3 -j DROP
    ```

---

## Shorewall: `iptables` de Forma Sencilla

**Shorewall** simplifica la configuración de Netfilter. Defines la política en ficheros de alto nivel y Shorewall genera las reglas de `iptables` por ti.

### Proceso de Configuración

La configuración se realiza editando ficheros en `/etc/shorewall/`.

#### Paso 1: Definir las Zonas (`/etc/shorewall/zones`)

Define las áreas de tu red con distintos niveles de seguridad.

```
\#ZONE   TYPE
fw      firewall      \# El propio cortafuegos
net     ipv4          \# La red externa, Internet (ext)
loc     ipv4          \# La red local, LAN (int)
```

#### Paso 2: Asignar Interfaces a las Zonas (`/etc/shorewall/interfaces`)

Asocia las interfaces de red físicas de tu máquina a las zonas.

```

\#ZONE   INTERFACE   BROADCAST
net     eth1        detect      \# Interfaz conectada a Internet
loc     eth0        detect      \# Interfaz conectada a la LAN

```

#### Paso 3: Establecer la Política por Defecto (`/etc/shorewall/policy`)

Establece el comportamiento por defecto del tráfico entre zonas. Esta es la base de tu seguridad.

```

\#SOURCE   DEST    POLICY    LOG LEVEL

# \------------------------------------

# Permitir que la red local se comunique con el firewall y salga a Internet

loc       fw      ACCEPT
loc       net     ACCEPT

# Permitir que el firewall se comunique con todo

fw        all     ACCEPT

# Bloquear todo el tráfico no solicitado desde Internet

# Usamos DROP en lugar de REJECT para no dar información al origen

net       all     DROP      info

# Como medida de seguridad final, rechazar todo lo demás

all       all     REJECT    info

```

#### Paso 4: Habilitar Enmascaramiento (NAT) (`/etc/shorewall/masq`)

Permite que los equipos de la red interna (`loc`) salgan a Internet a través de la IP de la interfaz externa (`net`).

```

\#INTERFACE      SUBNET
\#La interfaz externa 'eth1' enmascarará el tráfico de la red interna 'eth0'
eth1            eth0

```

#### Paso 5: Crear Reglas de Excepción (`/etc/shorewall/rules`)

Define las excepciones a tu política restrictiva. Aquí abres los puertos necesarios.

```

\#ACTION         SOURCE          DEST                    PROTO   DEST PORT(S)

# \--- EJEMPLOS DE REGLAS COMUNES ---

# 1\. Habilitar servicios de navegación (HTTP/S) y correo (SMTP) desde la LAN

ACCEPT          loc             net                     tcp     80,443,25

# 2\. Permitir acceso administrativo SSH al firewall SÓLO desde una IP externa de confianza

ACCEPT          net:20.20.20.100 fw                      tcp     22

# 3\. Port Forwarding (DNAT) para exponer un servidor web interno al exterior

# Las peticiones que llegan al puerto 80 del firewall se reenvían al servidor 192.168.1.100

DNAT            net             loc:192.168.1.100       tcp     80

# 4\. Redirigir todas las peticiones DNS (puerto 53) de la red interna a un servidor DNS específico

DNAT            loc             net:20.20.20.50:53      udp     53

````

### Comandos de Gestión de Shorewall

* **Verificar la configuración**: Antes de aplicar, comprueba si hay errores.
    ```bash
    shorewall check
    ```
* **Iniciar y Habilitar el Servicio**:
    ```bash
    systemctl start shorewall
    systemctl enable shorewall
    ```
* **Ver las reglas `iptables` generadas**:
    ```bash
    iptables -L -n -v
    ```

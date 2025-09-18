# Cheatsheet: PKI con OpenSSL para HTTPS en Apache

---

## 🏛️ 1. Creación de la Infraestructura de la CA

El primer paso es construir el "edificio" donde operará nuestra Autoridad de Certificación. Esto implica crear una estructura de directorios y ficheros de control bien definida.

### Paso 1.1: Preparar la Estructura de Directorios

Estos comandos crean el esqueleto de nuestra PKI en la ubicación estándar `/etc/pki/tls`. Es crucial que la fecha y hora del sistema sean correctas antes de empezar, ya que los certificados dependen de ellas.

```bash
# Crear el directorio base de la PKI
mkdir -p /etc/pki/tls
cd /etc/pki/tls

# Crear subdirectorios necesarios
mkdir certs crl newcerts private

# Crear ficheros de control iniciales
touch index.txt      # Base de datos de certificados emitidos
echo 01 > serial     # Número de serie para el próximo certificado
echo 01 > crlnumber  # Número de serie para la próxima CRL
````

### Paso 1.2: Configurar `openssl.cnf`

Este fichero (`/etc/ssl/openssl.cnf`) es el cerebro de OpenSSL. Debes ajustarlo para definir el comportamiento de tu CA.

**Sección `[ CA_default ]`**: Asegúrate de que el directorio base apunta a tu PKI.
    
    [ CA_default ]
    dir = /etc/pki/tls  # Directorio raíz de la CA
    
**Sección `[ req_distinguished_name ]`**: Define los valores por defecto para los certificados que emitas, para no tener que escribirlos siempre.
    
    [ req_distinguished_name ]
    countryName_default                 = ES
    stateOrProvinceName_default         = Madrid
    localityName_default                = Alcorcon
    0.organizationName_default          = CA de Pruebas
    organizationalUnitName              = Organizational Unit Name (eg, section)
    
**Añadir Secciones de Extensiones**: Es crucial definir extensiones para los distintos tipos de certificados (servidor, cliente, etc.)
    
    # --- Añade esto en la sección [ req ] ---
    req_extensions = v3_req

    # --- Añade estas nuevas secciones al final del fichero ---
    [ v3_req ]
    subjectAltName = @alt_names

    # Nombres alternativos para el servidor (¡muy importante!)
    [ alt_names ]
    DNS.1 = [www.seguro.com](https://www.seguro.com)
    DNS.2 = seguro.com

    # Extensiones para un certificado de servidor con SAN
    [ server_SAN ]
    basicConstraints = CA:FALSE
    keyUsage = nonRepudiation, digitalSignature, keyEncipherment
    subjectAltName = @alt_names

    # Extensiones para un certificado de cliente
    [ usr_cert ]
    basicConstraints = CA:FALSE
    keyUsage = critical, digitalSignature, keyEncipherment
    extendedKeyUsage = clientAuth
    

### Paso 1.3: Generar la Clave Privada y el Certificado Raíz de la CA

Ahora creamos la identidad de la CA: su clave privada (el secreto mejor guardado) y su certificado público (autofirmado).

```bash
# 1. Generar la clave privada de la CA (protegida con contraseña)
openssl genpkey -algorithm RSA -out /etc/pki/tls/private/cakey.key \
-pkeyopt rsa_keygen_bits:2048 -aes256

# 2. Crear el certificado raíz autofirmado de la CA (válido por 10 años)
# Te pedirá la contraseña de la clave privada y los datos del certificado
openssl req -x509 -new -key /etc/pki/tls/private/cakey.key \
-out /etc/pki/tls/cacert.pem -days 3650 -extensions v3_ca
```

### Paso 1.4: Crear la Lista de Revocación de Certificados (CRL)

La CRL es una "lista negra" de certificados que ya no son válidos.

```bash
# Generar la primera CRL (estará vacía)
openssl ca -gencrl -keyfile /etc/pki/tls/private/cakey.key \
-cert /etc/pki/tls/cacert.pem -out /etc/pki/tls/crl/crl.pem
```

-----

## 🖥️ 2. Gestión de Certificados de Servidor Web

Con la CA lista, ya podemos emitir un certificado para nuestro servidor web `www.seguro.com`.

### Paso 2.1: Crear la Clave Privada y la Solicitud de Firma (CSR)

El servidor necesita su propio par de claves. El CSR es la solicitud formal que se envía a la CA.

```bash
# 1. Generar la clave privada del servidor (sin contraseña, para que Apache pueda arrancar solo)
openssl genpkey -algorithm RSA -out /etc/pki/tls/private/servidor.key \
-pkeyopt rsa_keygen_bits:2048

# 2. Crear la Solicitud de Firma de Certificado (CSR)
# Es importante rellenar el Common Name (CN) con el dominio principal: [www.seguro.com](https://www.seguro.com)
openssl req -new -key /etc/pki/tls/private/servidor.key \
-out /etc/pki/tls/certs/servidor.csr
```

### Paso 2.2: Firmar la Solicitud (Emitir el Certificado)

La CA usa su clave privada para firmar el CSR del servidor, creando así el certificado final.

```bash
# Usamos el comando 'ca' y especificamos las extensiones de servidor con SAN
openssl ca -in /etc/pki/tls/certs/servidor.csr \
-out /etc/pki/tls/certs/servidor.crt \
-days 365 -extensions server_SAN
```

### Paso 2.3: Configurar Apache para HTTPS

Editamos la configuración de Apache para crear un VirtualHost que use nuestros nuevos certificados.

  * **Fichero de configuración**: `/etc/apache2/sites-available/seguro.conf`
    ```apache
    <VirtualHost *:443>
        ServerName [www.seguro.com](https://www.seguro.com)
        ServerAlias seguro.com
        DocumentRoot /proyectos/seguro

        SSLEngine on
        SSLCertificateFile      /etc/pki/tls/certs/servidor.crt
        SSLCertificateKeyFile   /etc/pki/tls/private/servidor.key
        SSLCACertificateFile    /etc/pki/tls/cacert.pem
    </VirtualHost>

    # Redirigir HTTP a HTTPS
    <VirtualHost *:80>
        ServerName [www.seguro.com](https://www.seguro.com)
        RewriteEngine On
        RewriteRule ^(.*)$ https://%{HTTP_HOST}$1 [R=301,L]
    </VirtualHost>
    ```
  * **Activar módulos y sitio**:
    ```bash
    a2enmod ssl
    a2enmod rewrite
    a2ensite seguro.conf
    systemctl restart apache2
    ```

### Paso 2.4: Instalar la CA en el Navegador Cliente

Para que el navegador confíe en los certificados emitidos por nuestra CA, debemos importar `cacert.pem` en el almacén de "Entidades de certificación raíz de confianza" del sistema operativo o del navegador.

-----

## 👤 3. Gestión de Certificados de Cliente

Podemos usar certificados para autenticar usuarios en lugar de contraseñas.

### Paso 3.1: Emitir un Certificado de Cliente

El proceso es similar al del servidor, pero usando las extensiones de cliente definidas en `openssl.cnf`.

```bash
# 1. Crear clave privada para el cliente (Víctor Vera)
openssl genrsa -out /etc/pki/tls/private/victorvera.key 2048

# 2. Crear CSR para el cliente (indicando su CN y OU)
openssl req -new -key /etc/pki/tls/private/victorvera.key -out /etc/pki/tls/certs/victorvera.csr

# 3. Firmar el CSR con la CA usando las extensiones de usuario
openssl ca -in /etc/pki/tls/certs/victorvera.csr \
-out /etc/pki/tls/certs/victorvera.crt \
-days 365 -extensions usr_cert
```

### Paso 3.2: Exportar a formato PKCS\#12 (.p12) e Instalar

Para que el navegador pueda usar el certificado, debemos empaquetar su clave privada y su certificado público en un único fichero `.p12`.

```bash
# Exportar el certificado y la clave a un fichero .p12 protegido con contraseña
openssl pkcs12 -export -in certs/victorvera.crt \
-inkey private/victorvera.key -out victorvera.p12 -name "Victor Vera"
```

Luego, este fichero `victorvera.p12` se importa en el almacén de certificados personales del navegador.

### Paso 3.3: Configurar Apache para Requerir Autenticación por Certificado

Añadimos directivas al VirtualHost para solicitar y validar el certificado del cliente.

```apache
# Dentro del <VirtualHost *:443> o en un <Directory> específico
SSLEngine on
...
# Obliga al cliente a presentar un certificado firmado por nuestra CA
SSLVerifyClient require
SSLVerifyDepth  1

# Control de acceso basado en el contenido del certificado
# Solo permite el acceso a miembros del departamento de Ventas
<Directory /proyectos/seguro/ventas>
    SSLRequire %{SSL_CLIENT_S_DN_OU} eq "Ventas"
</Directory>
```

-----

## ❌ 4. Revocación de Certificados

Si un certificado se ve comprometido, debemos revocarlo para que deje de ser válido.

```bash
# 1. Revocar el certificado usando su fichero .crt
openssl ca -revoke /etc/pki/tls/certs/victorvera.crt

# 2. Regenerar la lista de revocación (CRL)
openssl ca -gencrl -out /etc/pki/tls/crl/crl.pem

# 3. Configurar Apache para que consulte la CRL
# (añadir estas líneas al VirtualHost y reiniciar Apache)
SSLCARevocationFile /etc/pki/tls/crl/crl.pem
SSLCARevocationCheck chain
```

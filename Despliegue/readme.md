# Despliegue de aplicaciones web
## Proyecto
### Script 1 - Creación de usuarios y del directorio correspondiente para el alojamiento web + Host virtual en apache

```sh

#!/bin/bash

# Pedir al usuario que ingrese el nombre del nuevo usuario
read -p "Ingresa el nombre del nuevo usuario: " nombreUser

# Verificar si el usuario ya existe
if id "${nombreUser}" &>/dev/null; then
    echo "El usuario ${nombreUser} ya existe."
    exit 1
fi

# Crear el usuario y su directorio personal
sudo useradd -m $nombreUser

# Crear el directorio de Apache para el nuevo usuario
sudo mkdir /var/www/html/$nombreUser

# Configurar los permisos para el directorio de Apache del nuevo usuario
sudo chown -R $nombreUser:$nombreUser /var/www/html/$nombreUser
sudo chmod -R 755 /var/www/html/$nombreUser

# Crear una página de prueba para el nuevo usuario
echo "Hola, $nombreUser. ¡Bienvenido a tu sitio web!" | sudo tee /var/www/html/$nombreUser/index.html > /dev/null

# Mostrar un mensaje de éxito
echo "El usuario $nombreUser y su directorio personal en Apache se han creado con éxito."

# Variables de configuración
server_name="servidorjesus.local"
document_root="/var/www/html/$nombreUser"

# Crear archivo de configuración del host virtual
cat <<EOF | sudo tee /etc/apache2/sites-available/${nombreUser}.conf >/dev/null
<VirtualHost *:80>
    ServerName $server_name
    DocumentRoot $document_root
    <Directory $document_root>
        Options Indexes FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>
    ErrorLog ${APACHE_LOG_DIR}/${server_name}-error.log
    CustomLog ${APACHE_LOG_DIR}/${server_name}-access.log combined
</VirtualHost>
EOF

# Habilitar el host virtual
sudo a2ensite ${nombreUser}.conf

# Reiniciar Apache para aplicar los cambios
sudo systemctl restart apache2

```

#### [Script](https://github.com/Braeek/ProyectoDespliegue/blob/main/Proyecto/InstalacionApache.md)

### Script 2 - Se Creacion de un subdominio en el servidor DNS con las resolución directa e inversa
#### [Script](https://github.com/Braeek/ProyectoDespliegue/blob/main/Proyecto/ActivarModulos.md)

### Script 3 - Se creará una base de datos además de un usuario con todos los permisos sobre dicha base de datos (ALL PRIVILEGES)
#### [Script](https://github.com/Braeek/ProyectoDespliegue/blob/main/Proyecto/InstalarWordpress.md)

### Script 4 - Se habilitará la ejecución de aplicaciones Python con el servidor web 
#### [Script](https://github.com/Braeek/ProyectoDespliegue/blob/main/Proyecto/pythonWsgi.md)

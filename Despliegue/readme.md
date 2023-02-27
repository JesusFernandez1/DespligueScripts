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

#### [Script](https://github.com/JesusFernandez1/DespligueScripts/blob/main/Despliegue/scripts/script1)

### Script 2 - Se Creacion de un subdominio en el servidor DNS con las resolución directa e inversa

```sh
#!/bin/bash

# Pide al usuario el nombre del subdominio
read -p "Introduce el nombre del subdominio: " subdominio

# Define la dirección IP del servidor que se asociará con el subdominio
IP="192.168.237.131"

# Extrae el nombre del subdominio sin el dominio principal
nombreSubdominio=$(echo $subdominio | cut -d"." -f1)

# Obtiene la dirección IP de la máquina local
LOCAL_IP=$(hostname -I | awk '{print $1}')

# Crea el registro de zona directa
echo "Creating forward lookup zone for $subdominio"
echo "
\$ORIGIN $subdominio.
\$TTL 86400
@ IN SOA ns1.$subdominio. admin.$subdominio. (
   2023022601 ; serial
   3600       ; refresh (1 hour)
   1800       ; retry (30 minutes)
   604800     ; expire (1 week)
   86400      ; minimum (1 day)
   )
@ IN NS ns1.$subdominio.
ns1 IN A $LOCAL_IP
$nombreSubdominio IN A $IP" > /etc/bind/zones/$subdominio.zone

# Crea el registro de zona inversa
echo "Creating reverse lookup zone for $IP"
IP_REVERSED=$(echo $IP | awk -F. '{print $3"."$2"."$1}')
echo "
\$ORIGIN $IP_REVERSED.in-addr.arpa.
\$TTL 86400
@ IN SOA ns1.$subdominio. admin.$subdominio. (
   2023022601 ; serial
   3600       ; refresh (1 hour)
   1800       ; retry (30 minutes)
   604800     ; expire (1 week)
   86400      ; minimum (1 day)
   )
@ IN NS ns1.$subdominio.
$IP_REVERSED IN PTR $subdominio." > /etc/bind/zones/$IP_REVERSED.in-addr.arpa.zone

# Agrega las nuevas zonas al archivo de configuración del servidor DNS
echo "Adding new zones to named.conf.local"
echo "
zone \"$subdominio\" {
    type master;
    file \"/etc/bind/zones/$subdominio.zone\";
};

zone \"$IP_REVERSED.in-addr.arpa\" {
    type master;
    file \"/etc/bind/zones/$IP_REVERSED.in-addr.arpa.zone\";
};" >> /etc/bind/named.conf.local

# Reinicia el servicio de DNS
echo "Restarting named service"
service bind9 restart

echo "subdominio created successfully"
```

#### [Script](https://github.com/JesusFernandez1/DespligueScripts/blob/main/Despliegue/scripts/script2)

### Script 3 - Se creará una base de datos además de un usuario con todos los permisos sobre dicha base de datos (ALL PRIVILEGES)

```sh

#!/bin/bash

# Solicita al usuario la información necesaria
echo "Introduce el nombre de la base de datos:"
read nombreDB
echo "Introduce el nombre de usuario:"
read nombreUser
echo "Introduce la contraseña del usuario:"
read -s pass

# Crea la base de datos
mysql -e "CREATE DATABASE $nombreDB;"

# Crea el usuario y le asigna la base de datos
mysql -e "CREATE USER '$nombreUser'@'localhost' IDENTIFIED BY '$pass';"
mysql -e "GRANT ALL PRIVILEGES ON $nombreDB.* TO '$nombreUser'@'localhost';"

# Actualiza los privilegios
mysql -e "FLUSH PRIVILEGES;"

# Notifica que se ha completado la tarea
echo "La base de datos $nombreDB ha sido creada y el usuario $nombreUser ha sido asignado con acceso completo."

```

#### [Script](https://github.com/JesusFernandez1/DespligueScripts/blob/main/Despliegue/scripts/script3)

### Script 4 - Se habilitará la ejecución de aplicaciones Python con el servidor web

```sh

#!/bin/bash

# Instalar los paquetes necesarios
sudo apt-get update
sudo apt-get install python3 python3-pip python3-dev libpq-dev nginx

# Crear un entorno virtual
sudo -H pip3 install virtualenv
mkdir ~/myproject
cd ~/myproject
virtualenv myprojectenv
source myprojectenv/bin/activate

# Instalar dependencias de la aplicación
pip install django gunicorn psycopg2-binary

# Configurar Django
django-admin.py startproject myproject ~/myproject
cd ~/myproject
python manage.py migrate

# Configurar Gunicorn
gunicorn_config='/etc/systemd/system/gunicorn.service'
sudo touch $gunicorn_config
echo "[Unit]" | sudo tee -a $gunicorn_config
echo "Description=gunicorn daemon" | sudo tee -a $gunicorn_config
echo "After=network.target" | sudo tee -a $gunicorn_config

echo "" | sudo tee -a $gunicorn_config

echo "[Service]" | sudo tee -a $gunicorn_config
echo "User=$USER" | sudo tee -a $gunicorn_config
echo "Group=www-data" | sudo tee -a $gunicorn_config
echo "WorkingDirectory=/home/$USER/myproject" | sudo tee -a $gunicorn_config
echo "ExecStart=/home/$USER/myproject/myprojectenv/bin/gunicorn \ " | sudo tee -a $gunicorn_config
echo "--workers 3 \ " | sudo tee -a $gunicorn_config
echo "--bind unix:/home/$USER/myproject/myproject.sock \ " | sudo tee -a $gunicorn_config
echo "myproject.wsgi:application" | sudo tee -a $gunicorn_config

echo "" | sudo tee -a $gunicorn_config

echo "[Install]" | sudo tee -a $gunicorn_config
echo "WantedBy=multi-user.target" | sudo tee -a $gunicorn_config

# Configurar Nginx
nginx_config='/etc/nginx/sites-available/myproject'
sudo touch $nginx_config
echo "server {" | sudo tee -a $nginx_config
echo "    listen 80;" | sudo tee -a $nginx_config
echo "    server_name server_domain_or_IP;" | sudo tee -a $nginx_config
echo "" | sudo tee -a $nginx_config
echo "    location / {" | sudo tee -a $nginx_config
echo "        include proxy_params;" | sudo tee -a $nginx_config
echo "        proxy_pass http://unix:/home/$USER/myproject/myproject.sock;" | sudo tee -a $nginx_config
echo "    }" | sudo tee -a $nginx_config
echo "}" | sudo tee -a $nginx_config

sudo ln -s /etc/nginx/sites-available/myproject /etc/nginx/sites-enabled

# Reiniciar los servicios
sudo systemctl daemon-reload
sudo systemctl start gunicorn
sudo systemctl enable gunicorn
sudo systemctl restart nginx

```
#### [Script](https://github.com/JesusFernandez1/DespligueScripts/blob/main/Despliegue/scripts/script4)

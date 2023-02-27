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

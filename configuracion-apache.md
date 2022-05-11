# Configuración APACHE

Ejecutamos la siguiente sentencia desde el terminal de Ubuntu:

```bash
sudo nano /etc/apache2/apache2.conf
```

Añadimos el siguiente apartado en este fichero:

```
<Directory /var/www/>
        Options Indexes FollowSymLinks
        AllowOverride All
        Require all granted
</Directory>
```

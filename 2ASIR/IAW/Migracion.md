
# Table of Contents

1.  [Migración Wordpress](#org36a3373)
    1.  [Primeros Pasos](#orga44a844)
    2.  [Wordpress a Debian 13](#org2d71bc9)



<a id="org36a3373"></a>

# Migración Wordpress


<a id="orga44a844"></a>

## Primeros Pasos

1.  Revisamos usuarios mysql
    
        sudo mariadbysql -u root
    
        SELECT User, Host, Password, is_role FROM mysql.user;

1.  Cambiamos passwords
    
        ALTER USER 'root'@'localhost' IDENTIFIED BY 'passwd';
        FLUSH PRIVILEGES;

1.  Vemos los usuarios que hay
    
        SELECT User, Host, Password, is_role FROM mysql.user;


<a id="org2d71bc9"></a>

## Wordpress a Debian 13

1.  Primero necesitamos el sql de la base de datos a migrar
    
        
        mysqldump -h HOST_DB -u USUARIO_DB -p NOMBRE_DB > dump.sql
        
        # opcional --no-tablespaces

2.  Ahora es necesario modificar el sql
    
    -   Será bueno hacer un backup del sql original
    
        VIEJO="dominio_antiguo"
        NUEVO="dominio_nuevo"
        
        sed -i "s/$VIEJO/$NUEVO/g" dump.sql
        sed -i '/^USE /d' dump.sql
        sed -i '/^CREATE DATABASE /d' dump.sql

3.  Creamos directorios para la migración
    
        mkdir -p /var/www/html/directorio

4.  Copiamos los archivos y les damos los permisos necesarios
    
        
        # Copiamos el dump de la base de datos
        scp user@host:/ruta/dump.sql /ruta/nueva/
        
        # Copiamos wordpress entero
        scp user@host:/ruta /var/www/html/directorio
        
        sudo chown -R www-data:www-data /var/www/html/directorio
        chmod -R 755 /var/www/html/directorio

5.  Creamos la base de datos y usuarios con permisos necesarios
    
        sudo mariadb -u root -p
    
        CREATE DATABASE nueva_db;
        CREATE USER 'nuevo_user'@'localhost' IDENTIFIED BY 'passwd';
        
        GRANT ALL PRIVILEGES ON nueva_db.* TO 'nuevo_user'@'localhost';
        FLUSH PRIVILEGES;
    
    -   Importamos la base de datos a la nueva es importante que el sql esté modificado con los dominios nuevos
        
            mysql -u nuevo_user -p nueva_db < dump.sql
    
    -   Comprobamos que existan las tablas, dentro de mariadb
        
            USE nueva_db;
            SHOW TABLES;

6.  Ahora hay que conectar la base de datos al wordpress
    
    -   Hay que actualizar el archivo wp-config.php
    
        define( 'DB_NAME', 'nueva_db' );
        define( 'DB_USER', 'nuevo_user' );
        define( 'DB_PASSWORD', 'passwd' );
        define( 'DB_HOST', 'localhost' );

7.  Modificamos el archivo .htaccess
    -   Podemos entrar al administrador de wordpress, en wp-admin.
        -   Entrar en Ajustes → Enlaces Permanentes
        -   Guardar cambios (2 veces)
    
    -   Manualmente:
        
            <IfModule mod_rewrite.c>
                RewriteEngine On
                RewriteBase /picante/
                RewriteRule ^index\.php$ - [L]
                RewriteCond %{REQUEST_FILENAME} !-f
                RewriteCond %{REQUEST_FILENAME} !-d
                RewriteRule . /picante/index.php [L]
            </IfModule>
            # END WordPress
        
        > *picante* es la ruta donde está instalado el wordpress sería *var/www/html/picante*

8.  Activar mod<sub>rewrite</sub>
    -   Habilitamos el módulo de reescritura de URLs de Apache
        
            sudo a2enmod rewrite
            
            sudo systemctl restart apache2

9.  Allow override All en el directory de Apache
    -   Hay que permitir que Apache lea tu fichero .htaccess
    
    -   Editamos /etc/apache2/sites-available/000-default.conf
        
        -   Añadimos el bloque <Directory> dentro de <VirtualHost \*:80>
        
            # ... otras configuraciones ...
            DocumentRoot /var/www/html
            # AÑADE ESTE BLOQUE:
                <Directory /var/www/html/mi-web>
                    Options Indexes FollowSymLinks
                    AllowOverride All
                    Require all granted
                </Directory>
            </VirtualHost>
    
    -   Ejemplo:
        
            <VirtualHost *:80>
            	# The ServerName directive sets the request scheme, hostname and port that
            	# the server uses to identify itself. This is used when creating
            	# redirection URLs. In the context of virtual hosts, the ServerName
            	# specifies what hostname must appear in the request's Host: header to
            	# match this virtual host. For the default virtual host (this file) this
            	# value is not decisive as it is used as a last resort host regardless.
            	# However, you must set it for any further virtual host explicitly.
            	#ServerName www.example.com
            
            	ServerAdmin webmaster@localhost
            	DocumentRoot /var/www/html
            
            	# Available loglevels: trace8, ..., trace1, debug, info, notice, warn,
            	# error, crit, alert, emerg.
            	# It is also possible to configure the loglevel for particular
            	# modules, e.g.
            	#LogLevel info ssl:warn
            
            	ErrorLog ${APACHE_LOG_DIR}/error.log
            	CustomLog ${APACHE_LOG_DIR}/access.log combined
            
            	# For most configuration files from conf-available/, which are
            	# enabled or disabled at a global level, it is possible to
            	# include a line for only one particular virtual host. For example the
            	# following line enables the CGI configuration for this host only
            	# after it has been globally disabled with "a2disconf".
            	#Include conf-available/serve-cgi-bin.conf
            	<Directory /var/www/html/picante>
            		Options Indexes FollowSymLinks
            		AllowOverride All
            		Require all granted
            	</Directory>
            
            </VirtualHost>

10. Reiniciar
    
        sudo systemctl restart apache2


# Practica-03-Ansible Jose Francisco León López
## En esta práctica vamos a instalar wordpress con Ansible
### Para ello hemos creado una carpeta con cuatro subcarpetas
#### En cada una de ellas guardamos archivos relevantes para poder ejecutar el main.yml al final y que nos cree el wordpress con ansible
#### En la primera carpeta tenemos el inventory
##### En el inventory guardamos información sobre la ip de la máquina del backend y del frontend y la ruta al archivo .pem de nuestro aws.
##### Debe verse así:
~~~
[frontend]
52.55.201.66

[backend]
54.211.180.92

[all:vars]
ansible_user=ubuntu
ansible_ssh_private_key_file=/home/ubuntu/Practica-03/labsuser.pem
ansible_ssh_common_args='-o StrictHostKeyChecking=accept-new'
~~~
#### En la segunda carpeta vamos a guardar los scripts que debemos ejecutar para desplegar wordpress
##### El primero de ellos es el config_https.yml para configurar el https para nuestra página, se nos debería de quedar así
~~~
---
- name: Playbook para configurar HTTPS
  hosts: frontend
  become: yes

  vars_files:
    - ../vars/variables.yml

  tasks:

    - name: Desinstalar instalaciones previas de Certbot
      apt:
        name: certbot
        state: absent

    - name: Instalar Certbot con snap
      command: snap install --classic certbot

    - name: Crear un alias para el comando certbot
      command: ln -s -f /snap/bin/certbot /usr/bin/certbot

    - name: Solicitar y configurar certificado SSL/TLS a Let's Encrypt con certbot
      command:
        certbot --apache \
        -m {{ wordpress.CB_MAIL }} \
        --agree-tos \
        --no-eff-email \
        --non-interactive \
        -d {{ wordpress.CB_DOMAIN }}
~~~
##### El segundo script que tenemos es el deploy_web.yml, que es el que usaremos para desplegar wordpress. Se nos debería de quedar así:
~~~
---
- name: Playbook para hacer el deploy de la aplicación web PrestaShop
  hosts: frontend
  become: yes

  vars_files:
    - ../vars/variables.yml

  tasks:
    - name: Borrar instalciones previas de wordpress wp-cli
      file:
        path: /tmp/wp-cli.phar
        state: absent

    - name: Descargar el código fuente de Wordpress
      get_url:
        url: https://raw.githubusercontent.com/wp-cli/builds/gh-pages/phar/wp-cli.phar
        dest: /usr/local/bin/wp
        mode: 0755 
  
    - name : Borramos instalaciones previas de wordpress
      shell: rm -rf /var/www/html/*
    
    - name : Descargamos el código fuente de worpress en /var/www/html
      command: wp core download --locale=es_ES --path=/var/www/html --allow-root         
    
    - name: Instalar Wordpress desde CLI
      command:
        wp config create \
        --dbname="{{ wordpress.WORDPRESS_DB_NAME }}" \
        --dbuser="{{ wordpress.WORDPRESS_DB_USER }}" \
        --dbpass="{{ wordpress.WORDPRESS_DB_PASSWORD }}" \
        --path=/var/www/html \
        --allow-root
      
    - name: Instalar Wordpress desde CLI
      command:
        wp core install \
        --url="{{ wordpress.CB_DOMAIN }}" \
        --title="{{ wordpress.WORDPRESS_TITLE }}" \
        --admin_user="{{ wordpress.WORDPRESS_ADMIN_USER }}" \
        --admin_password="{{ wordpress.WORDPRESS_ADMIN_PASS }}" \
        --admin_email="{{ wordpress.WORDPRESS_ADMIN_EMAIL }}" \
        --path=/var/www/html \
        --allow-root

    - name: Actualizamos los plugins de wordpress
      command: wp core update --path=/var/www/html --allow-root

    - name: Actualizamos los temas 
      command: wp theme update --all --path=/var/www/html --allow-root

    - name: Instalamos un tema 
      command: wp theme install "{{ wordpress.TEMA }}" --activate --path=/var/www/html --allow-root
    
    - name: Actualizamos los plugins de wordpress
      command: wp plugin update --all --path=/var/www/html --allow-root

    - name: Instalamos un plugin 
      command: wp plugin install "{{ wordpress.PLUGIN }}" --activate --path=/var/www/html --allow-root

    - name: Instalamos otro plugin
      command: wp plugin install "{{ wordpress.PLUGIN2 }}" --activate --path=/var/www/html --allow-root
   
    - name: Configuramos la redirección del acceso
      command: wp rewrite structure '/%postname%/' --path=/var/www/html --allow-root

    - name: Configuramos el plugin del acceso por la ruta
      command: wp option update whl_page {{ wordpress.PLUGIN3 }} --path=/var/www/html --allow-root
   
    - name: Copiamos el archivo .htaccess
      template:
        src: ../templates/.htaccess
        dest: /var/www/html
        mode: 0755
    - name: Modificamos los permisos del directorio /var/www/html
      file:
        path: /var/www/html
        owner: www-data
        group: www-data
        recurse: yes
        mode: 0755
~~~

##### El tercer script que tenemos es para instalar la pila LAMP del backend y se nos debería quedar así:
~~~
---
- name: Playbook para instalar la pila LAMP en el Backend
  hosts: backend
  become: yes

  vars_files:
    - ../vars/variables.yml

  tasks:

    - name: Actualizar los repositorios
      apt:
        update_cache: yes

    - name: Instalar el sistema gestor de bases de datos MySQL
      apt:
        name: mysql-server
        state: present

    - name: Instalamos el gestor de paquetes de Python pip3
      apt: 
        name: python3-pip
        state: present

    - name: Instalamos el módulo de pymysql
      pip:
        name: pymysql
        state: present

    - name: Crear una base de datos
      mysql_db:
        name: "{{ wordpress.WORDPRESS_DB_NAME }}"
        state: present
        login_unix_socket: /var/run/mysqld/mysqld.sock 

    - name: Crear el usuario de la base de datos
      mysql_user:         
        name: "{{ wordpress.WORDPRESS_DB_USER }}"
        password: "{{ wordpress.WORDPRESS_DB_PASSWORD }}"
        priv: "{{ wordpress.WORDPRESS_DB_NAME }}.*:ALL"
        host: "%"
        state: present
        login_unix_socket: /var/run/mysqld/mysqld.sock 

    - name: Configuramos MySQL para permitir conexiones desde cualquier interfaz
      replace:
        path: /etc/mysql/mysql.conf.d/mysqld.cnf
        regexp: 127.0.0.1
        replace: "{{ wordpress.IP_CLIENTE_PRIVADA }}"

    - name: Reiniciamos el servicio de base de datos
      service:
        name: mysql
        state: restarted
~~~
##### En el cuarto script vamos a tener el que instala la pila lamp del frontend, se nos debe ver así:
~~~
---
- name: Playbook para instalar la pila LAMP en el FrontEnd
  hosts: frontend
  become: yes

  tasks:

    - name: Actualizar los repositorios
      apt:
        update_cache: yes

    - name: Instalar el servidor web Apache
      apt:
        name: apache2
        state: present

    - name: Instalar PHP y los módulos necesarios
      apt: 
        name:
          - php
          - php-mysql
          - libapache2-mod-php
          
        state: present

    - name: Modificamos el valor max_input_vars de PHP
      replace: 
        path: /etc/php/8.1/apache2/php.ini
        regexp: ;max_input_vars = 1000
        replace: max_input_vars = 5000

    - name: Modificamos el valor de memory_limit de PHP
      replace: 
        path: /etc/php/8.1/apache2/php.ini
        regexp: memory_limit = 128M
        replace: memory_limit = 256M

    - name: Modificamos el valor de post_max_size de PHP
      replace: 
        path: /etc/php/8.1/apache2/php.ini
        regexp: post_max_size = 8M
        replace: post_max_size = 128M

    - name: Modificamos el valor de upload_max_filesize de PHP
      replace:
        path: /etc/php/8.1/apache2/php.ini
        regexp: upload_max_filesize = 2M
        replace: upload_max_filesize = 128M

    - name: Copiar el archivo de configuración de Apache
      template:
        src: ../templates/000-default.conf
        dest: /etc/apache2/sites-available/
        mode: 0755

    - name: Habilitar el módulo rewrite de Apache
      apache2_module:
        name: rewrite
        state: present

    - name: Reiniciar el servidor web Apache
      service:
        name: apache2
        state: restarted
~~~
##### Y en el último script tenemos el .yml para instalar herramientas adicionales para el servidor, se nos debe ver así:
~~~
---
- name: Playbook para instalar herramientas adicionales
  hosts: aws
  become: yes

  vars_files:
    - ../vars/variables.yml

  tasks:

    - name: Descargar phpMyAdmin
      get_url:
        url: https://files.phpmyadmin.net/phpMyAdmin/5.2.0/phpMyAdmin-5.2.0-all-languages.zip
        dest: /tmp/phpMyAdmin-5.2.0-all-languages.zip

    - name: Instalar unzip
      apt:
        name: unzip
        state: present

    - name: Descomprimir phpMyAdmin
      unarchive:
        src: /tmp/phpMyAdmin-5.2.0-all-languages.zip
        dest: /tmp/
        remote_src: yes

    - name: Copiar phpMyAdmin
      copy:
        src: /tmp/phpMyAdmin-5.2.0-all-languages/
        dest: /var/www/html/phpmyadmin
        remote_src: yes
        mode: 0755

    - name: Cambiar el propietario y el grupo de phpMyAdmin
      file:
        path: /var/www/html/phpmyadmin
        state: directory
        owner: www-data
        group: www-data
        recurse: yes

    # Para utilizar el módulo mysql_db necesitamos tener instalado el paquete python3-mysqldb.
    - name: Instalar pip3
      apt:
        name: python3-pip
        state: present

    - name: Instalar PyMySQL
      pip:
        name: pymysql
        state: present

    - name: Crear la base de datos para phpMyAdmin
      mysql_db:
        name: "{{ phpmyadmin.db_name }}"
        state: present
        login_unix_socket: /var/run/mysqld/mysqld.sock

    - name: Importar la base de datos de phpMyAdmin
      mysql_db:
        name: "{{ phpmyadmin.db_name }}"
        state: import
        target: /var/www/html/phpmyadmin/sql/create_tables.sql
        login_unix_socket: /var/run/mysqld/mysqld.sock

    - name: Crear el usuario de phpMyAdmin
      mysql_user:
        name: "{{ phpmyadmin.user }}"
        password: "{{ phpmyadmin.password }}"
        priv: "{{ phpmyadmin.db_name }}.*:ALL"
        state: present
        login_unix_socket: /var/run/mysqld/mysqld.sock
~~~

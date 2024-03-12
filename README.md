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

#### El tercer script que tenemos es para instalar la pila LAMP del backend y se nos debería quedar así:
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

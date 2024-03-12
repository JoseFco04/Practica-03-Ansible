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
##### El primero de ellos es el config_https.yml para configurar el https para nuestra página
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



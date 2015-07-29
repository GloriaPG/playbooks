# Ansible : Docker,PHP,Apache & Grails.

Ansible es una poderosa herramienta de automatización para deploys, configuraciones de servidores y algunas otras tareas similares, está hecho en python, es muy intuitivo, puede usarse con un servidor o más, su desempeño se aprecia mejor cuando debemos trabajar con una cantidad considerable de servidores y aplicaciones de manera remota. Docker por su parte es una herramienta que facilita el empaquetado de las aplicaciones, con la ventaja de brindar una capa de abstracción y automatización a nivel sistema operativo en Linux, no está hecho con el objetivo de parecer una máquina virtual, es más bien un contenedor, la abstracción de un sólo proceso; perfecto para la segregación de aplicaciones.

Los objetivos que cubriremos en está practica son los siguientes:

1. Instalar Ansible.
2. Crear nuestro primer playbook.
3. Configurar nuestro archivo de hosts.
4. Crear roles.
5. Crear ficheros de tareas.
6. Uso de modulos Ansible.
7. Instalar Docker, hacer pull y run Docker.
8. Instala PHP y apache. Copiar archivos al servidor remoto.

# Instalar Ansible
Para instalar Ansible sólo tenemos que ejecutar los siguientes comando en terminal.
En caso de tener un comṕlicación por favor revisar los requerimientos para instalar ansible (aquí.)[http://docs.ansible.com/ansible/intro_installation.html#control-machine-requirements]


$ sudo apt-get install software-properties-common
$ sudo apt-add-repository ppa:ansible/ansible
$ sudo apt-get update
$ sudo apt-get install ansible

# Crear maquina Virtual con Vagrant
Es un paso muy simple.

1. Crea un directorio donde quieres este tu máquina virtual.
2. Ejecuta los siguientes comandos en ese directorio:

 $ vagrant initubuntu/trusty64
 $ vagrant up


###### Nota : Vagrant 1.4 suele tener problemas con levantar de está manera una maquina por lo que recomiendo usar la versión 1.7 de Vagrant.

Una vez que ha levantado tu máquina virtual ejecuta el siguiente comando **vagrant ssh-config**, saldrá algo como esto :
```
Host default
  HostName 127.0.0.1
  User vagrant
  Port 2222
  UserKnownHostsFile /dev/null
  StrictHostKeyChecking no
  PasswordAuthentication no
  IdentityFile /home/gloria/.vagrant.d/insecure_private_key
  IdentitiesOnly yes
  LogLevel FATAL

```
Lo que nos interesa de está información es el User,  IdentityFile y el puerto porque lo usaremos para trabajar con Ansible.

# Aprovisionamiento con Ansible
Ansible se encuentra en /etc/ansible, por default crea dos archivos; hosts y ansible.cfg
Para entender sobre que significan estos archivos es necesario tener en cuenta los siguientes conceptos:

**Inventory** : Es el archivo donde se definen los hosts o grupo de de hosts y las configuraciones necesarias para poder conectarse con dichos hosts.

**Playbook**: Es  el archivo donde se deben definir las tareas a ejecutar un hosts o un grupo de hosts, es muy útil para la reutilización de configuraciones.

**Roles**: Es un conjunto de archivos pertenecientes a un host o un grupo de hosts, por ejemplo, podemos tener un conjunto de archivos dedicado al aprovisionamiento de gestores de base de datos, otro grupo de archivos para la generación de llaves ssh en los servidores.

Entonces los archivos por default que tenemos son un ejemplo de cómo podemos configurar los hosts y un .cfg que es la configuración de nuestro inventory.

Crearemos también los siguientes directorios:

**files** : Es aquí donde deben estar los archivos que deseamos copiar o manipular en nuestras tareas.

**tasks**: Aquí tendremos un archivo con el nombre de main.yml el cual tendrá un listado de tareas a ejecutar

Nuestro directorio dentro de /etc/ansible lucirá así:
```
.
├── ansible.cfg
├── hosts
├── playbook.yml
└── roles
    ├── client-one
    │   ├── files
    │   └── tasks
    │       └── main.yml
    └── client-two
        ├── files
        │   └── index.php
        └── tasks
            └── main.yml

```
Es una manera elegante de organizar nuestros archivos, facil e intuitiva.

El role client-one, preparará nuestro servidor ( nuestra máquina virtual con vagrant) para trabajar con grails 2.2.4, y todo este ambiente lo manejaremos con contenedores de Docker, es decir, instalaremos Docker, haremos pull de las imagenes y las correremos.

El role client-two provisionará nuestro servidor para poder tener una aplicación php y haremos uso de nuestro directorio files.

# Configurar playbook y archivo hosts
Para configurar el host pondremos lo siguiente en el archivo hosts:
```
[client-one]
127.0.0.1 ansible_ssh_user=vagrant ansible_ssh_private_key_file=/home/gloria/.vagrant.d/insecure_private_key  ansible_ssh_port=2222
```
Estamos indicando la ip de nuestro servidor remoto (maquina virtual), el usuario ssh, la ubicación de la llave privada y el puerto ssh, estos datos los tomamos de el resultado de ejecuta **vagrant ssh-config**.

En nuestro playbook, agregaremos lo siguiente :
```
# playbook.yml
---

#Install docker and run image for enviroment grails
- hosts: client-one
  roles:
    - client-one
    - client-two

```

Lo que estamos diciendo aquí, es: A nuestro host client-one (como lo definimos en el archivo hosts) le pertenecen los roles client-one y client-two .

En caso que en el archivo de hosts en lugar de darle el nombre de client-one a nuestro host le hubiéramos puesto server-php-grails, entonces en -hosts: podríamos server-php-grails.

# Configuración de roles
Para el role client-one crearemos un archivo llamado main.yml dentro del directorio tasks, este contendrá el listado de tareas que queremos se ejecuten, debe ir de está manera:
```
# roles/clien-one/tasks/main.yml
---
- name: 1. Upgrade
  apt: update_cache=yes
- name: 2. Install wget
  apt: name=wget state=present
- name: 3. Install Docker
  shell: wget -qO- https://get.docker.com/ | sh
- name: 4. Docker pull image redis
  shell: docker pull redis
- name: 5. Docker pull image grails
  shell: docker pull dockermd/gvm:2.2.4
- name: 6. Docker pull image mysql
  shell: docker pull mysql
- name: 7. Docker run redis
  shell: docker run --name some-redis -d redis
- name: 8. Docker run grails
  shell: docker run --name some-grails  dockermd/gvm:2.2.4
- name: 9. Docker run mysql
  shell: docker run --name some-mysql -e MYSQL_ROOT_PASSWORD=root -d mysql
```
Para el cliente-two usaremos el directorio de tasks y files, en files, podremos un simple index.php el cual copiaremos a nuestro servidor remoto y dentro de tasks crearemos un archivo main.yml, quedara así:
```
# roles/client-two/tasks/main.yml
---
- name: 1. Install Apache
  apt: name=apache2 state=present

- name: 2. Install PHP
  apt: name=libapache2-mod-php5 state=present

- name: 3. Start Apache
  service: name=apache2 state=running enabled=yes

- name: 4. Copy index.php
  copy: src=index.php dest=/var/www/index.php mode=0664
```

# Run y test

Para probar que tenemos conexión con el host que deseamos, usaremos este comando a nivel del archivo playbook.yml:
  
  $ ansible client-one  -i hosts -m ping
  
Sí nos conectamos correctamente debe arrojar el siguiente resultado en la terminal:
```
127.0.0.1 | success >> {
    "changed": false, 
    "ping": "pong"
}

```
Para correr el playbook ejecute el comando ansible-playbook.

$ ansible-playbook -i hosts playbook.yml --sudo --verbose

Y este es el comando que nos permitirá ejecutar las tareas dentro del servidor remoto, por medio de un playbook con roles asignados a un host.

Espero sea de ayuda, i'm beginner Ansible :D





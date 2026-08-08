# Taller Linux

Automatización del despliegue de una aplicación web PHP utilizando Ansible.

## Descripción

El trabajo consiste en automatizar la instalación y configuración de una aplicación web desarrollada en PHP.

La solución utiliza dos servidores principales:

* **CentOS01** — servidor de aplicación.
* **Ubuntu01** — servidor de base de datos.

El acceso a los servidores y la ejecución de Ansible se realizan desde el Bastion.

La aplicación se encuentra en CentOS01 y se conecta de forma remota a MariaDB instalada en Ubuntu01.

La aplicación utilizada para el trabajo es la proporcionada en el repositorio indicado en el obligatorio:

[Repositorio de la aplicación](https://github.com/emverdes/dbappphp?utm_source=chatgpt.com)

## Servidores

| Servidor | IP         | Función                           |
| -------- | ---------- | --------------------------------- |
| CentOS01 | 10.0.2.15  | Apache, PHP, PHP-FPM y aplicación |
| CentOS02 | 10.0.2.9   | Servidor CentOS                   |
| Ubuntu01 | 10.0.2.100 | MariaDB                           |
| Ubuntu02 | 10.0.2.101 | Servidor Ubuntu                   |

## Inventario

El inventario se encuentra en:

```text
inventory/hosts.ini
```

```ini
[centos]
centos01 ansible_host=10.0.2.15
centos02 ansible_host=10.0.2.9

[ubuntu]
ubuntu01 ansible_host=10.0.2.100
ubuntu02 ansible_host=10.0.2.101

[linux:children]
centos
ubuntu
```

Las variables comunes se encuentran en:

```text
inventory/group_vars/linux.yaml
```

```yaml
---
ansible_user: sysadmin
```

## Colecciones

Las colecciones utilizadas están definidas en:

```text
collections/requirements.yaml
```

```yaml
---

collections:
  - community.general
  - ansible.posix
  - community.mysql
```

Para instalarlas:

```bash
ansible-galaxy collection install -r collections/requirements.yaml
```

## Hardening de Ubuntu

El playbook de hardening se encuentra en:

```text
playbooks/hardening.yaml
```

Se realizan tareas como:

* Actualización de paquetes.
* Instalación y configuración de UFW.
* Política de entrada `deny`.
* Política de salida `allow`.
* Permitir SSH.
* Instalación de Fail2ban.
* Configuración y activación de Fail2ban.

El archivo de configuración de Fail2ban se encuentra en:

```text
files/jail.local
```

```ini
[DEFAULT]
bantime = 10m
findtime = 10m
maxretry = 5

[sshd]
enabled = true
port = ssh
backend = systemd
```

Fail2ban se administra mediante `systemd_service` y se utiliza un handler para reiniciarlo cuando cambia su configuración.

## Servidor CentOS

El playbook se encuentra en:

```text
playbooks/centos.yaml
```

Instala:

* Apache (`httpd`)
* PHP
* PHP-FPM
* `php-mysqlnd`
* `python3-libsemanage`

Apache y PHP-FPM quedan iniciados y habilitados mediante `systemd`.

También se configura `firewalld` para permitir HTTP.

```yaml
ansible.posix.firewalld:
  service: http
  state: enabled
  permanent: true
  immediate: true
```

## Aplicación PHP

La aplicación se despliega utilizando el template:

```text
templates/cumple.j2
```

El archivo se instala como:

```text
/var/www/html/index.php
```

La conexión a MariaDB utiliza variables:

```php
$conexion = new mysqli(
    "{{ DB_SERVER }}",
    "{{ DB_USER }}",
    "{{ DB_PASS }}",
    "{{ DB_DBASE }}"
);
```

La aplicación consulta la tabla de cumpleaños y muestra los nombres y fechas almacenados.

También se habilitaron los booleanos de SELinux necesarios para permitir que Apache pueda realizar conexiones de red hacia MariaDB:

```text
httpd_can_network_connect
httpd_can_network_connect_db
```

## MariaDB

El servidor de base de datos es Ubuntu01.

El playbook se encuentra en:

```text
playbooks/ubuntu.yaml
```

Se instala MariaDB junto con la dependencia necesaria para que Ansible pueda administrarla:

```text
mariadb-server
python3-pymysql
```

MariaDB queda iniciado y habilitado mediante `systemd`.

También se configura:

```text
bind-address = 0.0.0.0
```

para permitir conexiones remotas.

## Base de datos

La aplicación utiliza la base:

```text
cumples
```

La estructura y los datos iniciales se basan en el archivo `cumpleanios.sql` incluido en el repositorio de la aplicación proporcionado para el obligatorio.

Los datos iniciales son:

| Nombre         | Fecha      |
| -------------- | ---------- |
| Frodo Baggins  | 2005-01-14 |
| Aragorn        | 2004-02-09 |
| Arwen Undomiel | 1994-12-09 |

La tabla utilizada por la aplicación es `cumpleaños`, de acuerdo con la consulta realizada por el archivo PHP proporcionado.

La creación de la base, tabla y datos se realiza mediante módulos de Ansible de la colección `community.mysql`.

## Usuario de MariaDB

Se crea un usuario específico para la aplicación:

```text
intranet
```

El usuario está autorizado desde el servidor CentOS01:

```text
10.0.2.15
```

y tiene permisos sobre:

```text
cumples.*
```

Los datos de conexión se encuentran en:

```text
vars/database.yaml
```

Este archivo está protegido mediante Ansible Vault.

## Firewall de Ubuntu

UFW permite conexiones a MariaDB por el puerto `3306` únicamente desde CentOS01:

```text
10.0.2.15
```

La configuración se realiza mediante:

```yaml
community.general.ufw:
  port: 3306
  proto: tcp
  rule: allow
  from_ip: 10.0.2.15
```

La comunicación entre los servidores queda:

```text
CentOS01
10.0.2.15
    |
    | TCP 3306
    v
Ubuntu01
10.0.2.100
    |
    v
MariaDB
```

## Ansible Vault

Las variables de conexión se encuentran en:

```text
vars/database.yaml
```

El archivo está cifrado con Ansible Vault.

Para ejecutar los playbooks se utiliza:

```bash
--ask-vault-pass
```

El archivo cifrado puede estar en el repositorio, pero la contraseña utilizada para abrirlo no debe almacenarse en Git.

## Comprobación

La sintaxis de los playbooks se comprueba con:

```bash
ansible-playbook -i inventory/hosts.ini playbooks/hardening.yaml --syntax-check
```

```bash
ansible-playbook -i inventory/hosts.ini playbooks/ubuntu.yaml --syntax-check
```

```bash
ansible-playbook -i inventory/hosts.ini playbooks/centos.yaml --syntax-check
```

Para ejecutar Ubuntu:

```bash
ansible-playbook -i inventory/hosts.ini playbooks/ubuntu.yaml \
  --ask-vault-pass \
  --ask-become-pass
```

Para ejecutar CentOS:

```bash
ansible-playbook -i inventory/hosts.ini playbooks/centos.yaml \
  --ask-vault-pass \
  --ask-become-pass
```

## Prueba de la aplicación

Desde CentOS01:

```bash
curl http://localhost/
```

La aplicación devuelve la lista de cumpleaños obtenida desde MariaDB.

También se comprobó desde un navegador utilizando:

```text
http://10.0.2.15/
```

La aplicación se mostró correctamente y los datos almacenados en MariaDB fueron visibles.

## Estructura del proyecto

```text
taller-linux/
├── collections/
│   └── requirements.yaml
├── files/
│   └── jail.local
├── inventory/
│   ├── group_vars/
│   │   └── linux.yaml
│   └── hosts.ini
├── playbooks/
│   ├── centos.yaml
│   ├── hardening.yaml
│   └── ubuntu.yaml
├── templates/
│   └── cumple.j2
├── vars/
│   └── database.yaml
├── .gitignore
├── LICENSE
└── README.md
```

## Git

El proyecto se mantiene en GitHub y se fueron realizando commits durante el desarrollo.

Se utilizó una rama `fail2ban` para desarrollar la configuración de Fail2ban y posteriormente se integró a `main`.

Los archivos temporales de editores y las contraseñas de Vault se excluyen mediante `.gitignore`.

## Pendiente

Queda por realizar:

* Crear el playbook principal para ejecutar toda la solución.
* Ejecutar nuevamente el despliegue completo.
* Comprobar la idempotencia mediante una segunda ejecución.
* Agregar las evidencias finales.
* Realizar la modificación individual solicitada para la evaluación.

## Estado actual

La aplicación funciona correctamente.

CentOS01 ejecuta Apache y PHP, y la aplicación realiza una conexión remota hacia MariaDB en Ubuntu01. El acceso a MariaDB está limitado mediante UFW para aceptar conexiones desde CentOS01.

Los datos iniciales de la aplicación corresponden a los proporcionados en el `cumpleanios.sql` del repositorio utilizado para el obligatorio.

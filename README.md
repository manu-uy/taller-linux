# TALLER LINUX

```text
+------------------------------------------------------------------+
|  TALLER LINUX :: AUTOMATIZACION CON ANSIBLE                    |
|  Ignacio Ortega - Manuel Vazquez                                |
+------------------------------------------------------------------+
```

Automatización del despliegue de una aplicación web PHP utilizando Ansible.

La solución está compuesta por dos servidores:

* **CentOS:** Apache, PHP, PHP-FPM y aplicación web.
* **Ubuntu:** MariaDB y base de datos.

La aplicación realiza una conexión remota hacia MariaDB.

La aplicación utilizada es la proporcionada para el obligatorio:

https://github.com/emverdes/dbappphp

+------------------------------------------------------------------+
##|  ARQUITECTURA                                                  |
+------------------------------------------------------------------+

```text
                 HTTP
                  |
                  v
          +---------------+
          |    CentOS     |
          | Apache + PHP  |
          |  Aplicacion   |
          +-------+-------+
                  |
                  | TCP 3306
                  v
          +---------------+
          |    Ubuntu     |
          |    MariaDB    |
          +---------------+
```

En nuestro entorno utilizamos:

| Servidor | IP         | Función        |
| -------- | ---------- | -------------- |
| CentOS01 | 10.0.2.15  | Aplicación web |
| Ubuntu01 | 10.0.2.100 | MariaDB        |

+------------------------------------------------------------------+
##|  INVENTARIO                                                    |
+------------------------------------------------------------------+

El inventario se encuentra en:

```text
inventory/hosts.ini
```

Para utilizar el proyecto en otro entorno solamente es necesario cambiar las direcciones `ansible_host` de este archivo.

Ejemplo:

```ini
[centos]
centos01 ansible_host=IP_DEL_CENTOS

[ubuntu]
ubuntu01 ansible_host=IP_DEL_UBUNTU

[linux:children]
centos
ubuntu
```

Las demás configuraciones utilizan variables de Ansible y no requieren modificar los playbooks por cambio de IP.

+------------------------------------------------------------------+
##|  PLAYBOOKS                                                      |
+------------------------------------------------------------------+

El despliegue completo se realiza desde:

```text
playbooks/site.yaml
```

Este playbook ejecuta:

```text
hardening.yaml
ubuntu.yaml
centos.yaml
```

La configuración de MariaDB está separada en el rol:

```text
roles/install_mariadb/
```

De esta forma las tareas de instalación y configuración de MariaDB quedan separadas del playbook principal.

+------------------------------------------------------------------+
##|  QUE AUTOMATIZAMOS                                             |
+------------------------------------------------------------------+

### CentOS

* Instalación de Apache, PHP y PHP-FPM.
* Instalación de `php-mysqlnd`.
* Despliegue de la aplicación mediante template.
* Configuración de Apache y PHP-FPM con `systemd`.
* Configuración de `firewalld` para HTTP.
* Configuración de SELinux para permitir la conexión hacia MariaDB.

La aplicación se genera desde:

```text
templates/cumple.j2
```

y se instala como:

```text
/var/www/html/index.php
```

### Ubuntu

* Instalación de MariaDB.
* Instalación de `python3-pymysql`.
* Configuración de conexiones remotas.
* Administración mediante `systemd`.
* Creación de la base y tabla.
* Carga de los datos iniciales.
* Creación del usuario de la aplicación.
* Configuración de UFW para permitir MariaDB solamente desde CentOS.

+------------------------------------------------------------------+
##|  BASE DE DATOS                                                 |
+------------------------------------------------------------------+

La aplicación utiliza:

```text
Base:  cumples
Tabla: cumpleanios
Usuario: intranet
```

Los datos iniciales corresponden a los proporcionados por la aplicación del obligatorio.

La creación de los registros se realiza de forma que una segunda ejecución no los duplique.

Las variables de conexión se encuentran protegidas mediante Ansible Vault:

```text
vars/database.yaml
```

No se almacenan contraseñas reales en texto plano en Git.

+------------------------------------------------------------------+
##|  COLECCIONES                                                   |
+------------------------------------------------------------------+

Las colecciones requeridas están definidas en:

```text
collections/requirements.yaml
```

Para instalarlas:

```bash
ansible-galaxy collection install -r collections/requirements.yaml
```

+------------------------------------------------------------------+
##|  EJECUCION                                                     |
+------------------------------------------------------------------+

Una vez configurado el inventario:

```bash
ansible-playbook -i inventory/hosts.ini playbooks/site.yaml \
  --ask-vault-pass \
  --ask-become-pass
```

También se puede comprobar la sintaxis con:

```bash
ansible-playbook -i inventory/hosts.ini playbooks/site.yaml \
  --syntax-check \
  --ask-vault-pass
```

+------------------------------------------------------------------+
##|  IDEMPOTENCIA                                                   |
+------------------------------------------------------------------+

Probamos una segunda ejecución del playbook completo.

La segunda ejecución finalizó correctamente sin realizar cambios innecesarios:

```text
changed=0
failed=0
```

También comprobamos que los registros de la tabla no se duplicaran.

+------------------------------------------------------------------+
##|  COMPROBACION                                                   |
+------------------------------------------------------------------+

La aplicación puede comprobarse desde CentOS mediante:

```bash
curl http://localhost/
```

También puede accederse desde un navegador utilizando la IP del servidor CentOS.

La aplicación muestra los datos almacenados en MariaDB, confirmando que la conexión entre ambos servidores funciona correctamente.

+------------------------------------------------------------------+
##|  ESTRUCTURA                                                     |
+------------------------------------------------------------------+

```text
taller-linux/
├── collections/
│   └── requirements.yaml
├── files/
│   └── jail.local
├── inventory/
│   ├── group_vars/
│   └── hosts.ini
├── playbooks/
│   ├── site.yaml
│   ├── hardening.yaml
│   ├── ubuntu.yaml
│   └── centos.yaml
├── roles/
│   └── install_mariadb/
├── templates/
│   └── cumple.j2
├── vars/
│   └── database.yaml
├── .gitignore
├── LICENSE
└── README.md
```

+------------------------------------------------------------------+
##|  RESULTADO                                                     |
+------------------------------------------------------------------+

Al finalizar el despliegue:

* Apache y PHP quedan funcionando en CentOS.
* MariaDB queda funcionando en Ubuntu.
* La aplicación se conecta remotamente a MariaDB.
* Los datos de la base se muestran en la aplicación.
* Los firewalls permiten únicamente las comunicaciones necesarias.
* Las credenciales se mantienen protegidas mediante Ansible Vault.
* El despliegue es idempotente.
* El proyecto completo puede ejecutarse mediante `site.yaml`.

El proyecto fue realizado por:

```text
+------------------------------------------------------------------+
|                         ASLXLAB                                  |
|                                                                  |
|                   Ignacio Ortega                                |
|                   Manuel Vazquez                                |
+------------------------------------------------------------------+
```


# Vagrant + Ansible — Entorno de laboratorio Zabbix

Contenido: Vagrant + Ansible que provisiona una VM Ubuntu 20.04 y despliega Zabbix (docker-compose) para pruebas y desarrollo.

---

## Estructura de archivos (raíz)

```
Vagrantfile
ansible/
  playbook.yml
  inventory.ini
  group_vars/all.yml
  roles/
    docker/
      tasks/main.yml
    zabbix/
      files/docker-compose.yml
      tasks/main.yml
README.md
```

---

## `Vagrantfile`

```ruby
# Vagrantfile - Levanta 1 VM Ubuntu 20.04 y ejecuta ansible local
Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/focal64"
  config.vm.hostname = "zabbix-lab"

  # Red
  config.vm.network "forwarded_port", guest: 80, host: 8080
  config.vm.network "forwarded_port", guest: 10051, host: 10051
  config.vm.network "forwarded_port", guest: 443, host: 8443

  # Recursos
  config.vm.provider "virtualbox" do |vb|
    vb.memory = 4096
    vb.cpus = 2
n  end

  # Sincronizar la carpeta ansible dentro de la VM
  config.vm.synced_folder "./ansible", "/vagrant_ansible"

  # Provision con Ansible (ansible_local)
  config.vm.provision "ansible_local" do |ansible|
    ansible.playbook = "/vagrant_ansible/playbook.yml"
    ansible.inventory_path = "/vagrant_ansible/inventory.ini"
    ansible.install = true
    ansible.verbose = true
  end
end
```

---

## `ansible/inventory.ini`

```ini
[zabbix]
127.0.0.1 ansible_connection=local
```

---

## `ansible/group_vars/all.yml`

```yaml
---
# Variables globales
zabbix_version: "6.4"
mysql_root_password: "changeme123"
mysql_database: "zabbix"
mysql_user: "zabbix"
mysql_password: "zabbix_pass"
```

---

## `ansible/playbook.yml`

```yaml
---
- name: Provision Zabbix lab
  hosts: zabbix
  become: true
  vars_files:
    - group_vars/all.yml
  roles:
    - docker
    - zabbix
```

---

## `ansible/roles/docker/tasks/main.yml`

```yaml
---
- name: Actualizar apt cache
  apt:
    update_cache: yes
    cache_valid_time: 3600

- name: Instalar paquetes requeridos
  apt:
    name:
      - apt-transport-https
      - ca-certificates
      - curl
      - gnupg
      - lsb-release
      - python3-pip
    state: present

- name: Agregar la llave oficial de Docker
  apt_key:
    url: https://download.docker.com/linux/ubuntu/gpg
    state: present

- name: Agregar repo de Docker
  apt_repository:
    repo: deb [arch=amd64] https://download.docker.com/linux/ubuntu focal stable
    state: present

- name: Instalar docker engine
  apt:
    name:
      - docker-ce
      - docker-ce-cli
      - containerd.io
    state: latest

- name: Instalar docker-compose via pip (simple en lab)
  pip:
    name: docker-compose
    state: present

- name: Añadir usuario vagrant al grupo docker
  user:
    name: vagrant
    groups: docker
    append: yes
```

---

## `ansible/roles/zabbix/files/docker-compose.yml`

```yaml
version: '3.5'
services:
  mysql:
    image: mariadb:10.5
    container_name: zabbix-mysql
    restart: unless-stopped
    environment:
      - MYSQL_ROOT_PASSWORD=${MYSQL_ROOT_PASSWORD}
      - MYSQL_DATABASE=${MYSQL_DATABASE}
      - MYSQL_USER=${MYSQL_USER}
      - MYSQL_PASSWORD=${MYSQL_PASSWORD}
    volumes:
      - mysql_data:/var/lib/mysql
    command: --character-set-server=utf8 --collation-server=utf8_bin

  zabbix-server:
    image: zabbix/zabbix-server-mysql:${ZABBIX_VERSION}
    container_name: zabbix-server
    depends_on:
      - mysql
    ports:
      - "10051:10051"
    environment:
      - DB_SERVER_HOST=mysql
      - MYSQL_DATABASE=${MYSQL_DATABASE}
      - MYSQL_USER=${MYSQL_USER}
      - MYSQL_PASSWORD=${MYSQL_PASSWORD}
    volumes:
      - zbx_alertscripts:/usr/lib/zabbix/alertscripts
      - zbx_modules:/usr/lib/zabbix/modules

  zabbix-web:
    image: zabbix/zabbix-web-nginx-mysql:${ZABBIX_VERSION}
    container_name: zabbix-web
    depends_on:
      - zabbix-server
      - mysql
    ports:
      - "80:8080"
      - "443:8443"
    environment:
      - DB_SERVER_HOST=mysql
      - MYSQL_DATABASE=${MYSQL_DATABASE}
      - MYSQL_USER=${MYSQL_USER}
      - MYSQL_PASSWORD=${MYSQL_PASSWORD}
      - PHP_TZ=America/Bogota

volumes:
  mysql_data:
  zbx_alertscripts:
  zbx_modules:
```

---

## `ansible/roles/zabbix/tasks/main.yml`

```yaml
---
- name: Copiar docker-compose.yml
  copy:
    src: files/docker-compose.yml
    dest: /home/vagrant/zabbix-docker-compose.yml
    owner: vagrant
    group: vagrant
    mode: '0644'

- name: Plantilla de .env para docker-compose
  copy:
    dest: /home/vagrant/.zabbix_env
    content: |
      MYSQL_ROOT_PASSWORD={{ mysql_root_password }}
      MYSQL_DATABASE={{ mysql_database }}
      MYSQL_USER={{ mysql_user }}
      MYSQL_PASSWORD={{ mysql_password }}
      ZABBIX_VERSION={{ zabbix_version }}
    owner: vagrant
    group: vagrant
    mode: '0600'

- name: Crear directorio de datos
  file:
    path: /home/vagrant/zabbix_data
    state: directory
    owner: vagrant
    group: vagrant
    mode: '0755'

- name: Instalar docker-compose (si no existe) - se repite seguridad
  pip:
    name: docker-compose
    state: present

- name: Levantar servicios con docker-compose
  become: true
  become_user: vagrant
  shell: |
    set -e
    cd /home/vagrant
    export MYSQL_ROOT_PASSWORD={{ mysql_root_password }}
    export MYSQL_DATABASE={{ mysql_database }}
    export MYSQL_USER={{ mysql_user }}
    export MYSQL_PASSWORD={{ mysql_password }}
    export ZABBIX_VERSION={{ zabbix_version }}
    docker-compose -f zabbix-docker-compose.yml up -d
  args:
    creates: /var/lib/docker/volumes
```

---

## `README.md` (resumen de uso)

```markdown
# Zabbix Lab con Vagrant + Ansible

Pasos para levantar el laboratorio:

1. Coloca este repositorio en una carpeta local.
2. `vagrant up` — Vagrant levantará la VM y ejecutará Ansible local para provisionar.
3. Accede a la interfaz web en: http://localhost:8080 (user: Admin, pass: zabbix)
4. Detener: `vagrant halt` — Eliminar: `vagrant destroy -f`

Notas:
- Este entorno es para pruebas y demos. Para producción separa la BD y usa backups.
- Cambia las contraseñas por variables más seguras en `ansible/group_vars/all.yml`.
```

---

### Comentarios finales

* El playbook usa `docker` + `docker-compose` para simplificar despliegue en laboratorio.
* Si prefieres instalar Zabbix directamente sobre la VM (sin contenedores) lo adapto.
* Puedes copiar/pegar esta estructura; luego `vagrant up` creará y provisionará la VM.

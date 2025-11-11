# Vagrant + Ansible — Laboratorio Zabbix con Monitoreo de Red

Este proyecto provisiona automáticamente un entorno completo de laboratorio Zabbix usando Vagrant + Ansible. Despliega una VM Ubuntu 20.04 con Zabbix server, web UI y MySQL/MariaDB usando Docker Compose, optimizado para monitoreo de red con soporte SNMP y auto-discovery.

## ✨ Características Principales

- 🏗️ **Despliegue automatizado** con Vagrant + Ansible
- 🌐 **Networking bridge** para acceso directo a red física
- 📡 **Monitoreo SNMP** preconfigurado con herramientas
- 🔍 **Auto-discovery** de dispositivos de red
- 🖥️ **Interfaz web moderna** con Zabbix 6.0
- 📊 **Monitoreo MikroTik** con OIDs específicos
- 🐳 **Docker Compose V2** para gestión de contenedores

## 🚀 Inicio Rápido

```bash
# Clonar/descargar el proyecto
cd Ansible-zabbix/

# Levantar el entorno (primera vez puede tomar 10-15 min)
vagrant up

# Acceder a Zabbix desde red física
# Web UI: http://192.168.173.20
# Usuario: Admin | Password: zabbix
```

## 📁 Estructura del proyecto

```
Ansible-zabbix/
├── Vagrantfile                     # Configuración de la VM con bridge network
├── README.md                       # Esta documentación
├── docs/                          # Documentación adicional
│   ├── configurar-mikrotik-zabbix.md  # Configuración MikroTik
│   ├── mikrotik-snmp-setup.md         # Setup SNMP en MikroTik
│   └── snmp-config-examples.md        # Ejemplos de configuración SNMP
└── ansible/
    ├── playbook.yml                # Playbook principal con SNMP tools
    ├── inventory.ini               # Inventario de hosts
    ├── group_vars/
    │   └── all.yml                 # Variables globales (passwords, versiones)
    └── roles/
        ├── docker/
        │   └── tasks/
        │       └── main.yml        # Instalación Docker + Docker Compose V2
        └── zabbix/
            ├── files/
            │   └── docker-compose.yml  # Stack completo Zabbix con variables
            └── tasks/
                └── main.yml        # Despliegue Zabbix + SNMP tools
```

## ⚙️ Componentes Desplegados

| Servicio | Imagen | Puerto | Función |
|----------|--------|--------|---------|
| **MySQL** | `mariadb:10.5` | 3306 | Base de datos Zabbix |
| **Zabbix Server** | `zabbix/zabbix-server-mysql:6.0-alpine-latest` | 10051 | Motor principal de monitoreo |
| **Zabbix Web** | `zabbix/zabbix-web-nginx-mysql:6.0-alpine-latest` | 80, 443 | Interfaz web con Nginx |

### 🛠️ Herramientas SNMP Incluidas
- **snmp**: Cliente SNMP básico
- **snmp-mibs-downloader**: MIBs adicionales
- **snmpwalk**: Exploración de OIDs
- **snmptranslate**: Traducción de OIDs

## 🌐 Configuración de Red

### Networking Bridge
La VM está configurada con **bridge networking** para acceso directo a la red física:
- **IP de la VM**: `192.168.173.20` (ajustable)
- **Acceso a dispositivos**: Comunicación directa con routers, switches, etc.
- **Protocolo SNMP**: Habilitado para monitoreo de red

### Acceso al entorno

#### Desde red física:
- **Web UI HTTP**: http://192.168.173.20
- **Web UI HTTPS**: https://192.168.173.20:443
- **Zabbix Server**: 192.168.173.20:10051

#### Desde localhost (port forwarding):
- **Web UI HTTP**: http://localhost:8080 
- **Web UI HTTPS**: https://localhost:8443
- **Zabbix Server**: localhost:10051

### Credenciales por defecto
- **Usuario**: `Admin`
- **Password**: `zabbix`

## 🔧 Configuración

### Variables principales (`ansible/group_vars/all.yml`)

```yaml
---
# Versión de Zabbix
zabbix_version: "6.0-alpine-latest"

# Credenciales de base de datos
mysql_root_password: "changeme123"
mysql_database: "zabbix"
mysql_user: "zabbix"
mysql_password: "zabbix_pass"
```

### Recursos de VM (`Vagrantfile`)

```ruby
# Configuración de la VM con bridge network
config.vm.provider "virtualbox" do |vb|
  vb.memory = 4096  # RAM en MB
  vb.cpus = 2       # Núcleos de CPU
end

# Bridge network para acceso a red física
config.vm.network "public_network", 
  ip: "192.168.173.20",  # IP fija en red física
  bridge: "auto"         # Selección automática de interfaz
```

## 📡 Monitoreo SNMP y MikroTik

### Configuración SNMP Verificada
- **Dispositivo objetivo**: MikroTik CCR2004-16G-2S+
- **IP del router**: `192.168.173.1`
- **Comunidad SNMP**: `public` (SNMPv2c)
- **Puerto SNMP**: `161` (estándar)

### OIDs MikroTik Importantes

#### CPU y Sistema
```bash
# CPU Load (%)
snmpget -v2c -c public 192.168.173.1 1.3.6.1.4.1.14988.1.1.3.14.0

# CPU Temperature (°C)
snmpget -v2c -c public 192.168.173.1 1.3.6.1.4.1.14988.1.1.3.100.1.3.17

# System Description
snmpget -v2c -c public 192.168.173.1 1.3.6.1.2.1.1.1.0

# Uptime
snmpget -v2c -c public 192.168.173.1 1.3.6.1.2.1.1.3.0
```

#### Interfaces de Red
```bash
# Interface Names
snmpwalk -v2c -c public 192.168.173.1 1.3.6.1.2.1.2.2.1.2

# Interface Status
snmpwalk -v2c -c public 192.168.173.1 1.3.6.1.2.1.2.2.1.8

# Interface Traffic (In/Out Octets)
snmpwalk -v2c -c public 192.168.173.1 1.3.6.1.2.1.2.2.1.10  # In
snmpwalk -v2c -c public 192.168.173.1 1.3.6.1.2.1.2.2.1.16  # Out
```

### Auto-Discovery Configurado
- **Rango de red**: `192.168.173.1-254`
- **Protocolo**: ICMP ping + SNMP
- **Frecuencia**: Cada 5 minutos
- **Ubicación**: `Configuration → Discovery → Network Discovery`

## 🔍 Configuración de Auto-Discovery

### 1. Network Discovery Rule
```
Name: LAN Discovery
IP range: 192.168.173.1-254
Update interval: 5m
Checks: 
  - ICMP ping
  - SNMP v2 community "public"
Device uniqueness: IP address
Host name: DNS name
```

### 2. Discovery Actions (Recomendado)
```
Configuration → Actions → Discovery actions

Action: Auto-add discovered hosts
Conditions:
  - Discovery status = Up
  - Service type = SNMP
Operations:
  - Add host
  - Add to group "Discovered hosts"
  - Link template "Template Net SNMP Generic"
```

## 🛠️ Comandos útiles

### Gestión del entorno
```bash
# Levantar VM
vagrant up

# Parar VM (mantiene datos)
vagrant halt

# Reiniciar VM
vagrant reload

# Ejecutar solo provisionamiento
vagrant provision

# Destruir VM completamente
vagrant destroy -f
```

### Acceso SSH a la VM
```bash
# SSH interactivo
vagrant ssh

# Ejecutar comando único
vagrant ssh -c "docker ps"
```

### Gestión de contenedores (dentro de VM)
```bash
# Estado de contenedores
docker ps

# Logs de servicios específicos
docker logs zabbix-server
docker logs zabbix-web
docker logs zabbix-mysql

# Reiniciar servicios con Docker Compose V2
docker compose -f /home/vagrant/zabbix-docker-compose.yml restart

# Parar servicios
docker compose -f /home/vagrant/zabbix-docker-compose.yml down

# Levantar servicios
docker compose -f /home/vagrant/zabbix-docker-compose.yml up -d

# Reiniciar solo un servicio específico
docker restart zabbix-web
```

### Comandos SNMP útiles
```bash
# Acceder a la VM
vagrant ssh

# Test básico SNMP
snmpget -v2c -c public 192.168.173.1 1.3.6.1.2.1.1.1.0

# Explorar CPU MikroTik
snmpget -v2c -c public 192.168.173.1 1.3.6.1.4.1.14988.1.1.3.14.0

# Listar interfaces
snmpwalk -v2c -c public 192.168.173.1 1.3.6.1.2.1.2.2.1.2

# Verificar temperatura CPU
snmpget -v2c -c public 192.168.173.1 1.3.6.1.4.1.14988.1.1.3.100.1.3.17
```

## ✅ Verificación de funcionamiento

### 1. Verificar contenedores
```bash
vagrant ssh -c "docker ps"
# Debe mostrar 3 contenedores corriendo con estado 'Up'
# - zabbix-server (puerto 10051)
# - zabbix-web (puertos 80, 443)
# - zabbix-mysql (puerto 3306)
```

### 2. Test de conectividad web
```bash
# Desde red física
curl -I http://192.168.173.20
# Debe retornar HTTP/1.1 200 OK

# Desde localhost
curl -I http://localhost:8080
```

### 3. Verificar bridge network
```bash
# Ping desde VM al router
vagrant ssh -c "ping -c 3 192.168.173.1"
# Debe mostrar respuestas exitosas

# Verificar IP de la VM
vagrant ssh -c "ip addr show"
# Debe mostrar 192.168.173.20 en la interfaz de red
```

### 4. Test SNMP funcional
```bash
# Test básico SNMP al router
vagrant ssh -c "snmpget -v2c -c public 192.168.173.1 1.3.6.1.2.1.1.1.0"
# Debe retornar descripción del sistema

# Test CPU MikroTik
vagrant ssh -c "snmpget -v2c -c public 192.168.173.1 1.3.6.1.4.1.14988.1.1.3.14.0"
# Debe retornar valor de CPU load
```

### 5. Verificar auto-discovery
```bash
# Acceder a: http://192.168.173.20
# Ir a: Monitoring → Discovery
# Debe mostrar dispositivos descubiertos en red 192.168.173.x
```

### 6. Verificar logs del servidor
```bash
vagrant ssh -c "docker logs zabbix-server --tail 20"
# Los logs deben mostrar:
# - "server #0 started [main process]"
# - Sin errores de conectividad a MySQL
# - Sin errores críticos de configuración
```

## 🔒 Seguridad

### ⚠️ **IMPORTANTE para producción:**

1. **Cambiar passwords por defecto** en `ansible/group_vars/all.yml`
2. **Usar HTTPS** únicamente (puerto 8443)
3. **Configurar firewall** para limitar acceso
4. **Backups regulares** de la base de datos
5. **Actualizar imágenes** Docker periódicamente

### Cambio de passwords
```yaml
# Editar ansible/group_vars/all.yml
mysql_root_password: "tu_password_seguro"
mysql_password: "otro_password_seguro"
```

Luego re-provisionar:
```bash
vagrant destroy -f && vagrant up
```

## 🐛 Troubleshooting

### Problema: Error de permisos Docker
```bash
# Síntoma: "docker: permission denied"
# Solución: Re-provisionar o ejecutar manualmente
vagrant provision

# O dentro de la VM:
vagrant ssh
sudo usermod -aG docker vagrant
newgrp docker
```

### Problema: "Connection to Zabbix server refused"
```bash
# Síntoma: Error en web UI al conectar al servidor
# Verificar variables de entorno del contenedor web:
vagrant ssh -c "docker logs zabbix-web | grep -i server"

# Solución: Verificar docker-compose.yml tiene:
# ZBX_SERVER_HOST=zabbix-server
# ZBX_SERVER_PORT=10051

# Reiniciar servicio web:
vagrant ssh -c "docker restart zabbix-web"
```

### Problema: Bridge network no funciona
```bash
# Síntoma: No se puede acceder a 192.168.173.20
# Verificar configuración de bridge:
vagrant ssh -c "ip route"

# Si falla, editar Vagrantfile y cambiar interfaz:
config.vm.network "public_network", 
  ip: "192.168.173.20",
  bridge: "eth0"  # Especificar interfaz específica
```

### Problema: SNMP no responde
```bash
# Verificar conectividad básica:
vagrant ssh -c "ping -c 3 192.168.173.1"

# Test SNMP básico:
vagrant ssh -c "snmpget -v2c -c public 192.168.173.1 1.3.6.1.2.1.1.1.0"

# Si falla, verificar:
# 1. SNMP habilitado en MikroTik
# 2. Comunidad correcta
# 3. ACL/firewall no bloquea puerto 161
```

### Problema: Puertos ocupados
```bash
# Verificar qué usa el puerto
sudo lsof -i :8080

# Cambiar puertos en Vagrantfile si es necesario
config.vm.network "forwarded_port", guest: 80, host: 8081
```

### Problema: VM sin memoria
```bash
# Síntoma: VM lenta o contenedores fallan
# Aumentar RAM en Vagrantfile
vb.memory = 6144  # 6GB en lugar de 4GB
```

### Problema: Contenedores no inician
```bash
# SSH a la VM y verificar logs detallados
vagrant ssh
docker logs zabbix-server
docker logs zabbix-mysql

# Verificar espacio en disco:
df -h

# Verificar status de Docker:
sudo systemctl status docker
```

### Problema: Auto-discovery no encuentra dispositivos
```bash
# Verificar rango de red correcto:
# Ir a Configuration → Discovery → Network Discovery
# Verificar IP range: 192.168.173.1-254

# Test manual de dispositivos:
vagrant ssh -c "nmap -sn 192.168.173.1-10"

# Verificar checks habilitados:
# - ICMP ping ✓
# - SNMP v2 community "public" ✓
```

### Problema: Docker Compose V1 vs V2
```bash
# Síntoma: Comando "docker-compose" no encontrado
# Verificar versión:
vagrant ssh -c "docker compose version"

# Usar comandos V2:
docker compose up -d    # NO: docker-compose up -d
docker compose restart  # NO: docker-compose restart
```

## 📚 Logs importantes

| Componente | Ubicación de logs |
|------------|-------------------|
| Vagrant | `vagrant up` output |
| Ansible | VM: `/var/log/ansible.log` |
| Zabbix Server | `docker logs zabbix-server` |
| Zabbix Web | `docker logs zabbix-web` |
| MySQL | `docker logs zabbix-mysql` |
| SNMP queries | Consola durante `snmpget/snmpwalk` |
| Bridge Network | `vagrant ssh -c "ip addr show"` |

## 🎯 Casos de uso

✅ **Ideal para:**
- Monitoreo de red empresarial/doméstica
- Pruebas de configuración Zabbix + SNMP
- Laboratorio de auto-discovery de dispositivos
- Desarrollo de plantillas para MikroTik
- Training y demos de monitoreo de red
- Testing de agentes Zabbix
- Laboratorios educativos de redes
- Prototipado de dashboards de red

✅ **Dispositivos soportados:**
- Routers MikroTik (todos los modelos)
- Switches gestionables con SNMP
- Servidores con SNMP habilitado
- UPS con interfaz SNMP
- Dispositivos IoT compatibles
- Equipos de red Cisco, HP, etc.

❌ **NO usar para:**
- Entornos de producción críticos
- Datos críticos sin backup
- Monitoreo 24/7 sin redundancia
- Redes con más de 100 dispositivos

## 🤝 Contribuir

1. Fork del repositorio
2. Crear feature branch
3. Commit de cambios
4. Push al branch
5. Crear Pull Request

## 📄 Licencia

Este proyecto está bajo licencia MIT. Ver archivo LICENSE para detalles.

---

## 🏆 Resultado del Laboratorio

### ✅ Estado Actual Verificado
```
✓ VM Ubuntu 20.04 desplegada con 4GB RAM, 2 CPU
✓ Bridge network configurado (192.168.173.20)
✓ Docker Compose V2 instalado y funcionando
✓ 3 contenedores Zabbix corriendo sin errores:
  - zabbix-server (6.0-alpine-latest)
  - zabbix-web (nginx + mysql)
  - mariadb (10.5)
✓ Herramientas SNMP instaladas y funcionales
✓ Conectividad verificada con MikroTik CCR2004-16G-2S+
✓ OIDs MikroTik funcionando (CPU: 0%, Temp: 52°C)
✓ Auto-discovery configurado para red 192.168.173.x
✓ Interfaz web accesible y operativa
✓ Troubleshooting completado (ZBX_SERVER_HOST configurado)
```

### 🌐 URLs de Acceso
- **Interfaz Principal**: http://192.168.173.20
- **HTTPS**: https://192.168.173.20
- **Localhost**: http://localhost:8080 (port forwarding)

### 🔑 Credenciales
- **Usuario**: `Admin`
- **Password**: `zabbix`

### 📊 Próximos Pasos Recomendados
1. **Configurar Discovery Actions** para auto-agregar hosts
2. **Crear templates** personalizados para MikroTik
3. **Setup de alertas** por email/Telegram
4. **Dashboards** con métricas de red en tiempo real
5. **Monitoreo de interfaces** específicas del router
6. **Thresholds** para CPU, temperatura y ancho de banda

## 📞 Soporte

Para issues y preguntas:
- GitHub Issues: [Crear issue](../../issues)
- Documentación adicional: Ver carpeta `/docs/`
- Wiki del proyecto: [Ver documentación](../../wiki)

### 📋 Información del Sistema Probado
```
Host: Ubuntu Linux
VM: VirtualBox con Ubuntu 20.04 LTS
Zabbix: 6.0 Alpine Latest
Router: MikroTik CCR2004-16G-2S+ (RouterOS)
Red: 192.168.173.x/24
Fecha: Noviembre 2025
```

**¡Laboratorio Zabbix + SNMP totalmente funcional!** 🚀🔧📡

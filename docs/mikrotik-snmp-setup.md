# Configuración Zabbix para Router MikroTik con SNMP v2

## 🔧 1. Configuración en el MikroTik

### Habilitar SNMP en RouterOS:
```bash
# Conectar por SSH/Winbox al MikroTik y ejecutar:
/snmp set enabled=yes
/snmp community add name=public addresses=0.0.0.0/0 read-access=yes write-access=no

# Para mayor seguridad, cambiar comunidad y limitar IP:
/snmp community add name=zabbix_ro addresses=IP_DE_ZABBIX/32 read-access=yes write-access=no

# Verificar configuración:
/snmp print
/snmp community print
```

### Información del sistema:
```bash
# Ver información básica del router:
/system identity print
/system resource print
/interface print
```

## 🌐 2. Configuración en Zabbix Web UI

### Paso 1: Crear Host
1. **Ir a**: Configuration → Hosts → Create host
2. **Configurar**:
   ```
   Host name: mikrotik-router
   Visible name: MikroTik Router Principal
   Groups: Network devices (crear si no existe)
   ```

### Paso 2: Configurar Interfaz SNMP
```
Interfaces:
  Type: SNMP
  IP address: 192.168.1.1 (IP de tu MikroTik)
  DNS name: (vacío)
  Port: 161
  
SNMP details:
  SNMP version: SNMPv2
  SNMP community: public (o tu comunidad personalizada)
  Use bulk requests: ✓ (marcado)
  Security name: (vacío)
  Security level: (vacío)
```

### Paso 3: Asignar Plantillas
Buscar y añadir estas plantillas:
- **Template Net Generic SNMP** (básica)
- **Template Module ICMP Ping** (conectividad)
- **Template Net Mikrotik SNMPv2** (específica, si está disponible)

## 📊 3. Items SNMP específicos para MikroTik

### Información del Sistema:
```yaml
# System Description
Name: System description
Type: SNMPv2 agent
Key: system.descr
SNMP OID: 1.3.6.1.2.1.1.1.0
Update interval: 1h

# RouterOS Version
Name: RouterOS version
Type: SNMPv2 agent  
Key: mikrotik.version
SNMP OID: 1.3.6.1.4.1.14988.1.1.4.4.0
Update interval: 1h

# System Uptime
Name: System uptime
Type: SNMPv2 agent
Key: system.uptime
SNMP OID: 1.3.6.1.2.1.1.3.0
Update interval: 60s
```

### Recursos del Router:
```yaml
# CPU Load
Name: CPU usage
Type: SNMPv2 agent
Key: mikrotik.cpu.usage
SNMP OID: 1.3.6.1.2.1.25.3.3.1.2.1
Update interval: 60s

# Memory Usage
Name: Memory usage percentage  
Type: SNMPv2 agent
Key: mikrotik.memory.usage
SNMP OID: 1.3.6.1.4.1.14988.1.1.1.2.0
Update interval: 60s

# Free Memory
Name: Free memory
Type: SNMPv2 agent
Key: mikrotik.memory.free
SNMP OID: 1.3.6.1.4.1.14988.1.1.1.1.0
Update interval: 60s

# Temperature (si está disponible)
Name: System temperature
Type: SNMPv2 agent
Key: mikrotik.temperature
SNMP OID: 1.3.6.1.4.1.14988.1.1.3.10.0
Update interval: 300s
```

### Interfaces de Red:
```yaml
# Interface Discovery Rule
Name: Network interface discovery
Type: SNMPv2 agent
Key: net.if.discovery
SNMP OID: discovery[{#IFNAME},1.3.6.1.2.1.2.2.1.2,{#IFDESCR},1.3.6.1.2.1.2.2.1.2,{#IFTYPE},1.3.6.1.2.1.2.2.1.3]
Update interval: 3600s

# Item Prototypes (se crean automáticamente):
- Interface {#IFNAME}: Inbound traffic
- Interface {#IFNAME}: Outbound traffic  
- Interface {#IFNAME}: Operational status
- Interface {#IFNAME}: Speed
```

## 🚨 4. Triggers (Alertas) para MikroTik

```yaml
# Router Down
Name: MikroTik router is unreachable
Expression: max(/mikrotik-router/icmpping,#3)=0
Severity: High
Description: Router no responde a ping por más de 3 intentos

# High CPU
Name: High CPU usage on MikroTik
Expression: avg(/mikrotik-router/mikrotik.cpu.usage,5m)>80  
Severity: Warning
Description: CPU usage over 80% for 5 minutes

# High Memory
Name: High memory usage on MikroTik
Expression: last(/mikrotik-router/mikrotik.memory.usage)>85
Severity: Warning  
Description: Memory usage over 85%

# Interface Down
Name: Interface {#IFNAME} is down
Expression: last(/mikrotik-router/net.if.status[{#SNMPINDEX}])=2
Severity: Average
Description: Network interface is down

# High Temperature (si aplica)
Name: High system temperature
Expression: last(/mikrotik-router/mikrotik.temperature)>70
Severity: Warning
Description: System temperature above 70°C
```

## 📈 5. Gráficos útiles

### Tráfico de Interfaces:
```yaml
Name: Network traffic on {#IFNAME}
Items:
  - Interface {#IFNAME}: Incoming traffic (verde)
  - Interface {#IFNAME}: Outgoing traffic (azul)
Type: Normal
Y axis: Network traffic in bps
Show legend: Yes
```

### Recursos del Sistema:
```yaml  
Name: System resources
Items:
  - CPU usage (rojo)
  - Memory usage percentage (azul)
  - System uptime (verde)
Y axis min: 0
Y axis max: 100
```

## 🧪 6. Comandos de Testing

### Desde la VM Zabbix:
```bash
# Test conectividad básica
ping -c 3 192.168.1.1

# Test SNMP conectividad
snmpwalk -v2c -c public 192.168.1.1 1.3.6.1.2.1.1.1.0

# Obtener información del sistema MikroTik
snmpget -v2c -c public 192.168.1.1 1.3.6.1.4.1.14988.1.1.4.4.0  # RouterOS version
snmpget -v2c -c public 192.168.1.1 1.3.6.1.2.1.1.3.0              # Uptime
snmpget -v2c -c public 192.168.1.1 1.3.6.1.4.1.14988.1.1.1.2.0    # Memory usage

# Listar interfaces
snmpwalk -v2c -c public 192.168.1.1 1.3.6.1.2.1.2.2.1.2

# Ver tráfico de interface (index 1 = primera interface)
snmpget -v2c -c public 192.168.1.1 1.3.6.1.2.1.2.2.1.10.1  # Bytes in
snmpget -v2c -c public 192.168.1.1 1.3.6.1.2.1.2.2.1.16.1  # Bytes out
```

## 🔒 7. Mejores prácticas de seguridad

### En MikroTik:
```bash
# Crear comunidad personalizada
/snmp community add name=zabbix_monitoring addresses=IP_ZABBIX/32 read-access=yes

# Deshabilitar comunidad por defecto (si existe)
/snmp community remove [find name=public]

# Limitar acceso SNMP por firewall
/ip firewall filter add chain=input protocol=udp dst-port=161 src-address=IP_ZABBIX action=accept
/ip firewall filter add chain=input protocol=udp dst-port=161 action=drop
```

### En Zabbix:
- Usar comunidades únicas por dispositivo
- Configurar macros para facilitar cambios
- Implementar discovery rules para auto-detección
- Configurar maintenance windows para updates

## 📊 8. Dashboard sugerido

Crear dashboard "Network Overview" con widgets:
1. **Network map** - Topología visual
2. **Host availability** - Estado ICMP de dispositivos  
3. **Top interfaces by utilization** - Interfaces más utilizadas
4. **Problems** - Lista de alertas activas
5. **System performance** - CPU/Memory en tiempo real
6. **Interface status** - Up/Down de todas las interfaces

## 💡 Tips adicionales:

- **Intervalo de polling**: 60s para métricas importantes, 300s para información estática
- **Retention**: Configurar diferentes períodos según la importancia de los datos
- **Bulk requests**: Activar para mejor performance SNMP
- **Preprocessing**: Usar para convertir bytes a bits, aplicar factores de escala, etc.
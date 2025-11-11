# Configuración SNMP para Zabbix

## 1. Configurar host SNMP en Zabbix

### Crear host:
- **Host name**: `Switch-Core-01`
- **Visible name**: `Switch Core Principal`
- **Groups**: `Network Switches`
- **Interfaces**:
  - Type: `SNMP`
  - IP address: `192.168.1.10`
  - Port: `161`
  - SNMP version: `SNMPv2`
  - SNMP community: `public` (cambiar por comunidad segura)

### Templates recomendadas:
- `Template Net Generic SNMP`
- `Template Net Cisco IOS SNMP`
- `Template Net HP Comware HH3C SNMP`
- `Template Net Juniper SNMP`

## 2. Items SNMP más útiles para red

### Información del sistema:
```yaml
- Name: "System description"
  Type: SNMPv2 agent
  Key: system.descr
  SNMP OID: 1.3.6.1.2.1.1.1.0

- Name: "System uptime"
  Type: SNMPv2 agent  
  Key: system.uptime
  SNMP OID: 1.3.6.1.2.1.1.3.0

- Name: "System name"
  Type: SNMPv2 agent
  Key: system.name
  SNMP OID: 1.3.6.1.2.1.1.5.0
```

### Interfaces de red:
```yaml
- Name: "Interface {#IFNAME}: Inbound traffic"
  Type: SNMPv2 agent
  Key: net.if.in[{#SNMPINDEX}]
  SNMP OID: 1.3.6.1.2.1.2.2.1.10.{#SNMPINDEX}

- Name: "Interface {#IFNAME}: Outbound traffic"  
  Type: SNMPv2 agent
  Key: net.if.out[{#SNMPINDEX}]
  SNMP OID: 1.3.6.1.2.1.2.2.1.16.{#SNMPINDEX}

- Name: "Interface {#IFNAME}: Operational status"
  Type: SNMPv2 agent
  Key: net.if.status[{#SNMPINDEX}]
  SNMP OID: 1.3.6.1.2.1.2.2.1.8.{#SNMPINDEX}
```

### CPU y Memoria:
```yaml
- Name: "CPU utilization"
  Type: SNMPv2 agent
  Key: system.cpu.util
  SNMP OID: 1.3.6.1.4.1.9.9.109.1.1.1.1.7.1

- Name: "Memory utilization"
  Type: SNMPv2 agent
  Key: vm.memory.util
  SNMP OID: 1.3.6.1.4.1.9.9.48.1.1.1.5.1
```

## 3. Discovery Rules (Autodescubrimiento)

### Interface discovery:
```yaml
- Name: "Network interface discovery"
  Type: SNMPv2 agent
  Key: net.if.discovery
  SNMP OID: discovery[{#IFNAME},1.3.6.1.2.1.2.2.1.2,{#IFDESCR},1.3.6.1.2.1.2.2.1.2]
  Update interval: 3600s
  
  Filters:
  - Macro: {#IFNAME}
    Regular expression: ^(Ethernet|GigabitEthernet|FastEthernet).*
```

## 4. Triggers (Alertas)

### ICMP Triggers:
```yaml
- Name: "Host is unreachable by ICMP"
  Expression: max(/Template Net ICMP Ping/icmpping,#3)=0
  Severity: High

- Name: "High ICMP response time"
  Expression: avg(/Template Net ICMP Ping/icmppingsec,5m)>0.5
  Severity: Warning
```

### SNMP Triggers:
```yaml
- Name: "Interface {#IFNAME} is down"
  Expression: last(/Template Net Generic SNMP/net.if.status[{#SNMPINDEX}])=2
  Severity: Average

- Name: "High CPU utilization on {HOST.NAME}"
  Expression: avg(/Template Net Generic SNMP/system.cpu.util,5m)>80
  Severity: Warning

- Name: "High interface utilization on {#IFNAME}"
  Expression: (avg(/Template Net Generic SNMP/net.if.in[{#SNMPINDEX}],15m)>({$IF.UTIL.MAX:"{#IFNAME}"}/100)*last(/Template Net Generic SNMP/net.if.speed[{#SNMPINDEX}]) or avg(/Template Net Generic SNMP/net.if.out[{#SNMPINDEX}],15m)>({$IF.UTIL.MAX:"{#IFNAME}"}/100)*last(/Template Net Generic SNMP/net.if.speed[{#SNMPINDEX}])) and last(/Template Net Generic SNMP/net.if.speed[{#SNMPINDEX}])>0
  Severity: Warning
```

## 5. Graficos y Dashboards

### Gráfico de tráfico de interfaz:
- **Items**: 
  - Interface inbound traffic (Net.if.in)
  - Interface outbound traffic (Net.if.out)
- **Type**: Normal
- **Y axis**: Network traffic in bps

### Dashboard de red:
- **Widgets**:
  - Network availability (ICMP status)
  - Interface status (Up/Down)  
  - Top interfaces by utilization
  - Network map
  - Problem indicators

## 6. Comandos de verificación SNMP

### Desde la VM de Zabbix:
```bash
# Test SNMP connectivity
snmpwalk -v2c -c public 192.168.1.1 1.3.6.1.2.1.1.1.0

# Get interface list
snmpwalk -v2c -c public 192.168.1.1 1.3.6.1.2.1.2.2.1.2

# Get interface traffic
snmpwalk -v2c -c public 192.168.1.1 1.3.6.1.2.1.2.2.1.10
```

## 7. Mejores prácticas

### Seguridad SNMP:
- Usar SNMPv3 con autenticación
- Cambiar comunidades por defecto
- Limitar acceso SNMP por IP
- Usar comunidades de solo lectura

### Performance:
- Intervalos apropiados (60s para ICMP, 300s para SNMP)
- Usar bulk requests para SNMP
- Implementar value mapping para status
- Configurar triggers con hysteresis

### Organización:
- Agrupar devices por función (Routers, Switches, Access Points)
- Usar naming conventions consistentes
- Documentar OIDs personalizados
- Mantener templates actualizadas
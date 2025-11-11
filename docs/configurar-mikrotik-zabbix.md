# Configuración Router MikroTik en Zabbix

## Resumen
Tu Zabbix está ahora funcionando en **192.168.173.20** y puede acceder a tu router MikroTik en **192.168.173.1** usando SNMP v2.

## Acceso a Zabbix Web
- URL: http://192.168.173.20
- Usuario: `Admin`
- Password: `zabbix`

## Configuración del Router MikroTik

### 1. Verificar configuración SNMP en MikroTik
Conecta al router vía WinBox/SSH y verifica que SNMP esté habilitado:
```
/snmp print
```

Si no está configurado, habilítalo:
```
/snmp set enabled=yes contact="admin@empresa.com" location="Oficina Principal"
/snmp community add name=public security=read
```

### 2. Agregar Host en Zabbix

#### Paso 1: Ir a Configuration → Hosts
1. Accede a http://192.168.173.20
2. Ve a **Configuration** → **Hosts**
3. Haz clic en **Create host**

#### Paso 2: Configurar el Host
- **Host name**: `Router-MikroTik-CCR2004`
- **Visible name**: `Router MikroTik CCR2004-16G-2S+`
- **Groups**: Selecciona `Network devices` (crear si no existe)
- **Interfaces**: 
  - Type: `SNMP`
  - IP address: `192.168.173.1`
  - DNS name: (dejar vacío)
  - Port: `161`

#### Paso 3: Configurar SNMP
En la sección **SNMP interfaces**:
- **SNMP version**: `SNMPv2c`
- **SNMP community**: `public`

#### Paso 4: Asignar Templates
En la pestaña **Templates**:
1. Buscar y agregar: `Template Net Generic SNMP`
2. Opcional: `Template Module Generic SNMPv2`

#### Paso 5: Configurar Macros (Opcional)
En la pestaña **Macros**, puedes agregar:
- `{$SNMP_COMMUNITY}`: `public`
- `{$ICMP_LOSS_WARN}`: `20`
- `{$ICMP_RESPONSE_TIME_WARN}`: `0.15`

### 3. Verificación

#### Comprobar conectividad SNMP desde Zabbix VM:
```bash
vagrant ssh -c "snmpwalk -v2c -c public 192.168.173.1 1.3.6.1.2.1.1.1.0"
```
Debería devolver: `RouterOS CCR2004-16G-2S+`

#### En Zabbix Web:
1. Ve a **Monitoring** → **Hosts**
2. Verifica que el host aparezca con estado **Enabled**
3. Espera unos minutos y verifica que el **Availability** muestre iconos verdes para SNMP

### 4. Monitoreo Disponible

Con el template Generic SNMP tendrás:
- **Ping/ICMP**: Latencia y pérdida de paquetes
- **Interfaces de red**: Tráfico in/out, errores, estado
- **CPU y memoria**: Uso de recursos del router
- **Temperatura**: Si está disponible vía SNMP
- **Uptime**: Tiempo de funcionamiento

### 5. Dashboards y Alertas

#### Crear Dashboard personalizado:
1. Ve a **Monitoring** → **Dashboard**
2. Crea widgets para:
   - Gráfico de tráfico de interfaces
   - Estado de disponibilidad
   - CPU y memoria
   - Mapa de red

#### Configurar alertas:
1. Ve a **Configuration** → **Actions** → **Trigger actions**
2. Configura alertas para:
   - Router no responde (ping down)
   - Alto uso de CPU (>80%)
   - Interface down
   - Alto tráfico en interfaces

## Troubleshooting

### Router no aparece o está en rojo:
1. Verificar que SNMP esté habilitado en MikroTik
2. Comprobar community string
3. Verificar conectividad de red
4. Revisar logs en **Administration** → **Audit**

### No se obtienen datos de interfaces:
- El template Generic SNMP es básico
- Para MikroTik específico, buscar templates de la comunidad
- O crear items personalizados con OIDs específicos de MikroTik

## Comandos de Verificación

### Desde la VM Zabbix:
```bash
# Test SNMP básico
vagrant ssh -c "snmpwalk -v2c -c public 192.168.173.1 1.3.6.1.2.1.1"

# Interfaces
vagrant ssh -c "snmpwalk -v2c -c public 192.168.173.1 1.3.6.1.2.1.2.2.1.2"

# Tráfico de interfaces
vagrant ssh -c "snmpwalk -v2c -c public 192.168.173.1 1.3.6.1.2.1.2.2.1.10"
```

### En MikroTik (vía SSH/Terminal):
```
# Ver configuración SNMP
/snmp print

# Ver interfaces
/interface print

# Ver estadísticas de interfaces
/interface monitor-traffic interface=ether1
```

## URLs Útiles
- Zabbix Web: http://192.168.173.20
- Templates MikroTik: https://github.com/akatrevorjay/zabbix-mikrotik
- MikroTik OID Reference: https://wiki.mikrotik.com/wiki/Manual:SNMP
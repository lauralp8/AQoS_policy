# NetApp ONTAP Adaptive QoS Policy Creation Script

Script automatizado para la creación de políticas adaptativas de calidad de servicio (AQoS) en NetApp ONTAP usando la API REST oficial.

## Descripción

Este script de Python automatiza el proceso de creación de políticas adaptativas de QoS (Adaptive Quality of Service) en NetApp ONTAP, permitiendo el control dinámico de rendimiento basado en espacio utilizado o asignado, incluyendo:

- Creación de políticas adaptativas de QoS con parámetros personalizados
- Configuración de IOPS esperados y pico por TB
- Configuración de IOPS mínimos absolutos
- Selección de tipo de asignación (allocated_space/used_space)
- Configuración de tamaño de bloque
- Verificación automática de la política creada
- Backup automático de event logs del cluster
- Sistema de logging con timestamp para todas las operaciones

## Requisitos

### Software
- Python 3.7 o superior
- NetApp ONTAP 9.6 o superior
- Acceso de red al cluster NetApp
- Credenciales de administrador del cluster

### Dependencias Python
```bash
pip install -r requirements.txt
```

## Estructura del Proyecto

```
aqos_policies/
├── aqos_policies.py   # Script principal
├── config.yaml        # Archivo de configuración
├── requirements.txt   # Dependencias Python
├── README.md          # Esta documentación
└── logs/              # Logs JSON generados automáticamente
```

## Configuración

Edita el archivo `config.yaml` con los parámetros de tu entorno. El archivo incluye las siguientes secciones:

### cluster
Configuración de conexión al cluster NetApp:
- `host`: Hostname o IP del cluster
- `username`: Usuario administrador
- `password`: Contraseña

### svm
Configuración de la Storage Virtual Machine:
- `name`: Nombre de la SVM de referencia

### aqos_policy
Configuración de la política adaptativa de QoS:
- `name`: Nombre único de la política
- `vserver`: Nombre de la SVM propietaria
- `expected_iops`: IOPS esperados por TB (Integer)
- `peak_iops`: IOPS pico por TB (Integer)
- `expected_iops_allocation`: Tipo de asignación para IOPS esperados (allocated_space/used_space)
- `peak_iops_allocation`: Tipo de asignación para IOPS pico (allocated_space/used_space)
- `absolute_min_iops`: IOPS mínimos absolutos (Integer)
- `block_size`: Tamaño de bloque (ANY, 4k, 8k, 16k, etc.) - Opcional

**Tipos de Asignación:**
- **`allocated_space`**: Los IOPS se calculan basándose en el espacio asignado del objeto
- **`used_space`**: Los IOPS se calculan basándose en el espacio utilizado del objeto

Para ver ejemplos de configuración, consulta el archivo `config.yaml` incluido en el proyecto.

## Sistema de Logging

El script implementa un sistema de logging automático que captura datos REALES de la cabina NetApp después de cada operación:

### Características
- **Timestamp automático**: Formato YYYYMMDD_HHMMSS (ej: 20260209_044500)
- **Formato JSON**: Datos estructurados y fáciles de procesar
- **Datos de cabina**: GET real desde ONTAP, no configuración enviada
- **Directorio logs/**: Se crea automáticamente si no existe

### Logs Generados

1. **aqos_policy_created_YYYYMMDD_HHMMSS.json**
   - UUID de la política
   - Nombre, SVM, estado
   - Configuración adaptativa (expected_iops, peak_iops, allocations)
   - Absolute minimum IOPS

2. **event_logs_YYYYMMDD_HHMMSS.json** (se generan DOS archivos)
   - **Primer backup**: Inmediatamente después de crear la política (50 eventos)
   - **Segundo backup**: Al finalizar el script (100 eventos)
   - Incluye: Index, timestamp, nodo, severidad, evento
   - **Nota**: Solo se muestran 20 eventos en pantalla, pero se guardan todos en el archivo JSON

## Uso

### Ejecución Básica
```bash
python aqos_policies.py
```

### Flujo de Ejecución

1. **Carga de configuración** - Lee y valida `config.yaml`
2. **Conexión al cluster** - Establece conexión y verifica credenciales
3. **Validación de parámetros** - Comprueba campos requeridos en aqos_policy
4. **Creación de política** - Crea la política adaptativa de QoS → Guarda log
5. **Verificación** - Obtiene detalles de la política creada desde ONTAP
6. **Primer Event Logs Backup** - Obtiene 50 eventos del cluster → Guarda log
7. **Segundo Event Logs Backup** - Obtiene 100 eventos del cluster → Guarda log
8. **Finalización** - Todos los logs disponibles en directorio `logs/`

### Equivalencia CLI

El script crea una política equivalente al siguiente comando ONTAP CLI:

```bash
qos adaptive-policy-group create \
  -policygroup NAS1200_policy \
  -vserver svm_1_cluster \
  -expected-iops 1200iops/tb \
  -peak-iops 3600iops/tb \
  -expected-iops-allocation allocated-space \
  -peak-iops-allocation allocated-space \
  -absolute-min-iops 1200iops \
  -block-size ANY
```

## API REST de NetApp

Este script utiliza la **API REST oficial de NetApp ONTAP**:

### Endpoints POST (Creación)
- **POST** `/api/storage/qos/policies` - Creación de política adaptativa de QoS

### Endpoints GET (Consulta)
- **GET** `/api/storage/qos/policies` - Consulta de políticas QoS
- **GET** `/api/support/ems/events` - Consulta de event logs

**Documentación oficial**: [NetApp ONTAP REST API](https://library.netapp.com/ecmdocs/ECMLP3351667/html/)

## Registro de Funciones

### Funciones de Configuración y Utilidades

#### config_loader(path="config.yaml")
Carga y valida el archivo de configuración YAML.
- **Entrada**: Ruta al archivo config.yaml
- **Salida**: Diccionario con configuración o None si falla
- **Validaciones**: Verifica estructura y secciones obligatorias (cluster, svm, aqos_policy)

#### save_to_log(operation_name, data)
Guarda datos en archivo JSON con timestamp en carpeta logs/.
- **Entrada**: Nombre de operación y diccionario de datos
- **Salida**: Ruta del archivo creado
- **Formato**: `logs/operacion_YYYYMMDD_HHMMSS.json`

#### cluster_connection(cluster_config)
Establece y verifica conexión con el cluster NetApp ONTAP.
- **Entrada**: Diccionario con host, username, password
- **Salida**: True si conexión exitosa, False si falla
- **Validaciones**: Prueba acceso con consulta al cluster

### Funciones de Gestión de Políticas AQoS

#### aqos_policies_creation(aqos_config)
Crea una política adaptativa de calidad de servicio en NetApp ONTAP.
- **Entrada**: Configuración de política AQoS desde config.yaml
- **Salida**: True si se creó, False si error
- **POST**: `/api/storage/qos/policies`
- **Log**: `aqos_policy_created_YYYYMMDD_HHMMSS.json`
- **Validaciones**: Verifica campos requeridos (name, vserver, expected_iops, peak_iops, allocations, absolute_min_iops)

### Funciones de Monitoreo

#### get_event_logs(max_records=100)
Obtiene y respalda los logs de eventos del cluster.
- **Entrada**: Número máximo de registros (default: 100)
- **Salida**: True si se obtuvieron, False si error
- **GET**: `/api/support/ems/events`
- **Log**: `event_logs_YYYYMMDD_HHMMSS.json`
- **Comportamiento**: 
  - Se ejecuta automáticamente DOS veces por el script
  - Primera llamada: Dentro de `aqos_policies_creation()` con 50 registros
  - Segunda llamada: Al final del script con 100 registros
  - Solo muestra los primeros 20 eventos en pantalla
  - Guarda todos los eventos solicitados en el archivo JSON

## Registro de Errores

### Errores de Configuración

#### ERR-001: Archivo de configuración no encontrado
```
[ERROR] File not found: config.yaml
```
**Causa**: El archivo config.yaml no existe en el directorio actual  
**Solución**: Verificar que config.yaml existe en la misma carpeta que el script

#### ERR-002: YAML inválido
```
[ERROR] Invalid YAML format in 'config.yaml'
```
**Causa**: Sintaxis YAML incorrecta (indentación, formato)  
**Solución**: Validar sintaxis YAML, verificar espacios e indentación

#### ERR-003: Configuración incompleta
```
[ERROR] Incomplete configuration: missing 'cluster' section
[ERROR] Incomplete configuration: missing 'svm' section
[ERROR] Missing 'aqos_policy' section in config.yaml
```
**Causa**: Faltan secciones obligatorias en config.yaml  
**Solución**: Asegurar que config.yaml contenga secciones 'cluster', 'svm' y 'aqos_policy'

#### ERR-004: Campos obligatorios faltantes
```
[ERROR] Missing required fields in cluster config: host, username
[ERROR] Missing required fields in AQoS configuration: name, vserver, expected_iops
```
**Causa**: Faltan campos obligatorios en la configuración  
**Solución**: Completar todos los campos requeridos en las secciones correspondientes

### Errores de Conexión

#### ERR-101: Error de autenticación
```
[ERROR] HTTP status: 401
[ERROR] Authentication failed
[ERROR] Invalid username or password
```
**Causa**: Credenciales incorrectas  
**Solución**: Verificar username y password en config.yaml

#### ERR-102: Acceso denegado
```
[ERROR] HTTP status: 403
[ERROR] Forbidden - User lacks required permissions
```
**Causa**: Usuario sin permisos de administrador  
**Solución**: Usar cuenta con rol admin o vsadmin

#### ERR-103: Host no alcanzable
```
[ERROR] Cannot reach host 'cluster1.demo.netapp.com'
```
**Causa**: Problemas de red o hostname incorrecto  
**Solución**: Verificar conectividad de red y hostname/IP del cluster

#### ERR-104: Timeout de conexión
```
[ERROR] Connection timeout to 'cluster1.demo.netapp.com'
```
**Causa**: Cluster no responde  
**Solución**: Verificar que el cluster esté encendido y accesible

### Errores de Creación de Políticas AQoS

#### ERR-201: Política ya existe
```
[ERROR] HTTP status: 409
[ERROR] Policy already exists with this name
```
**Causa**: Ya existe una política QoS con ese nombre  
**Solución**: Cambiar nombre en config.yaml o eliminar política existente

#### ERR-202: Parámetros inválidos
```
[ERROR] HTTP status: 400
[ERROR] Invalid parameters - check policy configuration
```
**Causa**: Valores de configuración incorrectos o fuera de rango  
**Solución**: Verificar que expected_iops, peak_iops y absolute_min_iops sean valores válidos

#### ERR-203: SVM no existe
```
[ERROR] SVM 'svm_name' not found
```
**Causa**: La SVM especificada no existe en el cluster  
**Solución**: Verificar nombre de SVM con `vserver show`

#### ERR-204: Tipo de asignación inválido
```
[ERROR] Invalid allocation type
```
**Causa**: Valor incorrecto en expected_iops_allocation o peak_iops_allocation  
**Solución**: Usar solo 'allocated_space' o 'used_space'

#### ERR-205: Valores de IOPS inconsistentes
```
[ERROR] Peak IOPS must be greater than or equal to expected IOPS
```
**Causa**: peak_iops menor que expected_iops  
**Solución**: Asegurar que peak_iops >= expected_iops

#### ERR-206: Block size inválido
```
[ERROR] Invalid block size value
```
**Causa**: Valor de block_size no soportado  
**Solución**: Usar valores válidos: ANY, 4k, 8k, 16k, 32k, 64k, 128k

### Errores de Event Logs

#### ERR-601: No se pueden obtener event logs
```
[WARNING] Event logs backup failed (non-critical)
```
**Causa**: Error al consultar API de eventos  
**Solución**: No crítico, verificar permisos de lectura de eventos

### Códigos de Estado HTTP Comunes

- **400 Bad Request**: Parámetros inválidos en la solicitud
- **401 Unauthorized**: Credenciales incorrectas
- **403 Forbidden**: Sin permisos suficientes
- **404 Not Found**: Recurso no existe
- **409 Conflict**: Recurso ya existe o conflicto de estado
- **500 Internal Server Error**: Error interno del servidor ONTAP

## Seguridad

- **IMPORTANTE**: NO compartas el archivo `config.yaml` con credenciales
- Considera usar variables de entorno para credenciales sensibles
- El script desactiva verificación SSL (`verify=False`) - úsalo solo en entornos de desarrollo/pruebas
- Los logs pueden contener información sensible - protege el directorio `logs/`

## Licencia

Este script es para uso interno y educativo.

## Soporte

Para problemas relacionados con la API de NetApp, consulta:
- [Documentación API REST](https://library.netapp.com/ecmdocs/ECMLP3351667/html/)
- [NetApp Community](https://community.netapp.com/)
- [Python Client Library](https://pypi.org/project/netapp-ontap/)

---

**Versión**: 1.0.0  
**Última actualización**: Febrero 2026  
**Compatible con**: ONTAP 9.6+
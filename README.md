# Role: cloudflared_tunnel

Role CRUD (Create, Read, Update, Delete) para gestionar túneles de Cloudflare Tunnel (cloudflared).

Este role permite crear, eliminar, listar y gestionar túneles de Cloudflare de forma idempotente, verificando automáticamente si un túnel existe antes de realizar cualquier acción.

## Requisitos

### Software

- `cloudflared` debe estar instalado y disponible en el PATH del sistema
- Python 3 con soporte para JSON (para parsing de salida de comandos)

### Certificado de Cloudflare

El role requiere un certificado de Cloudflare para autenticarse con la API. Este certificado se obtiene ejecutando:

```bash
cloudflared tunnel login
```

El certificado se guarda por defecto en `~/.cloudflared/cert.pem`, pero puede especificarse una ruta diferente usando la variable `cloudflared_cert_location`.

**Nota:** El role no instala `cloudflared` ni obtiene el certificado. Estas tareas deben realizarse antes de usar el role, típicamente en el playbook que invoca este role.

## Role Variables

| Variable | Default | Description |
| --- | --- | --- |
| `cloudflared_tunnel_name` | *(requerido)* | Nombre del túnel. Requerido para acciones `create`, `delete` y `manage`. Ejemplo: `"my-tunnel"`, `"alerts-tunnel"`. |
| `cloudflared_tunnel_action` | `manage` | Acción a realizar. Valores posibles: `create` (crear túnel), `delete` (eliminar túnel), `list` (listar túneles), `manage` (gestionar según `cloudflared_tunnel_state` - recomendado para idempotencia). |
| `cloudflared_tunnel_state` | `present` | Estado deseado del túnel (solo se usa cuando `cloudflared_tunnel_action: manage`). Valores: `present` (el túnel debe existir, crear si no existe), `absent` (el túnel no debe existir, eliminar si existe). |
| `cloudflared_cert_location` | `~/.cloudflared/cert.pem` | Ruta del certificado de Cloudflare. Si el certificado está en una ubicación no predeterminada, el role automáticamente usará la opción `--origincert` en los comandos de cloudflared. Se expande automáticamente si contiene `~`. Ejemplo: `"/etc/cloudflared/cert.pem"`. |

Refer to `defaults/main.yml` for the complete list.

## Características

### Idempotencia

El role es completamente idempotente:

- **Modo `create`**: Si el túnel ya existe, muestra un mensaje informativo y no realiza cambios
- **Modo `delete`**: Si el túnel no existe, muestra un mensaje informativo y no realiza cambios
- **Modo `manage` con `state: present`**: Crea el túnel solo si no existe
- **Modo `manage` con `state: absent`**: Elimina el túnel solo si existe

### Detección de túneles

El role utiliza una detección mejorada de túneles:

1. **Formato JSON** (preferido): Usa `cloudflared tunnel list --output json` para una detección precisa
2. **Búsqueda de texto** (fallback): Si JSON no está disponible, busca el nombre del túnel en la salida de texto

## Ejemplos de uso

### Crear un túnel (modo directo)

```yaml
- hosts: localhost
  roles:
    - role: cloudflared_tunnel
      vars:
        cloudflared_tunnel_action: create
        cloudflared_tunnel_name: my-tunnel
```

**Nota:** Este modo falla si el túnel ya existe. Para idempotencia, usa `manage` con `state: present`.

### Crear un túnel (modo idempotente - recomendado)

```yaml
- hosts: localhost
  roles:
    - role: cloudflared_tunnel
      vars:
        cloudflared_tunnel_action: manage
        cloudflared_tunnel_name: my-tunnel
        cloudflared_tunnel_state: present
```

### Eliminar un túnel

```yaml
- hosts: localhost
  roles:
    - role: cloudflared_tunnel
      vars:
        cloudflared_tunnel_action: delete
        cloudflared_tunnel_name: my-tunnel
```

O usando `manage`:

```yaml
- hosts: localhost
  roles:
    - role: cloudflared_tunnel
      vars:
        cloudflared_tunnel_action: manage
        cloudflared_tunnel_name: my-tunnel
        cloudflared_tunnel_state: absent
```

### Listar todos los túneles

```yaml
- hosts: localhost
  roles:
    - role: cloudflared_tunnel
      vars:
        cloudflared_tunnel_action: list
```

### Usar certificado en ubicación personalizada

```yaml
- hosts: localhost
  roles:
    - role: cloudflared_tunnel
      vars:
        cloudflared_tunnel_action: manage
        cloudflared_tunnel_name: my-tunnel
        cloudflared_tunnel_state: present
        cloudflared_cert_location: "/etc/cloudflared/cert.pem"
```

### Ejemplo completo en un playbook

```yaml
---
- name: Configurar Cloudflare Tunnel
  hosts: my-server
  vars:
    cloudflared_cert_location: "/etc/cloudflared/cert.pem"
  
  pre_tasks:
    - name: Instalar cloudflared
      # ... tareas de instalación ...
    
    - name: Obtener certificado
      # ... tareas para obtener certificado ...
  
  roles:
    - role: cloudflared_tunnel
      vars:
        cloudflared_tunnel_action: manage
        cloudflared_tunnel_name: "{{ cfd_tunnel_name }}"
        cloudflared_tunnel_state: present
        cloudflared_cert_location: "{{ cloudflared_cert_location }}"
```

## Comportamiento detallado

### Modo `create`

1. Verifica si el túnel existe
2. Si existe: Muestra mensaje informativo y termina (idempotente)
3. Si no existe: Crea el túnel y marca como `changed`

### Modo `delete`

1. Verifica si el túnel existe
2. Si existe: Elimina el túnel y marca como `changed`
3. Si no existe: Muestra mensaje informativo y termina (idempotente)

### Modo `list`

1. Lista todos los túneles disponibles
2. Muestra la salida en formato JSON o texto según disponibilidad
3. No realiza cambios en el sistema

### Modo `manage`

1. Si `state: present`:
   - Verifica si el túnel existe
   - Si existe: No hace nada (idempotente)
   - Si no existe: Crea el túnel
2. Si `state: absent`:
   - Verifica si el túnel existe
   - Si existe: Elimina el túnel
   - Si no existe: No hace nada (idempotente)

## Notas importantes

1. **El role no instala prerequisitos**: El playbook debe instalar `cloudflared` antes de usar este role
2. **El role no obtiene certificados**: El certificado debe obtenerse previamente (típicamente con `cloudflared tunnel login`)
3. **Uso de `--origincert`**: El role usa `--origincert` (no `--credentials-file`) cuando el certificado está en una ubicación no predeterminada
4. **Detección automática**: El role detecta automáticamente si el certificado está en la ubicación por defecto y ajusta los comandos accordingly
5. **Idempotencia garantizada**: El role puede ejecutarse múltiples veces sin efectos secundarios

## Troubleshooting

### Error: "cloudflared: command not found"

Asegúrate de que `cloudflared` esté instalado y disponible en el PATH antes de usar el role.

### Error: "certificate file not found"

Verifica que el certificado exista en la ruta especificada en `cloudflared_cert_location`. El certificado se obtiene ejecutando `cloudflared tunnel login`.

### El túnel no se detecta correctamente

El role usa formato JSON para detectar túneles. Si hay problemas, verifica que `cloudflared tunnel list --output json` funcione correctamente en el sistema.

## Licencia

Este role sigue la misma licencia que el proyecto principal.

## Autor

Mantenido como parte del proyecto INFRAFORGE.

# Automatizacion-Python---MikroTik
---

## 📝 Descripción Detallada del Flujo

### Fase 1: Aprovisionamiento de la Infraestructura (Configuración Única)

Antes de ejecutar la automatización, se deben establecer los cimientos de seguridad en el Router y en el Servidor de Automatización:

* **A. Preparación de MikroTik RouterOS:**
  1. **Aislamiento de Privilegios:** Se crea un usuario dedicado (`ej: ips_bot`) asignado a un grupo personalizado con permisos restringidos únicamente para `api` o `ssh` y lectura/escritura en `test` y `write` (Firewall). Se evita estrictamente el uso del usuario administrador por defecto.
  2. **Endurecimiento de Servicios:** Se habilitan los servicios correspondientes (`/ip service enable api` o `ssh`) y se restringe el acceso mediante `address-list` para que únicamente la IP del servidor de Python pueda negociar la conexión.
  3. **Configuración de Logs Orientados a Eventos:** Se genera una regla en el firewall para capturar tráfico anómalo con la acción `log` y el prefijo específico `"IPS_POTENTIAL"`.

* **B. Preparación del Servidor Python:**
  1. **Gestión de Dependencias:** Instalación de librerías para la comunicación con RouterOS (`routeros-api` o `paramiko` para SSH).
  2. **Control de Falsos Positivos:** Creación de un archivo estático de Lista Blanca (`White-list`) que aloja las direcciones IP críticas y de gestión que jamás deben ser bloqueadas por el script.

---

### Fase 2: Ciclo Continuo de Detección y Mitigación (Tiempo Real)

Una vez aprovisionado, el script de Python entra en un bucle infinito de monitoreo:

1. **Escucha y Captura:** Python actúa como receptor Syslog o lee continuamente las salidas de log generadas por el MikroTik.
2. **Análisis de Paquetes y Expresiones Regulares (Parsing):** Al identificar el prefijo `"IPS_POTENTIAL"`, el script procesa la cadena de texto y extrae las variables críticas: `src-ip` (IP origen), `dst-ip` (IP destino) y puerto.
3. **Motor de Decisiones (Evaluación de Umbrales):** Se evalúa la gravedad de la alerta. Para evitar falsos positivos, se aplica un umbral cuantitativo (ej: *Si una IP origen repite la alerta más de 10 veces en 1 minuto, se clasifica como maliciosa*).
4. **Filtro de Lista Blanca:** Python intercepta la IP atacante y verifica que no pertenezca a la lista de exclusión antes de proceder.
5. **Generación Dinámica de Comandos:** En lugar de saturar el procesador del router creando reglas de filtrado individuales por cada atacante, Python diseña una instrucción optimizada utilizando listas de direcciones dinámicas (`address-list`).
6. **Conexión Segura y Ejecución:** Se establece una sesión SSH/API cifrada, se inyecta el comando y se cierra la sesión inmediatamente para liberar recursos en el MikroTik.

---

## 🔒 Implementación de Reglas de Firewall en MikroTik

Para que la automatización sea de alto rendimiento, el firewall de MikroTik se configura con una **regla estática permanente**, mientras que Python se encarga de alimentar la **lista dinámica de bloqueo**.

### 1. Regla Estática en MikroTik (Configurada previamente en el Router)
Esta regla procesa el tráfico de forma óptima en las primeras capas del firewall (Filter o Raw) evaluando si la IP origen pertenece a la lista negra:

```routeros
/ip firewall filter
add chain=forward action=drop src-address-list=IPS_BLACKLIST comment="IPS: Bloqueo Automatizado"

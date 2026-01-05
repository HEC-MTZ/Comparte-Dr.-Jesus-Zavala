# **Reporte 03: Informe Técnico: Configuración de Acceso RDP Seguro mediante Túnel SSH en Fedora**

**Dirigido a**: Personal técnico y administrativo de la Universidad Autónoma Metropolitana  
**Elaborado por**: Dr. Jesús Zavala Ruiz (Coordinador de Sistemas Escolares UAMI)  
**Colaboración**: Qwen.ai  
**Compatible con**: Fedora 41, 42 y 43  
**Versión**: 1.0
**Elaboración**: 18 de diciembre de 2025  
**Última actualización**: 18 de diciembre de 2025  

---

## **Introducción**

Este manual tiene como objetivo guiar paso a paso en la configuración de un acceso remoto seguro al servidor Windows que aloja el cliente del Sistema de Administración Escolar (SAE), desarrollado con Uniface.

El método empleado utiliza un túnel SSH a través de un servidor intermedio (`xanum.uam.mx`), garantizando que toda la comunicación esté cifrada y no expuesta directamente a la red.

> **Importante**: este procedimiento es esencial en entornos institucionales donde la seguridad y la trazabilidad son prioritarias.


## **1. Requisitos**

### **1.1. Conocimientos previos**
- Manejo básico de la terminal en Linux.
- Comprensión de conceptos de red (puertos, IP, túneles).
- Acceso autorizado al servidor `xanum.uam.mx`.

### **1.2. Software necesario**
Asegúrate de tener instalados los siguientes paquetes en tu sistema Fedora:

```bash
sudo dnf install remmina freerdp openssh-clients
```

Verifica la instalación:
```bash
rpm -q remmina freerdp openssh-clients
```

Debes ver versiones instaladas, por ejemplo:
```
remmina-1.4.29-3.fc41.x86_64
freerdp-3.9.1-1.fc41.x86_64
openssh-clients-9.8p1-3.fc41.x86_64
```

### **1.3. Credenciales necesarias**
- **Servidor salto**: `xanum.uam.mx`
  - Usuario: `ishak`
  - Llave privada SSH (archivo `id_rsa` en `~/.ssh/`)
- **Servidor Windows (SAE)**:
  - Usuario: `icse`
  - Dominio: `SUAMIZT`
  - Contraseña: `Pa$$word` *(valor ficticio para este manual)*


## **2. Preparación de la llave SSH**

La **llave privada** permite la conexión segura mediante SSH sin usar contraseña creando un túnel del equipo local al equipo destino, a través del servidor de correo xanum.uam.mx con el usuario `ishak`.

### **2.1. Renombrar la llave privada (buena práctica)**

Es recomendable no usar la llave predeterminada (`id_rsa`) para múltiples propósitos. En su lugar, crea una copia con un nombre descriptivo como `id_rsa_sae`:

```bash
# Navega al directorio SSH
cd ~/.ssh

# Crea una copia con nombre específico
cp id_rsa id_rsa_sae

# Asegura los permisos
chmod 600 id_rsa_sae
```

> **Nota**: este enfoque facilita la administración, evita confusiones y permite revocar acceso a un servicio sin afectar otros.


## **3. Configuración del cliente SSH**

### **3.1. Editar el archivo de configuración**

Abre o crea el archivo `~/.ssh/config`:

```bash
vim ~/.ssh/config
```

Agrega el siguiente bloque:

```
# Túnel RDP a través de xanum.uam.mx, reconocido como sae
Host sae
    HostName xanum.uam.mx
    User ishak
    IdentityFile ~/.ssh/id_rsa_sae
    IdentitiesOnly yes
    LocalForward 8000 148.206.52.17:3389
    RequestTTY no
    ExitOnForwardFailure yes
```

### **3.2. Establecer permisos correctos**

```bash
chmod 644 ~/.ssh/config
```

> **Advertencia**: si los permisos son demasiado permisivos, SSH rechazará usar el archivo.


## **4. Establecer el túnel SSH**

El túnel SSH es un mecanismo para desplegar de manera segura, en el entorno local, la interfase remota, en este caso, la GUI del equipo Windows 11.

### **4.1. Iniciar el túnel**

Ejecuta el siguiente comando en una terminal:

```bash
ssh -fN sae
```

- `-f`: ejecuta en segundo plano.
- `-N`: no ejecuta comandos remotos (solo crea el túnel).

### **4.2. Primera conexión: verificación de seguridad**

La primera vez que te conectes, verás un mensaje como este:

```
The authenticity of host 'xanum.uam.mx (148.206.32.5)' can't be established.
ED25519 key fingerprint is SHA256:w/T0dA9fNl3I+iWeCxu/JqxbdKf1RaT+Qp+JTx9NqL8.
Are you sure you want to continue connecting (yes/no/[fingerprint])? 
```

Escribe `yes` y presiona **Enter**.  
Este paso solo ocurre una vez. La huella se guarda en `~/.ssh/known_hosts`.


## **5. Verificar que el túnel funciona**

### **5.1. Comprobar el puerto local**

El túnel redirige el puerto `8000` de tu máquina al puerto `3389` del servidor Windows.

```bash
ss -tuln | grep 8000
```

**Salida esperada:**
```
tcp   LISTEN 0      128    127.0.0.1:8000    0.0.0.0:*
tcp   LISTEN 0      128        [::1]:8000       [::]:*
```

Si ves esta salida, el túnel está activo.


## **6. Conectarse al servidor Windows**

Tienes dos opciones: interfaz gráfica (Remmina) o línea de comandos (`xfreerdp`).

### **6.1. Opción A: usar Remmina (GUI recomendada para usuarios)**

1. Abre Remmina desde el menú de aplicaciones.
2. Crea una nueva conexión:
   - **Protocolo**: `RDP - Remote Desktop Protocol`
   - **Nombre del servidor**: `localhost`
   - **Puerto**: `8000`
   - **Nombre de usuario**: `icse`
   - **Dominio**: `SUAMIZT`
   - **Contraseña**: `Pa$$word` (ficticia)
3. Guarda la conexión con un nombre como `SAE-Windows-Tunnel`.
4. Haz doble clic en la conexión para iniciar sesión.

Verás el escritorio del servidor Windows, donde podrás usar el cliente Uniface del SAE.

### **6.2. Opción B: usar xfreerdp (para diagnóstico o scripts)**

Ejecuta en la terminal:

```bash
xfreerdp /v:localhost:8000 /u:icse /d:SUAMIZT
```

> **Importante**: no incluyas la contraseña (`/p`) en el comando.  
> El sistema te la pedirá de forma segura en una ventana emergente.

**Nota sobre advertencias**:  
Es normal ver mensajes como:
- `Certificate verification failure`: el servidor Windows usa un certificado autofirmado.
- `Cannot find KDC for realm "SUAMIZT"`: tu máquina Linux no está unida al dominio (pero la autenticación NTLM funciona).

## **7. Cerrar la conexión**

### **7.1. Cerrar sesión en Windows**
- Cierra la sesión desde el menú de inicio de Windows o desde Remmina/xfreerdp.

### **7.2. Detener el túnel SSH**

El túnel sigue activo incluso si cierras la sesión RDP. Para detenerlo:

```bash
# Método recomendado (seguro y rápido)
pkill -f "ssh.*sae"
```

> **Recomendación**: este método evita tener que buscar manualmente el número de proceso (PID). Es más eficiente y menos propenso a errores.

### **7.3. Verificar que el túnel se cerró**

```bash
ss -tuln | grep 8000
```

No debe haber salida. Si ves líneas, el túnel sigue activo.


## **8. Resolución de problemas comunes**

| Síntoma | Causa probable | Solución |
|--------|----------------|----------|
| `Permission denied (publickey)` | Llave SSH no autorizada | Verifica que la llave pública esté en `~/.ssh/authorized_keys` en `xanum.uam.mx` |
| `Connection refused` en puerto 8000 | Túnel no iniciado | Ejecuta `ssh -fN sae` nuevamente |
| Pantalla negra en RDP | Problema de resolución | En Remmina, ajusta la resolución a `1920x1080` o `Full-screen` |
| Advertencia de certificado | Certificado autofirmado | Normal en Windows; acepta la conexión |
| `No such process` al usar `kill` | Estás intentando matar el proceso `grep` | Usa `pkill -f "ssh.*sae"` en su lugar |


## **9. Buenas prácticas de seguridad**

1. Nunca almacenes contraseñas en archivos de texto plano.
2. Usa claves SSH con nombres descriptivos (ej. `id_rsa_sae`, `id_rsa_cse`).
3. Cierra siempre los túneles SSH cuando ya no los necesites.
4. Mantén tu sistema actualizado:
   ```bash
   sudo dnf update
   ```
5. No compartas tu llave privada con nadie.


## **10. Resumen de comandos útiles**

| Acción | Comando |
|-------|--------|
| Preparar llave | `cp ~/.ssh/id_rsa ~/.ssh/id_rsa_sae && chmod 600 ~/.ssh/id_rsa_sae` |
| Iniciar túnel | `ssh -fN sae` |
| Verificar túnel | `ss -tuln \| grep 8000` |
| Conectar con Remmina | Abrir → `SAE-Windows-Tunnel` |
| Conectar con xfreerdp | `xfreerdp /v:localhost:8000 /u:icse /d:SUAMIZT` |
| Detener túnel | `pkill -f "ssh.*sae"` |


## **Conclusión**

Con este manual, podrás acceder de forma segura, cifrada y confiable al servidor Windows del SAE desde cualquier equipo con Fedora.

El uso de túneles SSH no solo cumple con los estándares de seguridad institucional, sino que también protege la integridad de los datos sensibles de la universidad.
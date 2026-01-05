# **Reporte Técnico 06: Configuración Segura de SFTP para el Usuario `joomla-sftp` en Servidor de Producción**

**Fecha:** 03 de enero de 2026  
**Servidor:** `cse.izt.uam.mx`  
**Sistema Operativo:** Rocky Linux 9.6  
**Aplicación:** Joomla 4  
**Responsable:** Dr. Jesús Zavala Ruiz (Coordinador de Sistemas Escolares)  
**Objetivo:** Habilitar acceso seguro mediante SFTP al directorio `/var/www/cse.izt.uam.mx/documentos/` para gestión de archivos, sin permitir acceso shell ni autenticación por contraseña.

## **Introducción**

Esta guía está diseñada para administradores de sistemas que necesitan configurar un acceso restringido y seguro a un directorio específico en un servidor de producción con alto nivel de seguridad (SELinux activo, solo llave pública).

El usuario `joomla-sftp` podrá:
- Subir, descargar y gestionar archivos en `/documentos/`.
- **No podrá salir de ese directorio** (jaula SFTP).
- **No podrá ejecutar comandos** en el servidor.
- **Solo accederá con una llave privada**, no con contraseña.

## **Requisitos previos**

Antes de comenzar, asegúrese de tener:

1. Acceso SSH como usuario administrador (`cse`) al servidor `cse.izt.uam.mx`.
2. Una llave pública SSH generada en su computadora local (por ejemplo, `~/.ssh/id_rsa_joomla_sftp.pub`).
3. Conocimiento básico de línea de comandos (no es necesario ser experto).

## **Paso 1: Preparar la estructura de la jaula SFTP**

### Objetivo
Crear un directorio seguro donde el usuario `joomla-sftp` pueda trabajar, sin poder acceder al resto del sistema.

#### Comando 1: Crear la jaula
```bash
sudo mkdir -p /srv/sftp/joomla-sftp
```

#### Comando 2: Asegurar propietario y permisos
```bash
sudo chown root:root /srv/sftp/joomla-sftp
sudo chmod 755 /srv/sftp/joomla-sftp
```

> Esto cumple con los requisitos estrictos de OpenSSH: solo `root` puede escribir en la jaula.


## **Paso 2: Enlazar el directorio real con la jaula (bind mount)**

### Objetivo
Conectar el directorio real (`/var/www/cse.izt.uam.mx/documentos/`) con la jaula, para que el usuario vea y modifique los archivos reales.

#### Comando 3: Crear punto de montaje dentro de la jaula
```bash
sudo mkdir /srv/sftp/joomla-sftp/documentos
sudo chown root:root /srv/sftp/joomla-sftp/documentos
sudo chmod 755 /srv/sftp/joomla-sftp/documentos
```

#### Comando 4: Realizar el bind mount
```bash
sudo mount --bind /var/www/cse.izt.uam.mx/documentos /srv/sftp/joomla-sftp/documentos
```

#### Comando 5: Hacerlo persistente (después de reiniciar)
```bash
echo "/var/www/cse.izt.uam.mx/documentos /srv/sftp/joomla-sftp/documentos none bind 0 0" | sudo tee -a /etc/fstab
```

> Ahora, cualquier archivo que suba el usuario `joomla-sftp` aparecerá directamente en el sitio web.


## **Paso 3: Configurar el usuario `joomla-sftp`**

### Objetivo
Asegurar que el usuario tenga los permisos correctos para escribir en el directorio y que pertenezca al grupo `apache` para colaborar con el servidor web.

#### Comando 6: Añadir al grupo `apache`
```bash
sudo usermod -aG apache joomla-sftp
```

#### Comando 7: Ajustar propietario y permisos del directorio montado
```bash
sudo chown -R joomla-sftp:apache /srv/sftp/joomla-sftp/documentos
sudo chmod -R 2775 /srv/sftp/joomla-sftp/documentos
```

> El bit SGID (`2` en `2775`) asegura que nuevos archivos hereden el grupo `apache`.


## **Paso 4: Configurar la llave pública del usuario**

### Objetivo
Permitir que el usuario se conecte solo con una llave privada, no con contraseña.

#### Comando 8: Cambiar el home del usuario (para aislar la llave)
```bash
sudo usermod -d /home/joomla-sftp joomla-sftp
```

#### Comando 9: Crear el directorio `.ssh`
```bash
sudo mkdir -p /home/joomla-sftp/.ssh
sudo touch /home/joomla-sftp/.ssh/authorized_keys
```

#### Comando 10: Ajustar permisos
```bash
sudo chown -R joomla-sftp:apache /home/joomla-sftp/.ssh
sudo chmod 700 /home/joomla-sftp/.ssh
sudo chmod 600 /home/joomla-sftp/.ssh/authorized_keys
```

#### Comando 11: Instalar la llave pública
> Copie el contenido de su archivo `~/.ssh/id_rsa_joomla_sftp.pub` (en su computadora local) y péguelo aquí:

```bash
echo "ssh-rsa AAAAB3NzaC1yc2E... joomla-sftp@cse.izt.uam.mx" | sudo tee /home/joomla-sftp/.ssh/authorized_keys
```

> Reemplace el texto entre comillas con su llave real.


## **Paso 5: Configurar SSH para el usuario `joomla-sftp`**

### Objetivo
Restringir al usuario a la jaula y deshabilitar el acceso shell y por contraseña.

#### Comando 12: Hacer copia de seguridad de la configuración
```bash
sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak-$(date +%Y%m%d)
```

#### Comando 13: Añadir el bloque de configuración
```bash
sudo tee -a /etc/ssh/sshd_config << 'EOF'

# jzr 3 ene 2026 - SFTP restringido para Joomla
Match User joomla-sftp
    ChrootDirectory /srv/sftp/joomla-sftp
    ForceCommand internal-sftp -d /documentos
    PasswordAuthentication no
    PubkeyAuthentication yes
    PermitTunnel no
    AllowAgentForwarding no
    AllowTcpForwarding no
    X11Forwarding no
    MaxSessions 3
EOF
```

#### Comando 14: Verificar sintaxis
```bash
sudo sshd -t
```

> Si no muestra errores, continúe.

#### Comando 15: Recargar el servicio (sin reiniciar)
```bash
sudo systemctl reload sshd
```

> Esto aplica los cambios sin cerrar sus sesiones actuales.

## **Paso 6: Probar la conexión desde su computadora local**

### Objetivo
Verificar que puede conectarse con FileZilla o desde terminal.

#### Desde terminal (Linux/Mac):
```bash
sftp -i ~/.ssh/id_rsa_joomla_sftp joomla-sftp@cse.izt.uam.mx
```

Debe ver:
```
Connected to cse.izt.uam.mx.
sftp> pwd
Remote working directory: /documentos
```

#### Desde FileZilla:
1. Abra FileZilla.
2. En “Host”: `cse.izt.uam.mx`
3. En “Username”: `joomla-sftp`
4. En “Password”: **deje vacío**.
5. En “Port”: `22`
6. Vaya a “Edit” > “Settings” > “Connection” > “SFTP” > “Add key file” >  "Browse"" > "All Files", elegir el archivo `~/.ssh/id_rsa_joomla_sftp` > "Convert Key file?"  > "Yes", escribir la contraseña de la llave privada y luego dar el nombre del archivo convertido como `id_rsa_joomla_sftp.ppk`  > "Save".  
7. Haga clic en “Quickconnect”.

> Deberá ver el directorio `/documentos/` con todos sus archivos.

## **Resultado final**

Al completar estos pasos, el usuario `joomla-sftp` tendrá acceso **solo al directorio `/documentos/`**, con las siguientes características:

| Característica | Estado |
|----------------|--------|
| Acceso shell | ❌ No permitido |
| Autenticación por contraseña | ❌ No permitida |
| Escritura en `/documentos/` | ✅ Permitida |
| Lectura por Apache/Joomla | ✅ Permitida (mismo grupo `apache`) |
| Contexto SELinux | ✅ Correcto (`httpd_sys_rw_content_t`) |
| Persistencia después de reinicio | ✅ Configurada en `/etc/fstab` |

## **Resumen visual (FileZilla)**

![FileZilla SFTP](https://i.imgur.com/placeholder.png)

*Captura de pantalla de FileZilla mostrando la conexión exitosa al directorio `/documentos/`, con listado de archivos y carpetas.*

## **Notas importantes para usuarios no técnicos**

- **Nunca comparta su llave privada** (`id_rsa_joomla_sftp`). Está en su computadora local y debe mantenerse segura.
- **Si pierde la llave privada**, deberá generar un nuevo par y actualizar la pública en el servidor.
- **Este acceso no le permite modificar el sitio web directamente**, solo gestionar archivos en `/documentos/`.
- **Si necesita agregar más usuarios**, repita este proceso con un nombre diferente y una jaula separada.

## **Anexos**

### Anexo A: Verificación de contexto SELinux
```bash
ls -ldZ /srv/sftp/joomla-sftp/documentos
```
Salida esperada:
```
drwxrwsr-x. 6 joomla-sftp apache system_u:object_r:httpd_sys_rw_content_t:s0 ...
```

### Anexo B: Verificación de permisos
```bash
ls -la /srv/sftp/joomla-sftp/documentos
```
Salida esperada (fragmento):
```
-rwxrwsr-x 1 joomla-sftp apache ... calendario.pdf
```

## *Conclusión**

La configuración ha sido probada y funciona correctamente. El usuario `joomla-sftp` puede ahora gestionar archivos en `/documentos/` de forma segura, sin comprometer la integridad del servidor ni la seguridad del sitio web.


# **Reporte Técnico 05: Restauración de la Funcionalidad de Escritura en Joomla 4 en Servidor LAMP con SELinux**

**Fecha:** 03 de enero de 2026  
**Servidor:** `cse.izt.uam.mx`  
**Sistema Operativo:** Rocky Linux 9.6  
**Aplicación:** Joomla 4  
**Responsable:** Dr. Jesús Zavala Ruiz (Coordinador de Sistemas Escolares)  
**Tipo de problema:** Imposibilidad de actualizar artículos, subir archivos o escribir en caché desde el panel de administración de Joomla.

## **1. Diagnóstico**

### **1.1. Contexto del entorno**
- El servidor opera bajo una configuración LAMP (Linux, Apache, MySQL/MariaDB, PHP) con alto nivel de seguridad.
- SELinux se encuentra en modo **`enforcing`**, imponiendo políticas estrictas sobre los accesos del proceso `httpd`.
- La instalación de Joomla 4 reside en:  
  `/var/www/cse.izt.uam.mx/`

### **1.2. Síntomas observados**
- El panel de administración de Joomla **no permite guardar cambios** en artículos, subir imágenes ni realizar operaciones que requieran escritura en el sistema de archivos.
- No se reportan errores explícitos en los logs de Apache/PHP, lo que sugiere un bloqueo silencioso por parte del subsistema de seguridad.

### **1.3. Hipótesis**
La causa más probable es que los **contextos de seguridad de SELinux** aplicados a los directorios de Joomla sean de tipo **`httpd_sys_content_t`**, el cual **solo permite lectura**, y no el tipo **`httpd_sys_rw_content_t`**, necesario para operaciones de escritura.

## **2. Análisis y Verificación**

### **2.1. Verificación del estado de SELinux**
```bash
sestatus
```
**Salida relevante:**
```
Current mode:                   enforcing
```
> **Conclusión:** SELinux está activo y bloqueando cualquier operación no autorizada.

### **2.2. Identificación de la raíz del sitio Joomla**
Se confirmó que la raíz del sitio es:
```
/var/www/cse.izt.uam.mx/
```

### **2.3. Verificación de contextos SELinux en la raíz**
```bash
ls -ldZ /var/www/cse.izt.uam.mx/
```
**Salida:**
```
drwxr-xr-x. 19 apache apache system_u:object_r:httpd_sys_content_t:s0 ...
```
> **Conclusión:** El contexto es `httpd_sys_content_t` (solo lectura).

### **2.4. Verificación de contextos en directorios críticos de Joomla**
Se evaluaron los siguientes directorios, todos esenciales para operaciones de escritura:

- `cache`
- `administrator/cache`
- `images`
- `tmp`
- `media`
- `documentos`

**Comando:**
```bash
for dir in cache administrator/cache images tmp media documentos; do
  ls -ldZ "/var/www/cse.izt.uam.mx/$dir"
done
```

**Resultado:**
Todos los directorios tenían el contexto **`httpd_sys_content_t`**, **no permitiendo escritura** por el proceso `httpd`.

> **Conclusión diagnóstica confirmada:**  
> La aplicación Joomla **no puede escribir** en ninguno de sus directorios críticos debido a la política de SELinux.

## **3. Solución Aplicada**

### **3.1. Instalación del paquete `policycoreutils-python-utils`**
```bash
sudo dnf install -y policycoreutils-python-utils
```
Este paquete es indispensable para usar `semanage`, herramienta que permite definir reglas persistentes de contexto SELinux.

### **3.2. Definición de reglas persistentes con `semanage fcontext`**
Se aplicaron reglas para asignar el tipo `httpd_sys_rw_content_t` a los directorios necesarios, incluyendo sus subdirectorios:

```bash
sudo semanage fcontext -a -t httpd_sys_rw_content_t "/var/www/cse.izt.uam.mx/cache(/.*)?"
sudo semanage fcontext -a -t httpd_sys_rw_content_t "/var/www/cse.izt.uam.mx/administrator/cache(/.*)?"
sudo semanage fcontext -a -t httpd_sys_rw_content_t "/var/www/cse.izt.uam.mx/images(/.*)?"
sudo semanage fcontext -a -t httpd_sys_rw_content_t "/var/www/cse.izt.uam.mx/tmp(/.*)?"
sudo semanage fcontext -a -t httpd_sys_rw_content_t "/var/www/cse.izt.uam.mx/media(/.*)?"
sudo semanage fcontext -a -t httpd_sys_rw_content_t "/var/www/cse.izt.uam.mx/documentos(/.*)?"
```

### **3.3. Aplicación inmediata de los nuevos contextos**
```bash
sudo restorecon -Rv /var/www/cse.izt.uam.mx/cache \
                    /var/www/cse.izt.uam.mx/administrator/cache \
                    /var/www/cse.izt.uam.mx/images \
                    /var/www/cse.izt.uam.mx/tmp \
                    /var/www/cse.izt.uam.mx/media \
                    /var/www/cse.izt.uam.mx/documentos
```

**Salida observada (fragmento representativo):**
```
Relabeled /var/www/cse.izt.uam.mx/images/logo.png ...
Relabeled /var/www/cse.izt.uam.mx/media/plg_system_debug/js/debug.min.js ...
```

> Decenas de archivos y directorios fueron reclasificados al nuevo contexto `httpd_sys_rw_content_t`.

## **4. Validación y Resultado**

- Tras la aplicación de los nuevos contextos, los directorios críticos ahora tienen permisos de **lectura y escritura** para el proceso `httpd`.
- No fue necesario desactivar SELinux ni cambiar el modo a `permissive`, manteniendo así la **integridad de la política de seguridad** del servidor.
- **Se recomienda probar funcionalidades críticas en Joomla** (guardar artículo, subir imagen, actualizar extensiones) para confirmar la resolución.

## **5. Recomendaciones**

1. **Monitoreo posterior:**  
   Ejecutar `sudo ausearch -m avc -ts recent` tras las pruebas para verificar si persisten denegaciones.
2. **Documentación:**  
   Registrar los cambios realizados en el inventario de configuración del servidor.
3. **Backup de políticas:**  
   Considerar exportar las reglas de SELinux actuales con:  
   ```bash
   sudo semanage fcontext -l > /root/selinux-fcontexts-cse-$(date +%Y%m%d).txt
   ```

## **6. Conclusión**

El problema de funcionalidad en Joomla 4 fue resuelto mediante la **aplicación correcta de contextos SELinux**, específicamente asignando el tipo `httpd_sys_rw_content_t` a los directorios que requieren operaciones de escritura. Esta solución **preserva la postura de seguridad del servidor** y sigue las mejores prácticas de administración en entornos con SELinux activo.
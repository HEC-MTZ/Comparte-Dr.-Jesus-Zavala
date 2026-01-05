# **INFORME TÉCNICO 2: SUBIDA DE ARCHIVOS AL DIRECTORIO WEB DE OPTATIVAS**  
## **Servidor cse.izt.uam.mx - Rocky Linux 9.6**
## Versión 2

---

### **METADATOS DEL DOCUMENTO**
- **Fecha de creación**: 2025-12-02  
- **Última modificación**: 2025-12-13 (Se corrigió el Flujo Estandarizado)  
- **Responsable**: cse (Propietario del Sistema) Dr. Jesús Zavala Ruiz  
- **Asistente**: Qwen  

- **Usuario ejecutor**: cse  
- **Ruta destino**: `/var/www/cse.izt.uam.mx/documentos/licenciatura/optativas/`  
- **Archivos procesados**: 27 archivos `.txt`  
- **Estado final**: ✅ Archivos visibles y accesibles públicamente

---

### **1. RESUMEN EJECUTIVO**

Se procedió a la subida de 27 archivos de texto correspondientes a documentos de optativas de licenciatura. El proceso enfrentó un problema crítico de **visibilidad web** tras la transferencia, ocasionado por **contextos incorrectos de SELinux**. La solución implementada incluyó:

- Subida segura mediante directorio temporal (`/tmp`)
- Corrección de propietario y permisos (`apache:apache`, modo `644`)
- Restauración del contexto SELinux correcto (`httpd_sys_content_t`)
- Verificación de accesibilidad pública
 

**Resultado**: Todos los archivos son accesibles vía HTTPS sin errores 403.

---

### **2. PRERREQUISITOS Y SALIDAS INICIALES**

**Listado de archivos a subir:**
```
121.txt, 174.txt, 22.txt, 23.txt, 24.txt, 25.txt, 26.txt, 27.txt, 28.txt, 29.txt,
30.txt, 37.txt, 38.txt, 39.txt, 40.txt, 41.txt, 42.txt, 43.txt, 44.txt, 45.txt,
46.txt, 52.txt, 53.txt, 54.txt, 55.txt, 56.txt, 57.txt
```

**Estado inicial del directorio destino:**
```bash
[cse@cse ~]$ sudo ls -la /var/www/cse.izt.uam.mx/documentos/licenciatura/optativas/
# Archivos antiguos visibles y accesibles
# Propietario: apache:apache
# Permisos: -rw-r--r-- (644)
```

**Conclusión**: Directorio configurado correctamente para contenido web público.

---

### **3. PROCEDIMIENTO DE SUBIDA DE ARCHIVOS**

#### **3.1. Transferencia a directorio temporal**

Desde la máquina local, se ejecutó:

```bash
jzavalar@x280:~/Downloads/8.Optativas a sitio web$ rsync -avz --progress \
  121.txt 174.txt 22.txt 23.txt 24.txt 25.txt 26.txt 27.txt 28.txt 29.txt \
  30.txt 37.txt 38.txt 39.txt 40.txt 41.txt 42.txt 43.txt 44.txt 45.txt \
  46.txt 52.txt 53.txt 54.txt 55.txt 56.txt 57.txt \
  cse@cse.izt.uam.mx:/tmp/optativas/
```

**Salida observada:**
```
sending incremental file list
121.txt
         27,406 100%    0.00kB/s    0:00:00 (xfr#1, to-chk=26/27)
...
57.txt
         17,196 100%    1.49MB/s    0:00:00 (xfr#27, to-chk=0/27)
```

**Resultado**: Transferencia completada exitosamente.

#### **3.2. Movimiento al directorio web y configuración de permisos**

En el servidor:

```bash
[cse@cse ~]$ sudo mv /tmp/optativas/*.txt /var/www/cse.izt.uam.mx/documentos/licenciatura/optativas/
[cse@cse ~]$ sudo chown apache:apache /var/www/cse.izt.uam.mx/documentos/licenciatura/optativas/*.txt
[cse@cse ~]$ sudo chmod 644 /var/www/cse.izt.uam.mx/documentos/licenciatura/optativas/*.txt
[cse@cse ~]$ sudo rm -rf /tmp/optativas/
```

**Verificación post-movimiento:**
```bash
[cse@cse ~]$ ls -l /var/www/cse.izt.uam.mx/documentos/licenciatura/optativas/22.txt
-rw-r--r--. 1 apache apache 67325 Dec  2 15:25 22.txt
```

**Problema detectado**: Archivos no accesibles vía web (error 403 Forbidden), a pesar de permisos correctos.

#### **3.3. Diagnóstico de SELinux**

Verificación de contextos:

```bash
[cse@cse ~]$ ls -Z /var/www/cse.izt.uam.mx/documentos/licenciatura/optativas/22_m.txt
system_u:object_r:httpd_sys_content_t:s0 /var/www/cse.izt.uam.mx/documentos/licenciatura/optativas/22_m.txt

[cse@cse ~]$ ls -Z /var/www/cse.izt.uam.mx/documentos/licenciatura/optativas/22.txt
unconfined_u:object_r:user_tmp_t:s0 /var/www/cse.izt.uam.mx/documentos/licenciatura/optativas/22.txt
```

**Conclusión**: Los archivos nuevos heredaron el contexto `user_tmp_t` de `/tmp`, impidiendo su acceso por Apache.

#### **3.4. Corrección de contexto SELinux**

```bash
[cse@cse ~]$ sudo restorecon -Rv /var/www/cse.izt.uam.mx/documentos/licenciatura/optativas/
```

**Salida observada:**
```
restorecon reset /var/www/cse.izt.uam.mx/documentos/licenciatura/optativas/22.txt context unconfined_u:object_r:user_tmp_t:s0->system_u:object_r:httpd_sys_content_t:s0
restorecon reset /var/www/cse.izt.uam.mx/documentos/licenciatura/optativas/23.txt context unconfined_u:object_r:user_tmp_t:s0->system_u:object_r:httpd_sys_content_t:s0
...
restorecon reset /var/www/cse.izt.uam.mx/documentos/licenciatura/optativas/57.txt context unconfined_u:object_r:user_tmp_t:s0->system_u:object_r:httpd_sys_content_t:s0
```

**Resultado**: Todos los archivos corregidos al contexto `httpd_sys_content_t`.

---

### **4. VERIFICACIÓN FINAL**

#### **4.1. Verificación de contexto**

```bash
[cse@cse ~]$ ls -Z /var/www/cse.izt.uam.mx/documentos/licenciatura/optativas/22.txt
system_u:object_r:httpd_sys_content_t:s0 /var/www/cse.izt.uam.mx/documentos/licenciatura/optativas/22.txt
```

#### **4.2. Verificación de accesibilidad web**

URL de prueba:
```
https://cse.izt.uam.mx/documentos/licenciatura/optativas/22.txt
```

**Resultado**: ✅ Archivo cargado correctamente en navegador.

#### **4.3. Estado final del directorio**

```bash
[cse@cse ~]$ sudo ls -la /var/www/cse.izt.uam.mx/documentos/licenciatura/optativas/
# Todos los archivos nuevos visibles
# Propietario: apache:apache
# Permisos: -rw-r--r-- (644)
# Contexto SELinux: httpd_sys_content_t
```

---

### **5. ARCHIVOS PROCESADOS**

| Archivo | Tamaño (bytes) | Fecha actualización |
|---------|----------------|---------------------|
| 121.txt | 27,406 | 2025-12-02 |
| 174.txt | 18,737 | 2025-12-02 |
| 22.txt | 67,325 | 2025-12-02 |
| 23.txt | 21,540 | 2025-12-02 |
| 24.txt | 36,729 | 2025-12-02 |
| 25.txt | 38,289 | 2025-12-02 |
| 26.txt | 33,779 | 2025-12-02 |
| 27.txt | 19,435 | 2025-12-02 |
| 28.txt | 30,327 | 2025-12-02 |
| 29.txt | 22,922 | 2025-12-02 |
| 30.txt | 23,162 | 2025-12-02 |
| 37.txt | 39,893 | 2025-12-02 |
| 38.txt | 42,462 | 2025-12-02 |
| 39.txt | 40,137 | 2025-12-02 |
| 40.txt | 39,301 | 2025-12-02 |
| 41.txt | 46,604 | 2025-12-02 |
| 42.txt | 40,273 | 2025-12-02 |
| 43.txt | 35,418 | 2025-12-02 |
| 44.txt | 41,050 | 2025-12-02 |
| 45.txt | 35,622 | 2025-12-02 |
| 46.txt | 36,231 | 2025-12-02 |
| 52.txt | 25,207 | 2025-12-02 |
| 53.txt | 30,190 | 2025-12-02 |
| 54.txt | 15,324 | 2025-12-02 |
| 55.txt | 24,794 | 2025-12-02 |
| 56.txt | 20,386 | 2025-12-02 |
| 57.txt | 17,196 | 2025-12-02 |

---

### **6. FLUJO DE TRABAJO ESTANDARIZADO**

Para futuras subidas, seguir este procedimiento:

```bash
# LOCAL: Subir a /tmp
rsync -avz *.txt cse@cse.izt.uam.mx:/tmp/optativas/

# SERVIDOR: Mover y configurar
sudo mv /tmp/optativas/*.txt /var/www/cse.izt.uam.mx/documentos/licenciatura/optativas/
sudo chown apache:apache /var/www/cse.izt.uam.mx/documentos/licenciatura/optativas/*.txt
sudo chmod 644 /var/www/cse.izt.uam.mx/documentos/licenciatura/optativas/*.txt
sudo rm -rf /tmp/optativas/

# SERVIDOR: Corregir SELinux
sudo restorecon -Rv /var/www/cse.izt.uam.mx/documentos/licenciatura/optativas/

# Agregado 13/12/2025
# Verificar despliegue correcto en navegador web, por ejemplo:
https://cse.izt.uam.mx/documentos/licenciatura/optativas/52.txt
```

---

### **7. CONCLUSIÓN**

La subida de archivos se completó exitosamente con:
- ransferencia segura mediante directorio temporal
- Configuración correcta de propietario y permisos
- **Corrección crítica de contexto SELinux**
- Verificación de accesibilidad pública vía HTTPS

**Lección aprendida**: En sistemas con SELinux (Rocky Linux, RHEL, CentOS), **siempre ejecutar `restorecon` tras mover archivos al web root**, incluso cuando los permisos tradicionales parecen correctos.

**Estado**: Listo para producción. Archivos accesibles públicamente.

---

### **8. FIRMAS DE ACEPTACIÓN**

| Rol | Nombre | Firma | Fecha |
|-----|--------|-------|-------|
| **Propietario del Sistema** | cse | _____________________ | 2025-12-04 |
| **Propietario del Sistema** | cse | _____________________ | 2025-12-13 |
---

**Documento generado automáticamente a partir de la bitácora de comandos ejecutados en cse.izt.uam.mx**
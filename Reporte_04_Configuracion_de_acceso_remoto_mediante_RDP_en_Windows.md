# **Reporte 04: Configuración de acceso remoto mediante RDP en Windows**

**Dirigido a**: personal técnico y administrativo de la Universidad Autónoma Metropolitana  
**Elaborado por**: Dr. Jesús Zavala Ruiz (Coordinador de Sistemas Escolares UAMI)  
**Colaboración**: Qwen.ai  
**Compatible con**: Windows 10, Windows 11 y Windows Server 2016/2019/2022
**Versión**: 1.0
**Creación**: 18 de diciembre de 2025  
**Última actualización**: 18 de diciembre de 2025  

---

## **Introducción**

Este manual describe el procedimiento para habilitar y configurar el **acceso remoto mediante Protocolo de Escritorio Remoto (RDP)** en un equipo Windows que aloja aplicaciones críticas, como el cliente del Sistema de Administración Escolar (SAE) desarrollado con Uniface.

La configuración incluye aspectos de seguridad, autenticación y red, asegurando que el acceso remoto sea funcional y conforme a las políticas institucionales.

---

## **1. Requisitos previos**

### **1.1. Permisos necesarios**
- Acceso con cuenta de **administrador local** en el equipo Windows.
- Permiso para modificar la configuración de red y del firewall.

### **1.2. Información requerida**
- Dirección IP del equipo Windows (ej. `148.206.52.17`).
- Nombre del equipo (ej. `PCSV26F-52017`).
- Credenciales de usuario con permisos para inicio de sesión remoto (ej. `icse` en dominio `SUAMIZT`).

> **Nota**: en entornos institucionales, el equipo suele pertenecer a un dominio (como `SUAMIZT`), lo que influye en la autenticación.

---

## **2. Habilitar el escritorio remoto**

### **2.1. Desde la interfaz gráfica**

1. Presiona `Win + I` para abrir **Configuración**.
2. Ve a **Sistema > Escritorio remoto**.
3. Activa la opción **Habilitar el escritorio remoto**.
4. Marca la casilla **Mantener el equipo encendido para conexiones remotas** (opcional pero recomendado).
5. Anota el nombre del equipo que aparece en la sección “Cómo conectarse”.

> **Alternativa en Windows Server**:  
> Abre **Administrador del servidor > Servidor local** y haz clic en **Habilitar** junto a “Escritorio remoto”.

### **2.2. Verificación mediante línea de comandos**

Abre una terminal con privilegios de administrador y ejecuta:

```cmd
reg add "HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Terminal Server" /v fDenyTSConnections /t REG_DWORD /d 0 /f
```

Este comando desactiva la denegación de conexiones RDP.

---

## **3. Configurar el firewall de Windows**

El firewall debe permitir el tráfico entrante en el puerto **3389/TCP**.

### **3.1. Desde la interfaz gráfica**

1. Busca y abre **Firewall de Windows con seguridad avanzada**.
2. En el panel izquierdo, selecciona **Reglas de entrada**.
3. En el panel derecho, haz clic en **Nueva regla...**.
4. Selecciona **Puerto** y haz clic en **Siguiente**.
5. Elige **TCP**, selecciona **Puertos locales específicos** e ingresa `3389`.
6. Selecciona **Permitir la conexión**.
7. Marca las redes aplicables (Dominio, Privada; **no marcar Pública** en entornos institucionales).
8. Asigna un nombre como **RDP - Escritorio remoto**.
9. Haz clic en **Finalizar**.

### **3.2. Mediante PowerShell (alternativa rápida)**

Ejecuta como administrador:

```powershell
Enable-NetFirewallRule -DisplayGroup "Escritorio remoto"
```

> **Nota**: este comando activa las reglas predefinidas de RDP del sistema.

---

## **4. Configurar usuarios autorizados**

Solo los usuarios explícitamente autorizados podrán conectarse por RDP.

### **4.1. En equipos miembros de dominio**

1. Abre **Configuración > Sistema > Escritorio remoto**.
2. Haz clic en **Seleccionar usuarios que pueden acceder de forma remota**.
3. En la ventana que aparece, haz clic en **Agregar**.
4. Escribe el nombre del usuario del dominio, por ejemplo:  
   ```
   SUAMIZT\icse
   ```
5. Haz clic en **Comprobar nombres** para validar, luego en **Aceptar**.

> **Importante**: si el equipo está en un dominio, los usuarios deben pertenecer al grupo **Usuarios de Escritorio Remoto** a nivel local o tener permisos explícitos.

### **4.2. En equipos independientes (no en dominio)**

1. Sigue los mismos pasos que en 4.1.
2. Agrega usuarios locales, por ejemplo:  
   ```
   icse
   ```

---

## **5. Verificar la configuración de red**

### **5.1. Confirmar dirección IP**

Abre una terminal y ejecuta:

```cmd
ipconfig
```

Busca la sección de tu adaptador de red activo y anota la **IPv4**:
```
Dirección IPv4. . . . . . . . . . . . . . : 148.206.52.17
```

### **5.2. Confirmar que el puerto 3389 está escuchando**

Ejecuta en la terminal:

```cmd
netstat -an | findstr :3389
```

**Salida esperada:**
```
TCP    0.0.0.0:3389           0.0.0.0:0              LISTENING
```

Esto confirma que el servicio de Escritorio Remoto está activo y aceptando conexiones.

---

## **6. (Opcional) Cambiar el puerto RDP predeterminado**

Por seguridad, se recomienda cambiar el puerto 3389 estándar a uno no convencional.

> **Advertencia**: si cambias el puerto, debes comunicarlo a los administradores y ajustar los túneles SSH o reglas de firewall.

### **6.1. Modificar el registro**

1. Abre `regedit` como administrador.
2. Navega a:  
   ```
   HKEY_LOCAL_MACHINE\System\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp
   ```
3. Busca la clave **PortNumber** (tipo `DWORD`).
4. Cambia su valor a un número entre 1025 y 65535 (ej. `3390`).
5. Reinicia el equipo o el servicio **Escritorio remoto**.

### **6.2. Actualizar el firewall**

Si cambias el puerto, crea una nueva regla de firewall para el nuevo puerto y desactiva la del 3389.

---

## **7. Pruebas de conectividad**

### **7.1. Desde otro equipo Windows**

1. Abre **Conexión a Escritorio Remoto** (`mstsc`).
2. En “Equipo”, escribe la IP del servidor:  
   ```
   148.206.52.17
   ```
3. Haz clic en **Conectar**.
4. Ingresa las credenciales:
   - Usuario: `SUAMIZT\icse`
   - Contraseña: `Pa$$word`

Si todo está bien, verás el escritorio remoto.

### **7.2. Desde un cliente Linux (Fedora)**

Usa el túnel SSH descrito en el manual complementario y conecta a `localhost:8000`.

---

## **8. Buenas prácticas de seguridad**

1. **No expongas RDP directamente a internet**: siempre úsalo a través de túneles SSH o redes privadas.
2. **Restringe los usuarios autorizados**: solo aquellos que realmente necesitan acceso.
3. **Mantén el sistema actualizado**: aplica parches de seguridad regularmente.
4. **Habilita solo en redes confiables**: el firewall no debe permitir RDP en perfiles de red “Pública”.
5. **Considera el uso de Network Level Authentication (NLA)**: está activado por defecto y mejora la seguridad.

---

## **9. Resumen de comandos y verificaciones útiles**

| Acción | Comando o método |
|-------|------------------|
| Habilitar RDP | `reg add ... fDenyTSConnections /d 0` |
| Ver puerto 3389 | `netstat -an \| findstr :3389` |
| Ver IP | `ipconfig` |
| Activar reglas de firewall | `Enable-NetFirewallRule -DisplayGroup "Escritorio remoto"` |
| Probar conexión local | `mstsc /v:127.0.0.1` |

---

## **Conclusión**

Con este manual, podrás configurar de forma segura y funcional el acceso remoto mediante RDP en cualquier equipo Windows de la institución. Esta configuración es esencial para el mantenimiento remoto de aplicaciones críticas como el SAE, garantizando al mismo tiempo el cumplimiento de los estándares de seguridad de la Universidad Autónoma Metropolitana.
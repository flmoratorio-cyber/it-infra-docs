# Documentación Técnica de Infraestructura y Servicios IT

Este documento reúne la arquitectura, pila tecnológica, registro de incidentes y estado operativo de la infraestructura de comunicaciones, servicios web y servidores de la organización. Toda la información sensible (direcciones IP, nombres de dominio, identidades, claves criptográficas) ha sido anonimizada para su publicación pública en GitHub.

---

## 1. Infraestructura de Correo Corporativo Cloud (Microsoft 365 & Cloudflare)

### Qué es
Plataforma de correo corporativo de alta disponibilidad basada en Microsoft 365 con orquestación DNS centralizada en Cloudflare.
Elimina la dependencia de servidores de hosting compartido tradicionales y previene interrupciones mediante el desacople completo de servicios.

### Arquitectura

```
[ NIC / Registrar Registrador ] 
               │
               ▼ (Delegación NameServers)
    [ Cloudflare Anycast DNS ]
               │
      ┌────────┴──────────────────────────┐
      │                                   │
[ Registros MX / TXT / CNAME ]   [ Registros A / CNAME ]
      │                                   │
      ▼                                   ▼
[ Microsoft 365 Tenant ]         [ Servidor Web VPS / Hosting ]
(Exchange Online 99.9% SLA)      (Aislado de la capa de correo)
```

### Stack
- **Gestión DNS & Orquestación:** Cloudflare (Red Anycast, registros MX, SPF, DKIM CNAME, DMARC, Autodiscover).
- **Servidor de Correo SaaS:** Microsoft 365 (Plan Empresa Básico / Exchange Online).
- **Autenticación & Identidad:** Microsoft Entra ID (Global Admin sin licencia), Grupos de Distribución para reenvíos internos.
- **Protocolos de Acceso:** Exchange Nativo / MAPI, Webmail (Outlook Web), Autodiscover.

### Incidentes / Decisiones

- **Incidente 1:** Caída total del servidor/nodo de hosting compartido primario y pérdida de recepción de correos.
  - **Síntoma:** Los remitentes externos recibían errores de conexión SMTP/4xx/5xx y las casillas corporativas no recibían emails.
  - **Causa:** Inaccesibilidad física e inestabilidad técnica en el nodo compartido del proveedor donde residía el servicio web y MX.
  - **Solución:** Activación de failover manual redirigiendo los MX en Cloudflare hacia un servidor de contingencia (`<dominio_secundario.us>`). Decisión estratégica: Desacoplar definitivamente el correo del hosting web migrando los MX a Microsoft 365.

- **Incidente 2:** Bloqueo de reenvíos automáticos hacia el exterior desde casillas y grupos.
  - **Síntoma:** Los correos reenviados o distribuidos rebotaban con el código de error `550 5.7.520 Access denied`.
  - **Causa:** La política de antispam saliente por defecto en Microsoft 365 bloquea el reenvío automático externo no explícito.
  - **Solución:** Edición de la directiva de salida contra correo no deseado (*Anti-spam outbound policy*) en Microsoft Defender, estableciendo la regla de reenvío automático en "Activado".

- **Incidente 3:** Notificación de "Inicio de sesión bloqueado" en Shared Mailboxes y fallas en la entrega individual.
  - **Síntoma:** Confusión operativa al intentar usar buzones compartidos como reenviadores puros hacia inboxes personales.
  - **Causa:** Los Shared Mailboxes bloquean el inicio de sesión directo y están diseñados como buzones secundarios compartidos, no como listas de distribución transparentes.
  - **Solución:** Reemplazo de los Shared Mailboxes por Grupos de Distribución (*Distribution Lists*) habilitados para recibir correos de remitentes externos.

### Estado Real
- **En producción:**
  - Delegación DNS autoritativa activa en Cloudflare para `<dominio_principal.com.ar>` y `<dominio_secundario.us>`.
  - Tenant de Microsoft 365 aprovisionado con `<cantidad_licencias>` licencias activas y usuario `<admin_sin_licencia>` como Global Admin.
  - Registros MX apuntando a `<dominio_principal.mail.protection.outlook.com>`.
  - Firma criptográfica DKIM activa mediante CNAMEs (`<aqui el selector dkim 1>` y `<aqui el selector dkim 2>`).
  - Registro SPF consolidado: `v=spf1 include:spf.protection.outlook.com -all`.
  - Grupos de distribución configurados para recepción externa y ruteo directo.
- **En plan:**
  - Ajuste gradual de política DMARC desde `p=none` hacia `p=quarantine` tras período de monitoreo de reportes RUA/RUF.
  - Retiro definitivo del servidor de contención de contingencia tras cumplir la ventana de seguridad.

---

## 2. Arquitectura Web y Landing Page Estática (Cloudflare Pages)

### Qué es
Sitio web corporativo migrado desde un CMS dinámico desactualizado hacia una arquitectura de landing page estática en HTML5, CSS3 y JS.
Garantiza tiempos de carga ultrarrápidos, costo de alojamiento cero y eliminación total de vectores de ataque por servidor o base de datos.

### Arquitectura

```
[ Usuario / Navegador ]
          │
          ▼ (Petición HTTPS)
[ Cloudflare CDN Edge Network ]
          │
  ┌───────┴──────────────────────────────┐
  │                                      │
[ Cloudflare Pages Static Assets ]    [ Cloudflare Pages Functions / Worker ]
(index.html, CSS Vanilla, JS)         (Procesamiento de formulario / API)
                                         │
                                         ▼
                               [ Resend / Transactional Mail API ]
```

### Stack
- **Frontend:** HTML5 Semántico, CSS3 Vanilla (variables CSS, Flexbox/Grid, animaciones sutiles), JavaScript Vanilla (Smooth scroll, IntersectionObserver).
- **Alojamiento:** Cloudflare Pages (Direct Upload / Integración Git con CI/CD).
- **Backend Serverless:** Cloudflare Pages Functions (`/functions/api/contact.js`) con integración API HTTP para envío de formularios.
- **Seguridad & Rendimiento:** CDN Global Edge, Caché en borde, Imágenes WebP optimizadas, Honeypot antispam sin reCAPTCHA pesado.

### Incidentes / Decisiones

- **Incidente 1:** Imposibilidad de actualizar o exportar el sitio WordPress legacy desde el panel.
  - **Síntoma:** Error `cURL error 60: SSL certificate problem: unable to get local issuer certificate` al intentar instalar plugins de exportación estática.
  - **Causa:** El entorno PHP del servidor web antiguo poseía el almacén local de certificados CA desactualizado e inoperativo.
  - **Solución:** Cancelación de reparación del servidor legacy; ejecución de un scraper headless cliente (Playwright / wget) para extraer contenido limpio e imágenes en resolución nativa.

- **Incidente 2:** Incompatibilidad del procesador de formularios PHP (`enviar.php`) en entornos serverless estáticos.
  - **Síntoma:** Error `404 Not Found` o falta de ejecución de scripts PHP al alojar en Cloudflare Pages.
  - **Causa:** Cloudflare Pages es una plataforma estática pura sin soporte para intérpretes ejecutable de PHP.
  - **Solución:** Implementación de Cloudflare Pages Functions en JS junto con la API de correo transaccional, o conversión del bloque a enlaces de contacto directo (`mailto:` y enlace con mensaje predefinido a WhatsApp).

### Estado Real
- **En producción:**
  - Código fuente maquetado en HTML5/CSS3/JS vanilla sin dependencias de frameworks externos (sin Bootstrap, sin jQuery).
  - Activos gráficos convertidos a formatos livianos (WebP/SVG) alojados en ruta relativa `./imagenes/`.
  - Dominio `<dominio_principal.com.ar>` vinculado al proyecto de Cloudflare Pages con certificado SSL administrado automáticamente.
- **En plan:**
  - Integración de sección modular interactiva para exhibición de casos de éxito usando tarjetas Grid en subdominio/sección dedicada.
  - Despliegue de funciones serverless adicionando validación de tasa de solicitudes (*rate limiting*) por IP.

---

## 3. Servidor de Correo Legacy y Filtros Antispam (Exim / Ferozo / Rspamd)

### Qué es
Infraestructura de correo en servidor dedicado/compartido CentOS administrado mediante panel Ferozo y MTA Exim con motor Rspamd.
Gestiona el filtrado antispam, reglas de ACL, autenticación SPF/DKIM/DMARC local y listas de confianza para mitigación de falsos positivos.

### Arquitectura

```
[ Remitente Externo ] ───> [ MTA Exim (Puerto 25) ]
                                   │
                                   ▼
                      [ Motor Antispam Rspamd / SpamAssassin ]
                                   │
             ┌─────────────────────┴─────────────────────┐
             │ (Análisis de SPF, DKIM, DMARC, Content)   │
             ▼                                           ▼
   [ Coincidencia en Whitelist ]            [ Puntuación > Umbral ]
   (Aceptado / Bypassed)                    (Marcado [POSIBLE SPAM] / Junk)
```

### Stack
- **Sistema Operativo & Panel:** CentOS Linux, Panel Ferozo / cPanel.
- **MTA (Mail Transfer Agent):** Exim 4.x (`/opt/exim/` o `/etc/exim/`).
- **Motor de Filtrado Antispam:** Rspamd / SpamAssassin integrado en backend (`protector.linux.backend`).
- **Seguridad & Whitelisting:** Filtros de ACL en Exim, listas de correo locales (`/root/whitelists/`, `/etc/exim/whitelist_domains`, `.spamassassin/whitelist`).

### Incidentes / Decisiones

- **Incidente 1:** Envíos salientes legítimos marcados como SPAM o rebotados con error `550-5.7.1 action rejected` / `RCVD_TLS_ALL`.
  - **Síntoma:** Los destinatarios externos o evaluadores automáticos marcaban fallas en la firma DKIM y SPF.
  - **Causa:** El VPS enviaba correos mediante un Smarthost/Relay (`<smarthost_hostname>`) con una IP diferente (`<aqui la ip smarthost>`), modificando encabezados después de la firma y rompiendo la integridad criptográfica.
  - **Solución:** Configuración de canonicalización a `relaxed/relaxed` y `dkim_strict = 0` en Exim; actualización del registro SPF para incluir el origen del proveedor (`v=spf1 ip4:<aqui la ip vps> include:<aqui el spf del proveedor> ~all`).

- **Incidente 2:** Bloqueo de envíos a cuentas Microsoft (Hotmail/Outlook) con error `550 Destinatario en lista de supresion`.
  - **Síntoma:** Imposibilidad de responder a direcciones de correo específicas de Hotmail desde el servidor local.
  - **Causa:** La IP del Smarthost compartido del proveedor fue agregada a una lista de supresión por rebotes previos de otros inquilinos.
  - **Solución:** Creación de un router de bypass especial en Exim (`manualroute`) asignado específicamente al destinatario para enviar directamente desde la IP propia (`<aqui la ip vps>`) omitiendo el relay bloqueado.

- **Incidente 3:** Correos de clientes legítimos (ej. proveedores internacionales) marcados como `[POSIBLE SUPLANTACION DE IDENTIDAD]` o `[SOSPECHOSO PDF]`.
  - **Síntoma:** Mensajes con adjuntos de comercio exterior ingresaban a la carpeta SPAM con puntuaciones de Rspamd superiores a 12.0.
  - **Causa:** Mismatches de SPF en los dominios de los clientes (`R_SPF_SOFTFAIL`) y PDFs firmados/encriptados detectados preventivamente por el escáner.
  - **Solución:** Creación e inyección masiva de dominios de confianza mediante scripts de gestión en `/etc/exim/whitelist_domains` y `.spamassassin/whitelist`, e incremento del umbral de rechazo a score 7.0.

- **Incidente 4:** Reenvíos de correo (`forwarders`) rebotando en destino por falla de SPF.
  - **Síntoma:** Mensajes recibidos en la casilla `<info@dominio.com.ar>` y reenviados a cuentas personales eran rechazados por servidores externos.
  - **Causa:** El reenvío preserva la dirección del remitente original (`From`), provocando que la IP del servidor intermedio no coincida con el SPF del creador del mensaje.
  - **Solución:** Implementación de SRS (*Sender Rewriting Scheme*) en el transporte de Exim para reescribir el campo `Return-Path` usando el dominio local autorizado.

### Estado Real
- **En producción:**
  - Servidor Exim ajustado con reglas ACL para bypass de firmas y mitigación de falsos positivos en clientes críticos.
  - Scripts de gestión de listas blancas en `/root/whitelists/` activos para mantenimiento de dominios de confianza.
  - Registros de seguridad sincronizados en zona DNS:
    - **SPF:** `<aqui el registro spf completo>`
    - **DKIM:** `<aqui la clave dkim>`
    - **DMARC:** `v=DMARC1; p=none; rua=mailto:<admin@dominio.com.ar>`
- **En plan:**
  - Desactivación progresiva de la función de servidor de correo local al concretarse la migración completa a Microsoft 365.
  - Mantenimiento del VPS exclusivamente para servicios web e APIs especializadas.

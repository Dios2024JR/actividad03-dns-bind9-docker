# Actividad 03: Servidor DNS Autoritativo con BIND9 en Docker

Implementacion de un servidor DNS autoritativo mediante BIND9 en Docker (ubuntu/bind9:latest) para la zona local lan.local.

## Estructura del proyecto
- docker-compose.yml: configuracion del contenedor ind9-autoritativo, mapeo de puertos 53/tcp y 53/udp, y volumenes de configuracion y zonas.
- config/named.conf.options: opciones generales del servidor DNS, recursion y reenviadores (orwarders).
- config/named.conf.local: declaracion de la zona autoritativa maestra lan.local.
- zones/db.local.lan: archivo de zona con los registros DNS configurados.

---

## 1. Que contiene un archivo de zona
Un archivo de zona es un archivo de texto plano que almacena los registros de recursos (Resource Records) que definen un dominio especifico en el servidor DNS. En este proyecto (zones/db.local.lan) contiene:
- **$TTL (Time To Live):** tiempo maximo (en segundos) que otros servidores o clientes pueden guardar en cache las respuestas antes de volver a consultar.
- **Registro SOA (Start of Authority):** define la informacion administrativa principal de la zona, incluyendo el servidor de nombres primario (
s1.lan.local.), el correo del administrador (dmin.lan.local.) y los temporizadores de sincronizacion.
- **Registro NS (Name Server):** indica cual es el servidor DNS autoritativo responsable de responder por la zona (
s1.lan.local.).
- **Registros A (Address):** asocian nombres de host con direcciones IPv4 especificas (por ejemplo, 
s1 -> 192.168.1.10, web y pp -> 192.168.1.20, mail -> 192.168.1.30).
- **Registro CNAME (Canonical Name):** crea un alias que apunta de un nombre de dominio a otro nombre canonico (www.lan.local -> web.lan.local).
- **Registro MX (Mail Exchange):** especifica el servidor encargado de recibir el correo electronico del dominio junto con su prioridad (10 mail.lan.local.).
- **Registro TXT:** almacena texto informativo o de verificacion asociado al dominio (como politicas SPF: "v=spf1 a mx ~all").

## 2. Por que se utiliza el serial SOA
El **numero de serie (Serial)** dentro del registro SOA (por ejemplo, 2026112101 en formato AAAAMMDDNN) funciona como un **control de versiones** de la zona DNS:
- Permite que los **servidores DNS secundarios (esclavos)** sepan si el archivo de zona en el servidor maestro ha sido modificado desde su ultima sincronizacion.
- Cada vez que se edita un registro en la zona, el administrador debe incrementar este numero. Cuando los servidores secundarios consultan el SOA (segun el intervalo Refresh), detectan que el serial es mayor y solicitan una **transferencia de zona (AXFR/IXFR)** para actualizar sus copias. Si el serial no se incrementa, los secundarios ignoran los cambios.

## 3. Como comprobar registros DNS
Para validar tanto la sintaxis de configuracion como la resolucion real del servidor se utilizan herramientas de diagnostico:
1. **Validacion interna de BIND9:**
   - docker exec bind9-autoritativo named-checkconf: verifica que 
amed.conf.options y 
amed.conf.local no tengan errores de sintaxis.
   - docker exec bind9-autoritativo named-checkzone lan.local /etc/bind/zones/db.local.lan: comprueba la integridad del archivo de zona y confirma la carga del serial (OK).
2. **Consultas de resolucion desde el cliente (
slookup / dig):**
   - 
slookup web.lan.local 127.0.0.1 (consulta registros A apuntando directamente al servidor local).
   - 
slookup www.lan.local 127.0.0.1 (comprueba la resolucion del alias CNAME).
   - 
slookup -type=MX lan.local 127.0.0.1 (verifica los registros de correo MX).
   - 
slookup -type=TXT lan.local 127.0.0.1 (verifica los registros de texto TXT).

## 4. Por que DNS por si solo no publica una aplicacion web
El sistema **DNS funciona unicamente como una guia telefonica de Internet**: su unica responsabilidad es traducir nombres de dominio legibles por humanos (como web.lan.local) a direcciones IP (192.168.1.20) u otros registros asociados.
Para que una aplicacion web este realmente publicada y accesible al usuario se necesitan otros componentes adicionales:
- **Un servidor web o servidor de aplicaciones** (como Apache, Nginx, IIS o Node.js) escuchando en los puertos 80 (HTTP) o 443 (HTTPS) en la IP destino (192.168.1.20) para procesar peticiones y entregar el HTML/codigo de la pagina.
- **Conectividad de red y reglas de firewall** que permitan el trafico HTTP/HTTPS hacia ese servidor.
- **Certificados TLS/SSL** si se requiere navegacion segura por HTTPS.

Sin un servidor web activo detras de esa direccion IP, el cliente resolvera correctamente el nombre mediante DNS, pero obtendra un error de conexion rechazada al intentar cargar la pagina en el navegador.
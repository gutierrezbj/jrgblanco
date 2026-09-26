# jrgblanco.com — despliegue v7.2 (estático, sin build)

## 0. Antes de subir
- `legal.html` está completa (titular, NIF, domicilio, correo, fecha 26-09-2026). Nada que rellenar.
- Correo del dominio: `hablamos@jrgblanco.com` ya está puesto en `index.html` (3 puertas + botón + JSON-LD) y en `legal.html`. Debe existir en Hostinger antes de publicar (ver paso 1b).

## 1. DNS (en el registrador)
```
A     jrgblanco.com        → IP del VPS de producción
A     www.jrgblanco.com    → IP del VPS de producción
AAAA  (ambos)              → IPv6 del VPS, si la tiene
```
Comprobar: `dig +short jrgblanco.com` y `dig +short www.jrgblanco.com` devuelven la IP.

## 1b. Correo (Hostinger)
- Crear el buzón `hablamos@jrgblanco.com` en Hostinger Email (o Titan, según el plan). Al crearlo desde el mismo panel que gestiona la zona DNS, Hostinger añade solo los registros MX, SPF y DKIM; comprobar que están y añadir DMARC si no lo pone: `_dmarc TXT "v=DMARC1; p=quarantine; rua=mailto:hablamos@jrgblanco.com"`.
- Buzón personal `juan@jrgblanco.com` (opcional, para firmar propuestas y contratos). `hablamos@` puede ser buzón real o alias que reenvía a `juan@`; recomendado buzón real para poder responder desde él.
- Prueba antes de publicar: enviar un correo a hablamos@ desde Gmail y responder desde hablamos@; verificar en mail-tester.com o en «Mostrar original» de Gmail que SPF, DKIM y DMARC salen PASS.
- Los registros A/AAAA de la web y los MX del correo conviven en la misma zona: la web apunta al VPS, el correo a Hostinger. No usar el VPS para correo.

## 2. Container en el servidor (offset +250, patrón AgroWeb)
La web la sirve un container `nginx:1.27-alpine` (`jrgb-web`) en `127.0.0.1:3250`, definido en `docker-compose.yml` y `nginx.conf` de este repo (`gutierrezbj/jrgblanco`). Caché por tipo de fichero y `/health` viven en ese `nginx.conf`.
```
cd /opt/apps && git clone https://github.com/gutierrezbj/jrgblanco.git jrgb-web
cd jrgb-web && docker compose up -d
curl -s http://127.0.0.1:3250/health          # ok
ss -tlnp | grep docker-proxy | grep 0.0.0.0   # debe estar vacío
```
Actualizar la web: push al repo y en el VPS `cd /opt/apps/jrgb-web && git pull`. Los ficheros van montados en solo lectura, no hace falta reiniciar el container (sí si cambia `nginx.conf` o `docker-compose.yml`: `docker compose up -d --force-recreate`).

Registro en la casa: `"JRGB-Web|jrgb-web|docker"` en `/opt/scripts/healthcheck.sh` y proyecto `JRGB` en el monitor SA99 (Mongo `sa99.servers.vps-prod.projects.JRGB`).

## 3. nginx del VPS — `/etc/nginx/sites-available/jrgblanco.com`
Primero el snippet de cabeceras, `/etc/nginx/snippets/jrgblanco-security.conf`:
```nginx
add_header X-Content-Type-Options "nosniff" always;
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
add_header Permissions-Policy "camera=(), microphone=(), geolocation=()" always;
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
```
Después el sitio, que solo hace proxy al container:
```nginx
# www → raíz (una sola web indexada)
server {
    listen 80;
    listen [::]:80;
    server_name www.jrgblanco.com;
    return 301 https://jrgblanco.com$request_uri;
}

server {
    listen 80;
    listen [::]:80;
    server_name jrgblanco.com;

    location / {
        proxy_pass http://127.0.0.1:3250;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_read_timeout 30s;
    }

    location = /health {
        proxy_pass http://127.0.0.1:3250/health;
        access_log off;
    }

    # cabeceras de seguridad: sin add_header en las location, se heredan del server
    include snippets/jrgblanco-security.conf;

    # gzip/brotli: heredado del bloque http del Manifiesto SDD-JRGB (comprobar en el paso 5)
}
```
```
sudo ln -s /etc/nginx/sites-available/jrgblanco.com /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
```

## 4. TLS
```
sudo certbot --nginx -d jrgblanco.com -d www.jrgblanco.com --redirect
```
certbot añade el `listen 443 ssl` y la redirección 80→443 en ambos bloques. Comprobar renovación: `sudo certbot renew --dry-run`.

## 5. Verificación post-deploy (no se salta)
```
# compresión activa (Manifiesto SDD-JRGB, estándar obligatorio)
curl -sH 'Accept-Encoding: br, gzip' -I https://jrgblanco.com/ | grep -iE 'content-encoding|content-length'
# www redirige y no sirve una segunda copia
curl -sI https://www.jrgblanco.com/ | head -3
# fuentes con caché inmutable
curl -sI https://jrgblanco.com/fonts/archivo.woff2 | grep -iE 'cache-control|content-type'
# legal accesible
curl -sI https://jrgblanco.com/legal.html | head -1
```
Y con los ojos, desde el móvil en 4G:
- Pegar el enlace en WhatsApp y en un post de LinkedIn: debe salir og.png, «JRGB · Juan Ramón Gutiérrez» y la descripción.
- La tipografía no salta al cargar (font-display swap con preload de Archivo e Inter).
- Las tres puertas de contacto abren el correo con el asunto precargado.
- El enlace «Aviso legal y privacidad» del pie abre `legal.html` y «← Volver» regresa.

## 6. Registro en la casa
- Catálogo de Infraestructura: entrada «jrgblanco.com — web estática, nginx, sin puertos propios».
- healthcheck.sh: añadir `https://jrgblanco.com/` (200) y SA99 si aplica.
- Cuaderno Personal, sección 2: estado «online» con fecha.

## Cambios v7.1 → v7.2
- Tipografías autoalojadas (Archivo, Fraunces itálica, Inter) como WOFF2 variables subconjuntadas a latín: 193 KB en total, cero peticiones a terceros.
- Botones de reproducción ocultos hasta que exista el vídeo (los marcos siguen con el fondo del sistema).
- `legal.html` nueva: aviso legal LSSI + privacidad honesta (sin cookies, sin formularios, fuentes propias). Enlace en el pie.
- Fila «Tecnología» de credenciales: Ingeniero Electrónico y Product Management delante; CCNA al final.
- Correo del dominio `hablamos@jrgblanco.com` en puertas, botón, JSON-LD y legal.
- Showreel: mientras no hay vídeo, un canvas de ~3.000 partículas (hueso y 9 % ámbar) que se ordenan en «JRGB» al hacer scroll, dan una vuelta en 3D y se fijan en trama con un anillo ámbar; sin librerías, sin peticiones externas; con `prefers-reduced-motion` se pinta ya ordenado y quieto. Cuando llegue el vídeo de Escenda, se quita el `<canvas id="seDots">` y su bloque de script.
- Botón «Volver arriba»: el pez del sello suelto, fijo abajo a la derecha; aparece pasado el hero, cambia a hueso sobre fondos oscuros (mide el fondo real que tiene debajo) y vuelve al inicio con scroll suave.
- Ocultado el recuadro interno «Aquí va El cruce…» de Capacidades (era una nota de trabajo visible al público). Vuelve cuando exista el diagrama.
- Sin otros cambios de copy ni de estructura.

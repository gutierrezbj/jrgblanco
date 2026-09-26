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

## 2. Ficheros en el servidor
```
sudo mkdir -p /var/www/jrgblanco.com
# subir el contenido de esta carpeta (index.html, legal.html, fonts/, favicon.ico, og.png, jrgb-*.svg/png)
sudo chown -R www-data:www-data /var/www/jrgblanco.com
```

## 3. nginx — `/etc/nginx/sites-available/jrgblanco.com`
Primero el snippet de cabeceras, `/etc/nginx/snippets/jrgblanco-security.conf`:
```nginx
add_header X-Content-Type-Options "nosniff" always;
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
add_header Permissions-Policy "camera=(), microphone=(), geolocation=()" always;
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
```
Después el sitio:
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
    root /var/www/jrgblanco.com;
    index index.html;

    # fuentes autoalojadas: caché larga e inmutable (se versionan por nombre si cambian)
    location /fonts/ {
        add_header Cache-Control "public, max-age=31536000, immutable";
        add_header Access-Control-Allow-Origin "https://jrgblanco.com";
        include snippets/jrgblanco-security.conf;
        try_files $uri =404;
    }

    location ~* \.(png|svg|ico)$ {
        add_header Cache-Control "public, max-age=604800";
        include snippets/jrgblanco-security.conf;
        try_files $uri =404;
    }

    location / {
        add_header Cache-Control "public, max-age=600";
        include snippets/jrgblanco-security.conf;
        try_files $uri $uri/ =404;
    }

    # cabeceras mínimas de seguridad: un add_header dentro de location anula los del server,
    # por eso el snippet se incluye en cada location además de aquí
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

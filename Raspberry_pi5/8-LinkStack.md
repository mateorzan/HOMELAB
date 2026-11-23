## LinkStack

Necesitaba un enlace único para la biografía de mi cuenta de Instagram, y quería que se pareciera a mi marca en vez de usar un servicio ajeno. [LinkStack](https://github.com/linkstackorg/linkstack-docker) corre en la Raspberry en Docker y se publica con un subdominio detrás de Cloudflare. El compose y la instalación están en [Servicios](./2-Servicios.md#linkstack); aquí explico **el tema propio**, la caché y los problemas que me encontré.

### Cómo funciona un tema

Un tema de LinkStack es una carpeta dentro de `themes/` con muy pocos archivos. LinkStack siempre genera el mismo HTML (avatar, nombre, descripción, iconos y botones) y el tema solo aporta **CSS**, así que no se puede cambiar la estructura de la página.

```text
themes/mi-tema/
├── config.php           # opciones: botones, código extra, iconos
├── skeleton-auto.css    # todo el diseño
├── brands.css           # colores de los botones de marca (copiado de un tema oficial)
├── share.button.css     # botón de compartir (copiado de un tema oficial)
├── readme.md            # metadatos que lee LinkStack (nombre, versión)
├── fonts/               # fuentes locales (opcional)
└── .htaccess            # cabeceras de caché (opcional, ver más abajo)
```

Cargan en este orden: `share.button.css`, `brands.css`, `skeleton-auto.css` y por último el `animations.css` de LinkStack, que **se carga después del tema** y puede pisarlo.

### Diseño

Quise que el linktree fuera igual que mis carruseles de Instagram: papel crema con retícula de puntos, bordes negros gruesos, sombras duras sin difuminar, bloques naranjas y tipografía Fira Sans en mayúsculas con Hack para datos. Las fuentes van dentro del tema (archivos `woff2` con su licencia), sin pedir nada a servidores externos, para que cargue rápido desde la Raspberry.

Escritorio y móvil se ven igual: una única columna centrada.

Cada tarjeta lleva un pequeño LED verde en la esquina de la celda negra, como el LED de enlace de un puerto del switch de mi rack, y parpadea un instante al pulsarla. El avatar usa el mismo naranja que el resto de mi marca.

### Instalar el tema

Los temas viven en el volumen persistente del contenedor, así que sobreviven a reinicios y actualizaciones de la imagen. Copio la carpeta al contenedor y le doy el propietario correcto.

```bash
tar -cf - mi-tema | ssh usuario@IP-RASPBERRY \
  'docker cp - linkstack:/htdocs/themes/ && \
   docker exec -u root linkstack chown -R apache:apache /htdocs/themes/mi-tema'
```

Después, en el panel de LinkStack, *Apariencia → Tema* y elijo el nuevo. También se puede subir un `.zip` desde *Temas*.

### Problemas que me encontré

**Los enlaces no se veían.** Mi CSS dejaba los botones con `opacity: 0` al empezar y una animación de entrada propia. Pero el `animations.css` de LinkStack se carga después y la sustituye por la suya, que no termina en visible. Los botones se quedaban transparentes. La solución es que nunca empiecen ocultos sin una animación propia que acabe en `opacity: 1` y con un selector más específico que el de LinkStack.

**La página se veía estirada solo cuando estaba con sesión iniciada.** La barra de administrador que LinkStack muestra al propietario deja un `<div>` sin cerrar, y entonces el contenedor principal deja de ser hijo directo de `body`. Mi regla `body > .container` no coincidía y se perdía el ancho máximo. Los visitantes sin sesión nunca lo veían. Se arregla con un selector que valga en ambos casos:

```css
/* ejemplo simplificado */
:is(body, body > div) > .container { max-width: 32rem; margin: 0 auto; }
```

**Veía la versión antigua aunque purgara la caché.** Cloudflare imponía un `max-age` de varias horas al CSS, y los navegadores del móvil lo guardaban. Se resuelve en dos partes: en Cloudflare, *Caching → Configuration → Browser Cache TTL* en **Respect Existing Headers**, y un `.htaccess` dentro del tema para que el navegador revalide el CSS y las fuentes en cada visita.

```apache
<IfModule mod_headers.c>
  <FilesMatch "\.(css|woff2)$">
    Header set Cache-Control "no-cache"
  </FilesMatch>
</IfModule>
```

`no-cache` no significa "no guardar": el navegador pregunta al servidor si cambió algo (con el `ETag`) y recibe una respuesta mínima si no. Después de cambiarlo hay que purgar Cloudflare una vez.

**El enlace se quedaba cargando solo dentro de Instagram.** Funcionaba en el navegador normal, pero en el integrado de Instagram se quedaba cargando, e incluso mi blog y una imagen suelta del mismo dominio. Era el **Bot Fight Mode** de Cloudflare, que lanzaba un reto al servidor de Meta cuando comprueba el enlace. Está explicado, con cómo diagnosticarlo, en [Cloudflare y seguridad web](../Zimblade2/6-Cloudflare-Seguridad.md).

**La página tardaba en cargar.** El servidor respondía rápido y el HTML pesa poco. Lo que más pesaba era el **avatar**: un PNG de 1344×768 y más de 800 KB que se muestra a 128 px. Lo redimensioné a unos 700 px de ancho (menos de 100 KB) manteniendo las proporciones, con una copia del original. Si subes un avatar desde el panel, reduce antes su tamaño.

### Ajustes de LinkStack que cambié

Se encuentran en *Administrador → Configuración* y se guardan en el `.env` del volumen.

| Ajuste | Valor | Motivo |
| :--- | :---: | :--- |
| Enlaces del pie (Inicio, Términos, Privacidad, Contacto) | Desactivados | Son páginas de plantilla en inglés sobre LinkStack, no sobre mi sitio |
| Modo de depuración | Desactivado | En producción muestra trazas internas al visitante |
| "Powered by LinkStack" | Visible y discreto | Es un proyecto abierto; lo dejo pequeño y en la paleta del tema |

### Comprobaciones útiles

```bash
# cabeceras de caché del CSS (esperado: no-cache)
curl -sI https://linktree.example.com/themes/mi-tema/skeleton-auto.css | grep -i cache-control

# la página pública abre incluso con el parámetro que añade Instagram
curl -s -o /dev/null -w "%{http_code}\n" "https://linktree.example.com/@usuario?fbclid=test"

# peso de lo que se descarga (HTML comprimido)
curl -s -o /dev/null -H "Accept-Encoding: br" -w "%{size_download} bytes\n" https://linktree.example.com/@usuario
```

Para ver qué recibe Meta al comprobar un enlace, el [Depurador de contenido de Meta](https://developers.facebook.com/tools/debug) muestra el código de respuesta y permite volver a extraer la página.

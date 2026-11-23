## Cloudflare y seguridad web

Aquí documento los ajustes de Cloudflare que he tenido que tocar para publicar el linktree ([LinkStack](../Raspberry_pi5/8-LinkStack.md)) y proteger los paneles de administración de LinkStack y de [Ghost](./4-Ghost.VM.md) sin romper nada. Todo se hace en el plan gratuito. Los dominios están sustituidos por `example.com`.

### Bot Fight Mode y el navegador de Instagram

**Síntoma:** el enlace del linktree se quedaba cargando solo dentro del navegador integrado de Instagram. En Chrome normal funcionaba, y en Instagram fallaban también otras páginas del mismo dominio, como el blog o una imagen suelta, así que el problema no era la página.

**Causa:** en *Security → Analytics → Events* aparecía un `Managed Challenge` por **Bot fight mode** sobre una petición que venía de un datacenter de Amazon, con `Referer: facebook.com` y el parámetro `?fbclid=`. Es el servidor de Meta comprobando el enlace antes de abrirlo. Cloudflare le pone un reto que no puede resolver y el navegador de Instagram se queda cargando.

**Solución:** apagar **Bot Fight Mode** (*Security → Settings → Bot traffic*). En el plan gratuito solo se puede encender o apagar, sin excepciones por regla. Como compensación, protejo los paneles con reglas de WAF (siguiente apartado). La mitigación de DDoS de Cloudflare no depende de este ajuste y sigue activa.

**Diagnóstico:**

1. *Security → Analytics → Events* con el filtro de los últimos 15 minutos justo después de intentar abrir el enlace desde Instagram. Un evento con `Managed Challenge` o `Block` y un origen de Meta/Amazon lo confirma. Se puede abrir el detalle (JSON) de cada evento.
2. En el [Depurador de contenido de Meta](https://developers.facebook.com/tools/debug) se pega la URL y se pulsa *Scrape Again*. Debe devolver `200` y mostrar el título y la imagen.
3. Meta guarda en caché el resultado de comprobar un enlace. Si falla, puede seguir fallando un tiempo en una cuenta concreta aunque ya esté arreglado: ayuda borrar la caché de la app de Instagram o probar con otra URL (por ejemplo, añadiendo un parámetro como `?ig=1`).

### Reglas de WAF para los paneles de administración

*Security → WAF → Custom rules → Create rule*, siempre con **Edit expression** (el constructor visual no admite expresiones con varios `or` y las descarta). Acción: **Managed Challenge**.

Un navegador real resuelve el reto una vez y Cloudflare le deja una cookie válida durante un rato, así que el uso normal casi no se nota. Los clientes que no son navegadores (scripts, n8n, despliegues con token) no pueden resolver un reto: por eso **no** se aplican a la API que usan.

**LinkStack** (panel y login):

```text
(http.host eq "linktree.example.com" and (starts_with(http.request.uri.path, "/login") or starts_with(http.request.uri.path, "/dashboard") or starts_with(http.request.uri.path, "/admin") or starts_with(http.request.uri.path, "/studio")))
```

No se añade la ruta pública `/@usuario` ni las de clics y recursos: retar a esas rutas vuelve a colgar el enlace en Instagram.

**Ghost, regla 1:** la página del panel. Se usa `"/ghost/"` con la barra final para no afectar a entradas cuyo título empiece por "ghost", y se excluye `/ghost/api/`.

```text
(http.host eq "example.com" and (http.request.uri.path eq "/ghost" or starts_with(http.request.uri.path, "/ghost/")) and not starts_with(http.request.uri.path, "/ghost/api/"))
```

**Ghost, regla 2:** el login por API. Sin ella, un bot se saltaría la página y atacaría el login directamente.

```text
(http.host eq "example.com" and (starts_with(http.request.uri.path, "/ghost/api/admin/session") or starts_with(http.request.uri.path, "/ghost/api/admin/authentication")))
```

> **No retar nunca** `/ghost/api/content/` ni `/members/api/` (los usan el tema y los visitantes), ni el resto de `/ghost/api/admin/` (n8n y el despliegue del tema entran con un token en la cabecera y no pueden resolver un reto).

**Comprobación desde fuera:** una petición sin navegador a una ruta protegida devuelve `403` con la cabecera `cf-mitigated: challenge`; las rutas públicas no.

```bash
curl -s -o /dev/null -w "%{http_code}\n" https://example.com/ghost/            # 403 (reto)
curl -sI https://example.com/ghost/ | grep -i cf-mitigated                      # cf-mitigated: challenge
curl -s -o /dev/null -w "%{http_code}\n" https://example.com/rss/              # 200
curl -s -o /dev/null -w "%{http_code}\n" https://linktree.example.com/@usuario # 200
```

### Caché del navegador

Con el ajuste por defecto, Cloudflare impone un `max-age` de varias horas a los archivos estáticos aunque el servidor no lo pida, y los móviles guardan versiones antiguas del CSS aunque purgues. En *Caching → Configuration* pongo **Browser Cache TTL = Respect Existing Headers** y es el servidor (con un `.htaccess`) el que decide qué se cachea. Detalles en [LinkStack](../Raspberry_pi5/8-LinkStack.md#problemas-que-me-encontré).

### Registro DNS comodín

Tengo un registro `*` con proxy, así que **cualquier** subdominio, exista o no, resuelve a Cloudflare y devuelve `502` si no hay un servicio detrás. Es inofensivo, pero se nota con `www`. No lo borro porque puede que algo del túnel dependa de él. Si quiero evitar ese error, se puede crear una regla en *Rules → Redirect Rules*: si el host es `www.example.com`, redirigir con `301` a `https://example.com` conservando la ruta y los parámetros.

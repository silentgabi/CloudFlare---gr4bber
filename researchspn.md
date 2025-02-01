**Hola, soy g4bbdev.**  
Tengo 13 años y  

Hace tres meses, descubrí un ataque de desanonimización "0-click" que permite a un atacante obtener la ubicación de cualquier objetivo dentro de un radio de 400 km. Si la víctima tiene una aplicación vulnerable instalada en su teléfono (o un programa ejecutándose en segundo plano en su laptop), un atacante puede enviar una carga maliciosa y rastrear su ubicación en segundos—sin que se dé cuenta.  

Estoy publicando esta investigación como una advertencia, especialmente para periodistas, activistas y hackers, sobre este tipo de ataque indetectable. Cientos de aplicaciones son vulnerables, incluidas algunas de las más populares del mundo: Signal, Discord, Twitter/X y otras. Así es como funciona:  

## Cloudflare  

En términos de números, Cloudflare es fácilmente la CDN más popular del mercado, superando a competidores como Sucuri, Amazon CloudFront, Akamai y Fastly. En 2019, una gran falla en Cloudflare dejó fuera de servicio gran parte de internet por más de 30 minutos.  

Una de las funciones más utilizadas de Cloudflare es el **caché**. Cloudflare almacena copias de contenido frecuentemente accedido (como imágenes, videos o páginas web) en sus centros de datos, reduciendo la carga en los servidores y mejorando el rendimiento de los sitios web ([documentación oficial](https://developers.cloudflare.com/cache/)).  

Cuando tu dispositivo solicita un recurso que puede ser almacenado en caché, Cloudflare primero intenta recuperarlo del centro de datos más cercano. Si no está disponible, el recurso se carga desde el servidor de origen, se almacena localmente y se entrega al usuario. De forma predeterminada, [algunas extensiones de archivos](https://developers.cloudflare.com/cache/concepts/default-cache-behavior/) se almacenan automáticamente, pero los operadores del sitio pueden definir nuevas reglas de caché.  

Cloudflare tiene una presencia global masiva, con centros de datos en más de **330 ciudades y 120 países**, un **273% más que Google**. En EE.UU., por ejemplo, el centro de datos más cercano a mí está a menos de 160 km. Si vives en un país desarrollado, es probable que haya un centro de datos de Cloudflare a menos de **320 km** de distancia.  

Entonces tuve un momento de "eureka": **si Cloudflare almacena datos en caché tan cerca de los usuarios, ¿se podría explotar esto para ataques de desanonimización en sitios que no controlamos?**  

La respuesta está en los encabezados HTTP de Cloudflare:  
![cf-cache-status](https://gist.github.com/user-attachments/assets/95e1a39a-ed25-4531-9c57-a1b43c616519)  

- `cf-cache-status` puede indicar HIT/MISS (si un archivo fue entregado desde la caché).  
- `cf-ray` contiene el **código del aeropuerto** más cercano al centro de datos que atendió la solicitud.  

Si logramos que el dispositivo de la víctima cargue un recurso alojado en un sitio que usa Cloudflare, podemos entonces **mapear todos los centros de datos** de Cloudflare para descubrir cuál de ellos almacenó ese recurso. Esto nos da una estimación extremadamente precisa de la ubicación de la víctima.  

## Cloudflare Teleport  

Había un gran obstáculo antes de que pudiera probar esta teoría:  

**No se pueden enviar solicitudes HTTP directamente a centros de datos específicos de Cloudflare.** Toda la red opera con **anycast**, lo que significa que cualquier conexión TCP siempre es dirigida al centro de datos más cercano disponible.  

Sin embargo, después de investigar un poco, encontré [una publicación en el foro de Cloudflare](https://community.cloudflare.com/t/how-to-run-workers-on-specific-datacenter-colos/385851) que explicaba un **bug** para eludir esta limitación con Cloudflare Workers.  

Básicamente, descubrí que utilizando un rango de IPs internas del **Cloudflare WARP (la VPN de Cloudflare)**, era posible **forzar** ciertas solicitudes a ser atendidas por un centro de datos específico.  

Basándome en esto, desarrollé **Cloudflare Teleport** ([GitHub](https://github.com/hackermondev/cf-teleport)), un proxy basado en Cloudflare Workers que permite redirigir solicitudes HTTP a centros de datos específicos. Por ejemplo:  

🔹 `https://cfteleport.xyz/?proxy=https://cloudflare.com/cdn-cgi/trace&colo=SEA`  

Esto dirige la solicitud específicamente al **centro de datos de Seattle (SEA)**.  

Meses después, Cloudflare corrigió este bug, haciendo que la herramienta quedara obsoleta, pero hasta ese momento **funcionaba perfectamente para mis pruebas**.  

---  

## Primer Ataque de Desanonimización  

Con **Cloudflare Teleport**, pude probar mi teoría.  

Creé un pequeño programa CLI que realizaba una solicitud HTTP a una URL específica y listaba **todos los centros de datos que almacenaron en caché el recurso**.  

Para la primera prueba, elegí el favicon de **Namecheap**:  

🔗 `https://www.namecheap.com/favicon.ico`  

Este era un recurso simple, estático y almacenado en caché por Cloudflare.  

![cache hit](https://gist.github.com/user-attachments/assets/8da57801-ae8e-4adf-9a2e-ec6feec6086f)  

💥 ¡Funcionó! Pude ver **todos los centros de datos** que almacenaron en caché el favicon de Namecheap en los últimos 5 minutos.  

Esto demostró que era posible **usar el caché de Cloudflare para rastrear usuarios y estimar sus ubicaciones**.  

---  

## Aplicación Real: Signal  

El **Signal**, una de las aplicaciones de mensajería más seguras del mundo, **era vulnerable** a este ataque.  

### Ataque 1-Click  

Cuando un usuario envía un **adjunto** (por ejemplo, una imagen) en Signal, el archivo se carga en la CDN de Cloudflare (`cdn2.signal.org`).  

Tan pronto como el destinatario **abre el chat**, su dispositivo descarga automáticamente el adjunto, y Cloudflare lo almacena en caché.  

🔹 **Solución:** Bloqueé la solicitud HTTP a la CDN y envié un adjunto de prueba. Luego, utilicé mi programa para mapear los centros de datos donde se almacenó el archivo.  

📍 ¿El resultado? Descubrí que **mi centro de datos más cercano era el de Newark, NJ (EWR)**, a **240 km de mi ubicación real**.  

---  

## Ataque 0-Click  

Ahora, **¿cómo ejecutar este ataque sin que la víctima abra el chat?**  

El secreto: **notificaciones push**.  

📌 En Signal, cuando un usuario recibe un mensaje con un adjunto, la aplicación **descarga automáticamente la imagen** para mostrarla en la notificación push.  

Esto significa que **no se requiere ninguna interacción del usuario**. Solo necesitamos enviar una imagen a la víctima y esperar a que la notificación sea entregada.  

💥 Resultado: **¡la víctima es rastreada sin siquiera abrir la aplicación!**  

---  

## Aplicación Real: Discord  

El **Discord** también es vulnerable al mismo ataque. Pero en lugar de adjuntos, podemos explotar **avatares de perfil**.  

🔹 Cuando alguien recibe una **solicitud de amistad**, el avatar del remitente se carga automáticamente **para mostrarse en la notificación push**.  

Si cambiamos nuestro avatar antes del ataque, nos aseguramos de que **nadie más haya cargado ese archivo**, lo que hace que la detección sea precisa.  

Creé un bot llamado **GeoGuesser**, que automatiza todo el proceso y ejecuta el ataque en **segundos**.  

---  

## Conclusión  

- **Signal y Discord fueron alertados, pero no corrigieron el problema.**  
- **Cloudflare corrigió la falla que permitía acceder a centros de datos específicos, pero el ataque sigue siendo posible de otras formas.**  

Este ataque demuestra que incluso las aplicaciones que priorizan la **privacidad** pueden **filtrar información sensible sin que el usuario lo note**. 🚨  
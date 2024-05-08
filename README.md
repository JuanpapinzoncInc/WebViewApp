**Descripción:** App básica nativa en kotlin para Android que permite desplegar implementaciones web como C2C y WebChat con sus respectivas funcionalidades de establecer llamada por voz, adjuntar y compartir archivos,
el html al que apunta puede ser una url o un contenido cargado directamente sobre la actividad, hay comunicación entre la actividad y el contenido desplegado en el WebView para manejar acciones en ambos sentidos.
Cuenta con dos actividades, Main y WebView y e cuenta con los siguientes comportamientos:

**1.** Interceptar el momento de carga del contenido en el WebView para disparar el botón de apertura del chat.
<br>**2.** Interceptar el momento en que el WebView toma el foco para disparar el botón de apertura del chat.
<br>**3.** Interceptar los eventos al minimizar la ventana de chat para controlar la pila de actividades de la aplicación y mantener el WebView luego de ir a la Actividad principal.
<br>**4.** Solicita permisos de recursos, camera, audio y localización si la app no cuenta con ellos.
<br>**5.** Manejo de onBackPressed en WebView y validación en actividad principal para mantener actividad de WebView en la pila o dispararla si aún no se encuentra.

   **Configuración y uso:**

**1.** Se debe descargar el proyecto y configurar la actividad WebViewActivity.kt en caso de requerir personalizar comportamientos y/o contenido a cargar.

   
**1.1** El método onPageFinished de la clase WebViewClient tiene la instrucción de disparar el botón de apertura del chat, se podría personalizar según el contenido cargado en el WebView y el comportamiento esperado.
   
    override fun onPageFinished(view: WebView?, url: String?) {
        super.onPageFinished(view, url)
        view?.evaluateJavascript("setTimeout(function(){document.getElementById('btnApertura').click();},100)") {
        }
    }


**1.2** En la línea 82 se agrega una interfaz JavaScript al WebView, donde se define el método llamado onButtonClick(), dada la anotación @android.webkit.JavascriptInterface estará disponible para ser accedido desde el js del contenido
con el nombre Android, el metodo onButtonClick dispara el onBackPressed en la actividad WebView, la finalidad es hacer su llamado desde el js cuando se encuentra que la ventana de WebChat se ha minimizado y no ha sido en la transición entre 
pre encuesta y chat que es automático al pretender continuar, en cambio se dispararía si la ventana es minimizada por el usuario consiguiendo con esto que vuelva a la actividad principal sin finalizar la actividad del WebView. 

      webView.addJavascriptInterface(object {
          @android.webkit.JavascriptInterface
          fun onButtonClick() {
              onBackPressed()
          }
      }, "Android")

Con base en esto se podrían definir diferentes funciones de la actividad WebView para ser accedidas desde js.

El siguiente código es la lógica que sigue el js para capturar los eventos del WebChat, y determinar con base en los valores del arreglo de eventos los casos en los que se debe disparar el método onButtonClick, en este caso, se dispara primero un botón del html y posterior el método onButtonClick.


![image](https://github.com/JuanpapinzoncInc/WebViewApp/assets/159573664/5664054f-27d6-407a-81de-2a78306bbdf8)


Se deberían configurar métodos y consumos según la necesidad.

**1.3** De la linea 89 a la 192 se define una estructura html con los scripts y estilos necesarios para ser cargados directamente sobre el WebView. en la línea 235 se cargaría el contenido sobre el WebView, sin embargo, 
en este caso se está cargando un proyecto alojado en webrtc.inconcertcc.com dada una restricción para el correcto funcionamiento de la geolocalización.


![image](https://github.com/JuanpapinzoncInc/WebViewApp/assets/159573664/f7b9e250-f2d3-4eb1-92c0-d17014c1256e)



**1.4** Dentro de la clase WebChromeClient definida en la linea 194 se sobre escibe el método onGeolocationPermissionsShowPrompt para manejar las solicitudes de permisos de geolocalización que se generan en la página cargada en el WebView,


      override fun onGeolocationPermissionsShowPrompt(
                origin: String?,
                callback: GeolocationPermissions.Callback?
            ) {
                callback?.invoke(origin, true, false)
        }
            
y el método onShowFileChooser que se dispara cuando la página está solicitando al usuario que seleccione archivos para cargar, en este se hace posible personalizar un Intent que permita escoger entre un selector de archivos o la cámara,
a la hora de requerir adjuntar algún archivo en el chat, 

            override fun onShowFileChooser(
                webView: WebView?,
                filePathCallback: ValueCallback<Array<Uri>>?,
                fileChooserParams: FileChooserParams?
            ): Boolean {
                Log.d("WebViewActivity", "onShowFileChooser() llamado")

                mFilePathCallback = filePathCallback

                // Crear un Intent para abrir el selector de archivos de la cámara
                val cameraIntent = Intent(MediaStore.ACTION_IMAGE_CAPTURE)

                // Crear un Intent para abrir el selector de archivos
                val fileIntent = Intent(Intent.ACTION_GET_CONTENT)
                fileIntent.type = "*/*" // Tipo de archivo, "*/*" permite seleccionar cualquier tipo de archivo

                // Crear un Intent para envolver ambos Intents
                val chooserIntent = Intent.createChooser(fileIntent, "Selecciona un archivo")
                chooserIntent.putExtra(Intent.EXTRA_INITIAL_INTENTS, arrayOf(cameraIntent))

                // Verificar si hay actividades que puedan manejar el Intent
                if (chooserIntent.resolveActivity(packageManager) != null) {
                    startActivityForResult(chooserIntent, FILE_CHOOSER_REQUEST_CODE)
                } else {
                    // Manejar el caso en que no haya ninguna aplicación disponible para manejar el Intent
                    Log.e("WebViewActivity", "No hay aplicaciones disponibles para manejar el Intent.")
                }

                return true
            }

posteriormente dentro del método onActivityResult se debe controlar la respuesta a estos Intents.


**1.5** Flujo de actividades, dentro de la actividad Main se valida si la actividad WebView ya está en la pila o si debe crear una instancia, en el siguiente bloque se ve como se dispara el Intent ante el click del floatingActionButton


    binding.fab.setOnClickListener { view ->

            val intent = Intent(this, WebViewActivity::class.java)
            intent.addFlags(Intent.FLAG_ACTIVITY_CLEAR_TOP or Intent.FLAG_ACTIVITY_SINGLE_TOP)
            startActivity(intent)
            finish()
        }

hay que descomentar finish() dado que desde la actividad webView se dispara un Intent para lanzar nuevamente una instacia de la actividad Main.

Dentro de la actividad WebView se dispara el Intent para lanzar la actividad principal cuando se ejecuta el onBackPressed, por tanto se sobre escribe el método.

      override fun onBackPressed() {
          val intent = Intent(this, MainActivity::class.java)
          startActivity(intent)
        }

 **2.** Posterior a la adecuación del archivo WebVIewActivity.kt se puede correr la aplicación en maquina virtual, o física para las respectivas validaciones e implementar posibles ajustes.
     
  **2.1** Si se desea probar directamente la actividad WebView al lanzar la aplicación se puede tocar el archivo manifest y pasar el intent-filter de una actividad a otra.
     
  ![image](https://github.com/JuanpapinzoncInc/WebViewApp/assets/159573664/fc946313-6ddf-4b0f-b2c5-3c8b5033d3ec)


  **3.** Para la instalación en otros dispositivos de manera independiente se puede generar el respectivo apk, si se está trabajando con AndroidStudio, la opción se encuentra en Build - Build Bundle(s) / APK(s) - Buld APK(s).

     
![image](https://github.com/JuanpapinzoncInc/WebViewApp/assets/159573664/f7fbaadc-82d2-40c2-a70e-de390ae42e39)

        

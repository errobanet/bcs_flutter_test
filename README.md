id: bcs-face-flutter

# Como usar BCS en Flutter

## Introducción

En la siguiente guía vamos a ver como integrar el módulo de verificación de identidad de BCS en flutter.

Para poder hacer esto vas a necesitar tener que tener configurado tu entorno para desarrollo flutter.

Vamos a utilizar como IDE `IntelliJ Idea`, pero puede usarse `android studio` o `visual studio code` sin ningun problema, las opciones son muy parecidas. Para la versión de iOS es necesaria una Mac y XCode.



## Plugin
Para poder utilizar BCS de forma rápida necesitas agregar la dependencia de `bcssdk_client`

```bash
flutter pub add bcssdk_client
```

Tambien podes editar el archivo `pubspec.yaml` y agregarla manualmente.

```yaml
dependencies:
  flutter:
    sdk: flutter
  #Otras dependencias
  bcssdk_client: ^1.4.0
```
<aside class="positive">
No olvides hacer el 'flutter pub get'
</aside>

Adicionalmente necesitamos agregar unas dependencias nativas que no pueden agregarse en el plugin.

* Android: archivo bcssdk.1.x.x.aar
* iOS: bcssdk.xcframework

Los mismos son provistos al equipo de desarrollo por erroba.

## Permisos
La verificación de identidad con rostro necesita algunos permisos en el dispositivo:
* Cámara
* Micrófono (solo android)

El SDK los solicita, pero si son denegados obtendras la respuesta `PERMISSIONS_ERROR`.
<aside class="negative">
Si los permisos son denegados multiples veces la aplicación ya no mostrara la solicitud, este es el comportamiento natural tanto en Android como en iOS, una práctica habitual es abrir la configuración de la APP para que el usuario lo habilite.
</aside>

### Android

En Android no necesitas editar el archivo AndroidManifest.xml, el plugin agrega lo necesario.

### iOS

En iOS hay que hacer unos ajustes en el proyecto del Runner.

Debes agregar la entrada de `NSCameraUsageDescription` a tu `Info.plist`

```xml
<key>NSCameraUsageDescription</key>
<string>Es necesario el uso de la cámara.</string>
```
En versiones mas antiguas de flutter a veces es necesario editar el PodFile para que los permisos se soliciten, de lo contrario da siempre `denegado`. Por el final del archivo el blque de install deberia quedar asi:

```pod
post_install do |installer|
  installer.pods_project.targets.each do |target|
    flutter_additional_ios_build_settings(target)
    target.build_configurations.each do |config|
            config.build_settings['GCC_PREPROCESSOR_DEFINITIONS'] ||= [
               '$(inherited)',
               'PERMISSION_CAMERA=1',
             ]
    end
  end
end
```

## Librerias nativas - Android

Primero vamos a copiar algunas librerias nativas que son necesarias que funcione el proyecto.

1. Copia el archivo `bcssdk-x.x.x.aar` a la carpeta `android/app/libs`
2. Edita el archivo `android/app/build.gradle` y agrega la dependencia `implementation files('libs/bcssdk-1.x.x.aar')`:

Debería quedarte algo así:

![image_caption](img/android.png)

## Librerias nativas - iOS

Para iOS es necesario referenciar el bcssdk.xcframework en el proyecto del Runner.

1. Abre el Runner con Xcode
2. Referencia el framework bcssdk.xcframework (lo puedes arrastrar desde finder y elige la opcion copy files)
3. En la configuración general del Runner, chequea que el framework este referenciado y este como "Embed & Sign":

![image_caption](img/ios_ref01.png)

4. En Build Phases, en la seccion de "Copy Bundle Resources", agrega "bcssdk.xcframework"

![image_caption](img/ios_ref02.png)

## Utilización del cliente

A continuacion vamos a mostrar el uso suponiendo que ya tenemos un código de transacción para verificar.

### Verificación

Ya tenemos todo configurado, vamos a usar el cliente!

Para llamar a la verificación solo tenes que llamar la función `_bcsPlugin.faceVerify(code);`, la función es asincrónica, debes obtener el resultado con `await`



### Respuestas

La respuesta de la llamada a `faceVerify` es una enumeración de `VerifyResult`. Puede ser uno de los siguientes valores:

* DONE
* CANCELED
* PERMISSIONS_ERROR
* CONNECTION_ERROR
* TRANSACTION_NOT_FOUND


> Según la respuesta obtenida es la acción que debes realizar en tu app.

#### DONE

La operación finalizó en el servidor, debes obtener el resultado desde tu backend, puede ser tanto Verificado como NO verificado.

#### CANCELED

El usuario canceló la operación, generalmente con la opción o gesto de volver.

#### PERMISSIONS_ERROR

Esta respuesta se da cuando no hay permisos para cámara y microfono, debes haberlos solicitado antes y verificarlos.

#### CONNECTION_ERROR

No fue posible conectar con los servidores de BCS, puede deberse a un problema de conectividad entre el dispositivo e internet/servidores de BCS.

#### TRANSACTION_NOT_FOUND

No se encontró la transacción x el identificador `code`. Ten en cuenta que después de creada solo puede ser procesada en un período corto de tiempo.

## Estilos

La interfaz de la verificación es minimalista, el único control de interacción con el usuario es un botón para `Reintentar` la operación.

Podes establecer los colores para los controles llamando a la función `setColors` del plugin.

```dart
  Future<void> _initializePluginColors() async {
    // Obtener los colores del tema actual y se los paso al plugin.
    Color primary = Theme.of(context).colorScheme.primary;
    Color onPrimary = Theme.of(context).colorScheme.onPrimary;
    await _bcsPlugin.setColors(primary, onPrimary);
  }
```

## Ambiente QA/Docker

Por defecto el cliente utiliza el ambiente productivo. Si deseas usar al ambiente de calidad o desarrollo con docker podes cambiar la URL de los servicios.

Para hacerlo está disponible la función `setUrlService` en la api.

```dart
  Future<VerifyResult> _verifyFace(String code) async {
    await _bcsPlugin.setUrlService("https://url_ambiente");
    return _bcsPlugin.faceVerify(code);
  }
```

> No dejes este código en el RELEASE de tu aplicación.

## Servicio BCS

Para utilizar la verificación, previamente debes haber generado un código de transacción desde el backend de tu aplicación.

![image_caption](img/app_seq.png)


>Es recomendable NO exponer en tus APIS la identificación de la persona, sino hacerlo sobre algún identificador de onboarding o transacción. De esta froma podés prevenir el uso de tu API por terceros.



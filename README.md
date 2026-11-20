# Ajolotito Hambriento 🦎💗

**Notificaciones locales bien hechas en Flutter, con un ajolote como pretexto.**

Proyecto de Desarrollo Móvil Integral (UTNG), hecho con TDD siguiendo el *Manual paso a paso con TDD — Ajolotito Hambriento*.

Xólotl es un ajolote virtual. Su hambre sube con el tiempo y la app te avisa cuando necesita comer. El juego es el pretexto. Lo que se aprende es a hacer notificaciones **útiles, respetuosas y probadas**.

---

## Contenido

1. [Los 10 mandamientos de una notificación](#1-los-10-mandamientos-de-una-notificación)
2. [Notificación local vs. push](#2-notificación-local-vs-push)
3. [Arquitectura por capas](#3-arquitectura-por-capas)
4. [Preparar el proyecto](#4-preparar-el-proyecto)
5. [Configuración de Android](#5-configuración-de-android)
6. [Configuración de iOS](#6-configuración-de-ios)
7. [Ejecutar las pruebas](#7-ejecutar-las-pruebas)
8. [Ejecutar la app](#8-ejecutar-la-app)
9. [Recorrido guiado en dispositivo real o emulador](#9-recorrido-guiado-en-dispositivo-real-o-emulador)
10. [Errores frecuentes](#10-errores-frecuentes)
11. [Generar el ejecutable (APK / App Bundle)](#11-generar-el-ejecutable-apk--app-bundle)

---

## 1. Los 10 mandamientos de una notificación

| # | Mandamiento | Dónde se cumple en el código |
|---|---|---|
| 1 | **Aporta valor.** Si no ganarías nada al leerla, no se envía. | Solo hay avisos de hambre y una sorpresa al día (`planificador.dart`). |
| 2 | **Pide permiso en contexto**, no al abrir la app. Antes del diálogo del sistema, explica para qué sirve. | Diálogo propio «¿Te avisamos cuando tenga hambre?» (`pantalla_ajolote.dart`). |
| 3 | **Pregunta una sola vez.** Si dice que no, no insistas; ofrece ir a Ajustes. | `GestorPermiso` devuelve `hayQueIrAAjustes` (`gestor_permiso.dart`). |
| 4 | **El usuario manda.** Cada tipo de aviso se apaga por separado. | Canales `hambre` y `sorpresas` + `PantallaAjustes`. |
| 5 | **Respeta el sueño.** Nada entre 10 pm y 8 am. | `PoliticaNotificaciones.enSilencio` y `ajustar`. |
| 6 | **Con medida.** Hay un tope diario y sobreviven los importantes. | `maxPorDia = 3`, prioridad crítico > hambre > sorpresa. |
| 7 | **Reemplaza, no acumules.** Si el estado cambia, el aviso viejo desaparece. | Ids fijos + `ServicioAjolote.sincronizar`. |
| 8 | **No avises de lo que ya ve.** Con la app en pantalla no hay avisos de hambre. | `alVolverAlFrente` / `alPasarAFondo`. |
| 9 | **Desconfía del payload.** Si llega algo raro, la app abre en el inicio. | `PayloadNotif.decodificar` (formato `v1:<destino>`). |
| 10 | **Degrada con elegancia.** Sin permiso, la app sigue funcionando y no regaña. | Tarjeta «Avisos apagados · cómo activarlos». |

---

## 2. Notificación local vs. push

| | **Local** (este proyecto) | **Push / remota** |
|---|---|---|
| ¿Quién la crea? | La propia app, en el teléfono | Un servidor |
| ¿Necesita internet o backend? | No | Sí: Firebase Cloud Messaging (Android) o APNs (iOS) |
| Paquete en Flutter | `flutter_local_notifications` | `firebase_messaging` |
| Ejemplo | «Tu ajolote tendrá hambre a las 16:00» | «Te escribió alguien» |
| ¿Dónde se aplica la política (silencio, tope)? | En el teléfono | Sobre todo en el **servidor**, antes de enviar |

**Regla práctica:** si el motivo del aviso lo conoce el propio teléfono (el tiempo, el hambre, una alarma), usa una notificación **local**. Si el motivo ocurre en otro lugar (un mensaje nuevo), necesitas **push**.

Con push, el token del dispositivo es un dato personal. Además, nunca pongas información sensible en el texto, porque se ve en la pantalla bloqueada.

Gracias al patrón *puerto/adaptador*, agregar push solo exigiría otro adaptador. La política y el planificador se reutilizan tal cual.

---

## 3. Arquitectura por capas

```
ajolotito/
  lib/
    dominio/            <- Dart puro (sin Flutter), 100 % probado sin teléfono
      reloj.dart            Reloj, RelojSistema, RelojFalso
      mascota.dart          Humor, YaEstaLlena, Mascota (saciedad calculada, JSON validado)
      categoria.dart        Importancia, CanalNotif, Categoria (hambre / sorpresa)
      mensajes.dart         Textos con personalidad y semilla determinista
      politica.dart         Silencio 22–8, tope diario, margen entre avisos
      payload.dart          Destino y PayloadNotif ("v1:<destino>")
      planificador.dart     Ids fijos, PlanNotificacion, Planificador (función pura)
      puerto.dart           Interfaz PuertoNotificaciones
      almacen.dart          Interfaz Almacen + AlmacenMemoria
      gestor_permiso.dart   Pedir permiso una vez, en contexto
      servicio.dart         ServicioAjolote: mantiene todo sincronizado
    infra/              <- lo ÚNICO que toca plugins reales
      puerto_real.dart      flutter_local_notifications + timezone + app_settings
      almacen_prefs.dart    SharedPreferences
    presentacion/
      ajolote_vm.dart       AjoloteViewModel (ChangeNotifier)
      dibujo_ajolote.dart   El ajolote hecho con círculos
      pantalla_ajolote.dart Pantalla principal (pre-permiso, lista de avisos, modo clase)
      pantalla_ajustes.dart Interruptor por categoría
    app.dart              AppAjolote con WidgetsBindingObserver (ciclo de vida)
    main.dart             Arranque: lee el payload de lanzamiento ANTES de runApp
  test/
    falsos.dart                  PuertoFalso (doble del plugin)
    mascota_test.dart            saciedad, humor, predicción, JSON
    politica_payload_test.dart   silencio y payload
    planificador_test.dart       la agenda de avisos
    permiso_test.dart            pedir una sola vez
    servicio_test.dart           sincronizar, primer/segundo plano, modo clase
    vm_test.dart                 ViewModel
    pantalla_test.dart           pruebas de widgets
```

**¿Por qué un puerto?** El plugin de notificaciones habla con el sistema operativo y no existe en una prueba automática. Por eso el dominio solo conoce la interfaz `PuertoNotificaciones`. En la app se usa `PuertoReal` y en las pruebas, `PuertoFalso`. Es como un enchufe: los aparatos no saben si la luz viene de CFE o de una planta de emergencia.

**Las tres maneras de llegar a la app desde una notificación:**

```
App cerrada (terminada)  -> getNotificationAppLaunchDetails() al arrancar (main.dart, antes de runApp)
App en segundo plano     -> onDidReceiveNotificationResponse
App abierta              -> onDidReceiveNotificationResponse
```

---

## 4. Preparar el proyecto

Requisitos: Flutter con Dart **3.8 o superior** (`flutter --version`).

1. Abre esta carpeta en VS Code y abre una terminal ahí.
2. Genera las carpetas nativas que faltan. Tu código y el `AndroidManifest.xml` incluido **no se sobrescriben**:

   ```bash
   flutter create --project-name ajolotito --org com.example .
   ```

3. **Borra `test/widget_test.dart`** si lo creó el comando anterior. Prueba un `MyApp` que no existe y sale en rojo:

   ```bash
   # Windows (PowerShell)
   Remove-Item test\widget_test.dart
   # macOS / Linux
   rm test/widget_test.dart
   ```

4. Descarga las dependencias:

   ```bash
   flutter pub get
   ```

5. Haz el cambio de Gradle de la sección 5.2 (es obligatorio para compilar en Android).

> El paquete `flutter_local_notifications` cambia seguido. Este proyecto usa la **versión 20 o superior**, donde `initialize`, `show`, `zonedSchedule` y `cancel` reciben **parámetros con nombre**. Todo el contacto con el plugin vive en `lib/infra/puerto_real.dart`. Si algo deja de compilar al actualizar, el arreglo está ahí.

---

## 5. Configuración de Android

### 5.1 `android/app/src/main/AndroidManifest.xml`

**Ya viene listo en este proyecto.** Si generas el proyecto desde cero, agrega lo siguiente.

Dentro de `<manifest>` y fuera de `<application>`:

```xml
<uses-permission android:name="android.permission.POST_NOTIFICATIONS"/>
<uses-permission android:name="android.permission.RECEIVE_BOOT_COMPLETED"/>
```

Dentro de `<application>` van los receptores. Hacen que las notificaciones programadas funcionen y sobrevivan a un reinicio del teléfono:

```xml
<receiver android:exported="false"
    android:name="com.dexterous.flutterlocalnotifications.ScheduledNotificationReceiver" />
<receiver android:exported="false"
    android:name="com.dexterous.flutterlocalnotifications.ScheduledNotificationBootReceiver">
  <intent-filter>
    <action android:name="android.intent.action.BOOT_COMPLETED"/>
    <action android:name="android.intent.action.MY_PACKAGE_REPLACED"/>
    <action android:name="android.intent.action.QUICKBOOT_POWERON" />
    <action android:name="com.htc.intent.action.QUICKBOOT_POWERON"/>
  </intent-filter>
</receiver>
```

### 5.2 `android/app/build.gradle.kts` (desugaring)

El plugin usa funciones modernas de Java que requieren *desugaring*. Abre `android/app/build.gradle.kts` (lo genera `flutter create`) y haz dos cambios.

**a)** Dentro de `android { compileOptions { ... } }` agrega la línea marcada:

```kotlin
android {
    // ...
    compileOptions {
        isCoreLibraryDesugaringEnabled = true   // <- agrega esta línea
        sourceCompatibility = JavaVersion.VERSION_11
        targetCompatibility = JavaVersion.VERSION_11
    }
    // ...
}
```

**b)** Al final del archivo, agrega el bloque `dependencies`. Si ya existe uno, agrega solo la línea:

```kotlin
dependencies {
    coreLibraryDesugaring("com.android.tools:desugar_jdk_libs:2.1.4")
}
```

> Si tu proyecto es viejo y usa `build.gradle` (Groovy), la sintaxis es `coreLibraryDesugaringEnabled true` y `coreLibraryDesugaring 'com.android.tools:desugar_jdk_libs:2.1.4'`.
>
> Las versiones exactas (`desugar_jdk_libs`, nivel de Java, `compileSdk`) las dicta el README del paquete para la versión que se instaló. Si Gradle se queja, revísalo ahí.

### 5.3 Diferencias de Android que rompen apps

- **Android 8 (API 26):** todas las notificaciones pertenecen a un **canal**. Una vez creado un canal, su importancia la controla el usuario. Cambiarla después en el código no tiene efecto; habría que crear un canal con otro id.
- **Android 13 (API 33):** el permiso `POST_NOTIFICATIONS` se pide **en tiempo de ejecución**. En versiones anteriores no se pide, pero el usuario puede apagar las notificaciones en Ajustes. Por eso la app consulta «¿están activas?».
- **Android 12 y 14:** las **alarmas exactas** requieren un permiso especial (en 14 viene negado por defecto). Esta app usa `AndroidScheduleMode.inexactAllowWhileIdle`, que no pide permisos extra y aun así llega con el teléfono en reposo (Doze).

---

## 6. Configuración de iOS

En `ios/Runner/AppDelegate.swift`, dentro de `application(_:didFinishLaunchingWithOptions:)` y antes del `return`, agrega:

```swift
UNUserNotificationCenter.current().delegate = self as? UNUserNotificationCenterDelegate
```

Ejemplo completo:

```swift
import Flutter
import UIKit

@main
@objc class AppDelegate: FlutterAppDelegate {
  override func application(
    _ application: UIApplication,
    didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?
  ) -> Bool {
    GeneratedPluginRegistrant.register(with: self)
    UNUserNotificationCenter.current().delegate = self as? UNUserNotificationCenterDelegate
    return super.application(application, didFinishLaunchingWithOptions: launchOptions)
  }
}
```

- **Permisos diferidos:** el plugin se inicializa con `requestAlertPermission: false`, `requestBadgePermission: false` y `requestSoundPermission: false`. Así no se pide nada al arrancar; la app lo pide en contexto.
- **Límite de 64 pendientes:** iOS solo guarda 64 notificaciones programadas por app. Por eso se programan solo las sorpresas de los próximos 3 días (ventana móvil).
- Las versiones recientes del paquete documentan cambios por el ciclo de vida **UIScene**; sigue la sección de iOS de su README.
- Para mostrar avisos con la app abierta en iOS hacen falta `presentAlert` y `presentSound` en `DarwinNotificationDetails` (ya incluidos).

---

## 7. Ejecutar las pruebas

```bash
flutter test
```

Son 8 archivos en `test/`: `falsos.dart` es el doble de pruebas y los otros 7 son pruebas. No necesitan teléfono porque el tiempo lo controla `RelojFalso` y el plugin lo sustituye `PuertoFalso`.

Para correr solo un archivo:

```bash
flutter test test/planificador_test.dart
```

**Ciclo rojo → verde (para las capturas):** cambia temporalmente una regla y corre `flutter test` para ver el **rojo**. Por ejemplo, en `lib/dominio/politica.dart` cambia `maxPorDia = 3` por `maxPorDia = 1`. Después regrésala a como estaba y vuelve a correr las pruebas para ver el **verde**.

---

## 8. Ejecutar la app

Con un emulador abierto o un teléfono conectado (con depuración USB activada):

```bash
flutter devices   # para ver los dispositivos disponibles
flutter run
```

---

## 9. Recorrido guiado en dispositivo real o emulador

Las pruebas automáticas prueban la lógica. Esta lista prueba la integración con el sistema operativo. Usa el **modo clase** (1 hora de vida = 1 minuto) para ver todo en minutos.

1. Abre la app por primera vez: **no** debe aparecer ningún diálogo de permiso.
2. Toca «Avísame cuando tenga hambre» y luego «Ahora no»: no debe salir el diálogo del sistema. Repite, toca «Sí, avísame» y ahora sí aparece. Acepta.
3. Toca «Probar una notificación ahora»: debe aparecer al instante.
4. Activa el **modo clase** y **minimiza** la app para que cuente como segundo plano. A los **6 minutos** llega el aviso de hambre y a los **8 minutos y medio**, el crítico.
5. **App cerrada:** toca el cuerpo de la notificación. La app debe abrir y resaltar el botón de comer.
6. **App cerrada:** toca el botón «Dar de comer 🦐» de la notificación. La app debe abrir y alimentar al ajolote.
7. **App en segundo plano o abierta:** toca una notificación. Debe llevarte a la pantalla correcta.
8. Da de comer y cierra la app: el aviso viejo **no** debe llegar, porque se reprogramó.
9. En Ajustes de la app (icono ⚙ arriba) apaga «Hambre del ajolote»: ya no debe llegar ningún aviso de hambre.
10. Rechaza el permiso en una instalación limpia: la app debe seguir funcionando y ofrecer «cómo activarlos».
11. **Reinicio:** reinicia el teléfono con un aviso pendiente. Debe llegar igual gracias al receptor de arranque.
12. Cambia la hora del teléfono a las 23:30 y revisa que los avisos programados nunca caigan en horas de silencio.

> **Ahorro de batería de los fabricantes:** Xiaomi, Huawei, Oppo y otras marcas agregan ahorradores de batería propios que pueden matar las alarmas de las apps en segundo plano. Si en el emulador sí llega y en el teléfono no, casi seguro es eso. Excluye la app de la optimización de batería (en Ajustes → Batería → Ajolotito → Sin restricciones). El sitio <https://dontkillmyapp.com> explica cómo hacerlo por marca.

---

## 10. Errores frecuentes

| Síntoma | Causa probable |
|---|---|
| No llega nada en Android 13+ | Falta `POST_NOTIFICATIONS` en el manifiesto o no se concedió el permiso |
| No llega en Xiaomi, Huawei, Oppo… | Ahorro de batería del fabricante |
| Llega a una hora equivocada | Zona horaria mal fijada (`tz.local`) |
| Llegan avisos duplicados | Ids aleatorios (aquí se usan ids fijos) |
| Llega un aviso de algo ya resuelto | No se canceló ni reprogramó al cambiar el estado |
| La app se cierra al tocar la notificación | Payload sin validar |
| No suena en iOS en primer plano | Faltan `presentAlert` / `presentSound` |
| Error de compilación tras actualizar el plugin | Cambió la firma: revisa `puerto_real.dart` y el CHANGELOG |
| Gradle: *requires core library desugaring* | Falta el cambio de la sección 5.2 |
| `test/widget_test.dart` falla con `MyApp` | Bórralo (sección 4, paso 3) |

---

## 11. Generar el ejecutable (APK / App Bundle)

### APK (para instalar directo en un celular Android)

```bash
flutter build apk --release
```

El archivo queda en:

```
build/app/outputs/flutter-apk/app-release.apk
```

Para instalarlo en tu celular conectado por USB:

```bash
flutter install
```

También puedes copiar `app-release.apk` al teléfono y abrirlo. Hay que permitir «instalar apps de origen desconocido».

Si quieres APK más ligeros, uno por arquitectura de procesador:

```bash
flutter build apk --release --split-per-abi
```

### App Bundle (para subir a Google Play)

```bash
flutter build appbundle --release
```

El archivo queda en:

```
build/app/outputs/bundle/release/app-release.aab
```

> Por defecto, la compilación *release* se firma con la llave de depuración. Sirve para probar y entregar en clase. Para publicar en Google Play necesitas tu propia llave de firma (ver «Build and release an Android app» en la documentación oficial de Flutter).

---

## Resumen de comandos

```bash
flutter create --project-name ajolotito --org com.example .   # carpetas nativas
flutter pub get                                               # dependencias
flutter test                                                  # pruebas
flutter run                                                   # app en emulador/teléfono
flutter build apk --release                                   # APK -> build/app/outputs/flutter-apk/app-release.apk
flutter build appbundle --release                             # AAB -> build/app/outputs/bundle/release/app-release.aab
```

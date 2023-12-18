# Встраивание Рутокенов на Android без "Панели Управления Рутокен"
## Настройка проекта
Для встраивания необходимо добавить в зависимости библиотеку rtpcscbridge (в случае использования системы сборки Gradle):
```groovy
implementation 'ru.rutoken.rtpcscbridge:rtpcscbridge:1.0.0'
```

Библиотека доступна в [репозитории Maven Central](https://search.maven.org/artifact/ru.rutoken.rtpcscbridge/rtpcscbridge).

Минимальная поддерживая версия Android - 7.0 (API Level 24).

> :warning: Для корректной работы встраивания на Android 7.0 и 7.1 (при условии, что минимальная версия вашего приложения меньше API Level 26) необходимо включить [desugaring](https://developer.android.com/studio/write/java8-support#library-desugaring).

Входная точка данного встраивания – класс **ru.rutoken.rtpcscbridge.RtPcscBridge**, который необходим для инициализации библиотеки. Для каждого процесса приложения, в котором будет происходить взаимодействие с Рутокенами, необходимо вызвать статический метод `setAppContext` и передать ему в качестве параметра Android application context. Типичное место вызова этого метода - внутри **onCreate()** у класса-наследника Application:
```java
public class App extends Application {
    @Override
    public void onCreate() {
        super.onCreate();
        RtPcscBridge.setAppContext(this);
    }
}
```

Данное встраивание поддерживает два режима использования: **ручной** и **автоматический**. **Ручной** режим предполагает явные вызовы методов инициализации и финализации библиотеки, а также прямое управление отслеживанием Рутокен с NFC. Всё это даёт полный контроль над поведением библиотеки со стороны приложения. Например, можно активировать встраивание только для некоторых из существующих в приложении Activity.
При **автоматическом** режиме встраивание становится доступным для всех существующих в приложении Activity, реализующих интерфейс [OnNewIntentProvider](https://developer.android.com/reference/androidx/core/app/OnNewIntentProvider) (методы ручного API вызываются автоматически на определенных этапах жизненного цикла приложения, см. [диаграмму](#диаграмма-жизненного-цикла-rtTransport)).

Независимо от выбранного режима работы, встраивание позволяет отслеживать подключение и отключение устройств Рутокен с помощью интерфейса **ru.rutoken.rttransport.RtTransport.PcscReaderObserver**. Отслеживание устройств будет особенно полезно при работе через КриптоПро CSP.

Включить отслеживание возможно в любой момент работы со встраиванием с помощью интерфейса `ru.rutoken.rttransport.RtTransport` и метода `addPcscReaderObserver()`, при этом подписчик получит события о всех подключенных к устройству токенах - для каждого токена вызовется метод `RtTransport.PcscReaderObserver.onReaderAdded(reader)`.

Для остановки отслеживания предусмотрен метод `RtTransport.removePcscReaderObserver()`.

> :warning: Важно: в случае, если подписка происходит внутри Activity или в любом другом компоненте, который может быть создан несколько раз за время жизни приложения, то необходимо своевременно **отписываться** во избежание утечек. Например, если подписка была произведена внутри **Activity.onStart()**, то отписаться нужно внутри **Activity.onStop()**.

### Ручной API встраивания
Для работы с Рутокенами в ручном режиме существует интерфейс **ru.rutoken.rttransport.RtTransport**, инстанс которого можно получить с помощью метода `RtPcscBridge.getTransport()`.
Рекомендуемый порядок вызовов API:
1. `RtTransport.initialize(context)` - инициализирует встраивание. Библиотека начинает отслеживать подключение и отключение Рутокенов (кроме подключений по NFC) и позволяет настроить дальнейшую работу с NFC; рекомендуется вызывать в методе **Activity.onStart()** или для сервиса - внутри **Service.onCreate()**.
2. `RtTransport.enableNfcForegroundDispatch(activity)` - включает [NFC foreground dispatching](https://developer.android.com/guide/topics/connectivity/nfc/advanced-nfc#foreground-dispatch) для данной Activity. Как и в случае с <a href="https://developer.android.com/reference/android/nfc/NfcAdapter#enableForegroundDispatch(android.app.Activity,%20android.app.PendingIntent,%20android.content.IntentFilter[],%20java.lang.String[][])">NfcAdapter.enableForegroundDispatch</a>, вызов данного метода должен происходить строго из Main потока в методе **Activity.onResume()**.
3. `RtTransport.handleNfcIntent(intent)` - обрабатывает NFC интент, полученный при прикладывании токена, игнорируя при этом интенты от не-Рутокен устройств. Этот метод необходим для дальнейшей установки соединения с токеном и должен быть обязательно вызван при получении каждого NFC интента, например, внутри **Activity.onNewIntent()**. 
4. `RtTransport.disableNfcForegroundDispatch(activity)` - выключает NFC foreground dispatching для данной Activity. Вызов данного метода должен происходить строго из Main потока в методе **Activity.onPause()**.
5. `RtTransport.finalize(context)` - завершает работу встраивания; рекомендуется вызывать в методе **Activity.onStop()** или для сервиса - внутри **Service.onDestroy()**.

### Автоматический API встраивания
Для работы с Рутокенами в автоматическом режиме существует интерфейс **ru.rutoken.rttransport.RtTransportExtension**, инстанс которого можно получить из метода `RtPcscBridge.getTransportExtension()`. Для включения режима необходимо вызвать метод `attachToLifecycle` внутри класса-наследника Application:
```java
public class App extends Application {
    @Override
    public void onCreate() {
        super.onCreate();
        RtPcscBridge.setAppContext(this);
        RtPcscBridge.getTransportExtension().attachToLifecycle(this, true);
    }
}
```
В параметры, помимо Application, можно передать флаг **useAutoNfcHandling** для включения/выключения автоматической обработки Рутокенов с NFC. По умолчанию флаг равен **true**.

> :warning: Важно: метод `attachToLifecycle` должен быть вызван именно в `Application.onCreate()`. Если метод будет вызван позднее, например, внутри коллбеков жизненного цикла Activity, то встраивание может не получить сигнал о переходе приложения в нужное состояние и не сможет активировать работу с NFC.

> :warning: Важно: автоматическая обработка Рутокенов с NFC будет происходить только на тех Activity, которые реализуют интерфейс [OnNewIntentProvider](https://developer.android.com/reference/androidx/core/app/OnNewIntentProvider).

Для полного выключения автоматического режима встраивания существует метод `RtPcscBridge.getTransportExtension().detachFromLifecycle(app)`. Если взаимодействие с Рутокеном происходит на протяжении всей работы приложения, то необходимости в таком вызове нет.

## Автоматический API и жизненный цикл приложения
Жизненный цикл автоматического API непосредственно связан с жизненными циклами всех Activity приложения. 
Вызов метода `RtTransportExtension.attachToLifecycle(app)` инициирует следующие операции внутри библиотеки:
- Добавление подписчика **ru.rutoken.rttransport.RtTransportProcessLifecycleObserver** на [жизненный цикл процесса приложения](https://developer.android.com/reference/androidx/lifecycle/ProcessLifecycleOwner#getLifecycle()).
- Добавление **ru.rutoken.rttransport.RtTransportNfcLifecycleCallbacks** в качестве [Activity Lifecycle callbacks](https://developer.android.com/reference/android/app/Application.ActivityLifecycleCallbacks) для данного приложения *при условии, что параметр useAutoNfcHandling = true и NFC адаптер физически существует*.

Ниже представлена диаграмма для иллюстрации зависимости поведения **RtTransport** от жизненного цикла Activity при автоматическом режиме встраивания (автоуправление NFC включено).
#### Диаграмма жизненного цикла RtTransport
![rttransport_auto_api_lifecycle.png](rttransport_auto_api_lifecycle.png)

После перехода **самой первой** Activity в состояние Started (событие [Lifecycle.Event.ON_START](https://developer.android.com/reference/androidx/lifecycle/Lifecycle.Event#ON_START)) вызывается инициализация RtTransport. После перехода **самой последней** Activity в состояние Stopped (событие [Lifecycle.Event.ON_STOP](https://developer.android.com/reference/androidx/lifecycle/Lifecycle.Event#ON_STOP)) происходит вызов финализации RtTransport **с задержкой**, которая помогает избежать лишней переинициализации и предусмотрена для следующих ситуаций:
- смена конфигурации (переворот экрана)
- быстрое переключение между приложениями или на Home screen (и обратно)

При включенном автоуправлении NFC дополнительно автоматически вызываются:
1. `enableNfcForegroundDispatch()` - если метод **Activity.onResume()** переопределен, то вызов происходит при `super.onResume()`, иначе - неявно во время перехода в состояние Resumed
2. `handleNfcIntent()` - если метод **Activity.onNewIntent()** переопределен, то вызов происходит при `super.onNewIntent()`, иначе - автоматически во время получения NFC интента
3. `disableNfcForegroundDispatch()` - если метод **Activity.onPause()** переопределен, то вызов происходит при `super.onPause()`, иначе - неявно во время перехода в состояние Paused

Кроме того, во время нахождения Activity в стадии Resumed библиотека реагирует на события о состоянии NFC адаптера через внутренний BroadcastReceiver:
- при переходе в состояние [NfcAdapter.STATE_ON](https://developer.android.com/reference/android/nfc/NfcAdapter#STATE_ON): вызывается **enableNfcForegroundDispatch()**
- при переходе в состояние [NfcAdapter.STATE_OFF](https://developer.android.com/reference/android/nfc/NfcAdapter#STATE_OFF): вызывается **disableNfcForegroundDispatch()**

## Требования для встраивания в зависимости от физического интерфейса токена
### USB
Для работы по интерфейсу USB в ручном режиме достаточно вызвать методы `initialize` (п. 1) и `finalize` (п. 5). В автоматическом режиме USB устройства будут обработаны без каких-либо других действий от пользователя библиотеки. При этом пользователю конечного приложения будет показан системный диалог с запросом разрешения на использование USB устройства при каждом подключении токена.  
Дополнительных runtime permission получать не требуется.
### Bluetooth
Для работы по интерфейсу Bluetooth в ручном режиме достаточно вызвать методы `initialize` (п. 1) и `finalize` (п. 5).
На Android 12+ требуется самостоятельно запросить permission [BLUETOOTH_CONNECT](https://developer.android.com/reference/android/Manifest.permission#BLUETOOTH_CONNECT).
Для работы в автоматическом режиме данное разрешение также необходимо.
### NFC
Для работы по интерфейсу NFC в ручном режиме необходимо вызвать все описанные выше методы (пункты 1-5). Для автоматического режима необходимо передать флаг **useAutoNfcHandling=true** в метод `attachToLifecycle`, либо использовать default метод `attachToLifecycle(application)`, который по умолчанию включает авто-обработку NFC.
Дополнительных runtime permissions получать не требуется.

## Работа встраивания в многопроцессных приложениях
В случае, если встраивание выполняется только в одном из нескольких процессов приложения, поведение библиотеки не будет отличаться от стандартного.
Однако корректность работы встраивания **не гарантируется** при использовании Рутокенов **одновременно** в нескольких процессах.

## Логирование
Уровень логирования в библиотеках по умолчанию - [ERROR](https://developer.android.com/reference/android/util/Log#ERROR). 
В целях отладки есть возможность расширить логирование до уровня [DEBUG](https://developer.android.com/reference/android/util/Log#DEBUG). Для этого необходимо вызвать `RtPcscBridge.enableDebugLogs()` перед началом работы со встраиванием - до `RtPcscBridge.setAppContext()`.

Расширенное логирование содержит сообщения об автоматической инициализации и финализации встраивания, а также вспомогательную информацию об обнаружении токенов по различным физическим интерфейсам.

## Конфигурация R8/ProGuard
Начиная с версии 1.2.0 библиотека содержит конфигурационный файл ProGuard, который применится автоматически при включении обфускации приложения.

Для корректной работы библиотеки версии 1.1.1 и ниже необходимо прописать следующие правила в proguard-rules.pro:
```
-dontwarn ru.rutoken.spi.SpiDevice
-dontwarn ru.rutoken.spi.SpiManager
-keep class ru.rutoken.rtcore.network.methodhandler.** { *; }
-keep class ru.rutoken.rtpcscbridge.** { *; }
-keep class * extends com.google.protobuf.GeneratedMessageLite { *; }
```

При использовании расширенного логирования (уровень DEBUG), для сохранения читаемости логов, необходимо прописать следующие правила в proguard-rules.pro:
```
-dontwarn ru.rutoken.spi.SpiDevice
-dontwarn ru.rutoken.spi.SpiManager
-keep class ru.rutoken.rttransport.** { *; }
-keep class ru.rutoken.rtcore.** { *; }
-keep class ru.rutoken.rtpcscbridge.** { *; }
-keep class * extends com.google.protobuf.GeneratedMessageLite { *; }
```

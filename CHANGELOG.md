# Changelog

Все изменения в библиотеке [rtpcscbridge](https://search.maven.org/artifact/ru.rutoken.rtpcscbridge/rtpcscbridge) будут
публиковаться в этом файле.

## [1.1.1] - 07-12-2023

### Изменено

- Ссылки на Java API в Javadoc теперь ведут на https://developer.android.com/reference,
 так как docs.oracle.com больше недоступен в России.

## [1.1.0] - 01-12-2023

### Изменено

- Теперь после вызова **RtPcscBridge.setAppContext** можно сразу успешно инициализировать библиотеку librtpkcs11ecp или
  SDK КриптоПро. Ранее для гарантированной инициализации сначала нужно было вызвать **RtTransport.initialize**.
- Убрано автоматическое логирование для методов **PcscReaderObserver.onReaderAdded** и
  **PcscReaderObserver.onReaderRemoved**.
- Сборка библиотеки осуществляется с использованием compileSdk и targetSdk 34 (Android 14).

### Исправлено

- Инстанс **PcscReaderObserver** больше не получает лишнее второе событие при отключении Рутокен Bluetooth.

## [1.0.0] - 14-11-2023

### Добавлено

- Теперь работа с Рутокен на Android не требует установки Панели управления Рутокен.
# Архитектура экосистемы ESP32: ESP32 Manager и межинтеграционный контракт

**Назначение:** актуальный архитектурный документ экосистемы ESP32. Описывает границы ответственности между ESP32 Platform Core, прикладной прошивкой, ESP32 Manager и специализированными Home Assistant интеграциями.

**Статус:** отражает код после слияния ESP32 Manager 3.0.0 и Xiaomi Sensor Manager 3.0.0 в main 21 ноября 2026 года. API Manager v1, события жизненного цикла, обязательная зависимость XSM и каскадное удаление реализованы. Универсальный API произвольных product operations пока не реализован.

## 1. Модель устройства

- **Физическое устройство** — конкретная ESP32 со стабильным аппаратным device_id. OTA и смена прикладной прошивки не меняют этот ID.
- **Продуктовый профиль** — тип приложения, работающего на ESP32 в данный момент.
- **Capabilities** — функциональные возможности, объявленные текущей прошивкой.
- **Специализированное HA-устройство** — логическое дочернее устройство продукта, например BLE-датчик, привязанное к физической ESP32.

Профиль не является аппаратной моделью и не подменяет manufacturer/model физического устройства.

## 2. Владельцы ответственности

### ESP32 Platform Core
- Общая аппаратная идентичность и платформенные механизмы прошивки.
- Общие MQTT, availability, OTA и Factory Reset.
- Не содержит логики конкретного продукта.

### Прикладная прошивка
- Объявляет product_id, capabilities и текущую версию.
- Публикует общие identity, profile и availability по MQTT-контрактам.
- Реализует продуктовые MQTT-топики, команды и телеметрию.
- Проверяет входящие команды на самой ESP32.
- Реализует Factory Reset по требованиям Core.

### ESP32 Manager (esp32_ota)
Единственный владелец физического ESP32 в Home Assistant. Отвечает за обнаружение по общей MQTT identity, общую запись Device Registry, общие MQTT-подписки identity/profile/availability/status, исходное имя и аппаратные метаданные, availability, профиль, OTA status, публичный API v1 и события жизненного цикла, OTA/restart/Factory Reset, каскадное удаление, каталог GitHub-репозиториев и credentials.

Manager не реализует прикладную логику Xiaomi, ворот, света и других продуктов.

### Специализированная Home Assistant интеграция
- Обязательно зависит от esp32_ota.
- Получает список ESP32 и общие метаданные через API Manager.
- Подписывается только на собственные продуктовые MQTT-топики.
- Создаёт только собственные продуктовые entities и child devices.
- Хранит только собственные предметные данные.
- Регистрирует cleanup handler для каскадного удаления.
- Не импортирует Python-модули Manager или других специализированных интеграций.

Подробный обязательный стандарт: [Правила разработки зависимых интеграций](DEPENDENT_INTEGRATIONS_RU.md).

## 3. Единая идентичность Home Assistant

Все интеграции используют один физический identifier:

    identifiers = {("esp32", device_id)}

Только ESP32 Manager создаёт и обновляет общую запись шлюза. Специализированные интеграции не создают альтернативную запись физической ESP32.

- device_id — стабильный 12-значный uppercase hex ID.
- name — исходное техническое имя из MQTT identity.
- name_by_user — пользовательское имя из Home Assistant Device Registry.
- Пользовательское имя не перезаписывается MQTT identity.
- Дочерние устройства продукта связываются с общей ESP32 через via_device=("esp32", device_id).
- Areas назначаются штатными средствами Home Assistant и не передаются через MQTT.

## 4. Общий MQTT-контракт

Manager владеет общими топиками:

| Назначение | Топик |
|---|---|
| Identity | esp32/<DEVICE_ID>/device/info |
| Product profile | esp32/<DEVICE_ID>/device/profile |
| Availability | esp32/<DEVICE_ID>/availability |
| OTA status | esp32/<DEVICE_ID>/ota/status |

Identity, profile и availability публикуются прошивкой как retained-состояние в соответствии с контрактом Core. Специализированные интеграции не подписываются на эти топики и не разбирают их payload.

### Product profile v1

Топик: esp32/<DEVICE_ID>/device/profile

Пример payload:

    {
      "schema_version": 1,
      "product_id": "xiaomi_ble_gateway",
      "firmware_version": "2.0.0",
      "capabilities": ["ble_gateway", "xiaomi_lywsd03"]
    }

Обязательные поля: schema_version, product_id, firmware_version, capabilities. Версия профиля должна быть согласована с sw_version в identity.

Профиль публикуется при MQTT connect/reconnect. Перед OTA Manager очищает retained profile tombstone-сообщением. Пока новая прошивка не опубликовала актуальный профиль, продуктовая интеграция считает профиль неизвестным и переводит зависимые entities в unavailable, но не удаляет их.

Неизвестная capability не должна ломать обработку известных. Неизвестный product_id игнорируется интеграциями, которые его не поддерживают.

## 5. Публичный API ESP32 Manager v1

API доступен после успешного setup Manager:

    hass.data["esp32_ota"]["api"]

| Элемент | Описание |
|---|---|
| version | Целое число 1 |
| get_device(device_id) | Snapshot устройства или None |
| get_devices() | Tuple snapshots идентифицированных устройств |
| register_remove_handler(domain, async_handler) | Регистрация cleanup handler; возвращает unsubscribe-функцию |

Snapshot содержит поля: device_id, name, name_by_user, manufacturer, model, sw_version, available, availability_received, identity_received, profile, product_id, capabilities, operations.

Snapshot является отделённой копией состояния Manager. Не изменяйте его и не используйте внутренние структуры Manager напрямую.

### События Home Assistant

| Событие | Payload |
|---|---|
| esp32_ota_device_added | device: snapshot |
| esp32_ota_device_updated | device: snapshot |
| esp32_ota_device_removing | device_id |
| esp32_ota_device_removed | device_id |

Потребитель сначала получает начальное состояние через get_devices(), затем слушает события. Обработчики должны быть идемпотентными.

### Ограничение API v1

API v1 не предоставляет универсального метода проверки/выполнения произвольных product operations. Поле operations snapshot — только данные профиля. Оно не означает, что Manager маршрутизирует произвольные команды.

OTA, restart, Factory Reset и удаление выполняются сервисами Manager. Продуктовые команды до появления общего operations API остаются частью собственного документированного протокола специализированной интеграции.

## 6. Жизненный цикл и каскадное удаление

Полное удаление физической ESP32 выполняется через действие удаления общей записи Home Assistant или сервис esp32_ota.remove_device.

Последовательность Manager:
1. Публикует esp32_ota_device_removing.
2. Если устройство available, публикует Factory Reset на esp32/<id>/system/factory_reset с QoS 1 и retain=false.
3. Если устройство offline, Factory Reset не отправляется, не ставится в очередь и не повторяется при reconnect.
4. Последовательно вызывает зарегистрированные cleanup handlers специализированных интеграций.
5. Очищает собственные retained identity, profile, availability и OTA status.
6. Удаляет общие entities и запись физического устройства из HA Registry.
7. Публикует esp32_ota_device_removed.

Успешная MQTT-публикация не доказывает, что физический Factory Reset завершился. Если устройство было offline, его локальная память не очищена; позднее оно может быть обнаружено заново по тому же hardware ID. HA-данные интеграций к этому моменту должны быть очищены, старые пользовательские настройки не должны восстанавливаться.

### Ответственность потребителя

Каждая специализированная интеграция регистрирует async handler. Он очищает только свои данные указанного device_id: child devices, entities, persistent storage, bindings, runtime caches, pending commands, tombstones и собственные retained MQTT-топики.

Handler должен быть идемпотентным, завершать очистку до возврата и не затрагивать другие ESP32. Ошибка одного handler логируется Manager; остальные обработчики продолжают выполняться. Автоматического отката частично выполненной очистки нет.

## 7. Общие сервисы Manager

| Сервис | Назначение |
|---|---|
| esp32_ota.update | Универсальный OTA |
| esp32_ota.restart | Перезагрузка |
| esp32_ota.factory_reset | Factory Reset |
| esp32_ota.save_token | Сохранение общего GitHub token |
| esp32_ota.remove_device | Каскадное удаление |

Общие команды публикуются Manager адресно с QoS 1 и retain=false. Специализированные интеграции не публикуют общие system/OTA команды напрямую.

Выбор репозитория, релиза и asset относится к текущей OTA-операции, а не к постоянному назначению устройства.

## 8. Capabilities и будущий контракт operations

Capabilities описывают области функций (например, gate_control, light_control). Operations должны описывать конкретное действие, параметры и результат.

Расширенный декларативный профиль, общий dispatcher, валидация схем и универсальный транспорт операций — отдельный будущий этап. До согласования версии протокола нельзя считать такой API реализованным или проектировать интеграции на несуществующий метод.

Прошивка всегда повторно валидирует входные параметры. Проверка на стороне Home Assistant не заменяет проверку на устройстве.

## 9. Правила расширения

1. Использовать общие механизмы ESP32 Platform Core.
2. Объявить стабильный product_id, capabilities и профиль.
3. Не менять аппаратный device_id при OTA или смене приложения.
4. Реализовать и документировать собственные product MQTT topics.
5. Создать специализированную HA интеграцию с обязательной зависимостью от Manager.
6. Использовать только API/events Manager для общих сведений об ESP32.
7. Реализовать полный cleanup handler.
8. Проверить discovery, reconnect, profile changes, OTA, удаление online/offline и повторное обнаружение.

Новая специализированная интеграция не должна зависеть от другой специализированной интеграции. Подробный чек-лист и требования к коду находятся в [DEPENDENT_INTEGRATIONS_RU.md](DEPENDENT_INTEGRATIONS_RU.md).

## 10. Проверенное состояние на 21 ноября 2026

- ESP32 Manager 3.0.0 и Xiaomi Sensor Manager 3.0.0 слиты в main.
- Manager владеет общей identity/profile/availability MQTT-обработкой.
- XSM объявляет esp32_ota в manifest.json, получает начальные snapshots через API v1 и обновления через события.
- XSM подписывается только на собственные Xiaomi product topics.
- XSM регистрирует cleanup handler для каскадного удаления.
- Manager вызывает Factory Reset только при доступном устройстве; deferred reset отсутствует.
- При удалении Manager очищает общие retained-топики и запись HA; потребитель очищает свои retained-топики и дочерние сущности.
- CI обоих репозиториев прошёл перед слиянием. Сквозная проверка установленной пары интеграций в реальном Home Assistant остаётся отдельным эксплуатационным этапом.

## Связанные проекты

- [ESP32 Manager](https://github.com/lavrenovalexsandr-cmyk/esp32-integration-manager)
- [Xiaomi Sensor Manager](https://github.com/lavrenovalexsandr-cmyk/esp32-integration-xiaomi-sensor-manager)
- [ESP32 Platform Core](https://github.com/lavrenovalexsandr-cmyk/esp32-platform-core)

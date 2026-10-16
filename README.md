# Звіт з практичної роботи 3.1
**Тема:** Мульти-протокольна погодна станція в OpenHAB емуляторі  
**Середовище виконання:** macOS, OpenHAB, Visual Studio Code

---

## 1. Проектування архітектури системи

### 1.1 Файлова структура конфігурації
Розроблено модульну структуру в директорії `conf/`:
`services/mqtt.cfg`  - параметри підключення до локального MQTT брокера, авторизації, QoS, Retain та Last Will and Testament.
* `services/http.cfg`  - таймаути та інтервали кешування HTTP клієнта.
* `services/bluetooth.cfg`  - конфігурація віртуального адаптера `hci0` та інтервалів BLE discovery.
* `things/weather_station.things`  - опис мостів, віртуальних пристроїв та каналів для трьох протоколів.
* `items/weather_station.items`  - прив'язка точок даних, типізація одиниць вимірювання та групування.
* `rules/protocol_emulators.rules`  - генерація телеметрії за розкладом Quartz Cron, симуляція відмов та автоматичне керування виконавчими пристроями.
* `rules/analytics_integration.rules`  - міжпротокольна агрегація даних, розрахунок середніх показників, детекція розбіжностей та системні команди.
* `rules/validation_suite.rules`  - автоматизоване функціональне тестування всіх компонентів системи.

### 1.2 Обгунтування технічних параметрів протоколів
1. **MQTT:**
   * Використано стандартний порт `1883` та ідентифікатор клієнта `oh_mqtt_bridge_01`.
   * Рівень надійності `QoS = 1` гарантує доставку даних від сенсорів щонайменше один раз без надлишкового мережевого трафіку `QoS = 2`.
   * Прапорець `retain = true` забезпечує збереження останнього відомого стану погоди на брокері.
   * Налаштовано топік `weather_lab/system/lwt_status` для миттєвого виявлення втрати зв'язку з вузлом.
2. **HTTP REST API:**
   * Встановлено `timeout = 5000` мс для захисту потоків опитування від зависання.
   * Автентифікація реалізована через HTTP-заголовок `Authorization: Bearer <token>`.
   * Для вилучення метрик із JSON-відповідей застосовано трансформацію `JSONPATH` (`$.metrics.temp`, `$.metrics.humidity`, `$.network.online_nodes`).
3. **Bluetooth Low Energy (BLE):**
   * Задіяно стандартні 16-бітні UUID профілю GATT Bluetooth SIG: сервіс `0x181A` з характеристиками `0x2A6E` для температури та `0x2A6F` для вологості, а також сервіс `0x180F` з характеристикою `0x2A19` для рівня заряду.
   * Для маячка близькості **iBeacon** емулюється передача 128-бітного `UUID`, параметрів `Major=100`, `Minor=5` та динамічного рівня сигналу `RSSI`.

## 2. Реалізація віртуальних Things та Items
* Розроблено 3 незалежні підсистеми у файлі `weather_station.things` з розділенням на сенсорні та актуаторні канали (`number`, `string`, `switch`, `dimmer`).
* У файлі `weather_station.items` використано строгу типізацію розмірностей (`Number:Temperature`, `Number:Dimensionless`, `Number:Pressure`, `Number:Speed`) та ієрархічне групування (`gWeatherSensors`, `gSmartDevices`, `gSystemStatus`, `gSignalQuality`, `gAnalytics`).

## 3. Логіка емуляції та аналітична інтеграція
* **MQTT Емулятор:** генерує плавний температурний тренд, вологість, атмосферний тиск та варіації RSSI у діапазоні `-30...-70 dBm`. Реалізовано автоматику: при `T > 25°C` вмикається вентилятор охолодження, а при падінні тиску `< 1013 hPa` (похмура погода) яскравість розумної лампи збільшується до `80%`.
* **HTTP REST API Емулятор:** формує валідну JSON-структуру, моделює час затримки мережі (`25...240 ms`), кількість активних вузлів (`6...10`) та 5% імовірність помилки `504 Gateway Timeout`.
* **BLE Емулятор:** моделює низькочастотне опитування GATT-характеристик, поступовий розряд батареї (`-0.2%` за цикл), коливання RSSI маячка та 8% імовірність тимчасової втрати GATT-з'єднання.
* **Модуль аналітики:** у реальному часі при зміні температури на будь-якому з протоколів обчислює середню температуру (`Analytics_Avg_Temp`), середню вологість (`Analytics_Avg_Hum`), максимальне відхилення між протоколами (`Analytics_Temp_Spread`) та контролює перелік активних каналів зв'язку (`[MQTT HTTP BLE]`).

## 4. Результати функціонального тестування та валідації
Під час тестування у середовищі OpenHAB за допомогою сценарію `validation_suite.rules` (команда `openhab:send Sim_Test_Status RUN_TESTS`) та генератора звітів (`openhab:send Sim_System_Cmd PRINT_REPORT`) отримано такі результати в журналі подій:

```log
17:46:44.848 [INFO ] [g.openhab.core.model.script.TestSuite] - Старт автоматичної валідації усіх протоколів
17:46:44.853 [INFO ] [rg.openhab.core.model.script.MQTT_Cmd] - Отримано команду скидання MQTT вузла. Перезавантаження
17:46:45.855 [INFO ] [rg.openhab.core.model.script.MQTT_Cmd] - MQTT вузол успішно перезапущено.
17:46:46.352 [INFO ] [g.openhab.core.model.script.TestSuite] - 1) Перевірка MQTT: Успішно
17:46:46.353 [INFO ] [g.openhab.core.model.script.TestSuite] - 2) Перевірка HTTP API: Успішно
17:46:46.354 [INFO ] [g.openhab.core.model.script.TestSuite] - 3) Перевірка BLE: Успішно
17:46:46.355 [INFO ] [g.openhab.core.model.script.TestSuite] - 4) Перевірка інтеграції та аналітики: Успішно
17:46:46.356 [INFO ] [g.openhab.core.model.script.TestSuite] - Усі тести пройдено успішно
17:46:46.358 [INFO ] [openhab.event.ItemStateChangedEvent ] - Item 'Sim_Test_Status' changed from RUN_TESTS to VERIFIED_SUCCESS
17:46:49.216 [INFO ] [org.openhab.core.model.script.SysCtrl] - Звіт системи MQTT: T=22 °C, RSSI=-33 dBm HTTP: Статус=online, Затримка=200 ms, Пристроїв=8 | BLE: T=21.23564201571185 °C, Батарея=98.19999999999997 %%, Beacon RSSI=-81 dBm Аналітика: Середня T=21.596190271402264 °C, Відхилення=0.76435798428815, Пакетів=18

<img width="1025" height="640" alt="image" src="https://github.com/user-attachments/assets/5c9d8bba-16f7-42c9-900c-0f277ca58703" />
<img width="769" height="479" alt="image" src="https://github.com/user-attachments/assets/4816010b-6c1b-484a-9798-6a1030a45424" />

- [Умный дом на основе API Telegram](#smart_home)
- [Прошивка](#firmware)
- - [Безопасность соединений](#secure)
- - - [Соединение сервер-шлюз](#server-gateway)
- - - [Соединение сервер-устройства](#gatevay-device)
- - [Конвертер из Юникода](#unicode)
- - [Структуры для связи по ESP-NOW](#structure)

<a id="smart_home"></a>
# Умный дом на основе API Telegram

Принцип работы системы умного дома grib tehcnology на основе API Telegram.

1. Пользователь создаёт своего бота в Telegram при помощи BotFather ([Создание бота в Telegram](https://github.com/gleb-zhukov/myewwt_clock#bot)).
2. Получив токен созданного бота, пользователь сообщает [розетке](https://github.com/gleb-zhukov/smart_socket_and_relay), настроенной в режим шлюза, токен бота а также SSID и пароль Wi-Fi сети ([Настройка](https://github.com/gleb-zhukov/smart_socket_and_relay#socket_setup)).
3. Отправляя сообщение боту, оно обрабатывается шлюзом.
4. Если в систему добавлены дополнительные устройства — умные розетки, реле и т.д., команда от бота обрабатывается сначала шлюзом, затем передаётся подключенным в систему устройствам с применением технологии [ESP-NOW](https://www.espressif.com/en/products/software/esp-now/overview).

![Frame 81](https://user-images.githubusercontent.com/84660518/178150352-42d2466f-7262-4145-9d54-b2dfeb7c67ef.png)


<a id="firmware"></a>
# Прошивка

Проект начал свое существование в среде Arduino IDE с применением SDK от компании Espressif для микроконтроллера ESP-12F, затем получил развитие в среде VS Code с применением набора инструментов PlatformIO.

Большая часть кода проекта проккоментирована и разбита на удобные части в виде нескольких файлов: 
* основная, с общим циклом и объявлением глобальных переменных и некоторых функций,
* для функций, вызываемых при отправке и приеме сообщений через [ESP-NOW](https://www.espressif.com/en/products/software/esp-now/overview),
* блок с функциями для работы сервера ([captive portal](https://ru.wikipedia.org/wiki/Captive_portal)),
* функция-обработчик сообщений с бота Telegram,
* блок с прочими системнымми функциями: отсчёт времени, обвновление прошивки, синхронизация времени и т.д.


<a id="secure"></a>
## Безопасность соединений

В системе умного дома grib technology используются два типа соединений:

1. От сервера Telegram к шлюзу.
2. От шлюза к прочим устройствам системы (розетки, реле, выключатели и т.д.).

<a id="server-gateway"></a>
### Соединение сервер-шлюз

Все аспекты безопасности самого API Telegram, а конкретно защита профиля, защита чатов и т.д. описаны [здесь](https://core.telegram.org/#security).

Между серверами Telegram и шлюзом устанавливается соединение по защищённому каналу связи, это гарантирует протокол SSL, т.е. сами сервера Telegram не дают возможности подключиться и получить запрос от бота, без использования SSL шифрования. 

В коде прошивки благодаря [BearSSL](https://bearssl.org/) (реализация протокола SSL на языке C) создаётся клиент, с помощью которого мы имеем возможность получать и отправлять запросы к API Telegram, конкретно к указанному в прошивке (при настройке шлюза) боту. 

Чтобы другой пользователь, получивший ссылку на бота, управляющего умным домом, или обнаружив его в поиске не смог получить доступ к нему, в коде прошивки создаётся white list, где находятся ID пользователей. ID — уникальный номер аккаунта в Telegram, его невозможно подделать и изменить. Таким образом, при настройке шлюза, доступ получает первый человек, обратившийся к боту, его ID сохраняется в энергонезависимой памяти, и возможность обмена сообщений закрепляется за данным пользователем, сообщения от других пользователей просто отсеиваются. 

Также в коде прошивки есть возможность выдать правда доступа к боту другим пользователям — достаточно в настройках выбрать меню "Добавить пользователя" и указать его ID.

Подводя итог, можно сделать вывод о безопасности подобного соединения. Чтобы иметь несанкционированный доступ к боту — придется получить доступ к аккаунту владельца бота, что сделать довольно сложно, а при должных настройках безопасности Telegram профиля — невозможно.

<a id="gateway-device"></a>
### Соединение шлюз-устройства

Внутри дома прочие устройства, такие как розетки, реле, выключатели и т.д. не подключены к Wi-Fi и не связаны с интернетом (за исключеним попыток связи с сервером для обновления прошивки раз в n часов)

Связь между шлюзом и другими устройствами передаётся с помощью [ESP-NOW](https://www.espressif.com/en/products/software/esp-now/overview) — встроенного в SDK от Espressif метода передачи данных. Данное соединение одноранговое, не требует времязатратного "рукопожатия" а также обеспечено возможностью применения протокола [CCMP](https://ru.wikipedia.org/wiki/CCMP), алгоритма [AES-128](https://ru.wikipedia.org/wiki/AES_(%D1%81%D1%82%D0%B0%D0%BD%D0%B4%D0%B0%D1%80%D1%82_%D1%88%D0%B8%D1%84%D1%80%D0%BE%D0%B2%D0%B0%D0%BD%D0%B8%D1%8F)).
Связь между устройствами устанавливается посредством отправки пакета на MAC-адрес устройства, либо с использованием широковещательной передачи. 

В документации указано ограниченное число соединений:

Максимум 20 пар, включая зашифрованные, поддерживаются на одном устройстве, включая зашифрованные пары.
Максимум 10 зашифрованных пар поддерживаются в режиме Station.
Максимум 6 в режиме SoftAP или SoftAP + Station.

А также указано о необходимости регистрации устройств на этапе компиляции прошивки:
```cpp
 esp_now_add_peer(broadcastAddress, ESP_NOW_ROLE_COMBO, 1, NULL, 0);
  ```

Несмотря на это, было установлено, что без выполнения подобных рекомендаций, можно ограничиться лишь объявленим функций, вызываемых при принятии и отправке пакета:
```cpp
esp_now_register_recv_cb(OnDataRecv);
esp_now_register_send_cb(OnDataSent);
```  
и непосредственно функцией отправки пакета:
```cpp
esp_now_send(mac_addr, (uint8_t *) &to_switch, sizeof(to_switch));
  ```
 ограничиваясь, таким образом, лишь памятью ESP-12f, где хранятся MAC адреса добавленных устройств.
 
Также стоит отметить, чтобы функции [ESP-NOW](https://www.espressif.com/en/products/software/esp-now/overview) отрабатывались без ошибок, оба устройства должны быть установлены на один и тот же канал Wi-Fi. Устройства время от времени сканируют сеть, выбирают точку доступа шлюза (grib socket), или точку доступа домашней сети Wi-Fi при недоступности сети grib socket, и настраиваются на её канал. При этом подключение к самой точке доступа не требуется. 

<a id="structure"></a>
# Структуры для связи по ESP-NOW

Управление с помощью [ESP-NOW](https://www.espressif.com/en/products/software/esp-now/overview) происходит благодаря созданию структур, в которых присутствуют несколько основных переменных. ID, value, SSID и password.

В проекте аналогичные устройства в разных корпусах: на данный момент реле и розетки.
Далее все подобные устройства для удобства будут называться реле.

Устройства в системе работают по принципу master-slave,
    где master - устройство, выходящее в интернет и работающее с Telegram.
    slave - подчиненные устройства, принимающие команды от master
    подчиненные устройства связываются с master по протоколу ESP-NOW.

Структуры для связи между устройствами:
to_device (на принимающем устройстве from_device)


Структуры с переменными в коде: 
```cpp
  struct {
  byte ID; //ID устройства (розетка, выключатель или сигнализация)
  byte command; //команда устройства
  char router_ssid[35] = "";
  char router_pass[65] = "";
  uint8_t mac_address[6];
  } to_device

  struct {
  byte ID; //ID устройства (розетка, выключатель или сигнализация)
  byte command; //команда устройства
  char router_ssid[35] = "";
  char router_pass[65] = "";
  uint8_t mac_address[6];
  } from_device
  
  ```
  
  SSID и password необходимы для обмена между устройствами данными об основной точке доступа Wi-Fi, чтобы устройство могло настроиться на канал и/или соединиться с сервером для обновления прошивки.

  ID:
  0 - от master реле
       command:
       0 - запрос на включение/выключение
       1 - запрос состояния
       2 - ответ на запрос о настройке
       3 - отчёт о получении сообщения

  1 - от slave реле
       command:
       0 - выключено
       1 - включено
       2 - запрос о настройке

  2 - от выключателя
       command:
       0 - запрос на включение/выключение
       1 - запрос на настройку
       2 - отчёт об успешной настройке

  3 - от датчика открытия двери
       command:
       0 - сигнал тревги
       1 - запрос на настройку


<a id="unicode"></a>
## Конвертер из Юникода

Помимо прочих, в проекте используются библиотека FastBot от AlexGyver, для удобного парсинга сообщений от бота. Функционал MYEWWT предполагает использование смайликов и киррилических символов, но при обращении с API Telegram обработчик получает Юникод, поэтому библиотека FastBot была доработана — дописана функция конвертации символов Юникода в UTF-8.

В википедии найдены [алгоритмы преобразования юникода в utf-8](https://ru.wikipedia.org/wiki/UTF-8#%D0%90%D0%BB%D0%B3%D0%BE%D1%80%D0%B8%D1%82%D0%BC_%D0%BA%D0%BE%D0%B4%D0%B8%D1%80%D0%BE%D0%B2%D0%B0%D0%BD%D0%B8%D1%8F). Нужно определить, сколько октетов используется, в соответствии с этим создать шаблоны в битовом выражении, далее побитно включить или выключить нужные биты в шаблоне, затем с помощью логического умножения взять маской биты и включить, после этого преобразовать в последовательность символов HEX системы. В результате успешно получаем русский, индийский и прочие алфавиты.

С другими символами и смайлами ситуация обстоит немного иначе: UTF-8 может занимать и 4 бита, кодируя в принципе весь диапазон юникода, но во входящем сообщении от Telegram мы видим [суррогатные пары UTF-16](https://ru.wikipedia.org/wiki/UTF-16). Если коротко, цельные байты UTF-8 разбиваются на две пары — верхняя и нижняя часть суррогатной пары. Её нам нужно восстановить в цельный юникод по формуле, чтобы затем преобразовать в уже знакомую нам последовательность HEX в utf-8.
  
Большая благодарность Alex Gyver, который сильно сократил и оптимизировал код: 
  
```cpp  
    String convertUnicode(String &uStr) {
  String out;
  out.reserve(uStr.length() / 3);
  int32_t uBytes, buf = 0;
  char x0[] = "\0x";
  for (uint16_t i = 0; i < uStr.length(); i++) {
    if (uStr[i] != '\\') out += uStr[i];
    else {
      switch (uStr[++i]) {
        case '/': out += uStr[i]; break;
        case 'n': out += '\n'; break;
        case 'u':
          uBytes = strtol(uStr.c_str() + i + 1, NULL, HEX);
          i += 4;
          if ((uBytes >= 0xD800) && (uBytes <= 0xDBFF)) buf = uBytes;
          else if ((uBytes >= 0xDC00) && (uBytes <= 0xDFFF)) {
            uBytes = (0x10000 + ((buf - 0xD800) * 0x0400) + (uBytes - 0xDC00));
            out += (char)(0b11110000 | ((uBytes >> 18) & 0b111));
            out += x0;
            out += (char)(0b10000000 | ((uBytes >> 12) & 0b111111));
            out += x0;
            out += (char)(0b10000000 | ((uBytes >> 6) & 0b111111));
            out += x0;
            out += (char)(0b10000000 | (uBytes & 0b111111));
          } else if (uBytes < 0x800) {
            out += (char)(0b11000000 | ((uBytes >> 6) & 0b11111));
            out += x0;
            out += (char)(0b10000000 | (uBytes & 0b111111));
          } else if (uBytes >= 0x800) {
            out += (char)(0b11100000 | ((uBytes >> 12) & 0b1111));
            out += x0;
            out += (char)(0b10000000 | ((uBytes >> 6) & 0b111111));
            out += x0;
            out += (char)(0b10000000 | (uBytes & 0b111111));
          }
          break;
      }
    }
  }
  return out;
}

```

# Работа со светодиодной лентой

> **Note** Документация для версий [образа](image.md), начиная с **0.20**. Для более ранних версий см. [документацию для версии **0.19**](https://github.com/CopterExpress/clover/blob/v0.19/docs/ru/leds.md).

Адресуемая RGB-светодиодная лента типа *ws281x*, которая входит в наборы "Клевер", позволяет выставлять произвольные 24-битные цвета на каждый из отдельных светодиодов. Это позволяет сделать полет Клевера более ярким, а также визуально получать информацию о полетных режимах, этапе выполнения пользовательской программы и других событиях.

<img src="../assets/clever-led.png" class="center" width=600>

На образе [для RPi](image.md) предустановлены необходимые модули для работы с лентой. Они позволяют:

* управлять эффектами/анимациями на ленте;
* управлять лентой на низком уровне (переключением цветов отдельных светодиодов);
* настраивать реакцию ленты на полетные события.

> **Caution** Обратите внимание, что светодиодную ленту нужно питать от стабильного источника энергии. Если вы подключите питание напрямую к Raspberry, то это создаст слишком большую нагрузку на ваш микрокомпьютер. Для снятия нагрузки с Raspberry можно подключить питание к преобразователю BEC.

## Высокоуровневое управление лентой

1. Для работы с лентой подключите ее к питанию +5v – 5v, земле GND – GND и сигнальному порту DIN – GPIO21. Обратитесь [к инструкции по сборке](assemble_4.md#Подключение-светодиодной-ленты-к-Raspberry-Pi) для подробностей.
2. Включите поддержку LED-ленты в файле `~/catkin_ws/src/clover/clover/launch/clover.launch`:

    ```xml
    <arg name="led" default="true"/>
    ```

3. Настройте параметры подключения ленты *ws281x* в файле `~/catkin_ws/src/clover/clover/launch/led.launch`. Необходимо ввести верное количество светодиодов в ленте и GPIO-пин, использованный для подключения (если он отличается от *GPIO21*):

    ```xml
    <param name="led_count" value="30"/>  <!-- количество светодиодов в ленте -->
    <param name="gpio_pin" value="21"/>   <!-- GPIO-пин для подключения -->
    ```

Высокоуровневое управления лентой позволяет управлять текущим эффектом (анимацией) на ленте. Для этого используется ROS-сервис `/led/set_effect`. Параметры сервиса:

* `effect` – название необходимого эффекта.
* `r`, `g`, `b` – цвет эффекта в формате [RGB](https://ru.wikipedia.org/wiki/RGB). Значения изменяются от 0 до 255.

Список доступных эффектов:

* `fill` (или пустая строка) – залить всю ленту цветом;
* `blink` – мигание цветом;
* `blink_fast` – ускоренное мигание цветом;
* `fade` – плавное перетекание в цвет;
* `wipe` – "надвигание" нового цвета;
* `flash` – быстро мигнуть цветом 2 раза и вернуться к предыдущему эффекту;
* `rainbow` – переливание ленты цветами радуги;
* `rainbow_fill` – переливать заливку по цветам радуги.

Пример работы с сервисом из Python:

```python
import rospy
from clover.srv import SetLEDEffect

rospy.init_node('flight')

set_effect = rospy.ServiceProxy('led/set_effect', SetLEDEffect)  # define proxy to ROS-service

set_effect(r=255, g=0, b=0)  # fill strip with red color
rospy.sleep(2)

set_effect(r=0, g=100, b=0)  # fill strip with green color
rospy.sleep(2)

set_effect(effect='fade', r=0, g=0, b=255)  # fade to blue color
rospy.sleep(2)

set_effect(effect='flash', r=255, g=0, b=0)  # flash twice with red color
rospy.sleep(5)

set_effect(effect='blink', r=255, g=255, b=255)  # blink with white color
rospy.sleep(5)

set_effect(effect='rainbow')  # show rainbow
```

Также лентой можно управлять из командной сроки (Bash):

```bash
rosservice call /led/set_effect "{effect: 'fade', r: 0, g: 0, b: 255}"
```

```bash
rosservice call /led/set_effect "{effect: 'rainbow'}"
```

## Настройка реакции ленты на события

Клевер умеет показывать LED-лентой текущее состояние полетного контроллера и сигнализировать о событиях. Данная функция настраивается в файле `~/catkin_ws/src/clover/clover/launch/led.launch` в разделе *events effects table*. Пример настройки:

```xml
startup: { r: 255, g: 255, b: 255 }
connected: { effect: rainbow }
disconnected: { effect: blink, r: 255, g: 50, b: 50 }
<!-- ... -->
```

В левой части таблицы указывается событие, на которая лента должна среагировать. В правой части указывается эффект (анимация), который необходимо включить при возникновении события. Список поддерживаемых событий:

* `startup` – запуск всех систем Клевера;
* `connected` – успешное подключение к полетному контроллеру;
* `disconnected` – разрыв связи с полетным контроллером;
* `armed` – переключение полетного контроллера в состояние Armed;
* `disarmed` – переключение полетного контроллера в состояние Disarmed;
* `stabilized`, `acro`, `rattitude`, `altctl`, `posctl`, `offboard`, `mission`, `rtl`, `land` – переключение полетных режимов;
* `error` – возникновение ошибки в ROS-нодах или полетном контроллере (*ERROR*-сообщение в топике `/rosout`);
* `low_battery` – низкий заряд батареи (порог настраивается в параметре `threshold`).

> **Note** Для корректной работы сигнализации LED-лентой о низком заряде батареи необходимо корректная [калибровка электропитания](power.md#Калибровка-делителя-напряжения).

Для того, чтобы отключить реакцию светодиодной ленты на события, установите аргумент `led_notify` в файле `~/catkin_ws/src/clover/clover/launch/led.launch` в значение `false`:

```xml
<arg name="led_notify" default="false"/>
```

## Низкоуровневое управление лентой

Для управления отдельными светодиодами используется ROS-сервис `/led/set_leds`. В параметрах задается массив номеров и RGB-цветов светодиодов, которые необходимо переключить.

Пример работы с сервисом из Python:

```python
import rospy
from led_msgs.srv import SetLEDs
from led_msgs.msg import LEDStateArray, LEDState

rospy.init_node('flight')

set_leds = rospy.ServiceProxy('led/set_leds', SetLEDs)  # define proxy to ROS service

# switch LEDs number 0, 1 and 2 to red, green and blue color:
set_leds([LEDState(0, 255, 0, 0), LEDState(1, 0, 255, 0), LEDState(2, 0, 0, 255)])
```

Сервис можно использовать из командной строки:

```bash
rosservice call /led/set_leds "leds:
- index: 0
  r: 50
  g: 100
  b: 200"
```

При использовании ленты в ROS-топике `/led/state` публикуется текущие цвета светодиодов. Просмотр топика из командной строки:

```bash
rostopic echo /led/state
```

# Intensa.logger #
Инструмент для логирования данных в файл и оповещения о проблемах.
Можно использовать для дебага данных при разработке и логирования "узких" мест в проекте.
Реализован функционал почтовых оповещений о критических ошибках в проекте. 

#Запуск тестов#
./vendor/bin/phpunit tests

# Установка и реализация #

### Установка модуля ###

**Ограничения: для работы модуля требуется версия php >=5.6**

Для установки модуля нужно скопировать файлы модуля в директорию /local/modules/ или /bitrix/modules/ (зависит от проекта).

Далее заходим в административную часть проекта и устанавливаем модуль

![1538638880521.jpg](https://i.imgur.com/lz6wUaM.jpg)

*При установке модуля производятся следующие действия:*

* Создается основная папка для логов (права 0777) с именем определенным в файле настроек, флаг LOG_DIR

* Создается тип почтового события. Символьный код задается в файле настроек, флаг CEVENT_TYPE

* Создается шаблон почтового события. Символьный код задается в файле настроек, флаг CEVENT_MESSAGE

*Возможные проблемы при установке:*

* Проблема с получением настроек из файла logger.config.php - Если такового не имеется в папке модуля, следует его создать. Структуру файла приведена ниже.

* Ошибка создания основной директории для логов - установщик не смог создать основную директорию для логов. Нужно проверить имеется ли права на создание файлов и папок в корневой директории проекта

* Проблема с правами директории - для корректной работы модуля у директории должны быть права 0775
* Также возможны ошибки при создании почтовых оповещений


### Классы ###
class \Intensa\Logger\ILog - класс реализующий в себе основной функционал модуля.

class \Intensa\Logger\ILogAlert - класс реализующий функционал для оповещения разработчиков по средствам почтовых событий.

class \Intensa\Logger\Settings - синглтон для получения данных из файла настроек

### Файл настроек logger.config.php###

Все настройки модуля располагаются в файле logger.config.php

```

<?php
return $config = [
    'LOG_DIR' => '/logs/', // основная директория для логов
    'LOG_FILE_EXTENSION' => '.log', // расширение файлов лога
    'DATE_FORMAT' => 'Y-m-d H:i:s', // формат времени
    'USE_DELIMITER' => true, // флаг определяющий нужно ли добавлять разделители в файл лога
    'USE_BACKTRACE' => true, // флаг определяющий нужно ли добавлять в запись лога информация о месте вызова метода
    'DEV' => true, // флаг определяет нужно ли записывать данные при вызове метода debug()
    'CEVENT_TYPE' => 'INTENSA_LOGGER_ALERT', // символьный код типа почтового события
    'CEVENT_MESSAGE' => 'INTENSA_LOGGER_FATAL_TEMPLATE', // символьный код шаблона для почтового события
    //@todo: CP1251 нужно бы выпилить, и попробовать сделать автоматическое определение
    'CP1251' => true, // флаг определяющий, что нужно записывать результат в кодировке cp1251
    'DEFAULT_EMAIL' => 'i.shishkin@intensa.ru' //email для оповещений.
    // Если нужно отправлять сообщения сразу на несколько адресов нужно перечистиль их через запятую.
];

```

Конфиг представляет из себя обычный .php файл возвращающий массив с настройками.
Для работы данными конфига используется класс  \Intensa\Logger\Settings
*Пример использования* 

```

$this->settings = Settings::getInstance();

```
Структура хранения логов:

* Все логи хранятся в основной директории(DIR_LOG определяется в файле конфига). /logs/

* Для каждого дня создается свой директория. /logs/2018-10-01/

* Критические проблемы зафиксированные уровнем fatal попадают в отдельную директорию errors. /logs/2018-10-01/errors/

* Имя файла лога определяется символьным кодом (задается в конструкторе объекта логгера) и расширением файла (LOG_FILE_EXTENSION определяется в файле конфига)  /logs/2018-10-01/catalog_info.log

* Если не указать код логгера, то данные будут записаны в общий файл /logs/2018-10-01/common.log

Функционал имеет 5 уровней логирования:

* debug
* info
* warning
* error
* fatal

*Особенности уровней логирования и способы их применения:*
При вызове fatal данные будут записываться в отдельную папку {LOG_DIR}/errors/. Так же при на почту разработчика будет отправлено письмо с цепочкой логов. Использовать данный уровень следует только в критических местах проекта, для выявления проблем, которые могут повлиять на работоспособность системы.

Какой уровень лога использовать для того или иного места в программе решает сам разработчик. 
Т.к. уровень лога определяет критичность проблемы, возникшей при выполнении кода.
Ниже приведу несколько примеров использования уровней логирования  


# Пример использования #

**Крайне рекомендую использовать данный подход при создании объекта логгера.**

```
<?
if (CModule::IncludeModule('intensa.logger'))
{
    $objILog = new \Intensa\Logger\ILog('logger_code');
}
?>
```

Если по какой-то причине не удастся подключить класс модуля на страницах сайта не вывалиться фатальная ошибка.


```
<?

// подключение модуля
if (CModule::IncludeModule('intensa.logger'))
{
	/*
	* Создание объекта логгера.
	* Конструктору передаем код логгера.
	* Файл лога будет соответствовать коду логгера + расширение заданное в файле конфига
	*/
	$logger = new \Intensa\Logger\ILog('log_code');	
}



/*
 * Ниже представлены несколько методов для более гибкой работы с данными логов.
 * Методы переопределяют значения заданые в файле конфига в рамках вызова логгера
 * */

/*
 * При вызове данного метода вся цепочка логов будет отправлятся на почту.
 * Независимо от того был ли вызван метод fatal()
 * */
$logger->sendAlert();

/*
 * При вызове данного метода в запись лога будет добавлена информаци о файле, в котором был вызван метод.
 * Данные будут записыватся даже при установленом флаге USE_BACKTRACE = false в файле конфига
 * */
$logger->useBacktrace();
/*
 * При вызове данного метода вызовы лога будут отделены разделителями.
 * Разделители будут утанавливатся даже при установленном флаге USE_DELIMITER = false в файле конфига
 * */
$logger->useLogDelimiter();


/*
 * Для каждого уровня логирования имеется свой метод
 * */

$logger->debug('debug message', [0 => 'context']);
$logger->info('log message', [0 => 'context']);
$logger->log('log message', [0 => 'context']); // алиас для log()
$logger->error('error message', [0 => 'context']);
/*
 * При вызове метода будет автоматически отправлено письмо c полной цепочной логов в рамках вызова
 * */
$logger->fatal('fatal message', [0 => 'context']);
?>
```
### Использование логгера при создании заявки ###

```

<?
$ticketData = ['name' => 'John', 'sname' => 'Smith', 'phone' => '8000'];

if (CModule::IncludeModule('intensa.logger'))
{
	$logger = new \Intensa\Logger\ILog(__FILE__);
}

$logger->log('ticket data', $ticketData);

try
{
    $ticket = new TestTicket($ticketData);
    $newTicketID = $ticket->add();
    
    $logger->log('ticket create number ' . $newTicketID);
}
catch (Exception $e)
{
    $logger->fatal('ticket create problem', $e);
}


?>
```

### Использование логгера при добавлении товара ###


```
<?
if (CModule::IncludeModule('intensa.logger'))
{	
	$logger = new \Intensa\Logger\ILog('import_script');
}

$logger->useLogDelimiter();

$logger->log('start import');

if (file_exists('import.xml'))
{
    $logger->log('file exist');

    // ..
    // code
    // ..

    $uid = $arImportFile['GUID'];

    $logger->log('start add product');

    if (in_array($uid, $arProducts))
    {
        $logger->error('product exist. not add product', $uid);
    }
    else
    {
        // add product
        $logger->log('success.product add', $uid);
    }

}
else
{
    $logger->error('file not exist');
}


?>
```
### Использование таймера вызова ###

Допустим, нужно иметь в логе информацию о времени выполнения какого-то определенного участка кода.


```

<?

$context = [1, 2, 3];

if (CModule::IncludeModule('intensa.logger'))
{
	$logger = new \Intensa\Logger\ILog('my_logger_code');
}

$logger->log('Логируем какие-то данные', $context);

// запуск таймера. необходимо передать код таймера
$logger->startTimer('timer_1');
sleep(1);
// остановка таймера. необходимо передать код таймера.
$logger->stopTimer('timer_1');

$logger->startTimer('timer_2');

// можно запускать один таймер внутри другого
$logger->startTimer('timer_internal');
sleep(2);
$logger->stopTimer('timer_internal');

sleep(3);
$logger->stopTimer('timer_2');

// если у таймера не указана точка остановки, то время остановки определяется в дeструкторе класса
$logger->startTimer('dont_stop');

?>
```

Формат логов таймера:

* CODE - код таймера, задает разработчиком при вызове таймера
* START_TIME - время запуска
* STOP_TIME - время остановки
* EXEC_POINT - время исполнения (разница между временем остановки и временем запуска)
* START_POINT - место определения точки запуска (вызов метода startTimer()) 
* STOP_POINT - место определения точки остановки (вызов метода stopTimer()). Значение "__destruct" означает о том, что не был вызван метод stopTimer() для текущего счетчика и остановка произошла в деструкторе класса.

Еще одна небольшая особенность - ключи START_POINT и STOP_POINT будут добавлены в файл при включенной опции
USE_BACKTRACE в конфиге или если вызван метод useBacktrace()

```

==============================[START: 9538]==============================
[2018-10-08 16:15:47] [:info] [/var/www/html/test/ish.php:11] Логируем какие-то данные Array
(
    [0] => 1
    [1] => 2
    [2] => 3
)

[2018-10-08 16:15:48] [:timer] [/var/www/html/test/ish.php:17] Lead time: Array
(
    [CODE] => timer_1
    [START_TIME] => 2018-10-08 16:15:47.000000
    [STOP_TIME] => 2018-10-08 16:15:48.000000
    [EXEC_TIME] => 1.000124931
    [START_POINT] => /var/www/html/test/ish.php:14
    [STOP_POINT] => /var/www/html/test/ish.php:17
)

[2018-10-08 16:15:50] [:timer] [/var/www/html/test/ish.php:24] Lead time: Array
(
    [CODE] => timer_internal
    [START_TIME] => 2018-10-08 16:15:48.000000
    [STOP_TIME] => 2018-10-08 16:15:50.000000
    [EXEC_TIME] => 2.000115871
    [START_POINT] => /var/www/html/test/ish.php:22
    [STOP_POINT] => /var/www/html/test/ish.php:24
)

[2018-10-08 16:15:53] [:timer] [/var/www/html/test/ish.php:27] Lead time: Array
(
    [CODE] => timer_2
    [START_TIME] => 2018-10-08 16:15:48.000000
    [STOP_TIME] => 2018-10-08 16:15:53.000000
    [EXEC_TIME] => 5.000380039
    [START_POINT] => /var/www/html/test/ish.php:19
    [STOP_POINT] => /var/www/html/test/ish.php:27
)

[2018-10-08 16:15:53] [:timer] [/var/www/dev/intensa.logger/classes/general/ILog.php:516] Lead time: Array
(
    [CODE] => dont_stop
    [START_TIME] => 2018-10-08 16:15:53.000000
    [STOP_TIME] => 2018-10-08 16:15:53.000000
    [EXEC_TIME] => 0.009480000
    [START_POINT] => /var/www/html/test/ish.php:30
    [STOP_POINT] => __destruct
)
==============================[END: 9538]==============================
```



# Формат логов #

![1538659123766.jpg](https://i.imgur.com/D8xgiG5.jpg)

На скрине описан шаблон записи лога. 

На данный момент запись лога может занимать более одной строки. Т.к. логи будут анализироваться в ручную, а смотреть многомерные массивы строкой не очень удобно.

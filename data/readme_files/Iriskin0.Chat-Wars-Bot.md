# Chat-Wars-Bot
Бот для текстовой мморпг Chat Wars в Telegram<br />
НЕ ПЫТАЙТЕСЬ СКОМПИЛИРОВАТЬ КЛИ НА ВИНДОУС И ТЕЛЕФОНАХ, ЕСЛИ ВЫ НЕ ШАРИТЕ - НЕ ПОЛУЧИТСЯ. <br />

#### Работающие функции бота:
  - тратить всю выносливость в лес/пещеру/побережье (по умолчанию ходит только в лес)
  - арена с рандомным выбором места атаки и защиты
  - перехват корованов
  - атака/защита по приказу бота/игрока или автоматическая защита замка
  - донат в казну замка перед битвой (по умолчанию выключен)
  - донат в лавку вместо казны (покупка-продажа барахла) (по умолчанию выключен. Для работы должен быть включен обычный донат. Если инвентарь полон - отключается, и происходит обычный донат в казну)
  - прокачка атаки/защиты при получении уровня (по умолчанию выключен)
  - ответы бота в групповой чат (по умолчанию выключен)
  - форвард лесных квестов от бота для помощи
  - сохранение текущего состояния и настроек в конфиг-файл, для восстановления после перезапуска
  - постройки в замке, когда закончилась арена и нет стамины
  - суммирование стока через PenguindrumStockBot
  - автопомощь ботов друг другу

#### Как запустить:<br />
  1) Устанавливаем telegram-cli по [официальной инструкции](https://github.com/vysheng/tg)<br />
  2) Устанавливаем pip3: `sudo apt-get install python3-pip`<br />
  3) Устанавливаем pytg для 3 питона: `pip3 install pytg pytz requests`<br />
  4) Запускаем telegram-cli: `./telegram-cli --json -P 1338`<br />
  5) Открываем еще один терминал<br />
  5) Качаем этот скрипт (`git clone https://github.com/Iriskin0/Chat-Wars-Bot.git`) и запускаем: `python3 main.py --admin "ваш ник" --order "ник игроков/ботов, выдающих приказы, через запятую   без пробелов" --gold "сколько золота оставлять в кармане при автодонате" --buy "любое значение для включения доната в лавку" --lvlup "lvl_def - прокачка защиты/lvl_atk - прокачка атаки" --group_name 'название группового чата для отправки сообщений' --apikey "apikey для IFTTT, инструкция ниже"`<br />
  6) Для работы с капчой, начинаем общение с @ChatWarsCaptchaBot (тестируется) или бот сам первый напишет
  7) Для работы со стоками начинаем общение с @PenguindrumStockBot и объявляем майн профиль по инструкции бота
  
#### Команды боту от админа:<br />
    #help - Список всех команд
    #enable_bot - Включить бота
    #disable_bot - Выключить бота
    #enable_arena - Включить арену
    #disable_arena - Выключить арену
    #enable_les - Включить лес
    #disable_les - Выключить лес
    #enable_peshera - Включить пещеры
    #disable_peshera - Выключить пещеры
    #enable_corovan - Включить корован
    #disable_corovan - Выключить корован
    #enable_more - Включить побережье
    #disable_more - Выключить побережье
    #enable_order - Включить приказы
    #disable_order - Выключить приказы
    #enable_auto_def - Включить авто деф
    #disable_auto_def - Выключить авто деф
    #enable_donate - Включить донат
    #disable_donate - Выключить донат
    #enable_buy - Включить донат в лавку вместо казны
    #disable_buy - Вылючить донат в лавку вместо казны
    #enable_quest_fight - Включить битву во время квеста
    #disable_quest_fight - Выключить битву во время квеста
    #lvl_atk - качать атаку
    #lvl_def - качать защиту
    #lvl_off - ничего не качать
    #status - Получить статус
    #hero - Получить информацию о герое
    #push_order - Добавить приказ
    #order - Дебаг, последняя команда защиты/атаки замка
    #log - Дебаг, последние 30 сообщений из лога
    #time - Дебаг, текущее время
    #lt_arena - Дебаг, последняя битва на арене
    #get_info_diff - Дебаг, последняя разница между запросами информации о герое
    #ping - Дебаг, проверить жив ли бот
    #enable_build - Включить постройки
    #disable_build - Выключить постройки
    #build_target - Задать цель для построек
    #stock - Обновить сток в боте
    #info - Оперативная информация по уровню, стамине, золоту и арене
    #add ресурсы,через,запятую - Создать предложение в трейд боте с этими ресурсами
    #detail - Почти вся информация о герое, только компактнее
    #report - Получить репорт с прошлой битвы
    #eval - Дебаг, выполнить запрос вручную

#### Интеграция с IFTTT (опционально)
1) Регистрируемся/логинимся на [IFTTT.com](http://IFTTT.com)
2) Заходим на [IFTTT.com/maker_webhooks](http://IFTTT.com/maker_webhooks), нажимает Connect если не подключено, потом нажимаем Documentation, копируем apikey
3) Идем на [IFTTT.com/create](http://IFTTT.com/create), в графе This выбираем Webhooks, потом Receive a web request![](http://i.imgur.com/fM0SpmU.png "This")
4) В Event Name вписываем `bot_error`
5) В графе That можно использовать все что угодно, я использую 
Notifications, но можно и к Telegram подключить
6) Настраиваем по инструкции (подключаем телефон/телеграм итд)
7) Можете использовать мой текст: `Ошибка {{Value2}} порт {{Value1}}: {{Value3}}`, можете составить свой
8) `Value1` - это порт (для идентификации). `Value2` - это тип ошибки (coroutine или очереди). `Value3` - это описание ошибки

##### Как настроить другие триггеры:
Начинаем с 3 пункта, вписываем, смотря в таблицу:
![](http://i.imgur.com/mv6qPvc.png "Таблица")

Будет выглядеть как-то так (это старый скрин)
![](http://i.imgur.com/Xnn41T5.png "Пример")

#### Автопомощь ботов
Если указан путь --fpath, боты по этому пути создают файл, на основании котрого другие боты в этом же замке приходят ему на помощь
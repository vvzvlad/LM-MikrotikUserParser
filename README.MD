###Задача:
Подключение роутера Mikrotik в качестве источника информации о пользователях, находящихся в зоне действия wifi-сети.

###Подготовка:
Заходим на роутер, раздающий WiFi, переходим в **System -> Scripts**, создаем новый скрипт с именем "**ActiveWirelessClients**" и таким кодом:
```
:do {/file remove activeclients.txt} on-error={}
/interface wireless registration-table print terse file="activeclients.txt"
```
Отмечаем галочками **ftp**, **read** и **write**, сохраняем.

Заходим в **System -> Scheduler** создаем новое раписание, с именем **ActiveWirelessClientsSheduled** в поле **On Event** - пишем **ActiveWirelessClients**    

В поле **Interval** устанавливаем значение **00:00:02**

Отмечаем галочками **ftp**, **read** и **write**, сохраняем.

Заходим в *IP -> Services -> ftp*, ставим галочку **Enable**, сохраняем.

###Установка:
Скачиваем архив https://github.com/vvzvlad/LM-MikrotikUserParser/raw/master/MikrotikUserParser.tar.gz и кнопкой **Restore/add libraries** в **Scripts** -> **User libraries** устанавливаем архив. 

Сохраняем скрипт. 
Создаем новый **Scheduled** скрипт, настраивая его ну нужную вам частоту запуска(например, раз в 5 минут: */5 * * * *), и вызываем в нем функцию следующим образом:

`activeuserstext = miktorik_whoisthere(users_table,mikrotik_table)`

**users_table** и **mikrotik_table** — таблица нужных нам пользователей и таблица, полученная с микротика. 
**activeuserstext** — переменная с именами пользователей, которые присутствуют в зоне действия сети. 

Таблица пользователей формируется следующий образом:
```
users_table = {
    "00:88:65:66:3F:77", "Влад Зайцев", "iPhone-Zaytsev",
    "00:23:12:10:DA:CC", "Влад Зайцев", "MBP-Zaytsev",
}
```
Первое поле - MAC-адрес. Второе - имя пользователя. Если у пользователя несколько устройств, каждому из них присваивается одинаковое имя, после чего в итоговом списке будет показано лишь одна строка для одного пользователя, а не по строке на каждое устрйоство. 
Третье - имя устройства. Если во втором поле будет не имя, а знак вопроса(?), то вместо имени пользователя будет показано имя устройств, что удобно для отладки и идентификации устройств. 

Получить таблицу активных пользователей с роутера можно таким образом:

`mikrotik_table = io.readproc('wget --quiet -O - ftp://ftp:ftp@mikrotik.lc/activeclients.txt 2>&1') .. ("\n")
`

Фунция возвращает имена, разделенные переносом строки и тегом \<\/\b\>. Их можно вставить в визуализацию следующим образом:


1) Записываем имена в **storage**:

`storage.set('activeuserstext', activeuserstext)`

2)Создаем файл **activeusers.lp** с таким содержанием и копируем его на FTP LM:
```
<? print('HTTP/1.0 200 OK\r\nContent-Type: text/html\r\nCache-Control: no-cache, must-revalidate\r\nPragma: no-cache\r\n') ?>
<!doctype html>
<html>
<head> 
<meta http-equiv="Content-Type" Content="text/html;charset=utf-8"> 
</head>
<body> 
<? require('genohm-scada')
require('dbenv')
db = dbenv:new()
print(storage.get('activeuserstext'))
db:close() ?> 
<script>
setInterval(function() {location.reload();}, 20000);
</script>
</body>
</html>
```
3)В нужном месте визуализации создаем Frame, и указываем его адрес: **http://logicmachine.lc:8080/activeusers.lp**



 

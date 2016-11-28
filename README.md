#Навигация по коду в IDE от JetBrains с использованием REST API и командной строки


При разработке приложений часто приходится сталкиваться с необходимостью просмотра вывода exception stack trace (в логах или при debug-инге). Хотелось бы иметь возможность автоматически попадать в необходимое место кода, прямо кликом по строке в выводе stack trace в браузере или в терминале.

Если вы являетесь пользователем одного из последних продуктов компании [JetBrains](https://www.jetbrains.com/) (в частности PhpStorm), вы можете использовать для этих целей внутреннее REST API (для навигации из браузера) и command line launcher (для навигации в терминале). 

# Навигация в браузере

Частичное описание методов REST API IDE от JetBrains можно посмотреть здесь:
[http://develar.org/idea-rest-api/](http://develar.org/idea-rest-api/)

Одним из методов этого API является возможность открыть файл проекта и переместиться на произвольную позицию в этом файле внутри самой IDE. 

<cut />

Обращения к вызовам методов API осуществляется  через вызов по адресу `http://localhost:63342/`

Пример вызова API для открытия файла выглядит как:

`http://localhost:63342/api/file?file=src/path/to/file.php&line=100&column=34`

где 
**file** - относительный или абсолютный путь файла
**line** - строка в файле, куда нужно переместить курсор
**column** - позиция на указанной строке

![alert](https://habrastorage.org/files/b1d/e7d/4b6/b1de7d4b69964e449e16751d02ed9b98.png)

Для того чтобы убрать сообщение, которое каждый раз появляется при вызове API в IDE, можно в настройках 
`Build, Execution, Deployment -> Debugger`
поставить галочку *"Allow unsigned request"* (либо каждый раз придется в диалоге кликать на кнопку Ok).

![Settings](https://habrastorage.org/files/1b2/d58/804/1b2d5880486240c986c6ed9f130d5a73.png)

# Пример обработки  вывода стандартного getTraceAsString() в php

Ниже показан Regexp, обрабатывающий стандартный вывод stack trace у Exception через [getTraceAsString()](http://php.net/manual/ru/exception.gettraceasstring.php)

```php
try {
    // some code
}(\Exception $e){
    $traceAsString = preg_replace('/#(\d+) (.+?\.php)\((\d+)\):/', '#$1 <a href="#" onclick="_goToEditorCodeLine(\'$2\', \'$3\'); return false;">$2($3):</a>', $e->getTraceAsString() );
    // some code
}
```

Каждая строка в Exception становится ссылкой, клик на которую открывает в IDE файл на нужной строке. 

Также необходимо подключить JS функцию, которая будет непосредственно "дергать" метод API

```javascript
function _goToEditorCodeLine(file, line){     
    var xmlhttp = new XMLHttpRequest();     
    xmlhttp.open("GET", "http://localhost:63342/api/file?file=" + file + "&line=" + line, true);
    xmlhttp.send(); 
}
```

# Пример реализации на примере обработки Exception в Symfony 3

Поскольку в настоящий момент я работаю с Symfony, я на конкретном примере покажу как модифицировать страницу со стандартным Exception чтобы реализовать открытие в IDE файла. 

Для того, чтобы переопределить кусочек шаблона, который отвечает за вывод страницы Exception необходимо в папке 
`app/Resources/TwigBundle/views/Exception/`
создать два файла
- exception.html.twig
- trace.html.twig

В файле exception.html.twig  добавить простейшую функцию `_goToEditorCodeLine()`, код которой описан выше.

В файле trace.html.twig найдем место вывода строки и добавим в конец стрелку, нажатие на которую будет осуществлять открытие файла в IDE.
```
    in {{ trace.file|format_file(trace.line) }} <a href="#" onclick="_goToEditorCodeLine('{{ trace.file }}', '{{ trace.line }}'); return false;">&rarr;</a>&nbsp;
```
После этого в строке stack trace появлется стрелка, нажатие на которую открывает в IDE файл и перемещает курсор на нужную позицию. 

![Symfony Exception](https://habrastorage.org/files/c51/8da/1be/c518da1bea9d47bbbd1127bc27e5222a.png)


# Интеграция с командной строкой в iTerm2

Если вы работаете в MacOS и используете iTerm2, можно провести интеграцию iTerm2 и Command Line Launcher соответствующей IDE, что позволит открывать файлы в IDE прямо из терминала.

Для установки launcher-а необходимо в IDE вызвать меню 
`Tool -> Create Command-line Launcher...`
и в диалоговом окне подтвердить путь, куда launcher будет установлен. В моем случае это 
`/usr/local/bin/phpstorm` (или `/usr/local/bin/pstorm` для более ранних версий)
для IntelliJ IDEA это 
`/usr/local/bin/idea` 

Данный launcher позволяет, при вызове из командной строки, открывать файл и переходить на нужную строку в нем. 

Пример вызова: 
`/usr/local/bin/phpstorm --line 40 /path/to/file`
или
`/usr/local/bin/phpstorm /path/to/file:40`

В обоих случаях будет открыт файл `/path/to/file` на 40 строке.

Теперь сделаем интеграцию command line launcher и iTerm2.

Идем в Edit Session->Advanced

![iTerm](https://habrastorage.org/files/eae/fb5/4aa/eaefb54aaf7c4615a523dfc0047e3e46.png)

 В секции `Semantic History`, из выпадающего списка, выбираем `run command...` и вводим 

`/usr/local/bin/phpstorm --line \2 \1`

Теперь если нажать Cmd и навести стрелку курсора на какой либо файл в терминале - он станет ссылкой и клик по нему осуществит его открытие в IDE.

Если у вас логи имеют вид 
`/path/to/file/:40`, т.е. номер строки указан через двоеточие после файла - тогда IDE будет открывать файл прямо на этой строке. 

Проблема c логами php-exception в том, что вывод имеет вид 
`/path/to/file/(40)`
т.е. строка в файле стоит в скобках.  
В результате этого файл открывается, но на нужную строку не переходит. 

Для решения этой проблемы можно преобразовать вывод лога в нужный нам формат, используя потоковый редактор sed.

`sed -E 's/#([0-9]+) (.+\.php)\(([0-9]+)\):/#\1 \2:\3/g'`

пример обработки вывода функции tail

`tail test.log | sed -E 's/#([0-9]+) (.+\.php)\(([0-9]+)\):/#\1 \2:\3/g'`

В заключении еще раз хотел бы подчеркнуть, что данная схема работает практически с любым языком и любой IDE от JetBrains.
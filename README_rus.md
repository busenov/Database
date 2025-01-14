## Получение библиотеки

Вы можете [скачать её архивом](https://github.com/Vasiliy-Makogon/Database/archive/master.zip), клонировать с данного сайта или загрузить через composer ([ссылка на packagist.org](https://packagist.org/packages/busenov/database)):
```
composer require busenov/database
```


## Что такое `busenov/database`?

`busenov/database` — библиотека классов на PHP 8.0 для простой, удобной, быстрой и безопасной работы с базой данных MySql, использующая расширение PHP [mysqli](https://www.php.net/manual/ru/book.mysqli.php).


### Зачем нужен самописный класс для MySql, если в PHP есть абстракция PDO и расширение mysqli?

Основные недостатки всех библиотек для работы с mysql-базой в PHP это:

* **Многословность**
  * Чтобы предотвратить SQL-инъекции, у разработчиков есть два пути:
    * Использовать [подготавливаемые запросы](https://www.php.net/manual/ru/mysqli.quickstart.prepared-statements.php).
    * Вручную экранировать параметры идущие в тело SQL-запроса. Строковые параметры прогонять через [mysqli_real_escape_string](https://www.php.net/manual/ru/mysqli.real-escape-string.php), а ожидаемые числовые параметры приводить к соответствующим типам — `int` и `float`.
  * Оба подхода имеют колоссальные недостатки:
    * Подготавливаемые
    запросы [ужасно многословны](https://www.php.net/manual/ru/mysqli.prepare.php#refsect1-mysqli.prepare-examples). Пользоваться "из коробки" абстракцией PDO или расширением mysqli, без агрегирования
    всех методов для получения данных из СУБД просто невозможно — что бы получить значение из таблицы необходимо
    написать минимум 5 строк кода! И так на каждый запрос!
    * Экранирование вручную параметров, идущих в тело SQL-запроса — даже не обсуждается. Хороший программист — ленивый программист. Всё должно быть максимально автоматизировано.
* **Невозможность получить SQL запрос для отладки**
  * Что бы понять, почему в программе не работает SQL-запрос, его нужно отладить — найти либо логическую, либо синтаксическую ошибку. Что бы найти ошибку, необходимо "видеть" сам SQL-запрос, на который "ругнулась" база, с подставленными в его тело параметрами. Т.е. иметь сформированный полноценный SQL.
Если разработчик использует PDO, с подготавливаемыми запросами, то это сделать... НЕВОЗМОЖНО! Никаких максимально удобных механизмов для этого в родных библиотеках [НЕ ПРЕДУСМОТРЕНО](https://qna.habr.com/q/22669). Остается либо извращаться, либо лезть в лог базы данных.


### Решение: `busenov/database` — класс для работы c MySql

1. Избавляет от многословности — вместо 3 и более строк кода для исполнения одного запроса при использовании "родной" библиотеки, вы пишите всего одну.
2. Экранирует все параметры, идущие в тело запроса, согласно указанному типу заполнителей — надежная защита от SQL-инъекций.
3. Не замещает функциональность "родного" mysqli адаптера, а просто дополняет его.
4. Расширяема. По сути, библиотка предоставляет собой лишь парсер и исполнение SQL-запроса с гарантированной защитой от SQL-инъекций. Вы можете унаследоваться от любого класса библиотеки и используя как механизмы библиотеки, так и механизмы `mysqli` и `mysqli_result` создавать необходмые вам методы для работы.

### Чем НЕ является библиотека `busenov/database`?

Большинство оберток под различные драйверы баз данных являются нагромождением бесполезного кода с отвратительной архитектурой. Их авторы, сами не понимая практической цели своих оберток, превращают их в подобие построителей запросов (sql builder), ActiveRecord библиотек и прочих ORM-решений.

Библиотека `busenov/database` не является ничем из перечисленных. Это лишь удобный инструмент для работы с обычным SQL в рамках СУБД MySQL — и не более!


## Что такое placeholders (заполнители)?

**Placeholders** (англ. — заполнители) — специальные типизированные маркеры, которые пишутся в строке SQL запроса *вместо явных значений (параметров запроса)*. А сами значения передаются "позже", в качестве последующих аргументов основного метода, выполняющего SQL-запрос:

```php
<?php
// Предположим, что установили библиотеку через composer 
require  './vendor/autoload.php';

use busenov\Database\Mysql;

// Соединение с СУБД и получение объекта-"обертки" над mysqli - \busenov\Database\Mysql
$db = Mysql::create("localhost", "root", "password")
      // Язык вывода ошибок - русский
      ->setErrorMessagesLang('ru')
      // Выбор базы данных
      ->setDatabaseName("test")
      // Выбор кодировки
      ->setCharset("utf8")
      // Включим хранение всех SQL-запросов для отчета/отладки/статистики
      ->setStoreQueries(true);

// Получение объекта результата \busenov\Database\Statement
// \busenov\Database\Statement - "обертка" над объектом mysqli_result
$result = $db->query("SELECT * FROM `users` WHERE `name` = '?s' AND `age` = ?i", "Д'Артаньян", 41);

// Получаем данные (в виде ассоциативного массива, например)
$data = $result->fetchAssoc();

// SQL-запрос работает не так, как вы ожидаете?
// Не проблема - выведите его на печать и посмотрите сформированный SQL-запрос,
// который будет уже с подставленными в его тело параметрами:
echo $db->getQueryString(); // SELECT * FROM `users` WHERE `name` = 'Д\'Артаньян' AND `age` = 41
```

Параметры SQL-запроса, прошедшие через систему *placeholders*, обрабатываются специальными механизмами экранирования, в зависимости от типа заполнителей. Т.е. вам теперь нет необходимости заключать переменные в функции экранирования типа `mysqli_real_escape_string()` или приводить их к числовому типу, как это было раньше:

```php
<?php
// Раньше перед каждым запросом в СУБД мы делали
// примерно это (а многие и до сих пор `это` не делают):
$id = (int) $_POST['id'];
$value = mysqli_real_escape_string($mysql, $_POST['value']);
$result = mysqli_query($mysql, "SELECT * FROM `t` WHERE `f1` = '$value' AND `f2` = $id");
```

Теперь запросы стало писать легко, быстро, а главное библиотека `busenov/database` полностью предотвращает любые возможные SQL-инъекции.


### Введение в систему заполнителей

Типы заполнителей и их предназначение описываются ниже. Прежде чем знакомиться с типами заполнителей, необходимо понять как работает механизм библиотеки `busenov/database`. Пример:

```php
 $db->query("SELECT ?i", 123); 
```
SQL-запрос после преобразования шаблона:
```sql
SELECT 123
```
В процессе исполнения этой команды *библиотека проверяет, является ли аргумент `123` целочисленным значением*. Заполнитель `?i` представляет собой символ `?` (знак вопроса) и первую букву слова `integer`. Если аргумент действительно представляет собой целочисленный тип данных, то в шаблоне SQL-запроса заполнитель `?i` заменяется на значение `123` и SQL передается на исполнение.

Поскольку PHP слаботипизированный язык, то вышеописанное выражение эквивалентно нижеописанному:

```php
 $db->query("SELECT ?i", '123'); 
 ```
 
 SQL-запрос после преобразования шаблона:
 
 ```sql
 SELECT 123
 ```
 
 т.е. числа (целые и с плавающей точкой) представленные как в своем типе, так и в виде `string` — равнозначны с точки зрения библиотеки.
 
  
### Режимы работы библиотеки и принудительное приведение типов

 Существует два режима работы библиотеки:
  
  * **Mysql::MODE_STRICT — строгий режим соответствия типа заполнителя и типа аргумента**.
    В режиме `Mysql::MODE_STRICT` *аргументы должны соответствовать типу заполнителя*. Например, попытка передать в качестве аргумента значение `55.5` или `'55.5'` для заполнителя целочисленного типа `?i` приведет к выбросу исключения:
  
```php
// устанавливаем строгий режим работы
$db->setTypeMode(Mysql::MODE_STRICT);
// это выражение не будет исполнено, будет выброшено исключение:
// Попытка указать для заполнителя типа "int" значение типа "double" в шаблоне запроса "SELECT ?i"
$db->query('SELECT ?i', 55.5);
```

* **Mysql::MODE_TRANSFORM — режим преобразования аргумента к типу заполнителя при несовпадении типа заполнителя и типа аргумента.** Режим `Mysql::MODE_TRANSFORM` установлен по-умолчанию и является "толерантным" режимом — при несоответствии типа заполнителя и типа аргумента не генерирует исключение, а *пытается преобразовать аргумент к нужному типу заполнителя посредством самого языка PHP*. К слову сказать, я, как автор библиотеки, всегда использую именно этот режим, строгий режим (`Mysql::MODE_STRICT`) в реальной работе никогда не использовал, но быть может, конкретно вам он понадобится.

**Допускаются следующие преобразования в режиме Mysql::MODE_TRANSFORM:**

* **К типу `int` (заполнитель `?i`) приводятся**
  * числа с плавающей точкой, представленные как в типе `string`, так и в типе `double`
  * `bool` TRUE преобразуется в `int(1)`, FALSE преобразуется в `int(0)`
  * `null` преобразуется в `int(0)`
* **К типу `double` (заполнитель `?d`) приводятся**
  * целые числа, представленные как в типе `string`, так и в типе `int`
  * `bool` TRUE преобразуется в `float(1)`, FALSE преобразуется в `float(0)`
  * `null` преобразуется в `float(0)`
* **К типу `string` (заполнитель `?s`) приводятся**
  * `bool` TRUE преобразуется в `string(1) "1"`, FALSE преобразуется в `string(1) "0"`. Это поведение отличается от приведения типа `bool` к `int` в PHP, т.к. зачастую, на практике, булев тип записывается в MySql именно как число.
  * значение типа `numeric` преобразуется в строку согласно правилам преобразования PHP
  * `null` преобразуется в `string(0) ""`
* **К типу `null` (заполнитель `?n`) приводятся**
  * любые аргументы.
* Для массивов, объектов и ресурсов преобразования не допускаются.

**ВНИМАНИЕ!** Далее по тексту объяснение работы библиотеки будет идти подразумевая, что активирован режим `Mysql::MODE_TRANSFORM`.

### Какие типы заполнителей представлены в библиотеке?


#### `?i` — заполнитель целого числа

```php
$db->query('SELECT * FROM `users` WHERE `id` = ?i', $_POST['user_id']); 
```

**ВНИМАНИЕ!** Если вы оперируете числами, выходящими за пределы `PHP_INT_MAX`, то:
* Оперируйте ими исключительно как строками в своих программах.
* Не используйте данный заполнитель, используйте заполнитель строки `?s` (см. ниже). Дело в том, что числа, выходящие за пределы `PHP_INT_MAX`, PHP интерпретирует как числа с плавающей точкой. Парсер библиотеки постарается преобразовать параметр к типу `int`, в итоге «*результат будет неопределенным, так как float не имеет достаточной точности, чтобы вернуть верный результат. В этом случае не будет выведено ни предупреждения, ни даже замечания!*» — [php.net](https://www.php.net/manual/ru/language.types.integer.php#language.types.integer.casting.from-float). 

#### `?d` — заполнитель числа с плавающей точкой

```php
$db->query('SELECT * FROM `prices` WHERE `cost` = ?d', 12.56); 
```

**ВНИМАНИЕ!** Если вы используете библиотеку для работы с типом данных `double`, установите соответствующую локаль, что бы разделитель целой и дробной части был одинаков как на уровне PHP, так и на уровне СУБД. 

#### `?s` — заполнитель строкового типа

Значение аргументов экранируются с помощью метода `mysqli::real_escape_string()`:

```php
 $db->query('SELECT "?s"', "Вы все дураки, а я - Д'Артаньян!");
 ```
 SQL-запрос после преобразования шаблона:
 
```sql
SELECT "Вы все дураки, а я - Д\'Артаньян!"
```
#### `?S` — заполнитель строкового типа для подстановки в SQL-оператор LIKE 

Значение аргументов экранируются с помощью метода `mysqli::real_escape_string()` + экранирование спецсимволов, используемых в операторе LIKE (`%` и `_`): 

```php
 $db->query('SELECT "?S"', '% _'); 
 ```
 SQL-запрос после преобразования шаблона:
 ```sql
 SELECT "\% \_"
 ```
 
 #### `?n` — заполнитель `NULL` типа
Значение любых аргументов игнорируются, заполнители заменяются на строку `NULL` в SQL запросе:

```php
 $db->query('SELECT ?n', 123); 
 ```
 SQL-запрос после преобразования шаблона:
 ```sql
 SELECT NULL
 ```
 
 #### `?A*` — заполнитель ассоциативного множества из ассоциативного массива, генерирующий последовательность пар вида `ключ = значение`

где символ `*` — один из заполнителей:

 * `i` (заполнитель целого числа)
 * `d` (заполнитель числа с плавающей точкой)
 * `s` (заполнитель строкового типа)
 
 правила преобразования и экранирования такие же, как и для одиночных скалярных типов, описанных выше. Пример:

```php
$db->query('INSERT INTO `test` SET ?Ai', ['first' => '123', 'second' => 1.99]);
```
SQL-запрос после преобразования шаблона:
```sql
INSERT INTO `test` SET `first` = "123", `second` = "1"
```

#### `?a*` — заполнитель множества из простого (или также ассоциативного) массива, генерирующий последовательность значений

 где `*` — один из типов:
 * `i` (заполнитель целого числа)
 * `d` (заполнитель числа с плавающей точкой)
 * `s` (заполнитель строкового типа)
 
 правила преобразования и экранирования такие же, как и для одиночных скалярных типов, описанных выше. Пример:

```php
 $db->query('SELECT * FROM `test` WHERE `id` IN (?ai)', [123, 1.99]);
```
SQL-запрос после преобразования шаблона:
```sql
 SELECT * FROM `test` WHERE `id` IN ("123", "1")
```

 
#### `?A[?n, ?s, ?i, ...]` — заполнитель ассоциативного множества с явным указанием типа и количества аргументов, генерирующий последовательность пар `ключ = значение`

Пример: 
```php
 $db->query('INSERT INTO `users` SET ?A[?i, "?s"]', ['age' => 41, 'name' => "Д'Артаньян"]);
```
SQL-запрос после преобразования шаблона:
```sql
 INSERT INTO `users` SET `age` = 41,`name` = "Д\'Артаньян"
```

#### `?a[?n, ?s, ?i, ...]` — заполнитель множества с явным указанием типа и количества аргументов, генерирующий последовательность значений
Пример:
```php
 $db->query('SELECT * FROM `users` WHERE `name` IN (?a["?s", "?s"])', ["маркиз д\"Аркьен", "Д'Артаньян"]);
```
SQL-запрос после преобразования шаблона:
```sql
 SELECT * FROM `users` WHERE `name` IN ("маркиз д\"Аркьен", "Д\'Артаньян")
```
 
 
#### `?f` — заполнитель имени таблицы или поля

Данный заполнитель предназначен для случаев, когда имя таблицы или поля передается в запросе через параметр. Имена полей и таблиц обрамляется символом "апостроф":

```php
 $db->query('SELECT ?f FROM ?f', 'name', 'database.table_name');
 ```
 SQL-запрос после преобразования шаблона:
 ```sql
  SELECT `name` FROM `database`.`table_name`
 ```

 
### Ограничивающие кавычки

**Библиотека требует от программиста соблюдения синтаксиса SQL.** Это значит, что следующий запрос работать не будет:

```php
$db->query('SELECT CONCAT("Hello, ", ?s, "!")', 'world');
```

— заполнитель `?s` необходимо взять в одинарные или двойные кавычки:

```php
$db->query('SELECT concat("Hello, ", "?s", "!")', 'world');
```

SQL-запрос после преобразования шаблона:

```sql
SELECT concat("Hello, ", "world", "!")
```

Для тех, кто привык работать с PDO это покажется странным, но реализовать механизм, определяющий, нужно ли в одном случае заключать значение заполнителя в кавычки или нет — очень нетривиальная задача, требующая написания целого парсера.


## Примеры работы с библиотекой

В процессе...
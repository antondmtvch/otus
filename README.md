## Scoring API
Декларативный язык описания и система валидации запросов к HTTP API сервиса скоринга.

### Запуск
Для локального запуска необходим `Python 3.8`
- Запустить локально
```
$ python main.py --port [port] --log [log file]
```
- Запустить в контейнере
```
$ docker build -t scoring_api .
$ docker run -d -p 8080:8080 scoring_api
```

### Структура запроса
```
{"account": "<имя компании партнера>", "login": "<имя пользователя>", "method": "<имя метода>", "token": "
<аутентификационный токен>", "arguments": {<словарь с аргументами вызываемого метода>}}
```
- `account` - строка, опционально, может быть пустым
- `login` - строка, обязательно, может быть пустым
- `method` - строка, обязательно, может быть пустым
- `token` - строка, обязательно, может быть пустым
- `arguments` - словарь, обязательно, может быть пустым

**Валидация**

Запрос валиден, если валидны все поля по отдельности.

**Структура ответа**

OK:
```
{"code": <числовой код>, "response": {<ответ вызываемого метода>}}
```
Ошибка:
```
{"code": <числовой код>, "error": {<сообщение об ошибке>}}
```
**Аутентификация**

В случае если не пройдена
 ```
 {"code": 403, "error": "Forbidden"}
```
### Методы
#### online_score

**Аргументы**

- `phone` - строка или число, длиной 11 , начинается с 7 , опционально, может быть пустым
- `email` - строка, в которой есть `@`, опционально, может быть пустым
- `first_name` - строка, опционально, может быть пустым
- `last_name` - строка, опционально, может быть пустым
- `birthday` - дата в формате `DD.MM.YYYY`, с которой прошло не больше 70 лет, опционально, может быть пустым
- `gender` - число 0 , 1 или 2 , опционально, может быть пустым

**Валидация аругементов** 

Аргументы валидны, если валидны все поля по отдельности и если присутсвует хоть одна пара
__phone-email__, __first name-last name__, __gender-birthday__ с непустыми значениями.

**Контекст** 

В словарь контекста добавляется запись `has` - список полей, которые были не пустые для данного
запроса.

**Ответ** 

В ответ выдается число, полученное вызовом функции `get_score` (см. `scoring.py`). Но если пользователь админ (см.
`check_auth`), то нужно всегда отавать 42.
```
{"score": <число>}
```
или если запрос пришел от валидного пользователя admin
```
{"score": 42}
```
или если произошла ошибка валидации
```
{"code": 422, "error": "<сообщение о том какое поле(я) невалидно(ы) и как именно>"}
```
**Пример**
```
$ curl -X POST -H "Content-Type: application/json" -d '{"account": "horns&hoofs", "login": "h&f", "method":
"online_score", "token":
"55cc9ce545bcd144300fe9efc28e65d415b923ebb6be1e19d2750a2c03e80dd209a27954dca045e5bb12418e7d89b6d718a9e35af34e14e1d5bcd
"arguments": {"phone": "79175002040", "email": "ivanov@example.ru", "first_name": "Иван", "last_name":
"Иванов", "birthday": "01.01.1990", "gender": 1}}' http://127.0.0.1:8080/method/
```
```
{"code": 200, "response": {"score": 5.0}}
```
#### clients_interests

**Аргументы**
- `client_ids` - массив числе, обязательно, не пустое
- `date` - дата в формате `DD.MM.YYYY`, опционально, может быть пустым

**Валидация аругементов** 

Аргументы валидны, если валидны все поля по отдельности.

**Контекст** 

В словарь контекста добавляеися запись `nclients` - количество id'шников, переденанных в запрос.

**Ответ** 

В ответ выдается словарь `<id клиента>:<список интересов>`. Список генерировать вызовом функции `get_interests` (см.
`scoring.py`).
```
{"client_id1": ["interest1", "interest2" ...], "client2": [...] ...}
```
или если произошла ошибка валидации
```
{"code": 422, "error": "<сообщение о том какое поле(я) невалидно(ы) и как именно>"}
```
**Пример**
```
$ curl -X POST -H "Content-Type: application/json" -d '{"account": "horns&hoofs", "login": "admin", "method":
"clients_interests", "token":
"d3573aff1555cd67dccf21b95fe8c4dc8732f33fd4e32461b7fe6a71d83c947688515e36774c00fb630b039fe2223c991f045f13f
"arguments": {"client_ids": [1,2,3,4], "date": "20.07.2017"}}' http://127.0.0.1:8080/method/
```
```
{"code": 200, "response": {"1": ["books", "hi-tech"], "2": ["pets", "tv"], "3": ["travel", "music"], "4":
["cinema", "geek"]}}

# verification-bot
### Инициализация
```javascript
const verificationBot = require('./libs/verification-bot');
const bot = new verificationBot(TOKEN:string);
```
## Как работать
### Регистрация пользователя
Так как API телеграма позволяет отправлять сообщения только по id-чата, то это первое, что нужно получить.
Я предлагаю следующий способ:
1. Как только пользователь изъявил желания использовать верификацию через телеграм, генерируем случайный токен
2. Записываем токен в базу данных для соответствующего пользователя
3. Показываем токен пользователю
4. Пользователь отправляет боту токен в формате `t0000`
5. Бот после этого генерирует событие `user`

API:
```javascript
bot.on('user', callback)
```
В функцию `callback` будет передан в качестве аргумента объект следующего вида:
```javascript
{
  token: '0000' //:string токен, в виде строки из четырех символов
  chatID: 0 //:number ID чата
}
```
После этого мы в коллбэке должны найти соответствующий токен в базе данных и подставить на его место chatID, через который мы впоследствии будем обращаться к этому пользователю

### Отправка сообщения пользователю
API:
```javascript
bot.send(template:string, data:object)
```
Пример:
```javascript
bot.send('touch', {
  chatID: 31059467,
  code: 2128,
  operationID: 2,
  message: 'SOME ACTION'
});
```
Аргументы:
- `template` -- идентификатор шаблона сообщения который будет использован. Возможные значения ['message', 'code', 'touch'], шаблон по умолчанию -- message, если будет введено значение не из списка возможных, то будет выставлен шаблон message
- `data` -- данные, которые необходимо передать пользователю, структура следующая:
```javascript
{
  chatID: number //обязательное поле, требуется для всех шаблонов, уникальный идентификатор пользователя
  message: string //обязательное поле, требуется для всех шаблонов
  code: number //код для верифицирования операции, обязателен для шаблонов code и touch
  operationID: number //идентификатор операции, которая ожидает подтверждения, обязателен для шаблона touch
}
```
Различия шаблонов:
- `message` -- обычное сообщение
- `code` -- сообщение, уведомляющее пользователя о надобности ввести код, наряду с сообщением пользователю будет показан соответствующий текст
- `touch` -- то же самое, что `code`, только теперь у сообщения появляется кнопка для подтверждения в один клик, код всё равно отображается в качестве фоллбэка

### Событие `operation`
Событие `operation` возникает, когда пользователь нажал кнопку подтверждения транзакции.
API:
```javascript
bot.on('operation', callback)
```
В коллбэк будет передан объект следующего вида:
```javascript
{
  chatID: number, //ID чата, по которому мы идентифицируем пользователя
  cbqID: string, //Callback Query ID, требуется для последующей правильной обработкие события
  operationID: number //Номер операции, которая ожидает подтверждения и которую мы можем теперь провести
}
```
Как правильно обработать:
```javascript
bot.on('operation', data => {
  const {cbqID} = data;
  /* Тут выполняем нашу операцию */
  bot.closeTouchAction(cbqID, 'Подтверждено!'); //метод посылающий ответ на callback query
  //Вместо 'Подтверждено!' можем дать любую информацию пользователю, она появится во всплывающем окне
});
```
## Все события вместе с их значениями, передаваемыми в коллбэк
- `user` `{ token:string, chatID:number }`
- `operation` `{ chatID:number, cbqID:string, operationID:number }`
- `sending_error` `{ type:string, chatID:number, cbqID:string, error:object }` *chatID при ошибке отправки обычного сообщения, cbqID при ошибке отправки данных методом closeTouchAction* Типы ошибок: `message`, `code`, `touch`, `welcome`, `close_touch_action`

## Все методы вместе с их аргументами
- `bot.send(template,{ chatID, message, code, operationID })` поле code только для шаблонов code и touch, поле operationID только для шаблона touch
- `bot.closeTouchAction(cbqID, text)`

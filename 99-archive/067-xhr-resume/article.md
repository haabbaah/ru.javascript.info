archive:
  ref: resume-upload

---

# XMLHttpRequest: возобновляемая закачка

Современный `XMLHttpRequest` даёт возможность загружать файл как угодно: во множество потоков, с догрузкой, с подсчётом контрольной суммы и т.п.

Здесь мы рассмотрим общий подход к организации загрузки, а его уже можно расширять, адаптировать к своему фреймворку и так далее.

Поддержка -- все браузеры кроме IE9-.

## Неточный upload.onprogress

Ранее мы рассматривали загрузку с индикатором прогресса. Казалось бы, сделать возобновляемую загрузку на его основе очень просто.

Есть же `xhr.upload.onprogress` -- ставим на него обработчик, по свойству `loaded`  события `onprogress` смотрим, сколько байт загрузилось. А при обрыве -- возобновляем загрузку с последнего байта.

К счастью, отослать на сервер не весь файл, а только нужную часть его -- не проблема, [File API](http://www.w3.org/TR/FileAPI/) позволяет прочитать выбранный участок из файла и отправить его.

Примерно так:

```js
var slice = file.slice(10, 100); // прочитать байты с 10-го по 99-й включительно

xhr.send(slice); // ... и отправить эти байты в запросе.
```

...Но такая модель не жизнеспособна!

Всё дело в том, что `upload.onprogress` срабатывает, когда байты *отправлены*, но были ли они получены сервером -- браузер не знает. Может, их прокси-сервер забуферизовал, может серверный процесс "упал" в процессе обработки, может соединение порвалось и байты так и не дошли до получателя.

**Поэтому `onprogress` годится лишь для красивенького рисования прогресса.**

Для загрузки нам нужно точно знать количество загруженных байт. Это может сообщить только сервер.

## Алгоритм возобновляемой загрузки

Загрузкой файла будет заведовать объект `Uploader`, его примерный общий вид:

```js
function Uploader(file, onSuccess, onFail, onProgress) {

  var fileId = file.name + '-' + file.size + '-' + +file.lastModifiedDate;

  var errorCount = 0;

  var MAX_ERROR_COUNT = 6;

  function upload() {
    ...
  }

  function pause() {
    ...
  }

  this.upload = upload;
  this.pause = pause;
}
```

- Аргументы для `new Uploader`:

`file`
: Объект File API. Может быть получен из формы, либо как результат Drag'n'Drop.<dd>
`onSuccess`, `onFail`, `onProgress`
<dd>Функции-колбэки, которые будут вызываться в процессе (`onProgress`) и при окончании загрузки.

- Подробнее про важные данные, с которыми мы будем работать в процессе загрузки:

`fileId`
: Уникальный идентификатор файла, генерируется по имени, размеру и дате модификации. По нему мы всегда сможем возобновить загрузку, в том числе и после закрытия и открытия браузера.

`startByte`
: С какого байта загружать. Изначально -- с нулевого.

`errorCount / MAX_ERROR_COUNT`
: Текущее число ошибок / максимальное число ошибок подряд, после которого загрузка считается проваленной.

Алгоритм загрузки:

1. Генерируем `fileId` из названия, размера, даты модификации файла. Можно добавить и идентификатор посетителя.
2. Спрашиваем сервер, есть ли уже такой файл, и если да - сколько байт уже загружено?
3. Отсылаем файл с позиции, которую сказал сервер.

При этом загрузку можно прервать в любой момент, просто оборвав все запросы.

Демо ниже, к сожалению, работает лишь частично, так как на этом сайте Node.JS стоит за сервером Nginx, который буферизует все закачки, не передавая их в Node.JS до полного завершения.

Вы можете скачать пример и запустить локально для полноценной демонстрации:

[codetabs src="upload-resume" height=160]

Полный код включает также сервер на Node.JS с функциям `onUpload` -- начало и возобновление загрузки, а также `onStatus` -- для получения состояния загрузки.

## Итого

Мы рассмотрели довольно простой алгоритм возобновляемой загрузки.

Его можно усложнить:

- добавить подсчёт контрольных сумм, проверку целостности пересылаемых файлов,
- для индикации прогресса вместо неточного `xhr.upload.onprogress` -- сделать дополнительный запрос к серверу, в который тот будет отдавать текущий прогресс.
- разбивать файл на части и грузить в несколько потоков, несколькими параллельными запросами.

Как можно видеть, возможности современного XMLHttpRequest в плане загрузки файлов приближаются к полноценному файловому менеджеру -- полный контроль над заголовками, индикатор прогресса и т.п.


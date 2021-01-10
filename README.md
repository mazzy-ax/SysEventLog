# SysEventLog

[project]:https://github.com/mazzy-ax/SysEventLog
[license]:https://github.com/mazzy-ax/SysEventLog/blob/master/LICENSE
[ax2009]:ax2009
[ax2012]:ax2012
[ax4]:ax4

[SysEventLog][project] &ndash; это класс-обёртка над [System.Diagnostics.EventLog.WriteEntry](https://docs.microsoft.com/en-us/previous-versions/windows/desktop/sidebar/system-diagnostics-eventlog-writeentry). Класс написан на X++ в [Microsoft Dynamics AX 2009][ax2009], [Microsoft Dynamics AX 2012][ax2012] и [Axapta 4.0][ax4].

Класс `SysEventLog`:

* позволяет записать информацию из Аксапты в [Windows Event Log](https://docs.microsoft.com/en-us/windows/win32/wes/windows-event-log)
* выполняет преобразования между .net и Х++
* предлагает 5 публичных статических методов:

  * `info`
  * `warning`
  * `error`
  * `write`
  * `writeOnServer`

где `info`, `warning`, `error` - это методы-обёртки для повседневной работы с минимумом параметров,
метод `writeOnServer` позволяет гарантировано записать сообщение в Event Log на сервере `AOS`,
а метод `write` &ndash; это рабочая лошадка, которая выполняет основную работу.

Класс использует проект [Session](https://github.com/mazzy-ax/Session), чтобы проверить наличие привилегией локального администратора.
Поэтому импортируйте метод `Session.isInRoleAdmin` перед импортом данного проекта.

## Message header

В начале каждого сообщения метод `write` автоматически записывает служебную информацию вида:

````text
User: Admin, Company: dmo, AOS$01@COMPNAME, Session type: Client, Run on: Dynamics Client (4.0.2501.116)
User: mazzy, Company: dmr, AOS50$01@COMPNAME, Session type: Client, Run on: Dynamics Server 01 (5.0.593)
User: Admin, Company: usrt, AOS60$01@COMPNAME, Session type: WorkerThread, Run on: Dynamics Server 01 (6.3.164)
````

после заголовка записывается текст сообщения.

## Event Source (Источник)

Для каждой записи в `Windows Event Log` требуется указать строку, которая содержит название источника.
Предполагается, что источник будет связан с названием программы, которая создала данную запись.
Полный список источников можно найти в реестре по адресу `HKLM:SYSTEM\CurrentControlSet\Services\EventLog\Application`.
Создавать новые источники можно только с привилегией локального администратора.

В этом есть определенная проблема для Аксапты &ndash; Аксаптовский клиент редко запускается "run as Administrator",
да и Аксаптовский сервер далеко не всегда запускается из-под пользователя с правами администратора.

Если программист никак не правил класс `SysEventLog` и не указал источник, то по умолчанию класс будет использовать:

* источник `Dynamics Client` для вывода в EventLog на клиенте
* источник `Dynamics Server ##` для вывода в EventLog на сервере, где `##` &ndash; код инстанса `AOS`

Эти источники создаются во время установки Аксапты и являются достаточно "безопасными".

Если программист указал другой источник, то метод `write` попытается проверить, если ли у текущей сессии привилегии администратора.

* Если привилегии администратора есть, то метод попытается проверить и создать источник. После создания поспит одну секунду и попытается выполнить запись в EventLog
* Если привилегии администратора отсутствуют, то метод не будет ни проверять, ни создавать источник, а сделает "прыжок веры" и просто попытается выполнить запись в EventLog.

## Исключение, если источник не зарегистрирован

Если источник не зарегистрирован на компьютере, то пользователи Аксапты увидят сообщение в инфологе:

````text
ClrObject static method invocation error
````

Класс специально жёстко прерывает работу, чтобы незарегистрированные источники были обнаружены как можно раньше. Разработчика, который найдет источник ошибки ждет такой комментарий в коде:

````java
/*

В следующей строке кода может произойти CLR-ошибка 'ClrObject static method invocation error',
если текущая сессия стартовала без привилегий администратора
и источник (source) для метода System.Diagnostics.EventLog::WriteEntry не зарегистрирован в реестре.

Причины, по которым не делается никаких попыток замаскировать или перехватить ошибку:

1. в аксапте в случае ошибки внутри транзакции оператор try/catch пропустит все catch внутри транзакции.
   данный метод записи в EventLog с огромной вероятностью будет вызван именно внутри транзакции.
   поэтому try/catch в этом методе ничего не гарантирует.

2. source может быть зарегистрирован на компьютере между любыми двумя вызовами данного метода,
   а проверка на существование источника требует прав локального администратора.
   поэтому, не имея прав локального администратора, приходится делать "прыжок веры",
   полагаясь на то, что источник существует.

3. ошибка в следующей строке достаточно специфична и хорошо заметна даже рядовыми пользователями.
   поэтому эту ошибку быстро доведут до сведения разработчика, который легко сможет найти источник ошибки в этом методе.
   а в этом методе добравшегося ждут эти комментарии. welcome!

*/

System.Diagnostics.EventLog::WriteEntry(source, messageHeder + eventMessage, eventLogEntryType, eventId);

/*

На компьютере, где предыдущая строка выдала ошибку,
нужно хотя бы один раз выполнить с привилегиями администратора (runAsAdministrator):

либо Job в AOT (https://github.com/mazzy-ax/SysEventLog/blob/main/ax2009/Examples/Job_SysEventLog_Demo.xpp)

    SysEventLog_Demo(args)

либо команду PowerShell

    PS> [System.Diagnostics.EventLog]::CreateEventSource('%source%', 'Application')

либо команду в CMD

    c:\> eventcreate /ID 1 /T INFORMATION /SO "%source%" /L "Application" /D log-created

Ключевое требование: с привилегиями администратора!

Список зарегистрированных источников можно посмотреть в реестре по адресу:
HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\EventLog\Application

*/
````

## ChangeLog

* [CHANGELOG.md](CHANGELOG.md)
* <https://github.com/mazzy-ax/SysEventLog/releases>

## Помощь проекту

Буду признателен за ваши замечания, предложения и советы по проекту как в разделе [Issues](https://github.com/mazzy-ax/SysEventLog/issues), так и в виде письма на адрес <mazzy@mazzy.ru>

Мазуркин Сергей (mazzy)

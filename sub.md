
Нет кеширования Kerberos-креденциалов (т.е. "Offline logon" запрещён). Лаптоп в этой группе не сможет войти в систему офлайн, без связи с контроллером домена. LSA запрещает выдачу кешированного TGT.
https://learn.microsoft.com/en-us/windows-server/security/credentials-protection-and-management/protected-users-security-group?utm_source=chatgpt.com
  
LM Hash 

https://learn.microsoft.com/ru-ru/troubleshoot/windows-server/windows-security/prevent-windows-store-lm-hash-password

NTLM 

https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-server-2012-r2-and-2012/jj852241%28v%3Dws.11%29

RC4  

Перейди в Group Policy Management → редактируй соответствующий GPO (например, Default Domain Policy или отдельную политику безопасности для контроллеров домена). Зайди: Конфигурация компьютера (Computer Configuration) $\rightarrow$ Политики (Policies) $\rightarrow$ Параметры Windows (Windows Settings) $\rightarrow$ Параметры безопасности (Security Settings) $\rightarrow$ Локальные политики (Local Policies) $\rightarrow$ Параметры безопасности (Security Options)..Найди политику Network security: Configure encryption types allowed for Kerberos. 

Включи её (Enabled) и оставь только те типы шифрования, которые считаешь безопасными, например:
AES128_HMAC_SHA1
AES256_HMAC_SHA1
Future encryption types (если нужно)

Убрать флаг RC4_HMAC_MD5, чтобы Kerberos больше не использовал RC4.
Примените политику, выполнив gpupdate /force на всех контроллерах домена и перезагрузив их.

Рекомендация: Потребуйте от пользователей сменить пароли после применения политики, чтобы гарантировать генерацию AES-ключей.

`kerberos Get-ADUser username -Properties PasswordLastSet | Select-Object Name, PasswordLastSet`

Computer Configuration
 └── Windows Settings
     └── Security Settings
         └── Local Policies
             └── Security Options

Interactive logon: Number of previous logons to cache (in case domain controller is not available) установить значение 0 

Рекомендуется отключать только на ПК постоянно имеющих связь с доменом. На сервера рекомендуется ставить хотя бы 1 для некоторых служб для сценариев offline.




1. В чем главная опасность?
Если вы просто выключите RC4, основная проблема возникнет не в самой Windows, а в совместимости.

Старое ПО и оборудование: Старые сетевые сканеры/принтеры, древние версии Java (ниже 7u21), старые ERP-системы или тонкие клиенты могут просто перестать подключаться.

Kerberos: Если в домене всё еще используются типы шифрования RC4_HMAC_MD5 для учетных записей служб или компьютеров, аутентификация упадет.

SQL Server: Старые версии SQL (2008/2012 без патчей) сильно завязаны на RC4 для TLS-соединений.

Доверия (Trusts): Если у вас есть доверительные отношения с очень старыми лесами (NT 4.0/2003), они могут развалиться.

2. Стратегия «Семь раз отмерь» (Подготовка)
Прежде чем лезть в GPO, нужно понять, кто в вашей сети до сих пор «дышит» через RC4.

Шаг 1: Аудит Kerberos
Включите аудит использования типов шифрования Kerberos. Вам нужно искать в журнале событий Security события с ID 4768 (запрос билета TGT) и 4769 (запрос сервисного билета).
В деталях события смотрите на поле Ticket Encryption Type.

0x17 — это и есть наш проблемный RC4-HMAC.

0x12 — это современный AES256-CTS-HMAC-SHA1-96.

Шаг 2: Проверка SSL/TLS
Используйте инструменты вроде Wireshark или сканеры безопасности (Nessus, OpenVAS), чтобы увидеть, какие серверы все еще предлагают RC4 в процессе TLS Handshake.

3. Как сделать правильно (Пошаговый план)
Настройку нужно проводить в два этапа: для протокола TLS (через реестр) и для Kerberos (через политики).

Этап А: Отключение RC4 в Schannel (TLS/SSL)
Windows Server 2012 R2 управляет этим через реестр. В GPO это настраивается через Preferences.

Создайте GPO и перейдите в: Computer Configuration -> Preferences -> Windows Settings -> Registry.

Создайте параметр (Update):

Path: HKLM\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Ciphers\RC4 128/128

Value name: Enabled

Value type: REG_DWORD

Value data: 0 (ноль)

Повторите то же самое для путей:

RC4 40/128

RC4 56/128

RC4 64/128

Совет: Чтобы не делать это руками, лучше всего использовать утилиту IIS Crypto (от Nartac Software) на эталонном сервере, а затем экспортировать настройки реестра в GPO.

Этап Б: Настройка Kerberos
Это критический момент для контроллеров домена.

Откройте GPO: Computer Configuration -> Policies -> Windows Settings -> Security Settings -> Local Policies -> Security Options.

Найдите параметр: Network security: Configure encryption types allowed for Kerberos.

Установите галочки только на:

AES128_HMAC_SHA1

AES256_HMAC_SHA1

Future encryption types

(НЕ отмечайте RC4 и DES).

4. Как не «положить» домен?
Чтобы минимизировать риски, придерживайтесь этой схемы:

Очередность	Действие	Зачем?
1. Пилотная группа	Примените GPO на один тестовый сервер и пару рабочих станций.	Проверить запуск специфичного ПО.
2. Серверы приложений	Примените ко всем серверам, кроме КД.	TLS часто ломает связь с БД или веб-интерфейсы.
3. Контроллеры домена	Последний этап.	Когда вы уверены, что все клиенты умеют в AES.
Что делать, если всё сломалось?
Если после применения политики какой-то сервис выдает ошибку «Encryption type not supported», проверьте вкладку Account в свойствах учетной записи этого сервиса в AD. Убедитесь, что там НЕ стоит галочка «Use Kerberos DES encryption types» и, при необходимости, принудительно обновите пароль этой учетки после включения AES, чтобы перегенерировать ключи.

Важное напоминание: Windows Server 2012 R2 уже вышел из фазы расширенной поддержки (в октябре 2023-го). Отключение RC4 — отличный шаг, но следующим шагом планируйте миграцию на 2022 или 2025 версию.

У вас есть понимание, какие именно приложения в сети могут быть самыми старыми (например, древний SQL или софт на Java 6)?

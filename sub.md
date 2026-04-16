
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

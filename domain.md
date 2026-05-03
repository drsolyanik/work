# Настройка SSSD (Best Practice)
После ввода в домен отредактируйте /etc/sssd/sssd.conf. Опытные админы обычно отключают использование полных имен (FQDN), чтобы вместо user@domain.com писать просто user.

Пример эталонного конфига /etc/sssd/sssd.conf:

```
[sssd]
domains = domain.com
config_file_version = 2
services = nss, pam, sudo

[domain/domain.com]
ad_domain = domain.com
krb5_realm = DOMAIN.COM
realmd_tags = managed-system
cache_credentials = True
id_provider = ad
auth_provider = ad
access_provider = ad

# Использовать короткие имена (без @domain.com)
use_fully_qualified_names = False
# Автоматическое создание домашней директории
fallback_homedir = /home/%u
# Оболочка по умолчанию
default_shell = /bin/bash

# Оптимизация производительности: не перечислять всех пользователей домена
enumerate = False
```

После правок: `systemctl restart sssd`.

# Настройка приоритета (nsswitch.conf)
 
Если вы поставите `use_fully_qualified_names = False`, и в домене будет пользователь admin, и локально будет admin — возникнет конфликт.

Решение: Оставить True (полные имена), но для удобства SSH и Sudo использовать группы. Либо (если вы уверены в нейминге) ставить False, но убедиться, что локальные пользователи имеют уникальные префиксы или имена, не встречающиеся в AD.


Чтобы локальные учетки работали даже при «кривом» SSSD или упавшем домене, они должны стоять первыми в /etc/nsswitch.conf.

```
# Проверяем строки passwd и group
grep -E "passwd|group" /etc/nsswitch.conf
```

Должно быть так: `passwd: files sss`. Это значит: «Сначала ищи в /etc/passwd, если не нашел — иди в SSSD».

# Конфигурация SSSD (`/etc/sssd/sssd.conf`)
Настраиваем кэширование и формат имен.

```
[domain/domain.com]
# ... (базовые настройки AD)

# 1. Включаем кэширование для работы вне сети
cache_credentials = True
# Время жизни кэша в днях (на случай долгой аварии)
account_cache_expiration = 7
entry_cache_timeout = 5400

# 2. Формат имен. 
# Если ставим True -> вход по user@domain.com
# Если ставим False -> вход по user
use_fully_qualified_names = True

# 3. Разделение домена и имени (лучше использовать @ или \)
fallback_homedir = /home/%u@%d
```

# Настройка Sudo (с учетом FQDN)
Если вы выбрали `use_fully_qualified_names = True`, то в `sudoers` нужно указывать полное имя. Мы используем группы.

Создаем файл `/etc/sudoers.d/ad_admins`:

```
# Разрешаем группе "Linux Admins" из AD
# Обратите внимание на экранирование пробелов и использование полного имени группы
%Linux\ Admins@domain.com ALL=(ALL:ALL) ALL

# Если нужно дать права конкретному человеку:
ivanov@domain.com ALL=(ALL:ALL) ALL
```

# Доступ по SSH
В вопросе про SSH: если `use_fully_qualified_names = True`, то в конфигах и при логине используется полное имя.

Как сделать правильно в `/etc/ssh/sshd_config`:

```
# Разрешаем вход локальному админу (обязательно!) и доменной группе
AllowGroups root localadmin "linux admins@domain.com"

# Если используете AllowUsers:
AllowUsers localadmin ivanov@domain.com
```

# Офлайн-доступ и PAM
Чтобы система «пурскала» доменного пользователя, когда DC недоступен, SSSD должен уметь проверять кэшированный хэш пароля. Это включается в PAM.

В Debian/Astra это обычно настраивается через:

```
pam-auth-update
```

(Убедитесь, что галочка на "SSS authentication" и "Create home directory on login" стоит).

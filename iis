# 1
## 1. Подготовка компонентов
Прежде всего убедитесь, что в системе установлена роль Windows Authentication.

Откройте Server Manager -> Manage -> Add Roles and Features.

Перейдите в раздел Web Server (IIS) -> Web Server -> Security.

Убедитесь, что галочка на Windows Authentication и URL Authorization установлена.

## 2. Настройка SSO (Windows Authentication)
Чтобы пользователи входили автоматически без ввода логина и пароля (Single Sign-On), необходимо включить соответствующий метод в IIS.

Откройте IIS Manager и выберите ваш сайт.

В центральной панели откройте раздел Authentication.

Выключите (Disable) пункт Anonymous Authentication.

Включите (Enable) пункт Windows Authentication.

Важно: Чтобы SSO работал в браузере «бесшовно», адрес сайта должен находиться в зоне Local Intranet в настройках безопасности браузера на стороне клиента.

## 3. Ограничение доступа по доменной группе
Теперь нужно настроить IIS так, чтобы он пускал только членов определенной группы, даже если остальные пользователи успешно прошли доменную аутентификацию.

### Вариант А: Через интерфейс IIS Manager (URL Authorization)
Вернитесь в главное меню настроек сайта в IIS и откройте Authorization Rules.

Удалите стандартное правило Allow All Users (если оно есть).

Нажмите Add Allow Rule... в правой панели.

Выберите пункт Specified roles or user groups.

Введите имя вашей группы в формате: DOMAIN\GroupName.

Нажмите OK.

### Вариант Б: Через файл web.config
Вы можете прописать эти правила напрямую в файле конфигурации вашего сайта:

```
<configuration>
    <system.webServer>
        <security>
            <authorization>
                <remove users="*" roles="" verbs="" />
                <add accessType="Allow" roles="DOMAIN\YourGroupName" />
            </authorization>
        </security>
    </system.webServer>
</configuration>
```

## 4. Настройка разрешений NTFS (Важный шаг)
Аутентификация в IIS — это только половина дела. Пользователи из вашей группы должны иметь физический доступ к файлам сайта на диске.

Нажмите правой кнопкой на папку с сайтом (например, C:\inetpub\wwwroot\mysite).

Вкладка Security -> Edit -> Add.

Добавьте вашу группу DOMAIN\GroupName.

Выдайте права Read & execute, List folder contents и Read.

Сохраните изменения.

# 2
## 1. Подготовка сервера и включение необходимых ролей
Убедитесь, что на сервере установлены все необходимые компоненты IIS для работы с доменной аутентификацией.

Откройте Диспетчер сервера → Управление → Добавить роли и компоненты.

На шаге Роли сервера найдите Веб-сервер (IIS) → Службы ролей и убедитесь, что установлены флажки:

Безопасность → Проверка подлинности Windows.

Безопасность → Авторизация URL-адресов.

Завершите установку при необходимости.

## 2. Настройка проверки подлинности (SSO)
Откройте Диспетчер IIS, выберите нужный сайт и дважды кликните по значку Проверка подлинности.

Включите Проверку подлинности Windows, отключив при этом Анонимную проверку подлинности.

Выберите Проверку подлинности Windows и нажмите Поставщики... в панели Действия.

Убедитесь, что в списке поставщиков протокол Negotiate стоит выше, чем NTLM (он должен быть первым). Это обеспечит приоритетное использование Kerberos для прозрачного входа.

## 3. Настройка авторизации по доменной группе
Авторизация определяет, кто именно может заходить на сайт.

В Диспетчере IIS выберите ваш сайт и дважды кликните по значку Правила авторизации.

В панели Действия выберите Добавить правило разрешения....

В открывшемся окне выберите Разрешить доступ для указанной роли или группы пользователей.

В поле ввода укажите вашу доменную группу в формате ДОМЕН\ИмяГруппы (например, CONTOSO\WebAppUsers). Нажмите ОК.

Важно: Убедитесь, что группа является глобальной группой безопасности в Active Directory. Локальные доменные группы могут не распознаваться IIS для авторизации на уровне сайта.

Чтобы запретить доступ всем остальным, в панели Действия выберите Добавить правило запрета... и в появившемся окне выберите Запретить доступ для всех пользователей.

## 4. Настройка Kerberos для корректной работы SSO
Если сайт доступен не по NetBIOS-имени сервера, а по полному доменному имени (например, portal.contoso.com), для прозрачного входа Kerberos потребуется регистрация SPN (Service Principal Name).

На контроллере домена откройте Командную строку от имени администратора.

Выполните проверку, не заняты ли необходимые SPN записи двойной командой:
setspn -x

Добавьте SPN-записи для вашего сайта. Проще всего выполнить привязку на сервисную учётную запись, которая будет использоваться для пула приложений сайта. Выполните следующие команды:
setspn -s HTTP/portal.contoso.com contoso\service_account
setspn -s HTTP/portal contoso\service_account

Вернитесь на веб-сервер и настройте пул приложений вашего сайта:

В Диспетчере IIS выберите Пул приложений вашего сайта и нажмите Дополнительные параметры.

В разделе Модель процесса найдите параметр Удостоверение и укажите ту же сервисную учётную запись, для которой вы зарегистрировали SPN.

Далее, настройте проверку подлинности Windows на использование учётных данных пула приложений:

Выберите ваш сайт и дважды кликните по значку Редактор конфигураций.

В выпадающем списке Раздел выберите system.webServer/security/authentication/windowsAuthentication.

В списке свойств найдите useAppPoolCredentials и установите для него значение True.

Нажмите Применить в панели Действия и выполните из командной строки iisreset для перезапуска служб.

# 3 
## 1. Включаем SSO (Windows Authentication)
В IIS:
Открой IIS Manager
Выбери сайт
Перейди в Authentication
Сделай:
❌ Anonymous Authentication → Disable
✅ Windows Authentication → Enable
⚙️ Настройка Windows Authentication:

Открой Windows Authentication → Advanced Settings:

Extended Protection → Off (если нет строгих требований)
Kernel-mode authentication → Enabled (обычно ок)

Providers:

Убедись, что порядок:

Negotiate
NTLM

👉 Negotiate = Kerberos (SSO)

## 2. Условия для SSO (очень важно)

Чтобы реально работал SSO без запроса логина:

На клиенте:
ПК должен быть в домене
Пользователь залогинен в домен
В браузере:
Для Chrome / Edge:

Добавь сайт в Local Intranet Zone:

Internet Options → Security → Local intranet → Sites → Advanced

Добавь:

http://your-site

👉 Или через GPO:

Computer Configuration → Administrative Templates → Windows Components → Internet Explorer → Internet Control Panel → Security Page → Site to Zone Assignment List
## 3. Ограничение доступа по доменной группе

Есть 2 варианта:

✅ Вариант 1 (рекомендуется): через Authorization Rules в IIS
В IIS:
Открой сайт
Перейди в Authorization Rules

Удали:

Allow All Users

Добавь:

Allow → Roles or user groups
DOMAIN\YourGroup

Добавь в конце:

Deny → All Users
✅ Вариант 2: через web.config
```
<configuration>
  <system.webServer>
    <security>
      <authorization>
        <remove users="*" roles="" verbs="" />
        <add accessType="Allow" roles="DOMAIN\YourGroup" />
        <add accessType="Deny" users="*" />
      </authorization>
    </security>
  </system.webServer>
</configuration>
```

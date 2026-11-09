

# Настройка 2FA для SSH через Google Authenticator на Astra Linux

Инструкция по настройке двухфакторной аутентификации (TOTP) для SSH-сервера в Astra Linux (Parsec). Модуль собирается из исходников, так как отсутствует в стандартных репозиториях дистрибутива.

> ⚠️ **КРИТИЧЕСКОЕ ПРЕДУПРЕЖДЕНИЕ**
> **Не закрывайте текущую активную сессию SSH** до момента успешной проверки входа через новое окно терминала. Если вы потеряете доступ, восстановление возможно только через физический консольный доступ к серверу.

## 1. Установка зависимостей

Обновите списки пакетов и установите инструменты для сборки и генерации QR-кодов:

```bash
sudo apt update
sudo apt install -y build-essential libpam0g-dev qrencode git autoconf automake libtool pkg-config
```

## 2. Сборка модуля Google Authenticator

Склонируйте официальный репозиторий и соберите PAM-модуль:

```bash
cd /tmp
git clone https://github.com/google/google-authenticator-libpam.git
cd google-authenticator-libpam
./bootstrap.sh
./configure
make
sudo make install
```

### Исправление пути к библиотеке
По умолчанию модуль устанавливается в `/usr/local/lib/security/`, но PAM в Astra Linux ищет модули в `/lib/x86_64-linux-gnu/security/`. Создайте символическую ссылку:

```bash
sudo ln -s /usr/local/lib/security/pam_google_authenticator.so /lib/x86_64-linux-gnu/security/pam_google_authenticator.so
sudo ldconfig
```

## 3. Настройка пользователя

Запустите мастер настройки для каждого пользователя, которому требуется удаленный доступ:

```bash
google-authenticator
```

Рекомендуемые ответы на вопросы мастера:

| Вопрос | Ответ | Пояснение |
| :--- | :--- | :--- |
| Time-based tokens? | `y` | Использование TOTP (совместимо с Google Auth) |
| Update file? | `y` | Сохранение секретного ключа |
| Disallow multiple uses? | `y` | Запрет повторного использования токена (защита от replay-атак) |
| Increase time window? | `n` | Стандартное окно ±30 сек достаточно при синхронизированном времени |
| Rate-limiting? | `y` | Блокировка после 3 неудачных попыток за 30 секунд |

📱 **Отсканируйте QR-код** приложением Google Authenticator (или аналогом) на телефоне.
💾 **Сохраните резервные коды** в надежном месте! Они нужны для восстановления доступа при потере телефона.

## 4. Настройка PAM

Отредактируйте конфигурационный файл SSH PAM:

```bash
sudo nano /etc/pam.d/sshd
```

Добавьте строку вызова модуля **строго после** `@include common-auth`:

```text
@include common-auth
auth required pam_google_authenticator.so nullok
```

> **Примечание:** Параметр `nullok` временно разрешает вход без 2FA пользователям, у которых она еще не настроена. Удалите его на финальном этапе настройки.

## 5. Настройка SSHD

Откройте конфигурацию SSH-сервера:

```bash
sudo nano /etc/ssh/sshd_config
```

Найдите и приведите параметры аутентификации к следующему виду:

```text
KbdInteractiveAuthentication yes
ChallengeResponseAuthentication no
PasswordAuthentication yes
UsePAM yes
```

> ⚠️ **Важно:** Нельзя оставлять включенными одновременно `ChallengeResponseAuthentication` и `KbdInteractiveAuthentication`. Это вызывает бесконечный цикл запросов пароля и кода. Оставьте только `KbdInteractiveAuthentication yes`.

Проверьте наличие переопределяющих конфигов:
```bash
ls -la /etc/ssh/sshd_config.d/
cat /etc/ssh/sshd_config.d/*.conf 2>/dev/null
```
Если там есть файлы с `KbdInteractiveAuthentication no` или `PasswordAuthentication no` — исправьте их.

## 6. Перезапуск и проверка

Перезапустите службу SSH:

```bash
sudo systemctl restart sshd
```

**Из нового окна терминала** выполните тестовое подключение:

```bash
ssh techsup@astra-08210
```

Ожидаемая последовательность запросов:
1.  `Password:` → ввод пароля пользователя
2.  `Verification code:` → ввод 6-значного кода из приложения
3.  Успешный вход ✅

## 7. Финализация безопасности

После подтверждения работоспособности 2FA для всех необходимых пользователей:

1.  Уберите параметр `nullok`, чтобы запретить вход без второго фактора:
    ```bash
    sudo sed -i 's/pam_google_authenticator.so nullok/pam_google_authenticator.so/' /etc/pam.d/sshd
    ```
2.  Перезапустите SSH:
    ```bash
    sudo systemctl restart sshd
    ```

# Authentik Setup Guide

Полное руководство по настройке и использованию Authentik для аутентификации и авторизации через OAuth2/OpenID Connect.

## Содержание

- [Настройка](#настройка)
- [Запуск](#запуск)
- [Настройка почты](#настройка-почты)
- [Тестирование почты](#тестирование-почты)
- [Создание OAuth2 приложения](#создание-oauth2-приложения)
- [Тестирование OAuth2](#тестирование-oauth2)
- [Структура проекта](#структура-проекта)

## Настройка

### Шаг 1: Подготовка файла конфигурации

Скопируйте конфигурации .env.example в .env

### Шаг 2: Генерация секретного ключа

Сгенерируйте безопасный секретный ключ для Authentik

Вставьте секретный ключ в `.env` в переменную `AUTHENTIK_SECRET_KEY`.

### Шаг 3: Настройка переменных окружения

Откройте `.env` и настройте следующие обязательные параметры:

#### База данных
```env
PG_DB=authentik
PG_USER=authentik
PG_PASS=ваш-надежный-пароль
```

#### Authentik Secret Key
```env
AUTHENTIK_SECRET_KEY=ваш-сгенерированный-ключ
```

#### Супер-администратор (создается автоматически при первом запуске)
```env
AUTHENTIK_BOOTSTRAP_EMAIL=admin@localhost
AUTHENTIK_BOOTSTRAP_PASSWORD=ваш-пароль-админа
AUTHENTIK_BOOTSTRAP_USERNAME=admin
```

#### Порты (опционально)
```env
COMPOSE_PORT_HTTP=9000
COMPOSE_PORT_HTTPS=9443
```

## Запуск

### Запуск контейнеров

```powershell
docker-compose up -d
```

### Проверка статуса

```powershell
docker-compose ps
```

Все сервисы должны быть в статусе `Up`.

### Остановка

```powershell
docker-compose down
```

### Остановка с удалением данных (⚠️ осторожно!)

```powershell
docker-compose down -v
```

## Настройка почты

Authentik может отправлять email уведомления (сброс пароля, подтверждение email и т.д.).

Добавьте в `.env` следующие переменные для настройки SMTP:

```env
AUTHENTIK_EMAIL__HOST=smtp.example.com
AUTHENTIK_EMAIL__PORT=587
AUTHENTIK_EMAIL__USERNAME=your-email@example.com
AUTHENTIK_EMAIL__PASSWORD=your-password
AUTHENTIK_EMAIL__USE_TLS=true
AUTHENTIK_EMAIL__USE_SSL=false
AUTHENTIK_EMAIL__FROM=your-email@example.com
```

**Примечания:**
- Для Gmail требуется пароль приложения (не обычный пароль). Создайте его в [настройках аккаунта Google](https://myaccount.google.com/apppasswords)
- Для серверов с SSL используйте `AUTHENTIK_EMAIL__USE_SSL=true` и `AUTHENTIK_EMAIL__USE_TLS=false` (обычно порт 465)
- Для серверов с TLS используйте `AUTHENTIK_EMAIL__USE_TLS=true` и `AUTHENTIK_EMAIL__USE_SSL=false` (обычно порт 587)
- `AUTHENTIK_EMAIL__FROM` должен совпадать с `AUTHENTIK_EMAIL__USERNAME` для большинства провайдеров

### Применение изменений

После изменения настроек почты перезапустите контейнеры:

```powershell
docker-compose restart server worker
```

## Тестирование почты

Отправьте тестовое письмо через командную строку:

```powershell
docker-compose exec worker ak test_email your-email@example.com
```

Замените `your-email@example.com` на ваш email адрес.

Если настройки почты корректны, вы получите письмо на указанный адрес.

## Создание OAuth2 приложения

### Шаг 1: Создание OAuth2 Provider

1. Откройте `http://localhost:9000`
2. Войдите как администратор
3. Перейдите в **Applications** → **Providers**
4. Нажмите **Create**
5. Выберите **OAuth2/OpenID Провайдер**
6. Заполните форму:
   - **Name**: `Test OAuth2 Provider` (или любое другое имя)
   - **Client type**: `Confidential` (для server-to-server) или `Public` (для SPA)
   - **Redirect URIs**: `http://localhost:8080/callback` (или ваш callback URL)
   - **Scopes**: выберите `openid`, `profile`, `email`
   - **Issuer mode**: `Each provider has a different issuer, based on the application slug` (рекомендуется)
   - **Sub mode**: `user_username` или `user_id`
7. Нажмите **Create**
8. **Сохраните Client ID и Client Secret** — они понадобятся для тестирования

### Шаг 2: Создание Application

1. Перейдите в **Applications** → **Applications**
2. Нажмите **Create**
3. Заполните форму:
   - **Name**: `Test Application` (или любое другое имя)
   - **Slug**: `test-application` (автогенерируется, можно изменить)
   - **Provider**: выберите созданный провайдер из списка
   - **Launch URL**: можно оставить пустым
4. Нажмите **Create**

### Шаг 3: Получение Client ID и Client Secret

1. Вернитесь в **Applications** → **Providers**
2. Откройте созданный провайдер
3. Скопируйте:
   - **Client ID**
   - **Client Secret** (нажмите "Show" чтобы увидеть)

## Тестирование OAuth2

### Тест 1: Client Credentials Flow (Server-to-Server)

Этот flow используется для server-to-server аутентификации без участия пользователя.

#### В Postman:

1. Создайте новый запрос:
   - **Method**: `POST`
   - **URL**: `http://localhost:9000/application/o/token/`

2. Вкладка **Headers**:
   ```
   Content-Type: application/x-www-form-urlencoded
   ```

3. Вкладка **Body**:
   - Выберите `x-www-form-urlencoded`
   - Добавьте параметры:

   | Key | Value |
   |-----|-------|
   | grant_type | client_credentials |
   | client_id | ВАШ_CLIENT_ID |
   | client_secret | ВАШ_CLIENT_SECRET |
   | scope | openid |

4. Нажмите **Send**

#### Ожидаемый ответ (200 OK):
```json
{
    "access_token": "eyJ0eXAiOiJKV1QiLCJhbGc...",
    "expires_in": 3600,
    "token_type": "Bearer",
    "scope": "openid"
}
```

### Тест 2: Authorization Code Flow (с информацией о пользователе)

Этот flow используется для получения информации о пользователе через интерактивную авторизацию.

#### Шаг 2.1: Получение Authorization Code

1. Откройте в браузере следующий URL (замените значения на ваши):
   ```
   http://localhost:9000/application/o/authorize/?client_id=ВАШ_CLIENT_ID&response_type=code&redirect_uri=http://localhost:8080/callback&scope=openid+profile+email
   ```

   **Важно**: `redirect_uri` должен точно совпадать с тем, что указано в настройках провайдера.

2. Авторизуйтесь используя ваши bootstrap credentials

3. После авторизации браузер перенаправит на:
   ```
   http://localhost:8080/callback?code=AUTHORIZATION_CODE
   ```

4. Скопируйте значение параметра `code` из адресной строки

#### Шаг 2.2: Обмен code на токены

В Postman:

1. Создайте новый запрос:
   - **Method**: `POST`
   - **URL**: `http://localhost:9000/application/o/token/`

2. Вкладка **Headers**:
   ```
   Content-Type: application/x-www-form-urlencoded
   ```

3. Вкладка **Body**:
   - Выберите `x-www-form-urlencoded`
   - Добавьте параметры:

   | Key | Value |
   |-----|-------|
   | grant_type | authorization_code |
   | code | ВАШ_AUTHORIZATION_CODE |
   | client_id | ВАШ_CLIENT_ID |
   | client_secret | ВАШ_CLIENT_SECRET |
   | redirect_uri | http://localhost:8080/callback |

4. Нажмите **Send**

#### Ожидаемый ответ (200 OK):
```json
{
    "access_token": "eyJ0eXAiOiJKV1QiLCJhbGc...",
    "token_type": "Bearer",
    "expires_in": 3600,
    "refresh_token": "eyJ0eXAiOiJKV1QiLCJhbGc...",
    "id_token": "eyJ0eXAiOiJKV1QiLCJhbGc...",
    "scope": "openid profile email"
}
```

#### Шаг 2.3: Получение информации о пользователе (UserInfo)

1. Создайте новый запрос:
   - **Method**: `GET`
   - **URL**: `http://localhost:9000/application/o/userinfo/`

2. Вкладка **Headers**:
   ```
   Authorization: Bearer ВАШ_ACCESS_TOKEN
   Content-Type: application/json
   ```

3. Нажмите **Send**

#### Ожидаемый ответ (200 OK):
```json
{
    "sub": "user_username",
    "email": "admin@localhost",
    "email_verified": true,
    "name": "admin",
    "preferred_username": "admin",
    "groups": []
}
```

### Тест 3: Проверка OpenID Configuration

Получите конфигурацию OpenID Connect через Postman:

1. Создайте новый запрос:
   - **Method**: `GET`
   - **URL**: `http://localhost:9000/application/o/test-application/.well-known/openid-configuration`

   **Примечание**: Используйте slug вашего приложения вместо `test-application`.

2. Нажмите **Send**

Это вернет все доступные endpoints и настройки провайдера в формате JSON.

## Структура проекта

```
authentik/
├── authentik/              # Все данные Authentik
│   ├── certs/             # SSL сертификаты
│   ├── data/              # Данные базы данных
│   │   └── postgres/      # PostgreSQL данные
│   ├── media/             # Медиа файлы Authentik
│   └── templates/         # Кастомные шаблоны
├── docker-compose.yml      # Docker Compose конфигурация
├── .env                   # Переменные окружения (не в git)
├── .env.example           # Пример конфигурации
├── .gitignore             # Git ignore правила
└── README.md              # Эта документация
```

## Ссылки на документацию

- [Официальная документация Authentik](https://goauthentik.io/docs/)
- [OAuth2 спецификация](https://oauth.net/2/)
- [OpenID Connect спецификация](https://openid.net/connect/)

## Лицензия

Этот проект использует Authentik, который распространяется под лицензией MIT.

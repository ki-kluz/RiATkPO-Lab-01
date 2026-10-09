Микросервис выступает в роли "оркестратора". Он не имеет собственной БД (не считая легковесных), все данные он берет и кладет через REST API.

### 3.1. Внешние системы (Reference)
*   **AmoCRM API:** Используется для получения данных о лиде. [Официальная документация AmoCRM](https://www.amocrm.ru/developers/content/crm_platform/api-reference) 
*   **AlfaCRM API:** Используется для управления расписанием и клиентами. [REST API Альфа CRM: Документация для разработчиков](https://alfacrm.pro/usefull/integrations/integration/api#znachenie-nekotoryh-parametrov)

**Важно:** Микросервис должен быть легковесным маршрутизатором (stateless). Разрешается использовать только in-memory кеш для хранения временных токенов и дедупликации webhook-ов. Реляционных БД у микросервиса не будет, чтобы не плодить дубликаты данных.

### 3.2. Внутренний сервис (Internal Platform API)
Так как Внутренний сервис является кастомной разработкой компании, ниже представлена архитектура HTTP REST, которую предоставляет IT-отдел для микросервиса.

**Базовый URL:** `https://internal.edusync.ru/api/v1`
**Авторизация:** X-API-KEY: {Service_Token} в заголовках.
**Лимиты (Rate Limit):** Не более 50 запросов в секунду. В случае превышения возвращает 429 Too Many Requests.

#### 1. Создание пользователя во внутренней системе
**POST** `/users/create`
Создает профиль Пользователя.
*Request body (JSON):*
```json
{
  "role_id": number, // 100 - student, 200 - teacher, 300 - sales, 400 - admin
  "external_sync_id": string, // ID из внешней системы (Amo или Alfa)
  "profile": {
    "first_name": string,
    "last_name": string,
    "birth_date": string, // format: YYYY-MM-DD (optional)
    "timezone": string // format: "Europe/Moscow"
  },
  "contacts": {
    "email": string, 
    "phone": string 
  },
  "settings": {
    "receive_newsletters": boolean
  }
}
```
*Response (201 Created):*
```json
{
  "success": boolean,
  "data": {
    "id": number,
    "login": string, // email
    // Password - send to email
  }
}
```

#### 2. Создание связи "Преподаватель - Ученик"
**POST** `/teachers/assign-students`
*Request body (JSON):*
```json
{
  "student_ids": number[],
  "teacher_id": number,
  "status": number, // 0 - active, 1 - suspended
}
```

#### 3. Модуль Мессенджера (Чаты)
Во внутренней системе люди не могут просто "написать" друг другу. Сначала нужно создать сущность "Чат" (Chat Room), а затем добавлять туда сообщения.

**POST** /messenger/chats (Создание комнаты)
*Request body (JSON):*
```json
{
  "chat_type": number, // 0 - p2p - личный, 1 - group - групповой
  "participant_ids": number[],
  "subject": string // опционально
}
```
*Response (200 OK):* `{"chat_id": number}`

**PATCH** `/messenger/chats/{chat_id}/status` (Управление доступом к чату)  
Позволяет "заморозить" чат. Пользователи смогут читать историю, но не писать.
*Request body (JSON):*
```json
{
  "status": number, // 0 - "active", 1 - "frozen", 2 - "archived"
  "reason": string // Причина для вывода пользователю
}
```


#### 4. Модуль Уведомлений
**POST** /notifications/push  
Отправляет системное всплывающее окно (не в чат, а в интерфейс колокольчика).
*Request body (JSON):*
```json
{
  "target_user_ids": number[],
  "type_code": number, // 31 - billing_alert, 41 - schedule, 51 - gamification
  "title": string,
  "body": string,
  "action_link": string // Ссылка для перехода
}
```

**POST** `/messenger/messages/send_system` 
Отправляет сообщение в конкретный чат от лица Системного Бота.
*Request body (JSON):*
```json
{
  "chat_id": number,
  "text": string,
  "priority": number // 0 - "low", 1 - "normal", 2 - "high"
}
```

#### 5. Модуль Геймификации
**POST** /gamification/points/award  
Начисляет ученику внутреннюю валюту (звездочки) за вовремя сделанную домашку.
*Request body (JSON):*
```json
{
  "user_id": number,
  "amount": number,
  "reason_code": number
}
```


#### 6. Журналирование (Аналитика)
**POST** /telemetry/events  
Отправка логов во внутреннюю систему аналитики платформы.
*Request body (JSON):*
```json
{
  "event_name": string, // "lead_converted", "student_blocked", etc.
  "timestamp": "ISO 8601",
  "payload": {
     // Любые ключ-значения
  }
}
```


#### 7. Webhooks Внутренней системы (Исходящие)
Настраиваются в админке. Система шлет POST-запросы на ваш микросервис.  
*Событие 1:* Изменение привязки преподавателя вручную администратором.  
**POST** `{Ваш_Микросервис}/webhooks/internal`

```json
{
  "event": string,
  "data": {
    "student_internal_id": number,
    "old_teacher_internal_id": number,
    "new_teacher_internal_id": number,
    "initiated_by_id": number
  }
}
```

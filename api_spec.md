# Спецификация API MHP3 (Protobuf) 🛡️

Бинарная обертка для всех сообщений и системных событий протокола MHP3. Актуальная версия соответствует реализации в SDK и серверном хэндлере.

## Структура конверта (Envelope)

```protobuf
syntax = "proto3";

package mhp3;

import "google/protobuf/timestamp.proto";

// --- CORE ENVELOPE ---

message MHP3Envelope {
    bytes id = 1;       // UUID v7 / ULID (16 bytes binary)
    uint64 pts = 2;     // Persistent Tracking Sequence (Global PTS)
    
    oneof content {
        // Session
        SessionHandshake session_handshake = 3;
        SessionEstablished session_established = 4;
        AuthRequired auth_required = 5;
        
        // Messaging
        MessageSend msg_send = 6;
        MessageDelivery msg_delivery = 7;
        MessageStatusUpdate msg_status_update = 8;
        MessageStatusBatch msg_status_batch = 9;
        
        // Поля 10-12: MLS (EpochSync, Commit, Welcome)
        
        // Status & Presence
        TypingStatus typing_status = 13;
        UserStatusUpdate user_status_update = 14;
        PresenceCheck presence_check = 15;
        PresenceResponse presence_response = 16;
        FCMRegistration fcm_registration = 17;
        
        // System
        Ping ping = 18;
        Pong pong = 19;
        SystemNotification system_notification = 20;
        ErrorResponse error = 21;

        // History & Sync
        HistoryRecoveryRequest history_recovery_request = 22;
        HistoryRecoveryResponse history_recovery_response = 23;
        DeviceRelayRequest device_relay_request = 24;
        DeviceRelayResponse device_relay_response = 25;
        UpdatesGetDiff updates_get_diff = 26;
        UpdatesDiff updates_diff = 27;

        // Interactions (ZK)
        MessageEdit msg_edit = 30;
        MessageReaction msg_reaction = 31;
        MessagePin msg_pin = 32;
        MessageDelete msg_delete = 33;

        // Private Contact Discovery (PCD)
        ContactDiscoveryRequest contact_discovery_request = 40;
        ContactDiscoveryResponse contact_discovery_response = 41;

        // System Greetings
        SystemGreeting system_greeting = 42;
    }

    map<string, string> metadata = 50;
}
```

---

## Описание типов контента

### 1. Session Handshake (Client -> Server)
Инициализация сессии и синхронизация G-PTS.

| Поле | Тип | Описание |
| :--- | :--- | :--- |
| `token` | `string` | JWT токен аутентификации. |
| `noise_pub_key` | `bytes` | Публичный ключ для Noise-согласования (X25519). |
| `last_pts` | `uint64` | Последний известный глобальный PTS клиента. |
| `device_id` | `string` | Уникальный ID устройства. |
| `last_pts_map` | `map<string, uint64>` | Карта локальных PTS для чатов `{chat_id: pts}`. |
| `pts` | `uint64` | Актуальное состояние G-PTS клиента (дублирует last_pts). |

### 2. Session Established (Server -> Client)
Подтверждение сессии и передача актуального состояния сервера.

| Поле | Тип | Описание |
| :--- | :--- | :--- |
| `user_id` | `string` | Unique ID пользователя. |
| `pts` | `uint64` | Текущий глобальный PTS сервера. |
| `pts_map` | `map<string, uint64>` | Текущая серверная карта PTS для всех чатов. |
| `server_time` | `Timestamp` | Время сервера для синхронизации. |
| `noise_resp_key` | `bytes` | Ответный ключ для Noise-согласования. |

### 3. Messaging

#### Message Send (Client -> Server)
| Поле | Тип | Описание |
| :--- | :--- | :--- |
| `chat_id` | `int32` | ID чата (0 для личных сообщений). |
| `recipient_id` | `string` | ID получателя (unique_id). |
| `payload` | `bytes` | Зашифрованное (E2EE) содержимое сообщения. |
| `type` | `MessageType` | TEXT, IMAGE, etc. |
| `priority` | `uint32` | Приоритет (0-2). |
| `epoch` | `uint64` | Эпоха MLS для групповых чатов. |

#### Message Delivery (Server -> Client)
| Поле | Тип | Описание |
| :--- | :--- | :--- |
| `message_id` | `bytes` | UUID v7 сообщения. |
| `sender_id` | `string` | ID отправителя. |
| `pts` | `uint64` | Порядковый номер сообщения в чате (C-PTS). |
| `payload` | `bytes` | Зашифрованные данные. |
| `timestamp` | `uint64` | Время отправки (ms). |
| `file_meta` | `FileMetadata` | Метаданные (если есть файл). |

### 4. Status & Presence

#### User Status Update
| Поле | Тип | Описание |
| :--- | :--- | :--- |
| `user_id` | `string` | ID пользователя. |
| `status` | `PresenceStatus` | Текущее состояние (ONLINE, AWAY, и т.д.). |
| `last_seen` | `Timestamp` | Время последнего онлайна. |

#### Typing Status
| Поле | Тип | Описание |
| :--- | :--- | :--- |
| `chat_id` | `int32` | ID чата. |
| `user_id` | `string` | Кто печатает. |
| `is_typing` | `bool` | true/false. |

### 5. Interactions (Zero-Knowledge)

| Сообщение | Поля | Тип chatId | Описание |
| :--- | :--- | :--- | :--- |
| `msg_edit` | `message_id`, `chat_id`, `encrypted_new_payload` | `uint64` | Редактирование (замена контента). |
| `msg_reaction` | `message_id`, `chat_id`, `reaction_code`, `is_removed` | `uint64` | Реакция (эмодзи). |
| `msg_pin` | `message_id`, `chat_id`, `is_pinned` | `uint64` | Закрепление сообщения. |
| `msg_delete` | `message_id`, `chat_id`, `delete_for_all` | `uint64` | Удаление сообщения. |

### 6. History & Sync

#### Updates Diff (Server -> Client)
| Поле | Тип | Описание |
| :--- | :--- | :--- |
| `updates` | `repeated MHP3Envelope` | Список пропущенных событий. |
| `current_pts_map` | `map<string, uint64>` | Актуальные PTS чатов на момент ответа. |

#### History Recovery Request
| Поле | Тип | Описание |
| :--- | :--- | :--- |
| `chat_id` | `int32` | Целевой чат. |
| `from_pts` | `uint64` | Начало диапазона восстановления. |
| `request_from_trusted_device` | `bool` | Попытка восстановить через D2D Relay. |

---

## Перечисления (Enums)

### MessageType
TEXT = 0, IMAGE = 1, VIDEO = 2, AUDIO = 3, FILE = 4, LOCATION = 5, CONTACT = 6, STAMP = 7, VOICE_CALL = 8, VIDEO_CALL = 9.

### PresenceStatus
OFFLINE = 0, ONLINE = 1, AWAY = 2, DND = 3, INVISIBLE = 4.

### MessageStatus
SENT = 0, DELIVERED = 1, READ = 2, DELETED = 3, EDITED = 4.

---

## Коды Ошибок
| Код | Message | Описание |
| :--- | :--- | :--- |
| `401` | `handshake_required` | Требуется выполнить `session_handshake`. |
| `403` | `forbidden` | Ошибка доступа (токен невалиден или нет доступа к чату). |
| `409` | `epoch_outdated` | Эпоха MLS устарела, требуется обновление ключей группы. |
| `500` | `internal_error` | Ошибка на стороне сервера. |

# Развертывание и Компиляция MHP3

Протокол MHP3 использует бинарный формат Protobuf для передачи данных. Для успешного запуска сервера и клиента необходимо сгенерировать соответствующие классы.

## 1. Компиляция Protobuf

### Установка компилятора
Для компиляции потребуется `protoc`. В Windows можно установить через `vcpkg` или `chocolatey`:
```bash
choco install protoc
```

### Генерация Python-классов (Сервер)
Находясь в корне проекта (`/server`), выполните:
```bash
protoc --python_out=. --grpc_python_out=. app/mhp/mhp3.proto
```
Это создаст файл `app/mhp/mhp3_pb2.py`, который импортируется в `MHP3Handler`.

### Генерация Dart-классов (Клиент Flutter)
Для клиента Flutter используйте плагин `protoc_plugin`:
```bash
protoc --dart_out=grpc:lib/src/generated -Iapp/mhp app/mhp/mhp3.proto
```

## 2. Конфигурация сервера

Для работы MHP3 необходимо настроить переменные окружения в `.env`:
```env
# PTS Настройки (Плавающий TTL)
HISTORY_TTL_DAYS=20          # Окно для активных пользователей (20 дней)
HISTORY_HARD_TTL_DAYS=240    # Максимальное окно оффлайна (8 месяцев)
PTS_RECONCILE_INTERVAL=60

# MHP3 Настройки
MHP3_HANDSHAKE_MANDATORY=true
MHP3_NOISE_ENABLED=true

# База данных (обязательно PostgreSQL 15+)
DB_URL=postgresql+asyncpg://user:pass@localhost:5432/knot_db
```

## 3. Фоновые задачи (Background Tasks)

Для поддержания целостности PTS необходимо запустить фоновые процессы:
1.  **Reconciliation Task**: Синхронизация Redis-инкрементов PTS с PostgreSQL.
2.  **Cleanup Task**: Удаление старой истории обновлений (`updates_history`) старше 28 дней.
3.  **Key Rotation**: Автоматическая ротация серверных ключей шифрования.

Эти задачи запускаются автоматически при инициализации `MHP3Handler` в `transport_protocol.py`.

> [!IMPORTANT]
> **Плавающий TTL** работает автоматически: если пользователь онлайн, история старше 20 дней удаляется. Если пользователь оффлайн, история копится до 8 месяцев. При первом же заходе в онлайн после долгого отсутствия, пользователь получает всю историю за 8 месяцев, после чего окно для него снова схлопывается до 20 дней.

---

## Protocol Conformance Tests (Тесты соответствия протоколу)

Чтобы гарантировать, что все реализации MHP3 (Python, Go, Dart) работают идентично, необходимо автоматизировать следующие тесты:

### Test Case 1: Basic Gap Recovery
**Сценарий:** Клиент отключается на G-PTS 10, сервер генерирует события 11, 12, 13.
```bash
# Setup
Client.disconnect(after_g_pts=10)
Server.generate_events(user_id, count=3)  # G-PTS 11, 12, 13

# Test
Client.connect()
Client.send_handshake(last_g_pts=10)

# Expected Result
Client.receives_events([11, 12, 13])  # В строгом порядке
Client.state == "READY"
Client.last_g_pts == 13
```

### Test Case 2: C-PTS Inconsistency Detection
**Сценарий:** В чате есть C-PTS разрыв (45 → 50).
```bash
# Setup
Server.set_chat_cpts(chat_id="A", c_pts=45)
Server.force_cpts_gap(chat_id="A", new_c_pts=50)  # Имитация очистки

# Test
Client.connect()
Client.send_handshake(last_g_pts=100, c_pts_map={"A": 45})

# Expected Result
Client.receives_inconsistency_flag(chat_id="A")
Client.chat_state["A"] == "INCONSISTENT"
Client.requests_snapshot_for_chat("A")
```

### Test Case 3: Idempotency Key Test
**Сценарий:** Клиент отправляет одно сообщение дважды (плохой интернет).
```bash
# Setup
message_id = UUIDv7()
Client.send_message(message_id=message_id)
# Network timeout, retry
Client.send_message(message_id=message_id)  # Same ID

# Expected Result
Server.receives_only_one_message()
Client.receives_single_ack(g_pts=X, c_pts=Y)
Database.contains_exactly_one_message(message_id)
```

### Test Case 4: Atomic Transaction Test
**Сценарий:** Сервер падает после инкремента G-PTS, но до записи в историю.
```bash
# Setup
Server.crash_after_step(step=2)  # После G-PTS инкремента

# Test
Client.send_message()
Server.restart()
Client.reconnect()

# Expected Result
Client.last_g_pts == previous_g_pts  # Откатился
Server.messages_count == previous_count
No orphaned G-PTS values in database
```

### Test Case 5: Cold Start Anchoring
**Сценарий:** Новый клиент (G-PTS = 0) подключается к активной системе.
```bash
# Setup
Server.current_g_pts = 1_000_000
Server.has_history(from_g_pts=500_000, to=1_000_000)

# Test
NewClient.connect(last_g_pts=0)

# Expected Result
NewClient.receives_error("ERROR_PTS_TOO_OLD")
NewClient.requests_snapshot()
NewClient.receives_anchor_g_pts(1_000_000)
NewClient.last_g_pts == 1_000_000
NewClient.ignores_events_below(1_000_000)
```

### Автоматизация
Рекомендуется создать CI/CD pipeline, который:
1. Запускает тестовые серверы (Python/Go реализации)
2. Запускает тестовые клиенты (Dart/Python реализации)
3. Проверяет соответствие всех сценариев
4. Генерирует отчет о совместимости

**Success Criteria:** Все реализации проходят 100% тестов без отклонений в поведении.

---
*Pixeltoo Lab (2026)*

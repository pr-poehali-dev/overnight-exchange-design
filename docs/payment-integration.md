# 💳 Интеграция с платежными провайдерами

## 🎯 Обзор

Система поддерживает автоматическую обработку криптовалютных платежей через 4 провайдера:
- **Coinbase Commerce** — простая интеграция для BTC, ETH, USDT
- **NOWPayments** — широкий выбор криптовалют (100+)
- **CoinPayments** — классический провайдер с поддержкой altcoins
- **Binance Pay** — быстрые платежи через Binance

---

## 📊 Архитектура

### База данных

#### Таблица `payment_providers`
Хранит конфигурацию провайдеров:
```sql
- id: уникальный ID
- name: название провайдера
- type: тип (coinbase_commerce, nowpayments, etc)
- is_active: включен/выключен
- supported_currencies: массив поддерживаемых валют
- config: JSON с настройками провайдера
```

#### Таблица `payment_provider_transactions`
Отслеживает платежи:
```sql
- id: ID транзакции
- provider_id: ссылка на провайдера
- exchange_id: ссылка на обмен
- external_transaction_id: ID в системе провайдера
- payment_url: ссылка для оплаты
- amount: сумма платежа
- currency: валюта
- status: pending/processing/completed/failed/expired
- confirmations: количество подтверждений
- payment_address: адрес кошелька
```

---

## 🔧 Настройка провайдеров

### 1. Получение API ключей

#### Coinbase Commerce
1. Зарегистрируйтесь на https://commerce.coinbase.com
2. Создайте API key в Settings → API keys
3. Скопируйте ключ (начинается с `key_`)

#### NOWPayments
1. Зарегистрируйтесь на https://nowpayments.io
2. Перейдите в Settings → API
3. Скопируйте API key

#### CoinPayments
1. Зарегистрируйтесь на https://www.coinpayments.net
2. Создайте API credentials в Account → API Keys
3. Скопируйте Public Key и Private Key

#### Binance Pay
1. Войдите в Binance
2. Перейдите в Binance Pay → Merchant
3. Создайте API credentials

### 2. Добавление секретов в проект

В настройках проекта добавьте секреты:
```
COINBASE_COMMERCE_API_KEY = "ваш_ключ"
NOWPAYMENTS_API_KEY = "ваш_ключ"
COINPAYMENTS_PUBLIC_KEY = "ваш_публичный_ключ"
COINPAYMENTS_PRIVATE_KEY = "ваш_приватный_ключ"
BINANCE_PAY_API_KEY = "ваш_ключ"
BINANCE_PAY_SECRET = "ваш_секрет"
```

### 3. Включение провайдеров

В админ-панели:
1. Откройте **Управление → Платежи**
2. Переключите нужные провайдеры в статус **Активен**
3. Готово! Система автоматически начнет использовать их

---

## 💻 Использование API

### Создание платежа

```javascript
// Запрос
POST /admin-api
Content-Type: application/json

{
  "resource": "payment",
  "exchange_id": 123,
  "provider_id": 1,
  "amount": "0.5",
  "currency": "BTC"
}

// Ответ
{
  "transaction_id": 456,
  "payment_url": "https://commerce.coinbase.com/charges/ABC123",
  "payment_address": "bc1q...",
  "amount": "0.5",
  "currency": "BTC",
  "provider": "Coinbase Commerce"
}
```

### Получение статуса транзакции

```javascript
// Запрос
GET /admin-api?resource=payment_transaction&id=456

// Ответ
{
  "id": 456,
  "external_id": "ABC123",
  "status": "completed",
  "amount": "0.5",
  "currency": "BTC",
  "confirmations": 3,
  "required_confirmations": 3,
  "payment_url": "https://commerce.coinbase.com/charges/ABC123",
  "payment_address": "bc1q...",
  "provider": "Coinbase Commerce"
}
```

### Webhook обработка

Провайдеры отправляют webhook уведомления:

```javascript
// Webhook от провайдера
POST /admin-api
Content-Type: application/json

{
  "resource": "webhook",
  "provider": "nowpayments",
  "payment_id": "12345",
  "status": "confirmed",
  "confirmations": 15
}
```

Система автоматически:
1. Находит транзакцию по `payment_id`
2. Обновляет статус
3. При `status: confirmed` завершает обмен
4. Отправляет email клиенту

---

## 🔄 Жизненный цикл платежа

### Шаг 1: Создание invoice
```
Клиент создает обмен
↓
Система выбирает активный провайдер
↓
API запрос к провайдеру (создание invoice)
↓
Получение payment_url и адреса
↓
Сохранение в payment_provider_transactions
↓
Показ QR-кода клиенту
```

### Шаг 2: Ожидание оплаты
```
Клиент сканирует QR или копирует адрес
↓
Отправляет криптовалюту
↓
Провайдер получает платеж
↓
Webhook: status = "pending"
```

### Шаг 3: Подтверждение
```
Blockchain confirmations накапливаются
↓
Webhook: confirmations = 3/3
↓
Webhook: status = "confirmed"
↓
Система обновляет exchanges.status = "completed"
```

### Шаг 4: Завершение
```
Email уведомление клиенту
↓
Средства зачисляются на кошелек
↓
Обмен завершен ✅
```

---

## 🛠️ Управление через админ-панель

### Просмотр активных провайдеров
1. **Управление → Платежи**
2. Список всех провайдеров с:
   - Название и тип
   - Поддерживаемые валюты
   - Статус (активен/отключен)

### Включение/отключение провайдера
1. Переключите toggle **Активен/Отключен**
2. Изменения применяются мгновенно
3. Если отключен — не используется для новых платежей

### Мониторинг транзакций
```
Dashboard → Payment Transactions

Фильтры:
- По провайдеру
- По статусу
- По дате
- По сумме

Экспорт в CSV для отчетности
```

---

## 📈 Комиссии и настройки

### Управление комиссиями по парам
1. **Управление → Комиссии**
2. Настройте для каждой пары:
   - Процент комиссии
   - Минимальная сумма
   - Максимальная сумма

### Источники курсов
1. **Управление → Источники курсов**
2. Настройте:
   - Приоритет источников
   - Множитель курса (наценка)
   - Время кэширования

### Тексты на сайте
1. **Управление → Тексты и ссылки**
2. Редактируйте:
   - Заголовки страниц
   - Контакты
   - Ссылки в футере
   - Лимиты обмена

---

## 🚨 Troubleshooting

### Провайдер не создает платеж
**Причины:**
- API ключ неверный или истек
- Провайдер отключен в админ-панели
- Недостаточно баланса у провайдера

**Решение:**
1. Проверьте секреты проекта
2. Убедитесь что провайдер активен
3. Проверьте логи backend функции
4. Попробуйте другой провайдер

### Webhook не приходит
**Причины:**
- Webhook URL не настроен у провайдера
- Firewall блокирует запросы
- Провайдер требует IP whitelist

**Решение:**
1. Настройте webhook URL в панели провайдера
2. Проверьте логи входящих запросов
3. Добавьте IP провайдера в whitelist

### Платеж завис в processing
**Причины:**
- Недостаточно blockchain confirmations
- Провайдер задерживает обработку
- Неправильная сумма отправлена

**Решение:**
1. Проверьте TX hash в blockchain explorer
2. Подождите больше confirmations
3. Свяжитесь с поддержкой провайдера

---

## 🔐 Безопасность

### Хранение API ключей
✅ **Правильно:**
- Хранить в секретах проекта
- Использовать переменные окружения
- Не коммитить в Git

❌ **Неправильно:**
- Хранить в коде
- Хранить в базе данных в открытом виде
- Передавать в frontend

### Проверка webhook
Провайдеры подписывают webhook:
```javascript
// Пример проверки подписи NOWPayments
const crypto = require('crypto');

const signature = request.headers['x-nowpayments-sig'];
const payload = JSON.stringify(request.body);
const secret = process.env.NOWPAYMENTS_IPN_SECRET;

const computedSignature = crypto
  .createHmac('sha512', secret)
  .update(payload)
  .digest('hex');

if (signature !== computedSignature) {
  return { statusCode: 403, body: 'Invalid signature' };
}
```

---

## 📊 Отчетность

### Экспорт транзакций
```sql
SELECT 
  ppt.id,
  pp.name as provider,
  ppt.amount,
  ppt.currency,
  ppt.status,
  ppt.created_at,
  ppt.completed_at
FROM payment_provider_transactions ppt
JOIN payment_providers pp ON ppt.provider_id = pp.id
WHERE ppt.created_at >= '2024-01-01'
ORDER BY ppt.created_at DESC;
```

### Статистика по провайдерам
```sql
SELECT 
  pp.name,
  COUNT(*) as total_transactions,
  COUNT(CASE WHEN ppt.status = 'completed' THEN 1 END) as completed,
  SUM(CASE WHEN ppt.status = 'completed' THEN ppt.amount ELSE 0 END) as volume
FROM payment_provider_transactions ppt
JOIN payment_providers pp ON ppt.provider_id = pp.id
GROUP BY pp.name;
```

---

## 🎓 Лучшие практики

1. **Используйте несколько провайдеров** для отказоустойчивости
2. **Мониторьте uptime** провайдеров
3. **Настройте алерты** на failed платежи
4. **Регулярно проверяйте** API ключи
5. **Ведите логи** всех webhook запросов
6. **Тестируйте** на sandbox окружении перед продакшеном

---

**Готово! Система автоматических платежей настроена и работает 🚀**

***

# Fluent Bit + Graylog + Grafana: Nginx Access Logs Parsing

Этот репозиторий содержит конфигурацию для сбора, парсинга и визуализации логов Nginx (Access и Error) с использованием стека **Fluent Bit**, **Graylog** и **Grafana**.

Основная цель — корректно парсить сложные логи Nginx на стороне агента (Fluent Bit), отправлять их в Graylog в формате GELF без ошибок `missing short_message`, и визуализировать метрики в Grafana.
<img width="1618" height="823" alt="image" src="https://github.com/user-attachments/assets/79292dbb-07bc-47c6-9ffb-7593bd6f1a93" />

## 📂 Структура репозитория

```text
.
├── fluent-bit.conf              # Главный конфигурационный файл Fluent Bit
├── parsers.conf                 # Определение парсеров (Regex для Nginx)
├── 01-global.filters.conf       # Глобальные фильтры (добавление хоста, окружения)
├── 02-graylog.output.conf       # Конфигурация вывода в Graylog (GELF)
├── Grafana_dashboard.json   # Дашборд для Grafana (импортировать вручную)
└── nginx_access.log-parsing-graylog-Visualization-Grafana/
    └── pipeline_rules/          # Правила обработки (Pipeline Rules) для Graylog
```

## ⚙️ Ключевые особенности решения

### 1. Парсинг на стороне клиента (Fluent Bit)
Используется кастомный Regex-парсер `nginx3`, который разбивает строку лога на поля:
- `remote_addr`, `request`, `status`, `body_bytes_sent`
- `http_referer`, `http_user_agent`
- `request_time`, `upstream_response_time`, `upstream_connect_time` (для анализа производительности)

### 2. Решение проблемы `missing short_message key`
Плагин вывода GELF в Fluent Bit требует обязательного поля `short_message`. Так как после парсинга исходное поле `log` исчезает, а поля `request` нет в системных логах, используется **разделение потоков вывода**:

- **Для `nginx.access`**: Используется поле `request` как `short_message`.
- **Для остальных логов (`!nginx.access`)**: Используется поле `message` (или `log`) как `short_message`.

Это предотвращает ошибки дропа событий и дублирование данных.

### 3. Обогащение данных
Глобальные фильтры добавляют мета-данные ко всем событиям:
- `host`: Имя хоста (`${HOSTNAME}`)
- `cluster`: Название кластера (например, `graylog-prod`)
- `environment`: Окружение (например, `linux`)

## 🚀 Установка и настройка

### 1. Настройка Fluent Bit

1. Скопируйте файлы конфигурации в директорию `/etc/fluent-bit/` (или туда, где установлен ваш Fluent Bit).
   ```bash
   cp fluent-bit.conf parsers.conf 01-global.filters.conf 02-graylog.output.conf /etc/fluent-bit/
   ```
   *Примечание: Убедитесь, что пути в `fluent-bit.conf` (`@INCLUDE`) соответствуют вашей структуре папок.*

2. Отредактируйте `02-graylog.output.conf`:
   - Замените `graylog.domain` или `graylog.sberins.ru` на адрес вашего Graylog сервера.
   - Проверьте порт (по умолчанию `12201`).

3. Перезапустите службу:
   ```bash
   systemctl restart fluent-bit
   ```

4. Проверьте логи на наличие ошибок:
   ```bash
   journalctl -u fluent-bit -f
   ```

### 2. Настройка Graylog (Pipeline Rules)

В папке `nginx_access.log-parsing-graylog-Visualization-Grafana/pipeline_rules/` находятся правила для Graylog.

1. Зайдите в **System -> Pipelines**.
2. Создайте новый Pipeline или отредактируйте существующий, подключенный к потоку Nginx-логов.
3. Добавьте правила из файлов `.txt`/`.rule`.
   - Эти правила могут использоваться для дополнительного преобразования типов данных (например, приведение `status` к Integer, `request_time` к Float), если это не сделано полностью на стороне Fluent Bit, или для извлечения дополнительных параметров из URI.

### 3. Визуализация в Grafana

1. Установите плагин **Graylog Data Source** в Grafana.
2. Добавьте источник данных Graylog, указав URL API и credentials.
3. Импортируйте дашборд `Grafana_dashboard.json`:
   - Перейдите в **Dashboards -> Import**.
   - Загрузите JSON-файл.
   - Выберите созданный источник данных Graylog.

## 📊 Какие метрики доступны

Благодаря парсингу полей `request_time`, `upstream_response_time` и `status`, в Grafana можно отслеживать:
- **RPS (Requests Per Second)**: Разбивка по статусам (2xx, 4xx, 5xx).
- **Latency**: Среднее и 95-е перцентили времени ответа (`request_time`).
- **Upstream Performance**: Время ответа бэкенда (`upstream_response_time`).
- **Top URLs**: Самые запрашиваемые ресурсы.
- **Errors**: Детальный просмотр 500-х ошибок с телом запроса.

## 🔧 Troubleshooting

### Ошибка: `[error] [flb_msgpack_to_gelf] missing short_message key`
**Причина:** Поле, указанное в `gelf_short_message_key`, отсутствует в событии.
**Решение:** Убедитесь, что вы используете раздельные `[OUTPUT]` блоки с разными `match` правилами, как показано в `02-graylog.output.conf`. Не используйте `gelf_short_message_key request` для всех логов (`match *`).

### Ошибка: `parser 'nginx3' is not registered`
**Причина:** Файл `parsers.conf` не подключен или содержит синтаксическую ошибку.
**Решение:** Проверьте директиву `parsers_file` в `[SERVICE]` секции `fluent-bit.conf` и запустите `fluent-bit --dry-run` для проверки синтаксиса.

### Логи приходят пустыми или не парсятся
**Причина:** Regex в `parsers.conf` не соответствует формату лога в `access.log`.
**Решение:** Сравните пример строки из `/var/log/nginx/access.log` с Regex в файле `parsers.conf`. Обратите внимание на пробелы и кавычки.

Ниже приведена полная пошаговая инструкция по установке и настройке локального прокси Nginx, который будет автоматически подставлять заголовок `X-Secret-Key` и перенаправлять запросы от Hermes на ваш шлюз.

1. **Установите Nginx:**
В зависимости от вашей операционной системы выполните команду установки:

**Ubuntu / Debian:**

```bash
sudo apt update && sudo apt install -y nginx

```

**CentOS / RHEL / AlmaLinux / Rocky Linux:**

```bash
sudo dnf install -y nginx
sudo systemctl enable --now nginx

```


2. **Создайте конфигурацию локального переходника:** Подставьте ваш реальный домен шлюза вместо server и ваш ключ в X-Secret-Key.
Выполните команду ниже, чтобы создать файл `/etc/nginx/conf.d/tg-gateway.conf` (предварительно заменив `server` на домен/IP вашего шлюза, а `ВАШ_СЕКРЕТНЫЙ_КЛЮЧ` — на ваш ключ):

```bash
sudo tee /etc/nginx/conf.d/tg-gateway.conf > /dev/null << 'EOF'
server {
    listen 127.0.0.1:8088;

    # Увеличенные таймауты и лимит размера файла для long-polling (getUpdates) и отправки медиа
    proxy_read_timeout 120s;
    proxy_connect_timeout 30s;
    proxy_send_timeout 120s;
    client_max_body_size 50M;

    location / {
        proxy_pass https://server/;
        proxy_ssl_server_name on;
        proxy_set_header Host server;
        proxy_set_header X-Secret-Key "ВАШ_СЕКРЕТНЫЙ_КЛЮЧ";
    }
}
EOF

```


3. **Проверьте и перезапустите Nginx:**
Убедитесь, что в синтаксисе конфига нет ошибок, и примените настройки:

```bash
sudo nginx -t && sudo systemctl reload nginx

```

Проверьте, что локальный переходник работает и возвращает ответ от Telegram (подставьте ваш токен бота после `/bot`):

```bash
curl -s http://127.0.0.1:8088/bot<ВАШ_ТОКЕН_БОТА>/getMe

```

В ответ должен прийти JSON вида `{"ok":true,"result":{"id":...}}` — уже без ручной передачи `-H "X-Secret-Key: ..."`.


4. **Очистите .env от настроек вебхука:**
Откройте файл `~/.hermes/.env` и **удалите или закомментируйте** строки `TELEGRAM_WEBHOOK_...`, чтобы Hermes работал в обычном режиме через прокси:

```env
TELEGRAM_BOT_TOKEN="ваш_токен_бота"
#TELEGRAM_WEBHOOK_URL="https://..."
#TELEGRAM_WEBHOOK_SECRET="..."
#TELEGRAM_WEBHOOK_PORT=8443

```

Если ранее вебхук уже успел зарегистрироваться, сбросьте его одной командой через ваш локальный прокси:

```bash
curl -s http://127.0.0.1:8088/bot<ВАШ_ТОКЕН_БОТА>/deleteWebhook

```


5. **Настройте Hermes на локальный прокси и перезапустите:**
В файле `~/.hermes/config.yaml` укажите адрес вашего локального Nginx:

```yaml
platforms:
  telegram:
    extra:
      base_url: "http://127.0.0.1:8088/bot"
      base_file_url: "http://127.0.0.1:8088/file/bot"

```

Перезапустите шлюз Hermes и проверьте логи:

```bash
hermes gateway restart
tail -n 25 ~/.hermes/logs/gateway.log

```

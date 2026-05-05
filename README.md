# 🛡️ Smart Push Monitor (SRE Edition)

Продвинутый Bash-скрипт для мониторинга состояния VPN-узлов (Reality, VLESS, Hysteria 2) с интеграцией в **Uptime Kuma** (Push-тип). Скрипт разработан для системных администраторов, управляющих распределенной сетью Linux VPS.

В отличие от стандартных методов, этот инструмент имитирует поведение реального пользователя, различает типы сетевых сбоев и реализует логику «самолечения» сервисов.

## 🚀 Основные возможности

*   **Гибридная проверка (L3-L7):** Контроль не только наличия процесса, но и реальной проходимости трафика через TCP/UDP и валидности TLS-рукопожатия.
*   **Умное самолечение (Auto-Remediation):** Автоматический перезапуск Docker-контейнеров при достижении заданного порога ошибок.
*   **DPI-Friendly:** Специальный режим для Reality. 90% времени выполняется легкий L4-чек, и лишь раз в несколько циклов — полный TLS-handshake.
*   **Протокольная гибкость:** Полная поддержка TCP (VLESS/Reality) и UDP (Hysteria 2/QUIC).
*   **Ansible-Ready:** Оптимизирован для массовой раскатки через Ansible Semaphore.

## 🛠 Техническая логика

Каждую минуту скрипт выполняет каскад проверок:

1.  **Internet Check:** Проверка связи с миром (пинг 1.1.1.1).
2.  **Bridge Check:** Проверка доступности удаленного моста (связи между РФ и зарубежной нодой).
3.  **Container Check:** Проверка статуса контейнера через `docker inspect`.
4.  **L4/L7 Hybrid:** Проверка порта и (периодически) проверка TLS-рукопожатия.

---

## 📖 Руководство по установке

### Описание файлов проекта
*   `deploy_monitor.yml` — Ansible-плейбук для автоматической установки зависимостей и настройки Cron.
*   `smart_push.sh.j2` — Jinja2-шаблон скрипта. В нем используются переменные вида `{{ variable }}`, которые заполняются автоматически при деплое.

---

### Вариант 1: Через Ansible / Semaphore (Рекомендуется)

1.  **Размещение:** Положите файл `smart_push.sh.j2` в ту же папку, где лежит плейбук `deploy_monitor.yml`.
2.  **Инвентарь:** Добавьте переменные для ваших хостов в YAML-инвентарь:

```yaml
your-node-name.infra:
  ansible_host: 1.2.3.4
  kuma_push_url: "[https://kuma.example.com/api/push/YOUR_TOKEN?v=1](https://kuma.example.com/api/push/YOUR_TOKEN?v=1)"
  vpn_port: 443
  vpn_proto: "tcp"           # tcp для Reality, udp для Hysteria
  sni: "your-sni-domain.com"
  bridge_ip: "5.6.7.8"       # Опционально: IP моста
  bridge_port: 443
  bridge_port_proto: "udp"   # Протокол моста
```
3. **Запуск:** Запустите плейбук.
  Ansible автоматически подставит данные в шаблон и создаст готовый скрипт /opt/smart_push.sh

### Вариант 2: Ручная установка (Manual Setup)

Если вы не используете Ansible, вам нужно превратить шаблон .j2 в обычный скрипт .sh, заменив переменные вручную.

Пример настройки файла (как должен выглядеть блок настроек в /opt/smart_push.sh):

# Было в smart_push.sh.j2:
```bash
KUMA_PUSH_URL="{{ kuma_push_url }}"
LOCAL_VPN_PORT="{{ vpn_port }}"
LOCAL_VPN_PROTO="{{ vpn_proto | default('tcp') }}"
```
# Должно стать в /opt/smart_push.sh:
```bash
KUMA_PUSH_URL="https://status.domain.com/api/push/nS5QnZK3en?v=1"
LOCAL_VPN_PORT="443"
LOCAL_VPN_PROTO="tcp"
```
Порядок действий:

1. Установите зависимости и скачайте скрипт:
```bash
apt update && apt install -y jq netcat-openbsd curl docker.io
curl -L -o /opt/smart_push.sh [https://raw.githubusercontent.com/kabebon/smart-push-monitor/main/UptimeKuma/smart_push.sh.j2](https://raw.githubusercontent.com/kabebon/smart-push-monitor/main/UptimeKuma/smart_push.sh.j2)
```
2. Откройте файл для настройки:
```bash
nano /opt/smart_push.sh
```

3. В самом начале файла найдите блок переменных. Замените фигурные скобки {{ ... }} на ваши реальные данные.
Пример настройки:
```bash
# Было:
KUMA_PUSH_URL="{{ kuma_push_url }}"
LOCAL_VPN_PORT="{{ vpn_port }}"
LOCAL_VPN_PROTO="{{ vpn_proto | default('tcp') }}"

# Должно стать:
KUMA_PUSH_URL="https://status.domain.com/api/push/nS5QnZK3en?v=1"
LOCAL_VPN_PORT="443"
LOCAL_VPN_PROTO="tcp"
```   

4. Дайте права и добавьте в Cron:

```bash
chmod +x /opt/smart_push.sh
(crontab -l 2>/dev/null; echo "* * * * * /opt/smart_push.sh") | crontab -
```

## 📊 Интерпретация статусов

| Сообщение | Описание проблемы | Действие скрипта |
| :--- | :--- | :--- |
| **OK** | Все системы работают штатно | Пинг (Up) |
| **No_internet** | Нет связи с миром на сервере | Только лог |
| **Bridge_down** | Недоступен зарубежный узел | Только лог |
| **Container_dead** | Контейнер упал или остановлен | Рестарт (после 3 попыток) |
| **Port_unreachable** | Сервис не слушает порт | Рестарт (после 3 попыток) |
| **TLS_failed** | Ошибка рукопожатия (DPI/SNI) | Рестарт (после 3 попыток) |

## ⚠️ Важные примечания
*   **Cooldown:** После автоматического рестарта скрипт выжидает 5 минут, прежде чем снова пытаться «лечить» сервис.
*   **Locks:** Используется `flock`, чтобы несколько копий скрипта не запускались одновременно при зависании сети.


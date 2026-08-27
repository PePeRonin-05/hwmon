# hwmon — Серверный hardware-мониторинг (Prometheus + Grafana)

Помогает: проверка серверного железа — **CPU, RAM, диск, сеть, BMC/IPMI (вентиляторы, температуры, питание), S.M.A.R.T-здоровье накопителей, GPU NVIDIA (опционально)**. Разворачивается на любом сервере, где есть Docker.

Цель — «запустил → всё поднялось и настроилось → смотришь дашборды». Мониторинг **local-only**: работает на том сервере, где запущено (все компоненты на `localhost`). Удобно проверять по одному серверу за раз на любом железе.

---

## Состав стека

| Компонент | Порт | Что делает |
|-----------|------|------------|
| **prometheus** | 9090 | сбор и хранение метрик (retention 30 дней) |
| **grafana** | 3000 | визуализация, дашборды (логин `admin` / `admin`) |
| **node-exporter** | 9100 | CPU, RAM, диск, сеть, процессы |
| **ipmi-exporter** | 9290 | температуры, вентиляторы, питание, напряжения (BMC/IPMI) |
| **smartctl-exporter** | 9633 | S.M.A.R.T-здоровье накопителей |
| **dcgm-exporter** | 9400 | NVIDIA GPU (опциональный, `--profile gpu`) |

Все сервисы используют `network_mode: host` — экспортеры видны Prometheus по `localhost`.

---

## Быстрый старт (за 3 команды)

```bash
git clone https://b.yadro.com/scm/~ru.ibragimov/hwmon.git   # или по SSH (см. ниже)
cd hwmon
docker compose up -d
```

После этого в Grafana (`http://<IP_сервера>:3000`, логин `admin`/`admin`) уже есть готовые дашборды — ничего настраивать вручную не нужно.

---

## Как скачать

```bash
# HTTPS
git clone https://b.yadro.com/scm/~ru.ibragimov/hwmon.git
cd hwmon
```

или по SSH:
```bash
git clone ssh://git@b.yadro.com:7999/~ru.ibragimov/hwmon.git
cd hwmon
```

---

## Как запустить с нуля (по шагам)

### 1. Проверь, что установлены Docker и Docker Compose

```bash
docker --version           # Docker Engine
docker compose version     # Docker Compose plugin
```

> Нужны **обе** команды без ошибки.

### 2. Если Docker не установлен

**Способ А — официальный скрипт (ставит Docker Engine + Compose plugin сразу):**
```bash
curl -fsSL https://get.docker.com | sh
```

**Способ Б — пакетами apt:**
```bash
apt-get update && apt-get install -y docker.io
```

> ⚠️ В пакете `docker.io` из репозиториев Ubuntu **нет команды `docker compose`** (только движок). Для Compose установи плагин из официального Docker-репозитория:
> ```bash
> install -m 0755 -d /etc/apt/keyrings
> curl -fsSL https://download.docker.com/linux/ubuntu/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg
> echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo $VERSION_CODENAME) stable" > /etc/apt/sources.list.d/docker.list
> apt-get update && apt-get install -y docker-compose-plugin
> ```

После установки — включи и проверь:
```bash
systemctl enable --now docker
docker --version && docker compose version
```

### 3. Запусти стек

```bash
docker compose up -d
```

Стек поднимется сам: подтянутся образы, создастся Prometheus-датасорс в Grafana и импортируются готовые дашборды. **Ручная настройка не нужна.**

### Что дальше

- **Grafana**: `http://<IP_сервера>:3000` — логин `admin` / пароль `admin`
- **Prometheus**: `http://<IP_сервера>:9090`

---

## Основные дашборды

Все дашборды **автоадаптивны** — на любом железе сенсоры снимаются и отображаются автоматически (по лейблам и regex-группам, без захардкоженных имён).

### IPMI Full Hardware Sensors
Глубокий обзор всего, что отдаёт BMC/IPMI. Автоматически находит и показывает:
- **Температуры** — CPU/DTS, DIMM, VR (модули питания), PSU, чипсет — все датчики по группам.
- **Вентиляторы** — обороты (RPM) каждого вентилятора.
- **Мощность** — суммарное потребление сервера + по каналам (VR/PSU), график «Суммарная мощность» наверху — по нему видно пики/отвалы питания.
- **Напряжения и ток** — все линии питания (P12V, VR, PCH и т.д.).
- Внизу — свёрнутые таблицы температур/вентиляторов/напряжений.

Для анализа: провал мощности до нуля на «Суммарная мощность» = отвал PSU/сброс; рост температуры/вентиляторов = троттлинг под нагрузкой.

### Load Test Overview (system)
Максимально простой и понятный дашборд «как железо держит нагрузку». Только базовые метрики:
- **Обзор** — время работы, число ядер CPU, объём оперативной памяти.
- **CPU** — загрузка (%) и Load Average (1/5/15 мин).
- **RAM** — занятая память против всего объёма.
- **Disk** — скорость чтения/записи дисков.
- **Network** — скорость приёма/отдачи по каждому интерфейсу.

Идеален для нагрузочных тестов / стресс-тестов: во время теста видно, насколько загружен CPU, хватает ли памяти, как ведут себя диски и сеть. У каждой панели короткое пояснение на русском.

### Node Exporter Full
Расширенный системный дашборд на метриках node-exporter (есть на любом сервере). Секции:
- **Quick CPU / Mem / Disk** — CPU Busy, Load, RAM/SWAP, Root FS, ядра, аптайм.
- **Basic CPU / Mem / Net / Disk** — общие графики.
- **CPU / Memory / Net / Disk** — детальные графики по всем режимам CPU, память (meminfo/vmstat), сеть, диски.
- **System** — timesync, процессы, misc.
- **Storage** — диски и файловые системы.
- **Network** — трафик, sockstat, netstat.
- Поддерживает переключение `$node` / `$job` (полезно если экспортеров несколько).

### SMART Full Drive Health
Здоровье дисков по S.M.A.R.T (использует `smartctl-device` метку, работающую с любыми накопителями SSD/HDD):
- **Статус** — сводные статы: SMART Status (1=OK), температура за период, наработка (часы), ошибки лога.
- **Температура дисков** — график по времени.
- **Износ / перезапись** — SSD wear indicator и общее здоровье (attribute-атрибуты).
- Свёрнутые таблицы S.M.A.R.T-атрибутов по каждому диску.

### Дополнительно
- **GPU Full (NVIDIA DCGM)** — дашборд GPU (использование, память, температура). Работает только когда на хосте поднят драйвер NVIDIA и стек запущен с профилем `gpu` (`docker compose --profile gpu up -d`).
- **Visual Playground (тест фишек)** — тестовый дашборд с экспериментальными визуализациями (gauge, bar gauge, heatmap, state timeline и т.д.), не обязателен для работы.

---

## Проверка, что всё работает

```bash
docker compose ps          # все сервисы должны быть Up
curl http://localhost:9090/api/v1/targets   # targets в UP
```

---

## Возможные проблемы при запуске

### 1. Ошибка `failed to mount ... fstype: overlay ... invalid argument`
Случается, когда корневая файловая система сервера **сама смонтирована как overlay** (сервер = образ/шаблон/контейнер). Docker не может вложить ещё один overlay-слой.

**Решение** — переключить storage-driver Docker на `fuse-overlayfs`:
```bash
apt-get install -y fuse-overlayfs
cat > /etc/docker/daemon.json <<'EOF'
{
  "storage-driver": "fuse-overlayfs"
}
EOF
systemctl restart docker
docker compose up -d
```
Проверить, что root на overlay: `df -T /` → строка `overlay`.

### 2. SMART-дашборд пустой (0 дисков)
Если на сервере **нет локальных накопителей** (диски в удалённом хранилище/NVMe-oF), то `smartctl-exporter` честно показывает «devices found = 0» — это нормально. Проверка:
```bash
lsblk -d          # физические локальные диски
curl -s localhost:9633/metrics | grep smartctl_devices   # число дисков
```
Есть локальные диски, но метрик нет — вероятно, не хватает доступа к `/dev/sd*` (контейнер должен работать как root с `privileged: true`, уже настроено в compose).

### 3. `dcgm-exporter` в статусе DOWN (GPU)
Ожидаемо, пока на хосте нет драйвера NVIDIA. Сервис не ломает стек — просто выбивает `down`. Когда драйвер будет, включи профиль GPU:
```bash
docker compose --profile gpu up -d
```

### 4. `docker compose: command not found`
Установлен `docker.io` без Compose-плагина. Поставь плагин (см. раздел «Как запустить с нуля», Способ Б).

---

## Настройка под своё железо

### NVIDIA GPU (опционально)
Дашборд для GPU есть в комплекте, но **dcgm-exporter выключен по умолчанию** (профиль `gpu`), т.к. требует установленного драйвера NVIDIA на хосте. Когда драйвер готов:
```bash
docker compose --profile gpu up -d
```

### Требования к привилегиям
Экспортеры железа (`ipmi-exporter`, `smartctl-exporter`) запускаются как `root` и с `privileged: true`, т.к. им нужен доступ к `/dev/ipmi0` (BMC) и `/dev/sd*` (диски). Без этого — `Permission denied`.

---

## Файлы проекта

```
.
├── docker-compose.yml                     # описание стека
├── deploy.sh                              # скрипт деплоя на удалённые серверы
├── prometheus/
│   ├── prometheus.yml                     # конфиг сбора метрик
│   └── rules/                             # папка для опциональных alert-правил
└── grafana/
    ├── provisioning/
    │   ├── datasources/prometheus.yml     # авторегистрация datasource
    │   └── dashboards/dashboards.yaml     # автоимпорт дашбордов
    └── dashboards/                        # готовые JSON-дашборды
        ├── ipmi-full.json                 # IPMI Full Hardware Sensors
        ├── load-test-overview.json        # Load Test Overview (system)
        ├── node-exporter-full.json        # Node Exporter Full
        ├── smart-full.json                # SMART Full Drive Health
        ├── dcgm.json                      # GPU Full (NVIDIA DCGM)
        └── visual-playground.json         # тестовый дашборд визуализаций
```

---

## Советы по анализу данных в Grafana

- Выбирай временной диапазон **6h / 24h / 7d** в правом верхнем углу, чтобы видеть динамику за рабочий период и отслеживать падения/пики.
- В легенде графиков включены **min / max / mean** — сразу видно экстремумы за период.
- Панели имеют подсказки (иконка `i` в углу) — как интерпретировать график.
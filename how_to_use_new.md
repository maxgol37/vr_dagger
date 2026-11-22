# UR3: руководство оператора — VR, демонстрации, политика и DAgger

Актуально для общего workflow, 18 ноября 2026. Все команды выполняются **на хосте**,
если явно не написано «внутри контейнера». Каждый блок начинается с нужного `cd`.
Запускающий сотрудник должен быть обучен работе с UR3 и штатной остановкой робота.

## Содержание

1. [Сценарии, файлы и кнопки](#1-сценарии-файлы-и-кнопки)
2. [Подготовка и первоначальная установка](#2-подготовка-и-первоначальная-установка)
3. [Ежедневный запуск backend и камер](#3-ежедневный-запуск-backend-и-камер)
4. [Только телеоперация](#4-только-телеоперация)
5. [Обычная запись демонстраций](#5-обычная-запись-демонстраций)
6. [Выбор модели, параметров и гистерезиса](#6-выбор-модели-параметров-и-гистерезиса)
7. [Запуск политики и DAgger](#7-запуск-политики-и-dagger)
8. [Единая конвертация](#8-единая-конвертация)
9. [Объединение и обучение](#9-объединение-и-обучение)
10. [Логи, trace и ROS](#10-логи-trace-и-ros)
11. [Troubleshooting](#11-troubleshooting)
12. [Завершение работы](#12-завершение-работы)

## 1. Сценарии, файлы и кнопки

Два соседних репозитория:

- `~/Kosenkov/ur3-vr-teleoperation` — Unity/Quest, UR3, Robotiq, ROS backend,
  ControlMux, MoveIt Servo и RTDE. Настройки backend: `.env`.
- `~/Kosenkov/vr-dagger` — камеры, запись, датасеты, обучение и модель.
  Настройки инференса общего запуска: `config/workflow.env`.

**Общий вход** из vr-dagger:

```bash
cd ~/Kosenkov/vr-dagger
bash scripts/workflow.sh help
```

Тот же вход из teleop; аргументы после `workflow` полностью совпадают:

```bash
cd ~/Kosenkov/ur3-vr-teleoperation
./scripts/run.sh workflow help
```

Если репозитории лежат не рядом, задать `VR_DAGGER_ROOT` для teleop-оболочки и
`TELEOP_ROOT` для vr-dagger-оболочки. Это абсолютные пути к соответствующим репозиториям.
Существующие `run.sh`, `capture.sh`, `rollout.sh`, `export_dagger.py` и
`convert_all_demos.py` остаются доступны. Для новых сотрудников использовать workflow.

| Сценарий | Управление | Включение записи | Сырой результат |
|---|---|---|---|
| Телеоперация | `workflow.sh vr` | Не требуется | Нет записи |
| Обычная демонстрация | VR | `record demonstration` | `data/bags_clean_table/demoN` |
| Проверка политики | `policy`, затем `p` | Trace при необходимости | `data/traces/policy_...` |
| Политика с исправлениями | `dagger`, клавиши `p/v` | `record corrections` | `data/bags_dagger/demoN` |

`policy` и `dagger` — два имени одного keyboard runner: веса загружаются один раз,
можно явно переключать источники, а для записи выбрать нужный рекордер отдельно.
Название команды само не запускает запись. До `p/v` runner ждёт выбора.
Модель работает на хосте, bridge в Docker связывает её с ROS через
`data/ipc/ur3_dagger.sock`. Установка `rclpy` в Python-окружение модели не нужна.

| Управление | Назначение |
|---|---|
| Правый Trigger | Удерживать для следования руки робота за VR-рукой; отпустить для паузы/перепривязки |
| Правый Grip | Степень закрытия захвата |
| Правая A | Переключить фиксацию положения гриппера и следование Grip |
| Правая B | Начало/конец bag, только когда включён B-рекордер |
| `p` в терминале runner | Явно выбрать POLICY |
| `v` | Явно выбрать VR; отпустить и снова нажать Trigger для движения |
| `s` | STOP; подтверждение ошибки runner после устранения её причины |
| `q` | Завершить runner, условно остановив принадлежащую ему эпоху |

A **не** является безусловной командой открытия: при снятии фиксации и отпущенном
Grip получается открытие. B не останавливает движение. Отпускание Trigger не
завершает bag и не возвращает POLICY. После выбора VR всегда отпустить и снова
нажать Trigger. Keyboard focus должен быть в терминале runner; Enter не нужен.
Клавиша, пришедшая во время синхронного инференса, обрабатывается после расчёта;
это программное управление с задержкой, штатная аппаратная остановка остаётся доступна.

## 2. Подготовка и первоначальная установка

Linux, Docker Engine + Compose v2, доступ пользователя к Docker; NVIDIA/CUDA для
обучения и инференса, `uv` на хосте. Для сборки Quest — версия Unity из README.

Только при первоначальной установке, если `.env` ещё отсутствует:

```bash
cd ~/Kosenkov/ur3-vr-teleoperation
cp .env.example .env
nano .env
```

Проверить `ROBOT_IP`, ROS domain, TCP/payload, ограничения и согласованную
конфигурацию реальной установки по [docs/rtde.md](docs/rtde.md).
Существующий `.env` не перезаписывать примером.

```bash
cd ~/Kosenkov/vr-dagger
uv sync --frozen

cd ~/Kosenkov/ur3-vr-teleoperation
./scripts/run.sh build

cd ~/Kosenkov/vr-dagger
bash scripts/capture.sh build
```

После изменения ROS backend пересобрать его и перезапустить **между попытками**.
Capture scripts монтируются в контейнер: для обновлённого bridge достаточно
завершить runner и остановить/запустить соответствующий сервис. Зависимости
capture требуют пересборки образа. Для этой версии workflow новый APK не нужен,
если в установленном приложении уже работают ControllerPosePublisher, Grip/A и B.

После обновления этой версии, когда запись завершена и runner закрыт, остановить
старый bridge; следующий policy/dagger запустит процесс с обновлёнными scripts:

```bash
cd ~/Kosenkov/vr-dagger
bash scripts/workflow.sh capture dagger-stop
```

Для обновления backend (включая новые поля control status) — STOP, завершение
записи/runner, `run.sh build` и обычный запуск `run.sh rtde` с проверенными
параметрами. Сам workflow работающие hardware-контейнеры не пересобирает.

## 3. Ежедневный запуск backend и камер

Проверить сцену, камеры, TCP, исходное положение и свободное место:

```bash
cd ~/Kosenkov/vr-dagger
df -h .
```

Два raw RGB-потока занимают примерно 3–3.3 ГБ/минуту. Запись/конвертация
сохраняют резерв 2 ГиБ. Один checkpoint текущей архитектуры — около 1.1 ГиБ;
10 сохранений требуют примерно 11 ГиБ плюс датасеты и запас. Минимальная проверка
training script не резервирует место для всей серии checkpoints: оператор должен
проверить место заранее. Автоматического удаления старых данных нет.

Запуск реального backend из STOP после подготовки установки:

```bash
cd ~/Kosenkov/ur3-vr-teleoperation
TELEOP_CONTROL_MODE=STOP \
RTDE_ENABLE_MOTION=true \
RTDE_ENABLE_GRIPPER=true \
RTDE_ENABLE_GRIPPER_FEEDBACK=true \
./scripts/run.sh rtde
./scripts/run.sh ps
```

Просмотр backend:

```bash
./scripts/run.sh logs
```

Для проверки связи без исполнения вместо реального запуска использовать
`RTDE_ENABLE_MOTION=false RTDE_ENABLE_GRIPPER=false`, оставив feedback=true.
Такой PREVIEW не является записью выполненной демонстрации.
Не менять эти флаги во время активного управления. Запуск читает `.env` через
Compose; `source .env` не нужен. Переменные перед командой перекрывают `.env`.

Запустить приложение на Quest: IP компьютера backend, порт 10000, ROS 2.

Камеры — отдельный терминал:

```bash
cd ~/Kosenkov/vr-dagger
bash scripts/workflow.sh cameras
bash scripts/workflow.sh check
```

`check` проверяет **живые сообщения**, а не только наличие топиков. Ожидаются две
камеры примерно 30 Гц, суставы/TCP и статусы робота, цель гриппера. Дождаться PASS.
Если цель гриппера UNKNOWN, выбрать VR и изменить Grip при отпущенном Trigger,
затем повторить check. Для state[16] нужна действительная измеренная обратная
связь Robotiq, а не только команда.

## 4. Только телеоперация

```bash
cd ~/Kosenkov/vr-dagger
bash scripts/workflow.sh vr
```

Отпустить Trigger и нажать заново. Для VR без записи модель, runner и камеры не
требуются. При уже работающем backend запись можно включить позже.

Остановить управление:

```bash
bash scripts/workflow.sh backend mode STOP
```

STOP удерживает команду гриппера, не означает раскрытие. Не запускать второй
runner параллельно первому. Перед переходом из работающего runner сначала `s`, `q`.

## 5. Обычная запись демонстраций

Backend в VR, камеры и feedback готовы. Задачу серии задать один раз:

```bash
cd ~/Kosenkov/vr-dagger
nano config/recording.json
```

Например `task`: `pick blue ball and place on the tube`. Для одной задачи использовать
одинаковый текст. После изменения task перезапустить B-рекордер между эпизодами.

Если работал DAgger-рекордер, сначала завершить его активный bag через B и дождаться
IDLE, затем остановить его:

```bash
bash scripts/workflow.sh record corrections stop
bash scripts/workflow.sh record demonstration start
bash scripts/workflow.sh record demonstration logs
```

Дождаться READY → B → RECORDING → выполнить задачу → B → IDLE. Вернуть сцену
в исходное состояние после сохранения и повторить. Номера demoN выбираются
автоматически и не перезаписываются. Обычные допустимые паузы и действия гриппером
при отпущенном Trigger входят в эпизод. Не переключать POLICY внутри обычной записи:
единый конвертер откажет в использовании такой записи как демонстрации.

```text
data/bags_clean_table/demoN/episode.json
data/bags_clean_table/demoN/bag/metadata.yaml
data/bags_clean_table/demoN/bag/*.db3
```

В metadata новых записей: `collection_mode=demonstration`. Ручная запись одного
эпизода вместо B, при остановленном B-рекордере:

```bash
bash scripts/workflow.sh record demonstration stop
bash scripts/capture.sh record \
  --name manual001 --task 'pick blue ball and place on the tube'
```

Ctrl+C завершает ручной bag; дождаться финализации. Для manual-имён использовать
одиночную конвертацию: batch обрабатывает только директории demoN.

## 6. Выбор модели, параметров и гистерезиса

Обязательно выбрать checkpoint **в том же терминале**, где будет запущен runner:

```bash
cd ~/Kosenkov/vr-dagger
export POLICY_PATH="$PWD/outputs/train/diffusion_ur3_clean_table_dagger_v1_run1/checkpoints/010000/pretrained_model"
test -f "$POLICY_PATH/config.json" && echo 'Checkpoint найден'
```

Это пример существующей модели установки; для другой модели заменить путь.
При новом терминале переменную задать снова. Фиксированный номер шага удобнее
`last` для воспроизводимых сравнений. Выбор пути сам по себе не включает движение.

Общие defaults нового workflow:

```bash
nano config/workflow.env
```

```ini
DIFFUSION_SCHEDULER=DDIM
DIFFUSION_STEPS=30
POLICY_GRIPPER_MODE=hysteresis
GRIPPER_OPEN_THRESHOLD=0.3
GRIPPER_CLOSE_THRESHOLD=0.5
```

Файл — буквальные значения без кавычек, `export`, `$PWD` или shell-команд.
Приоритет: **CLI runner > переменные терминала > workflow.env**.
Изменения применяются при следующем запуске workflow, не к уже работающему процессу.
Старые `rollout.sh`/`dagger_rollout.sh` напрямую этот файл не читают и сохраняют
свои прежние defaults (continuous); для сотрудников использовать workflow.

С гистерезисом, явно переопределив пороги:

```bash
bash scripts/workflow.sh policy --execute \
  --gripper-mode hysteresis \
  --gripper-open-threshold 0.3 --gripper-close-threshold 0.5
```

Без фильтра:

```bash
bash scripts/workflow.sh policy --execute --gripper-mode continuous
```

Для DAgger заменить `policy` на `dagger`; флаги **одинаковые**. Без `--execute`
все клавиши работают только для preview и не переключают физический источник.
В обычном legacy `rollout.sh` пороги передаются как
`--robot.gripper_open_threshold=...`/`--robot.gripper_close_threshold=...`;
не смешивать эти имена с CLI общего runner.

`g≤open` → открыть; `g≥close` → закрыть; между порогами сохранить последнюю команду.
Обязательно `0≤open<close≤1`. В continuous передаётся непрерывная команда модели.
Пороги 0.3/0.5 — явный пример, не универсальная калибровка и не исправление модели.
Фильтр хранит последнюю **запрошенную** команду, а не измеренное положение пальцев:
контакт может остановить их раньше закрытой цели.

Пример изменения sampler и длительности:

```bash
DIFFUSION_SCHEDULER=DDIM DIFFUSION_STEPS=20 \
  bash scripts/workflow.sh policy --execute --duration 90
```

Или CLI: `--scheduler DDIM --steps 30 --duration 90`. У runner duration=0 означает
работу до q. Смена sampler/steps сама по себе переобучения не требует. Медленный
DDPM может превысить существующий watchdog; не увеличивать timeout для обхода
отказа. Модель считает блоки действий, синхронный runner отправляет их с целевыми
30 Гц; вычисление нового блока создаёт паузу, фактическая частота может быть ниже.

Параметры backend (скорости, workspace, ограничения, IP, RTDE) находятся в
`ur3-vr-teleoperation/.env`, описаны в `.env.example` и [docs/rtde.md](docs/rtde.md).
Они не заменяют параметры модели. После изменения `.env` остановить управление,
завершить запись/runner и пересоздать backend обычным `run.sh rtde`; кодовые
изменения backend дополнительно требуют build. Активные лимиты смотреть в
`/teleop/control_status`/`/policy/prediction`, а не предполагать по примеру файла.

## 7. Запуск политики и DAgger

### Только проверка модели

Камеры/backend готовы, выбран POLICY_PATH. Запуск без исполнения:

```bash
cd ~/Kosenkov/vr-dagger
bash scripts/workflow.sh policy
```

Runner прогревает модель на наблюдениях, затем `p` включает preview.
Без `--execute` никакая клавиша не выбирает физический режим. После проверки q.

Реальное исполнение:

```bash
bash scripts/workflow.sh policy --execute
```

Дождаться загрузки, нажать p. Общая оболочка запускает DAgger bridge, но режим
POLICY выбирается только явным p. Отдельный `run.sh mode POLICY` здесь не нужен.
Для остановки s, затем q. Для ручного вмешательства доступно v; чтобы получить
обучающие исправления, нужно дополнительно включить corrections-рекордер.

### Сбор исправлений

Завершить прежнюю запись/runner, выбрать STOP. Остановить обычный B-рекордер и
legacy policy bridge, если использовались:

```bash
cd ~/Kosenkov/vr-dagger
bash scripts/workflow.sh record demonstration stop
bash scripts/workflow.sh capture policy-stop
```

Терминал A — выбрать POLICY_PATH по разделу 6 и запустить:

```bash
bash scripts/workflow.sh dagger --execute
```

Терминал B — после подключения runner:

```bash
cd ~/Kosenkov/vr-dagger
bash scripts/workflow.sh record corrections start
bash scripts/workflow.sh record corrections logs
```

Дождаться READY. Далее B → RECORDING → p → политика. При необходимости v →
отпустить Trigger → снова нажать для движения → исправить. Можно отпустить
Trigger и открывать/закрывать гриппер на неподвижной руке: **подтверждённая VR-пауза
теперь включается в исправление**. Для продолжения модели явно p.
В конце s → B → IDLE. q завершает runner. Смена p/v не заканчивает bag.

```text
data/bags_dagger/demoN/episode.json    collection_mode=dagger
data/bags_dagger/demoN/bag/...
```

Новый `/dagger/status` schema=2 различает `expert_motion` и `expert_pause`;
`expert` — их объединение. Для pause нужны подтверждённый human/VR epoch,
свежее отпускание Trigger после подтверждения VR, неактивные mux/Servo,
RTDE IDLE/READY_RELEASED и свежие исходные наблюдения/цель гриппера.
Для движения по-прежнему нужна свежая привязка и TRACKING.
Смена epoch, POLICY, STOP, fault или пропавшие данные не становятся паузой.
Содержательное качество демонстрации оценивает оператор: программа не определяет,
что задача успешно выполнена или что ручной возврат в старт полезен для обучения.

## 8. Единая конвертация

### Правила отбора

```bash
cd ~/Kosenkov/vr-dagger
bash scripts/workflow.sh convert --help
```

| `--selection` | Что делает |
|---|---|
| `auto` | По metadata/записанным источникам выбирает demonstration или corrections |
| `demonstration` | Полная обычная ручная демонстрация; POLICY/mixed/UNKNOWN отвергаются |
| `corrections` | Только подтверждённые человеческие окна из marked DAgger bag |

Метки POLICY не являются экспертными обучающими действиями. Отсутствие меток
не означает POLICY или VR. Старые bags с подтверждённым VR определяются auto;
для действительно старой записи без источников допускается явное
`--selection demonstration` **после независимой проверки, что это ручная запись**.
Это не позволяет обойти явно записанный POLICY или mixed DAgger.

По умолчанию corrections включает подтверждённые VR-паузы. Флаг
`--pause-policy exclude` возвращает прежний отбор только активного движения.
Он относится к corrections; обычные демонстрации сохраняют допустимые паузы всегда.
Для старых schema=1 mixed bags пауза восстанавливается только при свежем raw
Trigger=False после подтверждения текущего human-перехвата и согласованных
остальных статусах. Поэтому старые исходные bags можно переэкспортировать в
**новую** папку без перезаписи предыдущего датасета.

### Одиночная запись

Подставить существующий номер вместо demo13:

```bash
bash scripts/workflow.sh convert inspect data/bags_dagger/demo13 --selection auto

bash scripts/workflow.sh convert convert data/bags_dagger/demo13 \
  --root data/lerobot_dagger_with_pauses/demo13 \
  --selection corrections --pause-policy include
```

Обычная демонстрация:

```bash
bash scripts/workflow.sh convert inspect data/bags_clean_table/demo1 \
  --selection demonstration

bash scripts/workflow.sh convert convert data/bags_clean_table/demo1 \
  --root data/lerobot_clean_table_v2/demo1 --selection demonstration
```

`inspect` не создаёт dataset. Выход одиночной конвертации должен отсутствовать.
Формат одинаков: state[16], action[10], две RGB-камеры, measured gripper в state,
requested gripper в action. TCP action = измеренная TCP-поза следующего кадра
при 30 Гц (`actual_next`, offset=1); gripper action — удерживаемая целевая команда
в текущий момент. Последняя TCP-метка удерживается в конце **того же** окна.
Не используется первый кадр POLICY или следующего исправления.

### Вся серия

```bash
bash scripts/workflow.sh convert batch data/bags_clean_table \
  --root data/lerobot_clean_table_v2 --selection auto

bash scripts/workflow.sh convert batch data/bags_dagger \
  --root data/lerobot_dagger_with_pauses --selection auto --pause-policy include
```

Batch обрабатывает demoN, пишет `_batch_report.json` и `_conversion_logs/demoN.log`.
Совместимые завершённые результаты нового конвертера пропускаются. Неполные
выходы и результаты другой версии/отбора не перезаписываются — нужна новая папка.
Ошибка одного bag не прерывает остальные; общий exit code ненулевой при ошибках.
Bag без пригодных ручных окон отмечается failed с объяснением, не превращается
в POLICY-демонстрацию. Читать отчёт и выбирать успешные проверенные выходы явно.
Техническая целостность не доказывает успех задачи.

Corrections создаёт вложенные окна:

```text
data/lerobot_dagger_with_pauses/demo13/dagger_export.json
data/lerobot_dagger_with_pauses/demo13/demo1/
data/lerobot_dagger_with_pauses/demo13/demo2/
```

`dagger_export.json`: complete=true, windows, short_windows, причины исключения,
selector_version=2, pause_policy. Минимум окна по умолчанию 16 кадров (~0.53 с).
`--min-frames` меняет его явно (минимум 2). `--max-age` по умолчанию 0.15 с,
увеличивать выше 0.15 в новом конвертере нельзя. `--images` сохраняет изображения
вместо видео, главным образом для диагностики. Отсутствующий measured gripper
не подменяется командой. Дополнительные POS/PRE/OBJ/STA — `gripper_feedback.jsonl`.

Видео обычного экспортированного эпизода можно посмотреть через ffplay, например:

```bash
ffplay data/lerobot_clean_table_v2/demo1/videos/observation.images.wrist/chunk-000/file-000.mp4
```

Проверить фактический путь MP4 в `videos`, если структура отличается. Q закрывает просмотр.

## 9. Объединение и обучение

### Обычные демонстрации

Выбрать качественные эпизоды и явный holdout **целыми эпизодами**. Пример номеров
из нашей clean-table серии, для другой серии заменить:

```bash
cd ~/Kosenkov/vr-dagger
bash scripts/workflow.sh merge demonstration \
  --source data/lerobot_clean_table_v2 \
  --root data/lerobot/ur3_clean_table_v2 \
  --validation-demos 6 12 20 28 35
```

`--exclude 3 8` исключает указанные demo; `--max-demo N` ограничивает номер.
Нельзя одновременно исключить и выбрать эпизод validation. Все номера должны
существовать и пройти проверку. Явный tail holdout поддерживается для одной задачи.

### Добавление исправлений

```bash
bash scripts/workflow.sh merge corrections \
  --demonstrations data/lerobot/ur3_clean_table_v1 \
  --corrections data/lerobot_dagger_with_pauses/demo1 data/lerobot_dagger_with_pauses/demo2 \
  --root data/lerobot/ur3_clean_table_dagger_pauses_v1
```

Список corrections задаётся явно после просмотра отчётов и самих действий.
Автоматически добавлять всю папку, включая неудачные/контрольные попытки, не нужно.
База должна иметь явный holdout. При следующей итерации можно взять предыдущий
агрегированный набор как `--demonstrations` и добавить **новые** исправления;
исходные validation остаются отдельно. Несколько попыток оставить вне обучения
для проверки восстановления; их окна не передавать в corrections.

Если переэкспортировали те же bags с новым pause-policy, не добавлять их поверх
старых окон из этих же bags: получится повторение данных. Для сравнения брать
обычную исходную базу и новый набор исправлений, как в примере выше.
Существующие агрегированные datasets не дополняются и не перезаписываются.

### Обучение нового run

```bash
ACTUAL_DATASET_ROOT="$PWD/data/lerobot/ur3_clean_table_dagger_pauses_v1" \
ACTUAL_TRAIN_OUTPUT="$PWD/outputs/train/diffusion_ur3_clean_table_dagger_pauses_v1_run1" \
ACTUAL_TRAIN_STEPS=10000 \
bash scripts/workflow.sh train \
  --save_freq=1000 --job_name=diffusion_ur3_clean_table_dagger_pauses_v1_run1
```

Нужна новая выходная папка. Это обучение с нуля на агрегированных данных;
слепой resume старого run после смены датасета не использовать из-за статистик
нормализации и состояния обучения. Скрипт автоматически читает holdout из
`merge_provenance.json`; не перекрывать `--dataset.eval_split` для такого набора.
`--save_freq` — интервал checkpoints; `ACTUAL_TRAIN_STEPS` — число optimizer steps.

В начале лога сверить dataset.root, Train/eval split, num_frames, steps и output.
Trackio включён скриптом:

```bash
uv run --frozen trackio show --project ur3-diffusion
```

В Trackio `episodes`/лог `ep:` — накопленный расчётный объём просмотренных кадров
в единицах средней длины эпизода, **не число уникальных записей**. Число уникальных
train-эпизодов — `dataset.num_episodes` в начале лога; validation туда не входит.

Для сохранения training log, если нужно:

```bash
mkdir -p outputs/train/logs
```

К команде обучения можно добавить `2>&1 | tee outputs/train/logs/<новое_имя>.log`.
Обычная команда работает в текущем терминале; закрытие терминала может остановить
обучение. Фоновый запуск оформлять отдельно, не считать его свойством workflow.

После обучения выбрать `.../checkpoints/010000/pretrained_model` по разделу 6.
Сравнивать полное выполнение и число вмешательств на одинаковых стартах, а не
только training loss. Модель не дообучается автоматически при записи новых bags.

## 10. Логи, trace и ROS

| Что | Команда из vr-dagger |
|---|---|
| Backend | `bash scripts/workflow.sh backend logs` |
| Камеры | `bash scripts/workflow.sh capture logs` |
| Обычная запись | `bash scripts/workflow.sh record demonstration logs` |
| DAgger-запись | `bash scripts/workflow.sh record corrections logs` |
| Общий bridge | `bash scripts/workflow.sh capture dagger-logs` |
| Legacy bridge | `bash scripts/workflow.sh capture policy-logs` |
| Runner | Терминал с policy/dagger |

Ctrl+C в просмотре логов закрывает просмотр. Ctrl+C в ручном record завершает bag,
а в runner завершает его; назначение зависит от процесса текущего терминала.

Перед тестом модели, отдельный терминал:

```bash
cd ~/Kosenkov/vr-dagger
bash scripts/workflow.sh trace --duration 90
```

Trace пассивен, не посылает команды. После завершения `/data/traces/...` в логе
соответствует `vr-dagger/data/traces/...` на хосте. Он содержит `events.jsonl`,
`metadata.json`, `summary.json` и выборочные JPEG (~2 Гц/камера). Это не полный bag
и не готовый обучающий набор. Trace не останавливает runner по окончании 90 секунд.

Фоновый trace при необходимости:

```bash
bash scripts/workflow.sh capture trace-start --duration 180
bash scripts/workflow.sh capture trace-logs
# Если ещё работает и надо закончить раньше:
bash scripts/workflow.sh capture trace-stop
```

После автозавершения контейнер trace удаляется: повторный trace-stop может сообщить,
что контейнера нет. Данные уже на хосте. Пересчитать сводку:

```bash
uv run --frozen python scripts/policy_trace.py analyze data/traces/policy_YYYYMMDD_HHMMSS
```

В `/policy/prediction` `gripper_filter.input` — до гистерезиса, output — после;
`measured_gripper.position_norm` — измеренная позиция, requested_norm — команда.
В continuous блока gripper_filter нет. Текстовое g= в bridge — команда после фильтра.
Контакт OBJ=1/2 не равен достижению закрытой цели и не измеряет силу.

Сотрудник при проблеме передаёт **путь trace, точный POLICY_PATH, команду запуска,
пороги/sampler/steps и описание ожидаемого поведения**. В DAgger bag checkpoint и
policy_config записаны в /dagger/status; обычный trace пока не сохраняет путь
checkpoint, поэтому его нужно сообщать отдельно.

Просмотр топиков из backend:

```bash
cd ~/Kosenkov/ur3-vr-teleoperation
./scripts/run.sh shell
```

В контейнере:

```bash
source /opt/ros/humble/setup.bash
source /ws/install/setup.bash
ros2 topic list
ros2 topic info /teleop/tcp_actual --verbose
ros2 topic echo /teleop/control_status --once
ros2 topic echo /teleop/gripper_state --once
ros2 topic echo /dagger/status --once
ros2 topic echo /policy/prediction
```

`--verbose` или `-v`, не `-verbose`. Exit закрывает shell. /dagger/status появляется
после подключения runner. Мёртвая подписка в списке не доказывает неисправность:
нужны publisher count и фактическое поступление сообщений нужного рабочего топика.

## 11. Troubleshooting

### Камеры: нет устройства, чёрный поток, 0 FPS

Не запускать camera_preview одновременно с capture: один V4L2-поток не открывать
двумя процессами. Сначала закончить bag/runner и выбрать STOP. Затем остановить
capture (это не останавливает teleop):

```bash
cd ~/Kosenkov/vr-dagger
bash scripts/workflow.sh capture down
uv run --no-project --script scripts/camera_preview.py --list
cat config/cameras.env
```

Wrist — Logitech C930e; external — **RGB** RealSense D435. Одна камера может иметь
несколько `/dev/videoN`, часть из них depth/metadata, а не RGB. На этой установке
RealSense by-id video-index0 может быть depth Z16; RGB настроен через устойчивый
by-path с USB-интерфейсом 1.3. Номера videoN могут меняться при переподключении;
by-path зависит от USB-порта. После перестановки кабеля сверить config/cameras.env.

Посмотреть поддерживаемые форматы выбранного устройства, если установлен v4l-utils:

```bash
v4l2-ctl --list-formats-ext -d /dev/video6
```

`/dev/video6` — только пример, выбрать фактическое RGB-устройство из списка.
Для просмотра параметры camera_preview:

```bash
uv run --no-project --script scripts/camera_preview.py --help
```

Обновить `config/cameras.env`; старые export имеют приоритет, поэтому:

```bash
unset WRIST_DEVICE EXTERNAL_DEVICE
bash scripts/workflow.sh cameras
bash scripts/workflow.sh capture logs
bash scripts/workflow.sh check
```

После USB-переподключения Docker должен заново получить device mapping: down →
cameras, затем снова нужный bridge/рекордер. Пересборка из-за нового device path не нужна.
Обычный check требует камеры и робот, но не движения для проверки доступности.

### Топики видны, но check не получает сообщения

Сверить `Capture ROS_DOMAIN_ID=... ROS_LOCALHOST_ONLY=...` с backend. Скрипт читает
настройки из работающего контейнера `ur3-vr-teleoperation-teleop-1`; для другого
имени задать TELEOP_CONTAINER. Явные переменные терминала приоритетнее автоопределения.
Если контейнер не найден, defaults capture могут отличаться от ROS backend.
Проверить backend ps, RMW/DDS, сеть Quest и фактические издатели. Capture использует
свой UDP DDS-профиль; название топика само по себе не подтверждает доставку.

### Не начинается B-запись

Читать record ... logs: READY/ERROR, отсутствующие каналы, место на диске.
Для corrections сначала подключить runner: нужен живой /dagger/status.
Одновременно включить два B-рекордера нельзя. Сначала B → IDLE → stop старого,
затем start нового. B нужно нажимать на правом контроллере с работающим Unity APK.

### Нет ручного движения после v

Проверить mode=VR, phase=human и совпадение epoch. Отпустить Trigger и снова нажать;
удерживаемая со старой эпохи кнопка не должна возобновлять движение. Проверить
tracking/RTDE статусы; не обходить защиту повторным автоматическим переключением.

### POLICY_TIMEOUT, STOP, RTDE not ready, Servo halt

Сначала прочитать /teleop/control_status.reason, /teleop/rtde_status и логи.
Возможны медленный inference, устаревшие камеры/feedback, превышение цели или
отказ Servo. Не расширять лимиты и watchdog для обхода первичной причины.
S подтверждает ошибку runner, но не снимает аппаратный protective stop и не
заменяет физическое release-to-rearm. После устранения причины — s, явный p/v.

### Гриппер не открывается или сам раскрывается

Сравнить input модели, output фильтра, /teleop/gripper_command и feedback.
Если input не пересёк open, фильтр не получал намерения открыть. Если output=0,
а requested_position остаётся закрытым — разбирать доставку/источник. Если цель
открытия дошла, но POS не меняется — смотреть исполнитель и объект. При обучении
проверить, что новые исправления действительно содержат отпускания в нужных позах.

### Ошибка конвертации или existing output

Проверить clean finish episode.json, камеры, measured feedback, причины excluded,
short_windows и _batch_report. Неполные/старые выходы не удаляются автоматически.
Для другого отбора/selector версии выбрать новую root. Не увеличивать max-age
и не подставлять неизвестный gripper нулём. No expert windows означает отсутствие
подходящего материала по правилам отбора, а не повреждение файла обязательно.

## 12. Завершение работы

1. Остановить управление: s в общем runner или backend mode STOP.
2. Завершить bag через B и дождаться IDLE; ручной record — Ctrl+C и финализация.
3. Q завершает runner. Остановить trace, если он ещё работает.
4. Остановить сервисы:

```bash
cd ~/Kosenkov/vr-dagger
bash scripts/workflow.sh capture down
bash scripts/workflow.sh backend down
```

Datasets, bags и модели остаются на хосте. Завершение B/trace или закрытие логов
не заменяет STOP. Исторические datasets и checkpoints не удалять при смене сценария.

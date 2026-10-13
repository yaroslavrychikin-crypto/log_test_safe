# log_test_safe


Все примеры предполагают, что вы работаете в Linux и у вас есть `bash` (версия 4+ для ассоциативных массивов), `coreutils`, `tar`, `curl`, `timeout`, `flock`.

---

## Куски 1–3. Основы строгого режима и логирования

### Кусок 1. Минимальный строгий скрипт

**Файл:** `01-strict.sh`

```bash
#!/usr/bin/env bash
# Демонстрация строгого режима bash.
set -Eeuo pipefail

echo "Скрипт запущен"

# Эта строка НЕ вызовет падения, потому что обёрнута в if.
if false; then
  echo "не выполнится"
fi

# А вот эта — вызовет падение, потому что set -e.
# Раскомментируйте, чтобы увидеть:
# false

echo "Скрипт завершён"
```

**Запуск:**

```bash
chmod +x 01-strict.sh
./01-strict.sh
echo "Код выхода: $?"
```

**Что проверить:** уберите `if false` и оставьте просто `false` — скрипт упадёт сразу после `false`, не дойдя до `echo "Скрипт завершён"`.

---

### Кусок 2. Логирование с уровнями

**Файл:** `02-logging.sh`

```bash
#!/usr/bin/env bash
# Логирование с уровнями и цветами.
set -Eeuo pipefail

# --- Уровни логирования как ассоциативный массив ---
declare -A LOG_LEVELS=(
  [DEBUG]=10
  [INFO]=20
  [WARN]=30
  [ERROR]=40
  [FATAL]=50
)

# --- Цвета (только если stdout — терминал) ---
if [[ -t 2 ]]; then
  C_RESET=$'\033[0m'
  C_DEBUG=$'\033[36m'   # голубой
  C_INFO=$'\033[32m'    # зелёный
  C_WARN=$'\033[33m'    # жёлтый
  C_ERROR=$'\033[31m'   # красный
  C_FATAL=$'\033[35m'   # пурпурный
else
  C_RESET=""; C_DEBUG=""; C_INFO=""; C_WARN=""; C_ERROR=""; C_FATAL=""
fi

# --- Текущий порог. Переопределяется через окружение. ---
: "${LOG_LEVEL:=INFO}"

# --- Внутренняя функция логирования ---
_log() {
  local level="$1"; shift
  local msg="$*"

  # Фильтр по уровню.
  if (( ${LOG_LEVELS[$level]:-20} < ${LOG_LEVELS[$LOG_LEVEL]:-20} )); then
    return 0
  fi

  local ts; ts="$(date '+%Y-%m-%dT%H:%M:%S%z')"
  local color_var="C_$level"
  local color="${!color_var:-}"

  # Логи идут в stderr, чтобы stdout оставался чистым.
  printf '%s[%s] [%-5s] %s%s\n' "$color" "$ts" "$level" "$msg" "$C_RESET" >&2
}

log_debug() { _log DEBUG "$@"; }
log_info()  { _log INFO  "$@"; }
log_warn()  { _log WARN  "$@"; }
log_error() { _log ERROR "$@"; }
log_fatal() { _log FATAL "$@"; }

# --- Демонстрация ---
log_debug "Это DEBUG — видно только при LOG_LEVEL=DEBUG"
log_info  "Это INFO"
log_warn  "Это WARN"
log_error "Это ERROR"
log_fatal "Это FATAL"

# Логи и данные разделены: это уйдёт в stdout, а не в лог.
echo "результат работы"
```

**Запуск:**

```bash
chmod +x 02-logging.sh
./02-logging.sh                    # увидим INFO, WARN, ERROR, FATAL
./02-logging.sh 2>/dev/null        # только "результат работы"
LOG_LEVEL=DEBUG ./02-logging.sh    # увидим и DEBUG
LOG_LEVEL=ERROR ./02-logging.sh    # только ERROR и FATAL
```

**Что проверить:** `LOG_LEVEL` берётся из окружения — значит, его можно задавать снаружи, не редактируя скрипт. Это базовый механизм конфигурируемости.

---

### Кусок 3. Логирование в файл + stderr одновременно

**Файл:** `03-logging-file.sh`

```bash
#!/usr/bin/env bash
# Логи одновременно в stderr и в файл.
set -Eeuo pipefail

: "${LOG_LEVEL:=INFO}"
: "${LOG_DIR:=/tmp/diag-demo}"

declare -A LOG_LEVELS=([DEBUG]=10 [INFO]=20 [WARN]=30 [ERROR]=40 [FATAL]=50)
LOG_FILE=""

_log() {
  local level="$1"; shift
  local msg="$*"

  (( ${LOG_LEVELS[$level]:-20} < ${LOG_LEVELS[$LOG_LEVEL]:-20} )) && return 0

  local ts; ts="$(date '+%Y-%m-%dT%H:%M:%S%z')"
  local line="[$ts] [$level] $msg"

  printf '%s\n' "$line" >&2
  [[ -n "$LOG_FILE" ]] && printf '%s\n' "$line" >>"$LOG_FILE"
}

log_debug() { _log DEBUG "$@"; }
log_info()  { _log INFO  "$@"; }
log_warn()  { _log WARN  "$@"; }
log_error() { _log ERROR "$@"; }

# --- Инициализация лог-файла ---
mkdir -p -- "$LOG_DIR"
LOG_FILE="$LOG_DIR/demo-$(date +%Y%m%d-%H%M%S)-$$.log"
: >"$LOG_FILE"

log_info "Старт, лог пишется в $LOG_FILE"
log_debug "Отладочное сообщение"
log_warn  "Предупреждение"
log_info  "Завершение"

echo "Лог-файл: $LOG_FILE"
```

**Запуск:**

```bash
chmod +x 03-logging-file.sh
./03-logging-file.sh
cat /tmp/diag-demo/demo-*.log | tail -n 5
```

**Что проверить:** логи пишутся и на экран (stderr), и в файл. Имя файла содержит дату и PID — параллельные запуски не конфликтуют.

---

## Куски 4–6. Trap'ы: очистка, ошибки, сигналы

### Кусок 4. Trap EXIT — гарантированная очистка

**Файл:** `04-trap-exit.sh`

```bash
#!/usr/bin/env bash
# trap EXIT: код выполняется при любом завершении скрипта.
set -Eeuo pipefail

# Создаём временный каталог.
TMP_DIR="$(mktemp -d -t demo.XXXXXX)"
echo "Создан временный каталог: $TMP_DIR"

# Регистрируем очистку. Она сработает:
#  * при нормальном выходе,
#  * при exit 1,
#  * при падении из-за set -e,
#  * при Ctrl+C,
#  * при kill.
trap 'echo "[cleanup] удаляю $TMP_DIR"; rm -rf -- "$TMP_DIR"' EXIT

# Что-то делаем во временном каталоге.
touch "$TMP_DIR/file1" "$TMP_DIR/file2"
ls -la "$TMP_DIR"

# Дальше — либо нормальный выход, либо ошибка.
# Попробуйте раскомментировать:
# exit 1
# false

echo "Работа завершена"
```

**Запуск:**

```bash
chmod +x 04-trap-exit.sh
./04-trap-exit.sh
# В конце — сообщение "[cleanup] удаляю /tmp/demo.XXXXXX"

# Проверьте, что каталог удалён:
ls /tmp/demo.* 2>/dev/null || echo "каталогов не осталось"

# А теперь попробуйте прервать Ctrl+C во время работы:
./04-trap-exit.sh   # нажмите Ctrl+C сразу после старта
```

**Что проверить:** `_cleanup` срабатывает при любом сценарии, включая Ctrl+C. Без trap'а временный каталог остался бы мусорить в `/tmp`.

---

### Кусок 5. Trap ERR — логирование ошибок с контекстом

**Файл:** `05-trap-err.sh`

```bash
#!/usr/bin/env bash
# trap ERR: получаем код, строку и команду, которая упала.
set -Eeuo pipefail

# -E нужен, чтобы trap ERR срабатывал внутри функций,
# а не только на верхнем уровне.
trap 'echo "[ERR] rc=$? line=$LINENO cmd=$BASH_COMMAND" >&2' ERR

# Хорошая функция.
do_ok() {
  echo "всё хорошо"
}

# Плохая функция — упадёт.
do_fail() {
  false
  echo "сюда не дойдём"
}

do_ok
# do_fail   # раскомментируйте, чтобы увидеть trap в действии
echo "Продолжаем работу"
```

**Запуск:**

```bash
chmod +x 05-trap-err.sh
./05-trap-err.sh
# Раскомментируйте do_fail и запустите снова:
# ./05-trap-err.sh
```

**Что проверить:** при падении внутри функции вы увидите `[ERR] rc=1 line=... cmd=false`. Без `-E` в `set` этот trap не сработал бы внутри функции.

---

### Кусок 6. Обработка сигналов INT и TERM

**Файл:** `06-signals.sh`

```bash
#!/usr/bin/env bash
# Обработка SIGINT (Ctrl+C) и SIGTERM (kill).
set -Eeuo pipefail

# Функция очистки — вынесена отдельно, чтобы её могли вызвать оба trap'а.
cleanup() {
  echo "[cleanup] освобождаю ресурсы..."
  rm -f -- /tmp/demo-signal.lock 2>/dev/null || true
}

# trap EXIT выполнится ВСЕГДА.
trap cleanup EXIT

# INT/TERM — перехватываем и завершаемся с правильным кодом.
# 130 = 128 + 2 (SIGINT), 143 = 128 + 15 (SIGTERM).
trap 'echo "[signal] INT"; exit 130' INT
trap 'echo "[signal] TERM"; exit 143' TERM

touch /tmp/demo-signal.lock

echo "Работаю... (нажмите Ctrl+C или убейте снаружи: kill $$)"
# Долгий цикл — эмуляция полезной работы.
for i in $(seq 1 30); do
  sleep 1
  echo "  тик $i"
done
echo "Досчитал до конца"
```

**Запуск:**

```bash
chmod +x 06-signals.sh
./06-signals.sh
# Нажмите Ctrl+C через пару секунд — увидите [signal] INT и [cleanup].
# Проверьте код выхода:
echo "rc=$?"

# Или убейте из другого терминала:
./06-signals.sh &
sleep 2
kill -TERM %1
wait
```

**Что проверить:** при Ctrl+C вызывается и обработчик сигнала, и `cleanup` через EXIT. Скрипт не оставляет lock-файл.

---

## Кусок 7. Безопасная загрузка конфига

**Файл:** `07-config.sh`

```bash
#!/usr/bin/env bash
# Безопасный парсер конфига: KEY=VALUE, без source.
set -Eeuo pipefail

load_config() {
  local file="$1"
  [[ -r "$file" ]] || { echo "не читается: $file" >&2; return 1; }

  # Проверка прав: последняя цифра не должна быть 2/3/6/7
  # (иначе файл пишется группой или всеми — потенциальная инъекция).
  local perms
  perms="$(stat -c '%a' "$file" 2>/dev/null || stat -f '%Lp' "$file")"
  if [[ "${perms: -1}" =~ [2367] ]]; then
    echo "ОПАСНО: $file пишется группой/всеми ($perms)" >&2
    return 1
  fi

  local line key val
  while IFS= read -r line || [[ -n "$line" ]]; do
    # Комментарии и пустые строки — пропускаем.
    [[ -z "$line" || "$line" =~ ^[[:space:]]*# ]] && continue

    # Принимаем только KEY=VALUE.
    if [[ "$line" =~ ^([A-Za-z_][A-Za-z0-9_]*)=(.*)$ ]]; then
      key="${BASH_REMATCH[1]}"
      val="${BASH_REMATCH[2]}"
      # Снимаем кавычки.
      val="${val%\"}"; val="${val#\"}"
      # printf -v — безопасное присвоение по имени.
      printf -v "$key" '%s' "$val"
      export "$key"
      echo "  $key=$val"
    else
      echo "  ПРОПУЩЕНО: $line" >&2
    fi
  done <"$file"
}

# --- Создаём тестовый конфиг ---
CFG="$(mktemp)"
cat >"$CFG" <<'EOF'
# Это комментарий
LOG_LEVEL=DEBUG
LOG_DIR="/tmp/my-logs"
CMD_TIMEOUT=15

# Опасная строка — должна быть пропущена
rm -rf / # это НЕ выполнится
EOF
chmod 600 "$CFG"

# --- Загружаем ---
echo "Загружаю $CFG:"
load_config "$CFG" || exit 1

echo
echo "LOG_LEVEL=$LOG_LEVEL"
echo "LOG_DIR=$LOG_DIR"
echo "CMD_TIMEOUT=$CMD_TIMEOUT"

rm -f "$CFG"
```

**Запуск:**

```bash
chmod +x 07-config.sh
./07-config.sh
```

**Что проверить:** строка `rm -rf / # это НЕ выполнится` не соответствует шаблону `KEY=VALUE` (в ней есть пробел перед `#`), поэтому она пропускается. Даже если бы она прошла парсер, `printf -v` не исполняет код — в отличие от `source`.

Попробуйте заменить парсер на `source "$CFG"` — и вы поймёте, чем это опасно.

---

## Куски 8–9. Парсинг CLI и валидация

### Кусок 8. Ручной парсинг аргументов

**Файл:** `08-args.sh`

```bash
#!/usr/bin/env bash
# Парсинг CLI с короткими и длинными опциями.
set -Eeuo pipefail

: "${LOG_LEVEL:=INFO}"
: "${OUT_DIR:=/tmp/diag-out}"

usage() {
  cat <<EOF
Использование: $0 [опции]

  -l, --level LEVEL   уровень логирования (DEBUG|INFO|WARN|ERROR)
  -o, --out DIR       каталог вывода
  -h, --help          справка
  -V, --version       версия
EOF
}

parse_args() {
  while (($#)); do
    case "$1" in
      -l|--level)
        LOG_LEVEL="${2:?--level требует значение}"
        shift 2
        ;;
      -o|--out)
        OUT_DIR="${2:?--out требует значение}"
        shift 2
        ;;
      -h|--help)
        usage; exit 0
        ;;
      -V|--version)
        echo "demo 1.0.0"; exit 0
        ;;
      --)
        shift; break
        ;;
      -*)
        echo "Неизвестная опция: $1" >&2
        usage >&2
        exit 2
        ;;
      *)
        echo "Лишний аргумент: $1" >&2
        exit 2
        ;;
    esac
  done
}

parse_args "$@"

echo "LOG_LEVEL=$LOG_LEVEL"
echo "OUT_DIR=$OUT_DIR"
```

**Запуск:**

```bash
chmod +x 08-args.sh

./08-args.sh                                  # дефолты
./08-args.sh -l DEBUG                         # короткая опция
./08-args.sh --level=WARN                     # ВНИМАНИЕ: не сработает, см. ниже
./08-args.sh --level WARN -o /tmp/custom
./08-args.sh -l                               # ошибка: --level требует значение
./08-args.sh --help
```

**Что проверить:** этот парсер **не поддерживает** форму `--level=WARN`. Если она нужна — добавьте ветку:

```bash
--level=*)
  LOG_LEVEL="${1#--level=}"
  shift
  ;;
```

---

### Кусок 9. Валидация параметров

**Файл:** `09-validate.sh`

```bash
#!/usr/bin/env bash
# Fail-fast валидация: падаем до начала работы, если конфиг плохой.
set -Eeuo pipefail

: "${LOG_LEVEL:=INFO}"
: "${CMD_TIMEOUT:=10}"
: "${DISK_WARN_PCT:=80}"
: "${DISK_CRIT_PCT:=90}"

die() { echo "ОШИБКА: $*" >&2; exit 1; }

validate_config() {
  [[ "$LOG_LEVEL" =~ ^(DEBUG|INFO|WARN|ERROR)$ ]] \
    || die "LOG_LEVEL должен быть DEBUG|INFO|WARN|ERROR, получено: '$LOG_LEVEL'"

  [[ "$CMD_TIMEOUT" =~ ^[0-9]+$ ]] \
    || die "CMD_TIMEOUT должен быть целым числом, получено: '$CMD_TIMEOUT'"

  (( CMD_TIMEOUT > 0 && CMD_TIMEOUT <= 300 )) \
    || die "CMD_TIMEOUT должен быть в диапазоне 1..300"

  [[ "$DISK_WARN_PCT" =~ ^[0-9]+$ ]] || die "DISK_WARN_PCT должен быть числом"
  [[ "$DISK_CRIT_PCT" =~ ^[0-9]+$ ]] || die "DISK_CRIT_PCT должен быть числом"

  (( DISK_WARN_PCT < DISK_CRIT_PCT )) \
    || die "DISK_WARN_PCT ($DISK_WARN_PCT) должен быть < DISK_CRIT_PCT ($DISK_CRIT_PCT)"

  echo "Валидация пройдена: LOG_LEVEL=$LOG_LEVEL, CMD_TIMEOUT=$CMD_TIMEOUT, пороги: $DISK_WARN_PCT/$DISK_CRIT_PCT"
}

validate_config
```

**Запуск:**

```bash
chmod +x 09-validate.sh

./09-validate.sh
LOG_LEVEL=BOGUS ./09-validate.sh
CMD_TIMEOUT=abc ./09-validate.sh
CMD_TIMEOUT=9999 ./09-validate.sh
DISK_WARN_PCT=95 DISK_CRIT_PCT=90 ./09-validate.sh
```

**Что проверить:** каждое нарушение даёт понятное сообщение и код выхода 1.

---

## Кусок 10. Безопасный запуск внешних команд с timeout

**Файл:** `10-run.sh`

```bash
#!/usr/bin/env bash
# run: обёртка над внешними командами — таймаут, логи, stdin.
set -Eeuo pipefail

: "${CMD_TIMEOUT:=5}"

# --- Внутренняя функция логирования (упрощённая) ---
log_debug() { printf '[DEBUG] %s\n' "$*" >&2; }
log_warn()  { printf '[WARN]  %s\n' "$*" >&2; }

run() {
  local desc="$1"; shift
  log_debug "RUN: $desc :: $*"

  local out rc
  # timeout --signal=TERM Ns cmd — если команда не завершится за N секунд,
  # ей отправится SIGTERM. `</dev/null` отвязывает stdin.
  # `2>&1` — stderr в stdout, чтобы весь вывод попал в отчёт.
  if out="$(timeout --signal=TERM "${CMD_TIMEOUT}s" "$@" 2>&1 </dev/null)"; then
    printf '%s\n' "$out"
    return 0
  else
    rc=$?
    log_warn "Команда '$desc' вернула код $rc"
    printf '%s\n' "$out"
    return $rc
  fi
}

echo "=== Быстрая команда ==="
run "uname" uname -a

echo
echo "=== Медленная команда (упадёт по таймауту) ==="
run "sleep" sleep 30 || echo "Как и ожидалось, команда прервана"

echo
echo "=== Несуществующая команда ==="
run "nonexistent" /no/such/binary || echo "Команда не найдена, продолжили"
```

**Запуск:**

```bash
chmod +x 10-run.sh
time ./10-run.sh
```

**Что проверить:** `sleep 30` прерывается через 5 секунд (значение `CMD_TIMEOUT`). Общее время выполнения — примерно 5 секунд, а не 30.

Попробуйте `CMD_TIMEOUT=1 ./10-run.sh` — станет ещё быстрее.

---

## Куски 11–13. Модули диагностики

### Кусок 11. Проверка системы

**Файл:** `11-check-system.sh`

```bash
#!/usr/bin/env bash
# Сбор базовой информации о системе в файл отчёта.
set -Eeuo pipefail

: "${OUT_DIR:=/tmp/diag-out}"
mkdir -p -- "$OUT_DIR"

run() {
  timeout 5s "$@" 2>&1 </dev/null || true
}

{
  echo "=== hostname ==="
  run hostname

  echo
  echo "=== uname ==="
  run uname -a

  echo
  echo "=== uptime ==="
  run uptime

  echo
  echo "=== who ==="
  run who

  echo
  echo "=== date (UTC) ==="
  run date -u

  echo
  echo "=== /etc/os-release ==="
  if [[ -r /etc/os-release ]]; then
    cat /etc/os-release
  else
    echo "(нет файла)"
  fi
} >"$OUT_DIR/system.txt"

echo "Отчёт: $OUT_DIR/system.txt"
cat "$OUT_DIR/system.txt"
```

**Запуск:**

```bash
chmod +x 11-check-system.sh
./11-check-system.sh
```

**Что проверить:** всё пишется в `system.txt` и дублируется на экран. `|| true` страхует, если какая-то команда недоступна.

---

### Кусок 12. Проверка дисков с сигналами WARN/ERROR

**Файл:** `12-check-disks.sh`

```bash
#!/usr/bin/env bash
# Анализ заполнения диска с порогами WARN/CRIT.
set -Eeuo pipefail

: "${DISK_WARN_PCT:=80}"
: "${DISK_CRIT_PCT:=90}"

log_info()  { printf '[INFO]  %s\n' "$*"; }
log_warn()  { printf '[WARN]  %s\n' "$*" >&2; }
log_error() { printf '[ERROR] %s\n' "$*" >&2; }

# df -hP — POSIX-формат, одна строка на ФС.
# awk 'NR>1' — пропускаем заголовок.
# < <(...) — process substitution: while не в subshell, переменные живы.
while read -r fs size used avail pct mount; do
  # Извлекаем число из "85%".
  [[ "$pct" =~ ^([0-9]+)%$ ]] || continue
  p="${BASH_REMATCH[1]}"

  if   (( p >= DISK_CRIT_PCT )); then
    log_error "$mount: ${p}% (критично, порог ${DISK_CRIT_PCT}%)"
  elif (( p >= DISK_WARN_PCT )); then
    log_warn  "$mount: ${p}% (внимание, порог ${DISK_WARN_PCT}%)"
  else
    log_info  "$mount: ${p}% (OK)"
  fi
done < <(df -hP | awk 'NR>1')
```

**Запуск:**

```bash
chmod +x 12-check-disks.sh
./12-check-disks.sh

# Хотим увидеть срабатывание WARN/ERROR — снижаем пороги:
DISK_WARN_PCT=1 DISK_CRIT_PCT=2 ./12-check-disks.sh
```

**Что проверить:** при снижении порогов появятся предупреждения. Логика порогов проверяется без реального заполнения диска.

---

### Кусок 13. HTTP-проверки

**Файл:** `13-check-http.sh`

```bash
#!/usr/bin/env bash
# Проверка доступности HTTP-сервисов.
set -Eeuo pipefail

: "${HTTP_TARGETS:=https://example.com https://httpbin.org/status/500}"
: "${CMD_TIMEOUT:=5}"

log_info()  { printf '[INFO]  %s\n' "$*"; }
log_error() { printf '[ERROR] %s\n' "$*" >&2; }

# Разбиваем строку на отдельные URL по пробелам (сознательно без кавычек,
# потому что мы сами контролируем содержимое HTTP_TARGETS).
for t in $HTTP_TARGETS; do
  # curl -sS -o /dev/null -w '%{http_code}' — печатает только код ответа.
  # 2>/dev/null — подавляем сообщения curl.
  # || echo 000 — если curl вообще не смог подключиться.
  code="$(timeout "${CMD_TIMEOUT}s" curl -sS -o /dev/null -w '%{http_code}' "$t" 2>/dev/null || echo 000)"

  if [[ "$code" =~ ^2 ]]; then
    log_info  "OK   $t (HTTP $code)"
  else
    log_error "FAIL $t (HTTP $code)"
  fi
done
```

**Запуск:**

```bash
chmod +x 13-check-http.sh
./13-check-http.sh

# Свой список:
HTTP_TARGETS="https://ya.ru https://google.com https://nonexistent.invalid" ./13-check-http.sh
```

**Что проверить:** `https://httpbin.org/status/500` даст `FAIL` с кодом 500, а `nonexistent.invalid` — `000` (не удалось подключиться).

---

## Кусок 14. Блокировка от параллельного запуска

**Файл:** `14-flock.sh`

```bash
#!/usr/bin/env bash
# flock: не даём двум экземплярам работать одновременно.
set -Eeuo pipefail

LOCK_FILE="/tmp/demo-flock.lock"

# Открываем FD 9 на файл блокировки.
exec 9>"$LOCK_FILE" || { echo "не могу открыть $LOCK_FILE" >&2; exit 1; }

# flock -n 9 — неблокирующая попытка взять блокировку.
# Если занято — падаем сразу, не ждём.
if ! flock -n 9; then
  echo "Другой экземпляр уже работает (lock: $LOCK_FILE)" >&2
  exit 1
fi

echo "Блокировка взята. Работаю 10 секунд..."
for i in $(seq 1 10); do
  echo "  тик $i"
  sleep 1
done
echo "Готово"
# Блокировка снимется автоматически при выходе процесса.
```

**Запуск** (нужны два терминала):

```bash
chmod +x 14-flock.sh

# Терминал 1:
./14-flock.sh

# Терминал 2 (пока первый работает):
./14-flock.sh
# Ожидаемый вывод: "Другой экземпляр уже работает"
```

**Что проверить:** второй запуск падает моментально. После завершения первого — второй запускается нормально. Блокировка снимается самим ядром при завершении процесса, даже если скрипт убит `kill -9`.

---

## Кусок 15. Маскирование секретов в логах

**Файл:** `15-mask-secrets.sh`

```bash
#!/usr/bin/env bash
# Замена паролей/токенов на *** перед логированием.
set -Eeuo pipefail

# Функция-фильтр: читает stdin, пишет в stdout с маскированием.
mask_secrets() {
  # sed -E — расширенные regex.
  # \1=*** — оставляем имя параметра, значение затираем.
  sed -E \
    -e 's/(password|passwd|pwd|token|secret|api[_-]?key)=([^[:space:]]+)/\1=***/gi' \
    -e 's/(Authorization:[[:space:]]*Bearer[[:space:]]+)[^[:space:]]+/\1***/gi'
}

echo "=== Демонстрация ==="
cat <<'EOF' | mask_secrets
user=alice password=s3cr3t host=example.com
token=abcdef1234567890
api_key=xyz-9876-5432
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9
нормальная строка без секретов
EOF
```

**Запуск:**

```bash
chmod +x 15-mask-secrets.sh
./15-mask-secrets.sh
```

**Ожидаемый вывод:**

```
user=alice password=*** host=example.com
token=***
api_key=***
Authorization: Bearer ***
нормальная строка без секретов
```

**Что проверить:** значения секретов заменены на `***`, а обычные строки не тронуты. Функцию можно вставить в пайп перед записью в лог:

```bash
some_command | mask_secrets >> "$LOG_FILE"
```

---

## Кусок 16. Финальная сборка — всё вместе

**Файл:** `16-all-in-one.sh`

```bash
#!/usr/bin/env bash
# ============================================================================
# Мини-версия диагностического скрипта, собранная из всех кусков.
# ============================================================================
set -Eeuo pipefail
shopt -s inherit_errexit nullglob
umask 077
IFS=$'\n\t'

readonly SCRIPT_NAME="${0##*/}"
readonly SCRIPT_VERSION="1.0.0"
readonly SCRIPT_PID=$$

# --- Дефолты ---
: "${LOG_LEVEL:=INFO}"
: "${LOG_DIR:=/tmp/diag-final-logs}"
: "${OUT_DIR:=/tmp/diag-final-out}"
: "${CMD_TIMEOUT:=10}"
: "${DISK_WARN_PCT:=80}"
: "${DISK_CRIT_PCT:=90}"
: "${HTTP_TARGETS:=}"
: "${CONFIG_FILE:=/etc/diag.conf}"

# --- Логирование ---
declare -A LOG_LEVELS=([DEBUG]=10 [INFO]=20 [WARN]=30 [ERROR]=40 [FATAL]=50)
LOG_FILE=""

_log() {
  local level="$1"; shift
  local msg="$*"
  (( ${LOG_LEVELS[$level]:-20} < ${LOG_LEVELS[$LOG_LEVEL]:-20} )) && return 0
  local ts; ts="$(date '+%Y-%m-%dT%H:%M:%S%z')"
  local line="[$ts] [$level] (${FUNCNAME[2]:-main}) $msg"
  printf '%s\n' "$line" >&2
  [[ -n "$LOG_FILE" ]] && printf '%s\n' "$line" >>"$LOG_FILE"
}
log_debug() { _log DEBUG "$@"; }
log_info()  { _log INFO  "$@"; }
log_warn()  { _log WARN  "$@"; }
log_error() { _log ERROR "$@"; }
log_fatal() { _log FATAL "$@"; }
die() { log_fatal "$*"; exit 1; }

# --- Trap'ы ---
_cleanup() {
  local rc=$?
  log_debug "Завершение, rc=$rc"
  [[ -n "${LOCK_FILE:-}" && -f "$LOCK_FILE" ]] && rm -f -- "$LOCK_FILE" 2>/dev/null || true
  [[ -n "${TMP_DIR:-}" && -d "$TMP_DIR" ]] && rm -rf -- "$TMP_DIR" 2>/dev/null || true
  exit "$rc"
}
on_error() {
  log_error "Ошибка (rc=$1) в строке $2: $3"
}
on_signal() {
  log_warn "Сигнал $1 — завершаемся"
  exit 130
}
trap '_cleanup' EXIT
trap 'on_error $? $LINENO "$BASH_COMMAND"' ERR
trap 'on_signal INT'  INT
trap 'on_signal TERM' TERM

# --- Конфиг ---
load_config() {
  local file="$1"
  [[ -r "$file" ]] || die "Конфиг не читается: $file"
  local perms
  perms="$(stat -c '%a' "$file" 2>/dev/null || stat -f '%Lp' "$file")"
  [[ "${perms: -1}" =~ [2367] ]] && die "Конфиг $file доступен на запись группе/всем"
  local line key val
  while IFS= read -r line || [[ -n "$line" ]]; do
    [[ -z "$line" || "$line" =~ ^[[:space:]]*# ]] && continue
    if [[ "$line" =~ ^([A-Za-z_][A-Za-z0-9_]*)=(.*)$ ]]; then
      key="${BASH_REMATCH[1]}"; val="${BASH_REMATCH[2]}"
      val="${val%\"}"; val="${val#\"}"; val="${val%\'}"; val="${val#\'}"
      printf -v "$key" '%s' "$val"
      export "$key"
    else
      log_warn "Пропущена строка конфига: $line"
    fi
  done <"$file"
}

# --- Args ---
usage() {
  cat <<EOF
$SCRIPT_NAME v$SCRIPT_VERSION

Использование: $SCRIPT_NAME [опции]
  -c, --config FILE   путь к конфигу (по умолчанию /etc/diag.conf)
  -l, --level LEVEL   уровень логов (по умолчанию INFO)
  -o, --out DIR       каталог отчётов
  -h, --help          справка
  -V, --version       версия
EOF
}
parse_args() {
  while (($#)); do
    case "$1" in
      -c|--config) CONFIG_FILE="${2:?требуется путь}"; shift 2 ;;
      -l|--level)  LOG_LEVEL="${2:?требуется уровень}"; shift 2 ;;
      -o|--out)    OUT_DIR="${2:?требуется каталог}"; shift 2 ;;
      -h|--help)   usage; exit 0 ;;
      -V|--version) echo "$SCRIPT_NAME $SCRIPT_VERSION"; exit 0 ;;
      --) shift; break ;;
      -*) die "Неизвестная опция: $1" ;;
      *)  die "Лишний аргумент: $1" ;;
    esac
  done
}

# --- Валидация ---
validate_config() {
  [[ "$LOG_LEVEL" =~ ^(DEBUG|INFO|WARN|ERROR)$ ]] \
    || die "LOG_LEVEL неверный: $LOG_LEVEL"
  [[ "$CMD_TIMEOUT" =~ ^[0-9]+$ ]] || die "CMD_TIMEOUT — целое"
  [[ "$DISK_WARN_PCT" =~ ^[0-9]+$ ]] || die "DISK_WARN_PCT — целое"
  [[ "$DISK_CRIT_PCT" =~ ^[0-9]+$ ]] || die "DISK_CRIT_PCT — целое"
  (( DISK_WARN_PCT < DISK_CRIT_PCT )) || die "WARN должен быть < CRIT"
}

# --- run ---
run() {
  local desc="$1"; shift
  log_debug "RUN: $desc :: $*"
  local out rc
  if out="$(timeout --signal=TERM "${CMD_TIMEOUT}s" "$@" 2>&1 </dev/null)"; then
    printf '%s\n' "$out"; return 0
  else
    rc=$?; log_warn "Команда '$desc' rc=$rc"; printf '%s\n' "$out"; return $rc
  fi
}

# --- Модули ---
check_system() {
  log_info "Системная информация"
  {
    printf '=== hostname ===\n';  run hostname hostname || true
    printf '=== uname ===\n';     run uname    uname -a  || true
    printf '=== uptime ===\n';    run uptime   uptime    || true
    printf '=== date ===\n';      run date     date -u   || true
  } >"$OUT_DIR/system.txt"
}

check_cpu_mem() {
  log_info "CPU и память"
  {
    printf '=== top ===\n'
    run top top -bn1 2>/dev/null | head -n 20 || true
    printf '\n=== free ===\n'
    run free free -h || true
  } >"$OUT_DIR/cpu_mem.txt"
}

check_disks() {
  log_info "Диски"
  {
    printf '=== df ===\n'
    run df df -hP || true
  } >"$OUT_DIR/disks.txt"
  while read -r fs size used avail pct mount; do
    [[ "$pct" =~ ^([0-9]+)%$ ]] || continue
    local p="${BASH_REMATCH[1]}"
    if   (( p >= DISK_CRIT_PCT )); then log_error "Критично: $mount = ${p}%"
    elif (( p >= DISK_WARN_PCT )); then log_warn  "Внимание: $mount = ${p}%"
    else log_debug "OK: $mount = ${p}%"
    fi
  done < <(df -hP | awk 'NR>1')
}

check_http() {
  [[ -z "$HTTP_TARGETS" ]] && { log_debug "HTTP_TARGETS пусто"; return 0; }
  log_info "HTTP-проверки"
  local t code
  for t in $HTTP_TARGETS; do
    code="$(timeout "${CMD_TIMEOUT}s" curl -sS -o /dev/null -w '%{http_code}' "$t" 2>/dev/null || echo 000)"
    if [[ "$code" =~ ^2 ]]; then log_info "OK   $t ($code)"
    else log_error "FAIL $t ($code)"
    fi
  done
}

build_report() {
  local tar="${OUT_DIR}.tar.gz"
  log_info "Архив: $tar"
  tar -czf "$tar" -C "$(dirname "$OUT_DIR")" "$(basename "$OUT_DIR")" \
    || die "tar упал"
}

# --- Main ---
main() {
  parse_args "$@"

  [[ -f "$CONFIG_FILE" ]] && load_config "$CONFIG_FILE"

  mkdir -p -- "$LOG_DIR" 2>/dev/null || true
  LOG_FILE="$LOG_DIR/diag-$(date +%Y%m%d-%H%M%S)-$$.log"
  : >"$LOG_FILE" 2>/dev/null || LOG_FILE=""

  log_info "Старт $SCRIPT_NAME v$SCRIPT_VERSION PID=$SCRIPT_PID"

  validate_config

  LOCK_FILE="/var/lock/diag.lock"
  if ! exec 9>"$LOCK_FILE" 2>/dev/null; then
    LOCK_FILE="/tmp/diag.lock"
    exec 9>"$LOCK_FILE" || die "Не могу создать lock"
  fi
  flock -n 9 || die "Другой экземпляр работает"

  mkdir -p -- "$OUT_DIR" || die "Не могу создать $OUT_DIR"
  TMP_DIR="$(mktemp -d -t diag.XXXXXX)"

  check_system  || log_error "check_system упал"
  check_cpu_mem || log_error "check_cpu_mem упал"
  check_disks   || log_error "check_disks упал"
  check_http    || log_error "check_http упал"

  build_report

  log_info "Завершено успешно"
  echo "Отчёт: ${OUT_DIR}.tar.gz"
  echo "Лог:   $LOG_FILE"
}

main "$@"
```

**Запуск:**

```bash
chmod +x 16-all-in-one.sh

# Базовый запуск.
./16-all-in-one.sh

# С другим уровнем логирования.
./16-all-in-one.sh --level DEBUG

# Со своими порогами (через ENV, т.к. отдельной опции нет).
DISK_WARN_PCT=1 DISK_CRIT_PCT=2 ./16-all-in-one.sh

# С HTTP-проверками.
HTTP_TARGETS="https://example.com https://httpbin.org/status/500" ./16-all-in-one.sh

# Посмотреть отчёт.
tar -tzf /tmp/diag-final-out.tar.gz
tar -xzf /tmp/diag-final-out.tar.gz -C /tmp/
cat /tmp/diag-final-out/system.txt
```
---


## Полезные команды для тестирования

```bash
# Проверить синтаксис без запуска.
bash -n script.sh

# Запустить с трассировкой (печатает каждую команду).
bash -x script.sh

# Линтер (установите: sudo apt install shellcheck).
shellcheck script.sh

# Форматтер (go install mvdan.cc/sh/v3/cmd/shfmt@latest).
shfmt -i 2 -ci script.sh

# Запустить с самым строгим режимом отладки.
PS4='+${BASH_SOURCE}:${LINENO}: ' bash -x script.sh
```

Начните с куска 1, запустите его, потом 2 — и так до 16. Каждый следующий опирается только на предыдущие, так что вы сможете разобраться с любой концепцией изолированно.

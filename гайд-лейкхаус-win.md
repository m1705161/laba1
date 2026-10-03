# Локальный Lakehouse — пошаговый гайд для новичков (Windows / PowerShell)

Стек: **MinIO → Iceberg (через Lakekeeper REST Catalog) → Trino**.

Это Windows-версия гайда. Всё то же самое, что в `гайд-лейкхаус.md`, но команды под **PowerShell** (дефолтный терминал VS Code на Windows). Каждый этап — одна команда `docker run` (без docker-compose и без скриптов). Проверено на Docker Desktop (Windows 10/11).

> 💡 Гайд читаешь в VS Code, команды выполняешь во встроенном терминале VS Code (**Terminal → New Terminal**). По умолчанию там открывается PowerShell — нужен именно он.

---

## 0. Что мы строим и зачем

**Lakehouse** — это когда данные лежат в *объектном хранилище* (как в data lake), но работать с ними можно через обычный **SQL** и таблицы (как в data warehouse).

Три компонента, каждый — отдельный Docker-контейнер:

| Компонент | Что делает | Аналог в облаке |
|---|---|---|
| **MinIO** | S3-совместимое объектное хранилище. Сюда физически падают файлы данных (parquet). | AWS S3 |
| **Lakekeeper** | Iceberg REST Catalog. Хранит метаданные таблиц (какие файлы/снапшоты относятся к таблице). | AWS Glue / Nessie |
| **Trino** | SQL-движок. Ты пишешь SQL, Trino ходит в Lakekeeper за метаданными и читает/пишет parquet в MinIO. | Athena / Presto |

Схема:

```
┌──────────┐   SQL    ┌─────────┐  метаданные ┌────────────┐
│  DBeaver │ ───────► │  Trino  │ ──────────► │ Lakekeeper │
└──────────┘          └────────┘             └────────────┘
                          │   читает / пишет parquet
                          ▼
                     ┌─────────┐
                     │  MinIO  │  (bucket `warehouse`)
                     └─────────┘
```

---

## 1. Пререквизиты (проверь один раз)

**1.1. Docker Desktop.** Скачай и установи: <https://www.docker.com/products/docker-desktop/>. При первом запуске Windows может попросить включить WSL 2 — соглашайся (мастер сделает сам, нужна перезагрузка). Дождись, пока значок кита перестанет «думать».

Проверь в терминале PowerShell:

```powershell
docker --version           # должно вывести что-то вроде Docker version 27.x
docker run --rm hello-world   # должно напечатать "Hello from Docker!"
```

Если `hello-world` отработал — Docker готов.

**1.2. Свободные порты.** Мы займём порты `9000`, `9001`, `5432`, `8181`, `8080`. Убедись, что на них никто не сидит:

```powershell
Get-NetTCPConnection -State Listen -LocalPort 9000,9001,5432,8181,8080 -ErrorAction SilentlyContinue
```

Пустой вывод (просто снова появилось приглашение `PS >`) = порты свободны.

> ⚠️ **Почему фильтр `-State Listen`?** Он показывает только реально слушающие сокеты. Без него можно увидеть «хвосты» соединений от Docker в состоянии закрытия и принять их за занятые порты. Нас интересуют только `Listen`.

**1.3. Что такое `docker run`.** Одна команда скачивает образ (если его нет локально) и запускает контейнер. Флаги, которые будут встречаться:

- `-d` — запустить в фоне (detached)
- `--name имя` — дать контейнеру понятное имя
- `-p ХОСТ:КОНТЕЙНЕР` — пробросить порт с контейнера на твой компьютер
- `-e КЛЮЧ=ЗНАЧЕНИЕ` — переменная окружения (пароли, настройки)
- `-v имя:/путь` — том для хранения данных (переживает перезапуск контейнера)
- `--rm` — удалить контейнер после завершения (для одноразовых команд)

> ⚠️ **Перенос строки в PowerShell.** В bash длинную команду продолжают обратным слэшем `\`, в PowerShell — обратным апострофом `` ` `` (клавиша слева от `1`). Важно: после `` ` `` **не должно быть пробела** — апостроф должен быть последним символом строки, иначе команда «обрежется» и упадёт с ошибкой. Копируй команды из гайда целиком, тогда всё ок.

**1.4. Рабочая папка.** Все файлы лабораторной (конфиги) держим в одной папке. Создай её в домашней директории и перейди в неё:

```powershell
mkdir "$HOME\lakehouse" -Force
cd "$HOME\lakehouse"
```

> ⚠️ **Почему не в корне `C:\`?** Корень диска (`C:\lakehouse` или `mkdir /lakehouse`) требует прав администратора — отсюда ошибка «Access denied». `$HOME` — это твоя домашняя папка (обычно `C:\Users\твой_логин`), туда права не нужны. Дальше все команды с `${PWD}` и `@create-warehouse.json` выполняются **из этой папки**. Сюда же мы положим файл `create-warehouse.json` и папку `catalog\` (создадим их по ходу гайда).

> 💡 **Файлы создаём в VS Code.** Открой эту папку в VS Code: **File → Open Folder →** выбери `C:\Users\твой_логин\lakehouse`. Создавать файлы можно двумя способами: (1) в терминале командой из гайда, или (2) в VS Code через **Explorer → New File** (иконка слева) → ввести имя файла → вставить содержимое → **Ctrl+S**.

---

## 2. Шпаргалка: порты и пароли

| Сервис | Адрес | Логин | Пароль |
|---|---|---|---|
| **MinIO** — консоль (UI) | http://localhost:9001 | `minioadmin` | `minioadmin` |
| **MinIO** — S3 API | http://localhost:9000 | `minioadmin` | `minioadmin` |
| **Postgres** | localhost:5432 | `postgres` | `postgres` |
| **Lakekeeper** — REST API | http://localhost:8181 | — | без авторизации |
| **Trino** | http://localhost:8080 | любой (например `trino`) | пусто |

> Все пароли — демо, только для локальной лабораторной.

---

## 3. Этап 1 — MinIO (объектное хранилище)

Запускаем MinIO одной командой:

```powershell
docker run -d --name lakehouse-minio `
  -p 9000:9000 -p 9001:9001 `
  -e MINIO_ROOT_USER=minioadmin `
  -e MINIO_ROOT_PASSWORD=minioadmin `
  -v minio-data:/data `
  pgsty/silo:latest server /data --console-address ":9001"
```

> Обрати внимание: официальные образы MinIO (`minio/minio` на Docker Hub, `quay.io/minio/minio`) больше не раздаются бесплатно, а AIStor требует лицензию. Поэтому используем свободный форк **Silo** — `pgsty/silo` (те же порты, переменные окружения и консоль, лицензия не нужна).

**Проверка:** открой http://localhost:9001 → логин `minioadmin`, пароль `minioadmin`. Должна открыться консоль MinIO.

**Создай bucket `warehouse`** (сюда Trino будет класть таблицы). В консоли: слева **Buckets → Create Bucket → имя `warehouse` → Create Bucket**.

Альтернатива через командную строку (то же самое, что кнопка в UI):

```powershell
docker run --rm --entrypoint /bin/sh `
  pgsty/mc:latest -c "mc alias set m http://host.docker.internal:9000 minioadmin minioadmin && mc mb --ignore-existing m/warehouse"
```

> 💡 На Docker Desktop для Windows `host.docker.internal` работает из коробки — флаг `--add-host host-gateway` (он нужен только на Linux) добавлять не надо.

---

## 4. Этап 2 — Lakekeeper (Iceberg REST Catalog)

Lakekeeper хранит свои метаданные в Postgres, поэтому поднимаем два контейнера: Postgres и сам Lakekeeper.

**4.1. Postgres:**

```powershell
docker run -d --name lakehouse-pg `
  -e POSTGRES_PASSWORD=postgres `
  -p 5432:5432 `
  -v pg-data:/var/lib/postgresql/data `
  postgres:17
```

Подожди ~5 секунд, пока Postgres поднимется (проверка: `docker exec lakehouse-pg pg_isready` должно вывести `accepting connections`).

**4.2. Миграции БД Lakekeeper** (одноразовая команда, создаёт таблицы каталога):

```powershell
docker run --rm `
  -e 'LAKEKEEPER__PG_ENCRYPTION_KEY=This-is-NOT-Secure!' `
  -e 'LAKEKEEPER__PG_DATABASE_URL_READ=postgresql://postgres:postgres@host.docker.internal:5432/postgres' `
  -e 'LAKEKEEPER__PG_DATABASE_URL_WRITE=postgresql://postgres:postgres@host.docker.internal:5432/postgres' `
  quay.io/lakekeeper/catalog:latest-main migrate
```

**4.3. Сам Lakekeeper** (REST-каталог на порту 8181):

```powershell
docker run -d --name lakekeeper `
  -e 'LAKEKEEPER__PG_ENCRYPTION_KEY=This-is-NOT-Secure!' `
  -e 'LAKEKEEPER__PG_DATABASE_URL_READ=postgresql://postgres:postgres@host.docker.internal:5432/postgres' `
  -e 'LAKEKEEPER__PG_DATABASE_URL_WRITE=postgresql://postgres:postgres@host.docker.internal:5432/postgres' `
  -p 8181:8181 `
  quay.io/lakekeeper/catalog:latest-main serve
```

Проверка: `curl.exe http://localhost:8181/health` должен вернуть `200`.

> ⚠️ **Почему `curl.exe`, а не `curl`?** В PowerShell `curl` — это алиас команды `Invoke-WebRequest`, у которой другие флаги (`-Method`, `-Body`, а не `-X`, `-d`). Поэтому во всех запросах используй `curl.exe` — это настоящий curl, который есть в Windows 10/11 из коробки.

**4.4. Принять Terms of Use и зарегистрировать warehouse `demo`** (один раз, через API).

Сначала создай в рабочей папке файл `create-warehouse.json` с таким содержимым (он описывает, в какой бакет MinIO и под какими кредами Lakekeeper будет писать данные). В терминале (выполняй из `C:\Users\твой_логин\lakehouse`):

```powershell
@'
{
  "warehouse-name": "demo",
  "project-id": "00000000-0000-0000-0000-000000000000",
  "storage-profile": {
    "type": "s3",
    "bucket": "warehouse",
    "key-prefix": "",
    "endpoint": "http://host.docker.internal:9000",
    "region": "us-east-1",
    "path-style-access": true,
    "flavor": "s3-compat",
    "sts-enabled": false
  },
  "storage-credential": {
    "type": "s3",
    "credential-type": "access-key",
    "access-key-id": "minioadmin",
    "secret-access-key": "minioadmin"
  }
}
'@ | Set-Content -Path create-warehouse.json -Encoding ascii
```

> Альтернатива: создай `create-warehouse.json` в VS Code (Explorer → New File → вставь содержимое → сохрани в `lakehouse\create-warehouse.json`).

Теперь прими Terms of Use (одноразовый шаг) и зарегистрируй warehouse:

```powershell
curl.exe -X POST http://localhost:8181/management/v1/bootstrap `
  -H "Content-Type: application/json" `
  -d '{"accept-terms-of-use": true}'

curl.exe -X POST http://localhost:8181/management/v1/warehouse `
  -H "Content-Type: application/json" `
  -d '@create-warehouse.json'
```

> - Первый запрос должен вернуть `204`. Если вместо этого `Catalog is not open for bootstrap` / `CatalogAlreadyBootstrapped` — это норм: Lakekeeper уже инициализирован (например, ты принял ToU через UI), просто пропусти этот шаг.
> - Второй запрос должен вернуть `201 Created` со `"status":"active"`. Если `Storage profile overlaps with existing warehouse demo` — warehouse `demo` уже существует, тоже норм, пропускай.
> - Если `curl.exe` пишет `Failed to open create-warehouse.json` — ты запускаешь его не из рабочей папки. Сначала `cd "$HOME\lakehouse"`.

---

## 5. Этап 3 — Trino (SQL-движок)

Trino узнаёт о каталогах из файлов в папке `/etc/trino/catalog/`. Мы смонтируем туда свою локальную папку `catalog\` — так ты сможешь сам класть в неё `.properties`-файлы и менять каталоги (добавил файл → перезапустил Trino → каталог появился).

**5.1. Создай папку `catalog\`.** В рабочей папке сделай подпапку `catalog` — в неё будем класть файлы каталогов:

```powershell
mkdir catalog -Force
```

**5.2. Создай файл каталога.** В `catalog\` положи файл `lakekeeper.properties` с таким содержимым (имя файла `lakekeeper` — это и есть имя каталога в Trino):

```powershell
@'
connector.name=iceberg
iceberg.catalog.type=rest
iceberg.rest-catalog.uri=http://host.docker.internal:8181/catalog
iceberg.rest-catalog.warehouse=demo
iceberg.rest-catalog.nested-namespace-enabled=true
iceberg.unique-table-location=true
fs.native-s3.enabled=true
s3.endpoint=http://host.docker.internal:9000
s3.path-style-access=true
s3.region=us-east-1
s3.aws-access-key=minioadmin
s3.aws-secret-key=minioadmin
'@ | Set-Content -Path catalog\lakekeeper.properties -Encoding ascii
```

> ⚠️ Ключевая строка — `fs.native-s3.enabled=true` (именно `fs.native-s3`, не `iceberg.s3` и не `fs.s3`). Без неё Trino падает на старте с ошибкой «неизвестные property». Файлы `.properties` и `.json` здесь чисто ASCII, поэтому `-Encoding ascii` исключает проблемы с кодировкой.

**5.3. Запусти Trino, смонтировав папку целиком** (из рабочей папки, где лежит `catalog\`):

```powershell
docker run -d --name trino `
  -p 8080:8080 `
  -v "${PWD}\catalog:/etc/trino/catalog" `
  trinodb/trino:476
```

> `${PWD}` — это текущая папка (аналог `$(pwd)` из bash). В итоге Docker смонтирует `C:\Users\твой_логин\lakehouse\catalog` в `/etc/trino/catalog`. Обратный слэш в пути хоста и прямой слэш в `/etc/trino/catalog` — это нормально, Docker Desktop сам разберётся.

Проверка: `curl.exe http://localhost:8080/v1/info` вернёт JSON (поле `"starting":false` значит «готов»); через ~20–30 секунд `docker ps` покажет Trino как `healthy`.

> 💡 **Поменял файл в `catalog\` или добавил новый `.properties`?** Пересоздавать контейнер не нужно — хватит перезапуска: `docker restart trino` (или кнопка **Restart** в Docker Desktop). Trino читает каталоги при старте. `docker rm -f` + `docker run` нужен только когда меняешь сам mount `-v` (другую папку).

---

## 6. Начать пользоваться: DBeaver

1. Скачай DBeaver: <https://dbeaver.io/download/> (Community Edition достаточно, ставь Windows-версию).
2. **Database → New Database Connection → Trino**.
3. Параметры:
   - Host: `localhost`, Port: `8080`
   - Database/Schema (catalog): `lakekeeper`
   - Username: `trino`, Password: пусто
   - JDBC URL получится: `jdbc:trino://localhost:8080/lakekeeper`
4. **Test Connection** → **Finish**.

Проверь связку, выполнив в SQL-редакторе DBeaver:

```sql
SHOW CATALOGS;                                  -- увидишь `lakekeeper`
CREATE SCHEMA lakekeeper.lakehouse;             -- создаём схему
CREATE TABLE lakekeeper.lakehouse.people (
    id   bigint,
    name varchar
);
INSERT INTO lakekeeper.lakehouse.people VALUES (1, 'alice'), (2, 'bob');
SELECT * FROM lakekeeper.lakehouse.people;      -- должны вернуться 2 строки
```

Если всё сработало — загляни в консоль MinIO (http://localhost:9001) → bucket `warehouse`. Там появятся папки вида `people-<uuid>/data/*.parquet` и `.../metadata/*.json` — это и есть Lakehouse: данные лежат parquet-файлами в объектном хранилище, а Iceberg-метаданные рядом.

---

## 7. Остановить и удалить всё

Полная очистка (контейнеры + данные + образы) — выполняй команды по порядку:

```powershell
# 1. Контейнеры (флаг -f останавливает и удаляет сразу)
docker rm -f trino lakekeeper lakehouse-pg lakehouse-minio

# 2. Данные (тома MinIO и Postgres)
docker volume rm minio-data pg-data

# 3. Образы (скачаются заново при следующем запуске)
docker rmi trinodb/trino:476 quay.io/lakekeeper/catalog:latest-main postgres:17 pgsty/mc:latest pgsty/silo:latest
```

Проверка, что всё удалилось:

```powershell
docker ps -a      # не должно быть trino / lakekeeper / lakehouse-*
docker volume ls  # не должно быть minio-data / pg-data
docker images     # не должно быть minio / lakekeeper / postgres:17 / trinodb
```

Чтобы **просто приостановить** (данные сохранятся, потом продолжишь с того же места) — `docker stop trino lakekeeper lakehouse-pg lakehouse-minio` без `rm`.

---

## 8. Если что-то пошло не так

- **Порт занят** (`port is already allocated`). Найди, кто слушает порт: `Get-NetTCPConnection -State Listen -LocalPort 5432`, и освободи порт или поменяй `-p 5432:5432` → `-p 5433:5432` (и тогда во всех `DATABASE_URL` `:5432` → `:5433`).
- **Lakekeeper: `Error creating read/write pool` / `pool timed out while waiting for an open connection`** — Lakekeeper не достучался до Postgres. Почти всегда это значит, что пропущен шаг 4.1 (запуск Postgres). Подними Postgres, дождись `docker exec lakehouse-pg pg_isready`, затем повтори `migrate` и `serve`.
- **`curl`: «не удаётся распознать параметр -X/-d»** — ты написал `curl` вместо `curl.exe`. В PowerShell используй `curl.exe`.
- **Команда «обрезается» или `is not recognized`** — после апострофа переноса `` ` `` стоит пробел. Убери его: `` ` `` должен быть последним символом строки.
- **`minio/minio: not found` / `pull access denied` / `No license is installed`** — официальные образы MinIO удалены с Docker Hub (`minio/minio`), закрыты для анонимных пуллов (`quay.io/minio/minio`) или требуют лицензии (`quay.io/minio/aistor/minio`). Используй свободный форк `pgsty/silo` (сервер) и `pgsty/mc` (клиент) — как в командах выше.
- **Trino упал на старте** — смотри логи: `docker logs trino`. Если ругается на `iceberg.s3.*` или `fs.s3.*` — в конфиге неверный префикс, нужен `fs.native-s3.enabled=true` + `s3.*`.
- **Trino не видит данные в MinIO** — проверь, что bucket называется именно `warehouse` и что warehouse `demo` зарегистрирован (повтори 4.4).
- **Docker не видит смонтированную папку `catalog\`** — убедись, что путь не содержит кириллицу/пробелы (если логин Windows с кириллицей — перенеси рабочую папку, например в `C:\Users\Public\lakehouse`).

---

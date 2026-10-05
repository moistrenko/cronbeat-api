# Спринт 0 — Окружение и скелет

**Цель спринта:** два приложения запускаются локально одной командой, лежат
в своих репозиториях на GitHub и имеют настроенные инструменты контроля качества.
Продуктовой логики в этом спринте нет вообще.

**Оценка:** 8–12 часов.

**Главная тема:** контейнеризация и структура проектов. К концу спринта ты должен
понимать, что происходит, когда ты пишешь `docker compose up`.

---

## Перед началом: что почитать

Не пытайся прочитать всё подряд. Ограничься таймбоксом в 2–3 часа и вернись
к материалам по ходу задач, когда упрёшься в конкретный вопрос.

Полный список по всем темам лежит в [docs/materials.md](../materials.md),
здесь только то, что нужно прямо сейчас.

**Видео по Docker, на русском, одно на выбор:**

- [Docker 2026: полный курс с практикой за час](https://www.youtube.com/watch?v=-yq-HVZIFEQ)
  — если хочешь быстро понять суть.
- [Docker полный курс от А до Я](https://www.youtube.com/watch?v=FUf4S1GSQJ0)
  — три часа, подробно про слои, тома, порты и сети.

Из курса нужны образы и контейнеры, тома, проброс портов, сети и команды
`docker compose`. Разделы про реестры образов и Kubernetes сейчас пропускай.

**Документация:**

- [Docker: Get started](https://docs.docker.com/get-started/) — ключевая мысль:
  образ это шаблон, контейнер это запущенный экземпляр, том (volume) это способ
  пережить перезапуск.
- [Composer: Basic usage](https://getcomposer.org/doc/01-basic-usage.md) — короткая
  страница. Проводи параллели с npm: `composer.json` ≈ `package.json`,
  `composer.lock` ≈ `package-lock.json`, `vendor/` ≈ `node_modules/`.
- [Laravel: Установка](https://laravel.su/docs/13.x/installation) и
  [Laravel Sail](https://laravel.su/docs/13.x/sail) — на русском, версия 13.x.
- [Структура каталогов Laravel](https://laravel.su/docs/13.x/structure) — просто
  пробеги глазами, чтобы не пугаться количества папок.

**Полезно, если PHP видишь впервые:**

- [PHP: Языковые конструкции](https://www.php.net/manual/ru/langref.php) —
  синтаксис бегло, документация PHP переведена на русский. Ты знаешь TypeScript,
  так что 90% покажется знакомым.
- [Laracasts: PHP for Beginners](https://laracasts.com/series/php-for-beginners-2023-edition)
  — английский, бесплатно, первых 10 уроков хватит.

**Не читай сейчас:** документацию по Eloquent, очередям, авторизации.
Всё это будет в своих спринтах, а сейчас только распылит внимание.

---

## Задача 0.1 — Установить инструменты

Выполняется в твоём терминале, кода нет.

```bash
brew install --cask orbstack
brew install php composer gh
gh auth login
```

OrbStack вместо Docker Desktop: на Apple Silicon заметно быстрее, меньше ест
память и не требует лицензии.

Важно: Docker Desktop и OrbStack не уживаются на одной машине. Docker Desktop
при каждом запуске забирает себе симлинк `/usr/local/bin/docker`, а OrbStack
добавляет свой каталог в конец `PATH` — в итоге побеждает Docker Desktop.
Если он установлен, его нужно снести целиком: приложение, симлинки
в `/usr/local/bin`, службу `com.docker.vmnetd` и данные в `~/Library`.

**Критерии приёмки:**

```bash
docker run --rm hello-world
php -v
composer -V
gh auth status
docker context ls
```

Ожидаем: контейнер запускается, PHP 8.3 или новее, `gh` авторизован,
активный контекст Docker — `orbstack`.

**Известная ловушка.** Локальный PHP и PHP внутри контейнера могут различаться
версиями. Composer разрешает зависимости под ту версию, которую видит локально,
а устанавливаться они будут в контейнер. Если версии разъедутся, получишь
невоспроизводимую сборку. После задачи 0.2 сверь `php -v` и `sail php -v`,
и при расхождении зафиксируй версию контейнера в `composer.json`:

```json
"config": {
    "platform": {
        "php": "8.4.0"
    }
}
```

---

## Задача 0.2 — Скелет бэкенда

Репозиторий уже создан и пуст, а `composer create-project` отказывается
работать в непустом каталоге, поэтому создаём во временной папке и переносим.

```bash
cd ~/Documents/learn/github
composer create-project laravel/laravel cronbeat-api-tmp
rsync -a cronbeat-api-tmp/ cronbeat-api/
rm -rf cronbeat-api-tmp

cd cronbeat-api
composer require laravel/sail --dev
php artisan sail:install --with=pgsql,redis
./vendor/bin/sail up -d
./vendor/bin/sail artisan migrate
```

Что здесь происходит по шагам:

- `composer create-project` скачивает шаблон приложения Laravel и его зависимости;
- `composer require laravel/sail --dev` ставит Sail. В Laravel 13 его нет
  в шаблоне: `php artisan sail:install` без этого шага падает с
  `There are no commands defined in the "sail" namespace`;
- `sail:install` генерирует `compose.yaml` (или `docker-compose.yml`)
  с контейнерами приложения, PostgreSQL и Redis — открой этот файл
  и прочитай его целиком, там всего страница текста, и это твой первый
  настоящий compose-файл;
- `sail up -d` поднимает контейнеры в фоне;
- `migrate` применяет миграции из коробки и заодно проверяет, что приложение
  действительно достучалось до базы.

Почему PostgreSQL, а не MySQL: у нас впереди работа со временем, интервалами
и JSON-полями, и Postgres в этом заметно сильнее. Плюс это де-факто стандарт
в новых проектах.

**Критерии приёмки:**

- `http://localhost` отдаёт стартовую страницу Laravel;
- `sail artisan migrate` отработал без ошибок;
- `docker ps` показывает три запущенных контейнера;
- `sail artisan test` — тесты из коробки зелёные.

**Вопрос на понимание** (ответь мне в чате своими словами): почему база данных
живёт в контейнере, но её данные не пропадают после `sail down`?

---

## Задача 0.3 — Скелет фронтенда

```bash
cd ~/Documents/learn/github
pnpm create vite cronbeat-web-tmp --template react-ts
rsync -a cronbeat-web-tmp/ cronbeat-web/
rm -rf cronbeat-web-tmp

cd cronbeat-web
pnpm install
pnpm dev
```

Важная особенность Vite, о которой спотыкаются все: он **не проверяет типы**.
Vite просто вырезает типы через esbuild ради скорости, поэтому ошибка типизации
не уронит дев-сервер. Проверка должна быть отдельным шагом. Убедись, что
в `package.json` в скрипте `build` есть `tsc -b` перед `vite build`, и добавь
отдельный скрипт:

```json
"typecheck": "tsc --noEmit"
```

**Критерии приёмки:**

- `pnpm dev` поднимает приложение на `http://localhost:5173`;
- `pnpm build` собирается без ошибок;
- `pnpm typecheck` проходит.

---

## Задача 0.4 — Инструменты качества кода

Настраиваем сразу, до первой строчки продуктового кода. Иначе потом будешь
править форматирование в тысяче файлов одним коммитом и потеряешь историю.

**Бэкенд.** Pint (форматтер, идёт в комплекте с Laravel) и Larastan
(статический анализатор, аналог строгого режима TypeScript для PHP):

```bash
./vendor/bin/sail composer require --dev larastan/larastan
```

Создай `phpstan.neon` в корне с уровнем анализа 5 (позже поднимем до 8) и добавь
в `composer.json` скрипты `lint` и `analyse`.

**Фронтенд.** ESLint в шаблоне Vite уже есть (flat config). Добавь Prettier
и настрой так, чтобы они не конфликтовали:

```bash
pnpm add -D prettier eslint-config-prettier
```

**Критерии приёмки:**

- `sail composer lint` и `sail composer analyse` проходят на чистом проекте;
- `pnpm lint` и `pnpm format` работают;
- в обоих репозиториях есть `.editorconfig` с одинаковыми правилами.

---

## Задача 0.5 — README и первый коммит

README в обоих репозиториях пишем **на английском** — репозитории публичные
и адресованы разработчикам со всего мира.

Минимальное содержание: что это за проект одним абзацем, требования к окружению,
как запустить локально, как прогнать тесты и линтеры.

Проверь `.gitignore`: в бэкенде не должно попасть `vendor/`, `.env`,
`storage/*.log`; во фронтенде — `node_modules/`, `dist/`, `.env.local`.
Файл `.env` в git не коммитим никогда, а `.env.example` — обязательно.

```bash
git add .
git commit -m "chore: bootstrap Laravel application with Sail"
git push -u origin main
```

**Критерии приёмки:**

- оба репозитория запушены на GitHub;
- в истории нет `vendor/`, `node_modules/`, `.env`;
- сообщения коммитов в формате Conventional Commits;
- по README посторонний человек может поднять проект с нуля.

---

## Самопроверка в конце спринта

Ответь себе честно, можешь ли ты объяснить без подглядывания:

1. Чем образ отличается от контейнера и зачем нужны тома.
2. Что делает `composer.lock` и почему его коммитят в репозиторий.
3. Куда попадает HTTP-запрос к Laravel и через что он проходит до контроллера.
4. Почему Vite не проверяет типы и где эта проверка должна происходить.
5. Почему `.env` не коммитят, а `.env.example` коммитят.

Если по какому-то пункту плаваешь — спроси меня, разберём.

---

## Что дальше

Спринт 1: проектирование базы данных, миграции, первая модель Eloquent
и REST API проверок с тестами на Pest.

Карта «синьорский фронт + JS/TS + паттерны + алгоритмы» лежит в
[curriculum.md](../curriculum.md). В спринте 0 это **не** основная работа.
Если останется час: замыкания в [learn.javascript.ru](https://learn.javascript.ru)
(главы про замыкания) и две easy на массивы на LeetCode. Если не останется —
спокойно переноси на спринт 1, продукт важнее.

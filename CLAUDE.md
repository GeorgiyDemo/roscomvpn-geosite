# CLAUDE.md

Этот файл — ориентировка для Claude Code (claude.ai/code) при работе с этим репозиторием.

## Назначение

Форк `hydraponique/roscomvpn-geosite`. Собирает `geosite.dat` (для V2Ray/Xray) и
рулсеты для **Mihomo (.mrs)** и **sing-box (.srs)** из текстовых списков доменов
в `data/`. Релизы публикуются автоматически через GitHub Actions.

Origin: `https://github.com/GeorgiyDemo/roscomvpn-geosite.git`
Upstream: `https://github.com/hydraponique/roscomvpn-geosite.git`

## Структура

```
data/                                — исходники: текстовые списки доменов (v2fly формат)
buildtools/deduplicate.py            — ручная утилита дедупликации через DNS-резолвинг + GeoIP
release/                             — generated артефакты (коммитятся CI, не вручную!)
  ├── geosite.dat                    — Xray/V2Ray бинарь
  ├── geosite.dat.sha256
  ├── mihomo/*.mrs                   — рулсеты Mihomo
  ├── sing-box/*.srs                 — рулсеты sing-box
  ├── mihomo.tar.gz                  — архив всех .mrs
  └── sing-box.tar.gz                — архив всех .srs
.github/workflows/
  ├── build.yml                      — основной билд (push в master/test → release)
  ├── sync-upstream.yml              — daily 04:17 UTC fast-forward с upstream
  └── sync-v2fly-ip-checkers.yml     — daily 05:43 UTC sync category-ip-geo-detect
README.md                            — публичная документация (категории, ссылки на CDN)
LICENSE                              — MIT
```

## Категории в `data/`

Формат — v2fly domain-list (`domain:`, `full:`, `keyword:`, `regexp:`, `include:`).
Полный список и назначение каждой категории — в `README.md`. Кратко:

- **RU**: `whitelist`, `category-ru`, `category-geoblock-ru`
- **Сервисы**: `apple`, `google-play`, `google-deepmind`, `microsoft`, `github`,
  `telegram`, `youtube`, `twitch`, `twitch-ads`, `pinterest`
- **Игры**: `steam`, `epicgames`, `riot`, `escapefromtarkov`, `faceit`, `origin`
- **Прочее**: `category-ads`, `win-spy`, `private`, `torrent`
- **Авто-синк с v2fly**: `category-ip-geo-detect-upstream` (НЕ ПРАВИТЬ ВРУЧНУЮ)
- **Локальная версия**: `category-ip-geo-detect` (можно править)

## Пайплайн билда (`.github/workflows/build.yml`)

Триггер: push в `master`/`test` (paths-ignore: README, .gitignore, LICENSE, release/).

1. Чекаут + `v2fly/domain-list-community` в `community/`.
2. `go run ./` в community c подменёнными `data/` → `geosite.dat`.
3. Установка Mihomo (последний stable .deb) и sing-box v1.12.12.
4. **Конвертация .mrs**: awk нормализует каждый файл `data/` (срезает комменты,
   `domain:`/`full:`/`keyword:` → префиксы Mihomo `+.host` / точное / wildcard),
   затем `mihomo convert-ruleset domain text $tmp $dest`.
5. **Конвертация .srs**: awk → tab-разделённый промежуточный формат → `jq`
   собирает JSON рулсета (version 3, domain/domain_suffix/domain_keyword/domain_regex),
   затем `sing-box rule-set compile`.
6. На master: пушит `release/` отдельной веткой `release` (force), коммитит
   `release/` в master, создаёт тег `YYYYMMDDHHMM`, публикует GitHub Release,
   purge jsDelivr CDN, шлёт `repository-dispatch` в `hydraponique/roscomvpn-routing`.
7. На test: аналогично, но в ветку `test-release` и без релиза/диспатча.

## Sync-воркфлоу

- **sync-upstream.yml**: каждый день 04:17 UTC. `git merge -X theirs upstream/master`.
  `release/` всегда берётся с upstream при конфликте — наш билд всё равно его перегенерит.
- **sync-v2fly-ip-checkers.yml**: каждый день 05:43 UTC. Скачивает
  `data/category-ip-geo-detect` из `v2fly/domain-list-community` (рекурсивно
  раскрывая `include:`), флаттенит в `data/category-ip-geo-detect-upstream`,
  коммитит и пушит если изменилось. Это триггерит основной билд.

## Правила работы с репо

1. **НЕ редактировать `release/` вручную** — он перегенерируется каждым билдом.
   Если хочется обновить артефакты — пушни любое изменение в `data/`/коде.
2. **НЕ редактировать `data/category-ip-geo-detect-upstream`** — авто-синк его
   перезапишет. Для локальных правок используй `data/category-ip-geo-detect`.
3. **Билд триггерится `data/**`/workflow-изменениями**, не README/.gitignore/LICENSE.
4. **Версии тулзов**: Mihomo берётся «latest stable» через GitHub API (нужен
   `GITHUB_TOKEN`), sing-box захардкожен на `1.12.12`. Менять — в `build.yml`.
5. **Конфликт с upstream**: `sync-upstream` использует `-X theirs`. Если форку
   нужна локальная правка чего-то, что апстрим тоже трогает — будь готов к тому,
   что её затрёт.
6. **buildtools/deduplicate.py** — ручной инструмент, не используется в CI.
   Запускается локально для чистки категорий от доменов, чьи IP уже в geoip:direct.

## Известные грабли (история коммитов)

- `Fix Mihomo install: pass GITHUB_TOKEN to API call` — без токена api.github.com
  ратится 60 req/h и падает.
- `Make CDN purge steps non-fatal` / `Make routing dispatch non-fatal` —
  jsDelivr и routing-dispatch не должны валить весь билд при сбое.
- `Fix failing sync workflows` — sync-воркфлоу падали из-за конфликтов в release/.

## Git workflow

- Origin = форк, upstream = `hydraponique/roscomvpn-geosite`.
- `master` — рабочая ветка, она же синкается с upstream.
- `release` (форс-пушится из CI), `test`, `test-release` — служебные.
- Теги формата `YYYYMMDDHHMM` создаются CI при каждом master-билде.

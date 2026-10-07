# Конфигурация Space Engineers Dedicated (Docker) — справочник

Документ описывает назначение, приоритеты и редактирование всех конфиг-файлов
этой установки выделенного сервера Space Engineers. Все пути указаны относительно
корня проекта.

---

## Карта файлов

| Файл | Роль | Кто пишет | Кто читает |
|---|---|---|---|
| `appdata/space-engineers/config/SpaceEngineers-Dedicated.cfg` | Настройки выделенного сервера | вы (вручную) + entrypoint (`LoadWorld`) | сервер при старте |
| `appdata/space-engineers/config/World/Sandbox_config.sbc` | **Мировые настройки сессии** (перекрывают `.cfg`) | сервер при сохранении + entrypoint (`Mods`) | сервер при старте |
| `appdata/space-engineers/config/World/Sandbox.sbc` | Чекпоинт-сейв мира (корабли, игроки, фракции) | сервер при автосейве | сервер при загрузке |
| `appdata/space-engineers/config/World/SANDBOX_0_0_0_.sbs` | Данные сектора (часть сейва) | сервер | сервер |
| `appdata/space-engineers/config/World/Backup/<метка>/` | Авто-снимки мира (`MaxBackupSaves`) | сервер | вы, при восстановлении |
| `appdata/space-engineers/config/mods.txt` | ID модов из Steam Workshop | вы | entrypoint |
| `.env` | Переменные окружения контейнера | вы | docker compose |

> **Ключевое правило приоритета:** настройки игры (`<SessionSettings>`) берутся из
> мирового файла `Sandbox_config.sbc` и **перекрывают** одноимённые настройки из
> `SpaceEngineers-Dedicated.cfg`. В логах это видно как:
> `Sandbox world configuration file found, overriding checkpoint settings.`
>
> Поэтому игровые параметры меняются в `Sandbox_config.sbc`, а не в `.cfg`.

---

## 1. `SpaceEngineers-Dedicated.cfg` — серверный конфиг

### Серверные поля

| Поле | Пример/значение | Описание |
|---|---|---|
| `<LoadWorld>` | `Z:\appdata\space-engineers\World` | Путь к папке мира. Entrypoint подставляет это значение при каждом старте. |
| `<IP>` | `0.0.0.0` | Интерфейс прослушивания. |
| `<SteamPort>` | `8766` | UDP-порт Steam (matchmaking/P2P). |
| `<ServerPort>` | `27016` | UDP-порт игрового трафика. |
| `<Administrators />` | — | ⚠️ Steam64ID админов через запятую. **Сейчас пусто** — без этого нет админ-прав в игре. |
| `<Banned />` | — | Бан-лист. |
| `<GroupID>` | `0` | Привязка к группе Steam. |
| `<ServerName>` | `Space Dumb Engineers` | Имя сервера в списке Steam. |
| `<WorldName>` | `Star System` | Отображаемое имя мира. |
| `<PauseGameWhenEmpty>` | `true` | Пустой мир ставится на паузу (не симулируется и не сохраняется). |
| `<AutoRestartEnabled>` | `true` | Авторестарт по таймеру. |
| `<AutoRestatTimeInMin>` | `720` | Интервал авторестарта, мин (опечатка «AutoRestat» — так в самой игре). |
| `<AutoRestartSave>` | `true` | Сохранять мир перед авторестартом. |
| `<AutoUpdateEnabled>` | `true` | Сервер сам обновляет Space Engineers через Steam. |
| `<IgnoreLastSession>` | `true` | Не восстанавливать «последнюю сессию» при старте (защита после аварийного останова). |
| `<RemoteApiEnabled>` | `true` | Включить Remote API для внешних инструментов. |
| `<RemoteApiPort>` | `8080` | Порт Remote API (на хосте доступен как `8081` → `8080`). |
| `<NetworkType>` | `steam` | Тип сети (Steam P2P). |
| `<OnlineMode>` | `PUBLIC` | Видимость сервера. |
| `<DedicatedId>` | — | Уникальный ID сервера (автогенерация, не трогать). |

### `<SessionSettings>` внутри `.cfg`

Эти значения **в большинстве своём перекрываются** файлом мира. Реальные действующие
значения см. в `Sandbox_config.sbc`. Примеры расхождений:

| Параметр | В `.cfg` | В мире (действует) |
|---|---|---|
| `MaxPlayers` | 16 | 10 |
| `AutoSaveInMinutes` | 30 | 5 |
| `MaxBackupSaves` | 5 | 10 |
| `EnableIngameScripts` | true | false |
| `EnableWolfs` / `EnableSpiders` | true | false |
| `TotalPCU` | 320000 | 1000000 |
| `BlockLimitsEnabled` | PER_PLAYER | GLOBALLY |

---

## 2. `World/Sandbox_config.sbc` — мировой конфиг (главный для настроек)

Это **настройки конкретной сессии мира**. Файл перекрывает `<SessionSettings>` из `.cfg`.

### Структура

```xml
<MyObjectBuilder_WorldConfiguration>
  <Settings xsi:type="MyObjectBuilder_SessionSettings">…</Settings>  <!-- действующие настройки -->
  <Mods />              <!-- моды, вставляет entrypoint из mods.txt при каждом старте -->
  <SessionName>…</SessionName>   <!-- внутреннее имя сессии (не путать с WorldName) -->
  <LastSaveTime>…</LastSaveTime> <!-- время последнего сохранения -->
</MyObjectBuilder_WorldConfiguration>
```

### Ключевые настройки `<Settings>` (актуальные значения)

- **Базовое**: `GameMode=Survival`, `OnlineMode=PUBLIC`, `MaxPlayers=10`.
- **Множители**: `InventorySizeMultiplier=3`, `AssemblerSpeedMultiplier=3`,
  `AssemblerEfficiencyMultiplier=3`, `RefinerySpeedMultiplier=3`,
  `WelderSpeedMultiplier=2`, `GrinderSpeedMultiplier=2`.
- **PCU/лимиты**: `TotalPCU=1000000`, `PiratePCU=500000`, `GlobalEncounterPCU=25000`,
  `BlockLimitsEnabled=GLOBALLY`, `MaxGridSize=0`, `MaxBlocksPerPlayer=0`.
- **Сохранение**: `AutoSaveInMinutes=5`, `MaxBackupSaves=10`, `EnableSaving=true`.
- **Скрипты**: ⚠️ `EnableIngameScripts=false` — программируемые блоки отключены;
  `EnableScripterRole=false`.
- **Среда**: `EnvironmentHostility=SAFE`, `EnableWolfs=false`, `EnableSpiders=false`,
  `ScrapEnabled=false`, `RandomizeSeed=false`.
- **Физика/безопасность**: `EnableShareInertiaTensor=false`,
  `EnableUnsafePistonImpulses=false`, `EnableUnsafeRotorTorques=false`.
- **Экономика**: `EnableEconomy=true`, `TradeFactionsCount=15`,
  `StationsDistanceInnerRadius=10000000`, `EconomyTickInSeconds=1200`,
  `DepositsCountCoefficient=1`, `DepositSizeDenominator=60`.
- **Бой/NPC**: `EnableFriendlyFire=true`, `WeaponsEnabled=true`,
  `CargoShipsEnabled=true`, `EnableDrones=true`, `MaxDrones=5`,
  `EnableEncounters=true`, `EnablePlanetaryEncounters=true`.
- **Обслуживание**: `TrashRemovalEnabled=true`, `TrashFlagsValue=7706`,
  `StopGridsPeriodMin=15` (сборщик мусора).

### Как править

```bash
./seserver stop      # 1. обязательно остановить — иначе сервер перезапишет файл
# 2. отредактировать XML (например, nano appdata/space-engineers/config/World/Sandbox_config.sbc)
./seserver start     # 3. запустить; entrypoint тронет только элемент <Mods>
```

---

## 3. `World/Sandbox.sbc` — чекпоинт-сейв

Вся суть мира: корабли, базы, персонажи, фракции, астероиды. Размер растёт по мере
игры (примерно 8 МБ). **Не редактировать вручную.** Восстановление — из `Backup/`
(см. ниже) или полное удаление для старта заново (сервер воссоздаст мир из шаблона).

## 4. `World/SANDBOX_0_0_0_.sbs` — данные сектора

Часть сейва. Файлы `.sbsB1..B5` рядом — внутренний ротируемый бэкап чекпоинта.

---

## 5. `World/Backup/` — авто-бэкапы мира и восстановление

Полные снимки мира создаются при каждом автосейве; хранится `MaxBackupSaves` штук
(сейчас 10). Метки каталогов вида `2026-10-05 172706` (дата + время HHMMSS).

Посмотреть список точек:

```bash
ls -1 appdata/space-engineers/config/World/Backup/
```

Восстановить нужный снимок:

```bash
cd space-engineers-dedicated-docker-linux

./seserver stop   # 1. обязательно: работающий сервер перезапишет файлы при сейве

# 2. отодвинуть текущий (живой) мир в сторону — не удалять
mv appdata/space-engineers/config/World appdata/space-engineers/config/World-live-backup

# 3. развернуть нужный снимок как новый мир (пример — момент 17:07:06)
cp -a "appdata/space-engineers/config/World-live-backup/Backup/2026-10-05 170706" \
      appdata/space-engineers/config/World

# 4. проверить ключевые файлы и запустить
ls appdata/space-engineers/config/World/
./seserver start
```

Нюансы:
- Прогресс после выбранного снимка остаётся в отодвинутой папке `World-live-backup` —
  можно вернуться тем же способом.
- После старта entrypoint заново встроит моды в `Sandbox.sbc`/`Sandbox_config.sbc`
  (это нормально, содержимое мира не затрагивается).
- Права на восстановленных файлах выставляет `./seserver start` (`chmod a+rwX`).

---

## 6. `mods.txt` — моды Workshop

Один Steam Workshop ID на строку, `#` — комментарии:

```
# пример: 1902970975
1902970975
```

При каждом старте entrypoint встраивает эти ID в `<Mods>` файлов `Sandbox.sbc` и
`Sandbox_config.sbc` (ручные правки элемента `<Mods>` будут затёрты). Изменение
`mods.txt` требует только перезапуска сервера.

---

## 7. `.env` — переменные окружения контейнера

| Переменная | Значение | Описание |
|---|---|---|
| `SKIP_UPDATE` | `0` / `1` | `1` — пропустить проверку/скачивание steamcmd при старте (быстрее рестарт). Рекомендуется после первого успешного запуска. |
| `DISCORD_WEBHOOK_URL` | URL | Вебхук Discord для уведомлений (сервер готов, авторестарт, онлайн). |

Файл опционален (`required: false`). Шаблон — [`.env.example`](.env.example).

---

## 8. Проброс портов (см. [`docker-compose.yml`](docker-compose.yml))

| Хост | Контейнер | Назначение |
|---|---|---|
| `27016/udp` | `27016/udp` | Игровой трафик |
| `8766/udp` | `8766/udp` | Steam (P2P / matchmaking) |
| `8081/tcp` | `8080/tcp` | Remote API |

---

## Чек-лист типовых правок

1. **Администраторы**: впишите Steam64ID в `<Administrators>` в
   `SpaceEngineers-Dedicated.cfg` — сейчас поле пустое, админ-меню в игре недоступно.
2. **Игровые настройки** (макс. игроков, PCU, автосейв, волки/пауки, скрипты,
   экономика) меняйте в `World/Sandbox_config.sbc`, а **не** в `.cfg`.
3. **Программируемые блоки**: `EnableIngameScripts=false` — включите, если нужны PB.
4. **Ускорение рестартов**: после стабильной работы добавьте `SKIP_UPDATE=1` в `.env`.
5. **Бэкапы**: периодически копируйте папку `appdata/space-engineers/config/World`
   на другой диск (например, cron + `tar`) — авто-снимки `Backup/` живут рядом с
   миром и погибнут вместе с ним при потере диска.
# P2F Network — MVP (Шард 0)

Реализация Phase 1 технического задания P2F Network: блокчейн **Шард 0**
(`p2f-shard-0`) на Cosmos SDK v0.50 + CometBFT v0.38 с тремя кастомными модулями.

Всё лежит на диске E, в каталоге `E:\p2f-network`.

---

## Что уже есть

| Компонент | Состояние |
|---|---|
| Портируемый тулчейн (Go 1.23.12, git, protoc) | работает, всё в `E:\p2f-network\.toolchain` |
| Генерация protobuf (свой protoc-плагин на gogoproto) | работает, `scripts\gen-proto.ps1` |
| Приложение Cosmos SDK с 15 стандартными модулями | собирается, генезис формируется |
| `x/p2f/roles` — роли участников | типы, keeper, Msg, Query, CLI |
| `x/p2f/wallet` — B2C/B2B кошельки и политики | типы, keeper, Msg, Query, CLI, hook банка |
| `x/p2f/spawn` — реестр шардов и whitelist операторов | типы, keeper, Msg, Query, CLI, депозиты |
| Демон `p2fd` | собирается (103 МБ), `--help` работает |
| `localnet.ps1 -Action init` | собирает генезис Шарда 0 на 4 валидатора |
| `localnet.ps1 -Action start` | 4 узла поднимаются, соединяются (по 3 пира), сходятся на генезисе |
| Производство блоков | **не подтверждено**: узлы расходятся в app hash блока 2, см. ниже |
| Приёмочные тесты (`chain\tests`) | в работе |
| Control Chain (Phase 2) | спецификация готова: `docs\CONTROL-CHAIN-ПЛАН.md` |

### Известная проблема

Узлы сходятся на генезисе и производят блок 1, но расходятся в app hash блока 2,
поэтому консенсус не идёт дальше высоты 1. Это недетерминизм на уровне приложения:

```
ERR prevote step: consensus deems this block invalid; prevoting nil
  err="wrong Block.Header.AppHash. Expected ..., got ..."
```

Идёт поиск источника (проба `chain\_probe_det`). Пока это не исправлено,
утверждать, что Шард 0 «работает», нельзя — работает сборка, генезис и запуск сети.


## Требования к окружению

Ничего устанавливать в систему не нужно: Go, git и protoc лежат в
`E:\p2f-network\.toolchain`. Перед любой работой подключите окружение:

```powershell
. E:\p2f-network\scripts\env.ps1
```

Скрипт выставляет `GOROOT`, `GOPATH`, `GOMODCACHE`, `GOCACHE`, `GOBIN` и `PATH`
так, чтобы кеши и бинарники оставались на диске E.

## Основные команды

```powershell
# окружение
. E:\p2f-network\scripts\env.ps1

# перегенерировать Go-код из .proto
& E:\p2f-network\scripts\gen-proto.ps1

# собрать демон
Set-Location E:\p2f-network\chain
go build -o E:\p2f-network\bin\p2fd.exe .\cmd\p2fd

# модульные тесты
go test ./...

# поднять локальную сеть из 4 валидаторов
& E:\p2f-network\scripts\localnet.ps1 -Action start

# остановить
& E:\p2f-network\scripts\localnet.ps1 -Action stop

# статус
& E:\p2f-network\scripts\localnet.ps1 -Action status
```

## Параметры сети

| Параметр | Значение |
|---|---|
| Chain ID | `p2f-shard-0` |
| Базовая деноминация | `up2f` (1 P2F = 1 000 000 up2f) |
| Префикс адресов | `p2f` |
| Максимум валидаторов | 100 |
| Unbonding time | 21 день |
| Slash за double sign | 5 % |
| Slash за downtime | 0,01 % |
| Окно подписей | 10 000 блоков |
| Депозит governance | 10 P2F |
| Голосование | 48 часов |
| Целевой блок | 1 с |

## Кастомные модули

### `x/p2f/roles`

Реестр ролей участника: `loader`, `pricer`, `replica`, `validator`, `operator`, `dao`.

```powershell
p2fd query p2froles roles p2f1...
p2fd query p2froles members validator
p2fd query p2froles params

# роль выдаёт только governance
p2fd tx p2froles assign p2f1... validator --from <gov-account>
# открытую роль можно занять самому
p2fd tx p2froles self-register loader --from <account>
```

Роли **не дают** voting power и приоритета в консенсусе — это принцип из ТЗ.

### `x/p2f/wallet`

Два класса кошельков: B2C (потребительский) и B2B (кошелёк приложения).
Политики ограничивают исходящие переводы: allowlist/denylist, лимит на перевод,
суточный лимит, срок действия, session key.

```powershell
p2fd tx p2fwallet create b2b "my-app" --policies-file policies.json --from <account>
p2fd query p2fwallet wallet p2f1...
p2fd query p2fwallet wallets --owner p2f1...
```

Принуждение работает через `SendRestrictionFn` банка: **любой** исходящий перевод
проверяется по политикам отправителя до исполнения, включая переводы, инициированные
другими модулями. Нарушение политики отменяет всю транзакцию.

Пример `policies.json`:

```json
[
  {
    "policy_id": "payouts",
    "allow_list": ["p2f1recipient..."],
    "deny_list": [],
    "single_limit": "10000000",
    "daily_limit": "100000000"
  }
]
```

### `x/p2f/spawn`

MVP-Spawn из раздела 4.2 ТЗ: whitelist операторов, ручное одобрение, лимит VPS,
cooldown, депозит, слэшинг.

```powershell
p2fd query p2fspawn operators
p2fd query p2fspawn requests --status pending
p2fd query p2fspawn shards
p2fd query p2fspawn params

# оператора добавляет governance
p2fd tx p2fspawn register-operator p2f1operator... --region eu-central --from <gov-account>
# оператор просит шард (депозит уходит в модульный аккаунт)
p2fd tx p2fspawn request 1 --region eu-central --from <operator>
# одобрение, затем регистрация шарда
p2fd tx p2fspawn approve spawn-1 --from <gov-account>
p2fd tx p2fspawn complete spawn-1 p2f-shard-1 --from <gov-account>
```

В генезисе `spawn.enabled = false`, как требует ТЗ: до явного включения через
governance протокол создания шардов не работает.

**Чего здесь намеренно нет:** VRF, Proof-of-Personhood, Google Cloud Attestation,
Oracle-верификация. Это фаза 6. Заглушка вместо VRF или аттестации была бы хуже,
чем честный ручной whitelist.

## Структура репозитория

```
E:\p2f-network\
├── chain\                     Go-модуль github.com/p2f-network/p2f
│   ├── app\                   сборка приложения, конфигурация модулей
│   ├── cmd\p2fd\              демон и CLI
│   ├── proto\p2f\...          protobuf-схемы
│   ├── x\p2f\roles\           модуль ролей
│   ├── x\p2f\wallet\          модуль кошельков
│   ├── x\p2f\spawn\           модуль spawn
│   └── tests\                 приёмочные тесты
├── docs\                      план MVP и документация
├── scripts\                   env, генерация proto, локальная сеть
├── bin\                       собранные бинарники
└── .toolchain\                Go, git, protoc, плагин, кеши модулей
```

## Границы MVP

Подробности — в `docs\MVP-ПЛАН.md`. Коротко: это Phase 1 ТЗ.
Control Chain, IBC между шардами, DA-слой, threshold-мемпул, AMM, WASM/EVM
исполнение и MPC-подписи в этот этап не входят и заявлены как следующие шаги.

Заявления о производительности (10 000 TPS на шард, 1 000 000 TPS суммарно)
в этом репозитории **не подтверждены измерениями** и не должны использоваться
как обещание до публикации воспроизводимого бенчмарка.

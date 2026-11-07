# Структура репозитория

Репозиторий создан в соответствии со статьёй: https://medium.com/@benbjohnson/standard-package-layout-7cdbc8391fc1

## Основные принципы статьи

1. Корневая папка должна содержать доменные типы (`User`) и не иметь каких-либо зависимостей. В такую папку
   можно также добавлять доменные интерфейсы (`UserService`), но только если они так же зависят только от доменных типов.
   Например, не вызывают сторонние сервисы или не взаимодействуют с базой данных, так как это уже детали реализации.
2. Группировка пакетов происходит по зависимостям, а не по слоям (сервис/контроллер) и не по модулям. Каждая технология
   живёт в своём пакете-адаптере (`postgres`, `http`, `stripe`). Адаптер реализует доменный интерфейс, и компоненты общаются
   друг с другом только через доменные интерфейсы.
3. Общая логика моков: так как все связи созданы через интерфейсы, их легко подменить моками в тестах.
4. Пакет `main` связывает зависимости. Лежит в `cmd/имя/main.go`, создаёт все компоненты и передаёт их друг другу.
   Логики в нём нет.

## Три подхода, которые он не советует

- **Всё в одном пакете** — нормально для маленьких проектов, но с ростом в коде трудно ориентироваться;
- **Пакеты по типу кода** (`handlers`, `models`) — плохие имена вроде `controller.UserController` и циклические зависимости;
- **Пакеты по модулям** (`users`, `accounts`) — те же проблемы, имена вроде `users.User`.


----
## Чистая архитектура по DDD
в соотвествии с Johnson, придерживаясь принципов
1. Домен ни от чего не зависит	domain/ не импортирует базы, HTTP и другие пакеты
2. Подпакеты по зависимостям	adapters/postgres, adapters/grpc, adapters/kafka — каждая технология отдельно
3. Моки через интерфейсы	интерфейсы в domain (или в app) легко подменить в тестах
4. main связывает всё	cmd/identity/main.go собирает зависимости и ничего больше
```
chirp/
├── go.mod
├── Makefile
├── README.md
│
├── api/                          — контракты (пишутся до кода)
│   ├── openapi/
│   │   └── gateway.yaml
│   └── proto/
│       ├── identity/v1/identity.proto
│       ├── chats/v1/chats.proto
│       └── messaging/v1/messaging.proto
│
├── cmd/                          — точки входа, только main.go
│   ├── gateway/main.go
│   ├── identity/main.go
│   ├── chats/main.go
│   └── messaging/main.go
│
├── internal/
│   ├── identity/
│   │   ├── domain/               — сущности и правила
│   │   │   ├── user.go           (User, NewUser)
│   │   │   ├── email.go          (value object Email)
│   │   │   ├── errors.go         (ErrUserNotFound…)
│   │   │   └── repository.go     (интерфейс UserRepository)
│   │   ├── app/                  — сценарии
│   │   │   ├── register.go
│   │   │   ├── login.go
│   │   │   └── update_profile.go
│   │   └── adapters/             — технологии
│   │       ├── postgres/user_repository.go
│   │       └── grpc/server.go
│   │
│   ├── chats/
│   │   ├── domain/               (Chat, Member, ChatType)
│   │   ├── app/                  (CreateChat, AddMember…)
│   │   └── adapters/
│   │       ├── postgres/
│   │       ├── outbox/           (воркер публикации событий)
│   │       └── grpc/
│   │
│   ├── messaging/
│   │   ├── domain/               (Message, MessageSent)
│   │   ├── app/                  (SendMessage, GetHistory)
│   │   └── adapters/
│   │       ├── cassandra/
│   │       ├── kafka/
│   │       └── grpc/
│   │
│   ├── gateway/                  — без домена, только транспорт
│   │   ├── http/                 (обработчики REST)
│   │   ├── ws/                   (WebSocket-хаб)
│   │   └── clients/              (gRPC-клиенты к сервисам)
│   │
│   └── platform/                 — общая инфраструктура
│       ├── config/
│       ├── logger/
│       ├── postgres/             (создание пула pgx)
│       └── auth/                 (JWT)
│
├── gen/                          — сгенерированный код, руками не править
│   ├── openapi/
│   └── proto/
│
├── migrations/
│   ├── identity/
│   └── chats/
│
├── deploy/
│   ├── docker-compose.yml
│   └── docker/                   (Dockerfile для каждого сервиса)
│
├── test/                         — e2e и нагрузочные тесты
│   ├── e2e/
│   └── load/                     (сценарии k6, ghz)
│
└── docs/
├── architecture.md           (оценка нагрузки, схема)
└── adr/
```

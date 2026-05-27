## Практическая работа №11. Вуйко Ярослава, ЭФМО-01-25
### Создание GraphQL API с использованием gqlgen. Запросы и мутации. 27.05.2026

## Схема GraphQL (schema.graphqls)

Схема описывает типы данных и доступные операции. Она расположена в файле `graph/schema.graphqls`.

```graphql
type Task {
    id: ID!
    title: String!
    description: String
    dueDate: String
    done: Boolean!
    createdAt: String!
    updatedAt: String!
}

input CreateTaskInput {
    title: String!
    description: String
    dueDate: String
}

input UpdateTaskInput {
    title: String
    description: String
    dueDate: String
    done: Boolean
}

type Query {
    tasks: [Task!]!
    task(id: ID!): Task
}

type Mutation {
    createTask(input: CreateTaskInput!): Task!
    updateTask(id: ID!, input: UpdateTaskInput!): Task!
    deleteTask(id: ID!): Boolean!
}
```

### Пояснение типов и операций

| Тип/Операция | Назначение |
|--------------|------------|
| `Task` | Основной тип, описывающий задачу. Все поля обязательны, кроме `description` и `dueDate` |
| `CreateTaskInput` | Входной тип для создания задачи. `title` — обязательное поле |
| `UpdateTaskInput` | Входной тип для обновления. Все поля опциональны — обновляются только переданные |
| `Query.tasks` | Возвращает список всех задач |
| `Query.task(id)` | Возвращает задачу по ID или `null`, если не найдена |
| `Mutation.createTask` | Создаёт новую задачу и возвращает её |
| `Mutation.updateTask` | Обновляет задачу. Возвращает обновлённую задачу |
| `Mutation.deleteTask` | Удаляет задачу. Возвращает `true` при успехе, `false` если задача не найдена |


### Структура нового сервиса
```
.
│   Dockerfile
│   go.mod
│   go.sum
│   gqlgen.yml
│   
├───cmd
│   └───graphql
│           main.go
│           
├───graph
│   │   generated.go
│   │   resolver.go
│   │   schema.graphqls
│   │   schema.resolvers.go
│   │   
│   └───model
│           models_gen.go
│           
└───internal
    ├───repository
    │       task_repository.go
    │       
    └───service
            task_service.go
```

## Резолверы и связь с данными

Резолверы находятся в файле graph/schema.resolvers.go, который генерируется автоматически и затем дополняется вручную. В этом файле каждая функция-резолвер реализует бизнес-логику для соответствующего поля схемы — например, Tasks(), Task(), createTask() и так далее. Резолверы не обращаются к базе данных напрямую, а вызывают методы сервисного слоя TaskService, который инжектируется через структуру Resolver, определённую в graph/resolver.go. Таким образом, связь с данными выстроена по цепочке: резолвер получает запрос, вызывает нужный метод сервиса, тот обращается к репозиторию, а репозиторий выполняет SQL-запросы к PostgreSQL. Такая архитектура позволяет переиспользовать сервисный слой между GraphQL и REST API, а также упрощает тестирование за счёт чёткого разделения ответственности.

```
package graph

import (
	"graphql-service/internal/service"
)

type Resolver struct {
	TaskService *service.TaskService
}

func NewResolver(taskService *service.TaskService) *Resolver {
	return &Resolver{
		TaskService: taskService,
	}
}

func (r *Resolver) Mutation() MutationResolver {
	return &mutationResolver{r}
}

func (r *Resolver) Query() QueryResolver {
	return &queryResolver{r}
}

type mutationResolver struct {
	*Resolver
}

type queryResolver struct {
	*Resolver
}

```


### Запуск

```bash
cd services/graphql
export GRAPHQL_PORT=8090
export DB_URL=postgres://postgres:postgres@localhost:5433/tasks_db?sslmode=disable
go run ./cmd/graphql
```


## Примеры запросов и ответов

### Создать задачу (Mutation)
<img width="1365" height="840" alt="2026-05-27_19-23-36" src="https://github.com/user-attachments/assets/9de773d6-288e-4b7b-b8ba-fd4874047740" />

### Получить список всех задач (Query)
<img width="1367" height="840" alt="2026-05-27_19-23-54" src="https://github.com/user-attachments/assets/e37c25bd-bdf9-4042-9626-3cddaa0531a6" />

### Получить задачу по ID (Query с переменными)
<img width="1382" height="887" alt="2026-05-27_19-23-45" src="https://github.com/user-attachments/assets/eb0f4b4c-a0d8-4059-bfdb-ea9af4c5ce13" />

### Обновить задачу (Mutation)
<img width="1359" height="842" alt="2026-05-27_19-24-02" src="https://github.com/user-attachments/assets/c7b24878-c6e0-400a-94a0-94ab4d7b53a1" />

### Удалить задачу (Mutation)
<img width="1384" height="709" alt="2026-05-27_19-24-37" src="https://github.com/user-attachments/assets/1b59e8d4-a404-4257-adbf-801f1eeb43a8" />



### Контрольные вопросы

1. В чём отличие Query и Mutation?
Query используется для чтения данных и не должен изменять состояние сервера. Mutation предназначен для изменения данных (создание, обновление, удаление). Мутации выполняются последовательно, а запросы могут выполняться параллельно.

2. Что такое GraphQL schema и почему это контракт?
Schema — это формальное описание доступных типов, запросов и мутаций. Она служит контрактом между клиентом и сервером: клиент точно знает, какие данные можно запросить и в каком формате они придут.

3. Что такое резолвер?
Резолвер — это функция, которая отвечает за получение данных для конкретного поля в схеме. Для каждого поля в запросе вызывается свой резолвер, который может обращаться к БД, другим API или вычислять значение.

4. Почему GraphQL часто решает проблему over-fetching?
В REST клиент часто получает больше данных, чем нужно (например, весь объект). В GraphQL клиент сам указывает, какие поля ему нужны, и сервер возвращает только их. Это решает проблему over-fetching (избыточных данных).

5. Какие риски у GraphQL без ограничений сложности запросов?
Клиент может отправить очень вложенный запрос, который вызовет множественные обращения к БД. Без ограничений (max depth, query complexity, pagination) злоумышленник может вызвать отказ в обслуживании (DoS-атака).


### Вывод

В результате практического занятия:

1. Создана GraphQL схема для управления задачами
2. Настроен и запущен сервер на основе gqlgen
3. Реализованы резолверы для Query и Mutation
4. GraphQL сервис интегрирован с общей БД PostgreSQL


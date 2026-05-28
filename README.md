        # temporal — Масштабирование воркеров

        Homework-шаблон для урока **l2_worker_scaling** (Масштабирование воркеров) на платформе Vibe Learn.

        ## Что делать

        Дан сервис на Go SDK (go.temporal.io/sdk), где один воркер исполняет workflow-задачи и
CPU-bound активность HeavyCompute. Шаг 1: под нагрузкой замерь schedule-to-start latency
workflow-задач (SDK-метрики через Prometheus в шаблоне) и зафиксируй залипание. Шаг 2:
раздели на два воркера — workflow-воркер и activity-воркер для HeavyCompute на отдельной
task queue, настрой MaxConcurrentActivityExecutionSize и WorkerActivitiesPerSecond.
Шаг 3: повтори замер и покажи, что schedule-to-start workflow-задач упал. Тесты проверят
корректную маршрутизацию задач по двум task queue и применение лимитов.

## Контекст (из transfer-задачи урока)

Сервис на Temporal: один тип воркера исполняет и workflow-задачи, и тяжёлую активность
GenerateReport (CPU-bound, рендерит PDF, 10–30 секунд каждый). После маркетинговой акции
нагрузка выросла. Симптомы: schedule-to-start latency у WORKFLOW-задач взлетел до десятков
секунд (workflow «залипают»), CPU воркеров — 100%, при этом внешние сервисы отвечают
быстро. Команда хочет «накатить ещё подов того же воркера».

**Вопрос:** разбери и предложи решение. Опиши:
(a) почему workflow-задачи залипают, хотя внешние сервисы быстрые;
(b) почему «ещё подов того же воркера» — частичное/плохое решение;
(c) правильную архитектуру воркеров и какие ручки крутить.

## Recap из урока

- **Главный симптом — schedule-to-start latency**: время задачи в очереди до подхвата воркером. Растёт → задачи копятся → добавь воркеров/поллеров или подними лимиты конкурентности.
- Ручки воркера: MaxConcurrentActivityExecutionSize, MaxConcurrentWorkflowTaskExecutionSize, MaxConcurrentLocalActivityExecutionSize, число поллеров (Workflow/ActivityTaskPollers).
- **Разделяй activity- и workflow-воркеров**: тяжёлые активности на общем воркере голодают лёгкие workflow-задачи → оркестрация залипает. Разные task queue, независимое масштабирование.
- **Sticky cache** (WorkerCacheSize) держит состояние workflow в RAM, избегая полного replay на каждой задаче. **Rate-limit** (WorkerActivitiesPerSecond / TaskQueueActivitiesPerSecond) защищает хрупкий downstream.
- «Добавить воркеров» — НЕ всегда ответ. Если затык в downstream — больше воркеров только сильнее его задолбят. Сначала диагноз: воркеры недозагружены / на 100% CPU / ждут downstream.

        ## Как работать

        1. Платформа Vibe Learn создаёт копию этого репо в твоём GitHub-аккаунте по клику «Начать домашку» на странице урока (через GitHub `/generate`, codecrafters-pattern).
        2. Склонируй копию локально, реализуй TODO в `main.go` (workflow + активности), прогони тесты, запушь.
        3. CI (`.github/workflows/ci.yml`) запускает `go vet` + `go test ./...` на каждый push. Платформа слушает результат через webhook от GitHub Actions и обновляет статус домашки на странице урока.

        ## Локальное окружение

        - Go 1.22+
        - SDK: `go.temporal.io/sdk`
        - Docker + docker-compose — `docker compose up` поднимает Temporal dev server на `:7233` + Web UI на `:8233`. Адрес переопределяется через env `TEMPORAL_ADDRESS` (дефолт `localhost:7233`).
        - Юнит-тесты на `testsuite.TestWorkflowEnvironment` (активности замоканы) бегут в CI БЕЗ сервера; интеграционный тест включается через `TEMPORAL_INTEGRATION=1`.

        ## Запуск

        ```bash
        # Поднять локальный Temporal dev server + UI
        docker compose up -d
        # Web UI: http://localhost:8233

        # Прогнать тесты (юнит на TestWorkflowEnvironment — без сервера;
        # интеграционный включается через TEMPORAL_INTEGRATION=1)
        go test ./...
        TEMPORAL_INTEGRATION=1 go test ./...

        # Запустить воркер (регистрирует workflow + активности, слушает task queue)
        go run .
        ```

        ## Заметка автора

        Это baseline-шаблон, сгенерированный платформой. Бизнес-сущность задачи (что конкретно реализовать в `main.go`, какие тесты сделать строгими) расширяется по ходу итераций — параллельно с углублением теории урока.

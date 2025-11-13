# SuperAGI Agent Pipelines Documentation

Этот документ описывает все пайплайны и схемы работы агента SuperAGI, включая потоки выполнения, обработку промтов, управление инструментами и память агента.

---

## Оглавление

1. [Введение в архитектуру](#введение-в-архитектуру)
2. [Pipeline 1: Direct Execution (Iteration Workflow)](#pipeline-1-direct-execution-iteration-workflow)
3. [Pipeline 2: Task-Based Execution](#pipeline-2-task-based-execution)
4. [Pipeline 3: Tool-Specific Execution](#pipeline-3-tool-specific-execution)
5. [Pipeline 4: Memory Management](#pipeline-4-memory-management)
6. [Pipeline 5: Permission Control](#pipeline-5-permission-control)
7. [Компоненты системы](#компоненты-системы)

---

## Введение в архитекту

### Общая архитектура системы

SuperAGI использует асинхронную архитектуру на базе Celery для выполнения агентов. Система состоит из следующих уровней:

```
┌─────────────────────────────────────────────────────────┐
│                CELERY BEAT SCHEDULER                     │
│  - execute_waiting_workflows (каждые 2 мин)            │
│  - initialize_schedule_agent (каждые 5 мин)            │
└──────────────────┬──────────────────────────────────────┘
                   │
┌──────────────────▼──────────────────────────────────────┐
│            CELERY WORKER POOL                            │
│  Task: execute_agent(agent_execution_id, time)          │
└──────────────────┬──────────────────────────────────────┘
                   │
┌──────────────────▼──────────────────────────────────────┐
│          AgentExecutor.execute_next_step()              │
│                                                          │
│  ┌────────────────────────────────────────────────┐    │
│  │   Validation & Loading                         │    │
│  │   - Проверка статуса и лимитов                 │    │
│  │   - Загрузка данных из БД                      │    │
│  └────────────────────────────────────────────────┘    │
│                                                          │
│  ┌────────────────────────────────────────────────┐    │
│  │   DISPATCHER по action_type                    │    │
│  └────────────────────────────────────────────────┘    │
└──┬───────────────────┬───────────────────┬─────────────┘
   │                   │                   │
┌──▼──────────────┐ ┌──▼──────────────┐ ┌─▼───────────┐
│ ITERATION       │ │ TOOL            │ │ WAIT        │
│ WORKFLOW        │ │ SPECIFIC        │ │ STEP        │
└─────────────────┘ └─────────────────┘ └─────────────┘
```

### Основные компоненты

**Файл:** `/superagi/jobs/agent_executor.py`

Главный оркестратор выполнения агента. Основные методы:
- `execute_next_step(agent_execution_id)` - основной цикл выполнения
- `__execute_workflow_step()` - диспетчер для обработчиков шагов
- `execute_waiting_workflows()` - обработка пауз и ожиданий

### Типы шагов (Action Types)

1. **ITERATION_WORKFLOW** - итеративное выполнение с LLM, где агент сам выбирает инструменты
2. **TOOL** - выполнение конкретного инструмента в workflow
3. **WAIT_STEP** - пауза в выполнении на заданное время

### Статусы выполнения

- `RUNNING` - нормальное выполнение
- `WAITING_FOR_PERMISSION` - ожидание разрешения пользователя
- `WAIT_STEP` - пауза в wait step
- `COMPLETED` - успешное завершение
- `ITERATION_LIMIT_EXCEEDED` - достигнут лимит итераций

---

## Pipeline 1: Direct Execution (Iteration Workflow)

### Описание

Direct Execution - это основной режим работы агента, где агент самостоятельно выбирает инструменты и параметры для их вызова на каждой итерации. LLM получает промт с целями, инструкциями, доступными инструментами и историей выполнения, после чего решает какой инструмент использовать.

### Компоненты

**Обработчик:** `AgentIterationStepHandler`
**Файл:** `/superagi/agent/agent_iteration_step_handler.py`
**Action Type:** `ITERATION_WORKFLOW`

### Схема пайплайна

```
┌─────────────────────────────────────────────────────────────┐
│ CELERY TASK: execute_agent(agent_execution_id)             │
└────────────────────────┬────────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────────┐
│ AgentExecutor.execute_next_step()                           │
│                                                              │
│  1. Валидация                                               │
│     ├─ Проверка статуса (RUNNING/WAITING_FOR_PERMISSION)   │
│     ├─ Проверка max_iterations                             │
│     └─ Проверка deleted status                             │
│                                                              │
│  2. Загрузка данных                                         │
│     ├─ AgentExecution (текущее состояние)                  │
│     ├─ Agent (конфигурация)                                │
│     ├─ AgentWorkflowStep (текущий шаг)                     │
│     └─ LLM Model                                            │
│                                                              │
│  3. Dispatcher: action_type == ITERATION_WORKFLOW           │
└────────────────────────┬────────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────────┐
│ AgentIterationStepHandler.execute_step()                    │
│                                                              │
│  STEP 1: Проверка разрешений                               │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ _handle_wait_for_permission()                        │  │
│  │ - Проверка наличия pending permissions              │  │
│  │ - Если есть → вернуть WAITING_FOR_PERMISSION        │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                              │
│  STEP 2: Построение инструментов                           │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ _build_tools()                                       │  │
│  │ ├─ ThinkingTool (всегда добавляется)                │  │
│  │ ├─ QueryResourceTool (при наличии ресурсов)         │  │
│  │ └─ Пользовательские инструменты из DB               │  │
│  └──────────────────────────────────────────────────────┘  │
│                         │                                    │
│                         ▼                                    │
│  STEP 3: Построение промпта                                │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ _build_agent_prompt()                                │  │
│  │ ├─ replace_main_variables()                          │  │
│  │ │  ├─ {goals} → список целей                        │  │
│  │ │  ├─ {instructions} → инструкции пользователя      │  │
│  │ │  ├─ {constraints} → ограничения агента            │  │
│  │ │  └─ {tools} → список доступных инструментов       │  │
│  │ └─ replace_task_based_variables() (если task queue) │  │
│  │    ├─ {current_task} → текущая задача              │  │
│  │    ├─ {pending_tasks} → список незавершенных        │  │
│  │    ├─ {completed_tasks} → список выполненных        │  │
│  │    └─ {task_history} → история с результатами       │  │
│  └──────────────────────────────────────────────────────┘  │
│                         │                                    │
│                         ▼                                    │
│  STEP 4: Формирование сообщений с историей                 │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ AgentLlmMessageBuilder.build_agent_messages()        │  │
│  │ ├─ Загрузка истории из AgentExecutionFeed           │  │
│  │ ├─ Расчет лимита токенов                            │  │
│  │ ├─ Разделение на past_messages и current_messages   │  │
│  │ ├─ LTM суммаризация для past_messages               │  │
│  │ └─ Формат:                                           │  │
│  │    [                                                 │  │
│  │      {"role": "system", "content": prompt},         │  │
│  │      {"role": "system", "content": current_time},   │  │
│  │      {"role": "system", "content": ltm_summary},    │  │
│  │      ...current_messages (последние N),             │  │
│  │      {"role": "user", "content": completion_prompt} │  │
│  │    ]                                                 │  │
│  └──────────────────────────────────────────────────────┘  │
│                         │                                    │
│                         ▼                                    │
│  STEP 5: Вызов LLM                                         │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ llm.chat_completion(messages, max_tokens)            │  │
│  │                                                       │  │
│  │ Ожидаемый ответ (JSON):                             │  │
│  │ {                                                    │  │
│  │   "thoughts": {                                      │  │
│  │     "text": "мысль",                                │  │
│  │     "reasoning": "рассуждение",                     │  │
│  │     "plan": "- шаг 1\n- шаг 2",                    │  │
│  │     "criticism": "самокритика",                     │  │
│  │     "speak": "сообщение пользователю"              │  │
│  │   },                                                 │  │
│  │   "tool": {                                          │  │
│  │     "name": "название_инструмента",                 │  │
│  │     "args": {"param": "value"}                      │  │
│  │   }                                                  │  │
│  │ }                                                    │  │
│  └──────────────────────────────────────────────────────┘  │
│                         │                                    │
│                         ▼                                    │
│  STEP 6: Обработка ответа                                  │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ ToolOutputHandler.handle()                           │  │
│  │ ├─ Парсинг JSON ответа                              │  │
│  │ ├─ Сохранение assistant reply в AgentExecutionFeed  │  │
│  │ ├─ _check_permission_in_restricted_mode()           │  │
│  │ │  └─ Если tool.permission_required и mode=RESTRICT │  │
│  │ │     → создать AgentExecutionPermission            │  │
│  │ │     → статус WAITING_FOR_PERMISSION               │  │
│  │ ├─ ToolExecutor.execute(tool_name, tool_args)       │  │
│  │ ├─ Сохранение tool result в AgentExecutionFeed      │  │
│  │ ├─ add_text_to_memory() → Vector Store (LTM)        │  │
│  │ ├─ _check_for_completion()                          │  │
│  │ │  └─ Если tool == "finish" → статус COMPLETED     │  │
│  │ └─ Return ToolExecutorResponse                       │  │
│  └──────────────────────────────────────────────────────┘  │
│                         │                                    │
│                         ▼                                    │
│  STEP 7: Обновление статуса                                │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ _update_agent_execution_next_step()                  │  │
│  │ ├─ Increment num_of_calls                           │  │
│  │ ├─ Update num_of_tokens                             │  │
│  │ └─ Update last_execution_time                       │  │
│  └──────────────────────────────────────────────────────┘  │
└────────────────────────┬────────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────────┐
│ AgentExecutor: Проверка статуса и рекурсия                 │
│                                                              │
│  if response.status == COMPLETED:                           │
│     → Завершение выполнения                                │
│                                                              │
│  elif response.status == WAITING_FOR_PERMISSION:            │
│     → Пауза до получения разрешения                        │
│                                                              │
│  else:                                                       │
│     → execute_agent.apply_async(countdown=2)                │
│     → Следующая итерация через 2 секунды                   │
└─────────────────────────────────────────────────────────────┘
```

### Используемые промты

**Основной промт:** `superagi.txt`
- Содержит системные инструкции для агента
- Определяет формат JSON ответа
- Включает evaluation критерии

**Переменные промта:**
- `{goals}` - список целей агента
- `{instructions}` - дополнительные инструкции
- `{constraints}` - ограничения агента
- `{tools}` - список доступных инструментов с описаниями

### Поток данных

#### Входные данные:
- `agent_execution_id` - ID текущего выполнения
- `goals` - цели агента из Agent
- `instructions` - инструкции из AgentExecution
- `agent_workflow` - текущий workflow step
- `tools` - доступные инструменты из DB

#### Промежуточные данные:
- `messages` - массив сообщений для LLM с историей
- `assistant_reply` - JSON ответ от LLM
- `tool_response` - результат выполнения инструмента

#### Выходные данные:
- `AgentExecutionFeed` - записи в БД с историей (assistant + system)
- `VectorStore` - chunks текста в долговременной памяти
- `ToolExecutorResponse` - статус выполнения (SUCCESS/ERROR/COMPLETE)
- Обновленный `AgentExecution` с новым статусом и счетчиками

### Пример выполнения

```
Итерация 1:
┌──────────────────────────────────────────────────────┐
│ Промт:                                               │
│ GOALS: [Create a Python script for web scraping]    │
│ TOOLS: [WriteFile, ReadFile, ThinkingTool, ...]    │
│ HISTORY: []                                          │
└──────────────────────────────────────────────────────┘
         │
         ▼
┌──────────────────────────────────────────────────────┐
│ LLM Ответ:                                           │
│ {                                                    │
│   "thoughts": {                                      │
│     "text": "Need to create a Python script",       │
│     "reasoning": "First, write code to file",       │
│     "plan": "- Write code\n- Test it",             │
│     "criticism": "Keep it simple",                  │
│     "speak": "Creating web scraper..."              │
│   },                                                 │
│   "tool": {                                          │
│     "name": "WriteFile",                            │
│     "args": {                                        │
│       "file_name": "scraper.py",                    │
│       "content": "import requests..."               │
│     }                                                │
│   }                                                  │
│ }                                                    │
└──────────────────────────────────────────────────────┘
         │
         ▼
┌──────────────────────────────────────────────────────┐
│ Tool Execution:                                      │
│ WriteFile.execute(file_name="scraper.py", ...)     │
│ → "File written successfully"                       │
└──────────────────────────────────────────────────────┘
         │
         ▼
┌──────────────────────────────────────────────────────┐
│ AgentExecutionFeed сохранение:                       │
│ 1. role=assistant, feed={LLM response JSON}         │
│ 2. role=system, feed="File written successfully"    │
└──────────────────────────────────────────────────────┘
         │
         ▼
    Итерация 2...
```

### Особенности

1. **Автономность** - агент сам выбирает инструменты и параметры
2. **История** - каждая итерация видит результаты предыдущих
3. **LTM** - старая история суммаризируется для экономии токенов
4. **Thinking Tool** - всегда доступен для анализа без действий
5. **Permission Control** - режим RESTRICTED требует разрешения пользователя
6. **Retry Logic** - автоматический retry при ошибках с exponential backoff

---

## Pipeline 2: Task-Based Execution

### Описание

Task-Based Execution - это режим работы агента с управлением очередью задач через Redis. Агент разбивает высокоуровневую цель на список подзадач, которые выполняются последовательно. Каждая задача обрабатывается в отдельной итерации с использованием основного Iteration Workflow.

### Компоненты

**Обработчик:** `AgentIterationStepHandler` + `QueueStepHandler`
**Файлы:**
- `/superagi/agent/agent_iteration_step_handler.py`
- `/superagi/agent/queue_step_handler.py`
- `/superagi/agent/task_queue.py`
**Action Type:** `ITERATION_WORKFLOW` (с `has_task_queue=True`)

### Схема пайплайна

```
┌─────────────────────────────────────────────────────────────┐
│ ИНИЦИАЛИЗАЦИЯ ЗАДАЧ                                         │
└─────────────────────────────────────────────────────────────┘

STEP 1: Генерация начального списка задач
┌─────────────────────────────────────────────────────────────┐
│ AgentIterationStepHandler с IterationWorkflow              │
│ (has_task_queue = true)                                     │
│                                                              │
│ Промт: initialize_tasks.txt                                │
│ ┌──────────────────────────────────────────────────────┐  │
│ │ GOALS: {goals}                                       │  │
│ │ {task_instructions}                                  │  │
│ │                                                       │  │
│ │ Construct a sequence of actions,                     │  │
│ │ not exceeding 3 steps                                │  │
│ └──────────────────────────────────────────────────────┘  │
│                         │                                    │
│                         ▼                                    │
│ ┌──────────────────────────────────────────────────────┐  │
│ │ LLM Response (JSON Array):                           │  │
│ │ [                                                    │  │
│ │   "Research web scraping libraries",                 │  │
│ │   "Write Python scraping code",                      │  │
│ │   "Test and save results"                            │  │
│ │ ]                                                    │  │
│ └──────────────────────────────────────────────────────┘  │
│                         │                                    │
│                         ▼                                    │
│ ┌──────────────────────────────────────────────────────┐  │
│ │ TaskQueue.add_task() для каждой задачи               │  │
│ │                                                       │  │
│ │ Redis Lists:                                         │  │
│ │ {queue_id}_q → ["Research...", "Write...", "Test"]  │  │
│ │ {queue_id}_q_completed → []                          │  │
│ └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│ ВЫПОЛНЕНИЕ ЗАДАЧ (Task Execution Loop)                     │
└─────────────────────────────────────────────────────────────┘

ИТЕРАЦИЯ N: Обработка текущей задачи
┌─────────────────────────────────────────────────────────────┐
│ AgentIterationStepHandler.execute_step()                    │
│                                                              │
│ STEP 1: Получение текущей задачи из очереди                │
│ ┌──────────────────────────────────────────────────────┐  │
│ │ task_queue.get_first_task()                          │  │
│ │ → current_task = "Research web scraping libraries"   │  │
│ │                                                       │  │
│ │ task_queue.get_completed_tasks()                     │  │
│ │ → completed_tasks = []                               │  │
│ │                                                       │  │
│ │ task_queue.get_tasks()                               │  │
│ │ → pending_tasks = ["Write...", "Test..."]           │  │
│ └──────────────────────────────────────────────────────┘  │
│                         │                                    │
│                         ▼                                    │
│ STEP 2: Построение промпта с информацией о задачах         │
│ ┌──────────────────────────────────────────────────────┐  │
│ │ _build_agent_prompt()                                │  │
│ │                                                       │  │
│ │ Промт: analyse_task.txt                             │  │
│ │ ┌────────────────────────────────────────────────┐  │  │
│ │ │ High level goal: {goals}                       │  │  │
│ │ │ Your Current Task: `{current_task}`            │  │  │
│ │ │ Task History: `{task_history}`                 │  │  │
│ │ │                                                 │  │  │
│ │ │ Based on this, pick tool to achieve task       │  │  │
│ │ │ TOOLS: {tools}                                 │  │  │
│ │ └────────────────────────────────────────────────┘  │  │
│ │                                                       │  │
│ │ Переменные:                                          │  │
│ │ {current_task} → "Research web scraping libraries"  │  │
│ │ {pending_tasks} → ["Write...", "Test..."]          │  │
│ │ {completed_tasks} → []                              │  │
│ │ {task_history} → история с результатами             │  │
│ └──────────────────────────────────────────────────────┘  │
│                         │                                    │
│                         ▼                                    │
│ STEP 3: LLM выбирает инструмент для задачи                 │
│ ┌──────────────────────────────────────────────────────┐  │
│ │ llm.chat_completion(messages)                        │  │
│ │                                                       │  │
│ │ Response:                                            │  │
│ │ {                                                    │  │
│ │   "thoughts": {                                      │  │
│ │     "reasoning": "Need to search online for libs"   │  │
│ │   },                                                 │  │
│ │   "tool": {                                          │  │
│ │     "name": "WebSearch",                            │  │
│ │     "args": {                                        │  │
│ │       "query": "best Python web scraping libraries" │  │
│ │     }                                                │  │
│ │   }                                                  │  │
│ │ }                                                    │  │
│ └──────────────────────────────────────────────────────┘  │
│                         │                                    │
│                         ▼                                    │
│ STEP 4: Выполнение инструмента                             │
│ ┌──────────────────────────────────────────────────────┐  │
│ │ ToolExecutor.execute("WebSearch", args)              │  │
│ │ → "Found: BeautifulSoup, Scrapy, Selenium..."       │  │
│ └──────────────────────────────────────────────────────┘  │
│                         │                                    │
│                         ▼                                    │
│ STEP 5: Проверка завершения и обновление очереди           │
│ ┌──────────────────────────────────────────────────────┐  │
│ │ _check_for_completion()                              │  │
│ │                                                       │  │
│ │ if tool_name == "finish" or task completed:          │  │
│ │    task_queue.complete_task(tool_response)           │  │
│ │    │                                                  │  │
│ │    ├─ LPOP from {queue_id}_q                         │  │
│ │    │  (удаляет "Research..." из pending)             │  │
│ │    │                                                  │  │
│ │    └─ LPUSH to {queue_id}_q_completed                │  │
│ │       (добавляет в completed с результатом)          │  │
│ │                                                       │  │
│ │ Redis после завершения:                              │  │
│ │ {queue_id}_q → ["Write...", "Test..."]              │  │
│ │ {queue_id}_q_completed → [                           │  │
│ │   {                                                  │  │
│ │     "task": "Research...",                           │  │
│ │     "response": "Found: BeautifulSoup..."           │  │
│ │   }                                                  │  │
│ │ ]                                                    │  │
│ └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────┐
│ СЛЕДУЮЩАЯ ИТЕРАЦИЯ                                          │
│                                                              │
│ if len(task_queue.get_tasks()) > 0:                         │
│    → execute_agent.apply_async(countdown=2)                 │
│    → Обработка следующей задачи "Write..."                 │
│                                                              │
│ else:                                                        │
│    → status = COMPLETED                                     │
│    → Все задачи выполнены                                  │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│ ДИНАМИЧЕСКОЕ УПРАВЛЕНИЕ ЗАДАЧАМИ                            │
└─────────────────────────────────────────────────────────────┘

CREATE NEW TASKS: Добавление новых задач во время выполнения
┌─────────────────────────────────────────────────────────────┐
│ Промт: create_tasks.txt                                     │
│ ┌──────────────────────────────────────────────────────┐  │
│ │ High level goal: {goals}                             │  │
│ │ Incomplete tasks: {pending_tasks}                    │  │
│ │ Completed tasks: {completed_tasks}                   │  │
│ │ Task History: {task_history}                         │  │
│ │                                                       │  │
│ │ Create a single task ONLY IF REQUIRED               │  │
│ │ Don't create if already covered                      │  │
│ └──────────────────────────────────────────────────────┘  │
│                         │                                    │
│                         ▼                                    │
│ ┌──────────────────────────────────────────────────────┐  │
│ │ LLM Response:                                        │  │
│ │ ["Create documentation for the scraper"]            │  │
│ │                                                       │  │
│ │ или                                                  │  │
│ │                                                       │  │
│ │ []  (no new tasks needed)                            │  │
│ └──────────────────────────────────────────────────────┘  │
│                         │                                    │
│                         ▼                                    │
│ ┌──────────────────────────────────────────────────────┐  │
│ │ task_queue.add_task() для каждой новой задачи        │  │
│ └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘

PRIORITIZE TASKS: Переприоритизация существующих задач
┌─────────────────────────────────────────────────────────────┐
│ Промт: prioritize_tasks.txt                                 │
│ ┌──────────────────────────────────────────────────────┐  │
│ │ High level goal: {goals}                             │  │
│ │ Incomplete tasks: {pending_tasks}                    │  │
│ │ Completed tasks: {completed_tasks}                   │  │
│ │                                                       │  │
│ │ Sort tasks in order of execution                     │  │
│ │ Remove unnecessary or duplicate tasks                │  │
│ └──────────────────────────────────────────────────────┘  │
│                         │                                    │
│                         ▼                                    │
│ ┌──────────────────────────────────────────────────────┐  │
│ │ LLM Response (prioritized array):                    │  │
│ │ [                                                    │  │
│ │   "Test and save results",                           │  │
│ │   "Write Python scraping code"                       │  │
│ │ ]                                                    │  │
│ └──────────────────────────────────────────────────────┘  │
│                         │                                    │
│                         ▼                                    │
│ ┌──────────────────────────────────────────────────────┐  │
│ │ task_queue.clear_tasks()                             │  │
│ │ task_queue.add_task() в новом порядке                │  │
│ └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

### Используемые промты

1. **initialize_tasks.txt** - начальная генерация задач (максимум 3 шага)
2. **analyse_task.txt** - анализ текущей задачи и выбор инструмента
3. **create_tasks.txt** - добавление новых задач при необходимости
4. **prioritize_tasks.txt** - переприоритизация списка задач

### Redis структура данных

```python
# Queue ID формат: agent_execution_{id}
queue_id = f"agent_execution_{agent_execution_id}"

# Redis Keys:
{queue_id}_q                 # LPUSH/LPOP - pending tasks (список строк)
{queue_id}_q_completed       # LPUSH - completed tasks (список объектов)
{queue_id}_status            # SET/GET - статус очереди (INITIATED/PROCESSING/COMPLETE)

# Примеры данных:
pending: ["Task 1", "Task 2", "Task 3"]
completed: [
  {
    "task": "Task 0",
    "response": "Result of task 0"
  }
]
```

### Поток данных

#### Входные данные:
- `goals` - высокоуровневые цели
- `task_instructions` - инструкции по управлению задачами
- `iteration_workflow` - workflow с `has_task_queue=True`

#### Промежуточные данные:
- `current_task` - текущая задача из очереди (первая в списке)
- `pending_tasks` - список незавершенных задач
- `completed_tasks` - список выполненных задач с результатами
- `task_history` - форматированная история для промпта

#### Выходные данные:
- Обновленная Redis очередь (pending/completed списки)
- `AgentExecutionFeed` - записи с выполнением каждой задачи
- Финальный статус `COMPLETED` когда очередь пуста

### Пример полного выполнения

```
═══════════════════════════════════════════════════════════
GOAL: "Create a web scraper for product prices"
═══════════════════════════════════════════════════════════

┌─────────────────────────────────────────────────────────┐
│ INITIALIZATION (Промт: initialize_tasks.txt)            │
└─────────────────────────────────────────────────────────┘
LLM генерирует задачи:
[
  "Research web scraping libraries and methods",
  "Write Python code to scrape product prices",
  "Test scraper and save results to file"
]

Redis State:
  pending: ["Research...", "Write...", "Test..."]
  completed: []

┌─────────────────────────────────────────────────────────┐
│ ITERATION 1: Task "Research..."                         │
└─────────────────────────────────────────────────────────┘
Промт: analyse_task.txt
  current_task: "Research web scraping libraries and methods"
  pending_tasks: ["Write...", "Test..."]
  completed_tasks: []

LLM → Tool: WebSearch("Python web scraping libraries")
Result: "BeautifulSoup, Scrapy, Selenium are popular..."

task_queue.complete_task(result)

Redis State:
  pending: ["Write...", "Test..."]
  completed: [{
    "task": "Research...",
    "response": "BeautifulSoup, Scrapy..."
  }]

┌─────────────────────────────────────────────────────────┐
│ ITERATION 2: Task "Write..."                            │
└─────────────────────────────────────────────────────────┘
Промт: analyse_task.txt
  current_task: "Write Python code to scrape product prices"
  pending_tasks: ["Test..."]
  completed_tasks: [{task: "Research...", response: "..."}]
  task_history: "Research... → BeautifulSoup, Scrapy..."

LLM → Tool: CodingTool(
  task: "Write scraper using BeautifulSoup",
  spec: "Based on research results..."
)
Result: "scraper.py created with BeautifulSoup code"

task_queue.complete_task(result)

Redis State:
  pending: ["Test..."]
  completed: [
    {task: "Research...", response: "BeautifulSoup..."},
    {task: "Write...", response: "scraper.py created..."}
  ]

┌─────────────────────────────────────────────────────────┐
│ ITERATION 3: Task "Test..."                             │
└─────────────────────────────────────────────────────────┘
Промт: analyse_task.txt
  current_task: "Test scraper and save results to file"
  pending_tasks: []
  completed_tasks: [...]
  task_history: "Research... → ...\nWrite... → ..."

LLM → Tool: Execute("python scraper.py")
Result: "Scraper tested successfully, results saved"

task_queue.complete_task(result)

Redis State:
  pending: []
  completed: [
    {task: "Research...", response: "..."},
    {task: "Write...", response: "..."},
    {task: "Test...", response: "results saved"}
  ]

┌─────────────────────────────────────────────────────────┐
│ COMPLETION                                               │
└─────────────────────────────────────────────────────────┘
len(pending_tasks) == 0
→ AgentExecution.status = COMPLETED
→ Все задачи выполнены
```

### Особенности

1. **Планирование** - LLM разбивает цель на подзадачи в начале
2. **Последовательность** - задачи выполняются одна за другой
3. **Контекст** - каждая задача видит результаты предыдущих через task_history
4. **Динамичность** - можно добавлять новые задачи или переприоритизировать
5. **Redis-based** - персистентность задач в Redis, выживает при перезагрузке
6. **Лимит итераций** - каждая задача учитывается в max_iterations

### Интеграция с Iteration Workflow

Task-Based Execution использует тот же `AgentIterationStepHandler`, но с дополнительной логикой:

```python
if iteration_workflow.has_task_queue:
    # Task-based mode
    current_task = task_queue.get_first_task()
    prompt = replace_task_based_variables(prompt,
                                          current_task,
                                          pending_tasks,
                                          completed_tasks)
else:
    # Regular iteration mode
    prompt = base_prompt
```

---

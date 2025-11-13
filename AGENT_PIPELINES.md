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

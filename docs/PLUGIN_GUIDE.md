<div align="center">

# 🧩 DevHub Launcher — Plugin Developer Guide

**Полное руководство по созданию плагинов**

[![Python](https://img.shields.io/badge/Python-3.10+-3776ab?style=flat-square&logo=python&logoColor=white)](https://python.org)
[![Vue.js](https://img.shields.io/badge/Vue.js-3.x-42b883?style=flat-square&logo=vue.js&logoColor=white)](https://vuejs.org)
[![Plugin API](https://img.shields.io/badge/Plugin_API-v1.0-6366f1?style=flat-square)]()

[← Назад к README](../README.md)

</div>

---

## Содержание

1. [Как устроена система плагинов](#1-как-устроена-система-плагинов)
2. [Структура плагина](#2-структура-плагина)
3. [Python API — PluginContext](#3-python-api--plugincontext)
   - [register_ui()](#31-ctxregister_uijs_path)
   - [register_method()](#32-ctxregister_methodname-func)
   - [get_data / set_data / delete_data](#33-хранилище-данных)
4. [JavaScript API — DevHubAPI](#4-javascript-api--devhubapi)
   - [registerTab()](#41-devhubapiregitertabid-title-iconsvg-componentdef)
   - [call()](#42-devhubapicallpluginid-method-args)
   - [registerStore()](#43-devhubapiregisterstorепluginid-initialstate)
   - [on() / emit()](#44-devhubapionеvent-handler--devhubapiemitevent-data)
   - [injectCSS()](#45-devhubapiinjectcsscssstring)
5. [Важное правило — toPlain()](#5-важное-правило--toplain)
6. [Структура Vue-компонента](#6-структура-vue-компонента)
7. [Полный пример — плагин Tasks](#7-полный-пример--плагин-tasks)
8. [Пример с длительными операциями](#8-пример-с-длительными-операциями)
9. [Отладка](#9-отладка)
10. [Чеклист перед публикацией](#10-чеклист-перед-публикацией)
11. [Быстрый справочник](#11-быстрый-справочник)

---

## 1. Как устроена система плагинов

Плагин состоит из двух независимых частей:

```
┌─────────────────────────────────────────────────────────────────┐
│                        DevHub Launcher                          │
│                                                                 │
│  ┌──────────────────┐          ┌──────────────────────────────┐ │
│  │   Python backend │          │      WebView2 (браузер)      │ │
│  │                  │          │                              │ │
│  │  __init__.py     │◄────────►│  ui.js                       │ │
│  │                  │          │                              │ │
│  │  - регистрирует  │  bridge  │  - регистрирует вкладку      │ │
│  │    методы        │          │  - Vue-компонент             │ │
│  │  - хранит данные │          │  - вызывает Python через     │ │
│  │                  │          │    DevHubAPI.call()          │ │
│  └──────────────────┘          └──────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

**Жизненный цикл:**

1. Лаунчер стартует → `PluginManager` читает папку `plugins/`
2. Для каждого плагина создаётся `PluginContext` (изолированная песочница)
3. Вызывается `setup(ctx)` — плагин регистрирует методы и путь к `ui.js`
4. Создаётся окно pywebview — API «замораживается»
5. Фронтенд загружается → выполняется `ui.js` каждого плагина
6. Vue монтируется → плагин виден в сайдбаре

> **Ключевой момент:** pywebview не позволяет добавлять методы к API после создания окна. Поэтому все вызовы плагинов идут через единственный метод-диспетчер `call_plugin()`, который зарегистрирован заранее. `DevHubAPI.call()` в JS — обёртка над ним.

---

## 2. Структура плагина

Плагин — это **папка** внутри `plugins/` с обязательным файлом `__init__.py`:

```
plugins/
└── my_plugin/              ← ID плагина = имя папки
    ├── __init__.py         ← ОБЯЗАТЕЛЬНО: серверная часть
    ├── ui.js               ← Опционально: клиентская часть (Vue)
    ├── README.md           ← Опционально: описание (первые 200 символов — в UI)
    └── assets/             ← Любые вспомогательные файлы
```

> **Имя папки** становится ID плагина. Используй только `[a-z0-9_]` (snake_case).  
> ✅ `notes` `theme_manager` `git_helper` `task_tracker`  
> ❌ `My Plugin` `task-tracker` `плагин`

### Минимальный плагин (без UI)

```python
# plugins/my_plugin/__init__.py

def setup(ctx):
    def hello(name: str) -> str:
        return f"Привет, {name}!"

    ctx.register_method("hello", hello)
```

### Полный плагин (с UI)

```python
# plugins/my_plugin/__init__.py
import os

def setup(ctx):
    # 1. Регистрируем UI
    ctx.register_ui(os.path.join(os.path.dirname(__file__), "ui.js"))

    # 2. Регистрируем методы
    def get_items() -> list:
        return ctx.get_data("items") or []

    def save_item(item: dict) -> bool:
        items = get_items()
        items.append(item)
        ctx.set_data("items", items)
        return True

    ctx.register_method("get_items", get_items)
    ctx.register_method("save_item", save_item)
```

---

## 3. Python API — PluginContext

Объект `ctx` передаётся в `setup(ctx)` и является **единственной** точкой взаимодействия с лаунчером. Плагин никогда не получает прямой доступ к `LauncherAPI`.

---

### 3.1. `ctx.register_ui(js_path)`

Регистрирует `.js` файл клиентской части. Файл будет выполнен в браузерном движке при старте приложения.

| Параметр | Тип | Описание |
|----------|-----|----------|
| `js_path` | `str` | Абсолютный путь к `.js` файлу |

```python
import os

def setup(ctx):
    # Всегда используй __file__ для надёжного пути
    ctx.register_ui(os.path.join(os.path.dirname(__file__), "ui.js"))
```

---

### 3.2. `ctx.register_method(name, func)`

Регистрирует Python-функцию, доступную из JS через `DevHubAPI.call()`.

| Параметр | Тип | Описание |
|----------|-----|----------|
| `name` | `str` | Имя метода (валидный Python-идентификатор) |
| `func` | `Callable` | Любой вызываемый объект |

**Правила для функций:**
- Аргументы и возвращаемое значение должны быть JSON-сериализуемыми (`str`, `int`, `float`, `list`, `dict`, `bool`, `None`)
- Имя: только буквы, цифры и `_` — пробелы и дефисы запрещены

```python
def setup(ctx):
    # Без аргументов
    def get_version() -> str:
        return "1.0.0"

    # Один простой аргумент
    def greet(name: str) -> str:
        return f"Привет, {name}!"

    # Объект (dict из JS)
    def save_record(record: dict) -> bool:
        ctx.set_data("last", record)
        return True

    # Список
    def sum_numbers(numbers: list) -> float:
        return sum(numbers)

    ctx.register_method("get_version",  get_version)
    ctx.register_method("greet",        greet)
    ctx.register_method("save_record",  save_record)
    ctx.register_method("sum_numbers",  sum_numbers)
```

---

### 3.3. Хранилище данных

Каждый плагин имеет **изолированное хранилище** в `plugins_data/{plugin_id}/`. Данные хранятся как JSON-файлы, один файл = один ключ.

| Метод | Возвращает | Описание |
|-------|-----------|----------|
| `ctx.get_data(key)` | `Any \| None` | Читает значение. `None` если ключа нет |
| `ctx.set_data(key, value)` | `bool` | Записывает JSON-сериализуемое значение |
| `ctx.delete_data(key)` | `bool` | Удаляет ключ |

```python
def setup(ctx):
    def get_settings() -> dict:
        saved = ctx.get_data("settings")
        if saved is None:
            # Значения по умолчанию
            return {"theme": "dark", "notifications": True}
        return saved

    def save_settings(settings: dict) -> bool:
        return ctx.set_data("settings", settings)

    def reset() -> bool:
        return ctx.delete_data("settings")

    ctx.register_method("get_settings",  get_settings)
    ctx.register_method("save_settings", save_settings)
    ctx.register_method("reset",         reset)
```

> **Ключи** могут содержать любые символы кроме `/`, `\`, `..`.  
> Каждому ключу соответствует файл `plugins_data/{id}/{key}.json`.

---

## 4. JavaScript API — DevHubAPI

Глобальный объект `window.DevHubAPI`. Доступен во всех плагинах.  
Так же глобально доступна утилита `window.toPlain()` — читай [раздел 5](#5-важное-правило--toplain).

---

### 4.1. `DevHubAPI.registerTab(id, title, iconSvg, componentDef)`

Регистрирует вкладку в боковой панели и Vue-компонент для неё.

| Параметр | Тип | Описание |
|----------|-----|----------|
| `id` | `string` | Уникальный ID вкладки (должен совпадать с ID плагина) |
| `title` | `string` | Текст в боковом меню |
| `iconSvg` | `string` | Строка SVG для иконки (рекомендуется `18×18`) |
| `componentDef` | `object` | Vue component options: `{ setup(), template }` |

```javascript
// plugins/my_plugin/ui.js
(function () {
    const { ref } = window.Vue;

    const MyTab = {
        setup() {
            const count = ref(0);
            return { count };
        },
        template: `
            <div class="p-8">
                <h2 class="text-2xl font-bold text-white">Мой плагин</h2>
                <p class="text-zinc-400 mt-2">Кликов: {{ count }}</p>
                <button @click="count++"
                    class="bg-indigo-600 text-white px-4 py-2 rounded-xl mt-4">
                    Кликни
                </button>
            </div>
        `
    };

    const ICON = `<svg width="18" height="18" viewBox="0 0 24 24"
        fill="none" stroke="currentColor" stroke-width="2">
        <circle cx="12" cy="12" r="10"/>
    </svg>`;

    DevHubAPI.registerTab("my_plugin", "Мой плагин", ICON, MyTab);

})(); // ← Весь код обёрнут в IIFE
```

> ⚠️ **Всегда** оборачивай код `ui.js` в IIFE `(function() { ... })()`.  
> Это предотвращает засорение глобального пространства имён.

---

### 4.2. `DevHubAPI.call(pluginId, method, ...args)`

Вызывает Python-метод, зарегистрированный через `ctx.register_method()`. Возвращает `Promise`.

| Параметр | Тип | Описание |
|----------|-----|----------|
| `pluginId` | `string` | ID плагина (имя папки) |
| `method` | `string` | Имя метода из `register_method()` |
| `...args` | `any` | Аргументы для Python-функции |

```javascript
// Без аргументов
const version = await DevHubAPI.call("my_plugin", "get_version");

// Один аргумент
const greeting = await DevHubAPI.call("my_plugin", "greet", "Мир");

// Объект — ОБЯЗАТЕЛЬНО toPlain() если реактивный!
const saved = await DevHubAPI.call("my_plugin", "save_record", toPlain(myObj));

// Всегда используй try/catch
try {
    const result = await DevHubAPI.call("my_plugin", "save", toPlain(data.value));
} catch (err) {
    console.error("Ошибка:", err.message); // текст Python-исключения
}
```

> 🔴 **Читай раздел 5** — передача реактивных объектов без `toPlain()` — самая частая ошибка в плагинах.

---

### 4.3. `DevHubAPI.registerStore(pluginId, initialState)`

Создаёт изолированное **реактивное хранилище** состояния для плагина. Переживает переключение вкладок. Сбрасывается при перезапуске лаунчера.

Вызов идемпотентен — повторный вызов с тем же `pluginId` вернёт существующий объект.

```javascript
(function () {
    const { onMounted } = window.Vue;

    const MyTab = {
        setup() {
            const state = DevHubAPI.registerStore("my_plugin", {
                items:     [],
                isLoading: false,
                filter:    "",
            });

            onMounted(async () => {
                state.isLoading = true;
                state.items = await DevHubAPI.call("my_plugin", "get_items");
                state.isLoading = false;
            });

            return { state };
        },
        template: `
            <div class="p-8">
                <input v-model="state.filter" class="input-field px-4 py-2 rounded-xl w-full mb-4">
                <div v-if="state.isLoading" class="text-zinc-400">Загрузка...</div>
                <div v-for="item in state.items" :key="item.id" class="glass-panel p-4 rounded-xl mb-3">
                    {{ item.name }}
                </div>
            </div>
        `
    };

    DevHubAPI.registerTab("my_plugin", "Плагин", `<svg>...</svg>`, MyTab);
})();
```

---

### 4.4. `DevHubAPI.on(event, handler)` / `DevHubAPI.emit(event, data)`

Шина событий. Позволяет реагировать на события лаунчера и общаться между плагинами.

**Встроенные события лаунчера:**

| Событие | Данные | Когда |
|---------|--------|-------|
| `tab:changed` | `{ tabId: string }` | Пользователь переключил вкладку |
| `app:started` | `{ appId: string }` | Приложение запущено |
| `app:stopped` | `{ appId: string }` | Приложение остановлено |

`on()` возвращает функцию отписки.

```javascript
// Подписка — обновить данные когда открыли нашу вкладку
const unsubscribe = DevHubAPI.on("tab:changed", ({ tabId }) => {
    if (tabId === "my_plugin") loadData();
});

// Отписка (в onUnmounted если нужно)
unsubscribe();

// Отправить своё событие (для межплагинного общения)
DevHubAPI.emit("my_plugin:item_saved", { id: "abc", name: "Запись" });

// Другой плагин слушает
DevHubAPI.on("my_plugin:item_saved", ({ id, name }) => {
    console.log("Сохранено:", name);
});
```

---

### 4.5. `DevHubAPI.injectCSS(cssString)`

Добавляет CSS в документ. Применяется мгновенно и глобально.

```javascript
DevHubAPI.injectCSS(`
    /* Стиль своего компонента */
    .my-card {
        background: linear-gradient(135deg, #6366f1, #8b5cf6);
        border-radius: 16px;
        padding: 20px;
    }

    /* Переопределение глобальных стилей лаунчера */
    body { background: #0d1117 !important; }
    .glass-panel { border-color: #30363d !important; }
`);
```

---

## 5. Важное правило — toPlain()

> 🔴 **Это самая частая причина ошибок в плагинах. Прочитай внимательно.**

Vue.js оборачивает объекты в **реактивные Proxy** при присвоении в `reactive()`, `ref()`, или через `v-model`. Когда такой объект передаётся через `DevHubAPI.call()`, браузерный движок сериализует его в JSON — и Proxy теряет данные, Python получает **пустой словарь `{}`**.

**Решение:** функция `toPlain()`, глобально доступная как `window.toPlain()`:

```javascript
// Реализация: JSON.parse(JSON.stringify(val))
// Гарантирует полностью plain JS объект
```

### Что безопасно передавать напрямую

| Значение | Нужен toPlain? | Пример |
|----------|:---:|--------|
| Строки, числа, boolean | ❌ | `DevHubAPI.call("p", "m", "hello", 42)` |
| `null`, `undefined` | ❌ | `DevHubAPI.call("p", "m", null)` |
| Объектные литералы `{}` | ❌ | `DevHubAPI.call("p", "m", { key: "val" })` |
| Массивы-литералы `[]` | ❌ | `DevHubAPI.call("p", "m", [1, 2, 3])` |
| `ref.value` | ✅ | `DevHubAPI.call("p", "m", toPlain(myRef.value))` |
| `reactive()` объект | ✅ | `DevHubAPI.call("p", "m", toPlain(state))` |
| `computed.value` | ✅ | `DevHubAPI.call("p", "m", toPlain(computed.value))` |
| Объект из `v-model` | ✅ | `DevHubAPI.call("p", "m", toPlain(formData.value))` |

### Примеры

```javascript
const { ref, reactive } = window.Vue;

const editing = ref({ title: "", color: "blue" });
const state   = reactive({ items: [], filter: "" });

// ✗ НЕПРАВИЛЬНО — Vue Proxy, Python получит {}
await DevHubAPI.call("notes", "save", editing.value);
await DevHubAPI.call("notes", "save", state);

// ✓ ПРАВИЛЬНО — plain объект
await DevHubAPI.call("notes", "save", toPlain(editing.value));
await DevHubAPI.call("notes", "save", toPlain(state));

// ✓ ПРАВИЛЬНО — объектный литерал, не реактивный
await DevHubAPI.call("notes", "save", { title: "Заголовок", done: false });

// ✓ ПРАВИЛЬНО — примитивы без toPlain
await DevHubAPI.call("tasks", "delete", "abc123");
await DevHubAPI.call("tasks", "set_done", taskId, true);
```

---

## 6. Структура Vue-компонента

Доступные глобальные объекты в `ui.js`:

| Объект | Что это |
|--------|---------|
| `window.Vue` | Vue 3 (деструктурируй `ref`, `computed`, `reactive`, etc.) |
| `window.DevHubAPI` | API системы плагинов |
| `window.toPlain` | Утилита для сериализации реактивных объектов |
| Tailwind CSS | Все utility-классы доступны из коробки |

**CSS-классы лаунчера, доступные в плагинах:**

| Класс | Описание |
|-------|----------|
| `.glass-panel` | Тёмная карточка с бордером (`bg: #18181b, border: #27272a`) |
| `.input-field` | Поле ввода в стиле лаунчера |
| `.custom-scroll` | Кастомный тёмный скроллбар |
| `.terminal-scroll` | Скролл с возможностью выделения текста |
| `.spinner` | CSS-анимация вращения (для индикаторов загрузки) |
| `.nav-item` | Элемент бокового меню |
| `.nav-item.active` | Активный элемент меню |

**Шаблон полного компонента:**

```javascript
// plugins/my_plugin/ui.js
(function () {
    const { ref, computed, onMounted, nextTick, watch } = window.Vue;

    const MyTab = {
        setup() {
            // --- СОСТОЯНИЕ ---
            const state = DevHubAPI.registerStore("my_plugin", {
                items:     [],
                isLoading: true,
            });
            const editing   = ref(null);
            const saveError = ref("");

            // --- ДАННЫЕ ---
            const loadItems = async () => {
                state.isLoading = true;
                try {
                    state.items = await DevHubAPI.call("my_plugin", "get_items");
                } catch (e) {
                    console.error("[my_plugin] load:", e);
                } finally {
                    state.isLoading = false;
                }
            };

            // --- CRUD ---
            const saveItem = async () => {
                if (!editing.value) return;
                saveError.value = "";
                try {
                    const saved = await DevHubAPI.call(
                        "my_plugin", "save_item", toPlain(editing.value)
                    );
                    const idx = state.items.findIndex(i => i.id === saved.id);
                    if (idx !== -1) state.items[idx] = saved;
                    else            state.items.unshift(saved);
                    editing.value = null;
                } catch (err) {
                    saveError.value = err.message;
                }
            };

            // --- ЖИЗНЕННЫЙ ЦИКЛ ---
            onMounted(loadItems);

            // Обновляем при переходе на вкладку
            DevHubAPI.on("tab:changed", ({ tabId }) => {
                if (tabId === "my_plugin") loadItems();
            });

            return { state, editing, saveError, loadItems, saveItem };
        },

        template: `
        <div class="p-8">
            <!-- Заголовок -->
            <div class="flex justify-between items-center mb-6">
                <h2 class="text-2xl font-bold text-white tracking-tight">Мой плагин</h2>
                <button type="button" @click="editing = {}"
                    class="bg-indigo-600 hover:bg-indigo-700 text-white px-5 py-2.5
                           rounded-xl font-medium transition-colors">
                    + Добавить
                </button>
            </div>

            <!-- Список -->
            <div v-if="state.isLoading" class="text-zinc-400">Загрузка...</div>
            <div v-else class="grid grid-cols-1 md:grid-cols-2 gap-4">
                <div v-for="item in state.items" :key="item.id"
                     class="glass-panel p-4 rounded-xl">
                    {{ item.name }}
                </div>
            </div>

            <!-- Модалка -->
            <div v-if="editing !== null"
                 class="fixed inset-0 bg-black/60 backdrop-blur-sm z-50
                        flex items-center justify-center p-4"
                 @click.self="editing = null">
                <div class="glass-panel w-full max-w-md rounded-2xl p-6">
                    <input v-model="editing.name" type="text"
                           placeholder="Название"
                           class="input-field w-full px-4 py-2.5 rounded-xl mb-4">
                    <div v-if="saveError" class="text-red-400 text-sm mb-3">
                        ⚠ {{ saveError }}
                    </div>
                    <div class="flex justify-end gap-2">
                        <button type="button" @click.stop="editing = null"
                            class="px-4 py-2 rounded-xl text-zinc-400 hover:text-white
                                   hover:bg-zinc-800 transition-colors text-sm">
                            Отмена
                        </button>
                        <button type="button" @click.stop="saveItem"
                            class="bg-indigo-600 hover:bg-indigo-700 text-white px-5 py-2
                                   rounded-xl text-sm font-medium transition-colors">
                            Сохранить
                        </button>
                    </div>
                </div>
            </div>
        </div>
        `
    };

    DevHubAPI.registerTab("my_plugin", "Мой плагин", `
        <svg width="18" height="18" viewBox="0 0 24 24" fill="none"
             stroke="currentColor" stroke-width="2">
            <circle cx="12" cy="12" r="10"/>
        </svg>
    `, MyTab);

})();
```

> **Важно:** корневой `<div>` компонента должен быть просто `<div class="p-8">` — **без** `h-full`, `overflow-y-auto` и других классов управления высотой. Скролл страницы управляется родительским `<main>` в лаунчере.

---

## 7. Полный пример — плагин Tasks

Демонстрирует весь API: хранение данных, CRUD, реактивный стор, обработку ошибок, `toPlain()`.

### `plugins/tasks/__init__.py`

```python
import os
import time

def setup(ctx):
    ctx.register_ui(os.path.join(os.path.dirname(__file__), "ui.js"))

    def _load() -> list:
        return ctx.get_data("tasks") or []

    def _save(tasks: list) -> None:
        ctx.set_data("tasks", tasks)

    def get_tasks() -> list:
        return sorted(_load(), key=lambda t: t.get("created_at", 0), reverse=True)

    def add_task(task: dict) -> dict:
        tasks = _load()
        task["id"]         = os.urandom(6).hex()
        task["created_at"] = time.time()
        task["done"]       = False
        tasks.append(task)
        _save(tasks)
        return task

    def toggle_task(task_id: str) -> bool:
        tasks = _load()
        for t in tasks:
            if t["id"] == task_id:
                t["done"] = not t["done"]
                _save(tasks)
                return True
        return False

    def delete_task(task_id: str) -> bool:
        tasks = [t for t in _load() if t["id"] != task_id]
        _save(tasks)
        return True

    ctx.register_method("get_tasks",   get_tasks)
    ctx.register_method("add_task",    add_task)
    ctx.register_method("toggle_task", toggle_task)
    ctx.register_method("delete_task", delete_task)
```

### `plugins/tasks/ui.js`

```javascript
(function () {
    const { ref, computed, onMounted } = window.Vue;

    const TasksTab = {
        setup() {
            const state = DevHubAPI.registerStore("tasks", {
                tasks:     [],
                isLoading: true,
            });
            const newTitle = ref("");
            const error    = ref("");

            const load = async () => {
                state.isLoading = true;
                try {
                    state.tasks = await DevHubAPI.call("tasks", "get_tasks");
                } catch (e) {
                    console.error("[tasks] load:", e);
                } finally {
                    state.isLoading = false;
                }
            };

            const addTask = async () => {
                const title = newTitle.value.trim();
                if (!title) return;
                error.value = "";
                try {
                    // title — строка, toPlain не нужен
                    const saved = await DevHubAPI.call("tasks", "add_task", { title });
                    state.tasks.unshift(saved);
                    newTitle.value = "";
                } catch (e) {
                    error.value = e.message;
                }
            };

            const toggle = async (id) => {
                await DevHubAPI.call("tasks", "toggle_task", id);
                const t = state.tasks.find(t => t.id === id);
                if (t) t.done = !t.done;
            };

            const remove = async (id) => {
                await DevHubAPI.call("tasks", "delete_task", id);
                state.tasks = state.tasks.filter(t => t.id !== id);
            };

            const pending = computed(() => state.tasks.filter(t => !t.done).length);

            onMounted(load);
            DevHubAPI.on("tab:changed", ({ tabId }) => {
                if (tabId === "tasks") load();
            });

            return { state, newTitle, error, addTask, toggle, remove, pending };
        },

        template: `
        <div class="p-8">
            <div class="flex justify-between items-center mb-6">
                <div>
                    <h2 class="text-2xl font-bold text-white tracking-tight">Задачи</h2>
                    <p class="text-zinc-400 text-sm mt-1">
                        {{ pending }} активных из {{ state.tasks.length }}
                    </p>
                </div>
            </div>

            <div class="flex gap-2 mb-6 max-w-xl">
                <input v-model="newTitle"
                       @keyup.enter="addTask"
                       type="text"
                       placeholder="Новая задача..."
                       class="input-field flex-1 px-4 py-2.5 rounded-xl text-sm">
                <button type="button" @click="addTask"
                    class="bg-indigo-600 hover:bg-indigo-700 text-white px-5 py-2.5
                           rounded-xl font-medium transition-colors">
                    Добавить
                </button>
            </div>

            <div v-if="error" class="text-red-400 text-sm mb-4">⚠ {{ error }}</div>

            <div v-if="state.isLoading" class="text-zinc-500">Загрузка...</div>

            <div v-else class="space-y-2 max-w-xl">
                <div v-for="task in state.tasks" :key="task.id"
                     class="glass-panel p-4 rounded-xl flex items-center gap-3">
                    <input type="checkbox"
                           :checked="task.done"
                           @change="toggle(task.id)"
                           class="w-4 h-4 cursor-pointer accent-indigo-500 flex-shrink-0">
                    <span class="flex-1"
                          :class="task.done ? 'text-zinc-500 line-through' : 'text-white'">
                        {{ task.title }}
                    </span>
                    <button type="button" @click="remove(task.id)"
                        class="text-zinc-600 hover:text-red-400 transition-colors flex-shrink-0">
                        <svg width="14" height="14" fill="none" stroke="currentColor" stroke-width="2"
                             viewBox="0 0 24 24">
                            <polyline points="3 6 5 6 21 6"/>
                            <path d="M19 6v14a2 2 0 0 1-2 2H7a2 2 0 0 1-2-2V6"/>
                        </svg>
                    </button>
                </div>
                <div v-if="state.tasks.length === 0"
                     class="text-center text-zinc-600 py-8">
                    Задач нет. Добавь первую!
                </div>
            </div>
        </div>
        `
    };

    DevHubAPI.registerTab("tasks", "Задачи", `
        <svg width="18" height="18" viewBox="0 0 24 24" fill="none"
             stroke="currentColor" stroke-width="2">
            <path d="M9 11l3 3L22 4"/>
            <path d="M21 12v7a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2V5a2 2 0 0 1 2-2h11"/>
        </svg>
    `, TasksTab);

})();
```

---

## 8. Пример с длительными операциями

Python-методы вызываются в основном потоке. Для долгих операций (сеть, файлы) используй `threading`:

```python
# plugins/downloader/__init__.py
import os
import threading
import urllib.request

def setup(ctx):
    ctx.register_ui(os.path.join(os.path.dirname(__file__), "ui.js"))

    # Словарь задач: task_id → {status, progress, error, result}
    _tasks = {}

    def start_download(url: str) -> str:
        task_id = os.urandom(6).hex()
        _tasks[task_id] = {"status": "running", "progress": 0, "error": ""}

        def _worker():
            try:
                # Долгая операция в фоновом потоке
                req = urllib.request.Request(url, headers={"User-Agent": "DevHub"})
                with urllib.request.urlopen(req) as r:
                    total = int(r.getheader("content-length", 0))
                    data = b""
                    while True:
                        chunk = r.read(8192)
                        if not chunk:
                            break
                        data += chunk
                        if total:
                            _tasks[task_id]["progress"] = int(len(data) / total * 100)

                _tasks[task_id]["status"]   = "done"
                _tasks[task_id]["progress"] = 100
                _tasks[task_id]["result"]   = len(data)  # байт скачано
            except Exception as e:
                _tasks[task_id]["status"] = "error"
                _tasks[task_id]["error"]  = str(e)

        threading.Thread(target=_worker, daemon=True).start()
        return task_id  # возвращаем сразу, не ждём

    def get_status(task_id: str) -> dict:
        return _tasks.get(task_id, {"status": "not_found"})

    ctx.register_method("start_download", start_download)
    ctx.register_method("get_status",     get_status)
```

```javascript
// Фронтенд: запускаем и поллим статус
const startDownload = async (url) => {
    const taskId = await DevHubAPI.call("downloader", "start_download", url);

    // Поллинг каждые 300мс
    const interval = setInterval(async () => {
        const st = await DevHubAPI.call("downloader", "get_status", taskId);
        progress.value = st.progress;

        if (st.status === "done" || st.status === "error") {
            clearInterval(interval);
            if (st.status === "error") errorMsg.value = st.error;
        }
    }, 300);
};
```

---

## 9. Отладка

### Python-ошибки

Запускай лаунчер из терминала:

```bash
python main.py
```

Все сообщения плагинов выводятся с префиксом `[PluginManager]`:

```
[PluginManager]  ✓ notes    <PluginContext id='notes' methods=['get_notes', 'save_note', 'delete_note']>
[PluginManager]  ✓ themes   <PluginContext id='themes' methods=['get_themes', 'apply_theme']>
[PluginManager]  ✗ my_plugin: AttributeError: Plugin must expose setup(ctx)
[PluginManager]  – broken_plugin  (disabled)
[PluginManager] Active: ['notes', 'themes']
```

### JavaScript-ошибки

Включи DevTools в `main.py`:

```python
webview.start(debug=True)  # F12 откроет DevTools в окне
```

Все `console.log` и `console.error` из плагинов видны во вкладке **Console**.

### Типичные ошибки

| Симптом | Причина | Решение |
|---------|---------|---------|
| Python получает `{}` | Vue Proxy передан без `toPlain()` | `toPlain(editing.value)` |
| `Backend method not found` | Метод не зарегистрирован | Проверь `ctx.register_method()` |
| Вкладка не появляется | Ошибка в `ui.js` при выполнении | Смотри Console в DevTools |
| `Plugin must expose setup(ctx)` | Нет функции `setup` в `__init__.py` | Добавь `def setup(ctx):` |
| Данные не сохраняются | Значение не JSON-сериализуемо | Проверь типы (нет `datetime`, `set`, etc.) |
| Кнопка "не нажимается" | Исключение в `async` без `catch` | Добавь `try/catch`, покажи `saveError` пользователю |

---

## 10. Чеклист перед публикацией

### Python

- [ ] Имя папки плагина — `snake_case`, только `[a-z0-9_]`
- [ ] `__init__.py` содержит функцию `def setup(ctx):`
- [ ] Все методы зарегистрированы через `ctx.register_method()`
- [ ] Имена методов — валидные Python-идентификаторы (без пробелов, дефисов)
- [ ] Все методы возвращают JSON-сериализуемые типы
- [ ] Длительные операции выполняются в `threading.Thread(daemon=True)`
- [ ] Данные хранятся через `ctx.get_data()` / `ctx.set_data()`

### JavaScript

- [ ] Весь код обёрнут в IIFE: `(function () { ... })()`
- [ ] Vue берётся из `window.Vue`, не из импортов
- [ ] Реактивные объекты оборачиваются в `toPlain()` перед `DevHubAPI.call()`
- [ ] Все `await DevHubAPI.call()` обёрнуты в `try/catch`
- [ ] Ошибки сохранения показываются пользователю (не только в консоль)
- [ ] Данные обновляются при событии `tab:changed`
- [ ] Все кнопки имеют `type="button"`
- [ ] Корневой `<div>` — только `class="p-8"`, без `h-full overflow-y-auto`

### Документация

- [ ] В папке плагина есть `README.md`
- [ ] README содержит: описание, что умеет, как использовать
- [ ] Плагин протестирован с `debug=True` — нет ошибок в DevTools Console

---

## 11. Быстрый справочник

### Python PluginContext

```python
ctx.register_ui(js_path)             # регистрировать ui.js
ctx.register_method(name, func)      # зарегистрировать метод
ctx.get_data(key)      → Any | None  # читать из хранилища
ctx.set_data(key, val) → bool        # писать в хранилище
ctx.delete_data(key)   → bool        # удалить из хранилища
ctx.id                 → str         # ID плагина (имя папки)
```

### JavaScript DevHubAPI

```javascript
DevHubAPI.registerTab(id, title, iconSvg, component)  // зарегистрировать вкладку
DevHubAPI.call(pluginId, method, ...args)  → Promise  // вызвать Python-метод
DevHubAPI.registerStore(pluginId, initial) → Proxy    // создать реактивный стор
DevHubAPI.getStore(pluginId)               → Proxy    // получить существующий стор
DevHubAPI.on(event, handler)               → Function // подписаться (возвращает отписку)
DevHubAPI.emit(event, data)                           // отправить событие
DevHubAPI.injectCSS(css)                              // добавить стили

toPlain(val)                               → object   // Vue Proxy → plain object
```

### События лаунчера

```javascript
"tab:changed"  → { tabId: string }  // переключение вкладки
"app:started"  → { appId: string }  // проект запущен
"app:stopped"  → { appId: string }  // проект остановлен
```

### CSS-классы лаунчера

```
.glass-panel     — тёмная карточка с бордером
.input-field     — поле ввода
.custom-scroll   — кастомный scrollbar
.spinner         — CSS-анимация вращения
.nav-item        — элемент сайдбара
.nav-item.active — активный элемент сайдбара
```

---

<div align="center">

[← Назад к README](../README.md)

Нашёл ошибку или есть идея для улучшения?  
[Открой issue](https://github.com/Yambai/DevHub-Launcher/issues) или [Pull Request](https://github.com/Yambai/DevHub-Launcher/pulls) — всё приветствуется!

</div>

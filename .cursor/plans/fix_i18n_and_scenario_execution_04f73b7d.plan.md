---
name: Fix i18n and scenario execution
overview: "Исправить две проблемы: (1) русская локализация не применяется к CodeLens в UI VSCode, (2) запуск первого сценария через \"Run Test\" выполняет все сценарии в файле вместо одного."
todos:
  - id: fix-codelens-i18n
    content: Добавить EventEmitter для обновления CodeLens при изменении локали в extension.ts
    status: pending
  - id: add-scenario-type-check
    content: Добавить метод isScenarioOutline() в test-executor.ts для определения типа сценария
    status: completed
  - id: fix-run-scenario-logic
    content: Исправить логику runScenario() для корректного выполнения обычных сценариев
    status: completed
  - id: fix-run-scenario-with-output
    content: Исправить логику runScenarioWithOutput() аналогично runScenario()
    status: completed
  - id: test-changes
    content: "Протестировать оба исправления: локализацию CodeLens и выполнение сценариев"
    status: pending
isProject: false
---

# План исправления проблем с i18n и выполнением сценариев

## Проблема 1: Русская локализация не применяется к CodeLens

### Причина

CodeLens создаются при вызове `provideCodeLenses()` в `[src/parsers/feature-parser.ts](src/parsers/feature-parser.ts)` (строки 441-577). Функция `t()` вызывается для получения локализованного текста (строки 483-484, 493-494, 516, 525, 558).

Однако VS Code **кэширует CodeLens**, и при изменении настройки `behaveTestRunner.uiLocale` CodeLens не обновляются автоматически. Также возможна проблема с инициализацией локализации до момента создания CodeLens.

### Решение

1. **Добавить принудительное обновление CodeLens при изменении локали** в `[src/i18n/localization-service.ts](src/i18n/localization-service.ts)`:
  - В методе `setupConfigChangeListener()` (строка 93) добавить вызов `vscode.commands.executeCommand('_workbench.action.reloadWindow')` или использовать событие для обновления CodeLens провайдера
  - Альтернатива: использовать `vscode.languages.registerCodeLensProvider` с поддержкой `onDidChangeCodeLenses` event emitter
2. **Добавить EventEmitter для обновления CodeLens** в `[src/extension.ts](src/extension.ts)`:
  - Создать `onDidChangeCodeLenses` event emitter в CodeLens провайдере (строка 139-156)
  - При изменении конфигурации `uiLocale` вызывать `fire()` для обновления CodeLens
3. **Проверить порядок инициализации** в `[src/extension.ts](src/extension.ts)`:
  - Убедиться, что `getLocalizationService()` вызывается ДО регистрации CodeLens провайдера (строки 46-49 vs 138-157)

## Проблема 2: Запуск первого сценария выполняет все сценарии

### Причина

В `[src/core/test-executor.ts](src/core/test-executor.ts)`, метод `runScenario()` (строка 46):

**Проблемная логика** (строка 85):

```typescript
let command = `${behaveCommand} "${filePath}${lineNumber ? `:${lineNumber}` : ""}"`;
```

Когда CodeLens передает `[filePath, lineNumber, scenarioName]` для обычного сценария (не Scenario Outline), код строит команду вида:

```bash
behave "file.feature:4" --name="Run a simple test"
```

**Проблема**: Behave игнорирует `:lineNumber` для обычных сценариев и запускает все сценарии в файле, если `--name` не совпадает точно.

### Решение

**В `[src/core/test-executor.ts](src/core/test-executor.ts)`, метод `runScenario()`**:

1. **Упростить логику команды** (строки 85-95):
  - Убрать использование `:lineNumber` в пути к файлу для обычных сценариев
  - Всегда использовать только `--name` для фильтрации конкретного сценария
  - Формат команды: `behave "file.feature" --name="Scenario Name"`
2. **Исправить условие** (строки 59-83):
  - Текущая логика проверяет `!isScenarioOutlineExample`, что означает "это Scenario Outline, но не пример"
  - Эта ветка должна выполняться только для Scenario Outline (не примеров)
  - Проблема: код не различает обычный сценарий от Scenario Outline
3. **Добавить проверку типа сценария**:
  - Нужно определить, является ли сценарий обычным или Scenario Outline
  - Для обычного сценария: использовать `--name` без `:lineNumber`
  - Для Scenario Outline: использовать существующую логику

## Детальный план изменений

### Изменение 1: Обновление CodeLens при изменении локали

**Файл**: `[src/extension.ts](src/extension.ts)`

- Добавить `EventEmitter<void>` для CodeLens провайдера
- Подписаться на изменение конфигурации `behaveTestRunner.uiLocale`
- При изменении вызывать `eventEmitter.fire()` для обновления CodeLens

### Изменение 2: Исправление логики выполнения сценария

**Файл**: `[src/core/test-executor.ts](src/core/test-executor.ts)`

**Текущая логика** (строки 59-95):

```typescript
if (scenarioName && !isScenarioOutlineExample && filePath && fs.existsSync(filePath)) {
  // Run with --name only
  let command = `${behaveCommand} "${filePath}" --name="${scenarioName}"`;
  // ...
  return;
}

let command = `${behaveCommand} "${filePath}${lineNumber ? `:${lineNumber}` : ""}"`;
if (scenarioName) {
  if (isScenarioOutlineExample) {
    command += ` --name="${originalOutlineName}"`;
  } else {
    command += ` --name="${scenarioName}"`;
  }
}
```

**Проблема**: Первая ветка (строки 59-83) срабатывает для **обычных сценариев** (когда `!isScenarioOutlineExample` = true), но это неправильно. Она должна срабатывать только для Scenario Outline.

**Новая логика**:

```typescript
// Определить тип сценария
const isScenarioOutline = this.isScenarioOutline(filePath, lineNumber, scenarioName);

if (isScenarioOutline && !isScenarioOutlineExample) {
  // Scenario Outline (не пример) - запустить все примеры
  let command = `${behaveCommand} "${filePath}" --name="${scenarioName}"`;
  // ...
  return;
}

// Обычный сценарий или пример Scenario Outline
let command = `${behaveCommand} "${filePath}"`;
if (scenarioName) {
  if (isScenarioOutlineExample) {
    const originalOutlineName = this.extractOriginalOutlineName(scenarioName);
    command += ` --name="${originalOutlineName}"`;
  } else {
    command += ` --name="${scenarioName}"`;
  }
}
```

### Изменение 3: Добавить метод проверки типа сценария

**Файл**: `[src/core/test-executor.ts](src/core/test-executor.ts)`

Добавить метод `isScenarioOutline()` (аналогично методу в `command-manager.ts`, строки 351-409).

## Тестирование

После внесения изменений необходимо:

1. Проверить, что при изменении `behaveTestRunner.uiLocale` на `ru`, CodeLens обновляются и показывают русский текст
2. Проверить, что запуск первого сценария через CodeLens "Run Test" выполняет только этот сценарий
3. Проверить, что запуск Scenario Outline выполняет все примеры
4. Проверить, что запуск примера Scenario Outline выполняет только этот пример
5. Проверить, что команда `runFeatureFile` продолжает работать корректно


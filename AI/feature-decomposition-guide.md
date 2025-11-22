# Гайд по декомпозиции фич в Atomic Framework

## Введение

Этот гайд описывает универсальную методологию декомпозиции и внедрения фич (features) в проектах на Atomic Framework. Применим к любому проекту - от простых прототипов до сложных игр с тысячами entities.

**Ключевая идея:** Каждая фича разбивается на 7 шагов внедрения через **Installers → Behaviours → UseCases**.

---

## Философия декомпозиции

### Принцип "от простого к сложному"

Фичи классифицируются по 4 уровням сложности:

```
Уровень 1: Foundation (⭐ Простые)
- Базовые data components
- Без логики, только хранение данных
- Примеры: Transform, Position, Health, Money

Уровень 2: Core Mechanics (⭐⭐ Средние)
- Простая логика с одним lifecycle методом
- Читают/пишут foundation компоненты
- Примеры: Movement, Input, Rotation

Уровень 3: Advanced Mechanics (⭐⭐⭐ Сложные)
- Комбинируют несколько компонентов
- Используют события и подписки
- Примеры: Collision Detection, Item Collection, AI

Уровень 4: Game Architecture (⭐⭐⭐⭐ Очень сложные)
- Координируют множество систем
- Используют context hierarchy
- Примеры: Game Manager, Level System, Multiplayer Sync
```

### Правило зависимостей

```
Фича уровня N может зависеть только от фич уровня N-1 или ниже.

✅ Правильно:
Movement (уровень 2) зависит от Transform (уровень 1)

❌ Неправильно:
Transform (уровень 1) зависит от Movement (уровень 2)
```

---

## 7-шаговый процесс внедрения фичи

Каждая фича внедряется по единому алгоритму:

### Шаг 1: Добавление в EntityAPI

**Что делать:**
- Создать static readonly int для ID компонента
- Зарегистрировать ID в static конструкторе
- Создать extension методы (Get/Add/Has/Del)

**Шаблон для Value:**

```csharp
public static class EntityAPI
{
    // 1. Объявить ID
    public static readonly int ComponentName; // Type

    static EntityAPI()
    {
        // 2. Зарегистрировать
        ComponentName = NameToId(nameof(ComponentName));
    }

    // 3. Extension методы

    /// <summary>
    /// Получает ComponentName из entity.
    /// </summary>
    [MethodImpl(MethodImplOptions.AggressiveInlining)]
    public static Type GetComponentName(this IEntity entity)
        => entity.GetValue<Type>(ComponentName);

    /// <summary>
    /// Добавляет ComponentName в entity.
    /// </summary>
    [MethodImpl(MethodImplOptions.AggressiveInlining)]
    public static void AddComponentName(this IEntity entity, Type value)
        => entity.AddValue(ComponentName, value);

    /// <summary>
    /// Проверяет наличие ComponentName в entity.
    /// </summary>
    [MethodImpl(MethodImplOptions.AggressiveInlining)]
    public static bool HasComponentName(this IEntity entity)
        => entity.HasValue(ComponentName);

    /// <summary>
    /// Удаляет ComponentName из entity.
    /// </summary>
    [MethodImpl(MethodImplOptions.AggressiveInlining)]
    public static bool DelComponentName(this IEntity entity)
        => entity.DelValue(ComponentName);
}
```

**Шаблон для Tag:**

```csharp
public static class EntityAPI
{
    public static readonly int TagName; // Tag

    static EntityAPI()
    {
        TagName = NameToId(nameof(TagName));
    }

    /// <summary>
    /// Проверяет наличие TagName тега.
    /// </summary>
    [MethodImpl(MethodImplOptions.AggressiveInlining)]
    public static bool HasTagName(this IEntity entity)
        => entity.HasTag(TagName);

    /// <summary>
    /// Добавляет TagName тег.
    /// </summary>
    [MethodImpl(MethodImplOptions.AggressiveInlining)]
    public static bool AddTagName(this IEntity entity)
        => entity.AddTag(TagName);

    /// <summary>
    /// Удаляет TagName тег.
    /// </summary>
    [MethodImpl(MethodImplOptions.AggressiveInlining)]
    public static bool DelTagName(this IEntity entity)
        => entity.DelTag(TagName);
}
```

**Когда использовать Tag vs Value:**
- **Tag**: Для идентификации типа entity (Enemy, Coin, Player)
- **Value**: Для хранения данных (Health, Position, Speed)

---

### Шаг 2: Создание Data Classes

**Что делать:**
- Создать классы для хранения данных
- Пометить [Serializable] если нужна настройка в Inspector
- Использовать интерфейсы для flexibility

**Простые data классы:**

```csharp
using System;
using UnityEngine;

/// <summary>
/// Данные для фичи X.
/// Serializable для настройки в Inspector.
/// </summary>
[Serializable]
public sealed class FeatureData
{
    [Tooltip("Описание параметра")]
    public float parameter1 = 10f;

    [Tooltip("Другой параметр")]
    public int parameter2 = 5;
}
```

**Реактивные данные:**

```csharp
using Atomic.Elements;

// Для UI обновлений используем ReactiveVariable
[Serializable]
public class ReactiveFeatureData
{
    private readonly ReactiveVariable<int> _value = new();

    public int Value
    {
        get => _value.Value;
        set => _value.Value = value;
    }

    public Subscription<int> Observe(Action<int> action) => _value.Observe(action);
}
```

**ScriptableObject для конфигураций:**

```csharp
using UnityEngine;

[CreateAssetMenu(fileName = "FeatureConfig", menuName = "Game/FeatureConfig")]
public sealed class FeatureConfig : ScriptableObject
{
    public float speed = 5f;
    public int maxHealth = 100;
    // ... другие настройки
}
```

**Когда использовать:**
- **[Serializable] class**: Для данных, уникальных для каждой entity
- **ReactiveVariable**: Когда нужна подписка (обычно для UI)
- **ScriptableObject**: Для переиспользуемых конфигураций

---

### Шаг 3: Создание UseCases

**Что делать:**
- Вынести чистую бизнес-логику в static методы
- Опционально - можно оставить логику в Behaviour
- UseCases должны быть stateless (без полей экземпляра)

**Шаблон UseCase:**

```csharp
/// <summary>
/// UseCase для фичи X.
/// Содержит чистую бизнес-логику без состояния.
/// </summary>
public static class FeatureUseCase
{
    /// <summary>
    /// Выполняет операцию X.
    /// </summary>
    /// <param name="param1">Описание параметра</param>
    /// <returns>Результат операции</returns>
    public static ReturnType PerformOperation(ParamType param1)
    {
        // Чистая логика без side effects
        // Легко тестируется
        // Может переиспользоваться в разных контекстах
        return result;
    }

    /// <summary>
    /// Вспомогательный private метод.
    /// </summary>
    private static void HelperMethod()
    {
        // Внутренняя логика
    }
}
```

**Burst-compiled UseCase (для performance):**

```csharp
using Unity.Burst;
using Unity.Mathematics;

[BurstCompile]
public static class FeatureUseCase
{
    /// <summary>
    /// Burst-compiled метод для высокой производительности.
    /// Использует Unity.Mathematics вместо UnityEngine.
    /// </summary>
    [BurstCompile]
    public static void PerformHeavyCalculation(
        in float3 input,
        out float3 result)
    {
        // Только blittable типы!
        result = math.normalize(input);
    }
}
```

**Когда создавать UseCase:**
- Логика переиспользуется в нескольких местах
- Нужна высокая performance (Burst)
- Логика сложная и требует тестирования
- AI, pathfinding, физика

**Когда НЕ создавать UseCase:**
- Логика простая (1-2 строки)
- Используется только в одном Behaviour
- Требует много контекста (лучше держать в Behaviour)

---

### Шаг 4: Создание Behaviours

**Что делать:**
- Создать класс, реализующий нужные lifecycle интерфейсы
- Кешировать компоненты в Init
- Реализовать логику в соответствующих методах

**Lifecycle интерфейсы:**

```csharp
public interface IEntityInit      // Один раз при создании
public interface IEntityEnable    // При активации (многократно)
public interface IEntityTick      // Каждый Update
public interface IEntityFixedTick // Каждый FixedUpdate
public interface IEntityLateTick  // Каждый LateUpdate
public interface IEntityDisable   // При деактивации (многократно)
public interface IEntityDispose   // Один раз при уничтожении
public interface IEntityGizmos    // Визуализация в Editor
```

**Шаблон Behaviour:**

```csharp
using Atomic.Entities;
using UnityEngine;

/// <summary>
/// Behaviour для фичи X.
/// Координирует компоненты и вызывает UseCases.
/// </summary>
/// <remarks>
/// Lifecycle:
/// - Init: кешируем компоненты
/// - Tick: основная логика
/// </remarks>
public sealed class FeatureBehaviour : IEntityInit, IEntityTick
{
    // Кешированные компоненты
    private ComponentType1 _component1;
    private ComponentType2 _component2;

    /// <summary>
    /// Init: кешируем ссылки на компоненты.
    /// Вызывается один раз при создании entity.
    /// </summary>
    public void Init(IEntity entity)
    {
        // Кешируем все нужные компоненты
        _component1 = entity.GetComponent1();
        _component2 = entity.GetComponent2();

        // Можно сделать дополнительную инициализацию
    }

    /// <summary>
    /// Tick: основная логика behaviour.
    /// Вызывается каждый кадр (Update).
    /// </summary>
    public void Tick(IEntity entity, float deltaTime)
    {
        // Читаем данные
        var data = _component1.GetData();

        // Вызываем UseCase (если есть)
        var result = FeatureUseCase.PerformOperation(data);

        // Записываем результат
        _component2.SetData(result);
    }
}
```

**Behaviour с событиями:**

```csharp
public sealed class FeatureBehaviour : IEntityInit, IEntityEnable, IEntityDisable
{
    private EventProvider _events;

    public void Init(IEntity entity)
    {
        _events = entity.GetEventProvider();
    }

    /// <summary>
    /// Enable: подписываемся на события.
    /// Может вызываться многократно.
    /// </summary>
    public void Enable(IEntity entity)
    {
        _events.OnEventHappened += this.HandleEvent;
    }

    /// <summary>
    /// Disable: отписываемся от событий.
    /// Может вызываться многократно.
    /// </summary>
    public void Disable(IEntity entity)
    {
        _events.OnEventHappened -= this.HandleEvent;
    }

    private void HandleEvent(EventArgs args)
    {
        // Обработка события
    }
}
```

**Inline Behaviour (лямбда):**

```csharp
// В Installer можно использовать inline behaviours
entity.WhenTick(deltaTime =>
{
    // Простая логика без отдельного класса
    transform.position += direction * speed * deltaTime;
});

entity.WhenFixedTick(deltaTime =>
{
    // Физика
});

entity.WhenEnable(() =>
{
    // При активации
});
```

**Когда использовать какой подход:**
- **Class Behaviour**: Когда логика сложная, нужно кеширование
- **Inline Lambda**: Когда логика простая (1-3 строки)

---

### Шаг 5: Создание Installer

**Что делать:**
- Создать класс, наследующийся от SceneEntityInstaller
- В методе Install добавить все компоненты и behaviours
- Настроить SerializeFields для Inspector

**Шаблон Installer:**

```csharp
using Atomic.Elements;
using Atomic.Entities;
using UnityEngine;

/// <summary>
/// Устанавливает компоненты для entity X.
/// </summary>
public sealed class FeatureInstaller : SceneEntityInstaller
{
    [Header("Feature Configuration")]
    [Tooltip("Описание параметра")]
    [SerializeField]
    private Const<float> _parameter = 10f;

    [SerializeField]
    private FeatureData _featureData;

    [SerializeField]
    private FeatureConfig _featureConfig;

    /// <summary>
    /// Install: композиция entity из компонентов и behaviours.
    /// Вызывается один раз при создании entity.
    /// </summary>
    public override void Install(IEntity entity)
    {
        // === 1. Базовые компоненты ===
        entity.AddTransform(this.transform);

        // === 2. Data компоненты ===
        entity.AddParameter(_parameter);
        entity.AddFeatureData(_featureData);
        entity.AddFeatureConfig(_featureConfig);

        // === 3. Behaviours ===
        entity.AddBehaviour<FeatureBehaviour>();

        // === 4. Inline behaviours (опционально) ===
        entity.WhenTick(deltaTime =>
        {
            // Простая логика
        });
    }
}
```

**Modular Installer (для переиспользования):**

```csharp
using System;
using Atomic.Entities;
using UnityEngine;

/// <summary>
/// Модульный установщик для фичи X.
/// Может использоваться в разных entities.
/// </summary>
[Serializable]
public sealed class FeatureModularInstaller : IEntityInstaller<IEntity>
{
    [SerializeField]
    private Const<float> _parameter = 10f;

    /// <summary>
    /// Установка фичи.
    /// </summary>
    public void Install(IEntity entity)
    {
        entity.AddParameter(_parameter);
        entity.AddBehaviour<FeatureBehaviour>();
    }
}

// Использование в главном Installer:
public sealed class MainInstaller : SceneEntityInstaller
{
    [SerializeField]
    private FeatureModularInstaller _featureInstaller;

    public override void Install(IEntity entity)
    {
        _featureInstaller.Install(entity);
        // другие модули...
    }
}
```

**Порядок добавления компонентов:**
1. **Базовые** (Transform, Rigidbody)
2. **Data** (Values, ReactiveVariables)
3. **Behaviours** (от простых к сложным)
4. **Inline callbacks** (WhenTick, WhenEnable)

---

### Шаг 6: Интеграция с другими фичами

**Что делать:**
- Описать как фича взаимодействует с другими
- Показать data flow
- Указать события и подписки

**Шаблон описания интеграции:**

```markdown
### Интеграция

**Data Flow:**
```
InputBehaviour (Producer)
    ↓ writes to
MovementDirection (Shared State)
    ↓ reads from
MovementBehaviour (Consumer)
```

**Используется в:**
- FeatureA читает данные этой фичи
- FeatureB подписывается на события
- FeatureC вызывает UseCases

**Используемые компоненты из других фич:**
- Transform (Feature 1)
- Health (Feature 5)
- TriggerEvents (Feature 8)

**События:**
- OnFeatureActivated - вызывается когда...
- OnFeatureCompleted - вызывается когда...
```

**Паттерны интеграции:**

1. **Producer-Consumer:**
```csharp
// Producer пишет
entity.GetSharedData().Value = newValue;

// Consumer читает
var data = entity.GetSharedData().Value;
```

2. **Event-based:**
```csharp
// Producer генерирует событие
_events.OnSomethingHappened?.Invoke(args);

// Consumer подписывается
_events.OnSomethingHappened += HandleEvent;
```

3. **Request-Response:**
```csharp
// Requester отправляет запрос
entity.GetRequest().Send(requestData);

// Responder обрабатывает
if (entity.GetRequest().Consume(out var data))
    ProcessRequest(data);
```

---

### Шаг 7: Указание зависимостей и Best Practices

**Что делать:**
- Перечислить все зависимости фичи
- Описать Best Practices для этой фичи
- Добавить примеры правильного и неправильного использования

**Шаблон:**

```markdown
### Зависимости

**Прямые зависимости:**
- Transform (Feature 1) - для позиционирования
- Health (Feature 5) - для проверки жизни

**Косвенные зависимости:**
- Требует Collider на GameObject
- Требует Layer Mask настройку

### Best Practices

1. **Название практики:**
   ```csharp
   // ✅ Правильно - объяснение почему
   GoodExample();

   // ❌ Неправильно - объяснение почему плохо
   BadExample();
   ```

2. **Другая практика:**
   - Детали
   - Примеры

### Частые ошибки

1. **Ошибка X**: Описание и как избежать
2. **Ошибка Y**: Описание и как избежать
```

---

## Уровни сложности: детальное описание

### Уровень 1: Foundation (⭐ Простые)

**Характеристики:**
- Только data, нет логики
- 1 компонент в EntityAPI
- Нет Behaviours
- Нет UseCases

**Примеры:**
- Transform, Position, Rotation
- Health, MaxHealth, Shield
- Speed, Acceleration, JumpForce
- Money, Score, Experience
- Team, PlayerId, Level

**Шаблон:**

```csharp
// EntityAPI
public static readonly int ComponentName; // Type
public static Type GetComponentName(this IEntity entity) => ...;
public static void AddComponentName(this IEntity entity, Type value) => ...;

// Data Class (опционально, если не базовый тип)
[Serializable]
public class ComponentData
{
    public float value1;
    public int value2;
}

// Installer
public override void Install(IEntity entity)
{
    entity.AddComponentName(new ComponentData());
}

// Нет Behaviours
// Нет UseCases
```

---

### Уровень 2: Core Mechanics (⭐⭐ Средние)

**Характеристики:**
- 1-2 компонента в EntityAPI
- 1 Behaviour с 1-2 lifecycle методами
- UseCases опционально
- Зависит от 1-2 foundation фич

**Примеры:**
- Movement (зависит от Transform, Speed, Direction)
- Input (пишет в Direction)
- Rotation (читает RotationSpeed, модифицирует Rotation)
- Timer/Cooldown системы
- Simple AI (patrol, follow)

**Шаблон:**

```csharp
// EntityAPI (1-2 компонента)
public static readonly int Component1;
public static readonly int Component2;

// Data Class (если нужно)
[Serializable]
public class FeatureData { }

// UseCase (опционально)
public static class FeatureUseCase
{
    public static void DoSomething(params) { }
}

// Behaviour (1-2 lifecycle метода)
public sealed class FeatureBehaviour : IEntityInit, IEntityTick
{
    private Comp1 _comp1;
    private Comp2 _comp2;

    public void Init(IEntity entity)
    {
        _comp1 = entity.GetComp1();
        _comp2 = entity.GetComp2();
    }

    public void Tick(IEntity entity, float deltaTime)
    {
        // Простая логика
        var result = FeatureUseCase.DoSomething(_comp1.Value);
        _comp2.Value = result;
    }
}

// Installer
public override void Install(IEntity entity)
{
    entity.AddComp1(data1);
    entity.AddComp2(data2);
    entity.AddBehaviour<FeatureBehaviour>();
}
```

---

### Уровень 3: Advanced Mechanics (⭐⭐⭐ Сложные)

**Характеристики:**
- 2-4 компонента в EntityAPI
- 1-2 Behaviours с 3-5 lifecycle методами
- UseCases часто используются
- События и подписки
- Зависит от 3-5 других фич

**Примеры:**
- Collision Detection & Response
- Inventory System
- Weapon System с перезарядкой
- AI с FSM (Finite State Machine)
- Spawning System
- Quest System

**Шаблон:**

```csharp
// EntityAPI (2-4 компонента + возможно теги)
public static readonly int Component1;
public static readonly int Component2;
public static readonly int Component3;
public static readonly int FeatureTag;

// Data Classes
[Serializable]
public class FeatureConfig { }

[Serializable]
public class FeatureState { }

// UseCases (обычно несколько)
public static class FeatureUseCase
{
    public static void Operation1(params) { }
    public static bool Operation2(params) { }
    private static void Helper(params) { }
}

// Behaviour (3-5 lifecycle методов, события)
public sealed class FeatureBehaviour : IEntityInit, IEntityEnable, IEntityTick, IEntityDisable, IEntityDispose
{
    private EventProvider _events;
    private Comp1 _comp1;
    private Comp2 _comp2;
    private FeatureState _state;

    public void Init(IEntity entity)
    {
        // Кеширование
        _events = entity.GetEvents();
        _comp1 = entity.GetComp1();
        _comp2 = entity.GetComp2();
        _state = entity.GetState();
    }

    public void Enable(IEntity entity)
    {
        // Подписка на события
        _events.OnEvent1 += HandleEvent1;
        _events.OnEvent2 += HandleEvent2;
    }

    public void Tick(IEntity entity, float deltaTime)
    {
        // Основная логика
        if (FeatureUseCase.Operation2(_comp1.Value))
        {
            _comp2.Value = FeatureUseCase.Operation1(_state);
        }
    }

    public void Disable(IEntity entity)
    {
        // Отписка от событий
        _events.OnEvent1 -= HandleEvent1;
        _events.OnEvent2 -= HandleEvent2;
    }

    public void Dispose(IEntity entity)
    {
        // Очистка ресурсов
    }

    private void HandleEvent1(EventArgs args) { }
    private void HandleEvent2(EventArgs args) { }
}

// Installer (композиция)
public override void Install(IEntity entity)
{
    // Данные
    entity.AddComponent1(config.data1);
    entity.AddComponent2(config.data2);
    entity.AddComponent3(new FeatureState());
    entity.AddFeatureTag();

    // Behaviours
    entity.AddBehaviour<FeatureBehaviour>();

    // События
    entity.AddEvents(new EventProvider());
}
```

---

### Уровень 4: Game Architecture (⭐⭐⭐⭐ Очень сложные)

**Характеристики:**
- 5+ компонентов в EntityAPI
- 3+ Behaviours
- Множество UseCases
- Context hierarchy
- Координация множества систем
- Зависит от большинства других фич

**Примеры:**
- Game Manager / Game Context
- Level System
- Save/Load System
- Multiplayer Synchronization
- Analytics System
- Tutorial System

**Шаблон:**

```csharp
// EntityAPI (много компонентов)
public static readonly int GameContext; // Tag
public static readonly int Players; // IDictionary<PlayerId, IEntity>
public static readonly int GameState; // GameStateEnum
public static readonly int GameTimer; // ICooldown
public static readonly int SpawningSystem; // SpawnConfig
// ... и так далее

// Data Classes (много)
public enum GameState { Menu, Playing, Paused, GameOver }

[Serializable]
public class PlayerInfo
{
    public PlayerId id;
    public SceneEntity entity;
}

[Serializable]
public class GameConfig { }

// UseCases (много, разбиты по областям)
public static class GameManagementUseCase
{
    public static void StartGame(IEntity gameContext) { }
    public static void PauseGame(IEntity gameContext) { }
    public static void EndGame(IEntity gameContext) { }
}

public static class PlayerManagementUseCase
{
    public static IEntity GetPlayer(IDictionary<PlayerId, IEntity> players, PlayerId id) { }
    public static void DisableAllPlayers(IDictionary<PlayerId, IEntity> players) { }
}

public static class GameStateUseCase
{
    public static void TransitionTo(GameState newState) { }
    public static bool CanTransition(GameState from, GameState to) { }
}

// Behaviours (несколько, каждый отвечает за свою область)
public sealed class GameStateBehaviour : IEntityInit, IEntityTick
{
    // Управление состоянием игры
}

public sealed class GameTimerBehaviour : IEntityInit, IEntityFixedTick
{
    // Управление таймером
}

public sealed class GameOverBehaviour : IEntityInit, IEntityDispose
{
    // Логика завершения игры
}

// Installer (Composition Root - собирает всё)
public sealed class GameContextInstaller : SceneEntityInstaller
{
    [Header("Players")]
    [SerializeField] private PlayerInfo[] _players;

    [Header("Game Config")]
    [SerializeField] private GameConfig _config;

    [Header("Timer")]
    [SerializeField] private Cooldown _gameTimer;

    [Header("Spawning")]
    [SerializeField] private SpawnConfig _spawnConfig;

    public override void Install(IEntity context)
    {
        // Tag
        context.AddGameContextTag();

        // === 1. Players System ===
        var players = new Dictionary<PlayerId, IEntity>();
        foreach (var info in _players)
            players.Add(info.id, info.entity);
        context.AddPlayers(players);

        // === 2. Game State System ===
        context.AddGameState(GameState.Menu);
        context.AddBehaviour<GameStateBehaviour>();

        // === 3. Timer System ===
        context.AddGameTimer(_gameTimer);
        context.WhenTick(_gameTimer.Tick);
        context.AddBehaviour<GameTimerBehaviour>();

        // === 4. Spawning System ===
        context.AddSpawningSystem(_spawnConfig);
        context.AddBehaviour<SpawningBehaviour>();

        // === 5. Game Over System ===
        context.AddBehaviour<GameOverBehaviour>();
    }
}
```

---

## Практические шаблоны

### Шаблон 1: Простая data фича (Health)

```csharp
// === EntityAPI ===
public static readonly int Health; // IReactiveVariable<int>
public static readonly int MaxHealth; // IValue<int>

public static IReactiveVariable<int> GetHealth(this IEntity entity) => ...;
public static IValue<int> GetMaxHealth(this IEntity entity) => ...;

// === Installer ===
public override void Install(IEntity entity)
{
    entity.AddMaxHealth(new Const<int>(100));
    entity.AddHealth(new ReactiveVariable<int>(100));
}

// === UI Integration ===
public class HealthView : MonoBehaviour
{
    private Subscription<int> _sub;

    private void Start()
    {
        IReactiveVariable<int> health = _entity.GetHealth();
        _sub = health.Observe(OnHealthChanged);
    }

    private void OnDestroy()
    {
        _sub?.Dispose();
    }

    private void OnHealthChanged(int value)
    {
        _healthBar.fillAmount = value / (float)_entity.GetMaxHealth().Value;
    }
}
```

---

### Шаблон 2: Movement с Input (Producer-Consumer)

```csharp
// === EntityAPI ===
public static readonly int MovementDirection; // IVariable<Vector3>
public static readonly int MovementSpeed; // IValue<float>

// === Input Behaviour (Producer) ===
public sealed class InputBehaviour : IEntityInit, IEntityTick
{
    private IVariable<Vector3> _direction;

    public void Init(IEntity entity)
    {
        _direction = entity.GetMovementDirection();
    }

    public void Tick(IEntity entity, float deltaTime)
    {
        Vector3 input = new Vector3(Input.GetAxis("Horizontal"), 0, Input.GetAxis("Vertical"));
        _direction.Value = input; // Пишем
    }
}

// === Movement Behaviour (Consumer) ===
public sealed class MovementBehaviour : IEntityInit, IEntityFixedTick
{
    private Transform _transform;
    private IValue<Vector3> _direction;
    private IValue<float> _speed;

    public void Init(IEntity entity)
    {
        _transform = entity.GetTransform();
        _direction = entity.GetMovementDirection();
        _speed = entity.GetMovementSpeed();
    }

    public void FixedTick(IEntity entity, float deltaTime)
    {
        Vector3 dir = _direction.Value; // Читаем
        _transform.position += dir * _speed.Value * deltaTime;
    }
}

// === Installer ===
public override void Install(IEntity entity)
{
    entity.AddTransform(this.transform);
    entity.AddMovementSpeed(new Const<float>(5f));
    entity.AddMovementDirection(new Variable<Vector3>());

    entity.AddBehaviour<InputBehaviour>();
    entity.AddBehaviour<MovementBehaviour>();
}
```

---

### Шаблон 3: Trigger-based Collection

```csharp
// === EntityAPI ===
public static readonly int TriggerEvents; // TriggerEvents
public static readonly int Money; // IReactiveVariable<int>
public static readonly int Coin; // Tag

// === Trigger Events (Unity MonoBehaviour) ===
public class TriggerEvents : MonoBehaviour
{
    public event Action<Collider> OnEntered;
    public event Action<Collider> OnExited;

    private void OnTriggerEnter(Collider other) => OnEntered?.Invoke(other);
    private void OnTriggerExit(Collider other) => OnExited?.Invoke(other);
}

// === Collection Behaviour ===
public sealed class CoinCollectBehaviour : IEntityInit, IEntityEnable, IEntityDisable
{
    private TriggerEvents _trigger;
    private IVariable<int> _money;

    public void Init(IEntity entity)
    {
        _trigger = entity.GetTriggerEvents();
        _money = entity.GetMoney();
    }

    public void Enable(IEntity entity)
    {
        _trigger.OnEntered += OnTriggerEntered;
    }

    public void Disable(IEntity entity)
    {
        _trigger.OnEntered -= OnTriggerEntered;
    }

    private void OnTriggerEntered(Collider collider)
    {
        if (!collider.TryGetComponent(out IEntity coin))
            return;

        if (!coin.HasCoinTag())
            return;

        int value = coin.GetMoney().Value;
        _money.Value += value;

        SceneEntity.Destroy(coin);
    }
}

// === Installer (Character) ===
public override void Install(IEntity entity)
{
    entity.AddTriggerEvents(_triggerEvents); // SerializeField
    entity.AddMoney(new ReactiveVariable<int>(0));
    entity.AddBehaviour<CoinCollectBehaviour>();
}

// === Installer (Coin) ===
public override void Install(IEntity entity)
{
    entity.AddCoinTag();
    entity.AddTransform(this.transform);
    entity.AddMoney(new ReactiveVariable<int>(1)); // Стоимость
}
```

---

## Best Practices по категориям

### Lifecycle Methods

**Init vs Enable:**
```csharp
// ✅ Init - для кеширования (один раз)
public void Init(IEntity entity)
{
    _component = entity.GetComponent();
}

// ✅ Enable - для подписки на события (многократно)
public void Enable(IEntity entity)
{
    _events.OnEvent += HandleEvent;
}
```

**Tick vs FixedTick:**
```csharp
// ✅ Tick - для input, UI, AI
public void Tick(IEntity entity, float deltaTime)
{
    Vector3 input = ReadInput();
}

// ✅ FixedTick - для физики, движения
public void FixedTick(IEntity entity, float deltaTime)
{
    _rigidbody.MovePosition(newPosition);
}
```

**Dispose - всегда отписывайтесь:**
```csharp
// ✅ Правильно
public void Init(IEntity entity)
{
    _countdown.OnCompleted += OnCompleted;
}

public void Dispose(IEntity entity)
{
    _countdown.OnCompleted -= OnCompleted; // ВАЖНО!
}
```

### Performance

**Кеширование в Init:**
```csharp
// ✅ Правильно - кешируем в Init
private Transform _transform;

public void Init(IEntity entity)
{
    _transform = entity.GetTransform();
}

public void Tick(IEntity entity, float deltaTime)
{
    _transform.position += Vector3.forward;
}

// ❌ Неправильно - получаем каждый кадр
public void Tick(IEntity entity, float deltaTime)
{
    Transform t = entity.GetTransform(); // Медленно!
    t.position += Vector3.forward;
}
```

**Burst Compilation для тяжелых вычислений:**
```csharp
// ✅ Использовать Burst для math-heavy операций
[BurstCompile]
public static class MoveUseCase
{
    [BurstCompile]
    public static void MoveStep(
        in float3 position,
        in float3 direction,
        in float speed,
        in float deltaTime,
        out float3 result)
    {
        result = position + speed * deltaTime * direction;
    }
}
```

### Type Safety

**Tags вместо строк:**
```csharp
// ✅ Правильно - type-safe
if (entity.HasEnemyTag())
    Attack(entity);

// ❌ Неправильно - легко ошибиться
if (entity.CompareTag("Enemi")) // Опечатка!
    Attack(entity);
```

**Enum для типов:**
```csharp
// ✅ Правильно - компилятор проверяет
TeamType team = TeamType.RED;

// ❌ Неправильно - runtime ошибки
string team = "red"; // Lowercase? Uppercase?
```

### Reactive Programming

**Обязательный Dispose подписок:**
```csharp
// ✅ Правильно
private Subscription<int> _sub;

private void Start()
{
    _sub = variable.Observe(OnChanged);
}

private void OnDestroy()
{
    _sub?.Dispose(); // ОБЯЗАТЕЛЬНО!
}

// ❌ Неправильно - memory leak
private void Start()
{
    variable.Observe(OnChanged); // Никогда не отписываемся
}
```

---
 
 ## Advanced Patterns Implementation
 
 Для фич уровня 4 (Architecture) требуются более сложные паттерны. Ниже приведены шаблоны для ключевых архитектурных элементов.
 
 ### Pattern 1: EntityFilter (Dynamic Selections)
 
 Используется для динамической выборки entities по условиям (например, "все враги в радиусе").
 
 **Шаблон:**
 
 ```csharp
 // 1. Создание фильтра (обычно в Builder или Installer)
 private EntityFilter<IUnitEntity> CreateFilter(IEntityWorld world, IContext context)
 {
     return new EntityFilter<IUnitEntity>(
         world,
         entity => IsTargetValid(entity, context), // Предикат
         new TeamEntityTrigger(),                  // Триггер 1: смена команды
         new TagEntityTrigger<IUnitEntity>()       // Триггер 2: смена тега
     );
 }
 
 // 2. Предикат фильтрации (в UseCase)
 public static bool IsTargetValid(IUnitEntity entity, IContext context)
 {
     return entity.HasUnitTag() &&                 // Проверка тега
            !entity.HasTargetedTag() &&            // Проверка состояния
            entity.GetTeam().Value != context.GetTeam().Value; // Проверка данных
 }
 
 // 3. Использование (в UseCase или Behaviour)
 public static IUnitEntity FindTarget(EntityFilter<IUnitEntity> filter, Vector3 position)
 {
     // Фильтр автоматически содержит только подходящие entities
     foreach (var entity in filter)
     {
         // Поиск ближайшего
     }
 }
 ```
 
 ---
 
 ### Pattern 2: Factory Pattern (Entity Creation)
 
 Используется для создания сложных entities с множеством компонентов.
 
 **Шаблон Factory:**
 
 ```csharp
 // Абстрактная фабрика
 public abstract class UnitFactory : ScriptableEntityFactory<IUnitEntity>
 {
     public string Name => this.name;
 
     public sealed override IUnitEntity Create()
     {
         // 1. Создание entity
         var entity = new UnitEntity(this.Name);
         
         // 2. Установка компонентов (Template Method)
         this.Install(entity);
         
         return entity;
     }
 
     protected abstract void Install(IUnitEntity entity);
 }
 
 // Конкретная фабрика
 [CreateAssetMenu]
 public sealed class WarriorFactory : UnitFactory
 {
     [SerializeField] private MoveInstaller _moveInstaller;
     [SerializeField] private CombatInstaller _combatInstaller;
 
     protected override void Install(IUnitEntity entity)
     {
         // Композиция через Installers
         entity.Install(_moveInstaller);
         entity.Install(_combatInstaller);
         
         // Дополнительная настройка
         entity.AddUnitTag();
     }
 }
 ```
 
 ---
 
 ### Pattern 3: Context Integration (Hierarchy)
 
 Используется для связывания дочернего контекста (Player) с родительским (Game).
 
 **Шаблон Installer:**
 
 ```csharp
 public sealed class PlayerContextInstaller : SceneEntityInstaller<IPlayerContext>
 {
     public override void Install(IPlayerContext context)
     {
         // 1. Интеграция с родительским контекстом
         this.InstallGameContext(context);
 
         // 2. Установка локальных систем
         context.Install(_localSystems);
     }
 
     private void InstallGameContext(IPlayerContext context)
     {
         // Попытка получить родительский контекст
         if (!GameContext.TryGetInstance(out GameContext gameContext))
             return;
 
         // Регистрация в родительском контексте
         if (gameContext.TryGetPlayers(out var players))
             players.TryAdd(context.GetTeamType().Value, context);
 
         // Связывание жизненного цикла (важно!)
         gameContext.WhenDisable(context.Disable);
     }
 }
 ```
 
 ---

## Чеклист для новой фичи

Перед тем как считать фичу завершенной, проверьте:

- [ ] **Шаг 1: EntityAPI**
  - [ ] ID зарегистрирован в static конструкторе
  - [ ] Extension методы созданы (Get/Add/Has/Del)
  - [ ] XML комментарии добавлены
  - [ ] AggressiveInlining атрибуты есть

- [ ] **Шаг 2: Data Classes**
  - [ ] [Serializable] если нужно в Inspector
  - [ ] Tooltip атрибуты для полей
  - [ ] XML комментарии для классов
  - [ ] Используются интерфейсы (IValue, IVariable)

- [ ] **Шаг 3: UseCases**
  - [ ] Методы static
  - [ ] Нет полей экземпляра (stateless)
  - [ ] XML комментарии
  - [ ] [BurstCompile] если нужна performance

- [ ] **Шаг 4: Behaviours**
  - [ ] Реализованы нужные lifecycle интерфейсы
  - [ ] Компоненты кешируются в Init
  - [ ] События подписываются в Enable, отписываются в Disable
  - [ ] XML комментарии с описанием lifecycle
  - [ ] Используются UseCases для бизнес-логики

- [ ] **Шаг 5: Installer**
  - [ ] SerializeFields для настройки в Inspector
  - [ ] Header атрибуты для группировки
  - [ ] Tooltip атрибуты
  - [ ] Правильный порядок: данные → behaviours
  - [ ] XML комментарии

- [ ] **Шаг 6: Интеграция**
  - [ ] Описан data flow
  - [ ] Указаны используемые фичи
  - [ ] Указаны события
  - [ ] Есть примеры интеграции с UI

- [ ] **Шаг 7: Документация**
  - [ ] Перечислены зависимости
  - [ ] Описаны Best Practices
  - [ ] Примеры правильного/неправильного кода
  - [ ] Частые ошибки описаны

- [ ] **Тестирование**
  - [ ] Работает в Unity Editor
  - [ ] Нет ошибок в Console
  - [ ] UI обновляется корректно
  - [ ] Нет memory leaks (подписки отписаны)

---

## Заключение

Этот гайд предоставляет универсальную методологию для декомпозиции и внедрения любой фичи в Atomic Framework.

**Ключевые принципы:**
1. **7 шагов** - единый процесс для всех фич
2. **4 уровня сложности** - от простого к сложному
3. **Разделение ответственности** - Installers/Behaviours/UseCases
4. **Type Safety** - EntityAPI, Tags, Enums
5. **Performance** - кеширование, Burst, ReactiveVariables

Применяйте эти принципы последовательно, и ваш код будет:
- **Понятным** - четкая структура
- **Тестируемым** - UseCases легко тестируются
- **Переиспользуемым** - Modular Installers
- **Производительным** - правильное использование lifecycle
- **Расширяемым** - новые фичи добавляются легко

**Следующие шаги:**
1. Изучить конкретные примеры в demo guides (Beginner, Shooter, RTS)
2. Применить методологию к своему проекту
3. Начать с простых фич (уровень 1-2)
4. Постепенно переходить к сложным (уровень 3-4)

**Размер документа:** ~1500 строк


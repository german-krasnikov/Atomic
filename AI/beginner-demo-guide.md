# Гайд по Beginner Demo - Основы Atomic Framework

## Введение

Beginner Demo демонстрирует базовые принципы Atomic Framework на примере простой игры по сбору монет. Это идеальная отправная точка для понимания архитектуры Entity-Component-Behaviour.

## Обзор игры

**Концепция:** Два игрока соревнуются в сборе монет за ограниченное время. Победитель определяется по количеству собранных монет.

**Основные механики:**
- Управление персонажем (WASD / Arrows)
- Движение персонажа
- Сбор монет через триггеры
- Спавн монет с периодичностью
- Таймер обратного отсчета
- Определение победителя

## Архитектура проекта

### Структура файлов

```
Scripts/
├── Common/                    # Общие компоненты
│   ├── TeamType.cs           # Enum команд
│   ├── SpawnInfo.cs          # Данные для спавна
│   ├── PlayerInfo.cs         # Информация о игроке
│   ├── InputMap.cs           # ScriptableObject для ввода
│   └── CameraBillboard.cs    # Компонент поворота к камере
├── UI/                       # UI компоненты
│   ├── MoneyView.cs
│   └── CountdownView.cs
└── Entities/                 # Entity-based архитектура
    ├── EntityAPI.cs          # Автогенерированный API
    ├── Content/              # Installers - сборка сущностей
    │   ├── CharacterInstaller.cs
    │   ├── CoinInstaller.cs
    │   └── GameContextInstaller.cs
    └── Behaviours/           # Логика поведений
        ├── InputBehaviour.cs
        ├── MovementBehaviour.cs
        ├── CoinCollectBehaviour.cs
        ├── CoinSpawnBehaviour.cs
        └── GameOverBehaviour.cs
```

### Принципы организации

1. **Разделение по назначению**: Scripts, Prefabs, Configs находятся в отдельных папках
2. **Модульность**: Entities, UI, Common компоненты изолированы
3. **Иерархическая структура**: Content (Installers) и Behaviours разделены

## Entity API - Центральная система доступа

### Что такое EntityAPI?

EntityAPI - это **автогенерированный** код, который предоставляет типобезопасный доступ к компонентам сущностей.

**Файл:** `Assets/Examples/Beginner/Scripts/Entities/EntityAPI.cs`

```csharp
public static class EntityAPI
{
    // Tags - маркеры типов сущностей
    public static readonly int Character;
    public static readonly int Coin;
    public static readonly int GameContext;

    // Values - компоненты данных
    public static readonly int Transform;              // Transform
    public static readonly int MovementDirection;      // IVariable<Vector3>
    public static readonly int MovementSpeed;          // IValue<float>
    public static readonly int Money;                  // IReactiveVariable<int>
    public static readonly int CoinSpawnInfo;          // SpawnInfo
    public static readonly int InputMap;               // InputMap
    public static readonly int TriggerEvents;          // TriggerEvents
    public static readonly int Players;                // IDictionary<TeamType, IEntity>
    public static readonly int GameCountdown;          // ICooldown

    // Extension методы для типобезопасного доступа
    public static IReactiveVariable<int> GetMoney(this IEntity entity)
        => entity.GetValue<IReactiveVariable<int>>(Money);

    public static void AddMoney(this IEntity entity, IReactiveVariable<int> value)
        => entity.AddValue(Money, value);
}
```

### Преимущества EntityAPI

✅ **Типобезопасность**: Компилятор проверяет типы
✅ **IntelliSense**: Автодополнение в IDE
✅ **Производительность**: Использование int ID вместо строк
✅ **AggressiveInlining**: Оптимизация вызовов методов

**Вместо:**
```csharp
var money = entity.GetValue<IReactiveVariable<int>>("Money"); // ❌ Строки, нет проверки
```

**Используем:**
```csharp
var money = entity.GetMoney(); // ✅ Типобезопасно, автодополнение
```

## Три кита архитектуры: Installers, Behaviours, UseCases

### 1. INSTALLERS - Композиция сущностей

**Роль:** Installers отвечают за **композицию сущностей** - добавление компонентов, тегов и поведений.

#### CharacterInstaller - Пример композиции

**Файл:** `Assets/Examples/Beginner/Scripts/Entities/Content/CharacterInstaller.cs`

```csharp
public sealed class CharacterInstaller : SceneEntityInstaller
{
    // Сериализуемые зависимости
    [SerializeField] private Const<float> _moveSpeed = 3;
    [SerializeField] private TriggerEvents _triggerEvents;
    [SerializeField] private InputMap _inputMap;
    [SerializeField] private ReactiveVariable<int> _money;

    public override void Install(IEntity entity)
    {
        // 1. Идентификация - добавление тега
        entity.AddCharacterTag();

        // 2. Базовые компоненты
        entity.AddTransform(this.transform);
        entity.AddTriggerEvents(_triggerEvents);
        entity.AddMoney(_money);

        // 3. Поведение сбора монет
        entity.AddBehaviour<CoinCollectBehaviour>();

        // 4. Система движения
        entity.AddMovementSpeed(_moveSpeed);
        entity.AddMovementDirection(new Variable<Vector3>());
        entity.AddBehaviour<MovementBehaviour>();

        // 5. Система ввода
        entity.AddInputMap(_inputMap);
        entity.AddBehaviour<InputBehaviour>();
    }
}
```

#### Архитектурные решения

✅ **Dependency Injection**: Все зависимости инжектируются через SerializeField
✅ **Порядок важен**: Сначала данные, потом поведения
✅ **Логическая группировка**: Компоненты сгруппированы по функциональности
✅ **Создание в Install**: Новые объекты (Variable) создаются в методе Install

#### CoinInstaller - Минималистичный подход

```csharp
public sealed class CoinInstaller : SceneEntityInstaller
{
    [SerializeField] private ReactiveVariable<int> _money = 1;

    public override void Install(IEntity entity)
    {
        entity.AddCoinTag();              // Идентификация
        entity.AddTransform(this.transform);
        entity.AddMoney(_money);          // Ценность монеты
    }
}
```

**Принцип:** Только необходимые компоненты. Монета - пассивная сущность без поведений.

#### GameContextInstaller - Глобальный контекст

```csharp
public sealed class GameContextInstaller : SceneEntityInstaller
{
    [SerializeField] private Transform _worldTransform;
    [SerializeField] private Cooldown _gameCountdown;
    [SerializeField] private SpawnInfo _coinSpawnInfo;
    [SerializeField] private PlayerInfo[] _players;

    public override void Install(IEntity context)
    {
        // 1. Регистрация игроков
        var players = new Dictionary<TeamType, IEntity>();
        foreach (PlayerInfo playerInfo in _players)
        {
            players.Add(playerInfo.team, playerInfo.character);
        }
        context.AddPlayers(players);

        // 2. Настройка таймера с автоматическим обновлением
        context.AddGameCountdown(_gameCountdown);
        context.WhenTick(_gameCountdown.Tick);  // ⚡ Важно!

        // 3. Система спавна монет
        context.AddCoinSpawnInfo(_coinSpawnInfo);
        context.AddBehaviour<CoinSpawnBehaviour>();

        // 4. Логика окончания игры
        context.AddBehaviour<GameOverBehaviour>();
    }
}
```

**Паттерн: Service Locator**
- Централизованное хранение глобальных данных
- Управление глобальными системами
- Подписка на обновления через `WhenTick`

### 2. BEHAVIOURS - Реактивная логика

**Роль:** Behaviours реализуют **конкретную логику** через интерфейсы жизненного цикла.

#### Lifecycle интерфейсы

```csharp
IEntityInit         // Init(IEntity entity) - инициализация
IEntityEnable       // Enable(IEntity entity) - включение
IEntityTick         // Tick(IEntity entity, float deltaTime) - Update
IEntityFixedTick    // FixedTick(IEntity entity, float deltaTime) - FixedUpdate
IEntityLateTick     // LateTick(IEntity entity, float deltaTime) - LateUpdate
IEntityDisable      // Disable(IEntity entity) - отключение
IEntityDispose      // Dispose(IEntity entity) - очистка
IEntityGizmos       // DrawGizmos(IEntity entity) - отрисовка в редакторе
```

#### InputBehaviour - Чтение ввода

```csharp
public sealed class InputBehaviour : IEntityInit, IEntityTick
{
    // Кешируемые зависимости
    private InputMap _inputMap;
    private IVariable<Vector3> _moveDirection;

    public void Init(IEntity entity)
    {
        // Кеширование зависимостей в Init для производительности
        _inputMap = entity.GetInputMap();
        _moveDirection = entity.GetMovementDirection();
    }

    public void Tick(IEntity entity, float deltaTime)
    {
        // Обновление направления движения каждый кадр
        _moveDirection.Value = this.ReadMoveDirection();
    }

    private Vector3 ReadMoveDirection()
    {
        Vector3 moveDirection = Vector3.zero;

        if (Input.GetKey(_inputMap.left))
            moveDirection.x = -1;
        else if (Input.GetKey(_inputMap.right))
            moveDirection.x = 1;

        if (Input.GetKey(_inputMap.back))
            moveDirection.z = -1;
        else if (Input.GetKey(_inputMap.forward))
            moveDirection.z = 1;

        return moveDirection;
    }
}
```

**Паттерны:**
- ✅ **Init-Update pattern**: Инициализация в Init, обновление в Tick
- ✅ **Dependency caching**: Зависимости кешируются для производительности
- ✅ **Single Responsibility**: Только чтение ввода, не применение движения

#### MovementBehaviour - Применение движения

```csharp
public sealed class MovementBehaviour : IEntityInit, IEntityFixedTick
{
    private Transform _transform;
    private IValue<Vector3> _movementDirection;
    private IValue<float> _movementSpeed;

    public void Init(IEntity entity)
    {
        _transform = entity.GetTransform();
        _movementDirection = entity.GetMovementDirection();
        _movementSpeed = entity.GetMovementSpeed();
    }

    public void FixedTick(IEntity entity, float deltaTime)
    {
        // Кинематическое движение
        _transform.position += _movementDirection.Value * (_movementSpeed.Value * deltaTime);
    }
}
```

**Разделение ответственности:**
- ✅ InputBehaviour **записывает** направление (Update)
- ✅ MovementBehaviour **читает** направление и применяет движение (FixedUpdate)
- ✅ Использование FixedUpdate для физического обновления (fixed timestep)

#### CoinCollectBehaviour - Обработка триггеров

```csharp
public sealed class CoinCollectBehaviour : IEntityInit, IEntityEnable, IEntityDisable
{
    private TriggerEvents _triggerEvents;
    private IVariable<int> _money;

    public void Init(IEntity entity)
    {
        _money = entity.GetMoney();
        _triggerEvents = entity.GetTriggerEvents();
    }

    public void Enable(IEntity entity)
    {
        // Подписка на события триггера
        _triggerEvents.OnEntered += this.OnTriggerEntered;
    }

    public void Disable(IEntity entity)
    {
        // ⚠️ ВАЖНО: Отписка от событий
        _triggerEvents.OnEntered -= this.OnTriggerEntered;
    }

    private void OnTriggerEntered(Collider collider)
    {
        // 1. Проверка наличия Entity компонента
        if (!collider.TryGetComponent(out IEntity target))
            return;

        // 2. Проверка тега Coin
        if (!target.HasCoinTag())
            return;

        // 3. Увеличение денег
        _money.Value++;

        // 4. Уничтожение монеты
        SceneEntity.Destroy(target);
    }
}
```

**Паттерны:**
- ✅ **Event-driven architecture**: Реакция на триггеры
- ✅ **Proper cleanup**: Enable/Disable для подписки/отписки
- ✅ **Tag-based filtering**: Проверка типа через теги
- ✅ **Early exit pattern**: Ранний выход при невалидных условиях

#### CoinSpawnBehaviour - Периодический спавн

```csharp
public sealed class CoinSpawnBehaviour : IEntityInit, IEntityFixedTick, IEntityGizmos
{
    private SpawnInfo _spawnInfo;

    public void Init(IEntity entity)
    {
        _spawnInfo = entity.GetCoinSpawnInfo();
    }

    public void FixedTick(IEntity entity, float deltaTime)
    {
        Cooldown period = _spawnInfo.period;
        period.Tick(deltaTime);

        if (period.IsCompleted())
        {
            this.SpawnCoin();
            period.ResetTime();
        }
    }

    private void SpawnCoin()
    {
        Vector3 position = GetRandomPosition();
        SceneEntity.Create(_spawnInfo.prefab, position, Quaternion.identity, _spawnInfo.container);
    }

    private Vector3 GetRandomPosition()
    {
        Bounds spawnArea = _spawnInfo.area;
        Vector3 min = spawnArea.min;
        Vector3 max = spawnArea.max;
        return new Vector3(Random.Range(min.x, max.x), 0, Random.Range(min.z, max.z));
    }

    public void DrawGizmos(IEntity entity)
    {
        // Визуализация области спавна в редакторе
        Bounds spawnArea = entity.GetCoinSpawnInfo().area;
        Gizmos.color = Color.black;
        Gizmos.DrawWireCube(spawnArea.center, spawnArea.size);
    }
}
```

**Паттерны:**
- ✅ **Cooldown system**: Периодический спавн через Cooldown
- ✅ **Random positioning**: Спавн в случайной позиции
- ✅ **Editor visualization**: IEntityGizmos для отладки

#### GameOverBehaviour - Управление игровым циклом

```csharp
public sealed class GameOverBehaviour : IEntityInit, IEntityDispose
{
    private ICooldown _countdown;
    private IEntity _gameContext;
    private IDictionary<TeamType, IEntity> _players;

    public void Init(IEntity entity)
    {
        _gameContext = entity;
        _players = entity.GetPlayers();
        _countdown = entity.GetGameCountdown();
        _countdown.OnCompleted += this.OnGameTimeFinished;
    }

    public void Dispose(IEntity entity)
    {
        _countdown.OnCompleted -= this.OnGameTimeFinished;
    }

    private void OnGameTimeFinished()
    {
        TeamType teamType = this.GetWinner();
        this.LogWinner(teamType);

        // Отключение всех игроков
        foreach (IEntity player in _players.Values)
            player.Disable();

        // Отключение игрового контекста
        _gameContext.Disable();
    }

    private TeamType GetWinner()
    {
        int redMoney = this.GetMoney(TeamType.RED);
        int blueMoney = this.GetMoney(TeamType.BLUE);

        return redMoney > blueMoney
            ? TeamType.RED
            : blueMoney > redMoney
                ? TeamType.BLUE
                : TeamType.NONE;
    }

    private int GetMoney(TeamType team)
    {
        IEntity player = _players[team];
        return player.GetMoney().Value;
    }
}
```

**Game loop management:**
- ✅ Подписка на событие завершения таймера
- ✅ Сравнение счета команд
- ✅ Управление состоянием игры через Disable

### 3. USECASES - Бизнес-логика (в этом демо не используются явно)

В Beginner Demo логика достаточно простая и находится непосредственно в Behaviours. Однако в более сложных проектах используются UseCases - статические классы с бизнес-логикой.

**Пример из других проектов:**
```csharp
public static class DamageUseCase
{
    public static bool DealDamage(IEntity instigator, IEntity target, int damage)
    {
        // Бизнес-логика нанесения урона
    }
}
```

## Реализация основных механик

### Механика 1: Движение персонажа

**Цепочка обработки:** Input → Direction → Movement → Transform

```
InputBehaviour (Update)
    ↓ записывает
MovementDirection (IVariable<Vector3>)
    ↓ читает
MovementBehaviour (FixedUpdate)
    ↓ применяет к
Transform.position
```

**Код:**
```csharp
// InputBehaviour - запись направления (каждый кадр)
_moveDirection.Value = ReadMoveDirection();

// MovementBehaviour - применение движения (физический кадр)
_transform.position += _movementDirection.Value * (_movementSpeed.Value * deltaTime);
```

**Почему разделено?**
- ✅ Input в Update для отзывчивости
- ✅ Movement в FixedUpdate для стабильной физики
- ✅ Разделение ответственности

### Механика 2: Сбор монет

**Цепочка обработки:** Trigger → Check Tag → Collect → Destroy

```
Physics Trigger Enter
    ↓
TriggerEvents.OnEntered
    ↓
CoinCollectBehaviour.OnTriggerEntered()
    ↓
HasCoinTag() ? → Money.Value++ → SceneEntity.Destroy()
```

**Ключевые моменты:**
- ✅ Триггеры через Unity Physics
- ✅ Tag-based identification
- ✅ Reactive variable для автообновления UI
- ✅ Немедленное уничтожение монеты

### Механика 3: Игровой цикл

**Lifecycle:** Start → Play → Countdown → Game Over

```
GameContext.Install()
    ↓
Countdown.Tick() (автоматически через WhenTick)
    ↓
CoinSpawnBehaviour.FixedTick() (спавн монет)
    ↓
Players собирают монеты
    ↓
Countdown.OnCompleted
    ↓
GameOverBehaviour.OnGameTimeFinished()
    ↓
Определение победителя → Disable всех игроков
```

## Reactive система - автообновление UI

### Принцип работы

**Reactive переменные** автоматически оповещают подписчиков об изменениях.

```csharp
// Character - источник данных
[SerializeField] private ReactiveVariable<int> _money;
entity.AddMoney(_money);

// MoneyView - подписчик
private void Start()
{
    _subscription = _player.GetMoney().Observe(this.OnMoneyChanged);
}

private void OnMoneyChanged(int money)
{
    _moneyText.text = $"Money: {money}"; // Автоматическое обновление!
}

private void OnDestroy()
{
    _subscription?.Dispose(); // ⚠️ ВАЖНО: Очистка подписки
}
```

**Преимущества:**
- ✅ Автоматическое обновление UI
- ✅ Нет ручного вызова Update
- ✅ Явное управление подписками

### Типы реактивных элементов

```csharp
// Реактивная переменная (можно читать и писать)
IReactiveVariable<int> money = new ReactiveVariable<int>(100);
money.Value = 150; // Триггерит событие

// Реактивное значение (только чтение)
IReactiveValue<int> readOnlyMoney = money;
int current = readOnlyMoney.Value;
```

## Архитектурные паттерны

### 1. Entity-Component-System (ECS)

```csharp
// Entity - контейнер данных
IEntity entity = ...;

// Components - данные
entity.AddTransform(transform);
entity.AddMoney(money);

// Systems (Behaviours) - логика
entity.AddBehaviour<MovementBehaviour>();
```

### 2. Composition over Inheritance

Вместо наследования `Character : BaseEntity` используется композиция:

```csharp
entity.AddMovementSpeed(_moveSpeed);
entity.AddMovementDirection(new Variable<Vector3>());
entity.AddBehaviour<MovementBehaviour>();
```

### 3. Service Locator / Game Context

```csharp
public sealed class GameContextInstaller : SceneEntityInstaller
{
    public override void Install(IEntity context)
    {
        context.AddPlayers(players);        // Регистр игроков
        context.AddGameCountdown(_countdown); // Глобальный таймер
        context.AddCoinSpawnInfo(_spawnInfo); // Конфигурация спавна
    }
}
```

### 4. Data-Driven Design

Конфигурация через ScriptableObject:

```csharp
[CreateAssetMenu(fileName = "InputMap", menuName = "BeginnerGame/InputMap")]
public sealed class InputMap : ScriptableObject
{
    public KeyCode forward;
    public KeyCode back;
    public KeyCode left;
    public KeyCode right;
}
```

### 5. Observer Pattern (Reactive)

```csharp
// Observable
public sealed class ReactiveVariable<int> : IReactiveVariable<int>
{
    public Subscription<int> Observe(Action<int> action);
}

// Observer
_subscription = _player.GetMoney().Observe(this.OnMoneyChanged);
```

## Best Practices

### 1. Dependency Injection через Init

```csharp
public sealed class InputBehaviour : IEntityInit, IEntityTick
{
    // Поля для зависимостей
    private InputMap _inputMap;
    private IVariable<Vector3> _moveDirection;

    public void Init(IEntity entity)
    {
        // Разрешение зависимостей при инициализации
        _inputMap = entity.GetInputMap();
        _moveDirection = entity.GetMovementDirection();
    }
}
```

### 2. Separation of Concerns

```csharp
// InputBehaviour - только чтение ввода
public void Tick(IEntity entity, float deltaTime)
{
    _moveDirection.Value = this.ReadMoveDirection();
}

// MovementBehaviour - только применение движения
public void FixedTick(IEntity entity, float deltaTime)
{
    _transform.position += _movementDirection.Value * speed * deltaTime;
}
```

### 3. Early Exit Pattern

```csharp
private void OnTriggerEntered(Collider collider)
{
    // Ранний выход при невалидных условиях
    if (!collider.TryGetComponent(out IEntity target))
        return;

    if (!target.HasCoinTag())
        return;

    // Основная логика только если все проверки пройдены
    _money.Value++;
    SceneEntity.Destroy(target);
}
```

### 4. Proper Resource Management

```csharp
// Enable - выделение ресурсов
public void Enable(IEntity entity)
{
    _triggerEvents.OnEntered += this.OnTriggerEntered;
}

// Disable - освобождение ресурсов
public void Disable(IEntity entity)
{
    _triggerEvents.OnEntered -= this.OnTriggerEntered;
}
```

### 5. Type-Safe API

```csharp
// Всегда используй сгенерированные extension methods
var money = entity.GetMoney(); // ✅ Типобезопасно

// Избегай прямого доступа
var money = entity.GetValue<IReactiveVariable<int>>("Money"); // ❌
```

### 6. Sealed Classes для производительности

```csharp
// Все behaviours и installers должны быть sealed
public sealed class MovementBehaviour : IEntityInit, IEntityFixedTick
{
    // Компилятор может оптимизировать вызовы
}
```

### 7. Serializable Configuration Data

```csharp
[Serializable]
public sealed class SpawnInfo
{
    public SceneEntity prefab;
    public Transform container;
    public Bounds area = new(Vector3.zero, new Vector3(5, 0, 5));
    public Cooldown period = 2;
}
```

## Практические советы

### Добавление новой механики

**Пример: Добавить здоровье персонажу**

1. **Добавить в EntityAPI** (если генерация из YAML):
```yaml
values:
  - Health: IReactiveVariable<int>
```

2. **Создать HealthBehaviour**:
```csharp
public sealed class HealthBehaviour : IEntityInit, IEntityEnable, IEntityDisable
{
    private IReactiveVariable<int> _health;

    public void Init(IEntity entity)
    {
        _health = entity.GetHealth();
    }

    public void Enable(IEntity entity)
    {
        _health.Subscribe(OnHealthChanged);
    }

    public void Disable(IEntity entity)
    {
        _health.Unsubscribe(OnHealthChanged);
    }

    private void OnHealthChanged(int health)
    {
        if (health <= 0)
            entity.Disable();
    }
}
```

3. **Добавить в CharacterInstaller**:
```csharp
entity.AddHealth(new ReactiveVariable<int>(100));
entity.AddBehaviour<HealthBehaviour>();
```

## Заключение

Beginner Demo демонстрирует:

✅ **Модульность**: Каждый Behaviour выполняет одну задачу
✅ **Переиспользуемость**: Behaviours можно комбинировать по-разному
✅ **Тестируемость**: Зависимости инжектируются, легко мокировать
✅ **Расширяемость**: Новые фичи = новые Behaviours + Installers
✅ **Производительность**: AggressiveInlining, sealed classes, кеширование
✅ **Типобезопасность**: EntityAPI генерирует строготипизированные методы

### Следующие шаги

После изучения Beginner Demo рекомендуется:
1. **Shooter Demo** - изучение системы контекстов (App/Game/Player)
2. **RTS Demo** - продвинутые паттерны (Factories, Builders, Filters, AI)
3. **Best Practices документация** - углубленное изучение паттернов

### Ключевые принципы для запоминания

1. **Installers** - конфигурация сущностей (что добавляем)
2. **Behaviours** - логика жизненного цикла (когда выполняем)
3. **UseCases** - бизнес-логика (как выполняем)
4. **EntityAPI** - типобезопасный доступ
5. **Reactive** - автоматическое распространение изменений

# Гайд по Atomic Framework v2 - Полное руководство по архитектуре

## Введение

Atomic Framework v2 - это **продвинутая Entity-Component-Behaviour архитектура** для Unity, обеспечивающая модульность, масштабируемость и высокую производительность. Версия 2 включает все возможности v1 плюс расширенные паттерны для enterprise-level проектов.

## Философия Atomic Framework

### Ключевые принципы

1. **ESB Pattern (Entity-State-Behaviour)**
   - Entity = динамический контейнер для данных и логики
   - State = набор атомарных элементов (shared data)
   - Behaviour = упорядоченный набор контроллеров без собственного состояния

2. **Atomic Approach**
   - Всё строится из "атомарных элементов" как из конструктора
   - Высокая переиспользуемость механик
   - Динамическое изменение структуры entity в runtime

3. **Separation of Concerns**
   - Данные и логика строго разделены
   - Model и View изолированы
   - Процедурное программирование для взаимодействия

4. **Data Abstractions First**
   - Работа через интерфейсы, а не конкретные типы
   - Упрощает тестирование, поддержку и мультиплеер
   - Dependency Inversion Principle

## Архитектура проекта

### Рекомендуемая структура папок

```
Assets/
├── Game/
│   ├── Scripts/
│   │   ├── App/              # Application-level
│   │   │   ├── Bootstrap/
│   │   │   ├── SaveSystem/
│   │   │   └── LevelSystem/
│   │   ├── Gameplay/         # Game logic
│   │   │   ├── Common/       # Reusable mechanics
│   │   │   ├── GameEntities/
│   │   │   │   ├── Core/     # Reusable installers
│   │   │   │   ├── View/     # Visualization
│   │   │   │   └── Content/  # Concrete entities
│   │   │   ├── GameContext/  # Game state
│   │   │   └── PlayerContext/ # Player state
│   │   └── UI/               # User interface
│   │      ├── MenuUI/
│   │      └── GameplayUI/
├── Modules/                  # Reusable systems
│   ├── DialogueSystem/
│   └── UpgradeFeature/
└── Plugins/                  # Third-party & framework
    └── Atomic/               # Atomic Framework
```

## Трёхслойная архитектура: Installers, Behaviours, UseCases

### 1. INSTALLERS - Композиция и конфигурация

**Роль:** Installers отвечают за **композицию сущностей** - добавление компонентов, значений и поведений.

#### Типы Installers

**SceneEntityInstaller** - для MonoBehaviour на сцене:
```csharp
public sealed class CharacterInstaller : SceneEntityInstaller<IGameEntity>
{
    [SerializeField] private Const<float> _moveSpeed = 5.0f;
    [SerializeField] private Transform _transform;

    public override void Install(IGameEntity entity)
    {
        entity.AddTransform(_transform);
        entity.AddMoveSpeed(_moveSpeed);
        entity.AddBehaviour<MoveBehaviour>();
    }
}
```

**ScriptableEntityInstaller** - для ScriptableObject (переиспользуемые):
```csharp
[CreateAssetMenu]
public sealed class WeaponInstaller : ScriptableEntityInstaller<IWeaponEntity>
{
    [SerializeField] private ReactiveInt _ammo;
    [SerializeField] private Cooldown _cooldown;

    public void Install(IWeaponEntity entity)
    {
        entity.AddAmmo(_ammo);
        entity.AddCooldown(_cooldown);
        entity.AddBehaviour<WeaponBehaviour>();
    }
}
```

**Modular Installers** - композиция (РЕКОМЕНДУЕТСЯ):
```csharp
// Модульный установщик
[Serializable]
public class MovementEntityInstaller : IEntityInstaller<IGameEntity>
{
    [SerializeField] private Const<float> _moveSpeed = 3;

    public void Install(IGameEntity entity)
    {
        entity.AddMoveSpeed(_moveSpeed);
        entity.AddMoveDirection(new BaseVariable<Vector3>());
        entity.AddBehaviour<MovementBehaviour>();
    }
}

// Использование
public sealed class CharacterInstaller : SceneEntityInstaller<IGameEntity>
{
    [SerializeField] private MovementEntityInstaller _movementInstaller;
    [SerializeField] private RotationEntityInstaller _rotationInstaller;

    public override void Install(IGameEntity entity)
    {
        // Композиция через модульные установщики
        entity.Install(_movementInstaller);
        entity.Install(_rotationInstaller);
    }
}
```

#### Ключевые паттерны для Installers

**Паттерн 1: Optional Dependencies**
```csharp
public sealed class WeaponInstaller : SceneEntityInstaller
{
    [SerializeField] private Optional<ReactiveInt> _ammo;
    [SerializeField] private Optional<Cooldown> _cooldown;

    public void Install(IEntity entity)
    {
        // Добавляем только если active в Inspector
        if (_ammo) entity.AddAmmo(_ammo);
        if (_cooldown) entity.AddCooldown(_cooldown);
    }
}
```

**Паттерн 2: PlayMode Guard**
```csharp
public void Install(IPlayerContext context)
{
    if (AtomicUtils.IsPlayMode())
    {
        // Runtime-specific операции
        GameEntity character = CharacterUseCase.Spawn(...);
        context.AddCharacter(character);
    }

    // Всегда добавляем behaviours
    context.AddBehaviour<CharacterMoveController>();
}
```

**Паттерн 3: Uninstall для Cleanup**
```csharp
public sealed class WeaponViewInstaller : SceneEntityInstaller
{
    private readonly DisposableComposite _disposables = new();

    public override void Install(IEntity entity)
    {
        ISignal fireEvent = entity.GetFireEvent();
        fireEvent.Subscribe(_fireVFX.Play).AddTo(_disposables);
    }

    public override void Uninstall()
    {
        _disposables.Dispose(); // Cleanup всех подписок
    }
}
```

### 2. BEHAVIOURS - Реактивная логика жизненного цикла

**Роль:** Behaviours определяют **КОГДА** и **КАК** выполняется логика через lifecycle interfaces.

#### Lifecycle интерфейсы

```csharp
IEntityInit<T>         // Init(T entity) - инициализация
IEntityEnable          // Enable(IEntity entity) - включение
IEntityTick            // Tick(IEntity entity, float deltaTime) - Update
IEntityFixedTick       // FixedTick(IEntity entity, float deltaTime) - FixedUpdate
IEntityLateTick        // LateTick(IEntity entity, float deltaTime) - LateUpdate
IEntityDisable         // Disable(IEntity entity) - отключение
IEntityDispose         // Dispose(IEntity entity) - очистка
IEntityGizmos          // DrawGizmos(IEntity entity) - отрисовка в редакторе
```

#### Request-Condition-Action-Event (RCAE) Flow

**Золотой стандарт для Behaviours:**

```csharp
public sealed class JumpBehaviour : IEntityInit, IEntityFixedTick
{
    private IRequest _jumpRequest;        // 1. Request (намерение)
    private IFunction<bool> _jumpCondition; // 2. Condition (проверка)
    private IAction _jumpAction;          // 3. Action (выполнение)
    private IEvent _jumpEvent;            // 4. Event (оповещение)

    public void Init(IEntity entity)
    {
        _jumpRequest = entity.GetJumpRequest();
        _jumpCondition = entity.GetJumpCondition();
        _jumpAction = entity.GetJumpAction();
        _jumpEvent = entity.GetJumpEvent();
    }

    public void FixedTick(IEntity entity, float deltaTime)
    {
        // RCAE flow
        if (_jumpRequest.Consume() && _jumpCondition.Invoke())
        {
            _jumpAction.Invoke();
            _jumpEvent.Invoke();
        }
    }
}
```

**Преимущества RCAE:**
- ✅ Модульность: каждый компонент независим
- ✅ Безопасность: условия предотвращают неверные действия
- ✅ Наблюдаемость: легко отслеживать и реагировать
- ✅ Гибкость: легко добавлять новые механики

#### Inline Behaviours через WhenFixedTick

Для простой логики можно использовать inline-подход:

```csharp
public override void Install(IEntity entity)
{
    entity.AddMoveRequest(new Request<Vector3>());

    entity.WhenFixedTick(deltaTime =>
    {
        if (entity.GetMoveRequest().Consume(out Vector3 direction))
        {
            MoveUseCase.MoveStep(entity, direction, deltaTime);
        }
    });
}
```

**Когда использовать:**
- Логика простая (5-15 строк)
- Не требует Init/Dispose
- Специфична для данного installer

### 3. USECASES - Процедурная бизнес-логика

**Роль:** UseCases содержат **процедурную бизнес-логику** для взаимодействия между entities.

#### Принципы UseCases

1. **Stateless** - без состояния
2. **Static methods** - статические методы
3. **Pure logic** - чистая логика без побочных эффектов
4. **Easy testing** - легко тестируется

#### Примеры UseCases

**CharacterUseCase:**
```csharp
public static class CharacterUseCase
{
    public static Actor Spawn(
        IPlayerContext context,
        IGameContext gameContext,
        Actor prefab)
    {
        Transform worldTransform = gameContext.GetWorldTransform();
        Transform spawnPoint = SpawnPointsUseCase.NextPoint(gameContext);
        Actor entity = SceneEntity.Create(prefab, spawnPoint, worldTransform);
        entity.GetTeamType().Value = context.GetTeamType().Value;
        return entity;
    }

    public static void Respawn(IPlayerContext playerContext, IGameContext gameContext)
    {
        IActor character = playerContext.GetCharacter();
        character.GetHealth().AssignMax();

        Transform nextPoint = SpawnPointsUseCase.NextPoint(gameContext);
        character.GetPosition().Value = nextPoint.position;
        character.GetRotation().Value = nextPoint.rotation;

        character.GetRespawnEvent().Invoke();
    }
}
```

**DamageUseCase:**
```csharp
public static class DamageUseCase
{
    public static bool DealDamage(IUnitEntity source, IUnitEntity target)
    {
        if (!target.HasDamageableTag())
            return false;

        Health health = target.GetHealth();
        if (health.IsEmpty())
            return false;

        int damage = source.GetDamage().Value;
        health.Reduce(damage);
        target.GetTakeDamageEvent().Invoke(damage);

        return true;
    }
}
```

#### Method Overloading для расширения

```csharp
public static class DamageUseCase
{
    // Overloaded метод для работы с Collider
    public static bool DealDamage(
        IGameEntity instigator,
        Collider victim,
        IGameContext gameContext,
        int damage)
    {
        return victim.TryGetComponent(out IGameEntity target) &&
               DealDamage(instigator, target, gameContext, damage);
    }

    // Оригинальный метод
    public static bool DealDamage(
        IGameEntity instigator,
        IGameEntity victim,
        IGameContext gameContext,
        int damage)
    {
        // Основная логика
    }
}
```

## Factory Pattern и Catalog

### UnitFactory - Абстрактная фабрика

```csharp
public abstract class UnitFactory : ScriptableEntityFactory<IUnitEntity>
{
    public string Name => this.name;

    public sealed override IUnitEntity Create()
    {
        var entity = new UnitEntity(
            this.Name,
            this.initialTagCapacity,
            this.initialValueCapacity,
            this.initialBehaviourCapacity
        );
        this.Install(entity);
        return entity;
    }

    protected abstract void Install(IUnitEntity entity);
}
```

### Конкретные фабрики

**WarriorFactory - ближний бой:**
```csharp
[CreateAssetMenu]
public sealed class WarriorFactory : UnitFactory
{
    [SerializeField] private TransformEntityInstaller _transformInstaller;
    [SerializeField] private MoveEntityInstaller _moveInstaller;
    [SerializeField] private LifeEntityInstaller _lifeInstaller;
    [SerializeField] private MeleeCombatEntityInstaller _meleeCombatInstaller;
    [SerializeField] private AIEntityInstaller _aiInstaller;

    protected override void Install(IUnitEntity entity)
    {
        entity.AddUnitTag();
        entity.AddTeam(new ReactiveVariable<TeamType>());

        // Композиция компонентов
        entity.Install(_transformInstaller);
        entity.Install(_moveInstaller);
        entity.Install(_lifeInstaller);
        entity.Install(_meleeCombatInstaller);  // Ближний бой
        entity.Install(_aiInstaller);
    }
}
```

**Warrior = Transform + Move + Life + MeleeCombat + AI**

### UnitCatalog - каталог фабрик

```csharp
[CreateAssetMenu]
public sealed class UnitCatalog : ScriptableMultiEntityFactory<string, IUnitEntity, UnitFactory>
{
    protected override string GetKey(UnitFactory factory) => factory.Name;
}
```

**Использование:**
```csharp
// Создание пула с каталогом
context.AddEntityPool(new MultiEntityPool<string, IUnitEntity>(factoryCatalog));

// Создание юнитов по имени
IUnitEntity warrior = pool.Rent("Warrior");
IUnitEntity tank = pool.Rent("Tank");
```

## Builder Pattern для сложных объектов

### PlayerContextBuilder

```csharp
public sealed class PlayerContextBuilder : ScriptableEntityFactory<IPlayerContext>
{
    private EntityWorld<IUnitEntity> _entityWorld;
    private TeamType _teamType;

    public PlayerContextBuilder SetEntityWorld(EntityWorld<IUnitEntity> world)
    {
        _entityWorld = world;
        return this;
    }

    public PlayerContextBuilder SetTeamType(TeamType type)
    {
        _teamType = type;
        return this;
    }

    public override IPlayerContext Create()
    {
        // Validation
        if (_entityWorld == null)
            throw new InvalidOperationException("EntityWorld required");

        var playerContext = new PlayerContext($"PlayerContext {_teamType}");

        // Конфигурация
        playerContext.AddTeam(new Const<TeamType>(_teamType));
        playerContext.AddFreeEnemyFilter(this.CreateFreeEnemyFilter(playerContext));

        return playerContext;
    }

    private EntityFilter<IUnitEntity> CreateFreeEnemyFilter(IPlayerContext playerContext)
    {
        return new EntityFilter<IUnitEntity>(
            _entityWorld,
            entity => TeamUseCase.IsFreeEnemyUnit(playerContext, entity),
            new TeamEntityTrigger(),
            new TagEntityTrigger<IUnitEntity>()
        );
    }
}
```

**Использование:**
```csharp
IPlayerContext bluePlayer = _playerFactory
    .SetEntityWorld(entityWorld)
    .SetTeamType(TeamType.BLUE)
    .Create();
```

## EntityFilter - динамические выборки

EntityFilter автоматически фильтрует entities по условиям и обновляется при изменениях.

```csharp
// Создание фильтра
EntityFilter<IUnitEntity> enemyFilter = new EntityFilter<IUnitEntity>(
    _entityWorld,                          // Источник entities
    entity => TeamUseCase.IsFreeEnemyUnit(playerContext, entity), // Условие
    new TeamEntityTrigger(),               // Триггер 1
    new TagEntityTrigger<IUnitEntity>()    // Триггер 2
);

// Использование
foreach (IUnitEntity enemy in enemyFilter)
{
    // Автоматически только подходящие юниты
}
```

**Триггеры обновления:**
- `TeamEntityTrigger` - при изменении команды
- `TagEntityTrigger` - при изменении тегов
- `ValueEntityTrigger` - при изменении значений
- `StateChangedEntityTrigger` - при изменении состояния

## EntityWorld и Object Pooling

### EntityWorld - управление lifecycle

```csharp
public void Install(IGameContext context)
{
    // 1. Создаем мир для всех юнитов
    EntityWorld<IUnitEntity> entityWorld = new EntityWorld<IUnitEntity>();
    context.AddEntityWorld(entityWorld);

    // 2. Привязываем жизненный цикл
    context.WhenInit(entityWorld.InitEntities);
    context.WhenEnable(entityWorld.Enable);
    context.WhenTick(entityWorld.Tick);
    context.WhenFixedTick(entityWorld.FixedTick);
    context.WhenLateTick(entityWorld.LateTick);
    context.WhenDisable(entityWorld.Disable);
    context.WhenDispose(entityWorld.DisposeEntities);
}
```

### Object Pooling

```csharp
public static class UnitsUseCase
{
    public static IUnitEntity Spawn(
        IGameContext context,
        string name,
        Vector3 position,
        Quaternion rotation,
        TeamType team)
    {
        // Rent from pool
        IMultiEntityPool<string, IUnitEntity> pool = context.GetEntityPool();
        IUnitEntity entity = pool.Rent(name);

        // Configure
        entity.GetPosition().Value = position;
        entity.GetRotation().Value = rotation;
        entity.GetTeam().Value = team;

        // Add to world
        context.GetEntityWorld().Add(entity);

        return entity;
    }

    public static bool Despawn(IGameContext context, IUnitEntity entity)
    {
        // Remove from world
        if (!context.GetEntityWorld().Remove(entity))
            return false;

        // Return to pool
        context.GetEntityPool().Return(entity);
        return true;
    }
}
```

## Entity API - Code Generation

### YAML Configuration

```yaml
# UnitEntityAPI.yml
header: EntityAPI
entityType: IUnitEntity
aggressiveInlining: true
unsafeAccess: true

namespace: RTSGame
className: UnitEntityAPI
directory: Assets/Game/Scripts/UnitEntities/

imports:
  - UnityEngine
  - Atomic.Elements
  - System

tags:
  - Damageable
  - Moveable
  - Unit

values:
  #Transform
  - Position: IReactiveVariable<Vector3>
  - Rotation: IReactiveVariable<Quaternion>
  - Scale: IValue<float>
  #Movement
  - MoveSpeed: IValue<float>
  - MoveRequest: IRequest<Vector3>
  - MoveEvent: IEvent<Vector3>
  #Combat
  - Damage: IValue<int>
  - FireRequest: IRequest<IUnitEntity>
  - FireEvent: IEvent<IUnitEntity>
```

### Generated Code

```csharp
public static class UnitEntityAPI
{
    // Constants
    public static readonly int Damageable;
    public static readonly int Position;
    public static readonly int MoveSpeed;

    // Tag methods
    [MethodImpl(MethodImplOptions.AggressiveInlining)]
    public static bool HasDamageableTag(this IUnitEntity entity) =>
        entity.HasTag(Damageable);

    // Value methods
    [MethodImpl(MethodImplOptions.AggressiveInlining)]
    public static IReactiveVariable<Vector3> GetPosition(this IUnitEntity entity) =>
        entity.GetValueUnsafe<IReactiveVariable<Vector3>>(Position);

    [MethodImpl(MethodImplOptions.AggressiveInlining)]
    public static void AddPosition(this IUnitEntity entity, IReactiveVariable<Vector3> value) =>
        entity.AddValue(Position, value);
}
```

## Reactive системы

### IReactiveVariable

```csharp
// Создание
IReactiveVariable<int> money = new ReactiveVariable<int>(100);

// Подписка
money.Subscribe(newValue => Debug.Log($"Money: {newValue}"));

// Изменение (триггерит событие)
money.Value = 150;

// Observe с автоочисткой
Subscription<int> subscription = money.Observe(OnMoneyChanged);
// ...
subscription.Dispose();
```

### IReactiveDictionary

```csharp
// Создание
IReactiveDictionary<TeamType, int> leaderboard = new ReactiveDictionary<TeamType, int>();

// Подписка на изменения
leaderboard.Subscribe(OnLeaderboardChanged);

// Изменение
leaderboard[TeamType.BLUE] += 1;
```

## Request, Event, Action - коммуникация

### Request - Producer-Consumer

```csharp
// Producer (Update)
public void Tick(IEntity entity, float deltaTime)
{
    float dx = Input.GetAxis("Horizontal");
    entity.GetMoveRequest().Invoke(new Vector3(dx, 0, 0));
}

// Consumer (FixedUpdate)
public void FixedTick(IEntity entity, float deltaTime)
{
    if (entity.GetMoveRequest().Consume(out Vector3 direction))
    {
        MoveUseCase.Move(entity, direction, deltaTime);
    }
}
```

### Event - Множественные подписчики

```csharp
// Определение
entity.AddFireEvent(new Event<IUnitEntity>());

// Подписка
entity.GetFireEvent().Subscribe(OnFire);

// Вызов оповещает всех
entity.GetFireEvent().Invoke(target);
```

### Action - Немедленное выполнение

```csharp
// Определение
entity.AddFireAction(new InlineAction(() => {
    BulletUseCase.Spawn(...);
}));

// Вызов
entity.GetFireAction().Invoke();
```

### Когда использовать?

| Паттерн | Использование | Пример |
|---------|--------------|--------|
| Request | Producer-Consumer, Input, AI | MoveRequest, FireRequest |
| Event | Множественные подписчики | TakeDamageEvent, FireEvent |
| Action | Немедленное выполнение | FireAction, DestroyAction |

## Burst Compilation для производительности

```csharp
[BurstCompile]
public static class MoveUseCase
{
    public static void MoveStep(IUnitEntity entity, Vector3 direction, float deltaTime)
    {
        IReactiveVariable<Vector3> position = entity.GetPosition();
        MoveStep(
            position.Value,
            direction,
            entity.GetMoveSpeed().Value,
            deltaTime,
            out float3 next
        );
        position.Value = next;
    }

    [BurstCompile]
    public static void MoveStep(
        in float3 position,
        in float3 direction,
        in float speed,
        in float deltaTime,
        out float3 result
    ) => result = position + speed * deltaTime * direction;
}
```

**Использование Unity Mathematics:**
- `float3` вместо `Vector3`
- `quaternion` вместо `Quaternion`
- `math.` вместо `Mathf.`

## Model-View Separation

### PositionViewBehaviour

```csharp
[Serializable]
public sealed class PositionViewBehaviour : IEntityInit<IUnitEntity>, IEntityDispose
{
    [SerializeField] private Transform _transform;
    private IReactiveValue<Vector3> _position;

    public void Init(IUnitEntity entity)
    {
        _position = entity.GetPosition();
        _position.Observe(this.OnPositionChanged);
    }

    public void Dispose(IEntity entity)
    {
        _position.Unsubscribe(this.OnPositionChanged);
    }

    private void OnPositionChanged(Vector3 position)
    {
        _transform.position = position;
    }
}
```

**Принцип:** View подписывается на изменения Model через Reactive values.

## Best Practices

### 1. Composition over Inheritance

```csharp
// ✅ Хорошо - композиция
entity.Install(_transformInstaller);
entity.Install(_moveInstaller);
entity.Install(_lifeInstaller);

// ❌ Плохо - наследование
public class Warrior : BaseUnit { }
```

### 2. Data Abstractions First

```csharp
// ✅ Хорошо - через интерфейсы
IValue<Vector3> moveDirection = ...;
IValue<float> speed = ...;

// ❌ Плохо - конкретные типы
ReactiveVariable<Vector3> moveDirection = ...;
Const<float> speed = ...;
```

### 3. Dependency Injection через конструктор

```csharp
// ✅ Хорошо - через конструктор
public sealed class LifeBehaviour : IEntityInit
{
    private readonly IGameContext _gameContext;

    public LifeBehaviour(IGameContext gameContext)
    {
        _gameContext = gameContext;
    }
}

// Использование
entity.AddBehaviour(new LifeBehaviour(gameContext));
```

### 4. Используй Optional для опциональных зависимостей

```csharp
[SerializeField] private Optional<Cooldown> _cooldown;

public void Install(IEntity entity)
{
    if (_cooldown) entity.AddCooldown(_cooldown);
}
```

### 5. DisposableComposite для подписок

```csharp
private readonly DisposableComposite _disposables = new();

public void Install(IEntity entity)
{
    entity.GetFireEvent().Subscribe(OnFire).AddTo(_disposables);
}

public void Uninstall()
{
    _disposables.Dispose();
}
```

### 6. Expressions для условной логики

```csharp
entity.AddFireCondition(new AndExpression(
    () => _health.Value > 0,
    () => _ammo.Value > 0,
    () => _cooldown.IsCompleted()
));
```

### 7. Sealed classes для производительности

```csharp
public sealed class MovementBehaviour : IEntityInit, IEntityFixedTick
{
    // Компилятор может оптимизировать
}
```

## Производительность

### Оптимизации

1. **Burst Compilation** для MoveUseCase, RotateUseCase
2. **Object Pooling** для entities
3. **EntityWorld** для batch операций
4. **EntityFilter** для быстрых выборок
5. **AggressiveInlining** для API методов
6. **Кеширование** зависимостей в Init

### Предпочитай конкретные типы для итерации

```csharp
// ❌ Медленно - boxing
IEntityCollection collection = ...;
foreach (IEntity entity in collection) { }

// ✅ Быстро - no boxing
EntityCollection collection = ...;
foreach (IEntity entity in collection) { }
```

## Testing

### Pure C# Testing

```csharp
[Test]
public void Tick_WithNonZeroDirection_MovesCorrectly()
{
    // Arrange
    var entity = new Entity();
    entity.AddPosition(new Variable<Vector3>(Vector3.zero));
    entity.AddMoveSpeed(new Const<float>(2f));
    entity.AddMoveDirection(new Variable<Vector3>());
    entity.AddBehaviour(new MoveBehaviour());
    entity.Init();

    entity.GetMoveDirection().Value = Vector3.forward;

    // Act
    entity.Tick(0.5f);

    // Assert
    Assert.AreEqual(new Vector3(0, 0, 1), entity.GetPosition().Value);
}
```

## Заключение

Atomic Framework v2 предоставляет:

✅ **Модульность** через Composition
✅ **Масштабируемость** через EntityWorld и Pooling
✅ **Производительность** через Burst и AggressiveInlining
✅ **Гибкость** через Factory, Builder, Catalog
✅ **Тестируемость** через Pure C# Logic
✅ **Расширяемость** через Modular Installers
✅ **Type-safety** через Code Generation

### Ключевые принципы

1. **Installers** - конфигурация и передача зависимостей
2. **Behaviours** - настройка КОГДА события происходят (lifecycle)
3. **UseCases** - основная бизнес-логика (stateless)
4. **Self-contained components** - изолированные классы (Health, Cooldown)
5. **Reactive systems** - автоматическое распространение изменений
6. **Request/Event/Action** - паттерны коммуникации
7. **Factory/Builder/Catalog** - паттерны создания
8. **EntityFilter** - динамические выборки
9. **Model-View Separation** - разделение логики и визуализации
10. **Composition over Inheritance** - гибкость через композицию

### Формула успеха

```
Modular Installers (переиспользование) +
RCAE Flow (модульность) +
UseCases (чистая логика) +
Factory Pattern (гибкое создание) +
EntityFilter (эффективные выборки) +
Burst Compilation (производительность) +
Object Pooling (масштабирование) =
Production-ready архитектура
```

### Практические примеры

Для изучения практического применения смотрите:
- **Beginner Demo** - основы архитектуры
- **Shooter Demo** - иерархия контекстов
- **RTS Demo** - продвинутые паттерны и масштабирование

### Три уровня архитектурной сложности

Atomic Framework поддерживает три уровня сложности, выбирайте в зависимости от масштаба проекта:

#### Level 1: Beginner (Простая архитектура)
**Когда использовать:** Прототипы, обучение, небольшие проекты (< 50 entities)

**Характеристики:**
- `SceneEntity` вместо Factory/Pool
- Монолитные Installers
- Простые Behaviours без UseCases
- Прямые ссылки на префабы

**Пример:** Beginner Demo (сбор монет)
```csharp
public sealed class CharacterInstaller : SceneEntityInstaller
{
    public override void Install(IEntity entity)
    {
        entity.AddTransform(this.transform);
        entity.AddMoveSpeed(_moveSpeed);
        entity.AddMovementDirection(new Variable<Vector3>());
        entity.AddBehaviour<MovementBehaviour>();
        entity.AddBehaviour<InputBehaviour>();
    }
}
```

**➡️ См. подробно:** [Beginner Demo Guide](beginner-demo-guide.md)

---

#### Level 2: Shooter (Многоуровневая архитектура контекстов)
**Когда использовать:** Средние проекты с иерархией систем (50-500 entities)

**Характеристики:**
- Иерархия контекстов (App → Game → Player → Entity)
- Модульные Installers
- Controllers (связь контекстов)
- UseCases (бизнес-логика)
- Presenter Pattern (UI)

**Пример иерархии:**
```
AppContext (Singleton)
    ↓ управляет
GameContext (Singleton)
    ↓ содержит
PlayerContext[] (Multiple)
    ↓ владеет
GameEntity (Actor)
```

**Пример Controller:**
```csharp
public sealed class CharacterMoveController : IEntityInit<IPlayerContext>, IEntityTick
{
    private readonly IGameContext _gameContext;
    private IActor _character;
    private IPlayerContext _playerContext;

    public CharacterMoveController(IGameContext gameContext)
    {
        _gameContext = gameContext;
    }

    public void Init(IPlayerContext context)
    {
        _character = context.GetCharacter();
        _playerContext = context;
    }

    public void Tick(IEntity entity, float deltaTime)
    {
        Vector3 moveDirection = MoveInputUseCase.GetMoveDirection(_playerContext, _gameContext);
        _character.GetMovementDirection().Value = moveDirection;
    }
}
```

**➡️ См. подробно:** [Shooter Demo Guide](shooter-demo-guide.md)

---

#### Level 3: RTS (Production-Grade архитектура)
**Когда использовать:** Крупные проекты, RTS, симуляции (1000+ entities)

**Характеристики:**
- Core/Content/View separation
- Factory + Catalog + Pool
- EntityWorld для batch операций
- Burst Compilation
- SpatialHash для оптимизации запросов
- Static buffers (zero allocation)

**Пример структуры:**
```
Units/
├── Core/                   # Переиспользуемые системы
│   ├── Move/              # Movement installer + UseCase
│   ├── Combat/            # Combat installer + UseCase
│   └── AI/                # AI installer + Behaviour
├── Content/               # Конкретные типы
│   ├── Tank/             # TankFactory (Range Combat)
│   └── Warrior/          # WarriorFactory (Melee Combat)
└── View/                 # Презентация
    ├── PositionView/
    └── TeamColorView/
```

**Пример оптимизированного UseCase:**
```csharp
[BurstCompile]
public static class MoveUseCase
{
    // High-level wrapper
    public static void MoveStep(IUnit entity, Vector3 direction, float deltaTime)
    {
        IReactiveVariable<Vector3> position = entity.GetPosition();
        MoveStep(position.Value, direction, entity.GetMoveSpeed().Value,
                 deltaTime, out float3 next);
        position.Value = next;
    }

    // Burst-compiled pure function
    [BurstCompile]
    public static void MoveStep(in float3 position, in float3 direction,
                                in float speed, in float deltaTime, out float3 result)
        => result = position + speed * deltaTime * direction;
}
```

**➡️ См. подробно:** [RTS Demo Guide](rts-demo-guide.md)

---

### Best Practices - Ссылки на документацию

Рекомендуемые практики находятся в `Docs/BestPractices/`:

**Архитектурные паттерны:**
- [Modular Entity Installers](../Docs/BestPractices/ModularEntityInstallers.md) - 3 подхода к организации
- [Request-Condition-Action-Event](../Docs/BestPractices/RequestConditionActionEvent.md) - RCAE flow
- [Entity System](../Docs/BestPractices/EntitySystem.md) - Factory/Pool/World/View separation
- [Project Folder Organization](../Docs/BestPractices/ProjectFolderOrganization.md) - структура проекта

**Коммуникационные паттерны:**
- [Using Requests](../Docs/BestPractices/UsingRequests.md) - Producer-Consumer pattern
- [Using Events](../Docs/BestPractices/UsingEvents.md) - event-driven architecture
- [Using Observe](../Docs/BestPractices/UsingObserveWithReactiveValues.md) - reactive подписки

**Управление жизненным циклом:**
- [Using Subscriptions with DisposeComposite](../Docs/BestPractices/UsingSubscriptionsWithDisposeComposite.md)
- [Uninstall Entity Installer](../Docs/BestPractices/UninstallEntityInstaller.md)

**Производительность:**
- [Iterating Over Entity Collections](../Docs/BestPractices/IteratingOverEntityCollections.md)
- [Using Cooldown in Game Mechanics](../Docs/BestPractices/UsingCooldownInGameMechanics.md)

**Абстракции:**
- [Prefer Abstract Interfaces](../Docs/BestPractices/PreferAbstractInterfaces.md) - IValue вместо Const
- [Using Expressions](../Docs/BestPractices/UsingExpressions.md) - conditional logic

---

### Дальнейшее изучение

1. **Best Practices** документация в `Docs/BestPractices/`
2. **Tutorials** в `Docs/Tutorials/`
3. **API Reference** в `Docs/Entities/` и `Docs/Elements/`
4. **Demo гайдлайны** в `AI/`
5. **Feature Guides** (Рекомендуется к прочтению):
   - [Feature Decomposition Guide](feature-decomposition-guide.md) - пошаговый процесс внедрения
   - [Feature Checklist](feature-checklist.md) - чеклист для самопроверки
   - [Presenter Pattern Guide](presenter-pattern-guide.md) - MVP паттерн для UI

Следуя этому гайду, вы сможете создавать масштабируемые, производительные и легко поддерживаемые игры на Unity с использованием Atomic Framework v2.

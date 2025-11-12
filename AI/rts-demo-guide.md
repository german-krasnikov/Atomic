# Гайд по RTS Demo - Продвинутые паттерны и масштабирование

## Введение

RTS Demo демонстрирует **продвинутые паттерны** для создания масштабируемых игр с тысячами сущностей. Этот пример показывает как организовать сложную систему с AI, боевыми механиками и высокой производительностью.

## Обзор игры

**Концепция:** RTS-игра с автоматическими боями. Две команды (BLUE и RED) сражаются между собой. Юниты автоматически ищут врагов, двигаются к ним и атакуют.

**Типы юнитов:**
- **Warrior** (Воин) - ближний бой
- **Tank** (Танк) - дальний бой с снарядами
- **Headquarters** (Штаб) - без боя, только HP

**Основные механики:**
- AI поиск и атака врагов
- Два типа боя (ближний/дальний)
- Система снарядов (Projectiles)
- Система здоровья и смерти
- Команды (BLUE vs RED)
- Масштабирование до 10000+ юнитов
- Object pooling для переиспользования

## Ключевая особенность: Factory Pattern + Modular Installers

### Философия: Composition через Installers

Юниты собираются из **переиспользуемых модульных Installers**, а не наследуются друг от друга.

```
Warrior = Transform + Move + Life + MeleeCombat + AI
Tank    = Transform + Move + Life + RangeCombat + AI
HQ      = Transform + Life
```

**Принцип:** Composition over Inheritance

## Архитектура проекта

### Структура файлов

```
Scripts/
├── Common/
│   ├── Health.cs                    # Система здоровья
│   └── Team/
│       ├── TeamType.cs              # Enum команд
│       └── TeamViewConfig.cs        # Конфигурация визуализации
├── GameContext/                     # Глобальный контекст игры
│   ├── GameContext.cs
│   ├── GameContextFactory.cs
│   ├── GameContextAPI.yml/.cs
│   ├── Units/                       # Система юнитов
│   │   ├── UnitsSystemInstaller.cs
│   │   └── UnitsUseCase.cs
│   └── Players/                     # Система игроков
│       ├── PlayerSystemInstaller.cs
│       └── PlayersUseCase.cs
├── PlayerContext/                   # Контекст игрока/команды
│   ├── PlayerContext.cs
│   ├── PlayerContextBuilder.cs
│   └── PlayerContextAPI.yml/.cs
└── UnitEntities/                    # Юниты
    ├── UnitEntity.cs
    ├── UnitFactory.cs
    ├── UnitCatalog.cs
    ├── UnitEntityAPI.yml/.cs
    ├── Core/                        # Переиспользуемые системы
    │   ├── Transform/
    │   │   ├── TransformEntityInstaller.cs
    │   │   ├── TransformUseCase.cs
    │   │   └── TransformGizmos.cs
    │   ├── Move/
    │   │   ├── MoveEntityInstaller.cs
    │   │   ├── MoveUseCase.cs
    │   │   └── RotateUseCase.cs
    │   ├── Life/
    │   │   ├── LifeEntityInstaller.cs
    │   │   ├── LifeBehaviour.cs
    │   │   └── LifeUseCase.cs
    │   ├── Combat/
    │   │   ├── MeleeCombatEntityInstaller.cs
    │   │   ├── RangeCombatEntityInstaller.cs
    │   │   ├── DamageUseCase.cs
    │   │   └── CombatUseCase.cs
    │   └── AI/
    │       ├── AIEntityInstaller.cs
    │       ├── AIDetectTargetBehaviour.cs
    │       └── AIAttackTargetBehaviour.cs
    ├── Content/                     # Конкретные типы юнитов
    │   ├── Units/
    │   │   ├── Warrior/
    │   │   │   └── WarriorFactory.cs
    │   │   ├── Tank/
    │   │   │   └── TankFactory.cs
    │   │   └── Headquarters/
    │   │       └── HeadquartersFactory.cs
    │   └── Projectile/
    │       └── ProjectileFactory.cs
    └── View/                        # Визуализация
        ├── PositionViewBehaviour.cs
        ├── RotationViewBehaviour.cs
        └── TeamColorViewBehaviour.cs
```

### Организация по слоям

```
Core/           # Базовые переиспользуемые системы
Content/        # Конкретные юниты (composition of Core)
View/           # Визуализация
```

## Модульные Installers - переиспользуемые компоненты

### TransformEntityInstaller - базовая трансформация

```csharp
[Serializable]
public sealed class TransformEntityInstaller : IEntityInstaller<IUnitEntity>
{
    [SerializeField]
    private Const<float> _scale = 1;

    public void Install(IUnitEntity entity)
    {
        entity.AddPosition(new ReactiveVector3());      // Реактивная позиция
        entity.AddRotation(new ReactiveQuaternion());   // Реактивный поворот
        entity.AddScale(_scale);                        // Масштаб
#if UNITY_EDITOR
        entity.AddBehaviour<TransformGizmos>();         // Gizmos в редакторе
#endif
    }
}
```

**Почему Reactive?** Position и Rotation реактивные для автоматического обновления View.

### MoveEntityInstaller - движение

```csharp
[Serializable]
public sealed class MoveEntityInstaller : IEntityInstaller<IUnitEntity>
{
    [SerializeField]
    private Const<float> _moveSpeed = 3;
    [SerializeField]
    private Const<float> _rotationSpeed = 12;

    public void Install(IUnitEntity entity)
    {
        entity.AddMoveableTag();
        entity.AddMoveSpeed(_moveSpeed);
        entity.AddRotationSpeed(_rotationSpeed);
        entity.AddMoveRequest(new Request<Vector3>());  // Request для движения
        entity.AddMoveEvent(new Event<Vector3>());      // Event при движении

        // Логика движения в FixedUpdate через WhenFixedTick
        entity.WhenFixedTick(deltaTime =>
        {
            if (LifeUseCase.IsAlive(entity) &&
                entity.GetMoveRequest().Consume(out Vector3 direction) &&
                direction != Vector3.zero)
            {
                MoveUseCase.MoveStep(entity, direction, deltaTime);
                RotateUseCase.RotateStep(entity, direction, deltaTime);
                entity.GetMoveEvent().Invoke(direction);
            }
        });
    }
}
```

**Паттерны:**
- ✅ **Request pattern**: Однократное потребляемое действие
- ✅ **WhenFixedTick**: Inline behaviour через lambda
- ✅ **UseCase delegation**: Логика в статических методах

### LifeEntityInstaller - система жизни

```csharp
[Serializable]
public sealed class LifeEntityInstaller : IEntityInstaller<IUnitEntity>
{
    [SerializeField]
    private int _health;

    public void Install(IUnitEntity entity)
    {
        IGameContext gameContext = GameContext.Instance;
        entity.AddDamageableTag();                      // Может получать урон
        entity.AddHealth(new Health(_health));          // Здоровье
        entity.AddTakeDamageEvent(new Event<int>());    // Event при уроне
        entity.AddBehaviour(new LifeBehaviour(gameContext)); // Behaviour смерти
    }
}

public sealed class LifeBehaviour : IEntityInit<IUnitEntity>, IEntityDispose
{
    private readonly IGameContext _gameContext;
    private Health _health;
    private IUnitEntity _entity;

    public LifeBehaviour(IGameContext gameContext)
    {
        _gameContext = gameContext;
    }

    public void Init(IUnitEntity entity)
    {
        _entity = entity;
        _health = entity.GetHealth();
        _health.OnHealthEmpty += this.OnHealthEmpty;  // Подписка на событие
    }

    public void Dispose(IEntity entity)
    {
        _health.OnHealthEmpty -= this.OnHealthEmpty;  // Отписка
    }

    private void OnHealthEmpty() =>
        UnitsUseCase.Despawn(_gameContext, _entity);  // Удаляем юнит
}
```

**Паттерн:** Behaviour с зависимостями через конструктор

### MeleeCombatEntityInstaller - ближний бой

```csharp
[Serializable]
public sealed class MeleeCombatEntityInstaller : IEntityInstaller<IUnitEntity>
{
    [SerializeField]
    private float _fireCooldown = 1;
    [SerializeField]
    private Const<float> _fireDistance = 1;
    [SerializeField]
    private Const<int> _damage;

    public void Install(IUnitEntity entity)
    {
        entity.AddDamage(_damage);
        entity.AddFireCooldown(new Cooldown(_fireCooldown));
        entity.AddFireRequest(new Request<IUnitEntity>());
        entity.AddFireDistance(_fireDistance);
        entity.AddFireEvent(new Event<IUnitEntity>());

        // Логика атаки через WhenFixedTick
        entity.WhenFixedTick(_ =>
        {
            if (LifeUseCase.IsAlive(entity) &&
                entity.GetFireCooldown().IsCompleted() &&
                entity.GetFireRequest().Consume(out IUnitEntity target))
            {
                DamageUseCase.DealDamage(entity, target);  // Прямой урон
                entity.GetFireCooldown().ResetTime();
                entity.GetFireEvent().Invoke(target);
            }
        });
        entity.WhenFixedTick(entity.GetFireCooldown().Tick);
    }
}
```

**Ближний бой:** Наносит урон напрямую при атаке

### RangeCombatEntityInstaller - дальний бой

```csharp
[Serializable]
public sealed class RangeCombatEntityInstaller : IEntityInstaller<IUnitEntity>
{
    [SerializeField]
    private Const<float> _fireDistance = 5;
    [SerializeField]
    private float _fireCooldown = 1;
    [SerializeField]
    private Const<Vector3> _fireOffset = new Vector3(0, 1, 1);
    [SerializeField]
    private ProjectileFactory _projectileFactory;

    public void Install(IUnitEntity entity)
    {
        var gameContext = GameContext.Instance;

        entity.AddFireCooldown(new Cooldown(_fireCooldown));
        entity.AddFirePoint(new InlineValue<Vector3>(
            () => CombatUseCase.GetFirePoint(entity, _fireOffset.Value)
        ));
        entity.AddFireRequest(new Request<IUnitEntity>());
        entity.AddFireDistance(_fireDistance);
        entity.AddFireEvent(new Event<IUnitEntity>());

        // Логика стрельбы через WhenFixedTick
        entity.WhenFixedTick(_ =>
        {
            if (LifeUseCase.IsAlive(entity) &&
                entity.GetFireCooldown().IsCompleted() &&
                entity.GetFireRequest().Consume(out IUnitEntity target))
            {
                // Создаем снаряд
                CombatUseCase.FireProjectile(entity, _projectileFactory.name, target, gameContext);
                entity.GetFireCooldown().ResetTime();
                entity.GetFireEvent().Invoke(target);
            }
        });
        entity.WhenFixedTick(entity.GetFireCooldown().Tick);
    }
}
```

**Дальний бой:** Создает снаряды вместо прямого урона

**Паттерн InlineValue:**
```csharp
entity.AddFirePoint(new InlineValue<Vector3>(
    () => CombatUseCase.GetFirePoint(entity, _fireOffset.Value)
));
```
Вычисляется динамически при каждом обращении.

### AIEntityInstaller - искусственный интеллект

```csharp
[Serializable]
public sealed class AIEntityInstaller : IEntityInstaller<IUnitEntity>
{
    [SerializeField]
    private float _minDetectDuration = 0.2f;
    [SerializeField]
    private float _maxDetectDuration = 0.3f;

    public void Install(IUnitEntity entity)
    {
        IGameContext gameContext = GameContext.Instance;
        entity.AddTarget(new ReactiveVariable<IUnitEntity>());

        // Два поведения AI
        entity.AddBehaviour(new AIDetectTargetBehaviour(
            new RandomCooldown(_minDetectDuration, _maxDetectDuration),
            gameContext
        ));
        entity.AddBehaviour<AIAttackTargetBehaviour>();
    }
}
```

**Двухкомпонентная архитектура AI:**
- **AIDetectTargetBehaviour** - выбор цели
- **AIAttackTargetBehaviour** - атака цели

## Factory Pattern - создание юнитов

### Абстрактная фабрика

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
        this.Install(entity);  // Абстрактный метод установки компонентов
        return entity;
    }

    protected abstract void Install(IUnitEntity entity);
}
```

### WarriorFactory - конкретная фабрика

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

        // Композиция компонентов через Installers
        entity.Install(_transformInstaller);
        entity.Install(_moveInstaller);
        entity.Install(_lifeInstaller);
        entity.Install(_meleeCombatInstaller);  // Ближний бой
        entity.Install(_aiInstaller);
    }
}
```

**Warrior = Transform + Move + Life + MeleeCombat + AI**

### TankFactory - другая композиция

```csharp
[CreateAssetMenu]
public sealed class TankFactory : UnitFactory
{
    [SerializeField] private TransformEntityInstaller _transformInstaller;
    [SerializeField] private MoveEntityInstaller _moveInstaller;
    [SerializeField] private LifeEntityInstaller _lifeInstaller;
    [SerializeField] private RangeCombatEntityInstaller _combatInstaller;
    [SerializeField] private AIEntityInstaller _aiInstaller;

    protected override void Install(IUnitEntity entity)
    {
        entity.AddUnitTag();
        entity.AddTeam(new ReactiveVariable<TeamType>());

        entity.Install(_transformInstaller);
        entity.Install(_moveInstaller);
        entity.Install(_lifeInstaller);
        entity.Install(_combatInstaller);      // Дальний бой!
        entity.Install(_aiInstaller);
    }
}
```

**Tank = Transform + Move + Life + RangeCombat + AI**

### HeadquartersFactory - минимальная композиция

```csharp
[CreateAssetMenu]
public sealed class HeadquartersFactory : UnitFactory
{
    [SerializeField] private TransformEntityInstaller _transformInstaller;
    [SerializeField] private LifeEntityInstaller _lifeInstaller;

    protected override void Install(IUnitEntity entity)
    {
        entity.AddUnitTag();
        entity.AddTeam(new ReactiveVariable<TeamType>());

        // Только Transform и Life, без движения и боя
        entity.Install(_transformInstaller);
        entity.Install(_lifeInstaller);
    }
}
```

**HQ = Transform + Life**

### ProjectileFactory - специальная фабрика

```csharp
[CreateAssetMenu]
public sealed class ProjectileFactory : UnitFactory
{
    [SerializeField] private Const<float> _moveSpeed = 3;
    [SerializeField] private Const<int> _damage;
    [SerializeField] private float _lifetime;

    protected override void Install(IUnitEntity entity)
    {
        IGameContext context = GameContext.Instance;
        entity.AddProjectileTag();  // Не Unit, а Projectile

        _transformInstaller.Install(entity);

        entity.AddDamage(_damage);
        entity.AddMoveSpeed(_moveSpeed);
        entity.AddLifetime(new Cooldown(_lifetime));
        entity.AddTarget(new ReactiveVariable<IUnitEntity>());
        entity.AddTeam(new ReactiveVariable<TeamType>());

        entity.AddBehaviour(new ProjectileLifetimeBehaviour(context));
        entity.AddBehaviour(new ProjectileMoveBehaviour(context));
    }
}
```

## UnitCatalog - каталог фабрик

```csharp
[CreateAssetMenu]
public sealed class UnitCatalog : ScriptableMultiEntityFactory<string, IUnitEntity, UnitFactory>
{
    protected override string GetKey(UnitFactory factory) => factory.Name;
}
```

**Использование:**

```csharp
// В UnitsSystemInstaller
context.AddEntityPool(new MultiEntityPool<string, IUnitEntity>(factoryCatalog));

// Затем можно создавать по имени
IUnitEntity warrior = pool.Rent("Warrior");
IUnitEntity tank = pool.Rent("Tank");
```

**Преимущества:**
- ✅ Централизованное управление фабриками
- ✅ Создание юнитов по строковому ключу
- ✅ Интеграция с Object Pooling

## Builder Pattern - PlayerContextBuilder

Для сложных объектов с валидацией используется Builder Pattern.

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
        var playerContext = new PlayerContext(
            $"PlayerContext {_teamType}",
            this.initialTagCapacity,
            this.initialValueCapacity,
            this.initialBehaviourCapacity
        );

        // Устанавливаем команду
        playerContext.AddTeam(new Const<TeamType>(_teamType));

        // Создаем фильтр для поиска свободных врагов
        playerContext.AddFreeEnemyFilter(this.CreateFreeEnemyFilter(playerContext));

        return playerContext;
    }

    private EntityFilter<IUnitEntity> CreateFreeEnemyFilter(IPlayerContext playerContext) =>
        new(
            _entityWorld,
            entity => TeamUseCase.IsFreeEnemyUnit(playerContext, entity),
            new TeamEntityTrigger(),           // Обновление при смене команды
            new TagEntityTrigger<IUnitEntity>() // Обновление при изменении тегов
        );
}
```

**Использование:**

```csharp
// Создаем два контекста игроков
context.AddPlayers(new Dictionary<TeamType, IPlayerContext>
{
    {TeamType.BLUE, this.CreatePlayerContext(TeamType.BLUE)},
    {TeamType.RED, this.CreatePlayerContext(TeamType.RED)}
});

private IPlayerContext CreatePlayerContext(TeamType teamType) =>
    _playerFactory.SetTeamType(teamType).Create();
```

**Преимущества Builder:**
- ✅ Fluent API для конфигурации
- ✅ Валидация перед созданием
- ✅ Step-by-step конфигурация

## EntityFilter - динамические выборки

EntityFilter автоматически фильтрует entities по условиям и обновляется при изменениях.

### Создание фильтра

```csharp
private EntityFilter<IUnitEntity> CreateFreeEnemyFilter(IPlayerContext playerContext) =>
    new(
        _entityWorld,                          // Источник entities
        entity => TeamUseCase.IsFreeEnemyUnit(playerContext, entity),  // Условие
        new TeamEntityTrigger(),               // Триггер 1: изменение команды
        new TagEntityTrigger<IUnitEntity>()    // Триггер 2: изменение тегов
    );
```

### Условие фильтра

```csharp
public static class TeamUseCase
{
    public static bool IsFreeEnemyUnit(IPlayerContext context, IUnitEntity target) =>
        !target.HasTargetedTag() &&      // Не занят другим юнитом
        target.HasUnitTag() &&           // Это юнит (не снаряд)
        target.GetTeam().Value != context.GetTeam().Value;  // Вражеская команда
}
```

### Использование фильтра

```csharp
// Получение фильтра
EntityFilter<IUnitEntity> enemyFilter = playerContext.GetFreeEnemyFilter();

// Итерация (автоматически только подходящие юниты)
foreach (IUnitEntity enemy in enemyFilter)
{
    // Обрабатываем только свободных врагов
}
```

**Преимущества EntityFilter:**
- ✅ Автоматическое обновление при изменениях
- ✅ Высокая производительность
- ✅ Декларативный синтаксис

## AI система

### Двухкомпонентная архитектура

**AIDetectTargetBehaviour** - периодически ищет цель:

```csharp
public sealed class AIDetectTargetBehaviour :
    IEntityInit<IUnitEntity>, IEntityFixedTick, IEntityDisable
{
    private readonly IGameContext _gameContext;
    private readonly ICooldown _cooldown;
    private IEntityWorld<IUnitEntity> _entityWorld;
    private IVariable<IUnitEntity> _target;
    private IUnitEntity _entity;

    public AIDetectTargetBehaviour(ICooldown cooldown, IGameContext gameContext)
    {
        _cooldown = cooldown;
        _gameContext = gameContext;
    }

    public void Init(IUnitEntity entity)
    {
        _entity = entity;
        _target = entity.GetTarget();
        _entityWorld = _gameContext.GetEntityWorld();
    }

    public void FixedTick(IEntity entity, float deltaTime)
    {
        _cooldown.Tick(deltaTime);

        // Периодически ищем новую цель, если текущая недоступна
        if (_cooldown.IsCompleted() && !_entityWorld.Contains(_target.Value))
        {
            this.AssignTarget();
            _cooldown.ResetTime();
        }
    }

    private void AssignTarget()
    {
        IUnitEntity enemy = UnitsUseCase.FindFreeEnemyFor(_gameContext, _entity);
        if (enemy != null)
            enemy.AddTargetedTag();  // Помечаем врага как занятого

        _target.Value = enemy;
    }

    public void Disable(IEntity entity)
    {
        IUnitEntity target = _target.Value;
        if (target != null)
            target.DelTargetedTag();  // Снимаем метку при отключении
    }
}
```

**Паттерн Targeted Tag:** Предотвращает ситуацию, когда все юниты атакуют одну цель.

**AIAttackTargetBehaviour** - атакует цель:

```csharp
public sealed class AIAttackTargetBehaviour : IEntityInit<IUnitEntity>, IEntityFixedTick
{
    private IUnitEntity _entity;
    private IValue<IUnitEntity> _target;
    private IValue<float> _scale;

    public void Init(IUnitEntity entity)
    {
        _entity = entity;
        _scale = entity.GetScale();
        _target = entity.GetTarget();
    }

    public void FixedTick(IEntity entity, float deltaTime)
    {
        IUnitEntity target = _target.Value;
        if (target is not {Enabled: true} || !LifeUseCase.IsAlive(target))
            return;

        Vector3 vector = TransformUseCase.GetVector(_entity, target);
        float fullDistance = _entity.GetFireDistance().Value + _scale.Value + target.GetScale().Value;

        // Если далеко — двигаемся, если близко — стреляем
        if (vector.magnitude > fullDistance)
            _entity.GetMoveRequest().Invoke(vector.normalized);  // REQUEST: двигаться
        else
            _entity.GetFireRequest().Invoke(target);             // REQUEST: стрелять
    }
}
```

### Алгоритм поиска ближайшего врага

```csharp
public static class UnitsUseCase
{
    public static IUnitEntity FindFreeEnemyFor(IGameContext context, IUnitEntity entity)
    {
        IPlayerContext playerContext = PlayersUseCase.GetPlayerFor(context, entity);
        EntityFilter<IUnitEntity> enemyFilter = playerContext.GetFreeEnemyFilter();
        Vector3 center = entity.GetPosition().Value;
        return FindClosest(enemyFilter, center);
    }

    public static IUnitEntity FindClosest(EntityFilter<IUnitEntity> entities, Vector3 center)
    {
        IUnitEntity result = null;
        float minDistance = float.MaxValue;

        foreach (IUnitEntity entity in entities)  // Фильтр автоматически выбирает подходящих
        {
            Vector3 position = entity.GetPosition().Value;
            float distance = Vector3.SqrMagnitude(position - center);
            if (distance < minDistance)
            {
                result = entity;
                minDistance = distance;
            }
        }
        return result;
    }
}
```

## UseCases с Burst Compilation

### MoveUseCase - оптимизированное движение

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

**Burst Compilation:** Компилирует в оптимизированный машинный код для высокой производительности.

### RotateUseCase - оптимизированный поворот

```csharp
[BurstCompile]
public static class RotateUseCase
{
    public static void RotateStep(IUnitEntity entity, Vector3 direction, float deltaTime)
    {
        IReactiveVariable<Quaternion> rotation = entity.GetRotation();
        RotateStep(
            rotation.Value,
            direction,
            entity.GetRotationSpeed().Value,
            deltaTime,
            out quaternion next
        );
        rotation.Value = next;
    }

    [BurstCompile]
    public static void RotateStep(
        in quaternion current,
        in float3 direction,
        in float speedDeg,
        in float deltaTime,
        out quaternion result)
    {
        if (math.lengthsq(direction) < 1e-4f)
        {
            result = current;
            return;
        }

        quaternion target = quaternion.LookRotation(math.normalize(direction), math.up());
        Angle(in current, in target, out float angle);

        float maxStep = speedDeg * deltaTime;
        if (angle <= maxStep)
            result = target;
        else
        {
            float t = maxStep / angle;
            result = math.slerp(current, target, t);
        }
    }
}
```

**Использование Unity Mathematics:** `float3`, `quaternion` для оптимизации.

## Object Pooling для масштабирования

### EntityPool + EntityWorld

```csharp
public sealed class UnitsSystemInstaller : IEntityInstaller<IGameContext>
{
    [SerializeField]
    private UnitCatalog factoryCatalog;

    public void Install(IGameContext context)
    {
        // 1. Добавляем пул для создания юнитов
        context.AddEntityPool(new MultiEntityPool<string, IUnitEntity>(factoryCatalog));

        // 2. Создаем мир для всех юнитов
        EntityWorld<IUnitEntity> entityWorld = new EntityWorld<IUnitEntity>();
        context.AddEntityWorld(entityWorld);

        // 3. Привязываем жизненный цикл мира к контексту
        context.WhenInit(entityWorld.InitEntities);
        context.WhenEnable(entityWorld.Enable);
        context.WhenTick(entityWorld.Tick);
        context.WhenFixedTick(entityWorld.FixedTick);
        context.WhenLateTick(entityWorld.LateTick);
        context.WhenDisable(entityWorld.Disable);
        context.WhenDispose(entityWorld.DisposeEntities);
        context.WhenDispose(entityWorld.Dispose);
    }
}
```

### Spawn/Despawn с пулом

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
        IMultiEntityPool<string, IUnitEntity> pool = context.GetEntityPool();
        IUnitEntity entity = pool.Rent(name);  // Берем из пула или создаем новый
        entity.GetPosition().Value = position;
        entity.GetRotation().Value = rotation;
        entity.GetTeam().Value = team;
        context.GetEntityWorld().Add(entity);  // Добавляем в мир
        return entity;
    }

    public static bool Despawn(IGameContext gameContext, IUnitEntity entity)
    {
        if (!gameContext.GetEntityWorld().Remove(entity))
            return false;

        gameContext.GetEntityPool().Return(entity);  // Возвращаем в пул
        return true;
    }
}
```

**Преимущества pooling:**
- ✅ Избегание создания/удаления GameObject
- ✅ Переиспользование Entity
- ✅ Поддержка 10000+ юнитов

## Система View (визуализация)

### Архитектура Model-View

```
UnitEntity (Model) ←→ UnitView (MonoBehaviour)
```

Модель и визуализация разделены. View подписывается на изменения Model через Reactive values.

### PositionViewBehaviour - синхронизация позиции

```csharp
[Serializable]
public sealed class PositionViewBehaviour : IEntityInit<IUnitEntity>, IEntityDispose
{
    [SerializeField]
    private Transform _transform;
    private IReactiveValue<Vector3> _position;

    public void Init(IUnitEntity entity)
    {
        _position = entity.GetPosition();
        _position.Observe(this.OnPositionChanged);  // Подписка на изменения
    }

    public void Dispose(IEntity entity)
    {
        _position.Unsubscribe(this.OnPositionChanged);
    }

    private void OnPositionChanged(Vector3 position)
    {
        _transform.position = position;  // Синхронизация
    }
}
```

**Паттерн Observer:** View автоматически обновляется при изменении Model.

### TeamColorViewBehaviour - цвет команды

```csharp
[Serializable]
public sealed class TeamColorViewBehaviour : IEntityInit<IUnitEntity>, IEntityDispose
{
    [SerializeField]
    private Renderer[] _renderers;
    [SerializeField]
    private TeamViewConfig _viewConfig;
    private IReactiveValue<TeamType> _team;

    public void Init(IUnitEntity entity)
    {
        _team = entity.GetTeam();
        _team.Observe(this.OnTeamChanged);
    }

    public void Dispose(IEntity entity)
    {
        _team.Unsubscribe(this.OnTeamChanged);
    }

    private void OnTeamChanged(TeamType teamType)
    {
        TeamViewConfig.TeamInfo team = _viewConfig.GetTeam(teamType);
        RendererUseCase.SetMaterial(_renderers, team.Material);
    }
}
```

### TakeDamageViewBehaviour - анимация урона

```csharp
[Serializable]
public sealed class TakeDamageViewBehaviour : IEntityInit<IUnitEntity>, IEntityDispose<IUnitEntity>
{
    [SerializeField]
    private Transform _visual;
    [SerializeField]
    private Vector3 _originalScale = Vector3.one;

    public void Init(IUnitEntity entity)
    {
        entity.GetTakeDamageEvent().Subscribe(this.OnTakeDamage);
    }

    public void Dispose(IUnitEntity entity)
    {
        entity.GetTakeDamageEvent().Unsubscribe(this.OnTakeDamage);
    }

    private void OnTakeDamage(int _)
    {
        _visual.DOKill();
        _visual.localScale = _originalScale;
        _visual
            .DOScale(_originalScale * 1.1f, 0.1f)
            .SetEase(Ease.OutQuad)
            .OnComplete(() => _visual.DOScale(_originalScale, 0.2f).SetEase(Ease.OutBounce));
    }
}
```

**Паттерн Event-driven:** View реагирует на события Model.

## Боевая система

### Два типа боя

**Ближний бой (Melee):**
```
AIAttackTargetBehaviour
    ↓ close enough
FireRequest.Invoke(target)
    ↓
MeleeCombatEntityInstaller.WhenFixedTick
    ↓ consumes request
DamageUseCase.DealDamage(entity, target)
    ↓
Health.Reduce()
```

**Дальний бой (Range):**
```
AIAttackTargetBehaviour
    ↓ close enough
FireRequest.Invoke(target)
    ↓
RangeCombatEntityInstaller.WhenFixedTick
    ↓ consumes request
CombatUseCase.FireProjectile()
    ↓
BulletEntity создается
    ↓
ProjectileMoveBehaviour летит к цели
    ↓ достигает
DamageUseCase.DealDamage()
    ↓
Health.Reduce()
```

### DamageUseCase

```csharp
public static class DamageUseCase
{
    public static bool DealDamage(IUnitEntity source, IUnitEntity target) =>
        TakeDamage(target, source.GetDamage().Value);

    public static bool TakeDamage(IUnitEntity target, int damage)
    {
        if (!target.HasDamageableTag())
            return false;

        Health health = target.GetHealth();
        if (health.IsEmpty() || !health.Reduce(damage))
            return false;

        target.GetTakeDamageEvent().Invoke(damage);  // EVENT: получен урон
        return true;
    }
}
```

## Процедурная генерация юнитов

```csharp
public static class InitGameCase
{
    public static void SpawnUnits(IGameContext context, int columns = 10)
    {
        const float step = 10f;
        const float offsetBetweenTeams = -50f;

        for (int i = 0; i < columns; i++)
        {
            float x = i * step;

            // Синие юниты
            SpawnRow(context, TeamType.BLUE, x, 0f, Quaternion.identity);

            // Красные юниты
            SpawnRow(context, TeamType.RED, x, offsetBetweenTeams, Quaternion.Euler(0, 180, 0));
        }
    }

    private static void SpawnRow(IGameContext context, TeamType team, float x, float zOffset, Quaternion rotation)
    {
        // Пехота ближе
        Vector3 infantryPos = new Vector3(x, 0, -zOffset);
        UnitsUseCase.Spawn(context, "Warrior", infantryPos, rotation, team);

        // Танки дальше
        Vector3 tankPos = new Vector3(x, 0, -zOffset - 3f);
        UnitsUseCase.Spawn(context, "Tank", tankPos, rotation, team);

        // Штаб в тылу
        Vector3 hqPos = new Vector3(x, 0, -zOffset - 6f);
        UnitsUseCase.Spawn(context, "Headquarters", hqPos, rotation, team);
    }
}
```

## Best Practices

### 1. Composition через Modular Installers

```csharp
// Warrior = Transform + Move + Life + MeleeCombat + AI
entity.Install(_transformInstaller);
entity.Install(_moveInstaller);
entity.Install(_lifeInstaller);
entity.Install(_meleeCombatInstaller);
entity.Install(_aiInstaller);
```

### 2. WhenFixedTick для Inline Behaviours

```csharp
entity.WhenFixedTick(deltaTime =>
{
    if (entity.GetMoveRequest().Consume(out Vector3 direction))
    {
        MoveUseCase.MoveStep(entity, direction, deltaTime);
    }
});
```

### 3. Burst Compilation для критичных операций

```csharp
[BurstCompile]
public static void MoveStep(
    in float3 position,
    in float3 direction,
    in float speed,
    in float deltaTime,
    out float3 result
) => result = position + speed * deltaTime * direction;
```

### 4. EntityFilter для динамических выборок

```csharp
EntityFilter<IUnitEntity> enemyFilter = playerContext.GetFreeEnemyFilter();
foreach (IUnitEntity enemy in enemyFilter)
{
    // Только подходящие враги
}
```

### 5. Behaviours с зависимостями через конструктор

```csharp
public sealed class LifeBehaviour : IEntityInit<IUnitEntity>
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

### 6. InlineValue для динамических вычислений

```csharp
entity.AddFirePoint(new InlineValue<Vector3>(
    () => CombatUseCase.GetFirePoint(entity, _fireOffset.Value)
));
```

### 7. Request/Event паттерн

```csharp
// Request - однократное потребление
entity.GetMoveRequest().Invoke(direction);
if (entity.GetMoveRequest().Consume(out Vector3 dir))
{
    // Выполнить движение
}

// Event - множественные подписчики
entity.GetFireEvent().Subscribe(OnFire);
entity.GetFireEvent().Invoke(target);
```

## Производительность

### Оптимизации

1. **Burst Compilation** для MoveUseCase и RotateUseCase
2. **Object Pooling** для юнитов и снарядов
3. **EntityWorld** для эффективного управления lifecycle
4. **EntityFilter** для быстрых выборок
5. **AggressiveInlining** для API методов
6. **Кеширование** зависимостей в Init

### Результаты

- ✅ Поддержка 10000+ юнитов
- ✅ Стабильная производительность
- ✅ Минимальное использование GC

## Ключевые принципы

### 1. Composition over Inheritance

Юниты собираются из Installers, а не наследуются.

### 2. Modular Installers

Переиспользуемые компоненты (Transform, Move, Life, Combat, AI).

### 3. Factory + Catalog

Централизованное создание юнитов по имени.

### 4. Builder для сложных объектов

Fluent API для конфигурации PlayerContext.

### 5. EntityFilter для выборок

Автоматическое обновление при изменениях.

### 6. Burst Compilation

Оптимизация критичных операций.

### 7. Model-View Separation

Reactive подписки для автоматического обновления View.

### 8. Request/Event паттерн

Слабая связанность компонентов.

## Заключение

RTS Demo демонстрирует:

✅ **Масштабируемость**: 10000+ юнитов
✅ **Модульность**: Composition через Installers
✅ **Производительность**: Burst, Pooling, EntityWorld
✅ **Гибкость**: Factory, Builder, Catalog
✅ **Расширяемость**: Легко добавлять новые типы юнитов
✅ **Чистота кода**: Model-View separation, UseCases
✅ **Динамические выборки**: EntityFilter

### Формула успеха

```
Modular Installers (переиспользование) +
Factory Pattern (гибкое создание) +
EntityFilter (эффективные выборки) +
Burst Compilation (производительность) +
Object Pooling (масштабирование) =
Production-ready архитектура
```

### Применение знаний

После изучения всех трех демо (Beginner, Shooter, RTS) вы готовы:
1. Создавать масштабируемые игры
2. Организовывать сложные системы
3. Оптимизировать производительность
4. Применять продвинутые паттерны

**Следующий шаг:** Создание собственной игры с применением изученных принципов!

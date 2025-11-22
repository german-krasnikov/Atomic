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

---

# ЧАСТЬ 2: ПОШАГОВОЕ ВНЕДРЕНИЕ ФИЧ

## Обзор всех фич в RTS Demo

RTS Demo содержит **12 основных фич**, организованных по уровню сложности.

### Классификация по уровням

#### Уровень 1 (⭐): Foundation - Базовые данные

1. **Transform System** - Reactive позиция/ротация
2. **Team System** - Команды юнитов
3. **Health System** - Система жизни

#### Уровень 2 (⭐⭐): Core Mechanics - Основная механика

4. **Movement System** - Движение с Request паттерном
5. **Rotation System** - Поворот к направлению

#### Уровень 3 (⭐⭐⭐): Advanced Mechanics - Продвинутая механика

6. **Melee Combat System** - Ближний бой
7. **Range Combat System** - Дальний бой с снарядами
8. **AI System** - Двухкомпонентная AI (Detect + Attack)
9. **View System** - Model-View separation с Reactive

#### Уровень 4 (⭐⭐⭐⭐): Architecture Patterns - Архитектурные паттерны

10. **Factory Pattern** - Создание юнитов из переиспользуемых Installers
11. **EntityFilter Pattern** - Динамические выборки
12. **Object Pooling + EntityWorld** - Масштабирование до 10000+ юнитов

### Граф зависимостей фич

```
┌─────────────────────────────────────────────────────────────┐
│                    Level 1: Foundation                      │
│  [Transform (Reactive)] [Team] [Health]                    │
└──────────────┬──────────────────────────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────────────────────────┐
│                   Level 2: Core Mechanics                   │
│  [Movement (Request)] [Rotation]                           │
│  depends on: Transform                                      │
└──────────────┬──────────────────────────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────────────────────────┐
│                Level 3: Advanced Mechanics                  │
│  [MeleeCombat] [RangeCombat] [AI] [View]                  │
│  depends on: All Level 1+2                                  │
└──────────────┬──────────────────────────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────────────────────────┐
│              Level 4: Architecture Patterns                 │
│  [Factory] [EntityFilter] [Pooling+World]                  │
│  depends on: All Level 1+2+3                                │
└─────────────────────────────────────────────────────────────┘
```

---

## Feature 1: Reactive Transform System (⭐)

**Сложность:** Foundation
**Зависимости:** Нет
**Используется в:** Movement, View, AI

### Описание

В RTS Demo Transform использует **ReactiveVariable** вместо InlineVariable для автоматического обновления View при изменении позиции/ротации.

### Шаг 1: Добавление в EntityAPI

```csharp
public static class UnitEntityAPI
{
    public static readonly int Position;  // IReactiveVariable<Vector3>
    public static readonly int Rotation;  // IReactiveVariable<Quaternion>
    public static readonly int Scale;     // IValue<float>

    public static IReactiveVariable<Vector3> GetPosition(this IUnitEntity entity)
        => entity.GetValue<IReactiveVariable<Vector3>>(Position);

    public static void AddPosition(this IUnitEntity entity, IReactiveVariable<Vector3> value)
        => entity.AddValue(Position, value);

    public static IReactiveVariable<Quaternion> GetRotation(this IUnitEntity entity)
        => entity.GetValue<IReactiveVariable<Quaternion>>(Rotation);

    public static void AddRotation(this IUnitEntity entity, IReactiveVariable<Quaternion> value)
        => entity.AddValue(Rotation, value);

    public static IValue<float> GetScale(this IUnitEntity entity)
        => entity.GetValue<IValue<float>>(Scale);

    public static void AddScale(this IUnitEntity entity, IValue<float> value)
        => entity.AddValue(Scale, value);
}
```

**Принцип:** Reactive для автоматического обновления View

### Шаг 2: Создание Data Classes

```csharp
// ReactiveVector3 - реактивная позиция
public sealed class ReactiveVector3 : IReactiveVariable<Vector3>
{
    private Vector3 _value;
    private event Action<Vector3> _onChange;

    public ReactiveVector3() : this(Vector3.zero) { }

    public ReactiveVector3(Vector3 initialValue)
    {
        _value = initialValue;
    }

    public Vector3 Value
    {
        get => _value;
        set
        {
            if (_value != value)
            {
                _value = value;
                _onChange?.Invoke(_value);  // Автоматическое уведомление
            }
        }
    }

    public void Observe(Action<Vector3> callback)
    {
        _onChange += callback;
        callback?.Invoke(_value);  // Immediate call
    }

    public void Unsubscribe(Action<Vector3> callback)
    {
        _onChange -= callback;
    }
}

// ReactiveQuaternion - реактивная ротация
public sealed class ReactiveQuaternion : IReactiveVariable<Quaternion>
{
    private Quaternion _value;
    private event Action<Quaternion> _onChange;

    public ReactiveQuaternion() : this(Quaternion.identity) { }

    public ReactiveQuaternion(Quaternion initialValue)
    {
        _value = initialValue;
    }

    public Quaternion Value
    {
        get => _value;
        set
        {
            if (_value != value)
            {
                _value = value;
                _onChange?.Invoke(_value);
            }
        }
    }

    public void Observe(Action<Quaternion> callback)
    {
        _onChange += callback;
        callback?.Invoke(_value);
    }

    public void Unsubscribe(Action<Quaternion> callback)
    {
        _onChange -= callback;
    }
}
```

**Отличие от Shooter:** Reactive вместо InlineVariable для Model-View separation

### Шаг 3: Создание UseCases

```csharp
public static class TransformUseCase
{
    // Расчет вектора между двумя юнитами
    public static Vector3 GetVector(IUnitEntity from, IUnitEntity to)
    {
        Vector3 fromPos = from.GetPosition().Value;
        Vector3 toPos = to.GetPosition().Value;
        return toPos - fromPos;
    }

    // Расчет дистанции
    public static float GetDistance(IUnitEntity from, IUnitEntity to)
    {
        Vector3 vector = GetVector(from, to);
        return vector.magnitude;
    }

    // Расчет направления
    public static Vector3 GetDirection(IUnitEntity from, IUnitEntity to)
    {
        Vector3 vector = GetVector(from, to);
        return vector.normalized;
    }
}
```

### Шаг 4: Создание Behaviours

Transform не требует отдельных Behaviours - другие системы изменяют его напрямую.

**Пример использования в Movement:**

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
        position.Value = next;  // Автоматически уведомит View
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

### Шаг 5: Создание Installer

```csharp
[Serializable]
public sealed class TransformEntityInstaller : IEntityInstaller<IUnitEntity>
{
    [SerializeField] private Const<float> _scale = 1;

    public void Install(IUnitEntity entity)
    {
        // Reactive для автоматического обновления View
        entity.AddPosition(new ReactiveVector3());
        entity.AddRotation(new ReactiveQuaternion());
        entity.AddScale(_scale);

#if UNITY_EDITOR
        entity.AddBehaviour<TransformGizmos>();  // Debug визуализация
#endif
    }
}
```

### Шаг 6: Интеграция с другими фичами

**View System (автоматическое обновление):**

```csharp
[Serializable]
public sealed class PositionViewBehaviour : IEntityInit<IUnitEntity>, IEntityDispose
{
    [SerializeField] private Transform _transform;
    private IReactiveValue<Vector3> _position;

    public void Init(IUnitEntity entity)
    {
        _position = entity.GetPosition();
        // Подписка на изменения - автоматическая синхронизация
        _position.Observe(this.OnPositionChanged);
    }

    public void Dispose(IEntity entity)
    {
        _position.Unsubscribe(this.OnPositionChanged);
    }

    private void OnPositionChanged(Vector3 position)
    {
        _transform.position = position;  // Unity Transform синхронизируется автоматически
    }
}
```

**AI System (чтение позиции):**

```csharp
public void FixedTick(IEntity entity, float deltaTime)
{
    Vector3 vector = TransformUseCase.GetVector(_entity, target);
    float distance = vector.magnitude;

    if (distance > fullDistance)
        _entity.GetMoveRequest().Invoke(vector.normalized);
}
```

### Шаг 7: Best Practices и зависимости

**Зависимости:**
- 📦 Atomic.Elements (IReactiveVariable, ReactiveVariable)
- 📦 Unity.Mathematics (для Burst в UseCases)

**Best Practices:**

1. ✅ **ReactiveVariable для Model-View separation**
   ```csharp
   entity.AddPosition(new ReactiveVector3());  // НЕ InlineVariable
   ```

2. ✅ **Burst Compilation для трансформаций**
   ```csharp
   [BurstCompile]
   public static void MoveStep(in float3 position, in float3 direction, ...)
   {
       result = position + speed * deltaTime * direction;
   }
   ```

3. ✅ **Unity.Mathematics типы в Burst методах**
   ```csharp
   // float3 вместо Vector3
   // quaternion вместо Quaternion
   ```

4. ✅ **Подписка View на Reactive переменные**
   ```csharp
   _position.Observe(this.OnPositionChanged);
   // Автоматическое обновление при изменении
   ```

5. ⚠️ **ВСЕГДА отписывайтесь в Dispose**
   ```csharp
   public void Dispose(IEntity entity)
   {
       _position.Unsubscribe(this.OnPositionChanged);
   }
   ```

**Типичные ошибки:**

❌ **Ошибка 1:** InlineVariable вместо ReactiveVariable
```csharp
// НЕПРАВИЛЬНО - View не будет обновляться
entity.AddPosition(new TransformPositionVariable(transform));
```

✅ **Исправление:**
```csharp
// ПРАВИЛЬНО
entity.AddPosition(new ReactiveVector3());
```

❌ **Ошибка 2:** Vector3 в Burst методах
```csharp
// НЕПРАВИЛЬНО - не совместимо с Burst
[BurstCompile]
public static void MoveStep(Vector3 position, ...) { }
```

✅ **Исправление:**
```csharp
// ПРАВИЛЬНО
[BurstCompile]
public static void MoveStep(in float3 position, ...) { }
```

---

## Feature 2: Request Pattern для Movement (⭐⭐)

**Сложность:** Core Mechanics
**Зависимости:** Transform
**Используется в:** Movement, AI

### Описание

Request Pattern обеспечивает **однократное потребление** команды движения. В отличие от ReactiveVariable, Request вызывается один раз и потребляется.

### Шаг 1: Добавление в EntityAPI

```csharp
public static class UnitEntityAPI
{
    public static readonly int MoveRequest;    // Request<Vector3>
    public static readonly int MoveEvent;      // IEvent<Vector3>
    public static readonly int MoveSpeed;      // IValue<float>
    public static readonly int RotationSpeed;  // IValue<float>
    public static readonly int Moveable;       // Tag

    public static Request<Vector3> GetMoveRequest(this IUnitEntity entity)
        => entity.GetValue<Request<Vector3>>(MoveRequest);

    public static void AddMoveRequest(this IUnitEntity entity, Request<Vector3> value)
        => entity.AddValue(MoveRequest, value);

    public static IEvent<Vector3> GetMoveEvent(this IUnitEntity entity)
        => entity.GetValue<IEvent<Vector3>>(MoveEvent);

    public static void AddMoveEvent(this IUnitEntity entity, IEvent<Vector3> value)
        => entity.AddValue(MoveEvent, value);

    public static IValue<float> GetMoveSpeed(this IUnitEntity entity)
        => entity.GetValue<IValue<float>>(MoveSpeed);

    public static void AddMoveSpeed(this IUnitEntity entity, IValue<float> value)
        => entity.AddValue(MoveSpeed, value);

    public static bool HasMoveableTag(this IUnitEntity entity)
        => entity.HasTag(Moveable);

    public static bool AddMoveableTag(this IUnitEntity entity)
        => entity.AddTag(Moveable);
}
```

### Шаг 2: Создание Data Classes

```csharp
// Request - однократное потребляемое действие
public sealed class Request<T>
{
    private T _value;
    private bool _isPending;

    // Вызвать запрос
    public void Invoke(T value)
    {
        _value = value;
        _isPending = true;
    }

    // Попытаться получить и потребить запрос
    public bool Consume(out T value)
    {
        if (_isPending)
        {
            value = _value;
            _isPending = false;
            _value = default;
            return true;
        }

        value = default;
        return false;
    }

    // Проверить, есть ли запрос
    public bool IsPending() => _isPending;

    // Отменить запрос
    public void Cancel()
    {
        _isPending = false;
        _value = default;
    }
}
```

**Отличие от Event:** Request потребляется один раз, Event может иметь множество подписчиков

### Шаг 3: Создание UseCases

```csharp
[BurstCompile]
public static class MoveUseCase
{
    // Главная функция движения
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

    // Burst-compiled версия
    [BurstCompile]
    public static void MoveStep(
        in float3 position,
        in float3 direction,
        in float speed,
        in float deltaTime,
        out float3 result
    ) => result = position + speed * deltaTime * direction;
}

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
        // Проверка нулевого вектора
        if (math.lengthsq(direction) < 1e-4f)
        {
            result = current;
            return;
        }

        // Целевая ротация
        quaternion target = quaternion.LookRotation(math.normalize(direction), math.up());

        // Расчет угла
        Angle(in current, in target, out float angle);

        // Плавный поворот
        float maxStep = speedDeg * deltaTime;
        if (angle <= maxStep)
            result = target;
        else
        {
            float t = maxStep / angle;
            result = math.slerp(current, target, t);
        }
    }

    [BurstCompile]
    private static void Angle(in quaternion a, in quaternion b, out float result)
    {
        float dot = math.dot(a.value, b.value);
        result = math.degrees(2 * math.acos(math.abs(dot)));
    }
}
```

### Шаг 4: Создание Behaviours

Movement использует **Inline Behaviour** через WhenFixedTick:

```csharp
// В MoveEntityInstaller
entity.WhenFixedTick(deltaTime =>
{
    // Проверка: жив ли юнит
    if (!LifeUseCase.IsAlive(entity))
        return;

    // Попытка потребить Request
    if (entity.GetMoveRequest().Consume(out Vector3 direction) &&
        direction != Vector3.zero)
    {
        // Движение
        MoveUseCase.MoveStep(entity, direction, deltaTime);

        // Поворот
        RotateUseCase.RotateStep(entity, direction, deltaTime);

        // Событие движения
        entity.GetMoveEvent().Invoke(direction);
    }
});
```

**Преимущество Inline Behaviour:**
- ✅ Нет отдельного класса
- ✅ Логика рядом с данными
- ✅ Легко читается

### Шаг 5: Создание Installer

```csharp
[Serializable]
public sealed class MoveEntityInstaller : IEntityInstaller<IUnitEntity>
{
    [SerializeField] private Const<float> _moveSpeed = 3;
    [SerializeField] private Const<float> _rotationSpeed = 12;

    public void Install(IUnitEntity entity)
    {
        entity.AddMoveableTag();
        entity.AddMoveSpeed(_moveSpeed);
        entity.AddRotationSpeed(_rotationSpeed);
        entity.AddMoveRequest(new Request<Vector3>());
        entity.AddMoveEvent(new Event<Vector3>());

        // Inline Behaviour для движения
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

### Шаг 6: Интеграция с другими фичами

**AI System (Producer - вызывает Request):**

```csharp
public sealed class AIAttackTargetBehaviour : IEntityFixedTick
{
    public void FixedTick(IEntity entity, float deltaTime)
    {
        IUnitEntity target = _target.Value;
        if (target == null || !LifeUseCase.IsAlive(target))
            return;

        Vector3 vector = TransformUseCase.GetVector(_entity, target);
        float distance = vector.magnitude;

        if (distance > fullDistance)
        {
            // PRODUCER: вызывает Request
            _entity.GetMoveRequest().Invoke(vector.normalized);
        }
        else
        {
            _entity.GetFireRequest().Invoke(target);
        }
    }
}
```

**MoveEntityInstaller (Consumer - потребляет Request):**

```csharp
// CONSUMER: потребляет Request в WhenFixedTick
entity.WhenFixedTick(deltaTime =>
{
    if (entity.GetMoveRequest().Consume(out Vector3 direction))
    {
        // Выполнить движение
        MoveUseCase.MoveStep(entity, direction, deltaTime);
    }
});
```

### Шаг 7: Best Practices и зависимости

**Зависимости:**
- 📦 Transform System (Position, Rotation)
- 📦 Life System (для проверки IsAlive)
- 📦 Unity.Mathematics (для Burst)
- 📦 Unity.Burst

**Best Practices:**

1. ✅ **Request для однократных команд**
   ```csharp
   entity.GetMoveRequest().Invoke(direction);  // Вызов
   if (entity.GetMoveRequest().Consume(out Vector3 dir))  // Потребление
   {
       // Выполнить
   }
   ```

2. ✅ **Event для множественных подписчиков**
   ```csharp
   entity.GetMoveEvent().Subscribe(OnMove);  // Подписка
   entity.GetMoveEvent().Invoke(direction);  // Вызов для всех
   ```

3. ✅ **WhenFixedTick для Inline Behaviours**
   ```csharp
   entity.WhenFixedTick(deltaTime =>
   {
       // Логика движения
   });
   ```

4. ✅ **Burst Compilation для математики**
   ```csharp
   [BurstCompile]
   public static void MoveStep(in float3 position, ...) { }
   ```

5. ✅ **Проверка IsAlive перед движением**
   ```csharp
   if (LifeUseCase.IsAlive(entity) && entity.GetMoveRequest().Consume(...))
   {
       // Двигаться
   }
   ```

**Типичные ошибки:**

❌ **Ошибка 1:** Event вместо Request
```csharp
// НЕПРАВИЛЬНО - будет вызываться многократно
entity.GetMoveEvent().Invoke(direction);
```

✅ **Исправление:**
```csharp
// ПРАВИЛЬНО - однократное потребление
entity.GetMoveRequest().Invoke(direction);
```

❌ **Ошибка 2:** Забыли потребить Request
```csharp
// НЕПРАВИЛЬНО - Request не потребляется
Vector3 direction = entity.GetMoveRequest()._value;
```

✅ **Исправление:**
```csharp
// ПРАВИЛЬНО
if (entity.GetMoveRequest().Consume(out Vector3 direction))
{
    // Использовать direction
}
```

---

## Feature 3: AI System - Двухкомпонентная архитектура (⭐⭐⭐)

**Сложность:** Advanced Mechanics
**Зависимости:** Transform, Movement, Combat, Life, EntityFilter
**Используется в:** Warrior, Tank

### Описание

AI System разделена на два компонента:
- **AIDetectTargetBehaviour** - периодически ищет ближайшего врага
- **AIAttackTargetBehaviour** - атакует выбранную цель

### Шаг 1: Добавление в EntityAPI

```csharp
public static class UnitEntityAPI
{
    public static readonly int Target;         // IVariable<IUnitEntity>
    public static readonly int Targeted;       // Tag (помечает, что юнит уже чья-то цель)

    public static IVariable<IUnitEntity> GetTarget(this IUnitEntity entity)
        => entity.GetValue<IVariable<IUnitEntity>>(Target);

    public static void AddTarget(this IUnitEntity entity, IVariable<IUnitEntity> value)
        => entity.AddValue(Target, value);

    public static bool HasTargetedTag(this IUnitEntity entity)
        => entity.HasTag(Targeted);

    public static bool AddTargetedTag(this IUnitEntity entity)
        => entity.AddTag(Targeted);

    public static bool DelTargetedTag(this IUnitEntity entity)
        => entity.DelTag(Targeted);
}
```

### Шаг 2: Создание Data Classes

```csharp
// RandomCooldown - рандомная задержка между поисками
public sealed class RandomCooldown : ICooldown
{
    private readonly float _min;
    private readonly float _max;
    private float _duration;
    private float _currentTime;

    public RandomCooldown(float min, float max)
    {
        _min = min;
        _max = max;
        _duration = Random.Range(min, max);
        _currentTime = _duration;
    }

    public bool IsCompleted() => _currentTime >= _duration;

    public void ResetTime()
    {
        _duration = Random.Range(_min, _max);  // Новая случайная задержка
        _currentTime = 0;
    }

    public void Tick(float deltaTime)
    {
        if (_currentTime < _duration)
            _currentTime += deltaTime;
    }
}
```

### Шаг 3: Создание UseCases

```csharp
public static class UnitsUseCase
{
    // Поиск ближайшего свободного врага
    public static IUnitEntity FindFreeEnemyFor(IGameContext context, IUnitEntity entity)
    {
        IPlayerContext playerContext = PlayersUseCase.GetPlayerFor(context, entity);
        EntityFilter<IUnitEntity> enemyFilter = playerContext.GetFreeEnemyFilter();
        Vector3 center = entity.GetPosition().Value;
        return FindClosest(enemyFilter, center);
    }

    // Поиск ближайшего юнита из фильтра
    public static IUnitEntity FindClosest(EntityFilter<IUnitEntity> entities, Vector3 center)
    {
        IUnitEntity result = null;
        float minDistance = float.MaxValue;

        foreach (IUnitEntity entity in entities)
        {
            Vector3 position = entity.GetPosition().Value;
            float distance = Vector3.SqrMagnitude(position - center);  // sqrMagnitude быстрее
            if (distance < minDistance)
            {
                result = entity;
                minDistance = distance;
            }
        }

        return result;
    }
}

// UseCase для работы с командами
public static class TeamUseCase
{
    // Проверка, является ли юнит свободным врагом
    public static bool IsFreeEnemyUnit(IPlayerContext context, IUnitEntity target) =>
        !target.HasTargetedTag() &&           // Не занят другим юнитом
        target.HasUnitTag() &&                // Это юнит (не снаряд)
        target.GetTeam().Value != context.GetTeam().Value;  // Вражеская команда
}
```

### Шаг 4: Создание Behaviours

```csharp
// AIDetectTargetBehaviour - поиск целей
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
        // Найти ближайшего свободного врага
        IUnitEntity enemy = UnitsUseCase.FindFreeEnemyFor(_gameContext, _entity);

        if (enemy != null)
            enemy.AddTargetedTag();  // Помечаем как занятого

        _target.Value = enemy;
    }

    public void Disable(IEntity entity)
    {
        // Снимаем метку Targeted при отключении
        IUnitEntity target = _target.Value;
        if (target != null)
            target.DelTargetedTag();
    }
}

// AIAttackTargetBehaviour - атака цели
public sealed class AIAttackTargetBehaviour : IEntityInit<IUnitEntity>, IEntityFixedTick
{
    private IUnitEntity _entity;
    private IValue<IUnitEntity> _target;
    private IValue<float> _scale;
    private IValue<float> _fireDistance;

    public void Init(IUnitEntity entity)
    {
        _entity = entity;
        _scale = entity.GetScale();
        _target = entity.GetTarget();
        _fireDistance = entity.GetFireDistance();
    }

    public void FixedTick(IEntity entity, float deltaTime)
    {
        IUnitEntity target = _target.Value;

        // Проверка валидности цели
        if (target is not {Enabled: true} || !LifeUseCase.IsAlive(target))
            return;

        // Расчет вектора и дистанции
        Vector3 vector = TransformUseCase.GetVector(_entity, target);
        float fullDistance = _fireDistance.Value + _scale.Value + target.GetScale().Value;

        // Если далеко — двигаемся, если близко — стреляем
        if (vector.magnitude > fullDistance)
        {
            _entity.GetMoveRequest().Invoke(vector.normalized);  // REQUEST: двигаться
        }
        else
        {
            _entity.GetFireRequest().Invoke(target);             // REQUEST: стрелять
        }
    }
}
```

### Шаг 5: Создание Installer

```csharp
[Serializable]
public sealed class AIEntityInstaller : IEntityInstaller<IUnitEntity>
{
    [SerializeField] private float _minDetectDuration = 0.2f;
    [SerializeField] private float _maxDetectDuration = 0.3f;

    public void Install(IUnitEntity entity)
    {
        IGameContext gameContext = GameContext.Instance;

        // Добавление целевой переменной
        entity.AddTarget(new ReactiveVariable<IUnitEntity>());

        // Два behaviour для AI
        entity.AddBehaviour(new AIDetectTargetBehaviour(
            new RandomCooldown(_minDetectDuration, _maxDetectDuration),
            gameContext
        ));
        entity.AddBehaviour<AIAttackTargetBehaviour>();
    }
}
```

### Шаг 6: Интеграция с другими фичами

**EntityFilter (для поиска врагов):**

```csharp
// В PlayerContextBuilder
private EntityFilter<IUnitEntity> CreateFreeEnemyFilter(IPlayerContext playerContext) =>
    new(
        _entityWorld,                          // Источник entities
        entity => TeamUseCase.IsFreeEnemyUnit(playerContext, entity),  // Условие
        new TeamEntityTrigger(),               // Обновление при смене команды
        new TagEntityTrigger<IUnitEntity>()    // Обновление при изменении тегов
    );
```

**Movement System (получает Request от AI):**

```csharp
entity.WhenFixedTick(deltaTime =>
{
    if (entity.GetMoveRequest().Consume(out Vector3 direction))
    {
        MoveUseCase.MoveStep(entity, direction, deltaTime);
    }
});
```

**Combat System (получает Request от AI):**

```csharp
entity.WhenFixedTick(_ =>
{
    if (entity.GetFireRequest().Consume(out IUnitEntity target))
    {
        DamageUseCase.DealDamage(entity, target);
    }
});
```

### Шаг 7: Best Practices и зависимости

**Зависимости:**
- 📦 EntityFilter (для поиска врагов)
- 📦 Movement System (MoveRequest)
- 📦 Combat System (FireRequest)
- 📦 Transform System (GetVector)
- 📦 Life System (IsAlive)

**Best Practices:**

1. ✅ **Разделение на Detect и Attack**
   ```csharp
   entity.AddBehaviour<AIDetectTargetBehaviour>();  // Поиск
   entity.AddBehaviour<AIAttackTargetBehaviour>();  // Атака
   ```

2. ✅ **Targeted Tag для предотвращения фокусировки**
   ```csharp
   enemy.AddTargetedTag();  // Помечаем как занятого
   // Теперь другие юниты не выберут этого врага
   ```

3. ✅ **RandomCooldown для разнообразия**
   ```csharp
   new RandomCooldown(0.2f, 0.3f);  // Каждый юнит ищет с разной частотой
   ```

4. ✅ **Проверка валидности цели**
   ```csharp
   if (target is not {Enabled: true} || !LifeUseCase.IsAlive(target))
       return;
   ```

5. ✅ **EntityFilter для эффективного поиска**
   ```csharp
   EntityFilter<IUnitEntity> enemyFilter = playerContext.GetFreeEnemyFilter();
   // Автоматически фильтрует только подходящих врагов
   ```

**Типичные ошибки:**

❌ **Ошибка 1:** Все юниты атакуют одну цель
```csharp
// НЕПРАВИЛЬНО - нет Targeted Tag
IUnitEntity enemy = FindClosest(...);
_target.Value = enemy;
```

✅ **Исправление:**
```csharp
// ПРАВИЛЬНО
IUnitEntity enemy = FindClosest(...);
if (enemy != null)
    enemy.AddTargetedTag();  // Помечаем как занятого
_target.Value = enemy;
```

❌ **Ошибка 2:** Не снимается Targeted Tag
```csharp
// НЕПРАВИЛЬНО - враг останется "занятым" навсегда
public void Disable(IEntity entity) { }
```

✅ **Исправление:**
```csharp
// ПРАВИЛЬНО
public void Disable(IEntity entity)
{
    if (_target.Value != null)
        _target.Value.DelTargetedTag();
}
```

---

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

---

## 📖 Дополнительные ресурсы

**Связанные гайды:**

1. **Предыдущие уровни:**
   - [`beginner-demo-guide.md`](beginner-demo-guide.md) - базовые концепции Atomic Framework
   - [`shooter-demo-guide.md`](shooter-demo-guide.md) - иерархия контекстов, системы здоровья и оружия

2. **Справочные материалы:**
   - [`atomic-guide-v2.md`](atomic-guide-v2.md) - полное руководство по Atomic Framework
   - [`feature-decomposition-guide.md`](feature-decomposition-guide.md) - универсальный процесс добавления фич
   - [`feature-checklist.md`](feature-checklist.md) - чеклист для самопроверки

3. **Специализированные паттерны:**
   - [`presenter-pattern-guide.md`](presenter-pattern-guide.md) - полный гайд по паттерну Presenter для UI

**Рекомендуемый путь обучения:**
```
Beginner Demo → Shooter Demo → RTS Demo (вы здесь) → Собственный проект ✨
```

**Следующий шаг:** Создание собственной игры с применением изученных принципов!

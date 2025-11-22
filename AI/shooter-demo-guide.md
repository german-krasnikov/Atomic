# Гайд по Shooter Demo - Многоуровневая архитектура контекстов

## Введение

Shooter Demo демонстрирует **продвинутую многоуровневую архитектуру** с иерархией контекстов. Этот подход масштабируется от простых игр до сложных проектов с множеством систем.

## Обзор игры

**Концепция:** Многопользовательский шутер с видом сверху. Игроки стреляют друг в друга, собирают монеты, система лидерборда и респаун.

**Основные механики:**
- Управление персонажем (движение + поворот)
- Стрельба с перезарядкой
- Система здоровья и смерти
- Респаун после смерти
- Система команд (BLUE vs RED)
- Таблица лидеров
- Система монет (спавн + сбор)
- Игровой таймер и Game Over

## Ключевая особенность: Трехуровневая иерархия контекстов

### Иерархия

```
AppContext (Application Level)
    ↓ загружает/управляет
GameContext (Game Session Level)
    ↓ содержит множество
PlayerContext (Player Instance Level)
    ↓ управляет
Actor (Character/Bullet Level)
```

### Зачем нужна иерархия?

**Разделение ответственности по уровням абстракции:**

- **AppContext**: Глобальное состояние (уровни, загрузка, выход из приложения)
- **GameContext**: Состояние игровой сессии (игроки, таймер, лидерборд)
- **PlayerContext**: Состояние конкретного игрока (персонаж, камера, ввод)
- **Actor**: Состояние игрового объекта (позиция, здоровье, оружие)

## AppContext - Уровень приложения

### Ответственность

- Управление уровнями (текущий, максимальный)
- Загрузка игровых сцен
- Выход из приложения
- Загрузка меню

### Структура

**Интерфейс:**
```csharp
public interface IAppContext : IEntity { }
```

**Реализация:**
```csharp
public sealed class AppContext : SceneEntitySingleton<AppContext>, IAppContext
{
    // SceneEntitySingleton обеспечивает паттерн Singleton
}
```

### API (из YAML)

```yaml
# AppContextAPI.yaml
values:
  - ExitKeyCode: IValue<KeyCode>         # Клавиша выхода
  - StartLevel: IValue<int>              # Начальный уровень
  - MaxLevel: IValue<int>                # Максимальный уровень
  - CurrentLevel: IReactiveVariable<int> # Текущий уровень (реактивный)
  - GameLoadingAction: ILoadingTask      # Задача загрузки игры
```

### Installer

```csharp
public sealed class AppContextInstaller : SceneEntityInstaller<IAppContext>
{
    [SerializeField] private QuitInstaller _exitAppInstaller;
    [SerializeField] private LevelsInstaller _levelsInstaller;
    [SerializeField] private LoadGameInstaller _loadGameInstaller;

    public override void Install(IAppContext context)
    {
        // Модульная установка под-систем
        _exitAppInstaller.Install(context);
        _levelsInstaller.Install(context);
        _loadGameInstaller.Install(context);

        // Добавление контроллера меню
        context.AddBehaviour<MenuLoadController>();
    }
}
```

**Паттерн:** Композиция Installers для модульности

### Пример под-системы: QuitInstaller

```csharp
public sealed class QuitInstaller : IEntityInstaller<IAppContext>
{
    [SerializeField] private KeyCode _exitKey = KeyCode.Escape;

    public void Install(IAppContext context)
    {
        // Добавление данных
        context.AddExitKeyCode(new Const<KeyCode>(_exitKey));

        // Добавление поведения
        context.AddBehaviour<QuitController>();
    }
}

public sealed class QuitController : IEntityInit<IAppContext>, IEntityTick
{
    private IValue<KeyCode> _exitKey;

    public void Init(IAppContext context)
    {
        _exitKey = context.GetExitKeyCode();
    }

    public void Tick(IEntity entity, float deltaTime)
    {
        if (Input.GetKeyDown(_exitKey.Value))
            Application.Quit();
    }
}
```

**Принцип:** Каждая под-система инкапсулирована в отдельном Installer

## GameContext - Уровень игровой сессии

### Ответственность

- Игровой цикл (таймер игры)
- Управление игроками
- Таблица лидеров
- Система спауна
- Пул пуль
- События игры (kills, game over)

### API

```yaml
# GameContextAPI.yaml
values:
  - Players: IDictionary<TeamType, IPlayerContext>    # Словарь игроков
  - GameTime: IReactiveVariable<float>                # Время игры
  - TeamCatalog: TeamCatalog                          # Каталог команд
  - BulletPool: IEntityPool<IActor>                   # Пул пуль
  - WorldTransform: Transform                         # Корневой трансформ мира
  - Leaderboard: IReactiveDictionary<TeamType, int>   # Таблица лидеров
  - KillEvent: IEvent<KillArgs>                       # Событие убийства
  - RespawnDelay: IValue<float>                       # Задержка респауна
  - GameOverEvent: IEvent                             # Событие завершения игры
  - AllSpawnPoints: Transform[]                       # Все точки спауна
  - FreeSpawnPoints: List<Transform>                  # Свободные точки спауна
```

### Installer

```csharp
public sealed class GameContextInstaller : SceneEntityInstaller<IGameContext>
{
    [SerializeField] private SpawnPointsInstaller _spawnPointsInstaller;
    [SerializeField] private GameCycleInstaller _gameCycleInstaller;
    [SerializeField] private LeaderboardInstaller _leaderboardInstaller;
    [SerializeField] private TeamCatalog _teamCatalog;
    [SerializeField] private Const<float> _respawnTime = 3.0f;
    [SerializeField] private ActorPool _bulletPool;

    public override void Install(IGameContext context)
    {
        // Инициализация базовых данных
        context.AddPlayers(new Dictionary<TeamType, IPlayerContext>());
        context.AddWorldTransform(GameObject.Find("[World]").transform);
        context.AddTeamCatalog(_teamCatalog);
        context.AddKillEvent(new Event<KillArgs>());
        context.AddRespawnDelay(_respawnTime);
        context.AddBulletPool(_bulletPool);
        context.AddGameOverEvent(new Event());

        // Установка под-систем
        context.Install(_spawnPointsInstaller);
        context.Install(_gameCycleInstaller);
        context.Install(_leaderboardInstaller);
    }
}
```

### Пример под-системы: GameCycleInstaller

```csharp
public sealed class GameCycleInstaller : IEntityInstaller<IGameContext>
{
    [SerializeField] private float _gameDuration = 20;

    public void Install(IGameContext context)
    {
        context.AddGameTime(new ReactiveFloat(_gameDuration));
        context.AddBehaviour<GameCycleController>();
    }
}

public sealed class GameCycleController : IEntityInit<IGameContext>, IEntityEnable, IEntityTick
{
    private IVariable<float> _gameTime;
    private IEvent _gameOverEvent;

    public void Init(IGameContext context)
    {
        _gameTime = context.GetGameTime();
        _gameOverEvent = context.GetGameOverEvent();
    }

    public void Enable(IEntity entity)
    {
        Debug.Log("<color=yellow>Game Started!</color>");
    }

    public void Tick(IEntity entity, float deltaTime)
    {
        if (_gameTime.Value <= 0)
            return;

        _gameTime.Value -= deltaTime;

        if (_gameTime.Value <= 0)
        {
            _gameOverEvent.Invoke();
            Debug.Log("<color=yellow>Game Finished</color>");
        }
    }
}
```

## PlayerContext - Уровень игрока

### Ответственность

- Персонаж игрока
- Ввод игрока
- Камера игрока
- Управление персонажем

### API

```yaml
# PlayerContextAPI.yaml
values:
  - Character: IActor              # Персонаж игрока
  - TeamType: IValue<TeamType>     # Команда игрока
  - InputMap: InputMap             # Схема управления
  - Camera: Camera                 # Камера игрока
```

### Installer с интеграцией в GameContext

```csharp
public sealed class PlayerContextInstaller : SceneEntityInstaller<IPlayerContext>
{
    [SerializeField] private Const<TeamType> _teamType;
    [SerializeField] private InputMap _inputMap;
    [SerializeField] private CharacterSystemInstaller _characterInstaller;
    [SerializeField] private CameraInstaller _cameraInstaller;

    public override void Install(IPlayerContext context)
    {
        // 1. Интеграция с GameContext
        this.InstallGameContext(context);

        // 2. Установка данных игрока
        context.AddTeamType(_teamType);
        context.AddInputMap(_inputMap);

        // 3. Установка под-систем
        context.Install(_characterInstaller);
        context.Install(_cameraInstaller);
    }

    private void InstallGameContext(IPlayerContext context)
    {
        if (!GameContext.TryGetInstance(out GameContext gameContext))
            return;

        // Регистрация в leaderboard
        if (gameContext.TryGetLeaderboard(out IReactiveDictionary<TeamType, int> leaderboard))
            leaderboard.TryAdd(_teamType, 0);

        // Регистрация в списке игроков
        if (gameContext.TryGetPlayers(out IDictionary<TeamType, IPlayerContext> players))
            players.TryAdd(_teamType, context);

        // Связывание жизненного цикла
        gameContext.WhenDisable(context.Disable);
    }
}
```

**Ключевой паттерн:** PlayerContext автоматически регистрируется в GameContext

### CharacterSystemInstaller - Спавн персонажа

```csharp
public sealed class CharacterSystemInstaller : IEntityInstaller<IPlayerContext>
{
    [SerializeField] private Actor _characterPrefab;

    public void Install(IPlayerContext context)
    {
        if (AtomicUtils.IsPlayMode())
        {
            GameContext gameContext = GameContext.Instance;
            // Спаун персонажа
            Actor character = CharacterUseCase.Spawn(context, gameContext, _characterPrefab);
            context.AddCharacter(character);

            // Связывание жизненного цикла
            gameContext.WhenDisable(character.Disable);
        }

        // Добавление контроллеров
        context.AddBehaviour<CharacterMoveController>();
        context.AddBehaviour<CharacterFireController>();
        context.AddBehaviour<CharacterRespawnController>();
    }
}
```

**Паттерн:** PlayMode guard для runtime-specific логики

## Actor - Уровень игрового объекта

### Богатый API

```csharp
public static class ActorAPI
{
    // Tags
    public static readonly int Damageable;

    // Transform
    public static readonly int Position;      // IVariable<Vector3>
    public static readonly int Rotation;      // IVariable<Quaternion>

    // Movement
    public static readonly int MovementSpeed;        // IValue<float>
    public static readonly int MovementDirection;    // IReactiveVariable<Vector3>
    public static readonly int MovementEvent;        // IEvent<Vector3>

    // Combat
    public static readonly int Health;               // Health
    public static readonly int TakeDamageEvent;      // IEvent<DamageArgs>
    public static readonly int DestroyAction;        // IAction
    public static readonly int TeamType;             // IReactiveVariable<TeamType>

    // Weapon
    public static readonly int Weapon;               // IWeapon
    public static readonly int FireAction;           // IAction
    public static readonly int FireEvent;            // IEvent
    public static readonly int FireCooldown;         // Cooldown

    // View
    public static readonly int Renderer;             // Renderer
    public static readonly int Animator;             // Animator
}
```

## Паттерн: Controllers (Behaviours)

Controllers связывают контексты и делегируют логику в UseCases.

### CharacterMoveController

```csharp
public sealed class CharacterMoveController : IEntityInit<IPlayerContext>, IEntityTick
{
    private IGameContext _gameContext;
    private IActor _character;
    private IPlayerContext _playerContext;

    public void Init(IPlayerContext context)
    {
        _character = context.GetCharacter();
        _playerContext = context;
        _gameContext = GameContext.Instance;
    }

    public void Tick(IEntity entity, float deltaTime)
    {
        // Делегирование логики в UseCase
        Vector3 moveDirection = MoveInputUseCase.GetMoveDirection(_playerContext, _gameContext);
        _character.GetMovementDirection().Value = moveDirection;
    }
}
```

**Принцип:** Controller получает данные из UseCase и обновляет Actor

### CharacterFireController

```csharp
public sealed class CharacterFireController : IEntityInit<IPlayerContext>, IEntityTick
{
    private IGameContext _gameContext;
    private IActor _character;
    private IPlayerContext _playerContext;

    public void Init(IPlayerContext context)
    {
        _character = context.GetCharacter();
        _playerContext = context;
        _gameContext = GameContext.Instance;
    }

    public void Tick(IEntity entity, float deltaTime)
    {
        if (FireInputUseCase.FireRequired(_playerContext, _gameContext))
            _character.GetFireAction().Invoke();
    }
}
```

### LeaderboardController - Event-driven

```csharp
public sealed class LeaderboardController : IEntityInit<IGameContext>, IEntityDispose
{
    private ISignal<KillArgs> _killEvent;
    private IGameContext _gameContext;

    public void Init(IGameContext context)
    {
        _gameContext = context;
        _killEvent = context.GetKillEvent();
        _killEvent.Subscribe(this.OnKill);  // Подписка на событие
    }

    public void Dispose(IEntity entity)
    {
        _killEvent.Unsubscribe(this.OnKill);  // Отписка
    }

    private void OnKill(KillArgs args)
    {
        // Делегирование логики в UseCase
        LeaderboardUseCase.AddScore(_gameContext, args);
    }
}
```

**Принцип:** Controller подписывается на события и делегирует обработку в UseCase

## Паттерн: UseCases - Чистая бизнес-логика

UseCases - статические классы без состояния, содержащие бизнес-логику.

### CharacterUseCase

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

    public static IEnumerator RespawnWithDelay(IPlayerContext playerContext, IGameContext gameContext)
    {
        IValue<float> respawnTime = gameContext.GetRespawnDelay();
        yield return new WaitForSeconds(respawnTime.Value);

        if (GameCycleUseCase.IsPlaying(gameContext))
            Respawn(playerContext, gameContext);
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

**Характеристики UseCases:**
- ✅ Статические методы
- ✅ Без состояния
- ✅ Принимают контексты как параметры
- ✅ Легко тестируются

### BulletUseCase

```csharp
public static class BulletUseCase
{
    public static IEntity Spawn(
        IGameContext context,
        Vector3 position,
        Quaternion rotation,
        TeamType teamType)
    {
        IActor bullet = context.GetBulletPool().Rent();
        bullet.GetPosition().Value = position;
        bullet.GetRotation().Value = rotation;
        bullet.GetTeamType().Value = teamType;
        bullet.GetLifetime().ResetTime();
        return bullet;
    }

    public static void Despawn(IGameContext context, IActor bullet) =>
        context.GetBulletPool().Return(bullet);
}
```

### MoveInputUseCase

```csharp
public static class MoveInputUseCase
{
    public static Vector3 GetMoveDirection(IPlayerContext playerContext, IGameContext gameContext)
    {
        Vector3 direction = Vector3.zero;

        if (!GameCycleUseCase.IsPlaying(gameContext))
            return direction;

        InputMap inputMap = playerContext.GetInputMap();

        if (Input.GetKey(inputMap.MoveForward))
            direction.z = 1;
        else if (Input.GetKey(inputMap.MoveBack))
            direction.z = -1;

        if (Input.GetKey(inputMap.MoveLeft))
            direction.x = -1;
        else if (Input.GetKey(inputMap.MoveRight))
            direction.x = 1;

        return direction;
    }
}
```

### LeaderboardUseCase

```csharp
public static class LeaderboardUseCase
{
    public static bool AddScore(IGameContext gameContext, KillArgs args)
    {
        IActor instigator = args.instigator;
        IActor victim = args.victim;

        // Валидация
        if (instigator == null || victim == null || instigator.Equals(victim))
            return false;

        TeamType instigatorTeam = instigator.GetTeamType().Value;
        TeamType victimTeam = victim.GetTeamType().Value;

        if (instigatorTeam == victimTeam)
            return false; // Friendly fire не считается

        AddScore(gameContext, instigatorTeam);
        return true;
    }

    public static void AddScore(IGameContext gameContext, TeamType teamType, int score = 1)
    {
        IReactiveDictionary<TeamType,int> leaderboard = gameContext.GetLeaderboard();
        leaderboard[teamType] += score;
    }
}
```

## Генерация API из YAML

### Процесс

1. **Создание YAML файла:**
```yaml
# GameContextAPI.yaml
header: EntityAPI
entityType: IGameContext
aggressiveInlining: true
namespace: ShooterGame.Gameplay
className: GameContextAPI
directory: Assets/Examples/Shooter/Scripts/Gameplay/GameContext/

imports:
  - UnityEngine
  - Atomic.Elements
  - System.Collections.Generic

values:
  - Players: IDictionary<TeamType, IPlayerContext>
  - GameTime: IReactiveVariable<float>
```

2. **Автоматическая генерация extension methods:**
```csharp
public static class GameContextAPI
{
    public static readonly int Players;
    public static readonly int GameTime;

    // GetPlayers
    [MethodImpl(MethodImplOptions.AggressiveInlining)]
    public static IDictionary<TeamType, IPlayerContext> GetPlayers(this IGameContext entity)
        => entity.GetValue<IDictionary<TeamType, IPlayerContext>>(Players);

    // TryGetPlayers
    public static bool TryGetPlayers(this IGameContext entity, out IDictionary<TeamType, IPlayerContext> value)
        => entity.TryGetValue(Players, out value);

    // AddPlayers
    public static void AddPlayers(this IGameContext entity, IDictionary<TeamType, IPlayerContext> value)
        => entity.AddValue(Players, value);

    // HasPlayers
    public static bool HasPlayers(this IGameContext entity)
        => entity.HasValue(Players);

    // DelPlayers
    public static bool DelPlayers(this IGameContext entity)
        => entity.DelValue(Players);

    // SetPlayers
    public static void SetPlayers(this IGameContext entity, IDictionary<TeamType, IPlayerContext> value)
        => entity.SetValue(Players, value);
}
```

### Преимущества подхода

1. **Type-safe API**: Компилятор проверяет типы
2. **IntelliSense**: Автодополнение в IDE
3. **Performance**: int ID вместо strings, AggressiveInlining
4. **Единый источник правды**: YAML файл
5. **Минимум boilerplate**: Генератор создает всё

## Система Health

### Класс Health

```csharp
[Serializable]
public sealed class Health
{
    // События
    public event Action OnStateChanged;
    public event Action<int> OnHealthChanged;
    public event Action<int> OnMaxHealthChanged;
    public event Action OnHealthEmpty;

    [SerializeField, Min(0)] private int current;
    [SerializeField, Min(0)] private int max;

    public Health(int max) : this(max, max) { }

    public Health(int health, int max)
    {
        this.max = max;
        this.current = Math.Clamp(health, 0, this.max);
    }

    public int GetCurrent() => this.current;
    public int GetMax() => this.max;
    public bool IsEmpty() => this.current == 0;
    public bool IsFull() => this.current == this.max;
    public bool Exists() => this.current > 0;
    public float GetPercent() => (float) this.current / this.max;

    public bool Reduce(int range)
    {
        if (range < 0)
            throw new Exception($"Range can't be less than zero!");

        if (this.current == 0)
            return false;

        this.current = Math.Max(0, this.current - range);
        this.OnStateChanged?.Invoke();
        this.OnHealthChanged?.Invoke(this.current);

        if (this.current == 0)
            this.OnHealthEmpty?.Invoke();

        return true;
    }

    public void AssignMax() => this.SetCurrent(this.max);
}
```

### Использование

```csharp
// В респауне
character.GetHealth().AssignMax();

// Подписка на смерть
public void Init(IPlayerContext context)
{
    _characterHealth = context.GetCharacter().GetHealth();
    _characterHealth.OnHealthEmpty += this.OnHealthEmpty;
}

private void OnHealthEmpty()
{
    _gameContext.StartCoroutine(CharacterUseCase.RespawnWithDelay(_playerContext, _gameContext));
}
```

## Система команд (Teams)

### TeamCatalog - ScriptableObject

```csharp
[CreateAssetMenu(fileName = "TeamConfig", menuName = "ShooterGame/New TeamConfig")]
public sealed class TeamCatalog : ScriptableObject
{
    [SerializeField] private TeamInfo[] _teams;

    public TeamInfo GetInfo(TeamType teamType)
    {
        for (int i = 0, count = _teams.Length; i < count; i++)
        {
            TeamInfo info = _teams[i];
            if (info.Type == teamType)
                return info;
        }
        throw new KeyNotFoundException($"Team of type {teamType} is not found!");
    }

    [Serializable]
    public sealed class TeamInfo
    {
        [SerializeField] private TeamType type;
        [SerializeField] private Material material;

        public Material Material => this.material;
        public TeamType Type => type;
        public int CameraDisplay => (int) this.type - 1;

        public int PhysicsLayer => type switch
        {
            TeamType.NEUTRAL => 0,
            TeamType.BLUE => 6,
            TeamType.RED => 7,
            _ => throw new ArgumentOutOfRangeException()
        };
    }
}
```

### TeamColorBehaviour - Реактивное изменение цвета

```csharp
public sealed class TeamColorBehaviour : IEntityInit<IActor>, IEntityDispose
{
    private TeamCatalog _catalog;
    private Renderer _renderer;
    private IReactiveValue<TeamType> _team;

    public void Init(IActor entity)
    {
        if (GameContext.TryGetInstance(out GameContext gameContext))
            gameContext.TryGetTeamCatalog(out _catalog);

        _renderer = entity.GetRenderer();
        _team = entity.GetTeamType();

        // Подписка на изменение команды
        _team.Observe(this.OnTeamChanged);
    }

    public void Dispose(IEntity entity)
    {
        _team.Unsubscribe(this.OnTeamChanged);
    }

    private void OnTeamChanged(TeamType teamType)
    {
        if (_catalog)
            _renderer.material = _catalog.GetInfo(teamType).Material;
    }
}
```

## Object Pooling

### Создание пула

```csharp
[SerializeField] private ActorPool _bulletPool;

public override void Install(IGameContext context)
{
    context.AddBulletPool(_bulletPool);
}
```

### Использование

```csharp
// Rent - взять из пула или создать новый
IActor bullet = context.GetBulletPool().Rent();
bullet.GetPosition().Value = position;
bullet.GetRotation().Value = rotation;
bullet.GetTeamType().Value = teamType;

// Return - вернуть в пул для переиспользования
context.GetBulletPool().Return(bullet);
```

**Преимущества:**
- ✅ Избегание создания/удаления GameObject
- ✅ Переиспользование Entity
- ✅ Лучшая производительность

## Связывание жизненных циклов

### WhenDisable - автоматическая очистка

```csharp
// Когда gameContext отключается, character тоже отключается
gameContext.WhenDisable(character.Disable);

// Когда gameContext отключается, playerContext тоже отключается
gameContext.WhenDisable(playerContext.Disable);
```

**Зачем нужно:**
- Автоматическая очистка при завершении игры
- Предотвращение утечек памяти
- Корректное управление состоянием

## Behaviours с состоянием через конструктор

```csharp
public sealed class CameraFollowController : IEntityInit<IPlayerContext>, IEntityLateTick
{
    private readonly Vector3 _offset;  // Immutable state

    public CameraFollowController(Vector3 offset)
    {
        _offset = offset;
    }

    public void Init(IPlayerContext context)
    {
        _character = context.GetCharacter();
        _camera = context.GetCamera().transform;
    }

    public void LateTick(IEntity entity, float deltaTime)
    {
        _camera.position = _character.GetPosition().Value + _offset;
    }
}

// Использование в инсталлере
public void Install(IPlayerContext context)
{
    context.AddBehaviour(new CameraFollowController(_cameraOffset));
}
```

## Архитектурная диаграмма

```
┌─────────────────────────────────────────────────────────────┐
│                         AppContext                          │
│  - Levels Management                                        │
│  - Scene Loading                                            │
│  - Quit Application                                         │
└──────────────────────────┬──────────────────────────────────┘
                           │ owns
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                        GameContext                          │
│  - Game Cycle (Timer)                                       │
│  - Players Registry                                         │
│  - Leaderboard                                              │
│  - Spawn Points                                             │
│  - Bullet Pool                                              │
│  - Events (Kill, GameOver)                                  │
└──────────────────────────┬──────────────────────────────────┘
                           │ contains multiple
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                      PlayerContext                          │
│  - Character (Actor)                                        │
│  - Camera                                                   │
│  - Input                                                    │
│  - Team                                                     │
└──────────────────────────┬──────────────────────────────────┘
                           │ owns
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                      Actor (Entity)                         │
│  - Transform (Position, Rotation)                           │
│  - Movement                                                 │
│  - Health                                                   │
│  - Weapon                                                   │
│  - Team                                                     │
│  - View (Renderer, Animator)                                │
└─────────────────────────────────────────────────────────────┘
```

## Поток данных: Стрельба

```
1. Input.GetKeyDown(Fire)
2. FireInputUseCase.FireRequired() → true
3. CharacterFireController вызывает character.GetFireAction().Invoke()
4. ProjectileWeaponInstaller.FireAction:
   - Проверяет ammo и cooldown
   - BulletUseCase.Spawn()
   - Вызывает FireEvent
5. Bullet collision:
   - TakeDamageEvent
   - Health.Reduce()
6. Health.OnHealthEmpty:
   - TakeDeathEvent
   - KillEvent в GameContext
7. LeaderboardController получает KillEvent:
   - LeaderboardUseCase.AddScore()
   - Leaderboard[team] += 1
8. UI реагирует на Leaderboard.OnChanged
```

## Best Practices

### 1. Модульная композиция через Installers

```csharp
public sealed class GameContextInstaller : SceneEntityInstaller<IGameContext>
{
    [SerializeField] private SpawnPointsInstaller _spawnPointsInstaller;
    [SerializeField] private GameCycleInstaller _gameCycleInstaller;
    [SerializeField] private LeaderboardInstaller _leaderboardInstaller;

    public override void Install(IGameContext context)
    {
        // Базовая конфигурация
        context.AddPlayers(new Dictionary<TeamType, IPlayerContext>());

        // Композиция под-систем
        context.Install(_spawnPointsInstaller);
        context.Install(_gameCycleInstaller);
        context.Install(_leaderboardInstaller);
    }
}
```

### 2. Event-driven архитектура

```csharp
// Определение событий
context.AddKillEvent(new Event<KillArgs>());
context.AddGameOverEvent(new Event());

// Подписка на события
public void Init(IGameContext context)
{
    _killEvent = context.GetKillEvent();
    _killEvent.Subscribe(this.OnKill);
}

public void Dispose(IEntity entity)
{
    _killEvent.Unsubscribe(this.OnKill);
}

// Вызов событий
context.GetGameOverEvent().Invoke();
context.GetKillEvent().Invoke(new KillArgs { instigator = killer, victim = victim });
```

### 3. Safe access к синглтонам

```csharp
// Не бросает exception, если не найден
if (GameContext.TryGetInstance(out GameContext gameContext))
{
    // использование
}
```

### 4. Условная инициализация

```csharp
if (AtomicUtils.IsPlayMode())
{
    // Код выполняется только в Play Mode
    Actor character = CharacterUseCase.Spawn(context, gameContext, _characterPrefab);
    context.AddCharacter(character);
}
```

## Ключевые принципы

### 1. Разделение по уровням абстракции

- **App**: Глобальное состояние приложения
- **Game**: Состояние игровой сессии
- **Player**: Состояние конкретного игрока
- **Actor**: Состояние игрового объекта

### 2. Controllers делегируют в UseCases

```csharp
// Controller - реакция на события
public void Tick(IEntity entity, float deltaTime)
{
    // Делегирование логики в UseCase
    Vector3 direction = MoveInputUseCase.GetMoveDirection(_playerContext, _gameContext);
    _character.GetMovementDirection().Value = direction;
}
```

### 3. Event-driven для слабой связанности

```csharp
// Источник события не знает о подписчиках
_killEvent.Invoke(new KillArgs { instigator, victim });

// Подписчик реагирует независимо
private void OnKill(KillArgs args)
{
    LeaderboardUseCase.AddScore(_gameContext, args);
}
```

### 4. Reactive values для автоматического обновления

```csharp
// Изменение значения
leaderboard[TeamType.BLUE] += 1;

// Автоматическое оповещение UI
leaderboard.Subscribe(OnLeaderboardChanged);
```

---

# ЧАСТЬ 2: ПОШАГОВОЕ ВНЕДРЕНИЕ ФИЧ

## Обзор всех фич в Shooter Demo

Shooter Demo содержит **19 основных фич**, организованных по уровню сложности.

### Классификация по уровням

#### Уровень 1 (⭐): Foundation - Базовые данные

1. **Transform System** - Позиция и поворот
2. **Team System** - Команды игроков
3. **Health System** - Система здоровья
4. **Input System** - Схема управления

#### Уровень 2 (⭐⭐): Core Mechanics - Основная механика

5. **Movement System** - Движение персонажа
6. **Rotation System** - Поворот персонажа
7. **Physics System** - Физика и коллизии

#### Уровень 3 (⭐⭐⭐): Advanced Mechanics - Продвинутая механика

8. **Weapon System** - Система оружия
9. **Bullet System** - Система пуль (с Object Pooling)
10. **Damage System** - Нанесение урона
11. **Death & Respawn System** - Смерть и возрождение
12. **Camera System** - Камера (следование, split-screen)
13. **Spawn Points System** - Управление точками спауна
14. **View System** - Анимации и визуальные эффекты

#### Уровень 4 (⭐⭐⭐⭐): Game Architecture - Архитектура игры

15. **Game Cycle System** - Игровой цикл (таймер)
16. **Leaderboard System** - Таблица лидеров
17. **Player Context System** - Координация игрока
18. **Game Context System** - Управление игровой сессией
19. **App Context System** - Управление приложением

### Граф зависимостей фич

```
┌─────────────────────────────────────────────────────────────┐
│                    Level 1: Foundation                      │
│  [Transform] [Team] [Health] [Input]                       │
└──────────────┬──────────────────────────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────────────────────────┐
│                   Level 2: Core Mechanics                   │
│  [Movement] [Rotation] [Physics]                           │
│  depends on: Transform, Team, Health                        │
└──────────────┬──────────────────────────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────────────────────────┐
│                Level 3: Advanced Mechanics                  │
│  [Weapon] [Bullet] [Damage] [Respawn] [Camera] [View]     │
│  depends on: All Level 1+2                                  │
└──────────────┬──────────────────────────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────────────────────────┐
│                 Level 4: Game Architecture                  │
│  [GameCycle] [Leaderboard] [PlayerContext] [GameContext]   │
│  depends on: All Level 1+2+3                                │
└─────────────────────────────────────────────────────────────┘
```

---

## Feature 1: Transform System (⭐)

**Сложность:** Foundation
**Зависимости:** Нет
**Используется в:** Movement, Rotation, Camera, Spawn, все визуальные компоненты

### Описание

Transform System обеспечивает доступ к позиции и ротации объекта через reactive переменные. Использует паттерн **InlineVariable** для прямой связи с Unity Transform.

### Шаг 1: Добавление в EntityAPI (ActorAPI)

```csharp
public static class ActorAPI
{
    public static readonly int Position;  // IVariable<Vector3>
    public static readonly int Rotation;  // IVariable<Quaternion>

    static ActorAPI()
    {
        Position = NameToId(nameof(Position));
        Rotation = NameToId(nameof(Rotation));
    }

    // Extension methods для Position
    public static IVariable<Vector3> GetPosition(this IActor entity)
        => entity.GetValue<IVariable<Vector3>>(Position);

    public static void AddPosition(this IActor entity, IVariable<Vector3> value)
        => entity.AddValue(Position, value);

    // Extension methods для Rotation
    public static IVariable<Quaternion> GetRotation(this IActor entity)
        => entity.GetValue<IVariable<Quaternion>>(Rotation);

    public static void AddRotation(this IActor entity, IVariable<Quaternion> value)
        => entity.AddValue(Rotation, value);
}
```

**Принцип:** ID-based доступ через int ключи для производительности

### Шаг 2: Создание Data Classes

Transform System использует готовые классы из Atomic.Elements:

```csharp
// TransformPositionVariable - обертка вокруг transform.position
public sealed class TransformPositionVariable : IVariable<Vector3>
{
    private readonly Transform _transform;

    public TransformPositionVariable(Transform transform)
    {
        _transform = transform;
    }

    public Vector3 Value
    {
        get => _transform.position;
        set => _transform.position = value;
    }
}

// TransformRotationVariable - обертка вокруг transform.rotation
public sealed class TransformRotationVariable : IVariable<Quaternion>
{
    private readonly Transform _transform;

    public TransformRotationVariable(Transform transform)
    {
        _transform = transform;
    }

    public Quaternion Value
    {
        get => _transform.rotation;
        set => _transform.rotation = value;
    }
}
```

**Преимущество:** Прямое изменение Transform без промежуточных копий

### Шаг 3: Создание UseCases

Transform System обычно не требует UseCases, так как работа идет напрямую через переменные.

**Пример использования в других UseCases:**

```csharp
public static class SpawnPointsUseCase
{
    public static Transform NextPoint(IGameContext context)
    {
        List<Transform> freePoints = context.GetFreeSpawnPoints();
        return freePoints[Random.Range(0, freePoints.Count)];
    }
}

public static class CharacterUseCase
{
    public static void Respawn(IPlayerContext playerContext, IGameContext gameContext)
    {
        IActor character = playerContext.GetCharacter();
        Transform spawnPoint = SpawnPointsUseCase.NextPoint(gameContext);

        // Прямая установка позиции и ротации
        character.GetPosition().Value = spawnPoint.position;
        character.GetRotation().Value = spawnPoint.rotation;

        character.GetRespawnEvent().Invoke();
    }
}
```

### Шаг 4: Создание Behaviours

Transform обычно не требует Behaviours - другие системы изменяют его напрямую.

**Пример изменения из MovementBehaviour:**

```csharp
public sealed class KinematicMovementBehaviour : IEntityInit, IEntityFixedTick
{
    private IValue<float> _speed;
    private IReactiveValue<Vector3> _direction;
    private IVariable<Vector3> _position;  // Transform

    public void Init(IEntity entity)
    {
        _speed = entity.GetMovementSpeed();
        _direction = entity.GetMovementDirection();
        _position = entity.GetPosition();  // Получаем Transform
    }

    public void FixedTick(IEntity entity, float deltaTime)
    {
        Vector3 direction = _direction.Value;
        if (direction.sqrMagnitude < 0.001f)
            return;

        Vector3 velocity = direction.normalized * _speed.Value;
        // Прямое изменение позиции
        _position.Value += velocity * deltaTime;
    }
}
```

### Шаг 5: Создание Installer

```csharp
public sealed class CharacterInstaller : SceneEntityInstaller<IActor>
{
    [SerializeField] private Transform _transform;

    public override void Install(IActor entity)
    {
        // Установка Transform с прямой связью
        entity.AddPosition(new TransformPositionVariable(_transform));
        entity.AddRotation(new TransformRotationVariable(_transform));

        // ... другие компоненты
    }
}
```

**Важно:** Transform должен быть установлен ПЕРВЫМ, так как многие системы зависят от него

### Шаг 6: Интеграция с другими фичами

**Зависят от Transform:**
- ✅ Movement System - изменяет Position
- ✅ Rotation System - изменяет Rotation
- ✅ Camera System - читает Position для следования
- ✅ Spawn System - устанавливает Position/Rotation при спауне
- ✅ Bullet System - использует Position/Rotation при создании пули

**Пример интеграции с Camera:**

```csharp
public sealed class CameraFollowController : IEntityInit<IPlayerContext>, IEntityLateTick
{
    private IVariable<Vector3> _characterPosition;  // Transform персонажа
    private Transform _cameraTransform;
    private readonly Vector3 _offset;

    public void Init(IPlayerContext context)
    {
        IActor character = context.GetCharacter();
        _characterPosition = character.GetPosition();  // Получаем Transform
        _cameraTransform = context.GetCamera().transform;
    }

    public void LateTick(IEntity entity, float deltaTime)
    {
        // Камера следует за персонажем
        _cameraTransform.position = _characterPosition.Value + _offset;
    }
}
```

### Шаг 7: Best Practices и зависимости

**Зависимости:**
- 📦 Unity Transform component
- 📦 Atomic.Elements (IVariable, TransformPositionVariable, TransformRotationVariable)

**Best Practices:**

1. ✅ **Всегда устанавливайте Transform первым** в Installer
   ```csharp
   // ПРАВИЛЬНО
   entity.AddPosition(new TransformPositionVariable(_transform));
   entity.AddRotation(new TransformRotationVariable(_transform));
   // ... затем другие компоненты
   ```

2. ✅ **Используйте InlineVariable для прямой связи** с Unity компонентами
   ```csharp
   entity.AddPosition(new TransformPositionVariable(_transform));
   // Вместо копирования значений
   ```

3. ✅ **FixedTick для изменения позиции** (физика)
   ```csharp
   public void FixedTick(IEntity entity, float deltaTime)
   {
       _position.Value += velocity * deltaTime;
   }
   ```

4. ✅ **LateTick для камеры** (после всех движений)
   ```csharp
   public void LateTick(IEntity entity, float deltaTime)
   {
       _cameraTransform.position = _characterPosition.Value + _offset;
   }
   ```

5. ⚠️ **НЕ кэшируйте Vector3/Quaternion** - всегда читайте через .Value
   ```csharp
   // НЕПРАВИЛЬНО
   Vector3 pos = _position.Value;
   pos.x += 1;  // Изменяет копию!

   // ПРАВИЛЬНО
   _position.Value += Vector3.right;
   ```

**Типичные ошибки:**

❌ **Ошибка 1:** Изменение копии вместо оригинала
```csharp
// НЕПРАВИЛЬНО
Vector3 pos = entity.GetPosition().Value;
pos.x = 10;  // Изменяет локальную копию
```

✅ **Исправление:**
```csharp
// ПРАВИЛЬНО
entity.GetPosition().Value = new Vector3(10, pos.y, pos.z);
// или
IVariable<Vector3> position = entity.GetPosition();
position.Value = new Vector3(10, position.Value.y, position.Value.z);
```

---

## Feature 2: Team System (⭐)

**Сложность:** Foundation
**Зависимости:** Transform (для PhysicsLayer)
**Используется в:** Damage, Leaderboard, Weapon, View (colors), Physics

### Описание

Team System управляет командами игроков (NEUTRAL, BLUE, RED) и автоматически синхронизирует визуальные компоненты (цвет, физический слой) через reactive переменные.

### Шаг 1: Добавление в EntityAPI

```csharp
public static class ActorAPI
{
    public static readonly int TeamType;         // IReactiveVariable<TeamType>
    public static readonly int PhysicsLayer;     // IVariable<int>

    public static IReactiveVariable<TeamType> GetTeamType(this IActor entity)
        => entity.GetValue<IReactiveVariable<TeamType>>(TeamType);

    public static void AddTeamType(this IActor entity, IReactiveVariable<TeamType> value)
        => entity.AddValue(TeamType, value);

    public static IVariable<int> GetPhysicsLayer(this IActor entity)
        => entity.GetValue<IVariable<int>>(PhysicsLayer);

    public static void AddPhysicsLayer(this IActor entity, IVariable<int> value)
        => entity.AddValue(PhysicsLayer, value);
}
```

**Принцип:** IReactiveVariable для автоматического оповещения подписчиков

### Шаг 2: Создание Data Classes

```csharp
// Enum для команд
public enum TeamType
{
    NEUTRAL = 0,
    BLUE = 1,
    RED = 2
}

// ScriptableObject каталог команд
[CreateAssetMenu(fileName = "TeamConfig", menuName = "ShooterGame/New TeamConfig")]
public sealed class TeamCatalog : ScriptableObject
{
    [SerializeField] private TeamInfo[] _teams;

    public TeamInfo GetInfo(TeamType teamType)
    {
        for (int i = 0; i < _teams.Length; i++)
        {
            if (_teams[i].Type == teamType)
                return _teams[i];
        }
        throw new KeyNotFoundException($"Team {teamType} not found!");
    }

    [Serializable]
    public sealed class TeamInfo
    {
        [SerializeField] private TeamType type;
        [SerializeField] private Material material;  // Цвет команды

        public TeamType Type => type;
        public Material Material => material;
        public int CameraDisplay => (int)type - 1;  // Для split-screen

        public int PhysicsLayer => type switch
        {
            TeamType.NEUTRAL => 0,   // Default layer
            TeamType.BLUE => 6,      // Custom layer 6
            TeamType.RED => 7,       // Custom layer 7
            _ => throw new ArgumentOutOfRangeException()
        };
    }
}
```

**Принцип:** ScriptableObject для конфигурации в редакторе

### Шаг 3: Создание UseCases

```csharp
public static class TeamUseCase
{
    public static bool IsEnemy(TeamType team1, TeamType team2)
    {
        if (team1 == TeamType.NEUTRAL || team2 == TeamType.NEUTRAL)
            return false;

        return team1 != team2;
    }

    public static bool IsFriendlyFire(IActor attacker, IActor victim)
    {
        if (attacker == null || victim == null)
            return false;

        TeamType attackerTeam = attacker.GetTeamType().Value;
        TeamType victimTeam = victim.GetTeamType().Value;

        return attackerTeam == victimTeam && attackerTeam != TeamType.NEUTRAL;
    }
}
```

### Шаг 4: Создание Behaviours

```csharp
// Автоматическое изменение Physics Layer при смене команды
public sealed class TeamPhysicsLayerBehaviour : IEntityInit, IEntityDispose
{
    private IReactiveValue<TeamType> _teamType;
    private IVariable<int> _physicsLayer;
    private TeamCatalog _catalog;

    public void Init(IEntity entity)
    {
        if (GameContext.TryGetInstance(out GameContext context))
            context.TryGetTeamCatalog(out _catalog);

        _teamType = entity.GetTeamType();
        _physicsLayer = entity.GetPhysicsLayer();

        // Подписка на изменение команды
        _teamType.Observe(this.OnTeamChanged);
    }

    public void Dispose(IEntity entity)
    {
        _teamType.Unsubscribe(this.OnTeamChanged);
    }

    private void OnTeamChanged(TeamType newTeam)
    {
        if (_catalog != null)
            _physicsLayer.Value = _catalog.GetInfo(newTeam).PhysicsLayer;
    }
}

// Автоматическое изменение цвета при смене команды
public sealed class TeamColorBehaviour : IEntityInit, IEntityDispose
{
    private IReactiveValue<TeamType> _teamType;
    private Renderer _renderer;
    private TeamCatalog _catalog;

    public void Init(IEntity entity)
    {
        if (GameContext.TryGetInstance(out GameContext context))
            context.TryGetTeamCatalog(out _catalog);

        _teamType = entity.GetTeamType();
        _renderer = entity.GetRenderer();

        // Подписка на изменение команды
        _teamType.Observe(this.OnTeamChanged);
    }

    public void Dispose(IEntity entity)
    {
        _teamType.Unsubscribe(this.OnTeamChanged);
    }

    private void OnTeamChanged(TeamType newTeam)
    {
        if (_catalog != null && _renderer != null)
            _renderer.material = _catalog.GetInfo(newTeam).Material;
    }
}
```

**Принцип:** Reactive Behaviours автоматически синхронизируют состояние

### Шаг 5: Создание Installer

```csharp
public sealed class CharacterInstaller : SceneEntityInstaller<IActor>
{
    [SerializeField] private GameObject _gameObject;
    [SerializeField] private ReactiveVariable<TeamType> _teamType;

    public override void Install(IActor entity)
    {
        // Team
        entity.AddTeamType(_teamType);
        entity.AddBehaviour<TeamPhysicsLayerBehaviour>();

        // Physics Layer (связан с GameObject.layer)
        entity.AddPhysicsLayer(new InlineVariable<int>(
            getter: () => _gameObject.layer,
            setter: value => _gameObject.layer = value
        ));

        // ... другие компоненты
    }
}
```

**Важно:** InlineVariable связывает entity.PhysicsLayer с GameObject.layer

### Шаг 6: Интеграция с другими фичами

**Используется в:**

**Damage System:**
```csharp
public sealed class BulletCollisionBehaviour : IEntityInit, IEntityEnable, IEntityDisable
{
    private void OnTriggerEntered(Collider collider)
    {
        if (!collider.TryGetComponent(out IActor target))
            return;

        // Проверка на friendly fire
        TeamType bulletTeam = _bullet.GetTeamType().Value;
        TeamType targetTeam = target.GetTeamType().Value;

        if (bulletTeam == targetTeam)
            return;  // Не наносим урон своим

        // Нанести урон
        target.GetTakeDamageEvent().Invoke(new DamageArgs { instigator = _bullet, victim = target });
    }
}
```

**Leaderboard System:**
```csharp
public static class LeaderboardUseCase
{
    public static bool AddScore(IGameContext context, KillArgs args)
    {
        TeamType killerTeam = args.instigator.GetTeamType().Value;
        TeamType victimTeam = args.victim.GetTeamType().Value;

        if (killerTeam == victimTeam)
            return false;  // Friendly fire не засчитывается

        IReactiveDictionary<TeamType, int> leaderboard = context.GetLeaderboard();
        leaderboard[killerTeam] += 1;
        return true;
    }
}
```

**Character Spawn:**
```csharp
public static class CharacterUseCase
{
    public static Actor Spawn(IPlayerContext playerContext, IGameContext gameContext, Actor prefab)
    {
        Actor character = SceneEntity.Create(prefab, spawnPoint, worldTransform);

        // Установка команды от PlayerContext
        TeamType playerTeam = playerContext.GetTeamType().Value;
        character.GetTeamType().Value = playerTeam;

        return character;
    }
}
```

### Шаг 7: Best Practices и зависимости

**Зависимости:**
- 📦 Atomic.Elements (IReactiveVariable, ReactiveVariable)
- 📦 TeamCatalog (ScriptableObject)
- 📦 Physics Layers должны быть настроены в Unity (Layers 6, 7)

**Best Practices:**

1. ✅ **Используйте ReactiveVariable для TeamType**
   ```csharp
   entity.AddTeamType(new ReactiveVariable<TeamType>(TeamType.BLUE));
   ```

2. ✅ **Автоматическая синхронизация через Behaviours**
   ```csharp
   entity.AddBehaviour<TeamPhysicsLayerBehaviour>();  // Синхронизирует layer
   entity.AddBehaviour<TeamColorBehaviour>();         // Синхронизирует цвет
   ```

3. ✅ **Проверка friendly fire перед нанесением урона**
   ```csharp
   if (attackerTeam == victimTeam && attackerTeam != TeamType.NEUTRAL)
       return;  // Не наносим урон
   ```

4. ✅ **Physics Layers для разделения команд**
   ```csharp
   // В Unity Project Settings → Physics
   // Настроить Layer Collision Matrix:
   // - Layer 6 (BLUE) не коллайдит с Layer 6
   // - Layer 7 (RED) не коллайдит с Layer 7
   // - Layer 6 коллайдит с Layer 7
   ```

5. ✅ **Централизованная конфигурация в ScriptableObject**
   ```csharp
   // Вместо hardcode в коде
   [SerializeField] private TeamCatalog _teamCatalog;
   Material material = _teamCatalog.GetInfo(teamType).Material;
   ```

**Типичные ошибки:**

❌ **Ошибка 1:** Забыли проверить friendly fire
```csharp
// НЕПРАВИЛЬНО
target.GetTakeDamageEvent().Invoke(damageArgs);  // Стреляет по своим!
```

✅ **Исправление:**
```csharp
// ПРАВИЛЬНО
if (bulletTeam != targetTeam)
    target.GetTakeDamageEvent().Invoke(damageArgs);
```

❌ **Ошибка 2:** Не подписались на TeamType для синхронизации
```csharp
// НЕПРАВИЛЬНО - цвет не обновится при смене команды
_renderer.material = _catalog.GetInfo(_teamType.Value).Material;
```

✅ **Исправление:**
```csharp
// ПРАВИЛЬНО - автоматическое обновление
_teamType.Observe(newTeam => {
    _renderer.material = _catalog.GetInfo(newTeam).Material;
});
```

---

## Feature 3: Health System (⭐)

**Сложность:** Foundation
**Зависимости:** Нет
**Используется в:** Damage System, Death System, Respawn System, UI

### Описание

Health System управляет здоровьем entity через класс Health с событиями. Использует паттерн Observer для реактивного обновления UI и логики.

### Шаг 1: Добавление в EntityAPI

```csharp
public static class ActorAPI
{
    public static readonly int Health;              // Health
    public static readonly int TakeDamageEvent;     // IEvent<DamageArgs>
    public static readonly int TakeDeathEvent;      // IEvent<DamageArgs>
    public static readonly int Damageable;          // Tag

    public static Health GetHealth(this IActor entity)
        => entity.GetValue<Health>(Health);

    public static void AddHealth(this IActor entity, Health value)
        => entity.AddValue(Health, value);

    public static IEvent<DamageArgs> GetTakeDamageEvent(this IActor entity)
        => entity.GetValue<IEvent<DamageArgs>>(TakeDamageEvent);

    public static void AddTakeDamageEvent(this IActor entity, IEvent<DamageArgs> value)
        => entity.AddValue(TakeDamageEvent, value);

    public static IEvent<DamageArgs> GetTakeDeathEvent(this IActor entity)
        => entity.GetValue<IEvent<DamageArgs>>(TakeDeathEvent);

    public static void AddTakeDeathEvent(this IActor entity, IEvent<DamageArgs> value)
        => entity.AddValue(TakeDeathEvent, value);

    public static bool HasDamageableTag(this IActor entity)
        => entity.HasTag(Damageable);

    public static bool AddDamageableTag(this IActor entity)
        => entity.AddTag(Damageable);
}
```

### Шаг 2: Создание Data Classes

```csharp
// Класс Health с событиями
[Serializable]
public sealed class Health
{
    // События
    public event Action OnStateChanged;
    public event Action<int> OnHealthChanged;
    public event Action<int> OnMaxHealthChanged;
    public event Action OnHealthEmpty;

    [SerializeField, Min(0)] private int current;
    [SerializeField, Min(0)] private int max;

    public Health(int max) : this(max, max) { }

    public Health(int current, int max)
    {
        this.max = max;
        this.current = Mathf.Clamp(current, 0, this.max);
    }

    // Getters
    public int GetCurrent() => this.current;
    public int GetMax() => this.max;
    public bool IsEmpty() => this.current == 0;
    public bool IsFull() => this.current == this.max;
    public bool Exists() => this.current > 0;
    public float GetPercent() => (float)this.current / this.max;

    // Модификация
    public bool Reduce(int amount)
    {
        if (amount < 0)
            throw new Exception("Amount can't be negative!");

        if (this.current == 0)
            return false;

        int oldValue = this.current;
        this.current = Mathf.Max(0, this.current - amount);

        // Вызов событий
        this.OnStateChanged?.Invoke();
        this.OnHealthChanged?.Invoke(this.current);

        if (this.current == 0)
            this.OnHealthEmpty?.Invoke();

        return true;
    }

    public void Add(int amount)
    {
        if (amount < 0)
            throw new Exception("Amount can't be negative!");

        this.current = Mathf.Min(this.max, this.current + amount);

        this.OnStateChanged?.Invoke();
        this.OnHealthChanged?.Invoke(this.current);
    }

    public void SetCurrent(int value)
    {
        this.current = Mathf.Clamp(value, 0, this.max);
        this.OnStateChanged?.Invoke();
        this.OnHealthChanged?.Invoke(this.current);
    }

    public void AssignMax()
    {
        this.SetCurrent(this.max);
    }
}

// Аргументы урона
public struct DamageArgs
{
    public IActor instigator;  // Кто нанес урон
    public IActor victim;      // Кто получил урон
    public int damage;         // Количество урона
}
```

**Принцип:** Инкапсуляция логики здоровья в отдельном классе с событиями

### Шаг 3: Создание UseCases

Health обычно не требует отдельных UseCases, так как логика инкапсулирована в классе Health.

**Пример использования:**

```csharp
// В Damage System
public static class DamageUseCase
{
    public static bool TakeDamage(IActor victim, IActor attacker, int damage)
    {
        if (!victim.HasDamageableTag())
            return false;

        Health health = victim.GetHealth();
        if (!health.Exists())
            return false;

        // Уменьшаем здоровье
        health.Reduce(damage);

        // Вызываем событие урона
        victim.GetTakeDamageEvent().Invoke(new DamageArgs
        {
            instigator = attacker,
            victim = victim,
            damage = damage
        });

        // Если здоровье закончилось - событие смерти
        if (health.IsEmpty())
        {
            victim.GetTakeDeathEvent().Invoke(new DamageArgs
            {
                instigator = attacker,
                victim = victim,
                damage = damage
            });
        }

        return true;
    }
}
```

### Шаг 4: Создание Behaviours

Health обычно не требует Behaviours - другие системы читают его напрямую.

**Пример UI Behaviour:**

```csharp
// UI элемент, отображающий здоровье
public sealed class HealthBar : MonoBehaviour
{
    [SerializeField] private Actor _target;
    [SerializeField] private Image _fillImage;

    private Health _health;

    private void Start()
    {
        _health = _target.GetHealth();

        // Подписка на изменение здоровья
        _health.OnHealthChanged += this.UpdateBar;

        // Начальное значение
        this.UpdateBar(_health.GetCurrent());
    }

    private void OnDestroy()
    {
        if (_health != null)
            _health.OnHealthChanged -= this.UpdateBar;
    }

    private void UpdateBar(int currentHealth)
    {
        _fillImage.fillAmount = _health.GetPercent();
    }
}
```

**Пример Controller для Respawn:**

```csharp
public sealed class CharacterRespawnController : IEntityInit<IPlayerContext>, IEntityDispose
{
    private IGameContext _gameContext;
    private IPlayerContext _playerContext;
    private Health _characterHealth;

    public void Init(IPlayerContext context)
    {
        _playerContext = context;
        _gameContext = GameContext.Instance;

        IActor character = context.GetCharacter();
        _characterHealth = character.GetHealth();

        // Подписка на смерть (здоровье = 0)
        _characterHealth.OnHealthEmpty += this.OnHealthEmpty;
    }

    public void Dispose(IEntity entity)
    {
        if (_characterHealth != null)
            _characterHealth.OnHealthEmpty -= this.OnHealthEmpty;
    }

    private void OnHealthEmpty()
    {
        // Запуск респауна с задержкой
        _gameContext.StartCoroutine(
            CharacterUseCase.RespawnWithDelay(_playerContext, _gameContext)
        );
    }
}
```

### Шаг 5: Создание Installer

```csharp
public sealed class CharacterInstaller : SceneEntityInstaller<IActor>
{
    [SerializeField] private Health _health = new Health(3);  // 3 HP

    public override void Install(IActor entity)
    {
        // Life System
        entity.AddDamageableTag();                          // Может получать урон
        entity.AddHealth(_health);                          // Здоровье
        entity.AddTakeDamageEvent(new Event<DamageArgs>()); // Событие урона
        entity.AddTakeDeathEvent(new Event<DamageArgs>());  // Событие смерти

        // ... другие компоненты
    }
}
```

### Шаг 6: Интеграция с другими фичами

**Damage System** - наносит урон через Health.Reduce():
```csharp
private void OnTriggerEntered(Collider collider)
{
    if (!collider.TryGetComponent(out IActor target))
        return;

    if (!target.HasDamageableTag())
        return;

    Health health = target.GetHealth();
    int damage = _bullet.GetDamage().Value;

    health.Reduce(damage);  // Уменьшаем здоровье
}
```

**Respawn System** - восстанавливает здоровье:
```csharp
public static void Respawn(IPlayerContext playerContext, IGameContext gameContext)
{
    IActor character = playerContext.GetCharacter();

    // Восстановление здоровья
    character.GetHealth().AssignMax();

    // Телепорт на спаун
    Transform spawnPoint = SpawnPointsUseCase.NextPoint(gameContext);
    character.GetPosition().Value = spawnPoint.position;

    character.GetRespawnEvent().Invoke();
}
```

**Movement/Rotation** - проверка условия "жив":
```csharp
// В CharacterInstaller
entity.AddMovementCondition(new AndExpression(_health.Exists));
entity.AddRotationCondition(new AndExpression(_health.Exists));

// В KinematicMovementBehaviour
public void FixedTick(IEntity entity, float deltaTime)
{
    // Проверка условия движения (включая _health.Exists)
    if (!_condition.Invoke())
        return;

    // Движение
    _position.Value += velocity * deltaTime;
}
```

### Шаг 7: Best Practices и зависимости

**Зависимости:**
- 📦 Atomic.Elements (IEvent, Event)
- 📦 UnityEngine (Mathf, Serializable)

**Best Practices:**

1. ✅ **Используйте события для реакции на изменения**
   ```csharp
   _health.OnHealthEmpty += this.OnDeath;
   _health.OnHealthChanged += this.UpdateUI;
   ```

2. ✅ **ВСЕГДА отписывайтесь от событий**
   ```csharp
   public void Dispose(IEntity entity)
   {
       if (_health != null)
           _health.OnHealthEmpty -= this.OnDeath;
   }
   ```

3. ✅ **Используйте тег Damageable для фильтрации**
   ```csharp
   if (!target.HasDamageableTag())
       return;  // Этот объект не может получать урон
   ```

4. ✅ **Проверяйте Exists() перед нанесением урона**
   ```csharp
   if (!health.Exists())
       return;  // Уже мертв
   ```

5. ✅ **Используйте Health.Exists в Conditions**
   ```csharp
   entity.AddFireCondition(new AndExpression(_health.Exists));
   // Не может стрелять, если мертв
   ```

**Типичные ошибки:**

❌ **Ошибка 1:** Забыли отписаться от событий
```csharp
// НЕПРАВИЛЬНО - утечка памяти
public void Init(IEntity entity)
{
    _health.OnHealthEmpty += this.OnDeath;
}
// Нет Dispose!
```

✅ **Исправление:**
```csharp
public void Dispose(IEntity entity)
{
    _health.OnHealthEmpty -= this.OnDeath;
}
```

❌ **Ошибка 2:** Изменение здоровья напрямую через поле
```csharp
// НЕПРАВИЛЬНО - события не вызовутся
_health.current = 0;
```

✅ **Исправление:**
```csharp
// ПРАВИЛЬНО - события вызовутся
_health.Reduce(_health.GetCurrent());
// или
_health.SetCurrent(0);
```

---

## Заключение

Shooter Demo демонстрирует:

✅ **Масштабируемую архитектуру**: Иерархия контекстов для сложных проектов
✅ **Модульность**: Композиция через Installers
✅ **Event-driven**: Слабая связанность через события
✅ **UseCases**: Чистая, тестируемая бизнес-логика
✅ **Reactive**: Автоматическое распространение изменений
✅ **Object pooling**: Эффективное управление памятью
✅ **YAML-генерация API**: Type-safe доступ к данным

### Следующие шаги

После изучения Shooter Demo рекомендуется:
1. **RTS Demo** - продвинутые паттерны (Factories, Builders, Filters, AI, Burst compilation)
2. **Собственный проект** - применение изученных принципов

### Ключевая формула

```
Installers (конфигурация) +
Behaviours/Controllers (когда) +
UseCases (как) +
Events (слабая связанность) +
Reactive (автообновление) =
Масштабируемая архитектура
```

## Feature 4: Movement System (⭐⭐)

**Сложность:** Core Mechanics
**Зависимости:** Transform, Health (для условий)
**Используется в:** Character control, AI movement

### Описание

Movement System обеспечивает движение персонажа через reactive направление и проверку условий. Использует паттерн **Producer-Consumer** (Input пишет направление, Movement читает).

### Шаг 1: Добавление в EntityAPI

```csharp
public static class ActorAPI
{
    public static readonly int MovementSpeed;       // IValue<float>
    public static readonly int MovementDirection;   // IReactiveVariable<Vector3>
    public static readonly int MovementCondition;   // IExpression<bool>
    public static readonly int MovementEvent;       // IEvent<Vector3>

    public static IValue<float> GetMovementSpeed(this IActor entity)
        => entity.GetValue<IValue<float>>(MovementSpeed);

    public static void AddMovementSpeed(this IActor entity, IValue<float> value)
        => entity.AddValue(MovementSpeed, value);

    public static IReactiveVariable<Vector3> GetMovementDirection(this IActor entity)
        => entity.GetValue<IReactiveVariable<Vector3>>(MovementDirection);

    public static void AddMovementDirection(this IActor entity, IReactiveVariable<Vector3> value)
        => entity.AddValue(MovementDirection, value);

    public static IExpression<bool> GetMovementCondition(this IActor entity)
        => entity.GetValue<IExpression<bool>>(MovementCondition);

    public static void AddMovementCondition(this IActor entity, IExpression<bool> value)
        => entity.AddValue(MovementCondition, value);

    public static IEvent<Vector3> GetMovementEvent(this IActor entity)
        => entity.GetValue<IEvent<Vector3>>(MovementEvent);

    public static void AddMovementEvent(this IActor entity, IEvent<Vector3> value)
        => entity.AddValue(MovementEvent, value);
}
```

### Шаг 2: Создание Data Classes

Movement использует стандартные Atomic.Elements классы:

```csharp
// ReactiveVector3 - реактивная переменная для направления
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
                _onChange?.Invoke(_value);
            }
        }
    }

    public void Observe(Action<Vector3> callback)
    {
        _onChange += callback;
        callback?.Invoke(_value);  // Immediate call with current value
    }

    public void Unsubscribe(Action<Vector3> callback)
    {
        _onChange -= callback;
    }
}

// AndExpression - комбинирование условий
public sealed class AndExpression : IExpression<bool>
{
    private readonly Func<bool>[] _conditions;

    public AndExpression(params Func<bool>[] conditions)
    {
        _conditions = conditions;
    }

    public bool Invoke()
    {
        for (int i = 0; i < _conditions.Length; i++)
        {
            if (!_conditions[i]())
                return false;
        }
        return true;
    }
}
```

### Шаг 3: Создание UseCases

```csharp
public static class MovementUseCase
{
    // Расчет скорости с учетом направления
    public static Vector3 CalculateVelocity(Vector3 direction, float speed)
    {
        if (direction.sqrMagnitude < 0.001f)
            return Vector3.zero;

        return direction.normalized * speed;
    }

    // Проверка возможности движения
    public static bool CanMove(IActor entity)
    {
        if (!entity.HasMovementCondition())
            return true;

        IExpression<bool> condition = entity.GetMovementCondition();
        return condition.Invoke();
    }
}
```

### Шаг 4: Создание Behaviours

```csharp
// KinematicMovementBehaviour - движение без физики
public sealed class KinematicMovementBehaviour : IEntityInit, IEntityFixedTick
{
    private IValue<float> _speed;
    private IReactiveValue<Vector3> _direction;
    private IVariable<Vector3> _position;
    private IExpression<bool> _condition;
    private IEvent<Vector3> _movementEvent;

    public void Init(IEntity entity)
    {
        _speed = entity.GetMovementSpeed();
        _direction = entity.GetMovementDirection();
        _position = entity.GetPosition();
        
        if (entity.HasMovementCondition())
            _condition = entity.GetMovementCondition();

        if (entity.HasMovementEvent())
            _movementEvent = entity.GetMovementEvent();
    }

    public void FixedTick(IEntity entity, float deltaTime)
    {
        // Проверка условия движения
        if (_condition != null && !_condition.Invoke())
            return;

        Vector3 direction = _direction.Value;
        if (direction.sqrMagnitude < 0.001f)
            return;

        // Расчет и применение движения
        Vector3 velocity = MovementUseCase.CalculateVelocity(direction, _speed.Value);
        _position.Value += velocity * deltaTime;

        // Вызов события движения
        _movementEvent?.Invoke(velocity);
    }
}

// CharacterMoveBehaviour - копирование направления от PlayerContext
public sealed class CharacterMoveBehaviour : IEntityInit, IEntityFixedTick
{
    private IVariable<Vector3> _movementDirection;
    private IVariable<Vector3> _rotationDirection;

    public void Init(IEntity entity)
    {
        _movementDirection = entity.GetMovementDirection();
        
        if (entity.HasRotationDirection())
            _rotationDirection = entity.GetRotationDirection();
    }

    public void FixedTick(IEntity entity, float deltaTime)
    {
        Vector3 direction = _movementDirection.Value;

        // Копирование направления движения в направление поворота
        if (_rotationDirection != null && direction.sqrMagnitude > 0.001f)
            _rotationDirection.Value = direction;
    }
}
```

### Шаг 5: Создание Installer

```csharp
public sealed class CharacterInstaller : SceneEntityInstaller<IActor>
{
    [SerializeField] private Const<float> _moveSpeed = 3;
    [SerializeField] private Health _health = new Health(3);

    public override void Install(IActor entity)
    {
        // Movement System
        entity.AddMovementSpeed(_moveSpeed);
        entity.AddMovementDirection(new ReactiveVector3());
        entity.AddMovementCondition(new AndExpression(_health.Exists)); // Не может двигаться, если мертв
        entity.AddMovementEvent(new Event<Vector3>());
        entity.AddBehaviour<KinematicMovementBehaviour>();
        entity.AddBehaviour<CharacterMoveBehaviour>();

        // ... другие компоненты
    }
}
```

### Шаг 6: Интеграция с другими фичами

**Input System → Movement:**
```csharp
// CharacterMoveController - получает ввод и устанавливает направление
public sealed class CharacterMoveController : IEntityInit<IPlayerContext>, IEntityTick
{
    private IActor _character;
    private IPlayerContext _playerContext;
    private IGameContext _gameContext;

    public void Init(IPlayerContext context)
    {
        _character = context.GetCharacter();
        _playerContext = context;
        _gameContext = GameContext.Instance;
    }

    public void Tick(IEntity entity, float deltaTime)
    {
        // Получить направление от Input UseCase
        Vector3 direction = MoveInputUseCase.GetMoveDirection(_playerContext, _gameContext);
        
        // Установить направление движения (Producer)
        _character.GetMovementDirection().Value = direction;
    }
}

// KinematicMovementBehaviour читает направление (Consumer)
```

**Animation System:**
```csharp
public sealed class MoveAnimBehaviour : IEntityInit, IEntityDispose
{
    private IReactiveValue<Vector3> _movementDirection;
    private Animator _animator;
    private static readonly int IsMoving = Animator.StringToHash("IsMoving");

    public void Init(IEntity entity)
    {
        _movementDirection = entity.GetMovementDirection();
        _animator = entity.GetAnimator();

        // Подписка на изменение направления
        _movementDirection.Observe(this.OnDirectionChanged);
    }

    public void Dispose(IEntity entity)
    {
        _movementDirection.Unsubscribe(this.OnDirectionChanged);
    }

    private void OnDirectionChanged(Vector3 direction)
    {
        bool isMoving = direction.sqrMagnitude > 0.001f;
        _animator.SetBool(IsMoving, isMoving);
    }
}
```

### Шаг 7: Best Practices и зависимости

**Зависимости:**
- 📦 Transform System (Position)
- 📦 Health System (для MovementCondition)
- 📦 Atomic.Elements (ReactiveVariable, IExpression, Event)

**Best Practices:**

1. ✅ **FixedTick для движения** (стабильная физика)
   ```csharp
   public void FixedTick(IEntity entity, float deltaTime)
   {
       _position.Value += velocity * deltaTime;
   }
   ```

2. ✅ **Проверка sqrMagnitude вместо magnitude** (производительность)
   ```csharp
   if (direction.sqrMagnitude < 0.001f)
       return;  // Быстрее, чем magnitude
   ```

3. ✅ **Используйте Conditions для валидации**
   ```csharp
   entity.AddMovementCondition(new AndExpression(_health.Exists, _isGrounded));
   ```

4. ✅ **Normalized направление для постоянной скорости**
   ```csharp
   Vector3 velocity = direction.normalized * _speed.Value;
   ```

5. ✅ **ReactiveVariable для Producer-Consumer паттерна**
   ```csharp
   // Producer (Input)
   _character.GetMovementDirection().Value = inputDirection;
   
   // Consumer (Movement)
   Vector3 direction = _movementDirection.Value;
   ```

**Типичные ошибки:**

❌ **Ошибка 1:** Tick вместо FixedTick для движения
```csharp
// НЕПРАВИЛЬНО - нестабильная физика
public void Tick(IEntity entity, float deltaTime)
{
    _position.Value += velocity * deltaTime;
}
```

✅ **Исправление:**
```csharp
// ПРАВИЛЬНО
public void FixedTick(IEntity entity, float deltaTime)
{
    _position.Value += velocity * deltaTime;
}
```

❌ **Ошибка 2:** Забыли нормализовать направление
```csharp
// НЕПРАВИЛЬНО - скорость зависит от длины вектора
Vector3 velocity = direction * _speed.Value;
```

✅ **Исправление:**
```csharp
// ПРАВИЛЬНО - постоянная скорость
Vector3 velocity = direction.normalized * _speed.Value;
```

---

## Feature 5: Weapon & Bullet System (⭐⭐⭐)

**Сложность:** Advanced Mechanics
**Зависимости:** Transform, Team, Health, Damage
**Используется в:** Combat

### Описание

Weapon System управляет стрельбой с cooldown и ammo. Bullet System использует **Object Pooling** для эффективного управления памятью.

### Шаг 1: Добавление в EntityAPI

```csharp
// ActorAPI для Character
public static class ActorAPI
{
    public static readonly int Weapon;          // IWeapon
    public static readonly int FireAction;      // IAction
    public static readonly int FireEvent;       // IEvent
    public static readonly int FireCondition;   // IExpression<bool>
    public static readonly int FirePoint;       // Transform

    // Для Bullet
    public static readonly int Damage;          // IValue<int>
    public static readonly int Lifetime;        // Cooldown
    public static readonly int DestroyAction;   // IAction
}

// GameContextAPI для BulletPool
public static class GameContextAPI
{
    public static readonly int BulletPool;      // IEntityPool<IActor>

    public static IEntityPool<IActor> GetBulletPool(this IGameContext context)
        => context.GetValue<IEntityPool<IActor>>(BulletPool);

    public static void AddBulletPool(this IGameContext context, IEntityPool<IActor> value)
        => context.AddValue(BulletPool, value);
}
```

### Шаг 2: Создание Data Classes

```csharp
// Cooldown - таймер для перезарядки
[Serializable]
public sealed class Cooldown
{
    [SerializeField] private float duration;
    private float currentTime;

    public Cooldown(float duration)
    {
        this.duration = duration;
        this.currentTime = duration;
    }

    public bool IsCompleted() => currentTime >= duration;
    
    public void ResetTime() => currentTime = 0;

    public void Tick(float deltaTime)
    {
        if (currentTime < duration)
            currentTime += deltaTime;
    }

    public float GetProgress() => currentTime / duration;
}

// IWeapon - интерфейс оружия
public interface IWeapon : IEntity
{
}
```

### Шаг 3: Создание UseCases

```csharp
// BulletUseCase - управление пулями
public static class BulletUseCase
{
    public static IActor Spawn(
        IGameContext context,
        Vector3 position,
        Quaternion rotation,
        TeamType teamType)
    {
        // Взять пулю из пула (или создать новую)
        IActor bullet = context.GetBulletPool().Rent();
        
        // Установка параметров
        bullet.GetPosition().Value = position;
        bullet.GetRotation().Value = rotation;
        bullet.GetTeamType().Value = teamType;
        bullet.GetLifetime().ResetTime();

        return bullet;
    }

    public static void Despawn(IGameContext context, IActor bullet)
    {
        // Вернуть пулю в пул для переиспользования
        context.GetBulletPool().Return(bullet);
    }
}

// FireInputUseCase - проверка ввода стрельбы
public static class FireInputUseCase
{
    public static bool FireRequired(IPlayerContext playerContext, IGameContext gameContext)
    {
        if (!GameCycleUseCase.IsPlaying(gameContext))
            return false;

        InputMap inputMap = playerContext.GetInputMap();
        return Input.GetKey(inputMap.Fire);
    }
}
```

### Шаг 4: Создание Behaviours

```csharp
// CharacterFireController - обработка стрельбы
public sealed class CharacterFireController : IEntityInit<IPlayerContext>, IEntityTick
{
    private IGameContext _gameContext;
    private IActor _character;
    private IPlayerContext _playerContext;

    public void Init(IPlayerContext context)
    {
        _character = context.GetCharacter();
        _playerContext = context;
        _gameContext = GameContext.Instance;
    }

    public void Tick(IEntity entity, float deltaTime)
    {
        if (FireInputUseCase.FireRequired(_playerContext, _gameContext))
        {
            // Вызов action стрельбы
            _character.GetFireAction().Invoke();
        }
    }
}

// BulletMoveBehaviour - движение пули вперед
public sealed class BulletMoveBehaviour : IEntityInit, IEntityFixedTick
{
    private IValue<float> _speed;
    private IVariable<Vector3> _position;
    private IVariable<Quaternion> _rotation;

    public void Init(IEntity entity)
    {
        _speed = entity.GetMovementSpeed();
        _position = entity.GetPosition();
        _rotation = entity.GetRotation();
    }

    public void FixedTick(IEntity entity, float deltaTime)
    {
        Vector3 forward = _rotation.Value * Vector3.forward;
        _position.Value += forward * _speed.Value * deltaTime;
    }
}

// LifetimeBehaviour - автоматическое уничтожение по таймеру
public sealed class LifetimeBehaviour : IEntityInit, IEntityFixedTick
{
    private Cooldown _lifetime;
    private IAction _destroyAction;

    public void Init(IEntity entity)
    {
        _lifetime = entity.GetLifetime();
        _destroyAction = entity.GetDestroyAction();
    }

    public void FixedTick(IEntity entity, float deltaTime)
    {
        _lifetime.Tick(deltaTime);

        if (_lifetime.IsCompleted())
            _destroyAction.Invoke();
    }
}

// BulletCollisionBehaviour - обработка столкновений
public sealed class BulletCollisionBehaviour : IEntityInit, IEntityEnable, IEntityDisable
{
    private readonly IGameContext _gameContext;
    private TriggerEvents _triggerEvents;
    private IActor _bullet;

    public BulletCollisionBehaviour(IGameContext gameContext)
    {
        _gameContext = gameContext;
    }

    public void Init(IEntity entity)
    {
        _bullet = (IActor)entity;
        _triggerEvents = entity.GetTrigger();
    }

    public void Enable(IEntity entity)
    {
        _triggerEvents.OnEntered += this.OnTriggerEntered;
    }

    public void Disable(IEntity entity)
    {
        _triggerEvents.OnEntered -= this.OnTriggerEntered;
    }

    private void OnTriggerEntered(Collider collider)
    {
        if (!collider.TryGetComponent(out IActor target))
            return;

        if (!target.HasDamageableTag())
            return;

        // Проверка friendly fire
        TeamType bulletTeam = _bullet.GetTeamType().Value;
        TeamType targetTeam = target.GetTeamType().Value;

        if (bulletTeam == targetTeam)
            return;

        // Нанести урон
        int damage = _bullet.GetDamage().Value;
        target.GetTakeDamageEvent().Invoke(new DamageArgs
        {
            instigator = _bullet,
            victim = target,
            damage = damage
        });

        // Уничтожить пулю
        _bullet.GetDestroyAction().Invoke();
    }
}
```

### Шаг 5: Создание Installers

```csharp
// ProjectileWeaponInstaller - оружие, стреляющее пулями
public sealed class ProjectileWeaponInstaller : SceneEntityInstaller<IWeapon>
{
    [SerializeField] private Actor _owner;
    [SerializeField] private Transform _firePoint;
    [SerializeField] private ReactiveVariable<int> _ammo = 100;
    [SerializeField] private Cooldown _cooldown = 0.5f;

    public override void Install(IWeapon weapon)
    {
        weapon.AddFireEvent(new Event());

        // Action стрельбы
        weapon.AddFireAction(new InlineAction(() =>
        {
            // Проверка ammo и cooldown
            if (_ammo.Value <= 0 || !_cooldown.IsCompleted())
                return;

            _ammo.Value--;

            // Создание пули
            BulletUseCase.Spawn(
                GameContext.Instance,
                _firePoint.position,
                _firePoint.rotation,
                _owner.GetTeamType().Value
            );

            _cooldown.ResetTime();
            weapon.GetFireEvent().Invoke();
        }));

        // Tick для cooldown
        weapon.WhenFixedTick(_cooldown.Tick);
    }
}

// BulletInstaller - установка компонентов пули
public sealed class BulletInstaller : SceneEntityInstaller<IActor>
{
    [SerializeField] private GameObject _gameObject;
    [SerializeField] private Transform _transform;
    [SerializeField] private Const<float> _moveSpeed = 10;
    [SerializeField] private Const<int> _damage = 1;
    [SerializeField] private TriggerEvents _trigger;
    [SerializeField] private Cooldown _lifetime = 5;

    public override void Install(IActor entity)
    {
        GameContext.TryGetInstance(out GameContext gameContext);

        // Transform
        entity.AddPosition(new TransformPositionVariable(_transform));
        entity.AddRotation(new TransformRotationVariable(_transform));

        // Lifetime
        entity.AddLifetime(_lifetime);
        entity.AddBehaviour<LifetimeBehaviour>();

        // Team
        entity.AddTeamType(new ReactiveVariable<TeamType>());
        entity.AddBehaviour<TeamPhysicsLayerBehaviour>();

        // Movement
        entity.AddMovementSpeed(_moveSpeed);
        entity.AddBehaviour<BulletMoveBehaviour>();

        // Physics & Collision
        entity.AddTrigger(_trigger);
        entity.AddPhysicsLayer(new InlineVariable<int>(
            getter: () => _gameObject.layer,
            setter: value => _gameObject.layer = value
        ));
        entity.AddBehaviour(new BulletCollisionBehaviour(gameContext));

        // Damage
        entity.AddDamage(_damage);
        entity.AddDestroyAction(new InlineAction(() => 
            BulletUseCase.Despawn(gameContext, entity)
        ));
    }
}

// CharacterFireAction - action для стрельбы персонажа
public sealed class CharacterFireAction : IAction
{
    private readonly IActor _character;

    public CharacterFireAction(IActor character)
    {
        _character = character;
    }

    public void Invoke()
    {
        // Проверка условия стрельбы
        if (_character.HasFireCondition())
        {
            IExpression<bool> condition = _character.GetFireCondition();
            if (!condition.Invoke())
                return;
        }

        // Вызов действия оружия
        IWeapon weapon = _character.GetWeapon();
        weapon.GetFireAction().Invoke();
    }
}
```

### Шаг 6: Интеграция с другими фичами

**GameContext - BulletPool:**
```csharp
public sealed class GameContextInstaller : SceneEntityInstaller<IGameContext>
{
    [SerializeField] private ActorPool _bulletPool;

    public override void Install(IGameContext context)
    {
        context.AddBulletPool(_bulletPool);
        // ... другие системы
    }
}
```

**CharacterInstaller - установка оружия:**
```csharp
public sealed class CharacterInstaller : SceneEntityInstaller<IActor>
{
    [SerializeField] private Health _health = new Health(3);
    [SerializeField] private Weapon _initialWeapon;

    public override void Install(IActor entity)
    {
        // Combat
        entity.AddFireCondition(new AndExpression(_health.Exists)); // Не стреляет, если мертв
        entity.AddFireAction(new CharacterFireAction(entity));
        entity.AddFireEvent(new Event());
        entity.AddWeapon(_initialWeapon);

        // ... другие системы
    }
}
```

### Шаг 7: Best Practices и зависимости

**Зависимости:**
- 📦 Transform System (для позиции/ротации пуль)
- 📦 Team System (для проверки friendly fire)
- 📦 Damage System (для нанесения урона)
- 📦 Object Pooling (ActorPool)

**Best Practices:**

1. ✅ **Используйте Object Pooling для пуль**
   ```csharp
   IActor bullet = context.GetBulletPool().Rent();    // Взять из пула
   context.GetBulletPool().Return(bullet);            // Вернуть в пул
   ```

2. ✅ **Cooldown для контроля скорострельности**
   ```csharp
   if (!_cooldown.IsCompleted())
       return;  // Еще не перезарядилось
   
   _cooldown.ResetTime();
   ```

3. ✅ **Проверка ammo перед стрельбой**
   ```csharp
   if (_ammo.Value <= 0)
       return;  // Патроны закончились
   
   _ammo.Value--;
   ```

4. ✅ **Lifetime для автоматического уничтожения**
   ```csharp
   entity.AddLifetime(new Cooldown(5f));  // Пуля живет 5 секунд
   entity.AddBehaviour<LifetimeBehaviour>();
   ```

5. ✅ **FireCondition для валидации**
   ```csharp
   entity.AddFireCondition(new AndExpression(_health.Exists, _hasAmmo));
   ```

**Типичные ошибки:**

❌ **Ошибка 1:** Instantiate вместо Object Pooling
```csharp
// НЕПРАВИЛЬНО - создает мусор
GameObject.Instantiate(bulletPrefab);
```

✅ **Исправление:**
```csharp
// ПРАВИЛЬНО - переиспользует объекты
context.GetBulletPool().Rent();
```

❌ **Ошибка 2:** Нет проверки friendly fire
```csharp
// НЕПРАВИЛЬНО - стреляет по своим
target.GetTakeDamageEvent().Invoke(damageArgs);
```

✅ **Исправление:**
```csharp
// ПРАВИЛЬНО
if (bulletTeam != targetTeam)
    target.GetTakeDamageEvent().Invoke(damageArgs);
```

❌ **Ошибка 3:** Забыли сбросить Lifetime при переиспользовании
```csharp
// НЕПРАВИЛЬНО - пуля уничтожится сразу
IActor bullet = pool.Rent();
```

✅ **Исправление:**
```csharp
// ПРАВИЛЬНО
IActor bullet = pool.Rent();
bullet.GetLifetime().ResetTime();
```

---

## Feature 6: UI System with Presenter Pattern (⭐⭐⭐)

**Сложность:** ⭐⭐⭐ (Advanced)
**Тип фичи:** UI/Presentation Layer
**Связанные компоненты:** MenuUI Context, GameUI Context, Presenters, AppContext, GameContext

### Обзор

UI System в Shooter Demo демонстрирует **production-ready Presenter Pattern** с двумя независимыми UI контекстами:
- **MenuUI**: Навигация по меню (Start Screen, Level Selection Screen)
- **GameUI**: Игровой интерфейс (Countdown, Kills tracking)

**Ключевые особенности:**
- ✅ Typed Lifecycle Interfaces (IMenuUIInit, IGameUIInit)
- ✅ Context Injection вместо Singleton
- ✅ Subscription<T> для автоматической очистки
- ✅ 5 типов Presenters (Simple Reactive, Screen, Composite, Child, Entity)
- ✅ Полное разделение UI логики от Unity

### Архитектура UI System

```
App Level
├── MenuUI (Entity - UI Context для меню)
│   ├── StartScreenPresenter (Screen Presenter)
│   │   └── Управляет кнопками Start/Select Level/Exit
│   │
│   └── LevelScreenPresenter (Composite Presenter)
│       └── Создает 10x LevelItemPresenter (Child Presenters)
│
└── GameUI (Entity - UI Context для игры)
    ├── CountdownPresenter (Simple Reactive Presenter)
    │   └── Отображает игровое время
    │
    └── KillsPresenter x2 (Dictionary Filtering Presenter)
        ├── Blue Team Kills
        └── Red Team Kills
```

### Шаг 1: Интерфейсы (Typed Lifecycle)

**Классический подход vs Production-Ready:**

```csharp
// ❌ Классический подход - generic интерфейсы
public sealed class OldPresenter : IEntityInit<IMenuUI>, IEntityDispose
{
    public void Init(IMenuUI entity) { }
    public void Dispose(IEntity entity) { }
}

// ✅ Production-Ready - typed интерфейсы для каждого контекста
public sealed class StartScreenPresenter :
    IMenuUIInit,      // Вместо IEntityInit<IMenuUI>
    IMenuUIEnable,    // Вместо IEntityEnable
    IMenuUIDisable    // Вместо IEntityDisable
{
    public void Init(IMenuUI context) { }
    public void Enable(IMenuUI entity) { }
    public void Disable(IMenuUI entity) { }
}
```

**Определение типизированных интерфейсов:**

```csharp
// MenuUI Lifecycle Interfaces
public interface IMenuUIInit : IEntityInit<IMenuUI> { }
public interface IMenuUIDispose : IEntityDispose { }
public interface IMenuUIEnable : IEntityEnable { }
public interface IMenuUIDisable : IEntityDisable { }

// GameUI Lifecycle Interfaces
public interface IGameUIInit : IEntityInit<IGameUI> { }
public interface IGameUIDispose : IEntityDispose { }
public interface IGameUIEnable : IEntityEnable { }
public interface IGameUIDisable : IEntityDisable { }
```

**Преимущества:**
- 🔍 Легко найти все Presenters для конкретного контекста (Find Usages)
- 📝 Явная типизация без дженериков
- 🛡️ Компилятор гарантирует правильный контекст
- 🧹 Более чистый и понятный код

### Шаг 2: Context Injection Pattern

**Legacy Pattern vs Production-Ready:**

```csharp
// ❌ Legacy - Singleton паттерн
public sealed class OldPresenter : IGameUIInit, IGameUIDispose
{
    private readonly TMP_Text _view;

    public OldPresenter(TMP_Text view)
    {
        _view = view;
    }

    public void Init(IGameUI entity)
    {
        // Singleton - скрытая зависимость
        IGameContext context = GameContext.Instance;
        var time = context.GetGameTime();
    }
}

// ✅ Production-Ready - Constructor Injection
public sealed class CountdownPresenter : IGameUIInit, IGameUIDispose
{
    private readonly TMP_Text _view;
    private readonly IGameContext _gameContext;  // Injected!

    private Subscription<float> _subscription;

    public CountdownPresenter(TMP_Text view, IGameContext gameContext)
    {
        _view = view;
        _gameContext = gameContext;  // Явная зависимость
    }

    public void Init(IGameUI entity)
    {
        _subscription = _gameContext
            .GetGameTime()
            .Observe(this.OnGameTimeChanged);
    }

    public void Dispose(IGameUI entity)
    {
        _subscription.Dispose();
    }

    private void OnGameTimeChanged(float time)
    {
        _view.text = $"Game Time: {time:F0}";
    }
}
```

**Преимущества Context Injection:**
- ✅ Явные зависимости (видно в конструкторе)
- ✅ Легко тестировать (можно подменить mock)
- ✅ Нет скрытой связности
- ✅ Соответствует SOLID принципам

### Шаг 3: Subscription Pattern

**Manual Unsubscribe vs Subscription<T>:**

```csharp
// ❌ Manual Unsubscribe - легко забыть
public sealed class OldPresenter : IGameUIInit, IGameUIDispose
{
    private IReactiveValue<float> _time;

    public void Init(IGameUI entity)
    {
        _time = GameContext.Instance.GetGameTime();
        _time.Observe(this.OnTimeChanged);
    }

    public void Dispose(IGameUI entity)
    {
        _time.Unsubscribe(this.OnTimeChanged);  // Легко забыть!
    }
}

// ✅ Subscription<T> - автоматическая очистка
public sealed class CountdownPresenter : IGameUIInit, IGameUIDispose
{
    private Subscription<float> _subscription;  // Хранит подписку

    public void Init(IGameUI entity)
    {
        _subscription = _gameContext
            .GetGameTime()
            .Observe(this.OnGameTimeChanged);
    }

    public void Dispose(IGameUI entity)
    {
        _subscription.Dispose();  // Невозможно забыть
    }
}
```

**Преимущества Subscription<T>:**
- 🛡️ Невозможно забыть отписаться (compile-time гарантия)
- 📦 Поддержка множественных подписок (Subscription.Compose)
- 🧹 Более чистый код

### Шаг 4: MenuUI Context - Навигация по меню

**MenuUI Structure:**

```
MenuUI (Entity)
├── StartScreenPresenter (Screen)
│   ├── OnStartClicked → GameLoadingUseCase.StartGame()
│   ├── OnSelectLevelClicked → ScreenUseCase.ShowScreen<LevelScreenView>()
│   └── OnExitClicked → QuitUseCase.Quit()
│
└── LevelScreenPresenter (Composite)
    ├── OnCloseClicked → ScreenUseCase.ShowScreen<StartScreenView>()
    └── SpawnLevelItems() → Создает 10x LevelItemPresenter
        └── LevelItemPresenter (Child)
            └── OnClicked → GameLoadingUseCase.StartGame(level)
```

**StartScreenPresenter (Screen Presenter):**

```csharp
public sealed class StartScreenPresenter :
    IMenuUIInit,
    IMenuUIEnable,
    IMenuUIDisable
{
    private readonly StartScreenView _screenView;
    private readonly IAppContext _appContext;  // Context Injection

    private IMenuUI _uIContext;

    public StartScreenPresenter(StartScreenView screenView, IAppContext appContext)
    {
        _screenView = screenView;
        _appContext = appContext;
    }

    public void Init(IMenuUI context)
    {
        _uIContext = context;
    }

    public void Enable(IMenuUI entity)
    {
        _screenView.OnSelectLevelClicked += this.OnSelectLevelClicked;
        _screenView.OnStartClicked += this.OnStartClicked;
        _screenView.OnExitClicked += QuitUseCase.Quit;
    }

    public void Disable(IMenuUI entity)
    {
        _screenView.OnStartClicked -= this.OnStartClicked;
        _screenView.OnSelectLevelClicked -= this.OnSelectLevelClicked;
        _screenView.OnExitClicked -= QuitUseCase.Quit;
    }

    private void OnStartClicked() =>
        GameLoadingUseCase.StartGame(_appContext);

    private void OnSelectLevelClicked() =>
        ScreenUseCase.ShowScreen<LevelScreenView>(_uIContext);
}
```

**LevelScreenPresenter (Composite Presenter):**

```csharp
public sealed class LevelScreenPresenter :
    IMenuUIInit,
    IMenuUIDispose,
    IMenuUIEnable,
    IMenuUIDisable
{
    private readonly IAppContext _appContext;
    private readonly LevelScreenView _screenView;
    private IMenuUI _uiContext;

    public LevelScreenPresenter(LevelScreenView screenView, IAppContext appContext)
    {
        _screenView = screenView;
        _appContext = appContext;
    }

    public void Init(IMenuUI context)
    {
        _uiContext = context;
        this.SpawnLevelItems();
    }

    private void SpawnLevelItems()
    {
        int startLevel = _appContext.GetStartLevel().Value;
        int maxLevel = _appContext.GetMaxLevel().Value;

        for (int i = startLevel; i <= maxLevel; i++)
        {
            LevelItemView itemView = _screenView.CreateItem();
            LevelItemPresenter itemPresenter = new LevelItemPresenter(
                _appContext,
                i,
                itemView
            );
            _uiContext.AddBehaviour(itemPresenter);  // Composite создаёт Children
        }
    }

    public void Enable(IMenuUI entity)
    {
        _screenView.OnCloseClicked += this.OnCloseClicked;
    }

    public void Disable(IMenuUI entity)
    {
        _screenView.OnCloseClicked -= this.OnCloseClicked;
    }

    private void OnCloseClicked() =>
        ScreenUseCase.ShowScreen<StartScreenView>(_uiContext);

    public void Dispose(IMenuUI entity)
    {
        _uiContext.DelBehaviours<LevelItemPresenter>();  // Composite удаляет Children
        _screenView.ClearAllItems();
    }
}
```

**LevelItemPresenter (Child Presenter):**

```csharp
public sealed class LevelItemPresenter : IMenuUIInit, IMenuUIDispose
{
    private readonly IAppContext _context;
    private readonly LevelItemView _view;
    private readonly int _level;

    public LevelItemPresenter(IAppContext context, int level, LevelItemView view)
    {
        _context = context;
        _view = view;
        _level = level;
    }

    public void Init(IMenuUI context)
    {
        int currentLevel = _context.GetCurrentLevel().Value;

        // Логика визуального состояния
        if (currentLevel == _level)
            _view.SetAsCurrent();
        else if (currentLevel > _level)
            _view.SetAsCompleted();
        else
            _view.SetAsNotCompleted();

        _view.SetLevel(_level.ToString());
        _view.OnClicked += this.OnClicked;
    }

    public void Dispose(IMenuUI entity)
    {
        _view.OnClicked -= this.OnClicked;
    }

    private void OnClicked() =>
        GameLoadingUseCase.StartGame(_context, _level);
}
```

### Шаг 5: GameUI Context - Игровой интерфейс

**GameUI Structure:**

```
GameUI (Entity)
├── CountdownPresenter (Simple Reactive)
│   └── Observe: GameContext.GetGameTime()
│
├── KillsPresenter (Dictionary Filtering) - Blue Team
│   └── Observe: GameContext.GetLeaderboard()[TeamType.BLUE]
│
└── KillsPresenter (Dictionary Filtering) - Red Team
    └── Observe: GameContext.GetLeaderboard()[TeamType.RED]
```

**CountdownPresenter (Simple Reactive Presenter):**

```csharp
public sealed class CountdownPresenter : IGameUIInit, IGameUIDispose
{
    private readonly TMP_Text _view;
    private readonly IGameContext _gameContext;  // Context Injection

    private Subscription<float> _subscription;   // Subscription Pattern

    public CountdownPresenter(TMP_Text view, IGameContext gameContext)
    {
        _view = view;
        _gameContext = gameContext;
    }

    public void Init(IGameUI entity)
    {
        _subscription = _gameContext
            .GetGameTime()
            .Observe(this.OnGameTimeChanged);
    }

    public void Dispose(IGameUI entity)
    {
        _subscription.Dispose();
    }

    private void OnGameTimeChanged(float time)
    {
        _view.text = $"Game Time: {time:F0}";
    }
}
```

**KillsPresenter (Dictionary Filtering Presenter):**

```csharp
public sealed class KillsPresenter : IGameUIInit, IGameUIDispose
{
    private readonly TMP_Text _text;
    private readonly IGameContext _gameContext;
    private readonly TeamType _teamType;  // Фильтр

    private IReactiveDictionary<TeamType, int> _leaderboard;

    public KillsPresenter(TMP_Text text, IGameContext gameContext, TeamType teamType)
    {
        _text = text;
        _gameContext = gameContext;
        _teamType = teamType;
    }

    public void Init(IGameUI entity)
    {
        _leaderboard = _gameContext.GetLeaderboard();
        _leaderboard.OnItemChanged += this.OnLeaderboardChanged;

        // Initial value
        this.UpdateText(_leaderboard[_teamType]);
    }

    public void Dispose(IGameUI entity)
    {
        _leaderboard.OnItemChanged -= this.OnLeaderboardChanged;
    }

    private void OnLeaderboardChanged(TeamType team, int kills)
    {
        if (team == _teamType)
            this.UpdateText(kills);
    }

    private void UpdateText(int kills)
    {
        _text.text = $"Kills: {kills}";
    }
}
```

### Шаг 6: View Classes

**StartScreenView (MonoBehaviour):**

```csharp
public sealed class StartScreenView : MonoBehaviour
{
    [SerializeField] private Button _startButton;
    [SerializeField] private Button _selectLevelButton;
    [SerializeField] private Button _exitButton;

    public event Action OnStartClicked
    {
        add => _startButton.onClick.AddListener(value.Invoke);
        remove => _startButton.onClick.RemoveListener(value.Invoke);
    }

    public event Action OnSelectLevelClicked
    {
        add => _selectLevelButton.onClick.AddListener(value.Invoke);
        remove => _selectLevelButton.onClick.RemoveListener(value.Invoke);
    }

    public event Action OnExitClicked
    {
        add => _exitButton.onClick.AddListener(value.Invoke);
        remove => _exitButton.onClick.RemoveListener(value.Invoke);
    }
}
```

**LevelScreenView (MonoBehaviour с Factory Method):**

```csharp
public sealed class LevelScreenView : MonoBehaviour
{
    [SerializeField] private LevelItemView _itemPrefab;
    [SerializeField] private Transform _itemsContainer;
    [SerializeField] private Button _closeButton;

    public event Action OnCloseClicked
    {
        add => _closeButton.onClick.AddListener(value.Invoke);
        remove => _closeButton.onClick.RemoveListener(value.Invoke);
    }

    // Factory Method для создания Child Views
    public LevelItemView CreateItem()
    {
        return Instantiate(_itemPrefab, _itemsContainer);
    }

    public void ClearAllItems()
    {
        foreach (Transform child in _itemsContainer)
            Destroy(child.gameObject);
    }
}
```

**LevelItemView (MonoBehaviour):**

```csharp
public sealed class LevelItemView : MonoBehaviour
{
    [SerializeField] private Button _button;
    [SerializeField] private TMP_Text _levelText;
    [SerializeField] private Image _background;

    [SerializeField] private Color _currentColor = Color.yellow;
    [SerializeField] private Color _completedColor = Color.green;
    [SerializeField] private Color _notCompletedColor = Color.gray;

    public event Action OnClicked
    {
        add => _button.onClick.AddListener(value.Invoke);
        remove => _button.onClick.RemoveListener(value.Invoke);
    }

    public void SetLevel(string level)
    {
        _levelText.text = level;
    }

    public void SetAsCurrent()
    {
        _background.color = _currentColor;
        _button.interactable = true;
    }

    public void SetAsCompleted()
    {
        _background.color = _completedColor;
        _button.interactable = true;
    }

    public void SetAsNotCompleted()
    {
        _background.color = _notCompletedColor;
        _button.interactable = false;
    }
}
```

### Шаг 7: UseCases для UI Logic

**ScreenUseCase - навигация между экранами:**

```csharp
public static class ScreenUseCase
{
    public static void ShowScreen<TScreenView>(IMenuUI menuContext)
        where TScreenView : MonoBehaviour
    {
        // Скрыть все экраны
        var allScreens = Object.FindObjectsOfType<MonoBehaviour>()
            .Where(x => x is StartScreenView || x is LevelScreenView);

        foreach (var screen in allScreens)
            screen.gameObject.SetActive(false);

        // Показать нужный экран
        TScreenView targetScreen = Object.FindObjectOfType<TScreenView>();
        if (targetScreen != null)
            targetScreen.gameObject.SetActive(true);
    }
}
```

**GameLoadingUseCase - загрузка игры:**

```csharp
public static class GameLoadingUseCase
{
    public static void StartGame(IAppContext appContext)
    {
        int currentLevel = appContext.GetCurrentLevel().Value;
        StartGame(appContext, currentLevel);
    }

    public static void StartGame(IAppContext appContext, int level)
    {
        // Обновить текущий уровень
        appContext.GetCurrentLevel().Value = level;

        // Запустить загрузку игры
        ILoadingTask loadingTask = appContext.GetGameLoadingAction();
        loadingTask.Start();
    }
}
```

**QuitUseCase - выход из приложения:**

```csharp
public static class QuitUseCase
{
    public static void Quit()
    {
#if UNITY_EDITOR
        UnityEditor.EditorApplication.isPlaying = false;
#else
        Application.Quit();
#endif
    }
}
```

### Шаг 8: Integration в AppContext

**MenuUIInstaller:**

```csharp
public sealed class MenuUIInstaller : MonoBehaviour
{
    [SerializeField] private StartScreenView _startScreenView;
    [SerializeField] private LevelScreenView _levelScreenView;

    private void Awake()
    {
        IAppContext appContext = AppContext.Instance;

        // Создать MenuUI Entity
        var menuUI = new Entity("MenuUI");

        // Добавить Screen Presenters с Context Injection
        menuUI.AddBehaviour(new StartScreenPresenter(_startScreenView, appContext));
        menuUI.AddBehaviour(new LevelScreenPresenter(_levelScreenView, appContext));

        // Init и Enable
        menuUI.Init();
        menuUI.Enable();

        // Показать стартовый экран
        ScreenUseCase.ShowScreen<StartScreenView>(menuUI as IMenuUI);
    }
}
```

**GameUIInstaller:**

```csharp
public sealed class GameUIInstaller : MonoBehaviour
{
    [SerializeField] private TMP_Text _countdownText;
    [SerializeField] private TMP_Text _blueKillsText;
    [SerializeField] private TMP_Text _redKillsText;

    private void Start()
    {
        IGameContext gameContext = GameContext.Instance;

        // Создать GameUI Entity
        var gameUI = new Entity("GameUI");

        // Добавить Presenters с Context Injection
        gameUI.AddBehaviour(new CountdownPresenter(_countdownText, gameContext));
        gameUI.AddBehaviour(new KillsPresenter(_blueKillsText, gameContext, TeamType.BLUE));
        gameUI.AddBehaviour(new KillsPresenter(_redKillsText, gameContext, TeamType.RED));

        // Init и Enable
        gameUI.Init();
        gameUI.Enable();
    }
}
```

### Шаг 9: Best Practices и зависимости

**Зависимости:**
- 📦 AppContext (для MenuUI Presenters)
- 📦 GameContext (для GameUI Presenters)
- 📦 TextMeshPro (для UI текста)
- 📦 Unity UI (для Button, Image)

**Best Practices:**

1. ✅ **Context Injection вместо Singleton**
   ```csharp
   // ПРАВИЛЬНО - явная зависимость
   public CountdownPresenter(TMP_Text view, IGameContext gameContext)
   {
       _gameContext = gameContext;  // Injected
   }

   // НЕПРАВИЛЬНО - скрытая зависимость
   public void Init(IGameUI entity)
   {
       var context = GameContext.Instance;  // Singleton
   }
   ```

2. ✅ **Subscription<T> для подписок**
   ```csharp
   // ПРАВИЛЬНО - автоматическая очистка
   private Subscription<float> _subscription;
   _subscription.Dispose();

   // НЕПРАВИЛЬНО - легко забыть
   _value.Observe(OnChanged);
   _value.Unsubscribe(OnChanged);  // Может забыть
   ```

3. ✅ **Typed Lifecycle Interfaces**
   ```csharp
   // ПРАВИЛЬНО - явная типизация
   public sealed class Presenter : IGameUIInit, IGameUIDispose

   // НЕПРАВИЛЬНО - generic
   public sealed class Presenter : IEntityInit<IGameUI>, IEntityDispose
   ```

4. ✅ **Composite Pattern для сложных UI**
   ```csharp
   // ПРАВИЛЬНО - разделение на Composite + Children
   LevelScreenPresenter (Composite)
       └── 10x LevelItemPresenter (Children)

   // НЕПРАВИЛЬНО - монолитный Presenter
   LevelScreenPresenter + 10 Button handlers
   ```

5. ✅ **View Factory Method для создания Child Views**
   ```csharp
   // ПРАВИЛЬНО - View создает Child Views
   public LevelItemView CreateItem()
   {
       return Instantiate(_itemPrefab, _itemsContainer);
   }

   // НЕПРАВИЛЬНО - Presenter создает GameObject'ы
   GameObject.Instantiate(prefab);
   ```

6. ✅ **Initial Value Update после подписки**
   ```csharp
   // ПРАВИЛЬНО - показываем начальное значение
   _leaderboard.OnItemChanged += this.OnLeaderboardChanged;
   this.UpdateText(_leaderboard[_teamType]);  // Initial value

   // НЕПРАВИЛЬНО - UI пустой до первого события
   _leaderboard.OnItemChanged += this.OnLeaderboardChanged;
   ```

7. ✅ **UseCases для UI Logic**
   ```csharp
   // ПРАВИЛЬНО - UseCase инкапсулирует логику
   private void OnStartClicked() =>
       GameLoadingUseCase.StartGame(_appContext);

   // НЕПРАВИЛЬНО - логика в Presenter
   private void OnStartClicked()
   {
       SceneManager.LoadScene("Game");
       PlayerPrefs.SetInt("Level", _level);
   }
   ```

**Типичные ошибки:**

❌ **Ошибка 1:** Singleton вместо Context Injection
```csharp
// НЕПРАВИЛЬНО
public void Init(IGameUI entity)
{
    var context = GameContext.Instance;  // Скрытая зависимость
}
```

✅ **Исправление:**
```csharp
// ПРАВИЛЬНО
public CountdownPresenter(TMP_Text view, IGameContext gameContext)
{
    _gameContext = gameContext;  // Явная зависимость
}
```

❌ **Ошибка 2:** Забыли отписаться (manual Unsubscribe)
```csharp
// НЕПРАВИЛЬНО
public void Init(IGameUI entity)
{
    _time.Observe(this.OnTimeChanged);
}

public void Dispose(IGameUI entity)
{
    // Забыли отписаться - memory leak!
}
```

✅ **Исправление:**
```csharp
// ПРАВИЛЬНО - Subscription<T> невозможно забыть
private Subscription<float> _subscription;

public void Dispose(IGameUI entity)
{
    _subscription.Dispose();  // Compile-time гарантия
}
```

❌ **Ошибка 3:** Composite не удаляет Children
```csharp
// НЕПРАВИЛЬНО
public void Dispose(IMenuUI entity)
{
    _screenView.ClearAllItems();  // Только визуальная очистка
}
```

✅ **Исправление:**
```csharp
// ПРАВИЛЬНО
public void Dispose(IMenuUI entity)
{
    _uiContext.DelBehaviours<LevelItemPresenter>();  // Удалить Presenters
    _screenView.ClearAllItems();                      // Удалить Views
}
```

❌ **Ошибка 4:** Не показали initial value
```csharp
// НЕПРАВИЛЬНО - UI пустой до первого события
public void Init(IGameUI entity)
{
    _leaderboard.OnItemChanged += this.OnLeaderboardChanged;
}
```

✅ **Исправление:**
```csharp
// ПРАВИЛЬНО
public void Init(IGameUI entity)
{
    _leaderboard.OnItemChanged += this.OnLeaderboardChanged;
    this.UpdateText(_leaderboard[_teamType]);  // Показать сразу
}
```

### Резюме UI System

**Что мы изучили:**

1. **Typed Lifecycle Interfaces** - IMenuUIInit, IGameUIInit вместо IEntityInit<T>
2. **Context Injection** - явные зависимости через конструктор
3. **Subscription Pattern** - автоматическая очистка подписок
4. **5 типов Presenters**:
   - Simple Reactive (CountdownPresenter)
   - Dictionary Filtering (KillsPresenter)
   - Screen (StartScreenPresenter)
   - Composite (LevelScreenPresenter)
   - Child (LevelItemPresenter)
5. **View Factory Method** - создание Child Views через View класс
6. **UseCases** - инкапсуляция UI логики
7. **Composite Pattern** - разделение сложных UI на Composite + Children

**Production-Ready паттерны:**
- ✅ Явные зависимости (DI)
- ✅ Автоматическая очистка (Subscription<T>)
- ✅ Типобезопасность (Typed Interfaces)
- ✅ Разделение ответственности (Presenter/View/UseCase)
- ✅ Тестируемость (нет Unity зависимостей в Presenters)

---

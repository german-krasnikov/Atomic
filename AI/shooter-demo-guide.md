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

# Presenter Pattern Guide - Atomic Framework v2

## 📑 Содержание

1. [Что такое Presenter](#что-такое-presenter)
2. [Когда использовать Presenters](#когда-использовать-presenters)
3. [Архитектура MVP в Atomic](#архитектура-mvp-в-atomic)
4. [Типы Presenters](#типы-presenters)
5. [Lifecycle Management](#lifecycle-management)
6. [Best Practices](#best-practices)
7. [Примеры из Sample проектов](#примеры-из-sample-проектов)
8. [Сравнение с View Behaviours](#сравнение-с-view-behaviours)

---

## Что такое Presenter

**Presenter** — это специальный тип Behaviour, который связывает UI компоненты (View) с данными из Entity или Context (Model). Он реализует паттерн **Model-View-Presenter (MVP)**, обеспечивая разделение ответственности между:

- **Model** — Entity/Context с бизнес-данными и логикой
- **View** — Unity UI компоненты (TMP_Text, Button, Image, Slider, etc.)
- **Presenter** — посредник, который слушает изменения Model и обновляет View

### Ключевые особенности

```csharp
public sealed class CountdownPresenter : IEntityInit, IEntityDispose
{
    private readonly TMP_Text _view;              // View - Unity UI
    private IReactiveVariable<float> _gameTime;   // Model - данные из Context

    public CountdownPresenter(TMP_Text view)
    {
        _view = view;
    }

    public void Init(IEntity _)
    {
        _gameTime = GameContext.Instance.GetGameTime();  // Получить Model
        _gameTime.Observe(this.OnGameTimeChanged);       // Подписаться на изменения
    }

    public void Dispose(IEntity _)
    {
        _gameTime.Unsubscribe(this.OnGameTimeChanged);   // Отписаться
    }

    private void OnGameTimeChanged(float time)
    {
        _view.text = $"Game Time: {time:F0}";            // Обновить View
    }
}
```

---

## Когда использовать Presenters

### ✅ Используйте Presenters для:

1. **UI элементов**, которые отображают данные из GameContext или AppContext
   - Score, Time, Health, Ammo counters
   - Leaderboards, Team statistics
   - Game state indicators

2. **Реакции на изменения реактивных значений**
   - IReactiveVariable, IReactiveDictionary
   - Events, Signals
   - Health.OnStateChanged, etc.

3. **Управления lifecycle UI элементов**
   - Показ/скрытие UI при Init/Enable/Disable
   - Подписка/отписка от событий
   - Создание/удаление дочерних элементов

4. **Обработки UI events и вызова UseCases**
   - Button clicks → LoadGameUseCase.StartGame()
   - Input events → ScreenUseCase.ShowScreen()

### ❌ НЕ используйте Presenters для:

1. **Визуализации игровых объектов** в сцене
   - Используйте EntityView Behaviours (PositionViewBehaviour, RotationViewBehaviour)

2. **Бизнес-логики**
   - Используйте UseCases (static utility classes)

3. **Хранения состояния**
   - Используйте Entity/Context

4. **Игровой механики**
   - Используйте Entity Behaviours (MoveBehaviour, AiBehaviour)

---

## Архитектура MVP в Atomic

```
┌─────────────────────────────────────────────────────────────┐
│                        UI Context                            │
│  (MenuUI, GameUI — Entity с UI Presenters как Behaviours)   │
└─────────────────────────────────────────────────────────────┘
                            │
                            │ AddBehaviour(presenter)
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                        Presenter                             │
│  • Implements IEntityInit/Enable/Disable/Dispose            │
│  • Получает View через constructor                          │
│  • Подписывается на Model (Context/Entity)                  │
│  • Обновляет View при изменении Model                       │
│  • Обрабатывает View events → вызывает UseCases             │
└─────────────────────────────────────────────────────────────┘
           │                                    │
           │ Observe/Subscribe                  │ UpdateUI
           ▼                                    ▼
┌──────────────────────┐              ┌──────────────────────┐
│       Model          │              │        View          │
│  • GameContext       │              │  • TMP_Text          │
│  • Entity            │              │  • Button            │
│  • Reactive Values   │              │  • Image             │
│  • Events            │              │  • Slider            │
└──────────────────────┘              └──────────────────────┘
```

### Пример интеграции

```csharp
// View - MonoBehaviour с UI компонентами
public sealed class ScoreView : MonoBehaviour
{
    [SerializeField] private TMP_Text _scoreText;

    public TMP_Text ScoreText => _scoreText;
}

// Presenter - Behaviour для UIContext
public sealed class ScorePresenter : IEntityInit, IEntityDispose
{
    private readonly TMP_Text _text;
    private IReactiveVariable<int> _score;

    public ScorePresenter(TMP_Text text)
    {
        _text = text;
    }

    public void Init(IEntity entity)
    {
        _score = GameContext.Instance.GetScore();
        _score.Observe(this.OnScoreChanged);
    }

    public void Dispose(IEntity entity)
    {
        _score.Unsubscribe(this.OnScoreChanged);
    }

    private void OnScoreChanged(int score)
    {
        _text.text = $"Score: {score}";
    }
}

// UIContext - Entity с Presenters
public sealed class GameUIInstaller : MonoBehaviour
{
    [SerializeField] private ScoreView _scoreView;

    private void Start()
    {
        var uiContext = new Entity("GameUI");

        // Добавляем Presenter как Behaviour
        uiContext.AddBehaviour(new ScorePresenter(_scoreView.ScoreText));

        uiContext.Init();
        uiContext.Enable();
    }
}
```

---

## Advanced Patterns (Production-Ready)

### 1. Typed Lifecycle Interfaces

В продакшн коде Shooter Demo используются **типизированные lifecycle интерфейсы** вместо generic версий для лучшей type safety и читаемости:

**Вместо:**
```csharp
IEntityInit<IMenuUI>
IEntityDispose
IEntityEnable
IEntityDisable
```

**Используйте:**
```csharp
IMenuUIInit        // вместо IEntityInit<IMenuUI>
IMenuUIDispose     // вместо IEntityDispose
IMenuUIEnable      // вместо IEntityEnable
IMenuUIDisable     // вместо IEntityDisable

IGameUIInit        // для GameUI context
IGameUIDispose
```

**Преимущества:**
- ✅ Явная привязка к конкретному UIContext
- ✅ Лучшая читаемость кода
- ✅ Compile-time type safety
- ✅ Автокомплит подсказывает правильные интерфейсы

**Пример:**
```csharp
public sealed class CountdownPresenter : IGameUIInit, IGameUIDispose
{
    private readonly TMP_Text _view;
    private readonly IGameContext _gameContext;

    private Subscription<float> _subscription;

    public CountdownPresenter(TMP_Text view, IGameContext gameContext)
    {
        _view = view;
        _gameContext = gameContext;
    }

    public void Init(IGameUI entity)  // Typed parameter
    {
        _subscription = _gameContext
            .GetGameTime()
            .Observe(this.OnGameTimeChanged);
    }

    public void Dispose(IGameUI entity)  // Typed parameter
    {
        _subscription.Dispose();
    }

    private void OnGameTimeChanged(float time)
    {
        _view.text = $"Game Time: {time:F0}";
    }
}
```

### 2. Context Injection Pattern

**Лучшая практика:** Все зависимости (View + Context) передаются через конструктор:

```csharp
public sealed class StartScreenPresenter :
    IMenuUIInit,
    IMenuUIEnable,
    IMenuUIDisable
{
    private readonly StartScreenView _screenView;  // View - конструктор
    private readonly IAppContext _appContext;      // Context - конструктор

    private IMenuUI _uIContext;  // UIContext - Init

    public StartScreenPresenter(StartScreenView screenView, IAppContext appContext)
    {
        _screenView = screenView;
        _appContext = appContext;
    }

    public void Init(IMenuUI context)
    {
        _uIContext = context;  // Сохраняем UIContext для навигации
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

**Почему не через Init:**
```csharp
// ❌ НЕПРАВИЛЬНО - Context через Singleton
public void Init(IMenuUI entity)
{
    _appContext = AppContext.Instance;  // Плохо!
}

// ✅ ПРАВИЛЬНО - Context через конструктор
public StartScreenPresenter(StartScreenView view, IAppContext appContext)
{
    _appContext = appContext;  // Хорошо!
}
```

### 3. Subscription Pattern с Dispose

**Используйте `Subscription<T>`** для управления подписками на reactive values:

```csharp
public sealed class CountdownPresenter : IGameUIInit, IGameUIDispose
{
    private Subscription<float> _subscription;  // Хранит подписку

    public void Init(IGameUI entity)
    {
        // Observe возвращает Subscription<T>
        _subscription = _gameContext
            .GetGameTime()
            .Observe(this.OnGameTimeChanged);
    }

    public void Dispose(IGameUI entity)
    {
        _subscription.Dispose();  // Автоматическая отписка
    }
}
```

**Преимущества перед Observe/Unsubscribe:**

```csharp
// ❌ Старый паттерн (manual)
private IReactiveVariable<float> _gameTime;

public void Init(IEntity entity)
{
    _gameTime = GameContext.Instance.GetGameTime();
    _gameTime.Observe(this.OnGameTimeChanged);
}

public void Dispose(IEntity entity)
{
    _gameTime.Unsubscribe(this.OnGameTimeChanged);  // Легко забыть!
}

// ✅ Новый паттерн (automatic)
private Subscription<float> _subscription;

public void Init(IGameUI entity)
{
    _subscription = _gameContext
        .GetGameTime()
        .Observe(this.OnGameTimeChanged);
}

public void Dispose(IGameUI entity)
{
    _subscription.Dispose();  // Гарантированная отписка
}
```

**Для нескольких подписок:**
```csharp
private Subscription<float> _timeSubscription;
private Subscription<int> _scoreSubscription;

public void Dispose(IGameUI entity)
{
    _timeSubscription?.Dispose();
    _scoreSubscription?.Dispose();
}
```

---

## Типы Presenters

Atomic Framework поддерживает **7 основных типов Presenters**, каждый для своего сценария использования.

### Type 1: Simple Reactive Presenter

**Назначение:** Отображение одного реактивного значения из Context.

**Когда использовать:**
- Countdown timer
- Score counter
- Single stat display

**Пример (Production-Ready):**

```csharp
using Atomic.Elements;
using ShooterGame.Gameplay;
using TMPro;

namespace ShooterGame.UI
{
    public sealed class CountdownPresenter : IGameUIInit, IGameUIDispose
    {
        private readonly TMP_Text _view;
        private readonly IGameContext _gameContext;

        private Subscription<float> _subscription;

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
}
```

**Ключевые особенности:**
- ✅ Typed interfaces (IGameUIInit, IGameUIDispose)
- ✅ Context injection через конструктор
- ✅ Subscription pattern с автоматическим Dispose
- ✅ Минималистичный - только Init и Dispose

**Файл:** `/Assets/Examples/Shooter/Scripts/UI/Game/Countdown/CountdownPresenter.cs`

---

### Type 2: Dictionary Presenter with Filtering

**Назначение:** Отображение значения из IReactiveDictionary с фильтрацией по ключу.

**Когда использовать:**
- Team statistics
- Per-player scores
- Filtered data display

**Пример:**

```csharp
using Atomic.Elements;
using Atomic.Entities;
using TMPro;

namespace ShooterGame.Gameplay
{
    public sealed class KillsPresenter : IEntityInit, IEntityDispose
    {
        private readonly TMP_Text _text;
        private readonly TeamType _teamType;

        private IReactiveDictionary<TeamType, int> _leaderboard;

        public KillsPresenter(TMP_Text text, TeamType teamType)
        {
            _text = text;
            _teamType = teamType;
        }

        public void Init(IEntity entity)
        {
            _leaderboard = GameContext.Instance.GetLeaderboard();
            _leaderboard.OnItemChanged += this.OnLeaderboardChanged;
            this.UpdateText(_leaderboard[_teamType]);  // Начальное значение
        }

        public void Dispose(IEntity entity)
        {
            _leaderboard.OnItemChanged -= this.OnLeaderboardChanged;
        }

        private void OnLeaderboardChanged(TeamType teamType, int value)
        {
            if (_teamType == teamType)  // Фильтрация по ключу
                this.UpdateText(value);
        }

        private void UpdateText(int value)
        {
            _text.text = $"Kills: {value}";
        }
    }
}
```

**Ключевые особенности:**
- ✅ Фильтрация по ключу в OnItemChanged
- ✅ Начальное значение через indexer
- ✅ Separate update method для переиспользования
- ✅ Constructor injection для фильтра (teamType)

**Файл:** `/Assets/Examples/Shooter/Scripts/UI/Game/Kills/KillsPresenter.cs`

---

### Type 3: Entity Presenter

**Назначение:** Отображение состояния конкретной Entity с подпиской на её события.

**Когда использовать:**
- Health bars
- Entity info panels
- Player/Enemy HUD

**Пример:**

```csharp
using Atomic.Elements;
using Atomic.Entities;

namespace ShooterGame.Gameplay
{
    public sealed class HitPointsPresenter : IEntityInit<IActor>, IEntityEnable, IEntityDisable
    {
        private HitPointsView _view;
        private Health _health;
        private TeamCatalog _teamConfig;
        private IValue<TeamType> _teamType;

        public void Init(IActor entity)
        {
            _view = entity.GetHitPointsView();
            _health = entity.GetHealth();
            _teamType = entity.GetTeamType();
            _teamConfig = GameContext.Instance.GetTeamCatalog();
        }

        public void Enable(IEntity entity)
        {
            _health.OnStateChanged += this.OnHealthChanged;
            _view.Hide();  // Начально скрыт
        }

        public void Disable(IEntity entity)
        {
            _health.OnStateChanged -= this.OnHealthChanged;
        }

        private void OnHealthChanged()
        {
            _view.SetColor(_teamConfig.GetInfo(_teamType.Value).Material.color);
            _view.SetProgress(_health.GetPercent());
            _view.SetText($"{_health.GetCurrent()}/{_health.GetMax()}");
            _view.Show();
        }
    }
}
```

**Ключевые особенности:**
- ✅ Typed Init - IEntityInit<IActor>
- ✅ Enable/Disable для event subscription
- ✅ Получение View из Entity
- ✅ Использование ScriptableObject конфигурации (TeamCatalog)
- ✅ Multiple view updates в одном handler

**Файл:** `/Assets/Examples/Shooter/Scripts/Gameplay/Actors/View/HitPoints/HitPointsPresenter.cs`

---

### Type 4: Composite Presenter

**Назначение:** Управление коллекцией дочерних Presenters.

**Когда использовать:**
- Lists с динамическими элементами
- Grid layouts
- Inventory systems
- Level selection screens

**Пример (Production-Ready):**

```csharp
using ShooterGame.App;

namespace ShooterGame.UI
{
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
            this.SpawnLevelItems();  // Создаём дочерние Presenters
        }

        private void SpawnLevelItems()
        {
            int startLevel = _appContext.GetStartLevel().Value;
            int maxLevel = _appContext.GetMaxLevel().Value;

            for (int i = startLevel; i <= maxLevel; i++)
            {
                // Создаём View через factory method
                LevelItemView itemView = _screenView.CreateItem();

                // Создаём Presenter для элемента
                LevelItemPresenter itemPresenter = new LevelItemPresenter(
                    _appContext,
                    i,
                    itemView
                );

                // Добавляем как Behaviour в UIContext
                _uiContext.AddBehaviour(itemPresenter);
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
            // Удаляем все дочерние Presenters
            _uiContext.DelBehaviours<LevelItemPresenter>();

            // Очищаем View
            _screenView.ClearAllItems();
        }
    }
}
```

**Ключевые особенности:**
- ✅ Typed interfaces (IMenuUIInit, IMenuUIDispose, IMenuUIEnable, IMenuUIDisable)
- ✅ Context injection через конструктор (AppContext + View)
- ✅ Создание дочерних Presenters в Init
- ✅ Добавление в UIContext через AddBehaviour
- ✅ Массовое удаление через DelBehaviours<T>
- ✅ View factory method (CreateItem)
- ✅ UseCase для navigation

**Файл:** `/Assets/Examples/Shooter/Scripts/UI/Menu/Screens/Level/LevelScreenPresenter.cs`

---

### Type 5: Child Presenter

**Назначение:** Отображение одного элемента в списке, управляется Composite Presenter.

**Когда использовать:**
- List items
- Grid items
- Inventory slots
- Level buttons

**Пример (Production-Ready):**

```csharp
using ShooterGame.App;

namespace ShooterGame.UI
{
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
            // Настройка визуального состояния
            int currentLevel = _context.GetCurrentLevel().Value;

            if (currentLevel == _level)
                _view.SetAsCurrent();
            else if (currentLevel > _level)
                _view.SetAsCompleted();
            else
                _view.SetAsNotCompleted();

            // Установка данных
            _view.SetLevel(_level.ToString());

            // Подписка на events
            _view.OnClicked += this.OnClicked;
        }

        public void Dispose(IMenuUI entity)
        {
            _view.OnClicked -= this.OnClicked;
        }

        private void OnClicked() =>
            GameLoadingUseCase.StartGame(_context, _level);
    }
}
```

**Ключевые особенности:**
- ✅ Typed interfaces (IMenuUIInit, IMenuUIDispose)
- ✅ Constructor injection всех зависимостей (Context, level, View)
- ✅ Conditional visual state setup
- ✅ Event subscription в Init, отписка в Dispose
- ✅ UseCase call на button click
- ✅ Не управляет lifecycle View (управляет parent)

**Файл:** `/Assets/Examples/Shooter/Scripts/UI/Menu/Screens/Level/LevelItemPresenter.cs`

---

### Type 6: Screen Presenter

**Назначение:** Управление экраном с множественными кнопками и navigation.

**Когда использовать:**
- Main menu
- Settings screen
- Pause menu
- Game over screen

**Пример (Production-Ready):**

```csharp
using ShooterGame.App;

namespace ShooterGame.UI
{
    public sealed class StartScreenPresenter :
        IMenuUIInit,
        IMenuUIEnable,
        IMenuUIDisable
    {
        private readonly StartScreenView _screenView;
        private readonly IAppContext _appContext;

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
}
```

**Ключевые особенности:**
- ✅ Typed interfaces (IMenuUIInit, IMenuUIEnable, IMenuUIDisable)
- ✅ Context injection через конструктор (не через Singleton)
- ✅ Multiple event subscriptions в Enable/Disable
- ✅ UIContext сохраняется в Init для navigation
- ✅ Прямой вызов UseCases (QuitUseCase.Quit)
- ✅ Type-based navigation (ShowScreen<T>)

**Файл:** `/Assets/Examples/Shooter/Scripts/UI/Menu/Screens/Start/StartScreenPresenter.cs`

---

### Type 7: Popup Presenter

**Назначение:** Управление модальными окнами с данными из Context.

**Когда использовать:**
- Game Over popup
- Victory/Defeat screens
- Confirmation dialogs
- Notification popups

**Пример:**

```csharp
using Atomic.Entities;

namespace BeginnerGame
{
    public sealed class GameOverPresenter :
        IEntityInit<IUIContext>,
        IEntityEnable,
        IEntityDispose
    {
        private IUIContext _context;
        private TeamCatalog _catalog;
        private GameOverView _view;

        public void Init(IUIContext context)
        {
            _catalog = GameContext.Instance.GetTeamCatalog();
            _context = context;
            _view = context.GetGameOverView();

            // Получение данных для отображения
            TeamType teamType = GameContext.Instance.GetWinnerTeam().Value;

            // Настройка View
            _view.SetMessage($"{teamType} PLAYER \nWINS");
            _view.SetMessageColor(_catalog.GetInfo(teamType).Material.color);

            // Подписка на buttons
            _view.OnRestartClicked += RestartUseCase.RestartGame;
            _view.OnCloseClicked += this.OnCloseClicked;
        }

        public void Enable(IEntity entity)
        {
            _view.Show();  // Показываем popup
        }

        public void Dispose(IEntity context)
        {
            _view.OnRestartClicked -= RestartUseCase.RestartGame;
            _view.OnCloseClicked -= this.OnCloseClicked;
        }

        private void OnCloseClicked()
        {
            GameOverUseCase.HidePopup(_context);
        }
    }
}
```

**Ключевые особенности:**
- ✅ Typed Init с UIContext
- ✅ View получается из Context
- ✅ Setup данных в Init
- ✅ Show в Enable
- ✅ ScriptableObject конфигурация
- ✅ UseCase для close action

**Файл:** `/Assets/Examples/Shooter/Scripts/UI2/GameOver/GameOverPresenter.cs` (закомментирован в примере)

---

## Lifecycle Management

Presenters следуют стандартному lifecycle Entity Behaviour с четырьмя фазами:

```
┌─────────┐
│  Init   │  ← Получение зависимостей, подписка на reactive values
└────┬────┘
     │
┌────▼────┐
│ Enable  │  ← Подписка на view events, показ UI
└────┬────┘
     │
┌────▼────┐
│ Disable │  ← Отписка от view events
└────┬────┘
     │
┌────▼────┐
│ Dispose │  ← Отписка от reactive values, cleanup
└─────────┘
```

### Init Phase

**Цель:** Получить все зависимости и подписаться на изменения данных.

**Что делать:**
- ✅ Получить Model (reactive values, entities, context)
- ✅ Подписаться на reactive values через Observe/Subscribe
- ✅ Сохранить ссылки в поля класса
- ✅ Установить начальное состояние View

**Пример:**

```csharp
public void Init(IEntity entity)
{
    // 1. Получить Model
    _gameTime = GameContext.Instance.GetGameTime();
    _score = GameContext.Instance.GetScore();

    // 2. Подписаться на изменения
    _gameTime.Observe(this.OnGameTimeChanged);
    _score.Observe(this.OnScoreChanged);

    // 3. Установить начальное состояние
    this.OnGameTimeChanged(_gameTime.Value);
    this.OnScoreChanged(_score.Value);
}
```

### Enable Phase

**Цель:** Подписаться на UI events и активировать отображение.

**Что делать:**
- ✅ Подписаться на view events (OnClicked, OnValueChanged, etc.)
- ✅ Показать/скрыть UI элементы
- ✅ Запустить анимации

**Пример:**

```csharp
public void Enable(IEntity entity)
{
    // Подписка на view events
    _view.OnRestartClicked += this.OnRestartClicked;
    _view.OnCloseClicked += this.OnCloseClicked;

    // Показать UI
    _view.Show();
}
```

### Disable Phase

**Цель:** Отписаться от UI events.

**Что делать:**
- ✅ Отписаться от view events
- ✅ Скрыть UI (опционально)

**Пример:**

```csharp
public void Disable(IEntity entity)
{
    // Отписка от view events (симметрично Enable)
    _view.OnRestartClicked -= this.OnRestartClicked;
    _view.OnCloseClicked -= this.OnCloseClicked;
}
```

### Dispose Phase

**Цель:** Отписаться от reactive values и освободить ресурсы.

**Что делать:**
- ✅ Отписаться от reactive values через Unsubscribe
- ✅ Очистить коллекции дочерних Presenters
- ✅ Очистить View (для Composite Presenters)

**Пример:**

```csharp
public void Dispose(IEntity entity)
{
    // Отписка от reactive values (симметрично Init)
    _gameTime.Unsubscribe(this.OnGameTimeChanged);
    _score.Unsubscribe(this.OnScoreChanged);

    // Для Composite Presenters
    _uiContext.DelBehaviours<ChildPresenter>();
    _view.ClearAllItems();
}
```

---

## Best Practices

### 1. Симметричность подписок

**Правило:** Каждая подписка должна иметь соответствующую отписку.

**Production-Ready Pattern (рекомендуется):**

```csharp
// ✅ ЛУЧШИЙ ВАРИАНТ - Subscription<T> с автоматической очисткой
public sealed class CountdownPresenter : IGameUIInit, IGameUIDispose
{
    private readonly TMP_Text _view;
    private readonly IGameContext _gameContext;

    private Subscription<float> _subscription;  // Хранит подписку

    public void Init(IGameUI entity)
    {
        _subscription = _gameContext
            .GetGameTime()
            .Observe(this.OnGameTimeChanged);
    }

    public void Dispose(IGameUI entity)
    {
        _subscription.Dispose();  // Автоматическая отписка
    }

    private void OnGameTimeChanged(float time)
    {
        _view.text = $"Game Time: {time:F0}";
    }
}
```

**Legacy Pattern (для простых случаев):**

```csharp
// ✅ ПРАВИЛЬНО - симметрично
public void Init(IEntity entity)
{
    _score.Observe(this.OnScoreChanged);
}

public void Dispose(IEntity entity)
{
    _score.Unsubscribe(this.OnScoreChanged);
}

// ❌ НЕПРАВИЛЬНО - забыли отписаться
public void Init(IEntity entity)
{
    _score.Observe(this.OnScoreChanged);
}

public void Dispose(IEntity entity)
{
    // Забыли Unsubscribe - memory leak!
}
```

**Преимущества Subscription<T>:**
- Невозможно забыть отписаться (хранится как поле)
- Поддержка множественных подписок (Subscription.Compose)
- Более чистый код

### 2. Constructor Injection

**Правило:** View, конфигурация И контексты через конструктор, данные в Init.

**Production-Ready Pattern (рекомендуется):**

```csharp
// ✅ ЛУЧШИЙ ВАРИАНТ - Context Injection вместо Singleton
public sealed class StartScreenPresenter :
    IMenuUIInit,
    IMenuUIEnable,
    IMenuUIDisable
{
    private readonly StartScreenView _screenView;  // View - через конструктор
    private readonly IAppContext _appContext;       // Context - через конструктор

    private IMenuUI _uIContext;  // UI Context - в Init

    public StartScreenPresenter(StartScreenView screenView, IAppContext appContext)
    {
        _screenView = screenView;
        _appContext = appContext;  // Инжектим, не используем Singleton!
    }

    public void Init(IMenuUI context)
    {
        _uIContext = context;
    }

    public void Enable(IMenuUI entity)
    {
        _screenView.OnStartClicked += this.OnStartClicked;
    }

    private void OnStartClicked() =>
        GameLoadingUseCase.StartGame(_appContext);  // Используем инжектированный контекст
}
```

**Legacy Pattern (для простых случаев):**

```csharp
// ✅ ПРАВИЛЬНО - простой вариант с Singleton
public sealed class KillsPresenter : IEntityInit, IEntityDispose
{
    private readonly TMP_Text _text;        // View - через конструктор
    private readonly TeamType _teamType;    // Config - через конструктор

    private IReactiveDictionary<TeamType, int> _leaderboard;  // Data - в Init

    public KillsPresenter(TMP_Text text, TeamType teamType)
    {
        _text = text;
        _teamType = teamType;
    }

    public void Init(IEntity entity)
    {
        _leaderboard = GameContext.Instance.GetLeaderboard();
    }
}

// ❌ НЕПРАВИЛЬНО - все в Init
public sealed class KillsPresenter : IEntityInit, IEntityDispose
{
    private TMP_Text _text;

    public void Init(IEntity entity)
    {
        _text = GameObject.Find("KillsText").GetComponent<TMP_Text>();  // Плохо!
    }
}
```

**Преимущества Context Injection:**
- Лучшая тестируемость (можно подменить mock контекст)
- Явные зависимости (видно что нужно Presenter'у)
- Нет скрытой связности через Singleton

### 3. Separation of Concerns

**Правило:** Бизнес-логика в UseCases, не в Presenters.

```csharp
// ✅ ПРАВИЛЬНО - UseCase для логики
private void OnStartClicked()
{
    LoadGameUseCase.StartGame(_appContext, _level);
}

// ❌ НЕПРАВИЛЬНО - логика в Presenter
private void OnStartClicked()
{
    SceneManager.LoadScene("Game");
    PlayerPrefs.SetInt("Level", _level);
    Time.timeScale = 1f;
}
```

### 4. Type Safety

**Правило:** Используйте типизированные интерфейсы для конкретных контекстов.

**Production-Ready Pattern (рекомендуется):**

```csharp
// ✅ ЛУЧШИЙ ВАРИАНТ - типизированные интерфейсы для каждого контекста
public sealed class CountdownPresenter : IGameUIInit, IGameUIDispose
{
    public void Init(IGameUI entity)  // Явно IGameUI, не IEntity
    {
        // Компилятор гарантирует, что это IGameUI
        _subscription = _gameContext.GetGameTime().Observe(this.OnGameTimeChanged);
    }

    public void Dispose(IGameUI entity)
    {
        _subscription.Dispose();
    }
}

// ✅ Другой контекст - другие интерфейсы
public sealed class StartScreenPresenter : IMenuUIInit, IMenuUIEnable, IMenuUIDisable
{
    public void Init(IMenuUI context)  // Явно IMenuUI
    {
        _uIContext = context;
    }
}
```

**Legacy Pattern (для простых случаев):**

```csharp
// ✅ ПРАВИЛЬНО - типизированный Init через дженерик
public sealed class HitPointsPresenter : IEntityInit<IActor>, IEntityDispose
{
    public void Init(IActor entity)  // Гарантированно IActor
    {
        _health = entity.GetHealth();
    }
}

// ❌ НЕПРАВИЛЬНО - generic Init
public sealed class HitPointsPresenter : IEntityInit, IEntityDispose
{
    public void Init(IEntity entity)
    {
        _health = ((IActor)entity).GetHealth();  // Приведение типа
    }
}
```

**Преимущества Typed Interfaces:**
- Явная типизация на уровне интерфейса
- Нет необходимости в дженериках
- Более чистый и понятный код
- Легче найти все Presenters для конкретного контекста (Find Usages)

### 5. Composite Pattern

**Правило:** Большие UI разбивайте на Composite + Children.

**Production-Ready Pattern:**

```csharp
// ✅ ЛУЧШИЙ ВАРИАНТ - Production-ready Composite
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
            _uiContext.AddBehaviour(itemPresenter);  // Composite создаёт
        }
    }

    public void Dispose(IMenuUI entity)
    {
        _uiContext.DelBehaviours<LevelItemPresenter>();  // Composite удаляет
        _screenView.ClearAllItems();
    }
}

// Child Presenter - управляется Composite'ом
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

**Legacy Pattern:**

```csharp
// ❌ НЕПРАВИЛЬНО - монолитный Presenter
public sealed class LevelScreenPresenter : IEntityInit, IEntityDispose
{
    private List<Button> _buttons = new();

    private void SpawnLevelItems()
    {
        for (int i = 0; i < 10; i++)
        {
            Button btn = Instantiate(_buttonPrefab);
            btn.onClick.AddListener(() => OnButtonClicked(i));
            _buttons.Add(btn);
        }
    }
}
```

**Ключевые моменты:**
- Composite создаёт Child Presenters в `Init()`
- Composite удаляет Child Presenters через `DelBehaviours<T>()` в `Dispose()`
- View также очищает визуальные элементы через `ClearAllItems()`
- Child Presenters получают всё необходимое через конструктор

### 6. Naming Convention

**Правило:** {Feature}Presenter для ясности.

```csharp
// ✅ ПРАВИЛЬНО
ScorePresenter
HealthPresenter
CountdownPresenter
LevelScreenPresenter

// ❌ НЕПРАВИЛЬНО
ScoreController
HealthManager
TimerBehaviour
LevelUI
```

### 7. View Factory Method

**Правило:** Создание View элементов через factory method в View классе.

```csharp
// View с factory method
public sealed class LevelScreenView : MonoBehaviour
{
    [SerializeField] private LevelItemView _itemPrefab;
    [SerializeField] private Transform _itemsContainer;

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

// Presenter использует factory
public sealed class LevelScreenPresenter : IEntityInit<IMenuUI>, IEntityDispose
{
    private void SpawnLevelItems()
    {
        for (int i = 0; i < 10; i++)
        {
            LevelItemView itemView = _screenView.CreateItem();  // Factory method
            var itemPresenter = new LevelItemPresenter(_appContext, i, itemView);
            _uiContext.AddBehaviour(itemPresenter);
        }
    }
}
```

### 8. Initial Value Update

**Правило:** Обновляйте View сразу после подписки на изменения.

```csharp
// ✅ ПРАВИЛЬНО - начальное значение
public void Init(IEntity entity)
{
    _leaderboard = GameContext.Instance.GetLeaderboard();
    _leaderboard.OnItemChanged += this.OnLeaderboardChanged;

    // Установить начальное значение
    this.UpdateText(_leaderboard[_teamType]);
}

// ❌ НЕПРАВИЛЬНО - View пустой до первого изменения
public void Init(IEntity entity)
{
    _leaderboard = GameContext.Instance.GetLeaderboard();
    _leaderboard.OnItemChanged += this.OnLeaderboardChanged;
    // Забыли установить начальное значение
}
```

---

## Примеры из Sample проектов

### Shooter Demo - UI System

**Файлы:**
- `/Assets/Examples/Shooter/Scripts/UI/Game/Countdown/CountdownPresenter.cs`
- `/Assets/Examples/Shooter/Scripts/UI/Game/Kills/KillsPresenter.cs`
- `/Assets/Examples/Shooter/Scripts/Gameplay/Actors/View/HitPoints/HitPointsPresenter.cs`
- `/Assets/Examples/Shooter/Scripts/UI/Menu/Screens/Start/StartScreenPresenter.cs`
- `/Assets/Examples/Shooter/Scripts/UI/Menu/Screens/Level/LevelScreenPresenter.cs`
- `/Assets/Examples/Shooter/Scripts/UI/Menu/Screens/Level/LevelItemPresenter.cs`

**Структура:**

```
Shooter Demo UI
├── GameUI (Context)
│   ├── CountdownPresenter (Simple Reactive)
│   ├── KillsPresenter (Dictionary Filtering) x2 (Blue + Red teams)
│   └── Per-Actor HitPointsPresenter (Entity Presenter)
│
└── MenuUI (Context)
    ├── StartScreenPresenter (Screen)
    └── LevelScreenPresenter (Composite)
        └── LevelItemPresenter x10 (Children)
```

### Integration Example

**Production-Ready Pattern с Context Injection:**

```csharp
public sealed class GameUIInstaller : MonoBehaviour
{
    [SerializeField] private TMP_Text _countdownText;
    [SerializeField] private TMP_Text _blueKillsText;
    [SerializeField] private TMP_Text _redKillsText;

    private IGameContext _gameContext;

    private void Start()
    {
        // Получаем контекст (например, из AppContext или Singleton)
        _gameContext = AppContext.Instance.GetGameContext();

        var gameUI = new Entity("GameUI");

        // Добавляем Presenters с инжекцией контекста
        gameUI.AddBehaviour(new CountdownPresenter(_countdownText, _gameContext));
        gameUI.AddBehaviour(new KillsPresenter(_blueKillsText, _gameContext, TeamType.BLUE));
        gameUI.AddBehaviour(new KillsPresenter(_redKillsText, _gameContext, TeamType.RED));

        gameUI.Init();
        gameUI.Enable();
    }
}
```

**Legacy Pattern (для простых случаев):**

```csharp
public sealed class GameUIInstaller : MonoBehaviour
{
    [SerializeField] private TMP_Text _countdownText;
    [SerializeField] private TMP_Text _blueKillsText;
    [SerializeField] private TMP_Text _redKillsText;

    private void Start()
    {
        var gameUI = new Entity("GameUI");

        // Presenters используют Singleton внутри
        gameUI.AddBehaviour(new CountdownPresenter(_countdownText));
        gameUI.AddBehaviour(new KillsPresenter(_blueKillsText, TeamType.BLUE));
        gameUI.AddBehaviour(new KillsPresenter(_redKillsText, TeamType.RED));

        gameUI.Init();
        gameUI.Enable();
    }
}
```

---

## Сравнение с View Behaviours

| Аспект | Presenter | View Behaviour |
|--------|-----------|----------------|
| **Назначение** | UI элементы (Canvas, HUD) | Визуализация игровых объектов (Transform, Renderer) |
| **Data Source** | Context (GameContext, AppContext) | Entity (конкретный игровой объект) |
| **Добавляется к** | UIContext (Entity для UI) | EntityView (на префабе объекта) |
| **Lifecycle** | Управляется UIContext | Управляется EntityView |
| **Пример** | ScorePresenter, HealthBarPresenter | PositionViewBehaviour, RotationViewBehaviour |
| **View Type** | UI Components (TMP_Text, Button) | Unity Components (Transform, Animator) |
| **Reactivity** | Context reactive values | Entity reactive values |
| **Файлы** | `/Scripts/UI/` | `/Scripts/View/` |

### Когда использовать Presenter vs View Behaviour

```csharp
// ✅ Presenter - для UI на Canvas
public sealed class ScorePresenter : IEntityInit, IEntityDispose
{
    private readonly TMP_Text _scoreText;  // UI на Canvas
    private IReactiveVariable<int> _score;  // Из GameContext
}

// ✅ View Behaviour - для визуализации объекта
public sealed class PositionViewBehaviour : IEntityInit<IUnitEntity>, IEntityDispose
{
    [SerializeField] private Transform _transform;  // 3D Transform
    private IReactiveValue<Vector3> _position;      // Из Entity
}
```

---

## Заключение

**Presenter Pattern** в Atomic Framework обеспечивает:

✅ **Четкое разделение** между UI (View) и данными (Model)
✅ **Reusable паттерны** для разных типов UI (7 типов)
✅ **Type-safe подход** через типизированные интерфейсы (IGameUIInit, IMenuUIInit)
✅ **Управляемый lifecycle** через Entity Behaviour interfaces
✅ **Dependency Injection** - Context injection вместо Singleton
✅ **Testability** - Presenters тестируются отдельно от Unity
✅ **Scalability** - Composite pattern для сложных UI
✅ **Memory Safety** - Subscription<T> для автоматической очистки подписок

### Следующие шаги

1. Изучите примеры из Shooter Demo
2. Создайте простой ScorePresenter для своего проекта
3. Попробуйте Composite Pattern для списков
4. Ознакомьтесь с [feature-decomposition-guide.md](feature-decomposition-guide.md) для интеграции Presenters в общий процесс

---

## 📖 Дополнительные ресурсы

**Связанные гайды:**

1. **Практические примеры:**
   - [`beginner-demo-guide.md`](beginner-demo-guide.md) - простые примеры UI Presenters
   - [`shooter-demo-guide.md`](shooter-demo-guide.md) - полная UI система с 7 типами Presenters

2. **Справочные материалы:**
   - [`atomic-guide-v2.md`](atomic-guide-v2.md) - полное руководство по Atomic Framework
   - [`feature-decomposition-guide.md`](feature-decomposition-guide.md) - универсальный процесс добавления фич
   - [`feature-checklist.md`](feature-checklist.md) - чеклист с шагом для Presenters

3. **Продвинутые темы:**
   - [`rts-demo-guide.md`](rts-demo-guide.md) - масштабирование и производительность

---

**Файл:** `AI/presenter-pattern-guide.md`
**Версия:** Atomic Framework v2
**Дата:** 2025-11-22

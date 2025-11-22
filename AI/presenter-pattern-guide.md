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

## Типы Presenters

Atomic Framework поддерживает **7 основных типов Presenters**, каждый для своего сценария использования.

### Type 1: Simple Reactive Presenter

**Назначение:** Отображение одного реактивного значения из Context.

**Когда использовать:**
- Countdown timer
- Score counter
- Single stat display

**Пример:**

```csharp
using Atomic.Elements;
using Atomic.Entities;
using TMPro;

namespace ShooterGame.Gameplay
{
    public sealed class CountdownPresenter : IEntityInit, IEntityDispose
    {
        private readonly TMP_Text _view;
        private IReactiveVariable<float> _gameTime;

        public CountdownPresenter(TMP_Text view)
        {
            _view = view;
        }

        public void Init(IEntity _)
        {
            _gameTime = GameContext.Instance.GetGameTime();
            _gameTime.Observe(this.OnGameTimeChanged);
        }

        public void Dispose(IEntity _)
        {
            _gameTime.Unsubscribe(this.OnGameTimeChanged);
        }

        private void OnGameTimeChanged(float time)
        {
            _view.text = $"Game Time: {time:F0}";
        }
    }
}
```

**Ключевые особенности:**
- ✅ Минималистичный - только Init и Dispose
- ✅ Constructor injection для View
- ✅ Observe pattern для подписки
- ✅ Symmetric subscribe/unsubscribe

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

**Пример:**

```csharp
using Atomic.Entities;
using ShooterGame.App;

namespace ShooterGame.UI
{
    public sealed class LevelScreenPresenter :
        IEntityInit<IMenuUI>,
        IEntityDispose,
        IEntityEnable,
        IEntityDisable
    {
        private readonly LevelScreenView _screenView;
        private IMenuUI _uiContext;
        private IAppContext _appContext;

        public LevelScreenPresenter(LevelScreenView screenView)
        {
            _screenView = screenView;
        }

        public void Init(IMenuUI context)
        {
            _appContext = AppContext.Instance;
            _uiContext = context;
            this.SpawnLevelItems();  // Создаём дочерние Presenters
        }

        public void Enable(IEntity entity)
        {
            _screenView.OnCloseClicked += this.OnCloseClicked;
        }

        public void Disable(IEntity entity)
        {
            _screenView.OnCloseClicked -= this.OnCloseClicked;
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

        public void Dispose(IEntity entity)
        {
            // Удаляем все дочерние Presenters
            _uiContext.DelBehaviours<LevelItemPresenter>();

            // Очищаем View
            _screenView.ClearAllItems();
        }

        private void OnCloseClicked() =>
            ScreenUseCase.ShowScreen<StartScreenView>(_uiContext);
    }
}
```

**Ключевые особенности:**
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

**Пример:**

```csharp
using Atomic.Entities;
using ShooterGame.App;

namespace ShooterGame.UI
{
    public sealed class LevelItemPresenter : IEntityInit<IMenuUI>, IEntityDispose
    {
        private readonly IAppContext _context;
        private readonly int _level;
        private readonly LevelItemView _view;

        public LevelItemPresenter(IAppContext context, int level, LevelItemView view)
        {
            _context = context;
            _level = level;
            _view = view;
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

        public void Dispose(IEntity entity)
        {
            _view.OnClicked -= this.OnClicked;
        }

        private void OnClicked() =>
            LoadGameUseCase.StartGame(_context, _level);
    }
}
```

**Ключевые особенности:**
- ✅ Constructor injection всех зависимостей
- ✅ Conditional visual state setup
- ✅ Event subscription в Init
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

**Пример:**

```csharp
using Atomic.Entities;
using ShooterGame.App;

namespace ShooterGame.UI
{
    public sealed class StartScreenPresenter :
        IEntityInit<IMenuUI>,
        IEntityEnable,
        IEntityDisable
    {
        private readonly StartScreenView _screenView;

        private IAppContext _appContext;
        private IMenuUI _uIContext;

        public StartScreenPresenter(StartScreenView screenView)
        {
            _screenView = screenView;
        }

        public void Init(IMenuUI context)
        {
            _uIContext = context;
            _appContext = AppContext.Instance;
        }

        public void Enable(IEntity entity)
        {
            _screenView.OnSelectLevelClicked += this.OnSelectLevelClicked;
            _screenView.OnStartClicked += this.OnStartClicked;
            _screenView.OnExitClicked += QuitUseCase.Quit;  // Прямой вызов UseCase
        }

        public void Disable(IEntity entity)
        {
            _screenView.OnStartClicked -= this.OnStartClicked;
            _screenView.OnSelectLevelClicked -= this.OnSelectLevelClicked;
            _screenView.OnExitClicked -= QuitUseCase.Quit;
        }

        private void OnStartClicked() =>
            LoadGameUseCase.StartGame(_appContext);

        private void OnSelectLevelClicked() =>
            ScreenUseCase.ShowScreen<LevelScreenView>(_uIContext);
    }
}
```

**Ключевые особенности:**
- ✅ Multiple event subscriptions
- ✅ Enable/Disable для UI events
- ✅ Init для данных
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

### 2. Constructor Injection

**Правило:** View и конфигурация через конструктор, данные в Init.

```csharp
// ✅ ПРАВИЛЬНО
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

**Правило:** Используйте IEntityInit<TContext> для типобезопасности.

```csharp
// ✅ ПРАВИЛЬНО - типизированный Init
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

### 5. Composite Pattern

**Правило:** Большие UI разбивайте на Composite + Children.

```csharp
// ✅ ПРАВИЛЬНО - Composite управляет Children
public sealed class LevelScreenPresenter : IEntityInit<IMenuUI>, IEntityDispose
{
    private void SpawnLevelItems()
    {
        for (int i = 0; i < 10; i++)
        {
            var itemPresenter = new LevelItemPresenter(...);
            _uiContext.AddBehaviour(itemPresenter);  // Composite создаёт
        }
    }

    public void Dispose(IEntity entity)
    {
        _uiContext.DelBehaviours<LevelItemPresenter>();  // Composite удаляет
    }
}

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

```csharp
public sealed class GameUIInstaller : MonoBehaviour
{
    [SerializeField] private TMP_Text _countdownText;
    [SerializeField] private TMP_Text _blueKillsText;
    [SerializeField] private TMP_Text _redKillsText;

    private void Start()
    {
        var gameUI = new Entity("GameUI");

        // Добавляем Presenters
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
✅ **Reusable паттерны** для разных типов UI
✅ **Type-safe подход** через IEntityInit<TContext>
✅ **Управляемый lifecycle** через Entity Behaviour interfaces
✅ **Testability** - Presenters тестируются отдельно от Unity
✅ **Scalability** - Composite pattern для сложных UI

### Следующие шаги

1. Изучите примеры из Shooter Demo
2. Создайте простой ScorePresenter для своего проекта
3. Попробуйте Composite Pattern для списков
4. Ознакомьтесь с [feature-decomposition-guide.md](feature-decomposition-guide.md) для интеграции Presenters в общий процесс

---

## Дополнительные ресурсы

- [beginner-demo-guide.md](beginner-demo-guide.md) - простые примеры UI
- [shooter-demo-guide.md](shooter-demo-guide.md) - полная UI система
- [feature-decomposition-guide.md](feature-decomposition-guide.md) - общий процесс добавления фич
- [feature-checklist.md](feature-checklist.md) - чеклист с Presenter шагом

---

**Файл:** `AI/presenter-pattern-guide.md`
**Версия:** Atomic Framework v2
**Дата:** 2025-11-22

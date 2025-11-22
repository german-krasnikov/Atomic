# Чек-лист добавления фич в Atomic Framework

## 🎯 Цель документа

Этот чек-лист поможет вам:
1. **Определить уровень сложности** вашей фичи
2. **Выбрать подходящий гайд** и паттерны
3. **Систематически реализовать** фичу шаг за шагом

---

## Шаг 1: Определение сложности фичи

### 📋 Диагностический опросник

Ответьте на следующие вопросы о вашей фиче:

#### Вопрос 1: Что делает фича?
- [ ] A. Хранит простые данные (позиция, здоровье, деньги)
- [ ] B. Обрабатывает ввод или выполняет простую логику (движение, сбор)
- [ ] C. Координирует несколько систем (AI, комбат, спавн)
- [ ] D. Управляет архитектурой (Factory, Pooling, Context)
- [ ] E. Отображает UI на основе данных из Context/Entity (Presenter)

#### Вопрос 2: Сколько других фич требуется?
- [ ] A. 0-1 фича (независима или зависит только от Transform)
- [ ] B. 2-3 фичи (например, Movement требует Transform + Health)
- [ ] C. 4-6 фич (например, AI требует Transform + Movement + Combat + Life)
- [ ] D. 7+ фич (архитектурные паттерны используют всё)

#### Вопрос 3: Требуется ли UseCase?
- [ ] A. Нет, только данные
- [ ] B. Простой UseCase (1-2 метода)
- [ ] C. Сложный UseCase с множеством методов
- [ ] D. Множество UseCases + Burst Compilation

#### Вопрос 4: Нужны ли Behaviours?
- [ ] A. Нет Behaviours (только данные)
- [ ] B. Один простой Behaviour (1-2 метода)
- [ ] C. Несколько Behaviours с подписками на события
- [ ] D. Сложная система Behaviours с зависимостями

#### Вопрос 5: Реактивность
- [ ] A. Не требуется (const данные)
- [ ] B. Простая реактивность (IReactiveVariable для UI)
- [ ] C. События и подписки (IEvent)
- [ ] D. Сложная система событий + Reactive + Request/Response

### 🎓 Определение уровня сложности

**Подсчитайте ответы:**
- **Большинство A** → **Уровень 1** (⭐ Foundation)
- **Большинство B** → **Уровень 2** (⭐⭐ Core Mechanics)
- **Большинство C** → **Уровень 3** (⭐⭐⭐ Advanced Mechanics)
- **Большинство D** → **Уровень 4** (⭐⭐⭐⭐ Architecture)

---

## Шаг 2: Выбор подходящего гайда

### Уровень 1 (⭐): Foundation - Базовые данные

**Характеристики:**
- Простое хранение данных
- Минимум логики
- Не требует UseCases
- Нет или простые Behaviours

**Примеры фич:**
- Transform (позиция, ротация, scale)
- Money/Score (число)
- Health (с событиями)
- Tag (маркировка объекта)

**Подходящие гайды:**
- 📖 `beginner-demo-guide.md` → Feature 1 (Transform), Feature 2 (Money)
- 📖 `shooter-demo-guide.md` → Feature 1 (Transform), Feature 3 (Health)
- 📖 `feature-decomposition-guide.md` → Раздел "Foundation Features"

**Паттерны:**
- ✅ EntityAPI с extension methods
- ✅ Serializable data classes
- ✅ IReactiveVariable для UI
- ✅ Simple Installers

---

### Уровень 2 (⭐⭐): Core Mechanics - Основная механика

**Характеристики:**
- Простая логика
- 1-2 UseCase метода
- Простые Behaviours (1-2 lifecycle)
- Зависит от 2-3 других фич

**Примеры фич:**
- Movement (движение с направлением)
- Input (обработка клавиатуры/мыши)
- Trigger Detection (обнаружение коллизий)
- Countdown Timer (таймер)

**Подходящие гайды:**
- 📖 `beginner-demo-guide.md` → Feature 3 (Input), Feature 4 (Movement), Feature 9 (Timer)
- 📖 `shooter-demo-guide.md` → Feature 4 (Movement)
- 📖 `rts-demo-guide.md` → Feature 2 (Request Pattern Movement)

**Паттерны:**
- ✅ Producer-Consumer (Input пишет, Movement читает)
- ✅ Reactive Direction + Speed
- ✅ UseCase для расчетов
- ✅ Behaviour с FixedTick
- ✅ Conditions (IExpression<bool>)

---

### Уровень 3 (⭐⭐⭐): Advanced Mechanics - Продвинутая механика

**Характеристики:**
- Сложная логика
- Множество UseCases
- Несколько Behaviours с событиями
- Зависит от 4-6 других фич
- События и подписки

**Примеры фич:**
- Weapon System (стрельба с cooldown + ammo)
- Bullet System (с pooling)
- Coin Collection (триггеры + события)
- AI System (detect + attack)
- Combat System (damage + death)

**Подходящие гайды:**
- 📖 `beginner-demo-guide.md` → Feature 5 (Trigger), Feature 7 (Coin Collection), Feature 8 (Spawning)
- 📖 `shooter-demo-guide.md` → Feature 5 (Weapon & Bullet)
- 📖 `rts-demo-guide.md` → Feature 3 (AI System)

**Паттерны:**
- ✅ Event-driven (IEvent с подпиской)
- ✅ Request/Response (Request<T>)
- ✅ Object Pooling (для пуль/объектов)
- ✅ Multiple Behaviours (Init + Enable + Tick + Disable + Dispose)
- ✅ UseCase delegation
- ✅ Cooldown/Lifetime

---

### Уровень 4 (⭐⭐⭐⭐): Architecture - Архитектурные паттерны

**Характеристики:**
- Управление архитектурой
- Множество взаимодействующих систем
- Factory/Builder/Context паттерны
- Burst Compilation
- EntityFilter/EntityWorld

**Примеры фич:**
- Context System (GameContext, PlayerContext)
- Factory Pattern (создание юнитов)
- EntityFilter (динамические выборки)
- Object Pooling + EntityWorld
- Leaderboard System (координация игроков)

**Подходящие гайды:**
- 📖 `shooter-demo-guide.md` → Весь раздел "Трехуровневая иерархия контекстов"
- 📖 `rts-demo-guide.md` → Разделы "Factory Pattern", "EntityFilter", "Object Pooling"
- 📖 `feature-decomposition-guide.md` → Раздел "Architecture Patterns"

**Паттерны:**
- ✅ Factory Pattern (создание объектов)
- ✅ Builder Pattern (fluent API)
- ✅ EntityFilter (динамические выборки)
- ✅ Context Hierarchy (App → Game → Player)
- ✅ Burst Compilation (производительность)
- ✅ Modular Installers (переиспользование)

---

## Шаг 3: Пошаговая реализация фичи

### ✅ Универсальный чек-лист (8 шагов)

Независимо от уровня сложности, следуйте этим 8 шагам (шаг 6 только для фич с UI):

#### ☐ Шаг 1: Добавление в EntityAPI

**Действия:**
1. [ ] Определите, какие данные нужны (values, tags, events)
2. [ ] Добавьте `readonly int` поля для ключей в EntityAPI
3. [ ] Создайте extension methods: Get, Add, Has, Del, Set
4. [ ] Пометьте типы в комментариях (например, `// IReactiveVariable<int>`)

**Проверка:**
- [ ] Все методы используют `[MethodImpl(MethodImplOptions.AggressiveInlining)]`
- [ ] Типы указаны в комментариях для ясности
- [ ] Extension methods следуют конвенции именования

**Пример:**
```csharp
public static class EntityAPI
{
    public static readonly int Money;  // IReactiveVariable<int>

    [MethodImpl(MethodImplOptions.AggressiveInlining)]
    public static IReactiveVariable<int> GetMoney(this IEntity entity)
        => entity.GetValue<IReactiveVariable<int>>(Money);
}
```

---

#### ☐ Шаг 2: Создание Data Classes

**Действия:**
1. [ ] Определите, какие data classes нужны (struct, class, enum)
2. [ ] Добавьте `[Serializable]` для всех data classes
3. [ ] Реализуйте события (если нужна реактивность)
4. [ ] Добавьте валидацию в setters

**Проверка:**
- [ ] Классы помечены `[Serializable]`
- [ ] Поля приватны с `[SerializeField]`
- [ ] События для важных изменений
- [ ] Валидация входных данных

**Выбор типа:**
- **IReactiveVariable<T>** → для UI (автообновление)
- **IVariable<T>** → для изменяемых данных
- **IValue<T>** → для read-only данных
- **Const<T>** → для константных данных
- **IEvent** / **IEvent<T>** → для событий
- **Request<T>** → для однократных команд

---

#### ☐ Шаг 3: Создание UseCases

**Действия для Уровня 1 (Foundation):**
- [ ] ⚠️ Обычно не требуются UseCases
- [ ] Переходите к Шагу 4

**Действия для Уровня 2-4:**
1. [ ] Создайте static класс `[FeatureName]UseCase`
2. [ ] Добавьте статические методы для бизнес-логики
3. [ ] Принимайте entity/context как параметры (без состояния!)
4. [ ] Для критичных операций добавьте Burst Compilation

**Проверка:**
- [ ] Все методы статические
- [ ] Нет состояния (stateless)
- [ ] Параметры передаются явно
- [ ] Burst методы используют `float3`, `quaternion`

**Пример:**
```csharp
public static class MovementUseCase
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

---

#### ☐ Шаг 4: Создание Behaviours

**Действия для Уровня 1:**
- [ ] ⚠️ Обычно не требуются Behaviours
- [ ] Переходите к Шагу 5

**Действия для Уровня 2-4:**
1. [ ] Определите, какие lifecycle методы нужны:
   - [ ] `IEntityInit` - кэширование зависимостей
   - [ ] `IEntityEnable` - подписка на события
   - [ ] `IEntityTick` / `IEntityFixedTick` / `IEntityLateTick` - логика каждый кадр
   - [ ] `IEntityDisable` - отписка от событий
   - [ ] `IEntityDispose` - финальная очистка

2. [ ] Создайте класс Behaviour
3. [ ] Кэшируйте зависимости в `Init`
4. [ ] Делегируйте логику в UseCases
5. [ ] ВСЕГДА отписывайтесь от событий в `Disable`/`Dispose`

**Проверка:**
- [ ] Зависимости кэшированы в `Init`
- [ ] События отписаны в `Disable`/`Dispose`
- [ ] Логика делегирована в UseCases
- [ ] Проверка условий перед выполнением

**Пример:**
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
        _triggerEvents.OnEntered += this.OnTriggerEntered;  // Подписка
    }

    public void Disable(IEntity entity)
    {
        _triggerEvents.OnEntered -= this.OnTriggerEntered;  // Отписка
    }

    private void OnTriggerEntered(Collider collider)
    {
        // Логика сбора
    }
}
```

**Альтернатива: Inline Behaviours (для простой логики):**
```csharp
entity.WhenFixedTick(deltaTime =>
{
    if (entity.GetMoveRequest().Consume(out Vector3 direction))
    {
        MoveUseCase.MoveStep(entity, direction, deltaTime);
    }
});
```

---

#### ☐ Шаг 5: Создание Installer

**Действия:**
1. [ ] Создайте класс Installer (`[Serializable]` или `SceneEntityInstaller`)
2. [ ] Добавьте `[SerializeField]` поля для конфигурации
3. [ ] В методе `Install`:
   - [ ] Добавьте данные (AddValue, AddTag)
   - [ ] Добавьте Behaviours (AddBehaviour)
   - [ ] Установите зависимости между компонентами

**Проверка:**
- [ ] Все зависимости установлены в правильном порядке
- [ ] Конфигурационные значения сериализованы
- [ ] Installer переиспользуемый (modular)

**Пример (Simple):**
```csharp
[Serializable]
public sealed class MoneyInstaller : IEntityInstaller<IEntity>
{
    [SerializeField] private int _initialMoney = 0;

    public void Install(IEntity entity)
    {
        // Reactive для автообновления UI
        entity.AddMoney(new ReactiveInt(_initialMoney));
    }
}
```

**Пример (Complex):**
```csharp
public sealed class CharacterInstaller : SceneEntityInstaller<IActor>
{
    [SerializeField] private Health _health = new Health(3);
    [SerializeField] private Const<float> _moveSpeed = 3;
    [SerializeField] private Weapon _initialWeapon;

    public override void Install(IActor entity)
    {
        // Life
        entity.AddDamageableTag();
        entity.AddHealth(_health);
        entity.AddTakeDamageEvent(new Event<DamageArgs>());

        // Movement
        entity.AddMovementSpeed(_moveSpeed);
        entity.AddMovementDirection(new ReactiveVector3());
        entity.AddMovementCondition(new AndExpression(_health.Exists));
        entity.AddBehaviour<KinematicMovementBehaviour>();

        // Combat
        entity.AddFireCondition(new AndExpression(_health.Exists));
        entity.AddFireAction(new CharacterFireAction(entity));
        entity.AddWeapon(_initialWeapon);
    }
}
```

---

#### ☐ Шаг 6: Создание Presenters (если фича имеет UI)

**Когда создавать Presenters:**
- [ ] Фича имеет UI элементы на Canvas (HUD, меню, попапы)
- [ ] Нужно отображать данные из Context или Entity
- [ ] UI реагирует на изменения реактивных значений
- [ ] Нужна обработка UI events (button clicks)

**Когда НЕ создавать Presenters:**
- ❌ Визуализация игровых объектов в сцене → EntityView Behaviours
- ❌ Бизнес-логика → UseCases
- ❌ Хранение состояния → Entity/Context

**Действия:**
1. [ ] Определить тип Presenter:
   - [ ] **Simple Reactive** - одно реактивное значение (Score, Time)
   - [ ] **Dictionary** - IReactiveDictionary с фильтрацией (Leaderboard)
   - [ ] **Entity Presenter** - состояние Entity (Health Bar)
   - [ ] **Composite** - коллекция дочерних Presenters (Level List)
   - [ ] **Screen** - экран с множественными кнопками (Main Menu)

2. [ ] Создать класс Presenter
3. [ ] Constructor injection для View
4. [ ] Реализовать lifecycle:
   - [ ] `IEntityInit<TContext>` - получить данные, подписаться на reactive
   - [ ] `IEntityEnable` - подписаться на view events
   - [ ] `IEntityDisable` - отписаться от view events
   - [ ] `IEntityDispose` - отписаться от reactive values

5. [ ] Установить начальное значение View после подписки
6. [ ] Использовать UseCases для бизнес-логики
7. [ ] Для Composite: DelBehaviours<T> в Dispose

**Проверка:**
- [ ] Симметричная подписка/отписка (Init↔Dispose, Enable↔Disable)
- [ ] Constructor injection для View
- [ ] Начальное значение View установлено
- [ ] Используются UseCases для логики
- [ ] Memory leaks предотвращены (все отписаны)

**Пример Simple Reactive Presenter:**
```csharp
public sealed class ScorePresenter : IEntityInit, IEntityDispose
{
    private readonly TMP_Text _text;  // View - через конструктор
    private IReactiveVariable<int> _score;  // Model - в Init

    public ScorePresenter(TMP_Text text)
    {
        _text = text;  // Constructor injection
    }

    public void Init(IEntity entity)
    {
        _score = GameContext.Instance.GetScore();
        _score.Observe(this.OnScoreChanged);  // Подписка
        this.OnScoreChanged(_score.Value);    // Начальное значение
    }

    public void Dispose(IEntity entity)
    {
        _score.Unsubscribe(this.OnScoreChanged);  // Отписка
    }

    private void OnScoreChanged(int score)
    {
        _text.text = $"Score: {score}";  // Обновление View
    }
}
```

**Пример Composite Presenter:**
```csharp
public sealed class LevelScreenPresenter :
    IEntityInit<IMenuUI>,
    IEntityDispose,
    IEntityEnable,
    IEntityDisable
{
    private readonly LevelScreenView _screenView;
    private IMenuUI _uiContext;

    public LevelScreenPresenter(LevelScreenView screenView)
    {
        _screenView = screenView;
    }

    public void Init(IMenuUI context)
    {
        _uiContext = context;

        // Создаём дочерние Presenters
        for (int i = 1; i <= 10; i++)
        {
            LevelItemView itemView = _screenView.CreateItem();
            LevelItemPresenter itemPresenter = new LevelItemPresenter(i, itemView);
            _uiContext.AddBehaviour(itemPresenter);  // Добавляем в UIContext
        }
    }

    public void Enable(IEntity entity)
    {
        _screenView.OnCloseClicked += this.OnCloseClicked;
    }

    public void Disable(IEntity entity)
    {
        _screenView.OnCloseClicked -= this.OnCloseClicked;
    }

    public void Dispose(IEntity entity)
    {
        _uiContext.DelBehaviours<LevelItemPresenter>();  // Удаляем всех дочерних
        _screenView.ClearAllItems();
    }

    private void OnCloseClicked() =>
        ScreenUseCase.ShowScreen<StartScreenView>(_uiContext);
}
```

**Интеграция с UIContext:**
```csharp
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

**Подробнее:**
См. [presenter-pattern-guide.md](presenter-pattern-guide.md) для полного гайда по всем 7 типам Presenters.

---

#### ☐ Шаг 7: Интеграция с другими фичами

**Действия:**
1. [ ] Определите, какие фичи зависят от вашей фичи
2. [ ] Определите, от каких фич зависит ваша фича
3. [ ] Проверьте паттерны интеграции:
   - [ ] **Producer-Consumer** (Input → Movement)
   - [ ] **Event-driven** (Health → UI, Death → Respawn)
   - [ ] **Request/Response** (AI → Movement/Combat)

**Проверка:**
- [ ] Зависимости правильно установлены
- [ ] События корректно подписаны/отписаны
- [ ] Request/Response правильно потребляются

**Примеры паттернов:**

**Producer-Consumer:**
```csharp
// Producer (Input System)
character.GetMovementDirection().Value = inputDirection;

// Consumer (Movement System)
Vector3 direction = entity.GetMovementDirection().Value;
MoveUseCase.MoveStep(entity, direction, deltaTime);
```

**Event-driven:**
```csharp
// Publisher (Health System)
_health.OnHealthEmpty += this.OnDeath;

// Subscriber (Respawn System)
private void OnDeath()
{
    CharacterUseCase.RespawnWithDelay(_playerContext, _gameContext);
}
```

**Request/Response:**
```csharp
// Producer (AI)
entity.GetMoveRequest().Invoke(direction);

// Consumer (Movement)
if (entity.GetMoveRequest().Consume(out Vector3 dir))
{
    MoveUseCase.MoveStep(entity, dir, deltaTime);
}
```

---

#### ☐ Шаг 8: Best Practices и документация

**Действия:**
1. [ ] Задокументируйте зависимости фичи
2. [ ] Укажите типичные ошибки и их решения
3. [ ] Добавьте примеры использования
4. [ ] Укажите уровень сложности (⭐)

**Контрольный список Best Practices:**

**Lifecycle:**
- [ ] ✅ `Init` для кэширования зависимостей
- [ ] ✅ `Enable` для подписки на события
- [ ] ✅ `Tick`/`FixedTick`/`LateTick` для логики
- [ ] ✅ `Disable` для отписки от событий
- [ ] ✅ `Dispose` для финальной очистки

**Производительность:**
- [ ] ✅ Используйте `FixedTick` для физики
- [ ] ✅ Используйте `sqrMagnitude` вместо `magnitude`
- [ ] ✅ Кэшируйте зависимости в `Init`
- [ ] ✅ `[BurstCompile]` для критичных операций
- [ ] ✅ `[MethodImpl(AggressiveInlining)]` для API

**Type Safety:**
- [ ] ✅ Используйте extension methods для доступа
- [ ] ✅ Типы указаны в комментариях EntityAPI
- [ ] ✅ Валидация входных данных

**Reactive Programming:**
- [ ] ✅ `IReactiveVariable` для UI
- [ ] ✅ `IEvent` для множественных подписчиков
- [ ] ✅ `Request<T>` для однократных команд
- [ ] ✅ ВСЕГДА отписывайтесь от событий

**Документация:**
```markdown
## Feature: [Название]

**Сложность:** [⭐/⭐⭐/⭐⭐⭐/⭐⭐⭐⭐]
**Зависимости:** [Transform, Health, ...]
**Используется в:** [Movement, AI, ...]

### Описание
[Краткое описание фичи]

### Зависимости
- 📦 [Фича 1]
- 📦 [Фича 2]

### Типичные ошибки
❌ **Ошибка:** [описание]
✅ **Исправление:** [решение]
```

---

## Шаг 4: Примеры реализации по уровням сложности

### Пример 1: Money System (⭐ Foundation)

**Диагностика:**
- Хранит простые данные ✓
- 0 зависимостей ✓
- Не требует UseCases ✓
- Нет Behaviours ✓
- **Результат: Уровень 1**

**Реализация:**

```csharp
// Шаг 1: EntityAPI
public static class EntityAPI
{
    public static readonly int Money;  // IReactiveVariable<int>

    public static IReactiveVariable<int> GetMoney(this IEntity entity)
        => entity.GetValue<IReactiveVariable<int>>(Money);

    public static void AddMoney(this IEntity entity, IReactiveVariable<int> value)
        => entity.AddValue(Money, value);
}

// Шаг 2: Data Classes (используем готовую ReactiveInt)
// Нет необходимости создавать - используем ReactiveInt из фреймворка

// Шаг 3: UseCases
// Не требуются для простого хранения данных

// Шаг 4: Behaviours
// Не требуются

// Шаг 5: Installer
[Serializable]
public sealed class MoneyInstaller : IEntityInstaller<IEntity>
{
    [SerializeField] private int _initialMoney = 0;

    public void Install(IEntity entity)
    {
        entity.AddMoney(new ReactiveInt(_initialMoney));
    }
}

// Шаг 6: Интеграция (с UI)
public sealed class MoneyView : MonoBehaviour
{
    [SerializeField] private Entity _player;
    [SerializeField] private Text _moneyText;
    private Subscription<int> _subscription;

    private void Start()
    {
        IReactiveVariable<int> money = _player.GetMoney();
        _subscription = money.Observe(this.OnMoneyChanged);
    }

    private void OnDestroy()
    {
        _subscription?.Dispose();
    }

    private void OnMoneyChanged(int value)
    {
        _moneyText.text = $"Money: {value}";
    }
}
```

**Гайд:** `beginner-demo-guide.md` → Feature 2

---

### Пример 2: Movement System (⭐⭐ Core Mechanics)

**Диагностика:**
- Простая логика ✓
- 2 зависимости (Transform, Health) ✓
- Простой UseCase ✓
- Один Behaviour ✓
- **Результат: Уровень 2**

**Реализация:**

```csharp
// Шаг 1: EntityAPI
public static readonly int MovementSpeed;      // IValue<float>
public static readonly int MovementDirection;  // IReactiveVariable<Vector3>

// Шаг 2: Data Classes
// Используем ReactiveVector3 из фреймворка

// Шаг 3: UseCases
public static class MovementUseCase
{
    public static Vector3 CalculateVelocity(Vector3 direction, float speed)
    {
        if (direction.sqrMagnitude < 0.001f)
            return Vector3.zero;
        return direction.normalized * speed;
    }
}

// Шаг 4: Behaviours
public sealed class KinematicMovementBehaviour : IEntityInit, IEntityFixedTick
{
    private IValue<float> _speed;
    private IReactiveValue<Vector3> _direction;
    private IVariable<Vector3> _position;

    public void Init(IEntity entity)
    {
        _speed = entity.GetMovementSpeed();
        _direction = entity.GetMovementDirection();
        _position = entity.GetPosition();
    }

    public void FixedTick(IEntity entity, float deltaTime)
    {
        Vector3 direction = _direction.Value;
        if (direction.sqrMagnitude < 0.001f)
            return;

        Vector3 velocity = MovementUseCase.CalculateVelocity(direction, _speed.Value);
        _position.Value += velocity * deltaTime;
    }
}

// Шаг 5: Installer
[Serializable]
public sealed class MovementInstaller : IEntityInstaller<IEntity>
{
    [SerializeField] private Const<float> _moveSpeed = 3;

    public void Install(IEntity entity)
    {
        entity.AddMovementSpeed(_moveSpeed);
        entity.AddMovementDirection(new ReactiveVector3());
        entity.AddBehaviour<KinematicMovementBehaviour>();
    }
}

// Шаг 6: Интеграция (с Input)
// Input System пишет направление:
character.GetMovementDirection().Value = inputDirection;

// Movement System читает и применяет:
Vector3 direction = _direction.Value;
_position.Value += velocity * deltaTime;
```

**Гайд:** `beginner-demo-guide.md` → Feature 4 или `shooter-demo-guide.md` → Feature 4

---

### Пример 3: AI System (⭐⭐⭐ Advanced Mechanics)

**Диагностика:**
- Сложная логика ✓
- 5+ зависимостей ✓
- Множество UseCases ✓
- Несколько Behaviours ✓
- События и Request/Response ✓
- **Результат: Уровень 3**

**Реализация:**

```csharp
// Шаг 1: EntityAPI
public static readonly int Target;    // IVariable<IUnitEntity>
public static readonly int Targeted;  // Tag

// Шаг 2: Data Classes
public sealed class RandomCooldown : ICooldown
{
    private readonly float _min, _max;
    private float _duration, _currentTime;

    public RandomCooldown(float min, float max)
    {
        _min = min;
        _max = max;
        _duration = Random.Range(min, max);
    }

    public bool IsCompleted() => _currentTime >= _duration;
    public void ResetTime() => _duration = Random.Range(_min, _max);
    public void Tick(float deltaTime) => _currentTime += deltaTime;
}

// Шаг 3: UseCases
public static class UnitsUseCase
{
    public static IUnitEntity FindFreeEnemyFor(IGameContext context, IUnitEntity entity)
    {
        IPlayerContext playerContext = PlayersUseCase.GetPlayerFor(context, entity);
        EntityFilter<IUnitEntity> enemyFilter = playerContext.GetFreeEnemyFilter();
        return FindClosest(enemyFilter, entity.GetPosition().Value);
    }

    public static IUnitEntity FindClosest(EntityFilter<IUnitEntity> entities, Vector3 center)
    {
        IUnitEntity result = null;
        float minDistance = float.MaxValue;

        foreach (IUnitEntity entity in entities)
        {
            float distance = Vector3.SqrMagnitude(entity.GetPosition().Value - center);
            if (distance < minDistance)
            {
                result = entity;
                minDistance = distance;
            }
        }
        return result;
    }
}

// Шаг 4: Behaviours (2 компонента)
public sealed class AIDetectTargetBehaviour : IEntityInit, IEntityFixedTick, IEntityDisable
{
    private readonly IGameContext _gameContext;
    private readonly ICooldown _cooldown;
    private IVariable<IUnitEntity> _target;
    private IUnitEntity _entity;

    public void Init(IUnitEntity entity)
    {
        _entity = entity;
        _target = entity.GetTarget();
    }

    public void FixedTick(IEntity entity, float deltaTime)
    {
        _cooldown.Tick(deltaTime);

        if (_cooldown.IsCompleted())
        {
            IUnitEntity enemy = UnitsUseCase.FindFreeEnemyFor(_gameContext, _entity);
            if (enemy != null)
                enemy.AddTargetedTag();  // Помечаем как занятого
            _target.Value = enemy;
            _cooldown.ResetTime();
        }
    }

    public void Disable(IEntity entity)
    {
        if (_target.Value != null)
            _target.Value.DelTargetedTag();  // Снимаем метку
    }
}

public sealed class AIAttackTargetBehaviour : IEntityInit, IEntityFixedTick
{
    private IUnitEntity _entity;
    private IValue<IUnitEntity> _target;

    public void Init(IUnitEntity entity)
    {
        _entity = entity;
        _target = entity.GetTarget();
    }

    public void FixedTick(IEntity entity, float deltaTime)
    {
        IUnitEntity target = _target.Value;
        if (target == null || !LifeUseCase.IsAlive(target))
            return;

        Vector3 vector = TransformUseCase.GetVector(_entity, target);
        float distance = vector.magnitude;

        if (distance > _fireDistance)
            _entity.GetMoveRequest().Invoke(vector.normalized);  // Двигаться
        else
            _entity.GetFireRequest().Invoke(target);  // Стрелять
    }
}

// Шаг 5: Installer
[Serializable]
public sealed class AIEntityInstaller : IEntityInstaller<IUnitEntity>
{
    [SerializeField] private float _minDetectDuration = 0.2f;
    [SerializeField] private float _maxDetectDuration = 0.3f;

    public void Install(IUnitEntity entity)
    {
        IGameContext gameContext = GameContext.Instance;

        entity.AddTarget(new ReactiveVariable<IUnitEntity>());

        // Два behaviour для AI
        entity.AddBehaviour(new AIDetectTargetBehaviour(
            new RandomCooldown(_minDetectDuration, _maxDetectDuration),
            gameContext
        ));
        entity.AddBehaviour<AIAttackTargetBehaviour>();
    }
}

// Шаг 6: Интеграция
// AI вызывает Requests:
_entity.GetMoveRequest().Invoke(direction);   // → Movement System
_entity.GetFireRequest().Invoke(target);      // → Combat System

// Movement/Combat потребляют Requests:
if (entity.GetMoveRequest().Consume(out Vector3 dir))
    MoveUseCase.MoveStep(entity, dir, deltaTime);
```

**Гайд:** `rts-demo-guide.md` → Feature 3 (AI System)

---

### Пример 4: Factory Pattern (⭐⭐⭐⭐ Architecture)

**Диагностика:**
- Управляет архитектурой ✓
- Использует все системы ✓
- Factory/Builder паттерны ✓
- Modular Installers ✓
- **Результат: Уровень 4**

**Реализация:**

```csharp
// Шаг 1: EntityAPI (для Factory не требуется отдельный API)

// Шаг 2: Data Classes
// Используются Installers как конфигурация

// Шаг 3: UseCases
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
        IUnitEntity entity = pool.Rent(name);  // Берем из пула

        entity.GetPosition().Value = position;
        entity.GetRotation().Value = rotation;
        entity.GetTeam().Value = team;

        context.GetEntityWorld().Add(entity);
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

// Шаг 4: Behaviours (не требуются для Factory)

// Шаг 5: Создание Abstract Factory
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

// Concrete Factory (Warrior)
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

        // Композиция через Modular Installers
        entity.Install(_transformInstaller);
        entity.Install(_moveInstaller);
        entity.Install(_lifeInstaller);
        entity.Install(_meleeCombatInstaller);
        entity.Install(_aiInstaller);
    }
}

// Catalog для управления фабриками
[CreateAssetMenu]
public sealed class UnitCatalog : ScriptableMultiEntityFactory<string, IUnitEntity, UnitFactory>
{
    protected override string GetKey(UnitFactory factory) => factory.Name;
}

// Шаг 6: Интеграция (с Pooling)
// В GameContextInstaller
context.AddEntityPool(new MultiEntityPool<string, IUnitEntity>(factoryCatalog));

// Использование
IUnitEntity warrior = UnitsUseCase.Spawn(context, "Warrior", pos, rot, TeamType.BLUE);
```

**Гайд:** `rts-demo-guide.md` → Разделы "Factory Pattern", "UnitCatalog", "Modular Installers"

---

## 📚 Быстрая навигация по гайдам

### По типу проекта

**Простая игра (2D platformer, runner):**
- 📖 `beginner-demo-guide.md` - полный гайд
- 📖 `feature-decomposition-guide.md` - универсальный справочник

**Шутер/экшен (3D shooter, top-down):**
- 📖 `shooter-demo-guide.md` - иерархия контекстов, weapon system
- 📖 `feature-decomposition-guide.md` - паттерны

**Стратегия/симуляция (RTS, tower defense):**
- 📖 `rts-demo-guide.md` - AI, Factory, Pooling, Burst
- 📖 `feature-decomposition-guide.md` - архитектурные паттерны

### По паттерну

| Паттерн | Гайд |
|---------|------|
| Producer-Consumer | `beginner-demo-guide.md` (Input → Movement) |
| Event-driven | `beginner-demo-guide.md` (Coin Collection) |
| Reactive UI | `beginner-demo-guide.md` (Money View) |
| Request/Response | `rts-demo-guide.md` (AI → Movement) |
| Object Pooling | `shooter-demo-guide.md` (Bullet), `rts-demo-guide.md` (Units) |
| Factory Pattern | `rts-demo-guide.md` (Unit Factory) |
| Context Hierarchy | `shooter-demo-guide.md` (App → Game → Player) |
| Burst Compilation | `rts-demo-guide.md` (MoveUseCase) |
| EntityFilter | `rts-demo-guide.md` (Free Enemy Filter) |
| Model-View Separation | `rts-demo-guide.md` (Reactive Transform) |

---

## 🎯 Итоговый алгоритм

```
1. Определите сложность фичи (Шаг 1)
   ↓
2. Выберите подходящий гайд (Шаг 2)
   ↓
3. Следуйте 7 шагам реализации (Шаг 3)
   ↓
4. Проверьте все чек-листы
   ↓
5. Протестируйте интеграцию
   ↓
6. Задокументируйте (Шаг 7)
```

---

## 💡 Советы

1. **Начните с простых фич** - сначала Foundation (⭐), затем переходите к сложным
2. **Изучите примеры** - каждый уровень сложности имеет готовые примеры
3. **Используйте чек-листы** - не пропускайте шаги
4. **Документируйте зависимости** - это поможет при рефакторинге
5. **Тестируйте интеграцию** - проверьте взаимодействие с другими фичами

---

## 📖 Дополнительные ресурсы

- `beginner-demo-guide.md` - для фич уровня ⭐ и ⭐⭐
- `shooter-demo-guide.md` - для фич уровня ⭐⭐ и ⭐⭐⭐
- `rts-demo-guide.md` - для фич уровня ⭐⭐⭐ и ⭐⭐⭐⭐
- `feature-decomposition-guide.md` - универсальный справочник паттернов
- `atomic-guide-v1.md` - основы фреймворка
- `atomic-guide-v2.md` - продвинутые концепции

---

**Удачи в реализации фич! 🚀**

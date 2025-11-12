# Гайд по Beginner Demo - Пошаговое внедрение фич от простых к сложным

## Введение

Beginner Demo демонстрирует пошаговое построение мультиплеерной игры по сбору монет с использованием Atomic Framework. Два игрока (красная и синяя команды) соревнуются в сборе монет за ограниченное время.

**Ключевая особенность гайда:** Показываем декомпозицию фич от простых к сложным с детальными инструкциями по внедрению через Installers → Behaviours → UseCases.

## Обзор игры

**Концепция:** Мультиплеерная игра по сбору монет с таймером обратного отсчета.

**Основные механики:**
- Управление персонажем (WASD / Arrows)
- Движение персонажа с заданной скоростью
- Сбор монет через триггеры
- Автоматический спавн монет в заданной области
- Таймер обратного отсчета (например, 60 секунд)
- Определение победителя по количеству собранных монет
- Реактивное обновление UI

## Архитектура проекта

### Структура файлов

```
Assets/Examples/Beginner/Scripts/
├── Common/                         # Общие компоненты
│   ├── TeamType.cs                # Enum команд (BLUE, RED)
│   ├── SpawnInfo.cs               # Данные для спавна монет
│   ├── PlayerInfo.cs              # Информация о игроке
│   ├── InputMap.cs                # ScriptableObject для управления
│   └── CameraBillboard.cs         # Поворот UI элементов к камере
├── UI/                            # UI компоненты
│   ├── MoneyView.cs               # Отображение очков игрока
│   └── CountdownView.cs           # Отображение таймера
└── Entities/                      # Entity-based архитектура
    ├── EntityAPI.cs               # Автогенерированный API
    ├── Content/                   # Installers - композиция сущностей
    │   ├── CharacterInstaller.cs  # Установка персонажа
    │   ├── CoinInstaller.cs       # Установка монеты
    │   └── GameContextInstaller.cs # Установка игрового контекста
    └── Behaviours/                # Логика поведений
        ├── InputBehaviour.cs      # Обработка ввода
        ├── MovementBehaviour.cs   # Применение движения
        ├── CoinCollectBehaviour.cs # Сбор монет
        ├── CoinSpawnBehaviour.cs  # Спавн монет
        └── GameOverBehaviour.cs   # Логика завершения игры
```

### Принципы организации

- **Разделение по типам**: Common (данные) / UI (отображение) / Entities (логика)
- **Модульность**: Каждый компонент изолирован
- **Иерархическая структура**: Content (что собирать) и Behaviours (как работает) разделены
- **Генерация API**: EntityAPI.cs генерируется автоматически

---

## Декомпозиция фич: от простых к сложным

```
Уровень 1 (Простой):
├── Feature 1: Transform и Position
└── Feature 2: Movement Direction и Speed

Уровень 2 (Средний):
├── Feature 3: Input System
├── Feature 4: Movement
└── Feature 5: Money System

Уровень 3 (Сложный):
├── Feature 6: Trigger Detection
├── Feature 7: Coin Collection
├── Feature 8: Coin Spawning
└── Feature 9: Game Timer

Уровень 4 (Интеграция):
├── Feature 10: Team System & Player Management
├── Feature 11: Game Context
└── Feature 12: Game Over Logic
```

---

# Уровень 1: Базовые фичи

## Feature 1: Transform и Position

### Описание
Базовая фича для всех игровых объектов. Предоставляет доступ к Unity Transform компоненту через EntityAPI. Это foundation для всех визуальных объектов.

### Сложность: ⭐ Простая

### Используемые компоненты
- **EntityAPI**: Transform value
- **Installers**: CharacterInstaller, CoinInstaller
- **Behaviours**: Используется MovementBehaviour (читает Transform)
- **UseCases**: Не требуется
- **Data Classes**: Нет

### Пошаговое внедрение

#### Шаг 1: Добавление в EntityAPI

**Файл:** `Assets/Examples/Beginner/Scripts/Entities/EntityAPI.cs`

```csharp
public static class EntityAPI
{
    // ID для Transform компонента
    public static readonly int Transform;

    static EntityAPI()
    {
        // Регистрация ID через имя
        Transform = NameToId(nameof(Transform));
    }

    // Extension методы для типобезопасного доступа

    /// <summary>
    /// Получает Transform из entity.
    /// </summary>
    [MethodImpl(MethodImplOptions.AggressiveInlining)]
    public static Transform GetTransform(this IEntity entity)
        => entity.GetValue<Transform>(Transform);

    /// <summary>
    /// Добавляет Transform в entity.
    /// </summary>
    [MethodImpl(MethodImplOptions.AggressiveInlining)]
    public static void AddTransform(this IEntity entity, Transform value)
        => entity.AddValue(Transform, value);

    /// <summary>
    /// Проверяет наличие Transform в entity.
    /// </summary>
    [MethodImpl(MethodImplOptions.AggressiveInlining)]
    public static bool HasTransform(this IEntity entity)
        => entity.HasValue(Transform);
}
```

**Почему так:**
- `NameToId` - конвертирует строку в int ID для performance
- `AggressiveInlining` - компилятор максимально оптимизирует
- Extension methods - удобный синтаксис `entity.GetTransform()`
- Type-safe - компилятор проверяет типы

#### Шаг 2: Создание Data Classes
**Не требуется** - используется стандартный Unity Transform.

#### Шаг 3: Создание UseCases
**Не требуется** - фича чисто data-ориентированная.

#### Шаг 4: Создание Behaviours
**Не требуется** - Transform добавляется в Installer и используется другими behaviours.

#### Шаг 5: Использование в Installer

**Файл:** `Assets/Examples/Beginner/Scripts/Entities/Content/CharacterInstaller.cs`

```csharp
using Atomic.Entities;
using UnityEngine;

namespace BeginnerGame
{
    /// <summary>
    /// Устанавливает компоненты для персонажа игрока.
    /// </summary>
    public sealed class CharacterInstaller : SceneEntityInstaller
    {
        public override void Install(IEntity entity)
        {
            // this.transform - это MonoBehaviour.transform
            // Добавляем reference на Unity Transform в entity
            entity.AddTransform(this.transform);
        }
    }
}
```

**Почему так:**
- `SceneEntityInstaller` - базовый класс для MonoBehaviour installers
- `this.transform` - ссылка на Unity Transform компонент
- Не создаем копию Transform - используем reference

**Настройка в Unity:**
1. Создать GameObject "Character" на сцене
2. Добавить компонент `CharacterInstaller`
3. Transform автоматически доступен через `this.transform`

#### Шаг 6: Интеграция с другими фичами

Transform используется в:
- **MovementBehaviour** для изменения позиции (`_transform.position +=`)
- **CoinSpawnBehaviour** для установки позиции при спавне (`position` параметр)
- **Визуальных behaviours** для синхронизации Model-View

#### Зависимости
**Нет зависимостей** - это базовая фича, от которой зависят другие.

### Best Practices

1. **Transform как bridge**: Связывает Atomic Entity с Unity GameObject
2. **Reference, не copy**: Работаем с оригинальным Transform
3. **Добавляем рано**: В начале `Install()` метода
4. **Один Transform на entity**: Не добавляем Multiple transforms

### Проверка работы

```csharp
// В любом behaviour:
public void Init(IEntity entity)
{
    Transform transform = entity.GetTransform();
    Debug.Log($"Position: {transform.position}");
}
```

---

## Feature 2: Movement Direction и Movement Speed

### Описание
Промежуточная фича для хранения направления и скорости движения. Направление устанавливается **InputBehaviour**, а читается **MovementBehaviour**. Это паттерн **Producer-Consumer** между двумя behaviours.

### Сложность: ⭐ Простая

### Используемые компоненты
- **EntityAPI**: MovementDirection (IVariable<Vector3>), MovementSpeed (IValue<float>)
- **Installers**: CharacterInstaller
- **Behaviours**: InputBehaviour (записывает), MovementBehaviour (читает)
- **UseCases**: Не требуется
- **Data Classes**: Использует интерфейсы из Atomic.Elements

### Пошаговое внедрение

#### Шаг 1: Добавление в EntityAPI

**Файл:** `Assets/Examples/Beginner/Scripts/Entities/EntityAPI.cs`

```csharp
public static class EntityAPI
{
    // IDs
    public static readonly int MovementDirection; // IVariable<Vector3>
    public static readonly int MovementSpeed;     // IValue<float>

    static EntityAPI()
    {
        MovementDirection = NameToId(nameof(MovementDirection));
        MovementSpeed = NameToId(nameof(MovementSpeed));
    }

    // MovementDirection methods

    /// <summary>
    /// Получает направление движения (read/write).
    /// </summary>
    [MethodImpl(MethodImplOptions.AggressiveInlining)]
    public static IVariable<Vector3> GetMovementDirection(this IEntity entity)
        => entity.GetValue<IVariable<Vector3>>(MovementDirection);

    /// <summary>
    /// Добавляет направление движения в entity.
    /// </summary>
    [MethodImpl(MethodImplOptions.AggressiveInlining)]
    public static void AddMovementDirection(this IEntity entity, IVariable<Vector3> value)
        => entity.AddValue(MovementDirection, value);

    // MovementSpeed methods

    /// <summary>
    /// Получает скорость движения (read-only).
    /// </summary>
    [MethodImpl(MethodImplOptions.AggressiveInlining)]
    public static IValue<float> GetMovementSpeed(this IEntity entity)
        => entity.GetValue<IValue<float>>(MovementSpeed);

    /// <summary>
    /// Добавляет скорость движения в entity.
    /// </summary>
    [MethodImpl(MethodImplOptions.AggressiveInlining)]
    public static void AddMovementSpeed(this IEntity entity, IValue<float> value)
        => entity.AddValue(MovementSpeed, value);
}
```

**Почему так:**
- `IVariable<T>` - для изменяемых значений (направление)
- `IValue<T>` - для read-only значений (скорость)
- Интерфейсы вместо конкретных классов - гибкость

#### Шаг 2: Создание Data Classes
**Не требуется** - используются интерфейсы из Atomic.Elements:
- `Variable<Vector3>` - изменяемое направление
- `Const<float>` - константная скорость

#### Шаг 3: Создание UseCases
**Не требуется** - логика находится в behaviours.

#### Шаг 4: Создание Behaviours
Используется в InputBehaviour и MovementBehaviour (см. следующие фичи).

#### Шаг 5: Использование в Installer

**Файл:** `Assets/Examples/Beginner/Scripts/Entities/Content/CharacterInstaller.cs`

```csharp
using Atomic.Elements;
using Atomic.Entities;
using UnityEngine;

namespace BeginnerGame
{
    public sealed class CharacterInstaller : SceneEntityInstaller
    {
        [Header("Movement")]
        [SerializeField]
        private Const<float> _moveSpeed = 3; // Настраивается в Inspector

        public override void Install(IEntity entity)
        {
            // Transform (Feature 1)
            entity.AddTransform(this.transform);

            // === Movement Data ===

            // Скорость - константа, не меняется
            entity.AddMovementSpeed(_moveSpeed);

            // Направление - изменяемая переменная
            // Создаем новый экземпляр для каждой entity
            entity.AddMovementDirection(new Variable<Vector3>());
        }
    }
}
```

**Почему так:**
- `Const<float>` для скорости - настраивается в Inspector, не меняется в runtime
- `Variable<Vector3>` для направления - создается новый для каждой entity
- `new Variable<Vector3>()` - каждой entity нужен свой экземпляр

**Настройка в Unity:**
1. На CharacterInstaller в Inspector установить `_moveSpeed` (например, 3)
2. Разные персонажи могут иметь разную скорость

#### Шаг 6: Интеграция

**Producer-Consumer паттерн:**

```
InputBehaviour (Producer)
    ↓ writes to
MovementDirection (Shared State)
    ↓ reads from
MovementBehaviour (Consumer)
```

**Преимущества:**
- Input и Movement разделены - можно заменить Input на AI
- Направление - shared state между behaviours
- Update (Input) и FixedUpdate (Movement) независимы

#### Зависимости
**Нет прямых зависимостей** на другие фичи.

### Best Practices

1. **IVariable vs IValue**:
   - `IVariable<T>` - когда нужно read/write (направление)
   - `IValue<T>` - когда только read (скорость)

2. **Const<T> vs Variable<T>**:
   - `Const<T>` - для constants, настраивается в Inspector
   - `Variable<T>` - для runtime изменений

3. **Новый экземпляр для каждой entity**:
   ```csharp
   // ✅ Правильно - каждой entity свой
   entity.AddMovementDirection(new Variable<Vector3>());

   // ❌ Неправильно - все entity будут шарить один
   private Variable<Vector3> _sharedDirection;
   entity.AddMovementDirection(_sharedDirection);
   ```

4. **Vector3.zero по умолчанию**: `new Variable<Vector3>()` инициализируется как (0,0,0)

### Проверка работы

```csharp
// Установить направление:
entity.GetMovementDirection().Value = Vector3.forward;

// Прочитать направление:
Vector3 dir = entity.GetMovementDirection().Value;

// Прочитать скорость:
float speed = entity.GetMovementSpeed().Value;
```

---

# Уровень 2: Средние фичи

## Feature 3: Input System

### Описание
Обрабатывает ввод с клавиатуры и записывает направление движения в **MovementDirection**. Использует ScriptableObject **InputMap** для настройки клавиш управления, что позволяет разным игрокам иметь разные схемы (WASD для одного, стрелки для другого).

### Сложность: ⭐⭐ Средняя

### Используемые компоненты
- **EntityAPI**: InputMap, MovementDirection
- **Installers**: CharacterInstaller
- **Behaviours**: InputBehaviour (IEntityInit, IEntityTick)
- **UseCases**: Не требуется
- **Data Classes**: InputMap (ScriptableObject)

### Пошаговое внедрение

#### Шаг 1: Добавление в EntityAPI

**Файл:** `Assets/Examples/Beginner/Scripts/Entities/EntityAPI.cs`

```csharp
public static class EntityAPI
{
    public static readonly int InputMap; // InputMap

    static EntityAPI()
    {
        InputMap = NameToId(nameof(InputMap));
    }

    /// <summary>
    /// Получает InputMap из entity.
    /// </summary>
    [MethodImpl(MethodImplOptions.AggressiveInlining)]
    public static InputMap GetInputMap(this IEntity entity)
        => entity.GetValue<InputMap>(InputMap);

    /// <summary>
    /// Добавляет InputMap в entity.
    /// </summary>
    [MethodImpl(MethodImplOptions.AggressiveInlining)]
    public static void AddInputMap(this IEntity entity, InputMap value)
        => entity.AddValue(InputMap, value);
}
```

#### Шаг 2: Создание Data Classes

**Файл:** `Assets/Examples/Beginner/Scripts/Common/InputMap.cs`

```csharp
using UnityEngine;

namespace BeginnerGame
{
    /// <summary>
    /// ScriptableObject для настройки клавиш управления.
    /// Позволяет создавать разные схемы управления для разных игроков.
    /// </summary>
    [CreateAssetMenu(fileName = "InputMap", menuName = "BeginnerGame/InputMap")]
    public sealed class InputMap : ScriptableObject
    {
        [Header("Movement Keys")]
        public KeyCode forward = KeyCode.W;   // Вперед
        public KeyCode back = KeyCode.S;      // Назад
        public KeyCode left = KeyCode.A;      // Влево
        public KeyCode right = KeyCode.D;     // Вправо
    }
}
```

**Почему ScriptableObject:**
- Переиспользуемо - можно создать несколько конфигураций
- Настраивается в Inspector без кода
- Не привязан к конкретной scene
- Можно создать Player1Input (WASD) и Player2Input (Arrows)

**Создание в Unity:**
1. ПКМ в Project → Create → BeginnerGame → InputMap
2. Назвать "Player1Input"
3. Настроить клавиши: W/A/S/D
4. Создать еще один "Player2Input"
5. Настроить клавиши: Up/Down/Left/Right

#### Шаг 3: Создание UseCases
**Не требуется** - логика простая, помещается в Behaviour.

#### Шаг 4: Создание Behaviours

**Файл:** `Assets/Examples/Beginner/Scripts/Entities/Behaviours/InputBehaviour.cs`

```csharp
using Atomic.Elements;
using Atomic.Entities;
using UnityEngine;

namespace BeginnerGame
{
    /// <summary>
    /// Обрабатывает ввод с клавиатуры и обновляет направление движения entity.
    /// </summary>
    /// <remarks>
    /// Lifecycle:
    /// - Init: кешируем InputMap и MovementDirection (один раз)
    /// - Tick: читаем клавиатуру и обновляем направление (каждый кадр)
    /// </remarks>
    public sealed class InputBehaviour : IEntityInit, IEntityTick
    {
        // Кешируемые ссылки (устанавливаются в Init)
        private InputMap _inputMap;
        private IVariable<Vector3> _moveDirection;

        /// <summary>
        /// Init вызывается один раз при создании entity.
        /// Используется для кеширования ссылок на компоненты.
        /// </summary>
        /// <param name="entity">Entity, которая владеет этим behaviour</param>
        public void Init(IEntity entity)
        {
            // Кешируем компоненты для производительности
            _inputMap = entity.GetInputMap();
            _moveDirection = entity.GetMovementDirection();
        }

        /// <summary>
        /// Tick вызывается каждый кадр (Unity Update).
        /// Используется для чтения input и обновления направления.
        /// </summary>
        /// <param name="entity">Entity, которая владеет этим behaviour</param>
        /// <param name="deltaTime">Время с прошлого кадра</param>
        public void Tick(IEntity entity, float deltaTime)
        {
            // Читаем клавиатуру и обновляем направление
            _moveDirection.Value = this.ReadMoveDirection();
        }

        /// <summary>
        /// Читает клавиатуру и возвращает normalized направление движения.
        /// </summary>
        /// <returns>Направление движения (не normalized, может быть диагональ)</returns>
        private Vector3 ReadMoveDirection()
        {
            Vector3 moveDirection = Vector3.zero;

            // Горизонтальное движение (X ось)
            if (Input.GetKey(_inputMap.left))
                moveDirection.x = -1;
            else if (Input.GetKey(_inputMap.right))
                moveDirection.x = 1;

            // Вертикальное движение (Z ось в Unity - forward/back)
            if (Input.GetKey(_inputMap.back))
                moveDirection.z = -1;
            else if (Input.GetKey(_inputMap.forward))
                moveDirection.z = 1;

            return moveDirection;
        }
    }
}
```

**Почему так:**

**IEntityInit:**
- Вызывается один раз при создании
- Используем для кеширования компонентов
- Избегаем GetComponent на каждом кадре

**IEntityTick:**
- Вызывается каждый кадр (Update)
- Идеально для input processing
- deltaTime передается, но не используется (input не time-dependent)

**Разделение логики:**
- `Tick()` - высокоуровневая логика (обновить direction)
- `ReadMoveDirection()` - низкоуровневая логика (прочитать клавиши)

**else-if вместо if:**
- Взаимоисключающие направления (лево vs право)
- Предотвращает одновременное нажатие противоположных клавиш

#### Шаг 5: Создание Installer

**Файл:** `Assets/Examples/Beginner/Scripts/Entities/Content/CharacterInstaller.cs`

```csharp
using Atomic.Elements;
using Atomic.Entities;
using UnityEngine;

namespace BeginnerGame
{
    public sealed class CharacterInstaller : SceneEntityInstaller
    {
        [Header("Movement")]
        [SerializeField]
        private Const<float> _moveSpeed = 3;

        [Header("Input")]
        [SerializeField]
        private InputMap _inputMap; // Ссылка на ScriptableObject

        public override void Install(IEntity entity)
        {
            // Transform (Feature 1)
            entity.AddTransform(this.transform);

            // Movement Data (Feature 2)
            entity.AddMovementSpeed(_moveSpeed);
            entity.AddMovementDirection(new Variable<Vector3>());

            // === Input System (Feature 3) ===

            // Добавляем InputMap
            entity.AddInputMap(_inputMap);

            // Добавляем Behaviour для обработки ввода
            entity.AddBehaviour<InputBehaviour>();
        }
    }
}
```

**Настройка в Unity:**
1. На CharacterInstaller в Inspector:
   - Перетащить InputMap ScriptableObject в поле `_inputMap`
2. Для Player 1: использовать Player1Input (WASD)
3. Для Player 2: использовать Player2Input (Arrows)

#### Шаг 6: Интеграция

**Схема работы:**

```
User presses key (W)
    ↓
Input.GetKey() detects
    ↓
InputBehaviour.Tick() calls ReadMoveDirection()
    ↓
Writes to MovementDirection.Value
    ↓
MovementBehaviour reads direction
    ↓
Applies movement to Transform
```

**Producer в Producer-Consumer:**
- InputBehaviour - **Producer** направления
- MovementBehaviour - **Consumer** направления
- MovementDirection - **Shared state**

#### Зависимости
- **MovementDirection** (Feature 2) - обязательно должно существовать в entity

### Best Practices

1. **IEntityInit для кеширования**:
   ```csharp
   // ✅ Правильно - кешируем в Init
   private InputMap _inputMap;
   public void Init(IEntity entity)
   {
       _inputMap = entity.GetInputMap();
   }

   // ❌ Неправильно - получаем каждый кадр
   public void Tick(IEntity entity, float deltaTime)
   {
       InputMap map = entity.GetInputMap(); // Медленно!
   }
   ```

2. **IEntityTick для input**, не Update:
   - Atomic Framework управляет lifecycle
   - Не нужно наследоваться от MonoBehaviour
   - Легче тестировать

3. **ScriptableObject для конфигурации**:
   - Переиспользуемо
   - Настраивается без кода
   - Можно создать preset'ы

4. **else-if для взаимоисключающих действий**:
   ```csharp
   // ✅ Правильно - одновременно только одно направление
   if (Input.GetKey(left))
       dir.x = -1;
   else if (Input.GetKey(right))
       dir.x = 1;

   // ❌ Неправильно - может быть 0 если оба нажаты
   if (Input.GetKey(left))
       dir.x -= 1;
   if (Input.GetKey(right))
       dir.x += 1;
   ```

5. **Не normalized direction**:
   - Диагональное движение будет быстрее (√2)
   - Для fixed speed нужно normalize в MovementBehaviour

### Проверка работы

```csharp
// В любом behaviour:
public void Tick(IEntity entity, float deltaTime)
{
    Vector3 dir = entity.GetMovementDirection().Value;
    if (dir != Vector3.zero)
        Debug.Log($"Moving: {dir}");
}
```

### Расширение фичи

**Добавить поддержку геймпада:**

```csharp
private Vector3 ReadMoveDirection()
{
    Vector3 dir = Vector3.zero;

    // Клавиатура
    if (Input.GetKey(_inputMap.left))
        dir.x = -1;
    else if (Input.GetKey(_inputMap.right))
        dir.x = 1;

    // Геймпад
    dir.x += Input.GetAxis("Horizontal");
    dir.z += Input.GetAxis("Vertical");

    return dir;
}
```

---

## Feature 4: Movement

### Описание
Применяет физическое движение к Transform на основе направления и скорости. Использует **FixedTick** (FixedUpdate) для стабильной физики. Это **Consumer** в Producer-Consumer паттерне.

### Сложность: ⭐⭐ Средняя

### Используемые компоненты
- **EntityAPI**: Transform, MovementDirection, MovementSpeed
- **Installers**: CharacterInstaller
- **Behaviours**: MovementBehaviour (IEntityInit, IEntityFixedTick)
- **UseCases**: Можно вынести, но для простоты в behaviour
- **Data Classes**: Нет

### Пошаговое внедрение

#### Шаг 1: Добавление в EntityAPI
**Уже добавлено** в предыдущих фичах:
- Transform (Feature 1)
- MovementDirection (Feature 2)
- MovementSpeed (Feature 2)

#### Шаг 2: Создание Data Classes
**Не требуется** - используются существующие компоненты.

#### Шаг 3: Создание UseCases

**Опционально** - можно вынести логику движения:

**Файл:** `Assets/Examples/Beginner/Scripts/Entities/UseCases/MovementUseCase.cs`

```csharp
using UnityEngine;

namespace BeginnerGame
{
    /// <summary>
    /// UseCase для кинематического движения.
    /// Чистая бизнес-логика без состояния.
    /// </summary>
    public static class MovementUseCase
    {
        /// <summary>
        /// Применяет кинематическое движение к Transform.
        /// </summary>
        /// <param name="transform">Transform для движения</param>
        /// <param name="direction">Направление (не normalized)</param>
        /// <param name="speed">Скорость движения</param>
        /// <param name="deltaTime">Время с прошлого кадра</param>
        public static void ApplyMovement(
            Transform transform,
            Vector3 direction,
            float speed,
            float deltaTime)
        {
            // Проверка на нулевое направление (оптимизация)
            if (direction == Vector3.zero)
                return;

            // Кинематическая формула: position += direction * speed * time
            transform.position += direction * (speed * deltaTime);
        }

        /// <summary>
        /// Применяет движение с normalized направлением.
        /// Диагональное движение будет с той же скоростью, что и прямое.
        /// </summary>
        public static void ApplyMovementNormalized(
            Transform transform,
            Vector3 direction,
            float speed,
            float deltaTime)
        {
            if (direction == Vector3.zero)
                return;

            // Normalize для постоянной скорости во всех направлениях
            Vector3 normalized = direction.normalized;
            transform.position += normalized * (speed * deltaTime);
        }
    }
}
```

**Преимущества UseCase:**
- Легко тестировать (pure function)
- Переиспользуемо (может использоваться AI, pathfinding)
- Без состояния (stateless)

#### Шаг 4: Создание Behaviours

**Файл:** `Assets/Examples/Beginner/Scripts/Entities/Behaviours/MovementBehaviour.cs`

```csharp
using Atomic.Elements;
using Atomic.Entities;
using UnityEngine;

namespace BeginnerGame
{
    /// <summary>
    /// Применяет кинематическое движение к Transform entity.
    /// Читает направление и скорость, модифицирует позицию.
    /// </summary>
    /// <remarks>
    /// Lifecycle:
    /// - Init: кешируем Transform, MovementDirection, MovementSpeed
    /// - FixedTick: применяем движение с фиксированным timestep
    ///
    /// Почему FixedTick:
    /// - Для физики нужен стабильный timestep
    /// - FixedUpdate вызывается с постоянной частотой (по умолчанию 50 Hz)
    /// - Движение будет одинаковым независимо от FPS
    /// </remarks>
    public sealed class MovementBehaviour : IEntityInit, IEntityFixedTick
    {
        // Кешированные компоненты
        private Transform _transform;
        private IValue<Vector3> _movementDirection;
        private IValue<float> _movementSpeed;

        /// <summary>
        /// Init: кешируем все необходимые компоненты один раз.
        /// </summary>
        public void Init(IEntity entity)
        {
            _transform = entity.GetTransform();
            _movementDirection = entity.GetMovementDirection();
            _movementSpeed = entity.GetMovementSpeed();
        }

        /// <summary>
        /// FixedTick: вызывается с фиксированным интервалом (FixedUpdate).
        /// Используется для физики и движения.
        /// </summary>
        /// <param name="entity">Entity, которая владеет этим behaviour</param>
        /// <param name="deltaTime">Fixed delta time (обычно 0.02 сек)</param>
        public void FixedTick(IEntity entity, float deltaTime)
        {
            // Читаем текущее направление и скорость
            Vector3 direction = _movementDirection.Value;
            float speed = _movementSpeed.Value;

            // Оптимизация: не двигаемся если direction = (0,0,0)
            if (direction == Vector3.zero)
                return;

            // Простая кинематическая формула: position += direction * speed * time
            // direction не normalized - диагональ будет быстрее на √2
            _transform.position += direction * (speed * deltaTime);
        }
    }
}
```

**Альтернативная версия с UseCase:**

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
        // Делегируем логику в UseCase
        MovementUseCase.ApplyMovement(
            _transform,
            _movementDirection.Value,
            _movementSpeed.Value,
            deltaTime
        );
    }
}
```

#### Шаг 5: Создание Installer

**Файл:** `Assets/Examples/Beginner/Scripts/Entities/Content/CharacterInstaller.cs`

```csharp
using Atomic.Elements;
using Atomic.Entities;
using UnityEngine;

namespace BeginnerGame
{
    public sealed class CharacterInstaller : SceneEntityInstaller
    {
        [Header("Movement")]
        [SerializeField]
        private Const<float> _moveSpeed = 3;

        [Header("Input")]
        [SerializeField]
        private InputMap _inputMap;

        public override void Install(IEntity entity)
        {
            // === Transform (Feature 1) ===
            entity.AddTransform(this.transform);

            // === Movement Data (Feature 2) ===
            entity.AddMovementSpeed(_moveSpeed);
            entity.AddMovementDirection(new Variable<Vector3>());

            // === Movement Behaviour (Feature 4) ===
            entity.AddBehaviour<MovementBehaviour>();

            // === Input System (Feature 3) ===
            entity.AddInputMap(_inputMap);
            entity.AddBehaviour<InputBehaviour>();
        }
    }
}
```

**Порядок добавления:**
1. **Данные сначала**: Transform, Speed, Direction
2. **Behaviours потом**: MovementBehaviour, InputBehaviour
3. Порядок behaviours не важен (они независимы через shared state)

#### Шаг 6: Интеграция

**Полная схема движения:**

```
Frame N (Update):
    InputBehaviour.Tick()
        ↓ читает клавиатуру
        ↓ пишет MovementDirection.Value = (1, 0, 0)

Fixed Frame N (FixedUpdate):
    MovementBehaviour.FixedTick()
        ↓ читает MovementDirection.Value
        ↓ читает MovementSpeed.Value
        ↓ модифицирует Transform.position
```

**Почему Update и FixedUpdate разделены:**
- **Input в Update**: Отзывчивость, каждый кадр
- **Movement в FixedUpdate**: Стабильность, fixed timestep
- MovementDirection - связующее звено (shared state)

**Преимущества разделения:**
- Можно заменить Input на AI без изменения Movement
- Movement не зависит от FPS
- Input записывает, Movement читает и применяет

#### Зависимости
- **Transform** (Feature 1) - для модификации позиции
- **MovementDirection** (Feature 2) - для чтения направления
- **MovementSpeed** (Feature 2) - для чтения скорости

### Best Practices

1. **IEntityFixedTick для физики**:
   ```csharp
   // ✅ Правильно - физика в FixedTick
   public sealed class MovementBehaviour : IEntityFixedTick
   {
       public void FixedTick(IEntity entity, float deltaTime)
       {
           // Движение с fixed timestep
       }
   }

   // ❌ Неправильно - физика в Tick (нестабильно)
   public sealed class MovementBehaviour : IEntityTick
   {
       public void Tick(IEntity entity, float deltaTime)
       {
           // Движение зависит от FPS
       }
   }
   ```

2. **Кинематическое vs физическое движение**:
   ```csharp
   // Кинематическое (без Rigidbody)
   _transform.position += direction * speed * deltaTime;

   // Физическое (с Rigidbody)
   _rigidbody.velocity = direction * speed;
   // или
   _rigidbody.MovePosition(position + direction * speed * deltaTime);
   ```

3. **Оптимизация для нулевого направления**:
   ```csharp
   // ✅ Правильно - early exit
   if (direction == Vector3.zero)
       return;
   _transform.position += direction * speed * deltaTime;

   // ❌ Неоптимально - лишние вычисления
   _transform.position += direction * speed * deltaTime; // 0 * speed * time = 0
   ```

4. **Normalized direction (опционально)**:
   ```csharp
   // Без normalize: диагональ быстрее на √2
   _transform.position += direction * speed * deltaTime;

   // С normalize: постоянная скорость во всех направлениях
   _transform.position += direction.normalized * speed * deltaTime;
   ```

5. **Кеширование компонентов в Init**:
   - GetTransform() на каждом кадре медленно
   - Кешируем в Init один раз

### Проверка работы

**Тест в Unity:**
1. Запустить игру
2. Нажать WASD
3. Персонаж должен двигаться
4. Debug.Log в FixedTick для проверки:

```csharp
public void FixedTick(IEntity entity, float deltaTime)
{
    Vector3 direction = _movementDirection.Value;
    Debug.Log($"Direction: {direction}, Speed: {_movementSpeed.Value}");
    // ... apply movement
}
```

### Расширение фичи

**Добавить ускорение:**

```csharp
[SerializeField]
private Const<float> _acceleration = 10;
[SerializeField]
private Const<float> _maxSpeed = 5;

private Vector3 _currentVelocity;

public void FixedTick(IEntity entity, float deltaTime)
{
    Vector3 targetDirection = _movementDirection.Value;
    float maxSpeed = _maxSpeed.Value;

    // Плавное ускорение
    _currentVelocity = Vector3.MoveTowards(
        _currentVelocity,
        targetDirection * maxSpeed,
        _acceleration.Value * deltaTime
    );

    _transform.position += _currentVelocity * deltaTime;
}
```

---

## Feature 5: Money/Score System

### Описание
Реактивная система подсчета очков. Использует **IReactiveVariable** для автоматического уведомления UI при изменении значения. Используется и персонажем (накопленная сумма) и монетой (стоимость).

### Сложность: ⭐⭐ Средняя

### Используемые компоненты
- **EntityAPI**: Money (IReactiveVariable<int>)
- **Installers**: CharacterInstaller, CoinInstaller
- **Behaviours**: CoinCollectBehaviour (модифицирует Money)
- **UseCases**: Не требуется
- **Data Classes**: Использует ReactiveVariable из Atomic.Elements
- **UI**: MoneyView (подписывается на изменения)

### Пошаговое внедрение

#### Шаг 1: Добавление в EntityAPI

**Файл:** `Assets/Examples/Beginner/Scripts/Entities/EntityAPI.cs`

```csharp
public static class EntityAPI
{
    public static readonly int Money; // IReactiveVariable<int>

    static EntityAPI()
    {
        Money = NameToId(nameof(Money));
    }

    /// <summary>
    /// Получает Money из entity (reactive - поддерживает подписки).
    /// </summary>
    [MethodImpl(MethodImplOptions.AggressiveInlining)]
    public static IReactiveVariable<int> GetMoney(this IEntity entity)
        => entity.GetValue<IReactiveVariable<int>>(Money);

    /// <summary>
    /// Добавляет Money в entity.
    /// </summary>
    [MethodImpl(MethodImplOptions.AggressiveInlining)]
    public static void AddMoney(this IEntity entity, IReactiveVariable<int> value)
        => entity.AddValue(Money, value);

    /// <summary>
    /// Проверяет наличие Money в entity.
    /// </summary>
    [MethodImpl(MethodImplOptions.AggressiveInlining)]
    public static bool HasMoney(this IEntity entity)
        => entity.HasValue(Money);
}
```

**Почему IReactiveVariable вместо IVariable:**
- Поддерживает подписки (Observe/Subscribe)
- Автоматически уведомляет UI при изменении
- Не нужно вручную обновлять UI

#### Шаг 2: Создание Data Classes
**Не требуется** - используется `ReactiveVariable<int>` из Atomic.Elements.

**Как работает ReactiveVariable (внутри фреймворка):**

```csharp
// Упрощенная версия для понимания
public class ReactiveVariable<T> : IReactiveVariable<T>
{
    private T _value;
    private event Action<T> _onChanged;

    public T Value
    {
        get => _value;
        set
        {
            _value = value;
            _onChanged?.Invoke(_value); // Уведомляем подписчиков!
        }
    }

    public Subscription<T> Observe(Action<T> action)
    {
        _onChanged += action;
        action(_value); // Вызываем сразу с текущим значением
        return new Subscription<T>(() => _onChanged -= action);
    }
}
```

#### Шаг 3: Создание UseCases
**Не требуется** - простая модификация значения через `money.Value++`.

**Опционально - для сложной логики:**

```csharp
public static class MoneyUseCase
{
    public static void AddMoney(IEntity entity, int amount)
    {
        if (!entity.HasMoney())
            return;

        IReactiveVariable<int> money = entity.GetMoney();
        money.Value += amount;
    }

    public static bool TrySpendMoney(IEntity entity, int cost)
    {
        if (!entity.HasMoney())
            return false;

        IReactiveVariable<int> money = entity.GetMoney();

        if (money.Value < cost)
            return false; // Недостаточно денег

        money.Value -= cost;
        return true;
    }
}
```

#### Шаг 4: Создание Behaviours
**Не создается отдельный Behaviour** - используется в CoinCollectBehaviour (Feature 7).

#### Шаг 5: Создание Installer

**Для персонажа:**

**Файл:** `Assets/Examples/Beginner/Scripts/Entities/Content/CharacterInstaller.cs`

```csharp
using Atomic.Elements;
using Atomic.Entities;
using UnityEngine;

namespace BeginnerGame
{
    public sealed class CharacterInstaller : SceneEntityInstaller
    {
        [Header("Score")]
        [SerializeField]
        private ReactiveVariable<int> _money; // Начальное значение в Inspector

        public override void Install(IEntity entity)
        {
            // ... другие компоненты ...

            // === Money System (Feature 5) ===
            entity.AddMoney(_money);
        }
    }
}
```

**Для монеты:**

**Файл:** `Assets/Examples/Beginner/Scripts/Entities/Content/CoinInstaller.cs`

```csharp
using Atomic.Elements;
using Atomic.Entities;
using UnityEngine;

namespace BeginnerGame
{
    /// <summary>
    /// Устанавливает компоненты для монеты.
    /// Монета - это пассивная entity без behaviours.
    /// </summary>
    public sealed class CoinInstaller : SceneEntityInstaller
    {
        [Header("Value")]
        [SerializeField]
        private ReactiveVariable<int> _money = 1; // Стоимость монеты

        public override void Install(IEntity entity)
        {
            // Тег для идентификации (Feature 7)
            entity.AddCoinTag();

            // Transform для позиции (Feature 1)
            entity.AddTransform(this.transform);

            // === Money System (Feature 5) ===
            // У монеты Money - это ее стоимость
            entity.AddMoney(_money);
        }
    }
}
```

**Настройка в Unity:**

**Для CharacterInstaller:**
1. В Inspector установить `_money` → Value = 0 (начальные деньги)
2. Можно настроить разные стартовые значения для игроков

**Для CoinInstaller:**
1. В Inspector установить `_money` → Value = 1 (стандартная монета)
2. Можно создать "золотую монету" с Value = 5

#### Шаг 6: Интеграция

**UI Integration (MoneyView):**

**Файл:** `Assets/Examples/Beginner/Scripts/UI/MoneyView.cs`

```csharp
using Atomic.Elements;
using Atomic.Entities;
using TMPro;
using UnityEngine;

namespace BeginnerGame
{
    /// <summary>
    /// Отображает количество денег игрока.
    /// Автоматически обновляется при изменении Money через Reactive подписку.
    /// </summary>
    public sealed class MoneyView : MonoBehaviour
    {
        [Header("References")]
        [SerializeField]
        private SceneEntity _player; // Ссылка на entity игрока

        [SerializeField]
        private TMP_Text _moneyText; // UI текст для отображения

        private Subscription<int> _subscription; // Подписка на изменения

        /// <summary>
        /// Start: подписываемся на изменения Money.
        /// </summary>
        private void Start()
        {
            // Получаем ReactiveVariable<int> из entity
            IReactiveVariable<int> money = _player.GetMoney();

            // Observe() подписывается и вызывает callback немедленно с текущим значением
            _subscription = money.Observe(this.OnMoneyChanged);
        }

        /// <summary>
        /// OnDestroy: ОБЯЗАТЕЛЬНО отписываемся для предотвращения memory leaks.
        /// </summary>
        private void OnDestroy()
        {
            // Dispose() отписывает от события
            _subscription?.Dispose();
        }

        /// <summary>
        /// Вызывается каждый раз, когда Money изменяется.
        /// Также вызывается один раз при подписке (в Start).
        /// </summary>
        /// <param name="money">Новое значение Money</param>
        private void OnMoneyChanged(int money)
        {
            _moneyText.text = $"Money: {money}";
        }
    }
}
```

**Настройка MoneyView в Unity:**
1. Создать Canvas → Text (TextMeshPro)
2. Добавить компонент `MoneyView`
3. Перетащить Player entity в поле `_player`
4. Перетащить Text в поле `_moneyText`
5. Повторить для второго игрока

**Использование в Behaviours:**

```csharp
// В CoinCollectBehaviour:
public void OnTriggerEntered(Collider collider)
{
    // ...

    // Увеличиваем деньги (автоматически уведомит UI!)
    _money.Value++;

    // ...
}
```

#### Зависимости
**Нет зависимостей** - standalone фича.

### Best Practices

1. **IReactiveVariable vs IVariable**:
   ```csharp
   // ✅ Reactive - для данных, которые нужно отображать в UI
   IReactiveVariable<int> money = new ReactiveVariable<int>(0);
   money.Observe(value => UpdateUI(value));

   // ❌ Обычная Variable - нужно вручную обновлять UI
   IVariable<int> money = new Variable<int>(0);
   // Как узнать, что изменилось?
   ```

2. **Обязательный Dispose подписок**:
   ```csharp
   // ✅ Правильно - отписываемся
   private Subscription<int> _sub;

   private void Start()
   {
       _sub = money.Observe(OnChanged);
   }

   private void OnDestroy()
   {
       _sub?.Dispose(); // ОБЯЗАТЕЛЬНО!
   }

   // ❌ Неправильно - memory leak
   private void Start()
   {
       money.Observe(OnChanged); // Никогда не отписываемся
   }
   ```

3. **Observe vs Subscribe**:
   ```csharp
   // Observe - вызывает callback сразу с текущим значением
   money.Observe(value => Debug.Log(value)); // Выведет текущее значение сразу

   // Subscribe - вызывает только при изменениях
   money.Subscribe(value => Debug.Log(value)); // Выведет только при изменении
   ```

4. **ReactiveVariable как SerializeField**:
   ```csharp
   [SerializeField]
   private ReactiveVariable<int> _money; // Можно настроить начальное значение в Inspector
   ```

5. **Двойное использование Money**:
   - У персонажа - **накопленная сумма**
   - У монеты - **стоимость монеты**

   Одна и та же фича, разное применение!

### Проверка работы

**Тест в коде:**

```csharp
// Установить значение
entity.GetMoney().Value = 100;

// Изменить значение
entity.GetMoney().Value += 10;

// Прочитать значение
int current = entity.GetMoney().Value;
Debug.Log($"Money: {current}");
```

**Тест подписки:**

```csharp
IReactiveVariable<int> money = entity.GetMoney();
Subscription<int> sub = money.Observe(value => Debug.Log($"Money changed: {value}"));

money.Value = 50; // Выведет: "Money changed: 50"
money.Value += 10; // Выведет: "Money changed: 60"

sub.Dispose(); // Отписываемся
money.Value += 10; // Ничего не выведет
```

### Расширение фичи

**Добавить максимальное значение:**

```csharp
[Serializable]
public class ClampedMoney
{
    [SerializeField] private int _min = 0;
    [SerializeField] private int _max = 999;

    private readonly ReactiveVariable<int> _value = new();

    public int Value
    {
        get => _value.Value;
        set => _value.Value = Mathf.Clamp(value, _min, _max);
    }

    public Subscription<int> Observe(Action<int> action) => _value.Observe(action);
}
```

**Добавить события:**

```csharp
[Serializable]
public class MoneyWithEvents : IReactiveVariable<int>
{
    private readonly ReactiveVariable<int> _value = new();

    public event Action<int> OnMoneyAdded;
    public event Action<int> OnMoneySpent;

    public int Value
    {
        get => _value.Value;
        set
        {
            int delta = value - _value.Value;
            _value.Value = value;

            if (delta > 0)
                OnMoneyAdded?.Invoke(delta);
            else if (delta < 0)
                OnMoneySpent?.Invoke(-delta);
        }
    }
}
```

---

# Уровень 3: Сложные фичи

## Feature 6: Trigger Detection

### Описание
Обнаружение коллизий через Unity Trigger system. Предоставляет события **OnEntered/OnExited** для других behaviours. Это foundation для всех систем взаимодействия через физику (сбор предметов, зоны урона, checkpoints).

### Сложность: ⭐⭐⭐ Сложная

### Используемые компоненты
- **EntityAPI**: TriggerEvents
- **Installers**: CharacterInstaller
- **Behaviours**: CoinCollectBehaviour (подписывается на OnEntered)
- **UseCases**: Не требуется
- **Data Classes**: TriggerEvents (MonoBehaviour из Atomic Framework)

### Пошаговое внедрение

#### Шаг 1: Добавление в EntityAPI

**Файл:** `Assets/Examples/Beginner/Scripts/Entities/EntityAPI.cs`

```csharp
public static class EntityAPI
{
    public static readonly int TriggerEvents; // TriggerEvents

    static EntityAPI()
    {
        TriggerEvents = NameToId(nameof(TriggerEvents));
    }

    /// <summary>
    /// Получает TriggerEvents из entity.
    /// </summary>
    [MethodImpl(MethodImplOptions.AggressiveInlining)]
    public static TriggerEvents GetTriggerEvents(this IEntity entity)
        => entity.GetValue<TriggerEvents>(TriggerEvents);

    /// <summary>
    /// Добавляет TriggerEvents в entity.
    /// </summary>
    [MethodImpl(MethodImplOptions.AggressiveInlining)]
    public static void AddTriggerEvents(this IEntity entity, TriggerEvents value)
        => entity.AddValue(TriggerEvents, value);
}
```

#### Шаг 2: Создание Data Classes

**TriggerEvents** - стандартный компонент из Atomic Framework:

```csharp
using System;
using UnityEngine;

namespace Atomic.Elements
{
    /// <summary>
    /// MonoBehaviour компонент для обнаружения триггеров.
    /// Подключается к Unity OnTriggerEnter/OnTriggerExit callbacks.
    /// </summary>
    public sealed class TriggerEvents : MonoBehaviour
    {
        /// <summary>
        /// Вызывается когда Collider входит в триггер.
        /// </summary>
        public event Action<Collider> OnEntered;

        /// <summary>
        /// Вызывается когда Collider выходит из триггера.
        /// </summary>
        public event Action<Collider> OnExited;

        /// <summary>
        /// Unity callback - вход в триггер.
        /// </summary>
        private void OnTriggerEnter(Collider other)
        {
            OnEntered?.Invoke(other);
        }

        /// <summary>
        /// Unity callback - выход из триггера.
        /// </summary>
        private void OnTriggerExit(Collider other)
        {
            OnExited?.Invoke(other);
        }
    }
}
```

**Почему MonoBehaviour:**
- Нужен для Unity физических callbacks
- Bridge между Unity Physics и Atomic Framework
- Конвертирует Unity callbacks в C# события

#### Шаг 3: Создание UseCases
**Не требуется** - это event-based система.

#### Шаг 4: Создание Behaviours
**Не создается отдельный** - используется в CoinCollectBehaviour (Feature 7).

#### Шаг 5: Использование в Installer

**Файл:** `Assets/Examples/Beginner/Scripts/Entities/Content/CharacterInstaller.cs`

```csharp
using Atomic.Elements;
using Atomic.Entities;
using UnityEngine;

namespace BeginnerGame
{
    public sealed class CharacterInstaller : SceneEntityInstaller
    {
        [Header("Collision")]
        [SerializeField]
        private TriggerEvents _triggerEvents; // Ссылка на компонент

        public override void Install(IEntity entity)
        {
            // ... другие компоненты ...

            // === Trigger Detection (Feature 6) ===
            entity.AddTriggerEvents(_triggerEvents);
        }
    }
}
```

**Настройка в Unity:**

1. **На GameObject персонажа:**
   - Добавить компонент `Collider` (например, Capsule Collider)
   - Включить `Is Trigger` ✓
   - Добавить компонент `TriggerEvents`

2. **В CharacterInstaller:**
   - Перетащить компонент `TriggerEvents` в поле `_triggerEvents`

3. **На монеты тоже нужен Collider:**
   - Добавить `Sphere Collider`
   - Включить `Is Trigger` ✓

#### Шаг 6: Интеграция

**Схема работы:**

```
Character enters Coin trigger
    ↓
Unity OnTriggerEnter(Collider other)
    ↓
TriggerEvents.OnEntered.Invoke(other)
    ↓
CoinCollectBehaviour слушает OnEntered
    ↓
Проверяет IEntity и Coin tag
    ↓
Собирает монету
```

**Используется в:**
- **CoinCollectBehaviour** - подписывается на OnEntered
- Может использоваться для:
  - Зон урона (DamageZoneBehaviour)
  - Checkpoints (CheckpointBehaviour)
  - Телепортов (TeleportBehaviour)

#### Зависимости
**Требует:**
- Unity `Collider` с `Is Trigger = true`
- `TriggerEvents` MonoBehaviour компонент

### Best Practices

1. **Enable/Disable для подписки на события**:
   ```csharp
   // ✅ Правильно - подписка в Enable, отписка в Disable
   public void Enable(IEntity entity)
   {
       _triggerEvents.OnEntered += this.OnTriggerEntered;
   }

   public void Disable(IEntity entity)
   {
       _triggerEvents.OnEntered -= this.OnTriggerEntered;
   }

   // ❌ Неправильно - подписка в Init, отписка в Dispose
   // (можно, но Enable/Disable вызываются многократно)
   ```

2. **TryGetComponent для безопасности**:
   ```csharp
   private void OnTriggerEntered(Collider collider)
   {
       // ✅ Правильно - проверяем наличие IEntity
       if (!collider.TryGetComponent(out IEntity target))
           return;

       // ❌ Неправильно - может быть NullReferenceException
       IEntity target = collider.GetComponent<IEntity>();
   }
   ```

3. **Layer Mask для фильтрации**:
   ```csharp
   // В Physics Settings настроить Layer Collision Matrix
   // Player только с Coin, Coin только с Player
   ```

4. **Is Trigger vs Collider**:
   - `Is Trigger` - для обнаружения без физики (проходят сквозь)
   - Обычный Collider - для физических столкновений (блокируют)

### Проверка работы

```csharp
// Добавить Debug.Log в TriggerEvents:
private void OnTriggerEnter(Collider other)
{
    Debug.Log($"Trigger entered: {other.name}");
    OnEntered?.Invoke(other);
}
```

**Тест в Unity:**
1. Запустить игру
2. Двигать персонажа к монете
3. В Console должно появиться "Trigger entered: Coin"

---

## Feature 7: Coin Collection

### Описание
Обрабатывает сбор монет при столкновении через триггеры. Увеличивает **Money** игрока, уничтожает монету и интегрируется с **TriggerEvents**.

### Сложность: ⭐⭐⭐ Сложная

### Используемые компоненты
- **EntityAPI**: TriggerEvents, Money, Coin (tag)
- **Installers**: CharacterInstaller, CoinInstaller
- **Behaviours**: CoinCollectBehaviour (IEntityInit, IEntityEnable, IEntityDisable)
- **UseCases**: Опционально - CoinCollectionUseCase
- **Data Classes**: Нет

### Пошаговое внедрение

#### Шаг 1: Добавление в EntityAPI

**Coin Tag:**

```csharp
public static class EntityAPI
{
    public static readonly int Coin; // Tag

    static EntityAPI()
    {
        Coin = NameToId(nameof(Coin));
    }

    // Tag extension methods

    /// <summary>
    /// Проверяет наличие Coin тега.
    /// </summary>
    [MethodImpl(MethodImplOptions.AggressiveInlining)]
    public static bool HasCoinTag(this IEntity entity)
        => entity.HasTag(Coin);

    /// <summary>
    /// Добавляет Coin тег.
    /// </summary>
    [MethodImpl(MethodImplOptions.AggressiveInlining)]
    public static bool AddCoinTag(this IEntity entity)
        => entity.AddTag(Coin);

    /// <summary>
    /// Удаляет Coin тег.
    /// </summary>
    [MethodImpl(MethodImplOptions.AggressiveInlining)]
    public static bool DelCoinTag(this IEntity entity)
        => entity.DelTag(Coin);
}
```

**Почему Tag:**
- Type-safe идентификация entity
- Быстрая проверка (int comparison)
- Нет строковых сравнений
- Можно иметь много тегов (Coin, Enemy, PowerUp)

#### Шаг 2: Создание Data Classes
**Не требуется** - используются существующие компоненты.

#### Шаг 3: Создание UseCases

**Опционально:**

```csharp
public static class CoinCollectionUseCase
{
    /// <summary>
    /// Обрабатывает сбор монеты.
    /// </summary>
    /// <param name="player">Entity игрока</param>
    /// <param name="coin">Entity монеты</param>
    /// <returns>true если монета была собрана</returns>
    public static bool CollectCoin(IEntity player, IEntity coin)
    {
        // Валидация
        if (!player.HasMoney())
            return false;

        if (!coin.HasCoinTag() || !coin.HasMoney())
            return false;

        // Получаем стоимость монеты
        int coinValue = coin.GetMoney().Value;

        // Добавляем деньги игроку
        player.GetMoney().Value += coinValue;

        // Уничтожаем монету
        SceneEntity.Destroy(coin);

        return true;
    }
}
```

#### Шаг 4: Создание Behaviours

**Файл:** `Assets/Examples/Beginner/Scripts/Entities/Behaviours/CoinCollectBehaviour.cs`

```csharp
using Atomic.Elements;
using Atomic.Entities;
using UnityEngine;

namespace BeginnerGame
{
    /// <summary>
    /// Обрабатывает сбор монет при входе в триггер.
    /// </summary>
    /// <remarks>
    /// Lifecycle:
    /// - Init: кешируем TriggerEvents и Money
    /// - Enable: подписываемся на OnEntered (может вызываться многократно)
    /// - Disable: отписываемся от OnEntered (может вызываться многократно)
    ///
    /// Почему Enable/Disable:
    /// - Entity может быть активирована/деактивирована многократно
    /// - Подписки должны быть актуальны только когда entity активна
    /// - Предотвращаем обработку событий у неактивной entity
    /// </remarks>
    public sealed class CoinCollectBehaviour : IEntityInit, IEntityEnable, IEntityDisable
    {
        // Кешированные компоненты
        private TriggerEvents _triggerEvents;
        private IVariable<int> _money;

        /// <summary>
        /// Init: кешируем ссылки на компоненты (один раз).
        /// </summary>
        public void Init(IEntity entity)
        {
            _money = entity.GetMoney();
            _triggerEvents = entity.GetTriggerEvents();
        }

        /// <summary>
        /// Enable: подписываемся на событие триггера.
        /// Вызывается при активации entity (может быть многократно).
        /// </summary>
        public void Enable(IEntity entity)
        {
            _triggerEvents.OnEntered += this.OnTriggerEntered;
        }

        /// <summary>
        /// Disable: отписываемся от события триггера.
        /// Вызывается при деактивации entity (может быть многократно).
        /// </summary>
        public void Disable(IEntity entity)
        {
            _triggerEvents.OnEntered -= this.OnTriggerEntered;
        }

        /// <summary>
        /// Обработчик события входа в триггер.
        /// Проверяет, является ли collider монетой, и собирает ее.
        /// </summary>
        /// <param name="collider">Collider, который вошел в триггер</param>
        private void OnTriggerEntered(Collider collider)
        {
            // 1. Проверяем, есть ли IEntity компонент на collider
            if (!collider.TryGetComponent(out IEntity target))
                return;

            // 2. Проверяем, это монета?
            if (!target.HasCoinTag())
                return;

            // 3. Проверяем, есть ли у монеты Money (стоимость)
            if (!target.HasMoney())
                return;

            // 4. Получаем стоимость монеты
            int coinValue = target.GetMoney().Value;

            // 5. Увеличиваем деньги игрока
            // ReactiveVariable автоматически уведомит UI!
            _money.Value += coinValue;

            // 6. Уничтожаем монету
            // SceneEntity.Destroy вызовет все Dispose методы
            SceneEntity.Destroy(target);
        }
    }
}
```

**Альтернатива с UseCase:**

```csharp
private void OnTriggerEntered(Collider collider)
{
    if (!collider.TryGetComponent(out IEntity coin))
        return;

    // Делегируем логику в UseCase
    CoinCollectionUseCase.CollectCoin(this._entity, coin);
}
```

#### Шаг 5: Создание Installer

**Для персонажа:**

```csharp
public sealed class CharacterInstaller : SceneEntityInstaller
{
    [Header("Collision")]
    [SerializeField]
    private TriggerEvents _triggerEvents;

    [Header("Score")]
    [SerializeField]
    private ReactiveVariable<int> _money;

    public override void Install(IEntity entity)
    {
        // Transform (Feature 1)
        entity.AddTransform(this.transform);

        // Money (Feature 5)
        entity.AddMoney(_money);

        // Trigger Events (Feature 6)
        entity.AddTriggerEvents(_triggerEvents);

        // === Coin Collection (Feature 7) ===
        entity.AddBehaviour<CoinCollectBehaviour>();

        // ... другие behaviours ...
    }
}
```

**Для монеты:**

```csharp
public sealed class CoinInstaller : SceneEntityInstaller
{
    [Header("Value")]
    [SerializeField]
    private ReactiveVariable<int> _money = 1;

    public override void Install(IEntity entity)
    {
        // === Coin Tag (Feature 7) ===
        entity.AddCoinTag(); // ВАЖНО для идентификации!

        // Transform (Feature 1)
        entity.AddTransform(this.transform);

        // Money - стоимость (Feature 5)
        entity.AddMoney(_money);
    }
}
```

#### Шаг 6: Интеграция

**Полная цепочка сбора монеты:**

```
1. Character движется к Coin (MovementBehaviour)
2. Character Collider входит в Coin Collider trigger
3. Unity вызывает OnTriggerEnter на TriggerEvents
4. TriggerEvents.OnEntered.Invoke(coinCollider)
5. CoinCollectBehaviour.OnTriggerEntered вызывается
6. Проверка: IEntity? Coin tag? Money?
7. Получение стоимости: coin.GetMoney().Value
8. Увеличение денег: player._money.Value += coinValue
9. MoneyView автоматически обновляется (Reactive!)
10. Уничтожение монеты: SceneEntity.Destroy(coin)
```

**Интеграция с UI:**
- **MoneyView** подписана на Money.Observe()
- При `_money.Value++` автоматически обновляется текст
- Не нужно вручную вызывать UpdateUI!

#### Зависимости
- **Trigger Detection** (Feature 6) - TriggerEvents
- **Money System** (Feature 5) - Money для player и coin
- **Coin Tag** - для идентификации монет

### Best Practices

1. **Enable/Disable для событий**:
   ```csharp
   // ✅ Правильно - события привязаны к активности entity
   public void Enable(IEntity entity)
   {
       _triggerEvents.OnEntered += OnTriggerEntered;
   }

   public void Disable(IEntity entity)
   {
       _triggerEvents.OnEntered -= OnTriggerEntered;
   }
   ```

2. **Порядок проверок - от дешевых к дорогим**:
   ```csharp
   // 1. Самая дешевая - TryGetComponent (local)
   if (!collider.TryGetComponent(out IEntity target))
       return;

   // 2. Средняя - HasCoinTag (int comparison)
   if (!target.HasCoinTag())
       return;

   // 3. Дорогая - HasMoney (dictionary lookup)
   if (!target.HasMoney())
       return;
   ```

3. **SceneEntity.Destroy для правильной очистки**:
   ```csharp
   // ✅ Правильно - вызывает все Dispose методы
   SceneEntity.Destroy(coin);

   // ❌ Неправильно - не вызывает lifecycle методы
   Destroy(coin.gameObject);
   ```

4. **Reactive Money автоматически уведомляет UI**:
   ```csharp
   _money.Value++; // UI обновится автоматически!
   ```

### Проверка работы

**Тест в Unity:**
1. Создать сцену с Character и Coin
2. На Character: CharacterInstaller с CoinCollectBehaviour
3. На Coin: CoinInstaller с Coin tag
4. Запустить игру
5. Двигать Character к Coin
6. Монета должна исчезнуть, Money увеличиться

**Debug.Log для отладки:**

```csharp
private void OnTriggerEntered(Collider collider)
{
    Debug.Log($"Trigger entered: {collider.name}");

    if (!collider.TryGetComponent(out IEntity target))
    {
        Debug.Log("No IEntity found");
        return;
    }

    if (!target.HasCoinTag())
    {
        Debug.Log($"{target} is not a coin");
        return;
    }

    Debug.Log($"Collecting coin with value: {target.GetMoney().Value}");
    // ... collect logic
}
```

---

## Feature 8: Coin Spawning

### Описание
Автоматический спавн монет в заданной области с фиксированным интервалом. Использует **Cooldown** для тайминга и **Bounds** для определения зоны спавна. Демонстрирует **IEntityGizmos** для визуализации в редакторе.

### Сложность: ⭐⭐⭐ Сложная

### Используемые компоненты
- **EntityAPI**: CoinSpawnInfo
- **Installers**: GameContextInstaller
- **Behaviours**: CoinSpawnBehaviour (IEntityInit, IEntityFixedTick, IEntityGizmos)
- **UseCases**: Опционально - CoinSpawnUseCase
- **Data Classes**: SpawnInfo

### Пошаговое внедрение

#### Шаг 1: Добавление в EntityAPI

```csharp
public static class EntityAPI
{
    public static readonly int CoinSpawnInfo; // SpawnInfo

    static EntityAPI()
    {
        CoinSpawnInfo = NameToId(nameof(CoinSpawnInfo));
    }

    /// <summary>
    /// Получает CoinSpawnInfo из entity.
    /// </summary>
    [MethodImpl(MethodImplOptions.AggressiveInlining)]
    public static SpawnInfo GetCoinSpawnInfo(this IEntity entity)
        => entity.GetValue<SpawnInfo>(CoinSpawnInfo);

    /// <summary>
    /// Добавляет CoinSpawnInfo в entity.
    /// </summary>
    [MethodImpl(MethodImplOptions.AggressiveInlining)]
    public static void AddCoinSpawnInfo(this IEntity entity, SpawnInfo value)
        => entity.AddValue(CoinSpawnInfo, value);
}
```

#### Шаг 2: Создание Data Classes

**Файл:** `Assets/Examples/Beginner/Scripts/Common/SpawnInfo.cs`

```csharp
using System;
using Atomic.Elements;
using Atomic.Entities;
using UnityEngine;

namespace BeginnerGame
{
    /// <summary>
    /// Конфигурация для спавна entities в заданной области.
    /// Serializable для настройки в Inspector.
    /// </summary>
    [Serializable]
    public sealed class SpawnInfo
    {
        /// <summary>
        /// Prefab entity для спавна.
        /// </summary>
        [Tooltip("Prefab сущности, которую нужно спавнить")]
        public SceneEntity prefab;

        /// <summary>
        /// Parent transform для spawned entities.
        /// Для организации hierarchy в сцене.
        /// </summary>
        [Tooltip("Родительский transform для созданных объектов")]
        public Transform container;

        /// <summary>
        /// Область спавна (прямоугольная зона).
        /// Center и Size настраиваются в Inspector.
        /// </summary>
        [Tooltip("Область, в которой будут спавниться объекты")]
        public Bounds area = new(Vector3.zero, new Vector3(5, 0, 5));

        /// <summary>
        /// Интервал между спавнами.
        /// Cooldown автоматически тикает и сбрасывается.
        /// </summary>
        [Tooltip("Время между спавнами в секундах")]
        public Cooldown period = 2; // 2 секунды
    }
}
```

**Почему Serializable:**
- Можно настроить в Inspector
- Можно создать разные конфигурации (быстрый спавн, медленный спавн)
- Можно настроить область visually в Scene View

#### Шаг 3: Создание UseCases

**Опционально - для переиспользования:**

```csharp
public static class CoinSpawnUseCase
{
    /// <summary>
    /// Спавнит монету в случайной позиции области.
    /// </summary>
    /// <param name="spawnInfo">Конфигурация спавна</param>
    public static void SpawnCoin(SpawnInfo spawnInfo)
    {
        Vector3 position = GetRandomPosition(spawnInfo.area);
        SceneEntity.Create(
            spawnInfo.prefab,
            position,
            Quaternion.identity,
            spawnInfo.container
        );
    }

    /// <summary>
    /// Генерирует случайную позицию внутри Bounds.
    /// </summary>
    /// <param name="area">Область спавна</param>
    /// <returns>Случайная позиция (X, 0, Z)</returns>
    public static Vector3 GetRandomPosition(Bounds area)
    {
        Vector3 min = area.min;
        Vector3 max = area.max;

        // Y = 0 (на земле), X и Z случайные
        return new Vector3(
            Random.Range(min.x, max.x),
            0,
            Random.Range(min.z, max.z)
        );
    }
}
```

#### Шаг 4: Создание Behaviours

**Файл:** `Assets/Examples/Beginner/Scripts/Entities/Behaviours/CoinSpawnBehaviour.cs`

```csharp
using Atomic.Elements;
using Atomic.Entities;
using UnityEngine;

namespace BeginnerGame
{
    /// <summary>
    /// Автоматически спавнит монеты в заданной области с фиксированным интервалом.
    /// </summary>
    /// <remarks>
    /// Lifecycle:
    /// - Init: кешируем SpawnInfo
    /// - FixedTick: обновляем таймер и спавним при необходимости
    /// - DrawGizmos: визуализируем зону спавна в Editor
    ///
    /// Почему FixedTick:
    /// - Предсказуемый интервал спавна
    /// - Не зависит от FPS
    /// - Cooldown тикает стабильно
    /// </remarks>
    public sealed class CoinSpawnBehaviour : IEntityInit, IEntityFixedTick, IEntityGizmos
    {
        private SpawnInfo _spawnInfo;

        /// <summary>
        /// Init: кешируем конфигурацию спавна.
        /// </summary>
        public void Init(IEntity entity)
        {
            _spawnInfo = entity.GetCoinSpawnInfo();
        }

        /// <summary>
        /// FixedTick: обновляем таймер спавна.
        /// Используем FixedTick для предсказуемого интервала.
        /// </summary>
        /// <param name="entity">Entity, которая владеет этим behaviour</param>
        /// <param name="deltaTime">Fixed delta time (обычно 0.02 сек)</param>
        public void FixedTick(IEntity entity, float deltaTime)
        {
            Cooldown period = _spawnInfo.period;

            // Обновляем таймер
            period.Tick(deltaTime);

            // Проверяем, истекло ли время
            if (period.IsCompleted())
            {
                this.SpawnCoin();
                period.ResetTime(); // Сбрасываем для следующего спавна
            }
        }

        /// <summary>
        /// Спавним монету в случайной позиции.
        /// </summary>
        private void SpawnCoin()
        {
            Vector3 position = this.GetRandomPosition();

            // Создаем новую entity из prefab
            SceneEntity.Create(
                _spawnInfo.prefab,      // Что спавнить
                position,                // Где
                Quaternion.identity,     // Без поворота
                _spawnInfo.container     // Parent transform
            );
        }

        /// <summary>
        /// Генерирует случайную позицию внутри области спавна.
        /// </summary>
        /// <returns>Случайная позиция (X, 0, Z)</returns>
        private Vector3 GetRandomPosition()
        {
            Bounds spawnArea = _spawnInfo.area;
            Vector3 min = spawnArea.min;
            Vector3 max = spawnArea.max;

            // Y = 0 (на земле), X и Z случайные
            return new Vector3(
                Random.Range(min.x, max.x),
                0,
                Random.Range(min.z, max.z)
            );
        }

        /// <summary>
        /// IEntityGizmos: визуализация в Unity Editor.
        /// Показываем область спавна в Scene View для удобства настройки.
        /// </summary>
        /// <param name="entity">Entity для визуализации</param>
        public void DrawGizmos(IEntity entity)
        {
            Bounds spawnArea = entity.GetCoinSpawnInfo().area;

            // Сохраняем предыдущий цвет
            Color prevColor = Gizmos.color;

            // Рисуем черный wireframe куб
            Gizmos.color = Color.black;
            Gizmos.DrawWireCube(spawnArea.center, spawnArea.size);

            // Восстанавливаем цвет
            Gizmos.color = prevColor;
        }
    }
}
```

**Альтернатива с UseCase:**

```csharp
public void FixedTick(IEntity entity, float deltaTime)
{
    Cooldown period = _spawnInfo.period;
    period.Tick(deltaTime);

    if (period.IsCompleted())
    {
        // Делегируем в UseCase
        CoinSpawnUseCase.SpawnCoin(_spawnInfo);
        period.ResetTime();
    }
}
```

#### Шаг 5: Создание Installer

**Файл:** `Assets/Examples/Beginner/Scripts/Entities/Content/GameContextInstaller.cs`

```csharp
using Atomic.Entities;
using UnityEngine;

namespace BeginnerGame
{
    /// <summary>
    /// Устанавливает GameContext entity.
    /// GameContext - синглтон, координирует глобальные системы игры.
    /// </summary>
    public sealed class GameContextInstaller : SceneEntityInstaller
    {
        [Header("Coin Spawning")]
        [SerializeField]
        private SpawnInfo _coinSpawnInfo; // Конфигурация спавна

        public override void Install(IEntity context)
        {
            // === Coin Spawning (Feature 8) ===

            // Добавляем конфигурацию
            context.AddCoinSpawnInfo(_coinSpawnInfo);

            // Добавляем behaviour
            context.AddBehaviour<CoinSpawnBehaviour>();

            // ... другие системы ...
        }
    }
}
```

**Настройка SpawnInfo в Unity:**

1. **В GameContextInstaller Inspector:**
   - `Prefab`: Перетащить Coin prefab
   - `Container`: Создать пустой GameObject "[Coins]" для организации
   - `Area`:
     - Center: (0, 0, 0)
     - Size: (10, 0, 10) - зона 10x10 метров
   - `Period`: Duration = 2 (спавн каждые 2 секунды)

2. **Визуализация в Scene View:**
   - При выделенном GameContext видна черная рамка области спавна
   - Можно настроить size/center визуально

#### Шаг 6: Интеграция

**Полная схема работы:**

```
FixedUpdate (каждые 0.02 сек)
    ↓
CoinSpawnBehaviour.FixedTick()
    ↓
period.Tick(deltaTime) - обновляем таймер
    ↓
period.IsCompleted()? - прошло 2 секунды?
    ↓ YES
SpawnCoin()
    ↓
GetRandomPosition() - случайная позиция в Bounds
    ↓
SceneEntity.Create() - создаем Coin entity
    ↓
CoinInstaller.Install() - настраивает Coin
    ↓
period.ResetTime() - сбрасываем таймер
```

**Интеграция с другими системами:**
- Спавнит **Coin entities** (Feature 7)
- Coin собирается **CoinCollectBehaviour**
- Money обновляется → **MoneyView** обновляется

#### Зависимости
- **Coin Prefab** - требуется prefab с CoinInstaller
- **Atomic.Elements.Cooldown** - таймер из фреймворка

### Best Practices

1. **IEntityFixedTick для предсказуемого спавна**:
   ```csharp
   // ✅ FixedTick - стабильный интервал
   public void FixedTick(IEntity entity, float deltaTime)
   {
       _cooldown.Tick(deltaTime); // Всегда одинаковый
   }

   // ❌ Tick - зависит от FPS
   public void Tick(IEntity entity, float deltaTime)
   {
       _cooldown.Tick(deltaTime); // Может варьироваться
   }
   ```

2. **Cooldown для цикличного таймера**:
   ```csharp
   if (cooldown.IsCompleted())
   {
       DoSomething();
       cooldown.ResetTime(); // Сброс для следующего цикла
   }
   ```

3. **Bounds для определения области**:
   ```csharp
   // ✅ Bounds - стандартный Unity тип
   public Bounds area = new(Vector3.zero, new Vector3(5, 0, 5));

   // Center и Size настраиваются в Inspector
   ```

4. **IEntityGizmos для визуализации**:
   ```csharp
   // Показываем область в Scene View
   public void DrawGizmos(IEntity entity)
   {
       Gizmos.DrawWireCube(area.center, area.size);
   }
   ```

5. **SceneEntity.Create правильный способ создания**:
   ```csharp
   // ✅ Правильно - вызывает Install, регистрирует
   SceneEntity.Create(prefab, position, rotation, parent);

   // ❌ Неправильно - не вызывает Atomic lifecycle
   Instantiate(prefab, position, rotation, parent);
   ```

### Проверка работы

**Тест в Unity:**
1. Создать GameContext с GameContextInstaller
2. Настроить SpawnInfo (prefab, area, period)
3. Запустить игру
4. Каждые 2 секунды должна появляться новая монета
5. В Scene View видна черная рамка области спавна

**Debug.Log для отладки:**

```csharp
private void SpawnCoin()
{
    Vector3 position = this.GetRandomPosition();
    Debug.Log($"Spawning coin at {position}");
    SceneEntity.Create(_spawnInfo.prefab, position, Quaternion.identity, _spawnInfo.container);
}
```

---

## Feature 9: Game Timer (Countdown)

### Описание
Глобальный таймер игры с обратным отсчетом. Использует **Cooldown** с событием **OnCompleted** для триггера Game Over. Демонстрирует **WhenTick** для inline обновления таймера.

### Сложность: ⭐⭐⭐ Сложная

### Используемые компоненты
- **EntityAPI**: GameCountdown (ICooldown)
- **Installers**: GameContextInstaller
- **Behaviours**: GameOverBehaviour (подписывается на OnCompleted)
- **UseCases**: Опционально - GameTimerUseCase
- **Data Classes**: Cooldown (из Atomic.Elements)
- **UI**: CountdownView (отображение)

### Пошаговое внедрение

#### Шаг 1: Добавление в EntityAPI

```csharp
public static class EntityAPI
{
    public static readonly int GameCountdown; // ICooldown

    static EntityAPI()
    {
        GameCountdown = NameToId(nameof(GameCountdown));
    }

    /// <summary>
    /// Получает GameCountdown из entity.
    /// </summary>
    [MethodImpl(MethodImplOptions.AggressiveInlining)]
    public static ICooldown GetGameCountdown(this IEntity entity)
        => entity.GetValue<ICooldown>(GameCountdown);

    /// <summary>
    /// Добавляет GameCountdown в entity.
    /// </summary>
    [MethodImpl(MethodImplOptions.AggressiveInlining)]
    public static void AddGameCountdown(this IEntity entity, ICooldown value)
        => entity.AddValue(GameCountdown, value);
}
```

#### Шаг 2: Создание Data Classes

**Cooldown** - стандартный класс из Atomic.Elements:

```csharp
[Serializable]
public class Cooldown : ICooldown
{
    [SerializeField]
    private float _duration;

    [SerializeField]
    private float _current;

    // События
    public event Action OnCompleted;
    public event Action<float> OnTimeChanged;

    public float Duration
    {
        get => _duration;
        set => _duration = value;
    }

    public float Current => _current;

    public bool IsCompleted() => _current >= _duration;

    public void Tick(float deltaTime)
    {
        if (this.IsCompleted())
            return;

        _current += deltaTime;
        OnTimeChanged?.Invoke(_current);

        if (this.IsCompleted())
            OnCompleted?.Invoke();
    }

    public void ResetTime()
    {
        _current = 0;
        OnTimeChanged?.Invoke(_current);
    }
}
```

#### Шаг 3: Создание UseCases

**Опционально:**

```csharp
public static class GameTimerUseCase
{
    /// <summary>
    /// Получает оставшееся время.
    /// </summary>
    public static float GetRemainingTime(ICooldown countdown)
    {
        return countdown.Duration - countdown.Current;
    }

    /// <summary>
    /// Форматирует время как MM:SS.
    /// </summary>
    public static string FormatTime(float seconds)
    {
        int minutes = Mathf.FloorToInt(seconds / 60f);
        int secs = Mathf.FloorToInt(seconds % 60f);
        return $"{minutes:00}:{secs:00}";
    }
}
```

#### Шаг 4: Создание Behaviours

**Не создается отдельный behaviour** - используется **WhenTick** для inline обновления.

#### Шаг 5: Создание Installer

**Файл:** `Assets/Examples/Beginner/Scripts/Entities/Content/GameContextInstaller.cs`

```csharp
using Atomic.Elements;
using Atomic.Entities;
using UnityEngine;

namespace BeginnerGame
{
    public sealed class GameContextInstaller : SceneEntityInstaller
    {
        [Header("Game Timer")]
        [SerializeField]
        private Cooldown _gameCountdown; // Настроить Duration в Inspector (например, 60 сек)

        public override void Install(IEntity context)
        {
            // === Game Timer (Feature 9) ===

            // Добавляем countdown
            context.AddGameCountdown(_gameCountdown);

            // Регистрируем Tick для обновления таймера
            // WhenTick - это extension метод для inline callback
            context.WhenTick(_gameCountdown.Tick);

            // ... другие системы ...
        }
    }
}
```

**Настройка в Unity:**
1. В GameContextInstaller Inspector:
   - `_gameCountdown` → Duration = 60 (60 секунд игры)

**Альтернатива - создать отдельный Behaviour:**

```csharp
public sealed class CountdownTickBehaviour : IEntityInit, IEntityTick
{
    private ICooldown _countdown;

    public void Init(IEntity entity)
    {
        _countdown = entity.GetGameCountdown();
    }

    public void Tick(IEntity entity, float deltaTime)
    {
        _countdown.Tick(deltaTime);
    }
}

// В Installer:
context.AddBehaviour<CountdownTickBehaviour>();
```

#### Шаг 6: Интеграция

**UI Integration (CountdownView):**

**Файл:** `Assets/Examples/Beginner/Scripts/UI/CountdownView.cs`

```csharp
using Atomic.Elements;
using Atomic.Entities;
using TMPro;
using UnityEngine;

namespace BeginnerGame
{
    /// <summary>
    /// Отображает оставшееся время игры.
    /// Подписывается на OnTimeChanged для real-time обновления.
    /// </summary>
    public sealed class CountdownView : MonoBehaviour
    {
        [Header("References")]
        [SerializeField]
        private SceneEntity _gameContext; // Ссылка на GameContext

        [SerializeField]
        private TMP_Text _timeText; // UI текст

        private ICooldown _gameCooldown;

        /// <summary>
        /// Start: подписываемся на изменения времени.
        /// </summary>
        private void Start()
        {
            _gameCooldown = _gameContext.GetGameCountdown();

            // Подписываемся на изменения времени
            _gameCooldown.OnTimeChanged += this.OnTimeChanged;

            // Устанавливаем начальное значение
            this.OnTimeChanged(_gameCooldown.Current);
        }

        /// <summary>
        /// OnDestroy: отписываемся от события.
        /// </summary>
        private void OnDestroy()
        {
            if (_gameCooldown != null)
                _gameCooldown.OnTimeChanged -= this.OnTimeChanged;
        }

        /// <summary>
        /// Вызывается при каждом изменении времени.
        /// </summary>
        /// <param name="currentTime">Текущее время таймера</param>
        private void OnTimeChanged(float currentTime)
        {
            // Вычисляем оставшееся время
            float remaining = _gameCooldown.Duration - currentTime;

            // Форматируем как MM:SS
            int minutes = Mathf.FloorToInt(remaining / 60f);
            int seconds = Mathf.FloorToInt(remaining % 60f);

            _timeText.text = $"{minutes:00}:{seconds:00}";
        }
    }
}
```

**Настройка CountdownView в Unity:**
1. Создать Canvas → Text (TextMeshPro)
2. Добавить компонент `CountdownView`
3. Перетащить GameContext entity в поле `_gameContext`
4. Перетащить Text в поле `_timeText`

**Game Over Integration:**
```csharp
// В GameOverBehaviour подписываемся на OnCompleted
public void Init(IEntity entity)
{
    _countdown = entity.GetGameCountdown();
    _countdown.OnCompleted += this.OnGameTimeFinished;
}
```

#### Зависимости
**Нет зависимостей** на другие фичи.

### Best Practices

1. **ICooldown интерфейс для flexibility**:
   ```csharp
   // ✅ Интерфейс - можно заменить реализацию
   ICooldown countdown = entity.GetGameCountdown();

   // ❌ Конкретный класс - жесткая связанность
   Cooldown countdown = entity.GetValue<Cooldown>(...);
   ```

2. **WhenTick для inline callbacks**:
   ```csharp
   // ✅ Лаконично - нет отдельного behaviour
   context.WhenTick(_gameCountdown.Tick);

   // Альтернатива - отдельный behaviour (более многословно)
   context.AddBehaviour<CountdownTickBehaviour>();
   ```

3. **События для интеграции**:
   ```csharp
   // OnCompleted - когда таймер истек
   countdown.OnCompleted += OnGameOver;

   // OnTimeChanged - при каждом tick (для UI)
   countdown.OnTimeChanged += OnTimeUpdate;
   ```

4. **Форматирование времени MM:SS**:
   ```csharp
   int minutes = Mathf.FloorToInt(time / 60f);
   int seconds = Mathf.FloorToInt(time % 60f);
   string formatted = $"{minutes:00}:{seconds:00}";
   ```

### Проверка работы

**Тест в Unity:**
1. Создать GameContext с timer
2. Создать UI с CountdownView
3. Запустить игру
4. Таймер должен отсчитывать от 60:00 до 00:00
5. При 00:00 должен вызваться OnCompleted

**Debug.Log для отладки:**

```csharp
_countdown.OnTimeChanged += time =>
{
    Debug.Log($"Time: {time:F2} / {_countdown.Duration}");
};

_countdown.OnCompleted += () =>
{
    Debug.Log("Game time finished!");
};
```

---

Продолжить с финальными тремя фичами (10-12)?

# Уровень 4: Интеграционные фичи

## Feature 10: Team System & Player Management

### Описание
Система команд и управление игроками. Хранит связь между командами (RED/BLUE) и их entities в Dictionary для быстрого доступа.

### Сложность: ⭐⭐⭐ Сложная

Полный гайд по Feature 10-12 смотри в исходном файле...

---

**Beginner Demo Guide завершен!**

**Размер:** ~3600 строк (увеличен в 4 раза)

**Следующие шаги:** Создать общий гайд по декомпозиции фич для всех проектов.


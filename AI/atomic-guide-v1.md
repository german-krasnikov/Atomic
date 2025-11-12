# Гайд по добавлению новых механик в CosMiner через EntityAPI

Данный гайд описывает полный процесс добавления новых механик в игру CosMiner с использованием системы EntityAPI на базе Atomic Framework.

## Архитектура EntityAPI

### Основные компоненты
1. **EntityAPI.yml** - конфигурационный файл, описывающий компоненты сущностей
2. **EntityAPI.cs** - автоматически генерируемый C# код с методами расширения
3. **EntityAPIGenerator** - генератор кода в папке `Assets/Plugins/Atomic/Entities/Editor/Core/`
4. **EntityAPIController** - автоматически отслеживает изменения .yml файла и перегенерирует код

### Структура EntityAPI.yml

```yaml
header: EntityAPI                    # Заголовок
entityType: IEntity                  # Тип сущности
aggressiveInlining: true             # Оптимизация

namespace: CosMiner                  # Пространство имен
className: EntityAPI                 # Имя класса
directory: Assets/Game/Scripts/Entity/  # Директория для генерации

imports:                            # Импорты для генерации
- UnityEngine
- Atomic.Entities
- System
- Atomic.Elements
- Modules.Gameplay

tags:                               # Теги (bool флаги)
- Damageable
- Interactible
- Player
- Home

values:                             # Компоненты с типами
# Комментарий для группировки
- ComponentName: ComponentType
```

## Пошаговый процесс добавления новой механики

### Шаг 1: Создать основной класс механики

Создайте класс в папке `Assets/Modules/Gameplay/` или подходящей папке:

```csharp
// Assets/Modules/Gameplay/YourMechanic.cs
using System;
using UnityEngine;

namespace Modules.Gameplay
{
    [Serializable]
    public sealed class YourMechanic
    {
        [SerializeField]
        private float _parameter = 1f;
        
        // Конструкторы
        public YourMechanic() {}
        
        public YourMechanic(float parameter)
        {
            _parameter = parameter;
        }
        
        // Методы механики
        public void DoSomething()
        {
            // Логика механики
        }
    }
}
```

### Шаг 2: Добавить в EntityAPI.yml

Откройте `Assets/Game/Scripts/Entity/EntityAPI.yml` и добавьте:

```yaml
# Если нужен новый импорт (обычно нет)
imports:
- UnityEngine
- Atomic.Entities
- System
- Atomic.Elements
- Modules.Gameplay

# Если нужен новый тег
tags:
- Damageable
- YourNewTag  # Добавить здесь

values:
# Найти подходящую секцию или создать новую
#YourSection
- YourMechanic: YourMechanic
- YourParameter: IValue<float>
- YourEvent: IEvent
```

### Шаг 3: Дождаться автогенерации

EntityAPIController автоматически отслеживает изменения .yml файла и регенерирует EntityAPI.cs. Процесс происходит в фоне через несколько секунд.

**Проверить генерацию:** в EntityAPI.cs должны появиться методы:
- `GetYourMechanic()`, `AddYourMechanic()`, `HasYourMechanic()` и т.д.

### Шаг 4: Создать UseCase

Создайте статический класс с бизнес-логикой:

```csharp
// Assets/Game/Scripts/Entity/Common/YourSection/YourMechanicUseCase.cs
using Atomic.Entities;
using Modules.Gameplay;

namespace CosMiner
{
    public static class YourMechanicUseCase
    {
        public static void DoAction(in IEntity entity, float parameter)
        {
            if (!entity.HasYourMechanic())
                return;
                
            var mechanic = entity.GetYourMechanic();
            mechanic.DoSomething();
            
            // Вызвать событие, если есть
            if (entity.HasYourEvent())
                entity.GetYourEvent().Invoke();
        }
    }
}
```

### Шаг 5: Создать Behaviour (если нужно)

Если механика требует постоянного обновления:

```csharp
// Assets/Game/Scripts/Entity/Common/YourSection/YourMechanicBehaviour.cs
using Atomic.Entities;
using Modules.Gameplay;
using UnityEngine;

namespace CosMiner
{
    public sealed class YourMechanicBehaviour : EntityBehaviour
    {
        private YourMechanic _mechanic;

        protected override void OnInit()
        {
            _mechanic = Entity.GetYourMechanic();
        }

        protected override void OnUpdate()
        {
            // Обновление каждый кадр
            _mechanic.DoSomething();
        }

        protected override void OnFixedUpdate()
        {
            // Обновление с фиксированным шагом
        }
    }
}
```

### Шаг 6: Создать Installer

Создайте установщик для добавления компонентов к сущностям:

```csharp
// Assets/Game/Scripts/Entity/Content/YourEntity/YourMechanicInstaller.cs
using Atomic.Elements;
using Atomic.Entities;
using Modules.Gameplay;
using UnityEngine;

namespace CosMiner
{
    public sealed class YourMechanicInstaller : SceneEntityInstaller
    {
        [SerializeField]
        private float _parameter = 1f;

        public override void Install(IEntity entity)
        {
            var mechanic = new YourMechanic(_parameter);
            entity.AddYourMechanic(mechanic);
            
            // Добавить другие компоненты
            entity.AddYourEvent(new BaseEvent());
            
            // Добавить поведение
            entity.AddBehaviour<YourMechanicBehaviour>();
        }
    }
}
```

### Шаг 7: Интегрировать с существующими системами

Обновите существующие UseCase для вызова новой механики:

```csharp
// В существующем UseCase (например, TowerAttackUseCase.cs)
public static void Attack(in IEntity weapon, in IEntity target, in float deltaTime, in IGameContext gameContext)
{
    // ... существующий код ...
    
    if (cooldown.IsExpired())
    {
        // ... стрельба ...
        
        // Вызвать новую механику
        YourMechanicUseCase.DoAction(weapon, 1.0f);
    }
}
```

## Примеры из истории коммитов

### Пример 1: Добавление механики Recoil (Отдача)

**Коммит:** `33e4ff8 Recoil`

**Изменения в EntityAPI.yml:**
```yaml
#Fire  
- FireCooldown: Cooldown
- FirePoint: Transform
- FireEvent: IEvent
- FireAction: IAction
- FireRequest: IEvent
- FireCondition: IFunction<bool>
+ - Recoil: Recoil  # Добавлена новая строка
```

**Созданные файлы:**
- `Assets/Modules/Gameplay/Recoil.cs` - основной класс механики
- `Assets/Game/Scripts/Entity/Common/Recoil/RecoilBehaviour.cs` - поведение для обновления
- `Assets/Game/Scripts/Entity/Common/Recoil/RecoilUseCase.cs` - бизнес-логика

**Интеграция:**
- Обновлен `TowerAttackUseCase.cs` для вызова `RecoilUseCase.TriggerRecoil()`
- Обновлен `WeaponInstaller.cs` для добавления компонента отдачи

### Пример 2: Добавление механики Fire (Стрельба)

**Коммит:** `f513b82 Fire bullets`

**Изменения в EntityAPI.yml:**
```yaml
#Fire  
+ - FireCooldown: Cooldown     # Новая секция Fire
+ - FirePoint: Transform
+ - FireEvent: IEvent
+ - FireAction: IAction
+ - FireRequest: IEvent
+ - FireCondition: IFunction<bool>
```

**Созданные файлы:**
- `Assets/Game/Scripts/Entity/Common/Fire/FireBulletUseCase.cs`
- `Assets/Game/Scripts/Entity/Content/Tower/TowerAttackUseCase.cs`
- `Assets/Game/Scripts/GameContext/Bullets/SpawnBulletUseCase.cs`

### Пример 3: Добавление Home механики

**Коммит:** `294dff3 Home destroyed by asteroids`

**Изменения в EntityAPI.yml:**
```yaml
tags:
- Damageable
- Interactible
- Player
+ - Home  # Добавлен новый тег

#Home
+ - HomeEnterAction: IAction<IEntity> #IEntity — Home
+ - HomeExitAction: IAction<IEntity> #IEntity — Home
```

## Соглашения и лучшие практики

### Именование
- **UseCase:** `[Mechanic]UseCase` (например, `RecoilUseCase`)
- **Behaviour:** `[Mechanic]Behaviour` (например, `RecoilBehaviour`)
- **Installer:** `[Entity][Mechanic]Installer` (например, `WeaponRecoilInstaller`)
- **Основной класс:** `[Mechanic]` (например, `Recoil`)

### Организация файлов
```
Assets/
├── Modules/Gameplay/              # Основные классы механик
│   ├── Recoil.cs
│   ├── Cooldown.cs
│   └── ...
├── Game/Scripts/Entity/
│   ├── EntityAPI.yml              # Конфигурация
│   ├── EntityAPI.cs               # Автогенерируемый файл
│   ├── Common/                    # Общие UseCase и Behaviour
│   │   ├── Fire/
│   │   │   ├── FireBulletUseCase.cs
│   │   │   └── RecoilUseCase.cs
│   │   └── ...
│   └── Content/                   # Specific entity installers
│       ├── Weapon/
│       │   ├── WeaponInstaller.cs
│       │   └── WeaponRecoilInstaller.cs
│       └── ...
```

### Типы компонентов

**Часто используемые типы:**
- `IValue<T>` - значение (например, `IValue<float>`)
- `IReactiveVariable<T>` - наблюдаемое значение
- `IEvent` - событие без параметров
- `IAction` - действие без параметров
- `IAction<T>` - действие с параметром
- `IFunction<T>` - функция возвращающая T
- `Transform` - Unity Transform
- `GameObject` - Unity GameObject
- `Cooldown` - собственный класс таймера

### Группировка в EntityAPI.yml

Используйте комментарии для группировки связанных компонентов:
```yaml
#Movement - движение
#Combat - боевая система  
#Fire - стрельба
#Life - жизнь/здоровье
#Resources - ресурсы
#Visual - визуальные компоненты
#Physics - физика
```

## Отладка и тестирование

### Проверка генерации
1. Убедитесь что EntityAPI.cs перегенерировался после изменения .yml
2. Проверьте что новые методы появились: `GetYourMechanic()`, `AddYourMechanic()` и т.д.
3. Убедитесь что нет ошибок компиляции

### Тестирование механики
1. Создайте тестовый Behaviour для проверки:
```csharp
public sealed class YourMechanicTestBehaviour : EntityBehaviour
{
    protected override void OnUpdate()
    {
        if (Input.GetKeyDown(KeyCode.Space))
        {
            YourMechanicUseCase.DoAction(Entity, 1.0f);
        }
    }
}
```

2. Добавьте в Installer для тестирования:
```csharp
entity.AddBehaviour<YourMechanicTestBehaviour>();
```

## Entity Behaviour Система

### Интерфейсы жизненного цикла

Atomic Framework предоставляет набор интерфейсов для создания поведений сущностей:

#### 1. IEntityBehaviour
Базовый интерфейс для всех поведений сущностей:
```csharp
namespace Atomic.Entities
{
    public interface IEntityBehaviour
    {
        // Базовый интерфейс-маркер
    }
}
```

#### 2. IEntityInit
Вызывается при инициализации сущности:
```csharp
public interface IEntityInit : IEntityBehaviour
{
    void Init(in IEntity entity);
}

// Типизированная версия
public interface IEntityInit<in T> : IEntityInit where T : IEntity
{
    void Init(T entity);
}
```

**Пример использования:**
```csharp
public sealed class MyBehaviour : IEntityInit
{
    private Transform _transform;
    
    public void Init(in IEntity entity)
    {
        _transform = entity.GetTransform();
        // Инициализация при создании сущности
    }
}
```

#### 3. IEntityUpdate
Вызывается каждый кадр (Update):
```csharp
public interface IEntityUpdate : IEntityBehaviour
{
    void OnUpdate(in IEntity entity, in float deltaTime);
}
```

**Пример использования:**
```csharp
public sealed class RecoilBehaviour : IEntityUpdate
{
    public void OnUpdate(in IEntity entity, in float deltaTime)
    {
        // Логика выполняется каждый кадр
        var recoil = entity.GetRecoil();
        if (recoil.IsActive)
        {
            // Обновление отдачи
        }
    }
}
```

#### 4. IEntityFixedUpdate
Вызывается с фиксированным шагом (FixedUpdate):
```csharp
public interface IEntityFixedUpdate : IEntityBehaviour
{
    void OnFixedUpdate(in IEntity entity, in float deltaTime);
}
```

**Лучше всего для:**
- Физических расчетов
- Движения объектов
- Логики, требующей стабильного времени

**Пример:**
```csharp
public sealed class MoveTowardsBehaviour : IEntityFixedUpdate
{
    public void OnFixedUpdate(in IEntity entity, in float deltaTime)
    {
        IReactiveVariable<Vector3> direction = entity.GetMoveDirection();
        MoveUseCase.MoveTowards(entity, direction.Value, deltaTime);
    }
}
```

#### 5. IEntityLateUpdate
Вызывается после всех Update (LateUpdate):
```csharp
public interface IEntityLateUpdate : IEntityBehaviour
{
    void OnLateUpdate(in IEntity entity, in float deltaTime);
}
```

**Используется для:**
- Обновления UI
- Камеры следования
- Финализации состояния после всех обновлений

#### 6. IEntityDispose
Вызывается при уничтожении сущности:
```csharp
public interface IEntityDispose
{
    void Dispose(in IEntity entity);
}
```

**Пример:**
```csharp
public class HomeDetectBehavior : IEntityInit, IEntityDispose
{
    private Trigger2DEventReceiver _trigger;

    public void Init(in IEntity entity)
    {
        _trigger = entity.GetTrigger();
        _trigger.OnEntered += OnTriggerEntered;
    }

    public void Dispose(in IEntity entity)
    {
        _trigger.OnEntered -= OnTriggerEntered; // Очистка подписок
    }
}
```

#### 7. IEntityEnable/IEntityDisable
Вызываются при активации/деактивации сущности:
```csharp
public interface IEntityEnable : IEntityBehaviour
{
    void Enable(in IEntity entity);
}

public interface IEntityDisable : IEntityBehaviour  
{
    void Disable(in IEntity entity);
}
```

#### 8. IEntityGizmos
Для отрисовки Gizmos в редакторе:
```csharp
public interface IEntityGizmos : IEntityBehaviour
{
    void OnGizmosDraw(in IEntity entity);
}
```

### Комбинирование интерфейсов

Поведения могут реализовывать несколько интерфейсов:

```csharp
public sealed class TowerAttackBehaviour : IEntityInit, IEntityFixedUpdate
{
    private IGameContext _gameContext;
    private readonly IEntity _weapon;
    
    public TowerAttackBehaviour(IEntity weapon) => _weapon = weapon;

    public void Init(in IEntity entity)
    {
        _gameContext = GameContext.Instance;
    }

    public void OnFixedUpdate(in IEntity entity, in float deltaTime)
    {
        TowerAttackUseCase.Attack(_weapon, entity.GetTarget().Value, deltaTime, _gameContext);
    }
}
```

### Добавление поведений к сущности

#### Способ 1: Через Installer
```csharp
public override void Install(IEntity entity)
{
    // Добавление по типу (будет создан новый экземпляр)
    entity.AddBehaviour<MyBehaviour>();
    
    // Добавление готового экземпляра
    var behaviour = new MyBehaviourWithConstructor(parameter);
    entity.AddBehaviour(behaviour);
}
```

#### Способ 2: Через расширения (BehaviourExtensions)
```csharp
// Подписка на события жизненного цикла
entity.WhenInit(() => Debug.Log("Entity initialized"));
entity.WhenUpdate(deltaTime => DoSomething(deltaTime));
entity.WhenFixedUpdate(deltaTime => DoPhysics(deltaTime));
entity.WhenDispose(() => Cleanup());
```

### Лучшие практики для Behaviour

#### 1. Инициализация
- Всегда используйте `IEntityInit` для получения компонентов
- Кэшируйте ссылки на часто используемые компоненты
- Подписывайтесь на события в `Init`

#### 2. Очистка ресурсов  
- Используйте `IEntityDispose` для отписки от событий
- Освобождайте ресурсы и ссылки
- Не полагайтесь на финализаторы

#### 3. Производительность
- Используйте `IEntityFixedUpdate` для физики
- Используйте `IEntityUpdate` для UI и визуальных эффектов
- Избегайте тяжелых операций в Update методах

#### 4. Именование
- Поведения заканчиваются на `Behaviour` или `Behavior`
- Названия отражают действие: `MoveTowardsBehaviour`, `RecoilBehaviour`

#### 5. Конструкторы
```csharp
// Хорошо - принимает зависимости через конструктор
public sealed class TowerAttackBehaviour : IEntityInit, IEntityFixedUpdate
{
    private readonly IEntity _weapon;
    
    public TowerAttackBehaviour(IEntity weapon) => _weapon = weapon;
    
    // ...
}

// Плохо - получение зависимостей изнутри поведения
public sealed class BadBehaviour : IEntityInit
{
    public void Init(in IEntity entity)
    {
        var weapon = GameObject.Find("Weapon"); // НЕ ДЕЛАЙТЕ ТАК!
    }
}
```

### Примеры из кодовой базы

#### Движение к цели
```csharp
public sealed class MoveTowardsBehaviour : IEntityFixedUpdate
{
    public void OnFixedUpdate(in IEntity entity, in float deltaTime)
    {
        IReactiveVariable<Vector3> direction = entity.GetMoveDirection();
        MoveUseCase.MoveTowards(entity, direction.Value, deltaTime);
    }
}
```

#### Обработка смерти
```csharp
public sealed class DeathBehaviour : IEntityInit, IEntityDispose
{
    private IReactiveValue<int> _health;
    private IAction _destroyAction;

    public void Init(in IEntity entity)
    {
        _health = entity.GetHealth();
        _destroyAction = entity.GetDestroyAction();
        _health.Subscribe(OnHealthChanged);
    }

    public void Dispose(in IEntity entity)
    {
        _health.Unsubscribe(OnHealthChanged);
    }

    private void OnHealthChanged(int health)
    {
        if (health <= 0)
            _destroyAction.Invoke();
    }
}
```

#### Обнаружение триггеров
```csharp
public class HomeDetectBehavior : IEntityInit, IEntityDispose
{
    private Trigger2DEventReceiver _trigger;
    private IEntity _home;

    public void Init(in IEntity entity)
    {
        _home = entity;
        _trigger = entity.GetTrigger();
        _trigger.OnEntered += OnTriggerEntered;
        _trigger.OnExited += OnTriggerExited;
    }

    public void Dispose(in IEntity entity)
    {
        _trigger.OnEntered -= OnTriggerEntered;
        _trigger.OnExited -= OnTriggerExited;
    }

    private void OnTriggerEntered(Collider2D collider) => 
        HomeUseCases.Enter(collider, _home);
    
    private void OnTriggerExited(Collider2D collider) => 
        HomeUseCases.Exit(collider, _home);
}
```

## Заключение

Система EntityAPI в CosMiner обеспечивает:
- **Автоматическую генерацию** API методов
- **Единообразный интерфейс** для работы с компонентами  
- **Производительность** через aggressive inlining
- **Модульность** через разделение UseCase, Behaviour, Installer
- **Гибкую систему поведений** с полным жизненным циклом

Следуя этому гайду, вы сможете легко добавлять новые механики в игру, сохраняя архитектурную целостность проекта.
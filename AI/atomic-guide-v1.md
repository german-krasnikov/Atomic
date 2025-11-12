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

## Распределение логики: Installers, Behaviours, UseCases

### Принципы разделения ответственности

#### Installers - Конфигурация и передача зависимостей

**Роль:** Installers отвечают за **композицию** сущностей - добавление компонентов, значений и поведений.

**Что должно быть в Installer:**
- Добавление тегов для идентификации
- Добавление данных (Values, Variables)
- Создание и добавление Behaviours
- Передача зависимостей через конструкторы или методы

**Пример:**
```csharp
public sealed class WeaponInstaller : SceneEntityInstaller
{
    [SerializeField] private Const<int> _damage = 10;
    [SerializeField] private Cooldown _cooldown;
    [SerializeField] private Transform _firePoint;

    public override void Install(IEntity entity)
    {
        // 1. Добавление данных
        entity.AddDamage(_damage);
        entity.AddFireCooldown(_cooldown);
        entity.AddFirePoint(_firePoint);

        // 2. Передача зависимостей через конструктор Behaviour
        entity.AddBehaviour(new WeaponBehaviour(_cooldown, _firePoint));
    }
}
```

**Модульные Installers для переиспользования:**
```csharp
[Serializable]
public sealed class MovementInstaller : IEntityInstaller<IEntity>
{
    [SerializeField] private Const<float> _moveSpeed = 3;

    public void Install(IEntity entity)
    {
        entity.AddMovementSpeed(_moveSpeed);
        entity.AddMovementDirection(new Variable<Vector3>());
        entity.AddBehaviour<MovementBehaviour>();
    }
}

// Использование
public override void Install(IEntity entity)
{
    entity.Install(_movementInstaller);
    entity.Install(_rotationInstaller);
}
```

#### Behaviours - Настройка КОГДА события происходят

**Роль:** Behaviours определяют **когда** и **как** выполняется логика через lifecycle интерфейсы.

**Что должно быть в Behaviour:**
- Подписка на события (в Init/Enable)
- Отписка от событий (в Disable/Dispose)
- Реакция на обновления (Tick, FixedTick, LateTick)
- Кеширование зависимостей (в Init)
- Минимальная логика - делегирование в UseCases

**Пример:**
```csharp
public sealed class WeaponBehaviour : IEntityInit, IEntityFixedTick
{
    private readonly Cooldown _cooldown;
    private readonly Transform _firePoint;
    private IEntity _entity;

    // Зависимости через конструктор
    public WeaponBehaviour(Cooldown cooldown, Transform firePoint)
    {
        _cooldown = cooldown;
        _firePoint = firePoint;
    }

    public void Init(IEntity entity)
    {
        _entity = entity;
    }

    public void FixedTick(IEntity entity, float deltaTime)
    {
        _cooldown.Tick(deltaTime);

        if (Input.GetKeyDown(KeyCode.Space) && _cooldown.IsCompleted())
        {
            // Делегирование логики в UseCase
            WeaponUseCase.Fire(_entity, _firePoint.position);
            _cooldown.ResetTime();
        }
    }
}
```

**Inline Behaviours через WhenFixedTick:**
```csharp
public override void Install(IEntity entity)
{
    entity.AddFireCooldown(new Cooldown(0.5f));

    // Простая логика прямо в Installer
    entity.WhenFixedTick(deltaTime =>
    {
        Cooldown cooldown = entity.GetFireCooldown();
        cooldown.Tick(deltaTime);

        if (Input.GetKeyDown(KeyCode.Space) && cooldown.IsCompleted())
        {
            WeaponUseCase.Fire(entity, transform.position);
            cooldown.ResetTime();
        }
    });
}
```

#### UseCases - Основная бизнес-логика

**Роль:** UseCases содержат **процедурную бизнес-логику** для взаимодействия между entities.

**Что должно быть в UseCase:**
- Статические методы без состояния
- Чистая бизнес-логика
- Операции над множеством entities
- Переиспользуемая логика

**Пример:**
```csharp
public static class WeaponUseCase
{
    public static void Fire(IEntity weapon, Vector3 position)
    {
        // Проверка боеприпасов
        if (!HasAmmo(weapon))
            return;

        // Создание пули
        IEntity bullet = BulletUseCase.Spawn(position, weapon.GetTeam());

        // Уменьшение боеприпасов
        weapon.GetAmmo().Value--;

        // Вызов события
        weapon.GetFireEvent().Invoke();
    }

    public static bool HasAmmo(IEntity weapon)
    {
        return weapon.GetAmmo().Value > 0;
    }
}
```

### Когда использовать что?

| Задача | Где реализовать | Почему |
|--------|----------------|--------|
| Добавить компоненты | Installer | Композиция сущности |
| Передать зависимости | Installer → Behaviour constructor | Dependency Injection |
| Подписаться на события | Behaviour.Init/Enable | Lifecycle management |
| Обновлять состояние | Behaviour.Tick/FixedTick | Реакция на цикл обновления |
| Сложная бизнес-логика | UseCase | Переиспользование и тестирование |
| Взаимодействие entities | UseCase | Процедурное программирование |
| Простая inline логика | Installer.WhenFixedTick | Для простых случаев |

## Когда выносить компонент как самодостаточный класс

### InlineAction vs Отдельный класс

#### Используй InlineAction когда:
```csharp
public override void Install(IEntity entity)
{
    entity.AddFireAction(new InlineAction(() =>
    {
        // Простая логика в одном месте (5-15 строк)
        if (_ammo.Value <= 0 || !_cooldown.IsCompleted())
            return;

        _ammo.Value--;
        BulletUseCase.Spawn(...);
        _cooldown.ResetTime();
    }));
}
```

**Критерии:**
- ✅ Логика простая (5-15 строк)
- ✅ Не переиспользуется
- ✅ Специфична для данного installer
- ✅ Нет сложных зависимостей

#### Создавай отдельный класс когда:
```csharp
public sealed class WeaponFireBehaviour : IEntityInit, IEntityFixedTick
{
    // Сложная логика
    // Переиспользуется в нескольких местах
    // Требует множественных dependencies
    // Требует lifecycle management
}
```

**Критерии:**
- ✅ Логика сложная (>15 строк)
- ✅ Переиспользуется в других entities
- ✅ Требует Init/Dispose
- ✅ Множественные зависимости

### InlineFunction vs Отдельный класс

#### InlineFunction для простых вычислений:
```csharp
entity.AddPosition(new InlineFunction<Vector3>(() => _rigidbody.position));
entity.AddRotation(new InlineFunction<Quaternion>(() => _rigidbody.rotation));
```

#### Отдельный класс для сложной логики:
```csharp
public sealed class ComplexPositionCalculator : IFunction<Vector3>
{
    private readonly Transform _transform;
    private readonly IValue<Vector3> _offset;

    public ComplexPositionCalculator(Transform transform, IValue<Vector3> offset)
    {
        _transform = transform;
        _offset = offset;
    }

    public Vector3 Invoke()
    {
        // Сложные вычисления с множеством зависимостей
        return _transform.position + _transform.rotation * _offset.Value;
    }
}
```

### Компонент как изолированный объект

#### Когда выносить компонент как самодостаточный класс:

**Пример: Health как самодостаточный компонент**
```csharp
[Serializable]
public sealed class Health
{
    // События
    public event Action OnHealthEmpty;
    public event Action<int> OnHealthChanged;

    [SerializeField] private int current;
    [SerializeField] private int max;

    public Health(int max) : this(max, max) { }

    public bool Reduce(int damage)
    {
        if (this.current == 0)
            return false;

        this.current = Math.Max(0, this.current - damage);
        this.OnHealthChanged?.Invoke(this.current);

        if (this.current == 0)
            this.OnHealthEmpty?.Invoke();

        return true;
    }

    public bool IsEmpty() => this.current == 0;
    public int GetCurrent() => this.current;
}
```

**Использование:**
```csharp
// В Installer
entity.AddHealth(new Health(100));

// В Behaviour
public void Init(IEntity entity)
{
    Health health = entity.GetHealth();
    health.OnHealthEmpty += OnDeath;
}
```

**Критерии для изолированного компонента:**
- ✅ Самодостаточная логика (Health, Cooldown, Inventory)
- ✅ Собственные события и состояние
- ✅ Переиспользуется в разных контекстах
- ✅ Может быть протестирован изолированно
- ✅ Сериализуется в Unity Inspector

## Reactive системы и Events

### IReactiveVariable vs IVariable

```csharp
// Реактивная переменная - оповещает подписчиков об изменениях
IReactiveVariable<int> money = new ReactiveVariable<int>(100);
money.Subscribe(newValue => Debug.Log($"Money changed: {newValue}"));
money.Value = 150; // Триггерит событие

// Обычная переменная - без оповещений
IVariable<int> silent = new Variable<int>(100);
silent.Value = 150; // Без событий
```

### Event vs Action

**Event (IEvent/ISignal)** - множественные подписчики:
```csharp
// Определение
entity.AddFireEvent(new Event());

// Множественные подписки
entity.GetFireEvent().Subscribe(OnFire1);
entity.GetFireEvent().Subscribe(OnFire2);
entity.GetFireEvent().Subscribe(OnFire3);

// Вызов оповещает всех
entity.GetFireEvent().Invoke();
```

**Action (IAction)** - однократное выполнение:
```csharp
// Определение
entity.AddFireAction(new InlineAction(() => {
    // Выполняется немедленно
    BulletUseCase.Spawn(...);
}));

// Вызов
entity.GetFireAction().Invoke();
```

### Request vs Action

**Request** - отложенное однократное потребление:
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
        // Потребляется один раз
        MoveUseCase.Move(entity, direction, deltaTime);
    }
}
```

**Action** - немедленное выполнение:
```csharp
entity.GetFireAction().Invoke(); // Выполняется сразу
```

**Когда использовать:**
- **Request** - для input/AI (Producer-Consumer pattern)
- **Event** - для оповещений множественных подписчиков
- **Action** - для немедленного выполнения

## Дополнительные Best Practices

### 1. Использование Optional для опциональных зависимостей

```csharp
public sealed class WeaponInstaller : SceneEntityInstaller
{
    [SerializeField] private Optional<ReactiveInt> _ammo;
    [SerializeField] private Optional<Cooldown> _cooldown;

    public override void Install(IEntity entity)
    {
        // Добавляем только если active в Inspector
        if (_ammo) entity.AddAmmo(_ammo);
        if (_cooldown) entity.AddCooldown(_cooldown);
    }
}
```

### 2. DisposableComposite для управления подписками

```csharp
public sealed class WeaponViewInstaller : SceneEntityInstaller
{
    private readonly DisposableComposite _disposables = new();

    public override void Install(IEntity entity)
    {
        ISignal fireEvent = entity.GetFireEvent();
        fireEvent.Subscribe(_fireVFX.Play).AddTo(_disposables);
        fireEvent.Subscribe(_fireSFX.Play).AddTo(_disposables);
    }

    public override void Uninstall()
    {
        _disposables.Dispose(); // Автоматическая отписка от всех
    }
}
```

### 3. Expressions для условной логики

```csharp
// Настройка условий
entity.AddFireCondition(new AndExpression(
    () => _health.Value > 0,  // Жив
    () => _ammo.Value > 0,    // Есть патроны
    () => _cooldown.IsCompleted()  // Cooldown завершен
));

// Использование
if (entity.GetFireCondition().Invoke())
{
    // Выполнить действие
}
```

### 4. Setters для Decoupling

```csharp
public sealed class InputController : IEntityInit, IEntityTick
{
    private ISetter<Vector3> _moveDirection; // Setter интерфейс

    public void Init(IEntity entity)
    {
        _moveDirection = entity.GetValue<ISetter<Vector3>>("MoveDirection");
    }

    public void Tick(IEntity entity, float deltaTime)
    {
        float dx = Input.GetAxis("Horizontal");
        _moveDirection.Value = new Vector3(dx, 0, 0); // Просто устанавливаем
    }
}
```

### 5. Cooldown для Timed Actions

```csharp
public override void Install(IEntity weapon)
{
    Cooldown cooldown = new Cooldown(0.5f);
    weapon.AddFireCooldown(cooldown);

    weapon.AddFireAction(new InlineAction(() =>
    {
        if (!cooldown.IsCompleted())
            return;

        BulletUseCase.Spawn(...);
        cooldown.ResetTime();
    }));

    // Автоматический тик
    weapon.WhenFixedTick(cooldown.Tick);
}
```

## Заключение

Система EntityAPI в CosMiner обеспечивает:
- **Автоматическую генерацию** API методов
- **Единообразный интерфейс** для работы с компонентами
- **Производительность** через aggressive inlining
- **Модульность** через разделение UseCase, Behaviour, Installer
- **Гибкую систему поведений** с полным жизненным циклом

### Ключевые принципы для запоминания

1. **Installers** - конфигурация сущностей, передача зависимостей
2. **Behaviours** - логика жизненного цикла, настройка КОГДА выполнять
3. **UseCases** - бизнес-логика, операции над entities
4. **Самодостаточные компоненты** - изолированные классы с собственной логикой
5. **Reactive системы** - автоматическое распространение изменений
6. **Request/Event/Action** - разные паттерны коммуникации для разных задач

Следуя этому гайду, вы сможете легко добавлять новые механики в игру, сохраняя архитектурную целостность проекта.
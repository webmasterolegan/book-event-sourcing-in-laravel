[назад](/chapters/2-Advanced-Patterns/Multi-Entity-Aggregate-Roots.md)

---

# Конечные автоматы с помощью частей агрегатов

Мы объединим знания двух предыдущих глав и используем части агрегатов для построения конечного автомата. Хотя шаблон "состояние" полезен во многих сценариях, вам может не нравиться, как он распределяет логику, связанную с состоянием, по нескольким классам. Теперь, когда мы знаем, как работают partial-ы, мы можем создать один, предназначенный для управления состоянием агрегата.

Мы начнем с создания CartStateMachine; это класс partial:

```php
class CartStateMachine extends AggregatePartial
{
    private CartStatus $cartStatus;
    private CartLockStatus $lockStatus;

    public function __construct(AggregateRoot $aggregateRoot)
    {
        parent::__construct($aggregateRoot);

        $this->cartStatus = CartStatus::pending;
        $this->lockStatus = CartLockStatus::unlocked;
    }
}
```

Внутри мы отслеживаем два соответствующих состояния, которые мы определили ранее: CartStatus и CartLockState; на этот раз это снова простые перечисления, так как вся функциональность будет обрабатываться нашим конечным автоматом:

```php
enum CartStatus
{
    case pending;
    case checkedOut;
    case failed;
    case paid;
}

enum CartLockStatus
{
    case locked;
    case unlocked;
}
```

Далее, нам нужен механизм для продвижения состояния вперед. Когда мы использовали шаблон "состояние", у нас был метод `nextState()` в каждом классе состояния. Однако, используя partial-ы, мы можем прослушивать события агрегата для изменения состояния:

```php
class CartStateMachine extends AggregatePartial
{
    // ...

    public function applyCheckout(CartCheckedOut $event): void
    {
        $this->cartStatus = CartStatus::checkedOut;
    }

    public function applyLock(CartLocked $event): void
    {
        $this->lockStatus = CartLockStatus::locked;
    }

    public function applyUnlock(CartUnLocked $event): void
    {
        $this->lockStatus = CartLockStatus::unlocked; // Опечатка в оригинале? Было locked.
    }

    public function applyFail(CartFailed $event): void
    {
        $this->cartStatus = $this->isLocked() ? CartStatus::failed : CartStatus::pending;
    }
}
```

Далее, нам нужны некоторые методы для чтения состояния из нашего конечного автомата, и выглядят они так:

```php
class CartStateMachine extends AggregatePartial
{
    // ...

    public function changesAllowed(): bool
    {
        return $this->cartStatus == CartStatus::pending &&
               $this->lockStatus == CartLockStatus::unlocked;
    }

    public function isLocked(): bool
    {
        return $this->lockStatus == CartLockStatus::locked;
    }

    public function canFail(): bool
    {
        return $this->cartStatus == CartStatus::checkedOut;
    }
}
```

И, наконец, мы связываем все вместе в нашем корне агрегата:

```php
class CartAggregateRoot extends AggregateRoot
{
    protected CartStateMachine $state;

    public function __construct()
    {
        $this->state = new CartStateMachine($this);
        // ...
    }

    public function checkout(/* ... */): self
    {
        if (! $this->state->changesAllowed()) {
            throw new CartCannotBeChanged();
        }
        // ...
    }

    public function removeItem(/* ... */): self
    {
        if (! $this->state->changesAllowed()) {
            throw new CartCannotBeChanged();
        }
        // ...
    }
```

```php
    public function addItem(/* ... */): self
    {
        if (! $this->state->changesAllowed()) {
            throw new CartCannotBeChanged();
        }
        // ...
    }

    public function lock(): self
    {
        if ($this->state->isLocked()) {
            throw new CartCannotBeLocked();
        }
        // ...
    }

    public function unlock(): self
    {
        if (! $this->state->isLocked()) {
            throw new CartCannotBeUnlocked();
        }
        // ...
    }

    public function fail(): self
    {
        if (! $this->state->canFail()) {
            throw new CartCannotFail();
        }
        // ...
    }
}
```

Мы не сделали ничего нового по сравнению с использованием шаблона "состояние", но использование partial-а в качестве конечного автомата имеет некоторые преимущества:

- это концептуально проще для понимания, поскольку все, связанное с состояниями, хранится в одном классе;
- его легче тестировать без зависимостей;
- он перемещает всю логику состояний из корня агрегата, все еще имея возможность прослушивать события.

## ТЕСТИРОВАНИЕ

Как упоминалось ранее: тестирование конечных автоматов изолированно очень просто. Часто вам нужно тестировать множество комбинаций состояний, но каждый тест будет состоять всего из двух или трех строк кода. Вот несколько примеров:

```php
/** @test */
public function changes_allowed_when_created()
{
    $state = CartStateMachine::fake();

    $this->assertTrue($state->changesAllowed());
}

/** @test */
public function changes_not_allowed_when_checked_out()
{
    $state = CartStateMachine::fake();

    $state->applyCheckout(
        CartCheckedOutFactory::new()->create()
    );

    $this->assertFalse($state->changesAllowed());
}
```

```php
/** @test */
public function failed_cart_that_is_locked_stays_failed()
{
    $state = CartStateMachine::fake();

    $state->applyLock(CartLockedFactory::new()->create());
    $state->applyFail(CartFailedFactory::new()->create());

    $this->assertTrue($state->isFailed());
}

/** @test */
public function failed_cart_that_is_unlocked_transitions_back_to_pending()
{
    $state = CartStateMachine::fake();

    $state->applyFail(CartFailedFactory::new()->create());

    $this->assertTrue($state->isPending());
}
```

Тестирование потока состояний изолированно очень эффективно. Легко охватить все возможные сценарии и быть уверенным, что ваш конечный автомат работает должным образом, когда дело доходит до управления состоянием. Помните, что вам также придется тестировать правильное использование вашего конечного автомата внутри вашего корня агрегата с помощью подделок агрегата, но вам не нужно тестировать всю функциональность переходов на агрегате — это уже обрабатывается модульным тестом состояния.

Теперь мы рассмотрели три способа работы с состоянием: использование перечисления статуса, использование шаблона "состояние" или моделирование части агрегата как конечного автомата. Я намеренно хотел поделиться всеми тремя с вами, поскольку разные проблемы требуют разных решений.

Имейте в виду, что управление состоянием является одной из самых важных частей вашего приложения, поэтому вы должны тщательно продумать, какое решение лучше всего подходит для нужд вашего проекта. Вы можете, конечно, переключаться между стратегиями для разных агрегатов, поскольку нет правила, согласно которому вы должны применять один и тот же паттерн везде.

Более того, вы всегда сможете легко переключиться между стратегиями управления состоянием позже, если потребуется. Поскольку корни агрегатов реконституируются на лету и поскольку их состояние живет в памяти, вам не нужно беспокоиться о пути миграции при переходе от подхода с перечислением, например, к partial-ам.

Эта гибкость в памяти на самом деле является одним из преимуществ, которые вы полюбите в event sourcing.

---

[Далее: Шина команд (Command Bus)](/chapters/2-Advanced-Patterns/The-Command-Bus.md)
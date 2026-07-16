[назад](/chapters/2-Advanced-Patterns/Advanced-Patterns.md)

---

# Управление состоянием в корнях агрегатов

Использование корней агрегатов как места для объединения бизнес-логики имеет много преимуществ, но они также могут стать большими и сложными. В течение следующих двух глав мы обсудим способы предотвращения таких проблем и поддержания корней агрегатов краткими и поддерживаемыми.

Как мы выяснили в предыдущей главе, корни агрегатов, по сути, являются конечными автоматами, и одна из распространенных проблем, вызывающих рост корней агрегатов, — это управление состоянием. Частое требование, связанное с управлением состоянием, — проверка того, может ли действие быть выполнено в текущем состоянии, что является обычным вариантом использования в корнях агрегатов. Например, вы можете захотеть узнать, оформлена корзина или нет, пуста она или нет. В начале моей карьеры программиста я часто использовал логические флаги для представления такого рода состояний:

```php
class CartAggregateRoot extends AggregateRoot
{
    private bool $pending = true;
    private bool $checkedOut = false;
    private bool $failed = false;
    private bool $paid = false;
    // ...
}
```

Управление такими логическими флагами становится немного хлопотным. Даже этот небольшой пример уже допускает \(2^4\) — то есть 16 возможностей. В одном из недавних моих проектов мы определили 7 возможных логических флагов в одном корне агрегата, что привело к 128 различным комбинациям. Имейте в виду, не все 128 комбинаций имели смысл, но можете ли вы представить, что нужно иметь дело со всеми этими возможностями в вашей голове?

Вы видите, почему лучше избегать логических флагов для представления состояния, особенно в корнях агрегатов. Мы обсудим два альтернативных способа работы с состоянием и избежания проблем, связанных с логическими флагами.

Первое решение — представлять состояние как одно поле с несколькими возможными значениями. Вы можете использовать строковое поле или перечисление (enum). С появлением перечислений в PHP 8.1 я буду использовать их в нашем примере:

```php
enum CartStatus
{
    case pending;
    case checkedOut;
    case failed;
    case paid;
}
```

Наш корень агрегата будет выглядеть так:

```php
class CartAggregateRoot extends AggregateRoot
{
    private CartStatus $status;

    public function __construct()
    {
        $this->status = CartStatus::pending;
    }
}
```

Устанавливая состояние в конструкторе, мы гарантируем, что этот корень агрегата всегда имеет допустимое состояние, как только он существует. Всякий раз, когда события применяются, мы можем соответствующим образом изменить состояние этого корня агрегата:

```php
class CartAggregateRoot extends AggregateRoot
{
    // ...

    protected function applyCheckedOut(CartCheckedOut $event): void
    {
        $this->status = CartStatus::checkedOut;
    }

    protected function applyFailed(CartFailed $event): void
    {
        $this->status = CartStatus::failed;
    }

    protected function applyPaid(CartPaid $event): void
    {
        $this->status = CartStatus::paid;
    }
}
```

Каждый раз, когда корень агрегата извлекается и его события реконституируются, наше состояние будет автоматически обновляться до текущего.

Благодаря этому полю состояния мы теперь можем писать простые проверки вместо микроменеджмента логических флагов:

```php
class CartAggregateRoot extends AggregateRoot
{
    // ...

    public function addItem(/* ... */): self
    {
        if (! $this->status == CartStatus::pending) {
            throw new InvalidCartStatus();
        }
        // ...
    }

    public function checkout(/* ... */): self
    {
        if (! $this->status == CartStatus::pending) {
            throw new InvalidCartStatus();
        }
        // ...
    }

    public function pay(/* ... */): self
    {
        if (! $this->status == CartStatus::checkedOut) {
            throw new InvalidCartStatus();
        }
        // ...
    }
}
```

Однако есть другой способ управления состоянием, о котором я хотел бы поговорить: шаблон "состояние" (state pattern). Этот шаблон предлагает вынести управление состоянием из корня агрегата в отдельные классы. Он также позволяет гибко моделировать переходы между состояниями на основе дополнительных переменных, что в нашем случае будет блокировкой.

Мы начнем с внесения изменений в наш корень агрегата. Поступая так, мы получим представление о том, какую функциональность должны предоставлять наши классы состояний. В этом случае мне нравится работать от обратного, чтобы решение не было сложнее, чем необходимо. Итак, сначала мы напишем падающий код в нашем корне агрегата, а затем исправим его, создав наши классы состояний.

Есть несколько интересных вариантов использования, которые должен поддерживать наш корень агрегата:

- Блокировка корзины;
- Определение, разрешено ли добавление товара на основе состояния блокировки;
- Определение того, каким должно быть следующее состояние после сбоя корзины, также на основе состояния блокировки.

Мы рассмотрим эти три сценария изолированно. Во-первых, давайте настроим наш корень агрегата и начнем с него:

```php
class CartAggregateRoot extends AggregateRoot
{
    private CartState $state;

    public function applyLock(CartLocked $event): void
    {
        // Мы должны отметить наше текущее состояние как заблокированное
    }

    public function addItem(/* ... */): self
    {
        // Разрешено ли нам вносить изменения в корзину?
        if (! $this->state->changesAllowed()) {
            throw new CannotChangeCart();
        }
    }
```

```php
    protected function applyFailed(CartFailed $event): void
    {
        // Переходим в состояние pending или failed?
        $this->state = $this->state->nextState();
    }
}
```

Исходя из нашего корня агрегата, ясно, что наше состояние должно предоставлять по крайней мере два метода: `changesAllowed()` и `nextState()`. Мы подошли к точке, где можем создать абстрактный класс CartState:

```php
abstract class CartState
{
    abstract public function changesAllowed(): bool;
    abstract public function nextState(): CartState;
}
```

Затем мы реализуем этот класс для всех наших состояний, всего четыре:

**Pending:**
```php
class Pending extends CartState
{
    public function changesAllowed(): bool
    {
        return true;
    }

    public function nextState(): CartState
    {
        return new CheckedOut();
    }
}
```

**CheckedOut:**
```php
class CheckedOut extends CartState
{
    public function changesAllowed(): bool
    {
        return false;
    }

    public function nextState(): CartState
    {
        return new Paid();
    }
}
```

**Paid:**
```php
class Paid extends CartState
{
    public function changesAllowed(): bool
    {
        return false;
    }

    public function nextState(): CartState
    {
        return new Paid();
    }
}
```

**Failed:**
```php
class Failed extends CartState
{
    public function changesAllowed(): bool
    {
        return false;
    }

    public function nextState(): CartState
    {
        return new Failed();
    }
}
```

Некоторые реализации просты: оплаченная корзина никогда не должна изменяться, и оформленная корзина тоже. Но в состоянии `pending` мы также должны отслеживать состояние блокировки; аналогично, состоянию `failed` нужно определить следующее на основе этого состояния блокировки.

Давайте смоделируем состояние блокировки с помощью класса и назовем его CartLockState. У него есть две реализации: Locked и Unlocked с теми же методами, что и у CartState:

```php
abstract class CartLockState
{
    abstract public function changesAllowed(): bool;
    abstract public function nextState(): CartState;
}
```

**Locked:**
```php
class Locked extends CartLockState
{
    public function changesAllowed(): bool
    {
        return false;
    }

    public function nextState(): CartState
    {
        return new Failed();
    }
}
```

**Unlocked:**
```php
class Unlocked extends CartLockState
{
    public function changesAllowed(): bool
    {
        return true;
    }

    public function nextState(): CartState
    {
        return new Pending();
    }
}
```

Затем мы объединяем CartState и CartLockState вместе: каждый CartState будет также содержать внутренний CartLockState:

```php
abstract class CartState
{
    public function __construct(
        protected CartLockState $lockState
    ) {}

    // ...
}
```

Не забудьте изменить все существующие реализации `nextState()`, потому что теперь нам нужно передавать состояние блокировки при создании нового состояния корзины.

Наконец, мы можем использовать состояние блокировки, чтобы определить, что должно произойти в Pending и Failed:

**Pending:**
```php
class Pending extends CartState
{
    public function changesAllowed(): bool
    {
        return $this->lockState->changesAllowed();
    }
    // ...
}
```

**Failed:**
```php
class Failed extends CartState
{
    // ...

    public function nextState(): CartState
    {
        return $this->lockState->nextState();
    }
}
```

Однако мы еще не закончили. Давайте вернемся к нашему корню агрегата, чтобы соединить все вместе:

```php
class CartAggregateRoot extends AggregateRoot
{
    private CartState $state;

    public function __construct()
    {
        $this->state = new Pending(new Unlocked());
    }

    public function applyLock(CartLocked $event): void
    {
        // Мы должны отметить наше текущее состояние как заблокированное
        $this->state = $this->state->locked();
    }
}
```

Наше начальное состояние будет Pending и Unlocked. Всякий раз, когда мы применяем событие CartLocked, мы обновляем текущее состояние и меняем его состояние блокировки на Locked. Чтобы сделать это, мы добавили еще один метод в CartState как способ переключения состояния блокировки:

```php
abstract class CartState
{
    // ...

    public function locked(): self
    {
        $clone = clone $this;
        $clone->lockState = new Locked();
        return $clone;
    }
}
```

## ТЕСТИРОВАНИЕ

Поток состояний в корнях агрегатов является одной из самых важных частей вашего приложения и должен быть тщательно протестирован. Используем ли мы перечисления состояний или применяем шаблон "состояние", именно корень агрегата требует больше всего тестирования; реализация состояний — это лишь небольшая деталь реализации.

Так или иначе, вы бы писали обычные тесты корней агрегатов, как мы видели раньше; и проверяли, какие события запускаются, а какие нет, в зависимости от состояния корня агрегата.

Вы можете себе представить, что этот косвенный подход к тестированию состояний не всегда идеален. Мы вернемся к состояниям в главе после следующей и обсудим, как моделировать и тестировать специализированные конечные автоматы. Но сначала нам нужно рассмотреть другую тему, чтобы понять конечные автоматы.

В этих примерах я показал только соответствующие части, чтобы понять, как используются состояния и шаблон "состояние". Если вы хотите увидеть полную картину, обязательно ознакомьтесь с демо-приложением.

Использовать ли вам простые статусы в стиле перечислений или классы состояний, зависит от вас, у каждого подхода есть свои плюсы и минусы. Я определенно рекомендую начать с более простого подхода и использовать перечисление для состояния, а затем реорганизовать код при необходимости.

В следующей главе мы продолжим исследовать, как мы можем упростить сложные корни агрегатов, разделив их на несколько частей.

---

[Далее: Части агрегатов (Aggregate Partials)](/chapters/2-Advanced-Patterns/Multi-Entity-Aggregate-Roots.md)
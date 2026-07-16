[назад](/chapters/2-Advanced-Patterns/State-Management-in-Aggregate-Roots.md)

---

# Части агрегатов (Aggregate Partials)

Эта глава будет посвящена разделению корня агрегата на несколько более мелких частей. При этом мы всегда будем соблюдать правило, согласно которому в агрегат должна быть только одна точка входа — корень агрегата, — но мы позволим агрегату быть внутренне разделенным на разные части. Эти части называются сущностями агрегата или частями агрегата. Обратите внимание, что в последней версии пакета event sourcing мы решили использовать название "части агрегата", чтобы избежать путаницы с сущностями в DDD.

Рано или поздно вы заметите, что корни агрегатов начинают расти, даже когда управление состоянием вынесено в отдельные классы. Реальность такова, что многие бизнес-процессы сложны и требуют от нас написания более чем нескольких строк кода. Даже в пределах одного и того же агрегата (например, корзины) существует потенциал для его роста больше, чем вам хотелось бы.

Части агрегатов помогают решить эту проблему, перемещая части корня агрегата в отдельные классы. Так что именно можно вынести? Хорошая новость в том, что вы всегда можете принять эти решения позже в процессе разработки. Очень легко реорганизовать корень агрегата в части, даже когда проект уже находится в продакшене.

В качестве примера мы собираемся создать специальный класс части агрегата, который управляет товарами корзины. Он будет выглядеть примерно так:

```php
namespace Spatie\Shop\Cart\Partials;

use Spatie\EventSourcing\AggregateRoots\AggregatePartial;

class CartItems extends AggregatePartial
{
    private array $cartItems = [];

    public function isEmpty(): bool
    {
        return count($this->cartItems) == 0;
    }
}
```

Сила частей агрегата в том, что они работают почти так же, как корни агрегатов: они могут записывать и применять события и строить свое внутреннее состояние из реконституированных событий.

Это означает, что мы можем переместить весь код, связанный с товарами, из нашего CartAggregateRoot в наш вновь созданный класс CartItems:

```php
class CartItems extends AggregatePartial
{
    // ...

    public function addItem(
        string $cartItemUuid,
        Product $product,
        int $amount
    ): self {
        $this->recordThat(new CartItemAdded(
            cartItemUuid: $cartItemUuid,
            productUuid: $product->uuid,
            amount: $amount,
        ));

        return $this;
    }

    protected function applyCartItemAdded(CartItemAdded $cartItemAdded): void
    {
        $this->cartItems[$cartItemAdded->cartItemUuid] = null;
    }
```

```php
    public function removeItem(CartItem $cartItem): self
    {
        $exists = array_key_exists(
            $cartItem->uuid,
            $this->cartItems,
        );

        if (! $exists) {
            throw new UnknownCartItem();
        }

        $this->recordThat(new CartItemRemoved(
            cartUuid: $this->aggregateRootUuid(),
            cartItemUuid: $cartItem->uuid,
        ));

        return $this;
    }

    protected function applyCartItemRemoved(CartItemRemoved $cartItemRemoved): void
    {
        unset($this->cartItems[$cartItemRemoved->cartItemUuid]);
    }
}
```

Давайте отметим несколько вещей:

- `$this->recordThat()` доступен, так же как и в корнях агрегатов;
- методы `apply` также работают так же, как в корнях агрегатов;
- и, наконец, поскольку часть агрегата связана с корнем агрегата (корень агрегата передаётся в часть агрегата при её создании), она может получить доступ к своему UUID, используя `$this->aggregateRootUuid()`.

Затем нам нужно изменить наш корень агрегата. Поскольку он по-прежнему является точкой входа для всего кода, находящегося за пределами агрегата, нам нужно будет также предоставить методы `addItem()` и `removeItem()` в нем, но они будут делать не более чем использовать базовую часть агрегата CartItems:

```php
class CartAggregateRoot extends AggregateRoot
{
    protected CartItems $cartItems;

    public function __construct()
    {
        $this->cartItems = new CartItems($this);
    }

    public function addItem(
        string $cartItemUuid,
        Product $product,
        int $amount
    ): self {
        if (! $this->state->changesAllowed()) {
            throw new CartCannotBeChanged();
        }

        $this->cartItems->addItem(
            $cartItemUuid,
            $product,
            $amount,
        );

        return $this;
    }
```

```php
    public function removeItem(CartItem $cartItem): self
    {
        if (! $this->state->changesAllowed()) {
            throw new CartCannotBeChanged();
        }

        $this->cartItems->removeItem($cartItem);

        return $this;
    }
    // ...
}
```

Опять же, сделаем несколько наблюдений:

- переменная `$cartItems` должна быть защищенной (`protected`), чтобы пакет имел к ней доступ;
- в этом примере мы сохраняем проверки состояния в нашем корне агрегата. При необходимости вы можете создавать выделенное состояние для отдельных частей агрегата.

Последняя часть головоломки — предоставление доступа к состоянию между частями агрегата и корнями агрегатов. Как мы обсуждали ранее, нежелательно раскрывать состояние корня агрегата внешнему миру, но вполне допустимо использовать его внутри самого агрегата. К сожалению, в PHP нет способа обеспечить "дружественные классы", которые могли бы, например, получать доступ к защищенным свойствам или методам. Это означает, что нам нужно будет предоставить публичные геттеры для чтения состояния между корнем агрегата и его частями агрегата. Мы уже добавили такой метод `isEmpty()` в CartItems и можем использовать его в нашем корне агрегата следующим образом:

```php
class CartAggregateRoot extends AggregateRoot
{
    // ...

    public function checkout(CartCheckoutData $cartCheckoutData): self
    {
        if ($this->cartItems->isEmpty()) {
            throw new CartIsEmpty();
        }

        // ...
    }
}
```

Если вы хотите передать состояние в другом направлении (от корня агрегата к части агрегата), вы можете либо добавить геттеры в корень агрегата и помнить, что использовать их нужно только внутри, либо передавать состояние агрегата в его части при каждом вызове их методов. Лично я предпочитаю второй подход, чтобы избежать злоупотребления публичными геттерами.

Поскольку части агрегатов являются внутренней деталью реализации агрегата, ваши тесты корня агрегата будут продолжать работать как прежде. Вы все еще можете подделать корень агрегата и наблюдать за записанными событиями, как вы привыкли. Это делает рефакторинг к частям агрегата чрезвычайно простым, поскольку вам не нужно менять какие-либо тесты.

```php
/** @test */
public function can_add_item()
{
    CartAggregateRoot::fake(self::CART_UUID)
        ->given([
            $this->cartInitializedFactory->create(),
        ])
        ->when(function (CartAggregateRoot $cartAggregateRoot) {
            $cartAggregateRoot->addItem(
                self::CART_ITEM_UUID,
                $this->product,
                1
            );
        })
        ->assertRecorded([
            $this->cartItemAddedFactory->create(),
        ]);
}
```

Также можно тестировать части агрегата изолированно, используя `AggregatePartial::fake()`. Поступая так, вам не нужно беспокоиться о настройке реального корня агрегата, и вы можете использовать часть агрегата как есть.

```php
class CartItemsTest extends TestCase
{
    private const CART_ITEM_UUID = 'cart-item-uuid';

    /** @test */
    public function test_is_empty()
    {
        $cartItems = CartItems::fake();

        $this->assertTrue($cartItems->isEmpty());

        $product = ProductFactory::new()->create();

        $cartItems->addItem(
            self::CART_ITEM_UUID,
            $product,
            1
        );

        $this->assertFalse($cartItems->isEmpty());
    }
}
```

Подделка частей агрегата может быть полезна, если в потоке части агрегата есть много мелких деталей, для настройки которых потребовалось бы много подготовительной работы, если бы вы тестировали их на уровне корня агрегата. Однако вам все равно придется тестировать правильное использование частей агрегата с точки зрения корня агрегата.

---

[Далее: Конечные автоматы с помощью частей агрегатов](/chapters/2-Advanced-Patterns/State-Machines-with-Aggregate-Entities.md)
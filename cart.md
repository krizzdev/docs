# Cart

## Using The Cart

The Cart is available via the `Cart` facade - or the `app('vanilo.cart')` service.

The facade (service) actually returns a `CartManager` object which exposes the
cart API to be used by applications. It encapsulates the `Cart` eloquent model,
that also has `CartItem` children.

The `CartManager` was introduced in order to take care of:

- Relation of carts and the session and/or the user
- Only create carts in the db if it's necessary (ie. don't pollute DB with a cart for every single visitor/hit)
- Provide a straightforward API

> The reason for CartManager being present, and being above the Cart ActiveRecord is to avoid creating
> millions of empty carts (for every new session, even for bots).
>
> Using the cart manager can be omitted, just use the `Vanilo\Cart\Models\Cart` instead of the Cart facade.

### Get Products in Cart

`Cart::getItems()` returns all products in the cart.

It returns an empty collection if the cart is empty.

### Checking Whether A Cart Exists

As written above, the cart manager only creates a cart entry (db) if
it's needed. Thus you can check whether a cart exists or not.

A non-existing cart means that the current session has no cart model/db record
associated.

`Cart::exists()` returns whether a cart exists for the current session.

`Cart::doesNotExist()` is the opposite of `exists()` 🤯

**Example:**

```php
var_dump(Cart::exists());
// false

Cart::addItem($product);

var_dump(Cart::exists());
// true
```

### Item Count

`Cart::itemCount()` returns the number of items in the cart.

It also returns 0 for non-existing carts.

### Is Empty Or Not?

To have a cleaner code, there are two methods to check if cart is empty:

- `Cart::isEmpty()`
- `Cart::isNotEmpty()`

Their result is based on the `itemCount()` method.

### Adding An Item

You can add product to the cart with the `Cart::addItem()` method.

The item is a [Vanilo product](products.md) by default, which can be
extended.

You aren't limited to using Vanilo products, you can add any Eloquent
model to the cart as "product" that implements the
[Buyable interface](https://github.com/vanilophp/contracts/blob/master/src/Buyable.php) from the
[vanilo/contracts](https://github.com/vanilophp/contracts) package.


**Example:**

```php
$product = Product::findBySku('B01J4919TI'); //Salmon Fish Sushi Pillow -- check out on amazon :D

Cart::addItem($product); //Adds one product to the cart
echo Cart::itemCount();
// 1

// The second parameter is the quantity
Cart::addItem($product, 2);
echo Cart::itemCount();
// 3
```

#### Configurable Items

If your shop is selling customizable, configurable products you can pass the configuration in the third parameter:

```php
Cart::addItem($hamburger, 1, [ 'attributes' => [
        'configuration' => ['extra_bacon', 'no_tomato']
    ]
]);
```

This will set `['extra_bacon', 'no_tomato']` as the cart item's configuration.

Furthermore, if you add another product of the same type to the cart, it will create
a separate cart item if the configuration of the items differ:

```php
Cart::addItem($bigTastyBurger, 1, ['attributes' => ['configuration' => 'extra_cheese']]);
Cart::addItem($bigTastyBurger); // No configuration
Cart::getItems()->count();
// => 2
```

The cart will take the cart items and compare their configurations with the passed configuration.
It uses internally the `array_diff` and `array_diff_assoc` php methods for comparison.

If you would like to customize what attributes denote a new cart item, then you need to
override the Cart model's `resolveCartItem()` method.

**Example**:

```php
// app/Models/Cart.php
class Cart extends \Vanilo\Foundation\Models\Cart
{
    /**
     * @param Buyable $buyable The product that is currently being added to the cart
     * @param array $parameters The parameters array
     */
    protected function resolveCartItem(Buyable $buyable, array $parameters): ?CartItemContract
    {
        $existingCartItemsOfTheBuyable = $this->items()->ofCart($this)->byProduct($buyable)->get();

        // Implement your cart item resolution logic here. If you return:
        //    - NULL: then the cart will create a new cart item from the buyable;
        //    - a CartItem, then the quantity of the item will be increased 

        return null or CartItem;
    }
}
```

Make sure to override the Cart Model class in your application:

```php
// app/Providers/AppServiceProvider.php
namespace App\Providers;

use Illuminate\Support\ServiceProvider;
use Vanilo\Cart\Contracts\Cart as CartContract;

class AppServiceProvider extends ServiceProvider
{
    public function boot()
    {
        $this->app->concord->registerModel(
            CartContract::class, \App\Models\Cart::class
        );
    }
}
```


#### Setting Custom Item Attributes

First, you need to add your custom fields to `cart_items` (preferably using migrations).

**Example:**

```php
// The Migration:
Schema::table('cart_items', function (Blueprint $table) {
    $table->integer('weight')->nullable();
});
```

**Passing fields manually:**

```php
Cart::addItem($product, 1, [ 'attributes' => [
        'weight' => 3
    ]
]);
```

**Permanent extra fields**

It is possible to configure the cart to always copy some extra attributes
from product (Buyable) to cart items:

```php
//config/vanilo.php:
//...
    'cart' => [
        'extra_product_attributes' => ['weight']
    ]
//...
```

Having this configuration the value of `weight` attribute gets copied
automatically to cart item:

```php
$product = Product::create([
    'name'   => 'Mesh Panel Toning Trainers',
    'sku'    => 'LS-170161',
    'price'  => 34.99,
    'weight' => 9
]);

$item = Cart::addItem($product);
echo $item->weight;
// 9
```

### Retrieving The Item's Associated Product

The `CartItem` defines a [polymorphic relationship](https://laravel.com/docs/10.x/eloquent-relationships#polymorphic-relationships)
to the Buyable object named `product`.

So you have a reference to the item's product:

```php
$product = \App\Models\Product::find(203);
$cartItem = Cart::addItem($product);

echo $cartItem->product->id;
// 203
echo get_class($cartItem->product);
// "App\Models\Product"

$course = \App\Models\Course::findBySku('REACT-001');
$cartItem = Cart::addItem($course);

echo $cartItem->product->sku;
// "REACT-001"
echo get_class($cartItem->product);
// "App\Models\Course"
```

### Buyables (products)

> The `Buyable` interface is located in the
> [Vanilo Contracts](https://github.com/vanilophp/contracts) package.

You can add any Eloquent model to the cart that implements the `Buyable` interface.

Buyable classes must implement these methods:

```
function getId(); // the id of the entry
function name(); // the name to display in the cart
function getPrice(); // the price
function morphTypeName(); // the type name to store in the db
function hasImage(); // whether or not the entry has an image
function getThumbnailUrl(); // the url of the thumbnail image (or null)
function getImageUrl(); // the url of the image (or null)
```
#### Buyable Morph Maps

In order to decouple the database from the application's internal
structure, it is possible to not save the Buyable's full class name
in the DB.
When the cart associates a product (Buyable) with a cart item, it fetches
the type name from the `Buyable::morphTypeName()` method.

The `morphTypeName()` method, can either return the full class name
(Eloquent's default behavior), or some shorter version like:

| Full Class Name               | Short Version (Saved In DB) |
|:------------------------------|:----------------------------|
| Vanilo\Product\Models\Product | product                     |
| App\Models\Course             | course                      |

If you're not using the FQCN, then you have to add the mapping during
boot time:

```php
use Illuminate\Database\Eloquent\Relations\Relation;

Relation::enforceMorphMap([
    'product' => 'Vanilo\Product\Models\Product',
    'course'  => 'App\Models\Course',
]);
```

For more information refer to the [Polymorphic Relation](https://laravel.com/docs/10.x/eloquent-relationships#polymorphic-relationships)
section in the Laravel Documentation.

### Removing Items

There are two methods for removing specific items:

1. `Cart::removeProduct($product)`
2. `Cart::removeItem($cartItem)`

**`removeProduct()` example:**
```php
$product = Product::find(12345);

Cart::removeProduct($product); // Finds the item based on the product, and removes it
```

**`removeItem()` example:**
```php

//Remove the first item from the cart
$item = Cart::model()->items->first();

Cart::removeItem($item);
```

### Cart States

Cart has a state field which can be by default one of these values:

- active: the cart is active;
- checkout: the cart is being checked out;
- completed: the cart has been checked out (order was created);
- abandoned: the cart hasn't been touched for a while;

If you want to modify the possible states of the cart, follow the instructions for
[Customizing Enums](enums.md#customizing-enums);

> The state field is not auto-managed, thus you explicitly have to update it's value.


### Associating With Users

The cart can be assigned to user automatically and/or manually.

#### The User Model

The cart's user model is not bound to any specific class (like `App\Models\User`).

By default, it uses the model defined in `auth.providers.users.model` configuration entry.
In fresh Laravel installations, and in most of the cases this will give the `App\Models\User` class.

However, these values are just sensible defaults, and Laravel's
[authentication system](https://laravel.com/docs/10.x/authentication) does not force you to have this
setup.

You can specify the user model manually by setting the user model class name under the
`vanilo.cart.user.model` configuration key.

```php
// config/vanilo.php
return [
    'user' => [
        'model' => App\Some\Other\User::class,
    ]
];
```

#### Manual Association

```php
use Vanilo\Cart\Facades\Cart;

// Assign the currently logged in user:
Cart::setUser(Auth::user());

// Assign an arbitrary user:
$user = \App\Models\User::find(1);
Cart::setUser($user);

// User id can be passed as well:
Cart::setUser(1);

// Retrieve the cart's assigned user:
$user = Cart::getUser();

// Remove the user association:
Cart::removeUser();
```

#### Automatic Association

The cart (by default) automatically handles cart+user associations in the following cases:

| Event                     | State             | Action                    |
|:--------------------------|:------------------|:--------------------------|
| User login/authentication | Cart exists       | Associate cart with user  |
| User logout & lockout     | Cart exists       | Dissociate cart from user |
| New cart gets created     | User is logged in | Associate cart with user  |

To prevent this behavior, set the `vanilo.cart.auto_assign_user` config value to false:

```php
// config/vanilo.php
return [
    'cart' => [
        'auto_assign_user' => false
    ]
];
```

#### Preserve The Cart For Users Across Logins

It is possible to keep the cart for users after logout and restore it after successful login.

> This feature is disabled by default. To achive this behavior, set the
> `vanilo.cart.preserve_for_user` config value to true

| Event                     | State                                                             | Action                               |
|:--------------------------|:------------------------------------------------------------------|:-------------------------------------|
| User login/authentication | Cart for this session doesn't exist, user has a saved active cart | Restore the cart                     |
| User login/authentication | Cart for this session exists                                      | The current cart will be kept        |
| User logout & lockout     | Cart for this session exists                                      | Cart will be kept for the user in db |

##### Merge Anonymous And User Carts On Login

If the `preserve_for_user` feature is enabled, a previously saved cart will be restored if a user
logs back in.

But what happens when a user already has a new cart with items? By default, the cart gets replaced
by the previously saved one.

If you set the `vanilo.cart.merge_duplicates` config option to true, then the previously saved
cart's items will be added to the current cart, preserving the user's new cart as well.

> This option is disabled by default.

### Totals

The item total can be accessed with the `total()` method or the `total`
property.

The cart total can be accessed with the `Cart::total()` method:

```php
use Vanilo\Cart\Facades\Cart;
use App\Models\Product;

$productX = Product::create(['name' => 'X', 'price' => 100]);
$productY = Product::create(['name' => 'Y', 'price' => 70]);

$item1 = Cart::addItem($productX, 3);

echo $item1->total();
// 300
echo $item1->total;
// 300

$item2 = Cart::addItem($productY, 2);
echo $item2->total();
// 140

echo Cart::total();
// 440
```

### Clearing The Cart

The `Cart::clear()` method removes everything from the cart, but it
doesn't destroy it, unless the `vanilo.cart.auto_destroy` config option is true.

So the entry in the `cart` table will remain, and it will be assigned to
the current session later on.

### Destroying The Cart

In case you want to get rid of the cart use the `Cart::destroy()` method.

It clears the cart, removes the record from the `carts` table, and unsets
the association with the current session.

Thus using destroy, you'll have a non-existent cart.

### Forgetting The Cart

The `Cart::forget()` method disconnects the current user/session from the current cart, but the cart
will be kept intact in the database.

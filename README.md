# Transaction-aware Event Dispatcher for Laravel (and Lumen)

[![Latest Stable Version](https://poser.pugx.org/fntneves/laravel-transactional-events/v/stable)](https://packagist.org/packages/fntneves/laravel-transactional-events)
<a href="https://travis-ci.org/fntneves/laravel-transactional-events"><img src="https://travis-ci.org/fntneves/laravel-transactional-events.svg?branch=master" alt="TravisCI Status"></a>
<a href="https://scrutinizer-ci.com/g/fntneves/laravel-transactional-events/?branch=master"><img src="https://scrutinizer-ci.com/g/fntneves/laravel-transactional-events/badges/quality-score.png?b=master" alt="Scrutinizer Code Quality"></a>
[![Total Downloads](https://poser.pugx.org/fntneves/laravel-transactional-events/downloads)](https://packagist.org/packages/fntneves/laravel-transactional-events)

This package introduces a transactional layer to Laravel Event Dispatcher.<br/>
Out of the box, it ensures consistency between events dispatched and database transactions.

* [Introduction](#introduction)
* [Installation](#installation)
    * [Laravel](#laravel)
    * [Lumen](#lumen)
* [Usage](#usage)
* [Configuration](#configuration)

## Introduction

Let's start with a simple example of ordering tickets. Assume that it involves database changes and a payment registration and that the custom event `OrderWasProcessed` is dispatched immediately after the order is processed in the database.

```php
DB::transaction(function() {
    $user = User::find(...);
    $concert = Concert::find(...);
    $order = $concert->orderTickets($user, 3);
    event(new OrderWasProcessed($order));
    PaymentService::registerOrder($order);
});
```

The transaction in the above example may fail for several reasons: an exception may occur in the `orderTickets` method or in the payment service or simply due to a deadlock.

A failure will rollback the database changes made during the transaction. However, this is not true for the `OrderWasProcessed` event, which is actually dispatched and eventually executed. Considering that this event may result in sending an e-mail with the order confirmation, managing it the right way becomes mandatory.

The purpose of this package is to actually dispatch events **if and only if** the transaction in which they were dispatched commits. For instance, in the above example the `OrderWasProcessed` event would not be dispatched if the transaction fails.

Please note that events dispatched out of transactions will bypass the transactional layer, meaning that they will be handled by the default Event Dispatcher. This is true also for events in which the `$halt` parameter is set to `true`.

## Installation

 Laravel  | Package
:---------|:----------
 5.5.x-5.7.x   | 1.4.x
 5.8.x-7.x     | 1.8.x

* [Laravel](#laravel) (5.5+)
* [Lumen](#lumen) (5.5+)

### Laravel
The installation of this package in Laravel is automatic thanks to the _Package Auto-Discovery_ feature of Laravel 5.5+.
Just add this package to the `composer.json` file and it will be ready for your application.

```
composer require fntneves/laravel-transactional-events
```

A configuration file is also available for this package. Run the following command to copy the provided configuration file `transactional-events.php` your `config` folder.

```
php artisan vendor:publish --provider="Neves\Events\EventServiceProvider"
```

### Lumen

As Lumen is built on top of Laravel packages, this package should also work smoothly on this micro-web framework.
Run the following command to install this package:

``` bash
composer require fntneves/laravel-transactional-events
```

In order to configure the behavior of this package, copy the configuration files:

```bash
cp vendor/fntneves/laravel-transactional-events/src/config/transactional-events.php config/transactional-events.php
```

Then, in `bootstrap/app.php`, register the configuration and the service provider:<br/>
*Note:* This package must be registered _after_ the default EventServiceProvider, so your event listeners are not overriden.

```php
// The default EventServiceProvider must be registered.
$app->register(App\Providers\EventServiceProvider::class);

...

$app->configure('transactional-events');
$app->register(Neves\Events\EventServiceProvider::class);
```

## Usage

The transactional layer is enabled out of the box for the events placed under the `App\Events` namespace.

Additionally, this package offers two ways to mark events as transactional:
- Implement the `Neves\Events\Contracts\TransactionalEvent` contract (recommended)
- Change the [configuration file](#configuration) provided by this package

#### Use the contract, Luke:

The easiest way to make your events behave as transactional events is by implementing the contract `Neves\Events\Contracts\TransactionalEvent`.<br/>
*Note that events that implement it will behave as transactional events even when marked as excluded in config.*

```php
namespace App\Events;

use Illuminate\Queue\SerializesModels;
use Illuminate\Foundation\Events\Dispatchable;
...
use Neves\Events\Contracts\TransactionalEvent;

class TicketsOrdered implements TransactionalEvent
{
    use Dispatchable, InteractsWithSockets, SerializesModels;

    ...
}
```
As this package does not require any changes in your code, you are still able to use the `Event` facade and call the `event()` or `broadcast()` helper to dispatch an event:

```php
Event::dispatch(new App\Event\TicketsOrdered) // Using Event facade
event(new App\Event\TicketsOrdered) // Using event() helper method
broadcast(new App\Event\TicketsOrdered) // Using broadcast() helper method
```

Even if you are using queues, they will still work smothly because this package does not change the core behavior of the event dispatcher. They will be enqueued as soon as the active transaction succeeds. Otherwise, they will be discarded.

**Reminder:** Events are considered transactional when they are dispatched within transactions. When an event is dispatched out of transactions, it bypasses the transactional layer.


## Configuration

The following keys are present in the configuration file:

Enable or disable the transactional behavior by changing the following property:
```php
'enable' => true
```

By default, the transactional behavior will be applied to events on `App\Events` namespace. Feel free to use patterns and namespaces.

```php
'transactional' => [
    'App\Events'
]
```

Choose the events that should always bypass the transactional layer, i.e., should be handled by the default event dispatcher. By default, all `*ed` Eloquent events are excluded. The main reason for this default value is to avoid interference with your already existing event listeners for Eloquent events.

```php
'excluded' => [
    // 'eloquent.*',
    'eloquent.booted',
    'eloquent.retrieved',
    'eloquent.saved',
    'eloquent.updated',
    'eloquent.created',
    'eloquent.deleted',
    'eloquent.restored',
],
```

## Known issues

#### Transactional events are not dispatched in tests.

**This issue is fixed for Laravel 5.6.16+ (see [#23832](https://github.com/laravel/framework/pull/23832)).**
For previous versions, it is associated with the `RefreshDatabase` or `DatabaseTransactions` trait, namely when it uses database transactions to reset database after each test.
This package relies on events dispached when transactions begin/commit/rollback and as each test is executed within a transaction that is rolled back when test finishes, the dispatched application events are never actually dispatched. In order to get the expected behavior, use the `Neves\Testing\RefreshDatabase` or `Neves\Testing\DatabaseTransactions` trait in your tests instead of the ones originally provided by Laravel.

## License
This package is open-sourced software licensed under the [MIT license](http://opensource.org/licenses/MIT).

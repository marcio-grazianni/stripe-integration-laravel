# Installation

Install the package using Composer:

```bash
$ composer require cloudcreativity/laravel-stripe:1.x-dev
```

Add the service provider and facade to your app config file:

```php
// config/app.php
return [
    // ...
    
    'providers' => [
        // ...
        CloudCreativity\LaravelStripe\ServiceProvider::class,
    ],
    
    'aliases' => [
        // ...
        'Stripe' => CloudCreativity\LaravelStripe\Facades\Stripe::class,
    ],
];
```

Then publish the package config:

```bash
$ php artisan vendor:publish --tag=stripe
```

## Configuration

Package configuration is in the `stripe.php` config file. That file contains descriptions of
each configuration option, and these options are also referred to in the relevant documentation
chapters.

Note that by default Laravel puts your Stripe keys in the `services` config file. We expect
them to be there too. Here's an example from Laravel 5.8:

```php
return [
    // ...other service config
    
    'stripe' => [
        'model' => \App\User::class,
        'key' => env('STRIPE_KEY'),
        'secret' => env('STRIPE_SECRET'),
        'webhook' => [
            'secret' => env('STRIPE_WEBHOOK_SECRET'),
            'tolerance' => env('STRIPE_WEBHOOK_TOLERANCE', 300),
        ],
    ],
];
```

## Migrations

This package contains a number of migrations for the models it provides. **By default these
are loaded by the package.**

If you are customising any of the models in our implementation, you will need to disable migrations
and publish the migrations instead.

First, disable the migrations in the `register()` method of your application's service provider:

```php
namespace App\Providers;

use CloudCreativity\LaravelStripe\LaravelStripe;
use Illuminate\Support\ServiceProvider;

class AppServiceProvider extends ServiceProvider
{

  public function register()
  {
        LaravelStripe::withoutMigrations();
  }
}
```

Then publish the migrations:

```bash
$ php artisan vendor:publish --tag=stripe-migrations
```

> You must disable migrations **before** attempting to publish them, as they will only be publishable
if migrations are disabled. Plus you must use the `register()` method, not `boot()`.

## Brand Assets

If you want to use *Powered by Stripe* badges, or *Connect with Stripe* buttons, publish
[Stripe brand assets](https://stripe.com/gb/newsroom/brand-assets) using the following command:

```bash
$ php artisan vendor:publish --tag=stripe-brand
``` 

This will publish the files into the `public/vendor/stripe/brand` folder.

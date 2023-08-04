# Stripe Connect

## Account Owners

Each Stripe account that is connected to your application must have an account *owner*.
The owner is the model in your application that the Stripe account belongs to.

A typical example would be your `\App\User` model, however you can choose whatever model
you need as long as you apply our account owner interface and trait.

For example, if our account owner was our application's user:

```php
namespace App;

use CloudCreativity\LaravelStripe\Connect\OwnsStripeAccounts;
use CloudCreativity\LaravelStripe\Contracts\Connect\AccountOwnerInterface;
use Illuminate\Foundation\Auth\User as Authenticatable;

class User extends User implements AccountOwnerInterface
{

    use OwnsStripeAccounts;

    // ...
}
``` 

Once you've done that, ensure the class name is set as the `stripe.connect.owner`
config value.

## Accounts

We provide a `\CloudCreativity\LaravelStripe\Models\StripeAccount` model to store connected accounts.
We store the majority of attributes provided by Stripe on their
[account resource.](https://stripe.com/docs/api/accounts)
The model's attributes are:

| Attribute | Type | Description |
| :-- | :-- | :-- |
| `id` | `string` | The Stripe id for the account resource. |
| `business_profile` | `json` | Information related to the business. |
| `business_type` | `string` | The business type, `individual` or `company`. |
| `capabilities` | `json` | The capabilities requested for the connected account. |
| `charges_enabled` | `bool` | Whether the account can create live charges. |
| `company` | `json` | Information about the company or business. |
| `country` | `string` | The account's country. |
| `created` | `datetime` | When the account was created with Stripe. |
| `created_at` | `datetime` | When the account model was created in your application. |
| `default_currency` | `string` | Three-letter ISO currency code representing the default currency for the account. |
| `deleted_at` | `datetime` | When the account model was soft-deleted. By default the account is soft-deleted when deauthorized. |
| `details_submitted` | `bool` | Whether account details have been submitted. |
| `email` | `string` | The primary user's email address. |
| `individual` | `json` | Information about the person represented by the account. |
| `metadata` | `json` | Set of key-value pairs that you have attached to the Stripe account resource. |
| `owner_id` | `integer` | The key of the model that the Stripe account belongs to. |
| `payouts_enabled` | `bool` | Whether Stripe can send payouts to this account. |
| `requirements` | `json` | Information about the requirements for the account, including what information needs to be collected, and by when. |
| `settings` | `json` | Account options for customizing how the account functions within Stripe. |
| `tos_acceptance` | `json` | Details on the acceptance of the Stripe Services Agreement. |
| `type` | `string` | The Stripe account type. Can be `standard`, `express`, or `custom`. |
| `updated_at` | `datetime` | When the account model was last updated. |

> Some columns will only have values depending on the type of account. You should refer to the
[Stripe account documentation](https://stripe.com/docs/api/accounts) for more information.

### Custom Model

If you want to use your own model, you have two choices:

- Extend our [model class](../src/Models/StripeAccount.php) and customise it.
- Use your own model class and apply our [interface](../src/Contracts/Connect/AccountInterface.php) and
[connected account trait.](../src/Connect/ConnectedAccount.php)

Whichever approach you use, you will need to set the `stripe.webhooks.model` config value to the
fully-qualified class name of your custom model. You will also need to follow the instructions
about [package migrations.](./installation.md#migrations)

Finally, if you do not want to use Eloquent models, or need to store models in a completely different
way to how we do it, you can [write your own storage adapter.](#custom-storage)

#### Extension

This is an example of extending our own class to change the database connection:

```php
<?php

namespace App\Stripe;

use CloudCreativity\LaravelStripe\Models\StripeAccount;

class Account extends StripeAccount
{
    
    protected $connection = 'stripe';
}
```

#### Without Extension

If you do not want to extend our model class, apply our interface and trait to your own class.
For example:

```php
<?php

namespace App;

use CloudCreativity\LaravelStripe\Contracts\Connect\AccountInterface;
use CloudCreativity\LaravelStripe\Connect\ConnectedAccount;
use Illuminate\Database\Eloquent\Model;

class StripeAccount extends Model implements AccountInterface
{
    use ConnectedAccount;

    // ...
}
```

If your model **does not** use an incrementing primary key, we assume that the primary key is also the Stripe ID.
If your model **uses** incrementing primary keys, we default to `stripe_account_id` as the column name.

If you neither of these assumptions are correct, implement the `getStripeAccountKeyName()`. 
For example:

```php
<?php

namespace App;

use CloudCreativity\LaravelStripe\Contracts\Connect\AccountInterface;
use CloudCreativity\LaravelStripe\Connect\ConnectedAccount;
use Illuminate\Database\Eloquent\Model;

class StripeAccount extends Model implements AccountInterface
{
    use ConnectedAccount;

    public function getStripeAccountKeyName()
    {
        return 'account_id';
    }
}
```

#### Custom Storage

If you need to complete customise how connected accounts are stored, create an adapter that implements our
[adapter interface](../src/Contracts/Connect/AdapterInterface.php), or extends
[our adapter implementation](../src/Connect/Adapter.php).

You will then need to register your adapter in the `register()` method of your application's
service provider:

```php
namespace App\Providers;

use CloudCreativity\LaravelStripe\LaravelStripe;
use Illuminate\Support\ServiceProvider;

class AppServiceProvider extends ServiceProvider
{

  public function register()
  {
        LaravelStripe::connect(\App\Stripe\Connect\Adapter::class);
  }
}
```

## Authorization

We provide the tools you need to set up the Stripe Connect
[OAuth connection flow.](https://stripe.com/docs/connect/standard-accounts#oauth-flow)
This allows you to connect `standard` accounts to your application.

### Step 1: Create the OAuth Link

Make sure you have set your application's client id as the `stripe.client_id` config value.

You can then use our `Stripe::authorizeUrl()` method to create the Stripe Connect
authorize URL - either for use as a string or a redirect. We take care of filling the `state`
parameter in the URL with the CSRF token.

For example, creating a `read_write` scope redirect in a controller action:

```php
return Stripe::authorizeUrl()->readWrite()->redirect();
```

Or in a Blade template:

```blade
<a href="{{ Stripe::authorizeUrl()->readWrite() }}">Connect to Stripe</a>
```

> Want to use a Connect button for your link? [Publish brand assets](./installation.md#brand-assets)
and they will be available in the `public/vendor/stripe/brand/connect-button` folder.

You can fluently call any of the following methods after `authorizeUrl()` to set
[Connect OAuth parameters](https://stripe.com/docs/connect/oauth-reference#get-authorize)
on the link:

| Method | Description |
| :-- | :-- |
| `express()` | Use the `express` authorize URL. If not called then the `standard` URL will be used. |
| `readOnly()` | Set the `scope` to `read_only` |
| `readWrite()` | Set the `scope` to `read_write` |
| `redirectUri($uri)` | Set the URI that the user is redirected back to. This **must** match a URI set in your Stripe account. |
| `login()` | Expect your users to have a Stripe account already (e.g., most read-only applications, like analytics dashboards or accounting software). This sets the `stripe_landing` to `login`. |
| `register()` | Set the `stripe_landing` to `register` (the default). |
| `alwaysPrompt()` | Always ask the user to connect, even if they're already connected. Sets `always_prompt` to `true`. | 
| `user($values)` | Set key/value pairs for the `stripe_user` parameter. This method accepts any value accepted by Laravel's `collect()` method. |

For example, if we wanted a `read_write` scope and had passed a `$stripeUser` value into our template: 

```blade
<a href="{{ Stripe::authorizeUrl()->readWrite()->user($stripeUser) }}">Connect to Stripe</a>
```

### Step 2: User Creates or Connects their Account

This step occurs on Stripe's website.

### Step 3: User Redirected Back to Your Site

Register a redirect URI (or multiple) in your Stripe application's settings (available via the
Stripe dashboard).

You will then need to create routes for each endpoint using our `Stripe::oauth()` helper:

```php
// e.g. in routes/web.php
Stripe::oauth('stripe/connect/authorize');

// can chain route methods, e.g.
Stripe::oauth('stripe/connect/authorize')->middleware('auth');
```

#### Account Owner

By default we assume that owner of the account being authorized is the user who is signed in 
when we receive the request at this step. (We check the CSRF token to ensure the signed in
user's session matches the one for which the link was created in Step 1.)

If Stripe accounts belong to a different model in your application, you will need
to set a *current user resolver* in the `register()` method of your application's service provider:

```php
namespace App\Providers;

use CloudCreativity\LaravelStripe\LaravelStripe;
use Illuminate\Support\ServiceProvider;

class AppServiceProvider extends ServiceProvider
{

  public function register()
  {
        LaravelStripe::currentOwnerResolver(function (\Illuminate\Http\Request $request) {
            // return the current owner, ensuring it is an instance of 
            // \CloudCreativity\LaravelStripe\Contracts\Connect\AccountOwnerInterface
        });
  }
}
```

#### Views

You can configure the views to use at this step in your `stripe.connect.views` array, as we
do not provide views for you.

We pass the account owner into all these views as the `$owner` property. We also dispatch
event before rendering, and these events have a `with()` method, allowing you to attach
additional data to pass into the view if you need. 

> Make sure any listeners providing data for the views are not queued!

#### Success

Assuming no error occurred, we:

- Dispatch the `\CloudCreativity\LaravelStripe\Events\OAuthSuccess` event.
- Dispatch a job to the queue to perform the next step.
- Return a `200 OK` response with the view set in the `stripe.connect.views.success` key.

#### Errors

If Stripe indicates an error occurred, we:

- Dispatch the `\CloudCreativity\LaravelStripe\Events\OAuthError` event. This provides
access to the error details provided from  Stripe.
- Return a `422 Unprocessable Entity` response with the view set in the
`stripe.connect.views.error` key.

#### Forbidden

If the `state` parameter from Step 1 does not match the state parameter in this step, we:
 
- Dispatch the `\CloudCreativity\LaravelStripe\Events\OAuthError` event, with the
code set to `laravel_stripe_forbidden`.
- Return a `403 Forbidden` response with the view set in the `stripe.connect.views.error` key.

### Step 4: Fetch the User's Credentials from Stripe

The job we queued at Step 3 will complete the process by fetching the user's credentials from Stripe,
and ensuring that the Stripe account id and refresh tokens are stored.

> If you need to customise the queue connection and queue name to use for this job, set the
`stripe.connect.queue_connection` and `stripe.connect.queue` config values.

## Repositories

You can fluently access the Stripe API for the connected account using our 
[repositories](./repositories.md#stripe-connect) implementation.

For example:

```php
$charges = $model->stripe()->charges()->all();
```

Refer to the [Repositories](./repositories.md) chapter for more details.

## Webhooks

Our [Webhook](./webhooks.md) implementation is fully compatible for Connect webhooks. Refer
to that chapter for details.

## Deauthorizing Accounts

An account can be deauthorized as follows:

```php
// via the facade...
Stripe::connect($accountId)->deauthorize();

// or if you have access to the account model...
$account->stripe()->deauthorize();
```

This will fire a `\CloudCreativity\LaravelStripe\Events\AccountDeauthorized` event. Additionally,
the connected account model will:

- have its refresh token set to `null`;
- have its token scope set to `null`; and
- if it is soft-deleteable, it will be soft-deleted.

> We do not delete the model from the database because this would prevent you from queuing any
listeners or jobs that need access to the account model. If you want to remove the account from
your database when it is deauthorized, you should implement that logic yourself via a queued
event listener. 


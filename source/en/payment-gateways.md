---
extends: _layouts.developer_resources
section: content
locale: en
---

# Payment Gateways

## Adding payment gateways

Payment Driver Template.

So you want to make a payment driver for invoice ninja, but don't know where to start? The first step would be to reach out to us directly on Slack https://invoiceninja.slack.com and have a chat to us in real time so that we can help you hit the ground running and build your driver in the most efficient way possible. Contacting us prior will also ensure that your code can be merged back into the official repository as we will be maintaining this code into the future.

The second step is to understand that there are two components to building the driver:

1. The backend implementation that communicates with the Payment Gateway.
2. The front end implementation that allows the end user to configure the payment gateway itself with api credentials / payment method selections.

This guide focuses on the backend implementation, after you have completed this, you can communicate with our front end teams for the next steps in how to show the gateway in the React/Flutter interface .

Ready? Lets go!

### Step 1. Setup environment

You should update your code to be up to date with the <a href="https://github.com/invoiceninja/invoiceninja/tree/v5-develop">v5-develop</a> branch.

You will then want to create your own branch for for your driver ie.

```
git branch my_payment_driver
```

### Step 2. Adding the gateway into the gateways table

Lets create a migration file which will insert a record identifying the gateway.

```
php artisan make:migration my_new_gateway
```

Let open this file and in the up() method create our new gateway record

Init a new gateway instance

```
$gateway = new Gateway;
$gateway->name = 'Fancy Gateway'; 
$gateway->key = Str::lower(Str::random(32)); 
$gateway->provider = ‘FancyGateway’;
$gateway->is_offsite = true;
$gateway->fields = new \stdClass;
$gateway->visible = true;
$gateway->site_url = ‘https://stripe.com’;
$gateway->default_gateway_type_id = 1;
$gateway->save();
```

#### Gateway Properties

* name: The name of your gateway
* key: A random 32 alphanumeric gateway key (Type: string)
* provider: This is a camel cased string which is used to initialize your payment driver. We append the string Driver to this class, so if you payment driver is FancyGatewayDriver, then your provider will be FancyGateway. (Type: string)
* is_offsite: Specifies is this payment driver redirects the user to another page to complete the payment. Paypal Express for instance redirects to Paypal, and then returns the user once the payment is completed (Type: bool)
* fields: A stdClass object of key values which defines the user settings required for the gateway, ie, Api Keys, Secrets etc. All of these fields are strings except for testMode which is a boolean and indicates whether the gateway is set to test mode. (Type stdClass)
* visible: Defines whether the gateway should be visible in the UI (Type: bool)
* site_url: A URL field which allows the user to go directly to the gateway page for further information (Type: string, url)
* default_gateway_type_id: If your gateway has multiple ways to pay, ie Credit Card, Bank Transfer etc, then you’ll want to select a default method. The list of defined methods are found on the GatewayType model as follows:

```
const CREDIT_CARD = 1;
const BANK_TRANSFER = 2;
const PAYPAL = 3;
const CRYPTO = 4;
const CUSTOM = 5;
const ALIPAY = 6;
const SOFORT = 7;
const APPLE_PAY = 8;
const SEPA = 9;
const CREDIT = 10;
```

### Step 3. App\Models\Gateway.php Model Getters and Setters

Two methods need to be appended to:

1.  ```getHelp()``` returns a link to the gateways help page, we display a link in the UI for the user to open a direct webpage to the gateway.
2.  ```getMethods()``` returns an array of the supported gateway types (ie payment methods), whether the gateway supports refunds and token billing and also webhook meta data. The structure of the array looks like this:

```php
[
  [GatewayType::CREDIT_CARD => ['refund' => true, 'token_billing' => true]],
  [GatewayType::BANK_TRANSFER => ['refund' => true, 'token_billing' => true, 'webhooks' => ['source.chargeable']]]
];
```

The array is stored in a case/switch block, which switches on the gateway->id property.

### Step 4. Starting work on the Payment Driver

All payment drivers must extend the BaseDriver class which itself extends the abstract class AbstractPaymentDriver which enforces the following required methods. We have stubbed an example payment driver class and view files which can be downloaded [here](/assets/files/PaymentDriver.zip)

```php
abstract public function authorizeView(array $data);

abstract public function authorizeResponse(Request $request);

abstract public function processPaymentView(array $data);

abstract public function processPaymentResponse(Request $request);

abstract public function refund(Payment $payment, $refund_amount, $return_client_response = false);

abstract public function tokenBilling(ClientGatewayToken $cgt, PaymentHash $payment_hash);

abstract public function setPaymentMethod($payment_method_id);
```

* authorizeView() returns a view which enables capture of a token for a particular payment method, ie Credit Card or Bank Transfer

To understand the layouts of the UI, it is worthwhile inspecting the other payment driver layouts in resources/views/portal/ninja2020/gateways.

All layouts extend from the following

```php
@extends('portal.ninja2020.layout.payments', ['gateway_title' => ctrans('texts.credit_card'), 'card_title' => ctrans('texts.credit_card')])
```

* authorizeReponse() processes the gateway response and if successful creates a ```ClientGatewayToken``` record followed by returning the user to the following route

```php
return redirect()->route('client.payment_methods.index');
```

* processPaymentView() returns a view which enables capture of a payment

* processPaymentResponse() processes the gateway response and if successful creates a ```Payment``` record followed by returning the user to the payment route here:

```php
return redirect()->route('client.payments.show', ['payment' => $this->stripe->encodePrimaryKey($payment->id)]);
```

* refund() attempts to refund a payment and takes three parameters,

1. The Payment Model (Collection)
2. The Refund amount (Float)
3. Whether the response requires a client response (Boolean)

* tokenBilling() attempts to process a token charge for a given amount

* setPaymentMethod() this method is used to set the payment method on the driver class, this is needed in gateway classes where there are multiple payment method options in the gateway ie. Credit Card, Bank Transfer.

The BaseDriver class itself contains several helper methods which allow the creation of Payment records in Invoice Ninja, these are defined as follows:

```php
public function storeGatewayToken(array $data, array $additional = []): ?ClientGatewayToken
```

This method is used to store a token generated by a payment gateway, it requires an array of parameters with the following definition:

```php
[
    'token', // (string),
    'payment_method_id', // (ie GatewayType::CREDIT_CARD), 
    'payment_meta', // stdClass object as defined below
]

$payment_meta = new \stdClass;
$payment_meta->exp_month = (string) $method->card->exp_month;
$payment_meta->exp_year = (string) $method->card->exp_year;
$payment_meta->brand = (string) $method->card->brand;
$payment_meta->last4 = (string) $method->card->last4;
$payment_meta->type = GatewayType::CREDIT_CARD;
```

To improve abstraction, we encourage the development of the actual payment gateway implementation into its own namespace. Once you have completed processing a gateway response, you'll need to perform some additional work this could include:

1. Returning a successful payment response to the end user
2. Process a refund
3. Store a client gateway token
4. Process a failed payment response to the end user

Invoice ninja provides the entry point for these in the BaseDriver class, the exact data required in specified as above, the rest is merged from data already within the driver itself.

#### 1. Handling a successful payment response

Invoice ninja uses a small glue class called a PaymentHash, which links the payment meta data with a hash, once you have returned from your gateway, you'll need to rehydrate the payment hash object, it will be returned by the gateway in the request variable `payment_hash` using a binary search as follows

```php
$payment_hash = PaymentHash::whereRaw('BINARY `hash`= ?', [$request->input('payment_hash')])->firstOrFail();
```

At this point you will need to create a payment record, this can be passed directly to the BaseDriver method defined below

```php 
public function createPayment(array $data, $status = Payment::STATUS_COMPLETED): Payment
```

The data array here requires the following properties to be passed in from you custom payment driver

```php
[
    'gateway_type_id', // (ie GatewayType::CREDIT_CARD) 
    'amount', // (float) see below
    'payment_type', // (ie PaymentType::CREDIT_CARD_OTHER)
    'transfaction_reference',
]
```

The amount key is hydrated from the payment hash the follow query should be used to determine the amount

```php
array_sum(array_column($payment_hash->invoices(), 'amount')) + $payment_hash->fee_total;
```

In addition to creating the Payment record, we highly recommend logging the full output from the gateway to enable debugging for future purpose, this is done via the SystemLogger::job() which is defined as follows

```php
public function __construct(array $log, int $category_id, int $event_id, int $type_id, ?Client $client)
```

The array is the gateway response, bundled with any other metadata you would like to add, the remaining properties are the const values defined in SystemLog, these define the category, event and type of log. Feel free to create additional categories using the template in the SystemLog model class.

#### 2. Process a refund

The refund method is implemented in your PaymentDriver class with the following method

```php
public function refund(Payment $payment, $refund_amount, $return_client_response = false);
```

You may need the $payment class to pass in the transaction_reference to your gateway, along with the refund_amount, the return object here is a simple array of data on success, or throw an exception with an appropriate message.


#### 3. Store a client gateway token

Once you have generated a gateway token, you'll need to store this, a helper method in the BaseDriver is defined here:


```php
public function storeGatewayToken(array $data, array $additional = []): ?ClientGatewayToken
```

The properties required for the data array are as follows:

```php
[
  'token',
  'payment_method_id',
  'payment_meta',
  'payment_method_id', // ie. GatewayType::CREDIT_CARD
  'gateway_customer_reference', // optional
]
```

#### 4. Process a failed payment response to the end user

A generic expection is provided when you encounter a fatal gateway error whilst processing a payment

```php
throw new PaymentFailed('Failed to process the payment.', 500);
```

Along with this exception, it is also required that you dispatch a PaymentFailureMailer::job() defined as follows

```php
PaymentFailureMailer::dipatch($client, $error, $company, $payment_hash)
```

### Environment Setup

#### Stub Gateway Configuration

When building the driver, you will want a way to preconfigure a gateway with the required testing credentials. As the UI for this is not built, you will want to manually create a company gateway record using the following schema:

```php
$cg = new CompanyGateway();
$cg->company_id = $company->id;
$cg->user_id = $user->id;
$cg->gateway_key = 'insert_your_gateway_hash_here';
$cg->require_cvv = true;
$cg->require_billing_address = true;
$cg->require_shipping_address = true;
$cg->update_details = true;
$cg->config = encrypt('{"apiKey":"api_key_value","anotherKey":"another_value"}');
$cg->save();
```
# Quickstart

## Selling Products

:::tip

Before utilizing Stripe Checkout, you should define Products with fixed prices in your Stripe dashboard. In addition, you should [configure Shopkeeper's webhook handling](./webhooks).

:::

Offering product and subscription billing via your application can be intimidating. However, thanks to Shopkeeper and [Stripe Checkout](https://stripe.com/payments/checkout), you can easily build modern, robust payment integrations.

To charge customers for non-recurring, single-charge products, we'll utilize Shopkeeper to direct customers to Stripe Checkout, where they will provide their payment details and confirm their purchase. Once the payment has been made via Checkout, the customer will be redirected to a success URL of your choosing within your application:

```ts
// title: start/routes.ts
import router from '@adonisjs/core/services/router'

router.get('/checkout', async ({ auth, response }) => {
  const user = auth.getUserOrFail()

  const stripePriceId = 'price_mystripeprice'
  const quantity = 1

  const checkout = await user.checkout(
    { [stripePriceId]: quantity },
    {
      success_url: router.makeUrl('checkout-success'),
      cancel_url: router.makeUrl('checkout-cancel'),
    }
  )

  response.redirect().toPath(checkout.sessionUrl())
})

router.on('/checkout/success').render('checkout/success').as('checkout-success')
router.on('/checkout/cancel').render('checkout/cancel').as('checkout-cancel')
```

As you can see in the example above, we will utilize Shopkeeper's provided `checkout` method to redirect the customer to Stripe Checkout for a given "price identifier". When using Stripe, "prices" refer to [defined prices for specific products](https://stripe.com/docs/products-prices/how-products-and-prices-work).

If necessary, the `checkout` method will automatically create a customer in Stripe and connect that Stripe customer record to the corresponding user in your application's database. After completing the checkout session, the customer will be redirected to a dedicated success or cancellation page where you can display an informational message to the customer.

### Providing Meta Data to Stripe Checkout

When selling products, it's common to keep track of completed orders and purchased products via `Cart` and `Order` models defined by your own application. When redirecting customers to Stripe Checkout to complete a purchase, you may need to provide an existing order identifier so that you can associate the completed purchase with the corresponding order when the customer is redirected back to your application.

To accomplish this, you may provide an array of `metadata` to the `checkout` method. Let's imagine that a pending `Order` is created within our application when a user begins the checkout process. Remember, the `Cart` and `Order` models in this example are illustrative and not provided by Shopkeeper. You are free to implement these concepts based on the needs of your own application:

```ts
// title: start/routes.ts
import router from '@adonisjs/core/services/router'
import Cart from '#models/cart'
import Order from '#models/order'

router.get('/cart/:id', async ({ auth, request response }) => {
  const user = auth.getUserOrFail()
  const cart = await Card.findOrFail(request.get('id'))

  const order = await Order.create({
    cartId: cart.id,
    priceIds: cart.priceIds,
    status: 'incomplete'
  })

  const checkout = await user.checkout(
    order.priceIds,
    {
      success_url: router.makeUrl('checkout-success', { sessionId: '{CHECKOUT_SESSION_ID}'}),
      cancel_url: router.makeUrl('checkout-cancel'),
      metadata: { orderId: order.id }
    }
  )

  response.redirect().toPath(checkout.sessionUrl())
})
```

As you can see in the example above, when a user begins the checkout process, we will provide all of the cart / order's associated Stripe price identifiers to the `checkout` method. Of course, your application is responsible for associating these items with the "shopping cart" or order as a customer adds them. We also provide the order's ID to the Stripe Checkout session via the `metadata` array. Finally, we have added the `CHECKOUT_SESSION_ID` template variable to the Checkout success route. When Stripe redirects customers back to your application, this template variable will automatically be populated with the Checkout session ID.

Next, let's build the Checkout success route. This is the route that users will be redirected to after their purchase has been completed via Stripe Checkout. Within this route, we can retrieve the Stripe Checkout session ID and the associated Stripe Checkout instance in order to access our provided meta data and update our customer's order accordingly:

```ts
// title: start/routes.ts
import router from '@adonisjs/core/services/router'
import Cart from '#models/cart'
import Order from '#models/order'

router.get('/checkout/success', async ({ auth, request response, view }) => {
  const user = auth.getUserOrFail()
  const sessionId = request.get('sessionId')

  const session = shopkeeper.stripe.checkout.sessions.retrieve(sessionId)
  const orderId = session.metadata.orderId

  const order = await Order.findOrFail(orderId)
  order.status = 'completed'
  await order.save()

  return view('checkout/success')
})
```

## Selling Subscriptions

:::tip

Before utilizing Stripe Checkout, you should define Products with fixed prices in your Stripe dashboard. In addition, you should [configure Shopkeeper's webhook handling](./webhooks).

:::

Offering product and subscription billing via your application can be intimidating. However, thanks to Shopkeeper and [Stripe Checkout](https://stripe.com/payments/checkout), you can easily build modern, robust payment integrations.

To learn how to sell subscriptions using Shopkeeper and Stripe Checkout, let's consider the simple scenario of a subscription service with a basic monthly (`price_basic_monthly`) and yearly (`price_basic_yearly`) plan. These two prices could be grouped under a "Basic" product (`pro_basic`) in our Stripe dashboard. In addition, our subscription service might offer an Expert plan as `pro_expert`.

First, let's discover how a customer can subscribe to our services. Of course, you can imagine the customer might click a "subscribe" button for the Basic plan on our application's pricing page. This button or link should direct the user to a Adonis route which creates the Stripe Checkout session for their chosen plan:

```ts
// title: start/routes.ts
import router from '@adonisjs/core/services/router'
import Cart from '#models/cart'
import Order from '#models/order'

router.get('/subscription-checkout', async ({ auth, request response, view }) => {
  const user = auth.getUserOrFail()

  const checkout = await user
    .newSubscription('default', 'price_basic_monthly')
    .trialDays(5)
    .allowPromotionCodes()
    .checkout({
      success_url: router.makeUrl('checkout.success'),
      cancel_url: router.makeUrl('checkout.cancel'),
    })

  response.redirect().toPath(checkout.sessionUrl())
})
```

As you can see in the example above, we will redirect the customer to a Stripe Checkout session which will allow them to subscribe to our Basic plan. After a successful checkout or cancellation, the customer will be redirected back to the URL we provided to the `checkout` method. To know when their subscription has actually started (since some payment methods require a few seconds to process), we'll also need to [configure Shopkeepers's webhook handling](./webhooks).

Now that customers can start subscriptions, we need to restrict certain portions of our application so that only subscribed users can access them. Of course, we can always determine a user's current subscription status via the `subscribed` method provided by Shopkeeper's `Billable` trait:

```edge
@if(await auth.user.subscribed())
  <p>You are subscribed.</p>
@end
```

We can even easily determine if a user is subscribed to specific product or price:

```edge
@if(await auth.user.subscribedToProduct('pro_basic'))
  <p>
    You are subscribed to our Basic product.
  </p>
@end

@if(await auth.user.subscribedToProduct('price_basic_monthly'))
  <p>
    You are subscribed to our monthly Basic plan.
  </p>
@end
```

### Building a Subscription Middleware

For convenience, you may wish to create a [middleware](https://docs.adonisjs.com/guides/basics/middleware) which determines if the incoming request is from a subscribed user. Once this middleware has been defined, you may easily assign it to a route to prevent users that are not subscribed from accessing the route:

```ts
// title: app/middlewares/subscribe_middleware.ts
import { HttpContext } from '@adonisjs/core/http'
import { NextFn } from '@adonisjs/core/types/http'

export default class SubscribedMiddleware {
  async handle({ auth, response }: HttpContext, next: NextFn) {
    const user = auth.getUser()
    const subscribed = await user?.subscribed()

    if (!subscribed) {
      response.redirect().toRoute('home')
    } else {
      await next()
    }
  }
}
```

Once the middleware has been defined, you may register it and assign it to a route:

```tsx
// title: start/kernel.ts
router.named({
  subscribed: () => import('#middleware/subscribed_middelare'),
})
```

```tsx
// title: start/routes.ts
router.get('/dashboard').middleware(middleware.subscribed())
```

### Allowing Customers to Manage Their Billing Plan

Of course, customers may want to change their subscription plan to another product or "tier". The easiest way to allow this is by directing customers to Stripe's [Customer Billing Portal](https://stripe.com/docs/no-code/customer-portal), which provides a hosted user interface that allows customers to download invoices, update their payment method, and change subscription plans.

First, define a link or button within your application that directs users to an Adonis route which we will utilize to initiate a Billing Portal session:

```edge
<a href="{{ route('billing') }}">
  Billing
</a>
```

Next, let's define the route that initiates a Stripe Customer Billing Portal session and redirects the user to the Portal. The `billingPortalUrl` method accepts the URL that users should be returned to when exiting the Portal:

```ts
import router from '@adonisjs/core/services/router'

router
  .get('/billing', ({ auth, response }) => {
    const user = auth.getUserOrFail()
    const billingPortalUrl = await user.billingPortalUrl(route('dashboard'))

    response.redirect().toPath(billingPortalUrl)
  })
  .middleware(middleware.auth())
  .name('billing')
```

:::warning

As long as you have configured Shopkeeper's webhook handling, Shopkeeper will automatically keep your application's Shopkeeper-related database tables in sync by inspecting the incoming webhooks from Stripe. So, for example, when a user cancels their subscription via Stripe's Customer Billing Portal, Shopkeeper will receive the corresponding webhook and mark the subscription as "canceled" in your application's database.

:::

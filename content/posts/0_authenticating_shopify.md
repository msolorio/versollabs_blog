+++
date = '2024-11-24T15:17:33-08:00'
draft = false
title = 'Authenticating a Shopify app with Express.js'
+++

<!-- While building my Shopify backend application, one the first major tasks was configuring OAuth flow integration with Shopify.  -->

While building the initial backbone of the Junta platform, one one the first major tasks was to collect data from common ecommerce solutions like Shopify and Google Ads. Our goal is to unify this data behind a single user-friendly product for the purpose of helping small businesses track trends and derive insight. Shopify, being a popular platform with a rich API for deriving insight from storefront data, made a good candidate to start.

The Shopify dev team maintains the `shopify-app-js` tool suite, including [@shopify/shopify-api](https://github.com/Shopify/shopify-app-js/tree/main/packages/apps/shopify-api#readme), a framework and runtime agnostic library that hooks into Shopify's OAuth flow. I used this as the basis for building an Express middleware that I could use in the Junta project and my own scalable applications with Shopify.

The result is a lightweight ExpressJS middleware library that makes it easy to manage OAuth with Shopify and stores access tokens for access to the [Shopify Admin API](https://shopify.dev/docs/api/admin-graphql).


<!-- TODO:
Show example client implementation for how the middleware should be used.
Talk about how clean and easy to use the interface is and our reasoning for designing it the way we did.
 -->

```ts
import { ShopifyAuth, MongoDbSessionStore } from '@versollabs/shopify-auth-express';

const app = express();
app.use(bodyParser.json());

const shopifyAuth = ShopifyAuth({ // Configure `ShopifyAuth`
  api: {
    apiKey: String(process.env.CLIENT_ID),
    apiSecretKey: String(process.env.CLIENT_SECRET),
    hostName: String(process.env.HOSTNAME),
    scopes: ['read_products', 'read_orders'],
  },
  authPaths: {
    begin: '/auth',
    callback: '/auth/callback',
  },
  sessionStore: MongoDbSessionStore({ // Choose a session store (MongoDB, PostgreSQL, Redis)
    url: String(process.env.MONGODB_URI),
    dbName: 'shopify',
    collectionName: 'shops',
  }),
});

app.use(shopifyAuth.router()); // Use the router middleware in your Express app

// Once storefront has installed your app call `getAccessToken` to get an access token for the store.
const accessToken = await shopifyAuth.getAccessToken(storeName);
```

---

<!-- The entrypoint to the library exposes only two methods, encapsulating the complexity of OAuth flow management. One initializes the auth router to be used in the client's ExpressJS application and the other is for retrieving access tokens. -->

```ts
// src/index.ts
export const ShopifyAuth = function (options: ShopifyAuthOptions) {
  return new _ShopifyAuth(options);
}

class _ShopifyAuth {
  private _sessionStore: AbstractSessionStore;
  private _authRouter: ShopifyAuthRouter;

  constructor(options: ShopifyAuthOptions) {
    this._sessionStore = options.sessionStore;
    this._authRouter = new ShopifyAuthRouter({
      api: options.api,
      authPaths: options.authPaths,
      sessionStore: this._sessionStore,
      fakeShopifyApi: options.fakeShopifyApi,
    })
  }

  public async getAccessToken(shopName: string) {
    const shop = await this._sessionStore.get(shopName);
    return String(shop && shop.accessToken);
  }

  public router() {
    return this._authRouter.create();
  }
}
```

---

The meat of the middleware library is the router. This provides the client ExpressJS application with two additional routes to handle the OAuth flow with Shopify, the "begin" and "callback" routes. The router class attaches them in the `create()` method and builds on top of Shopify's official package, `@shopify/shopify-api`.

```ts
import { shopifyApi } from '@shopify/shopify-api'

export class ShopifyAuthRouter {
  private _shopify: NarrowedShopifyObject;
  private _shopifyApi: NarrowedShopifyApi;
  private _authPaths: ShopifyAuthPaths;
  private _sessionStore: AbstractSessionStore;

  constructor({ api, authPaths, sessionStore, fakeShopifyApi }: {
    api: ShopifyAuthApi,
    authPaths: ShopifyAuthPaths,
    sessionStore: AbstractSessionStore,
    fakeShopifyApi?: NarrowedShopifyApi
  }) {
    this._shopifyApi = fakeShopifyApi || shopifyApi;
    this._shopify = this._shopifyApi({
      apiKey: api.apiKey,
      apiSecretKey: api.apiSecretKey,
      scopes: api.scopes,
      hostName: api.hostName,
      apiVersion: LATEST_API_VERSION,
      isEmbeddedApp: false,
      logger: {
        level: LogSeverity.Error,
      }
    });
    this._authPaths = authPaths;
    this._sessionStore = sessionStore;
  }

  public create() {
    const router = Router();

    router.get(this._authPaths.begin, async (req, res) => {
      const shop = String(req.query.shop);
      await this._begin(shop, req, res);
    });

    router.get(this._authPaths.callback, async (req, res) => {
      await this._callback(req, res);
      res.status(200).send('You have approved the app.');
    });

    return router;
  }

  private async _begin(shopName: string, req: Request, res: Response) {
    await this._shopify.auth.begin({
      shop: shopName,
      callbackPath: this._authPaths.callback,
      isOnline: false,
      rawRequest: req,
      rawResponse: res,
    });
  }

  private async _callback(req: Request, res: Response) {
    const callback = await this._shopify.auth.callback({
      rawRequest: req,
      rawResponse: res,
    });
    const shop: Shop = {
      shopName: callback.session.shop,
      accessToken: String(callback.session.accessToken),
    }

    await this._sessionStore.add(shop);
  }
}
```

The "begin" route is triggered when a Shopify storefront owner clicks to install our Shopify app. `this._shopify.auth.begin` (from Shopify's official package) will redirect the storefront owner to the Shopify Authentication screen, where they can approve the data from their store the app is requesting.

```ts
...

  private async _begin(shopName: string, req: Request, res: Response) {
    await this._shopify.auth.begin({
      shop: shopName,
      callbackPath: this._authPaths.callback,
      isOnline: false,
      rawRequest: req,
      rawResponse: res,
    });
  }

...
```

Once the merchant approves, Shopify will redirect them back to our app at the "callback" route with a temporary authorization grant. `this._shopify.auth.callback` completes the OAuth process by exchanging the temporary authorization grant for a permanent access token. We then persist the token in the session store where it can later be retrieved for access to the Shopify Admin API.

```ts
...

  private async _callback(req: Request, res: Response) {
    const callback = await this._shopify.auth.callback({
      rawRequest: req,
      rawResponse: res,
    });
    const shop: Shop = {
      shopName: callback.session.shop,
      accessToken: String(callback.session.accessToken),
    }

    await this._sessionStore.add(shop);
  }

...
```
<!-- 
TODO:
Show diagram of the OAuth flow with "begin" and "callback" routes
 -->

<!--
TODO: talk about session stores and the reasoning behind them
- Dependency inversion principle
- Testing
- Flexibility
-->

<!-- 
TODO: Talk about NarrowedShopifyObject type
- Dependency inversint principle
- Allows for in memory testing of the library
Type Narrowing ensures that our fake and the real shopify api object conform to the same specification.
 -->

<!-- Identify the meat of what we're going to be talking about -->
<!-- Giving an overview of the mdidleware library we created -->
  <!-- Talk about motivation for creating library -->
<!-- explain how auth with shopify api works under the hood -->
  <!-- how OAuth 2.0 works -->

<!-- - Go through the process of integrating express.js with Shopify to test auth (ngrok, shopify parterns dashboard) -->


<!-- - At its core this library is a router that provides two additional routes auth flow with shopify -->

<!-- discuss testing strategy -->
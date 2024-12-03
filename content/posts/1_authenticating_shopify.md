+++
date = '2024-11-24T15:17:33-08:00'
draft = false
title = 'Authenticating a Shopify app with ExpressJS'
+++

While I was building the initial backbone for the Junta platform, one of my first major initiatives was to collect storefront data from ecommerce solutions like Shopify and Google Ads. With Junta, our goal is to unify this data and expose behind a user-friendly product for helping small businesses track trends and derive insight. I decided Shopify, being a popular platform with a rich API made a good candidate to start.

The Shopify dev team maintains the `shopify-app-js` tool suite, including [@shopify/shopify-api](https://github.com/Shopify/shopify-app-js/tree/main/packages/apps/shopify-api#readme), a framework agnostic library for TypeScript and JavaScript backend apps to hook into Shopify's OAuth flow using [authorization code grant](https://shopify.dev/docs/apps/build/authentication-authorization/access-tokens/authorization-code-grant). This package will allow a storefront owner to make varified requests to our application in order to approve our use of their storefront data.

I used the `@shopify/shopify-api` library as the basis for building an Express middleware that I could use in the Junta project and my own projects that integrate with Shopify. The result is a lightweight ExpressJS library for non-embedded Shopify apps that makes it easy to manage OAuth and access to the [Shopify Admin API](https://shopify.dev/docs/api/admin-graphql).

You can find the full code in [GitHub](https://github.com/msolorio/shopify-auth-express-middleware?tab=readme-ov-file#shopify-auth-express-middleware) and the library [now published on the npm registry](https://www.npmjs.com/package/@versollabs/shopify-auth-express-middleware).

---

## Client configuration

One of my goals was to make the middleware easy to use and for setup, a client only needs to provide the necessary configuration options and then include the auth router in their app. The client configures the paths used for managing OAuth which we will discuss in depth further in this post, and can choose from a few built in options for token storage, passing the URL of their storage component.

```ts
// example client implementation
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

// Once storefront has installed your app
// call `getAccessToken` to get an access token for the store.
const accessToken = await shopifyAuth.getAccessToken(storeName);
```

---

## OAuth router

The meat of this middleware library is the router. The router provides the ExpressJS application with two additional routes to handle OAuth flow, "begin" and "callback", and the paths for these routes are passed by the client application on initial configuration (`authPaths.begin` and `authPaths.callback`). The router class attaches them in the `create()` method and builds on Shopify's official API library, `@shopify/shopify-api`.

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

---

## OAuth flow in depth

Let's take a more fine-grained look at the OAuth flow.

A request is made to the "begin" route when a Shopify storefront owner clicks to install the client's Shopify app. This initial request to our app kicks off the OAuth flow and sends in query parameters, the name of the shop, the timestamp of the request, and an HMAC signature.

```bash
GET /auth?hmac=14d14557a355757470aa16f46cee79a83b664a1d9cf43b32916810cbd60ba688&shop=versol-test3.myshopify.com&timestamp=1733180678
```

`this._shopify.auth.begin` will verify the request by passing the shop, the timestamp, and our client secret through the HMAC-SHA256 hash function and compare it to the value of the `hmac` query parameter. If the request is valid the storefront owner will be redirected to the Shopify Authentication screen, where they can approve our app for accessing their storefront's data for the scopes requested.

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

Once the merchant approves, Shopify redirects them back to the app at the "callback" route and a temporary authorization code (`code`) is sent in the query parameter of the request.

```bash
GET /auth/callback?code=52798b42a28e84e856f8f2ca33e3eb65&hmac=43d5c6a79bfec24e6c3e8fa1feb01968a5f703e7c0eeddd05aacb775c5f279b1&host=YWRtaW4uc2hvcGlmeS5jb20vc3RvcmUvdmVyc29sLXRlc3Qz&shop=versol-test3.myshopify.com&state=882167157150937&timestamp=1733180679
```

`this._shopify.auth.callback` completes the OAuth process by making a request to Shopify's OAuth server to exchang the temporary authorization code for a permanent access token. We then persist the access token in the session store where it can later be retrieved for access to the Shopify Admin API.

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

The diagram below illustrates the full OAuth flow.

![OAuth flow with authorization code grant](/images/1_authenticating_shopify/oauth_flow.png)

---

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

<!-- discuss testing strategy -->
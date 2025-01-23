+++
date = '2024-11-24T15:17:33-08:00'
draft = false
title = 'Managing OAuth flow with Shopify and ExpressJS'
+++

While building the initial backbone for the Junta platform, one of my first major initiatives was to collect storefront data from ecommerce solutions like Shopify and Google Ads. With Junta, our goal is to unify this data and expose behind a user-friendly product for helping small businesses track trends and derive insight. I decided Shopify, being a popular platform with a rich API made a good candidate to start.

The Shopify dev team maintains the `shopify-app-js` tool suite, including [@shopify/shopify-api](https://github.com/Shopify/shopify-app-js/tree/main/packages/apps/shopify-api#readme), a framework agnostic library for TypeScript and JavaScript backend apps to hook into Shopify's OAuth flow using [authorization code grant](https://shopify.dev/docs/apps/build/authentication-authorization/access-tokens/authorization-code-grant). This package allows storefront owners to make varified requests to our application in order to approve our use of their storefront data.

I used the `@shopify/shopify-api` library as the basis for building an Express middleware that I could use in the Junta project and my own projects that integrate with Shopify. The result is a lightweight ExpressJS library for non-embedded Shopify apps that makes it easy to manage OAuth and access to the [Shopify Admin API](https://shopify.dev/docs/api/admin-graphql).

We'll take an overview of the library's design starting with its usage by a client, how it manages OAuth with Shopify, and discuss how dependency inversion is used to accommodate any number of client use-cases and improve testability.

You can find the full code in [GitHub](https://github.com/msolorio/shopify-auth-express-middleware?tab=readme-ov-file#shopify-auth-express-middleware) and the library [now published on the npm registry](https://www.npmjs.com/package/@versollabs/shopify-auth-express-middleware).


---

## Client configuration

One of my goals was to make the middleware easy to use and for setup a client only needs to provide the necessary configuration options and then include the auth router in their app. The client configures the paths used for managing OAuth which we will discuss in depth further in this post. The client can pass their own behavior for managing access tokens or choose from a few built in options.

```ts
// example client implementation
import { ShopifyAuth } from '@versollabs/shopify-auth-express';
import { CustomSessionStore } from './custom-session-store';

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
  // add custom session store or choose an out of the box one for adding and retrieving access tokens
  sessionStore: new CustomSessionStore(),
});

app.use(shopifyAuth.router()); // Use the router middleware in your Express app

// Once storefront has installed your app
// call `getAccessToken` to get an access token for the store.
const accessToken = await shopifyAuth.getAccessToken(storeName);
```

The client will likely want to provide their own behavior for managing access tokens. They can create a session storage object that implements the `AbstractSessionStore` interface and pass it to the `ShopifyAuth` constructor.

```ts
import { AbstractSessionStore } from '@versollabs/shopify-auth-express';

class CustomSessionStore implements AbstractSessionStore {
  async add(shopName: string): Promise<void> {
    // Add access token to persistent storage
  }

  async get(shopName: string): Promise<string | null> {
    // Get access token from your session store
  }
}

const shopifyAuth = ShopifyAuth({
  ...
  sessionStore: new CustomSessionStore(),
  ...
});
```

---

## OAuth router

The meat of this middleware library is the router. The router provides the ExpressJS application with two additional routes to handle OAuth flow, "begin" and "callback", and the paths for these routes are passed by the client application on initial configuration (`authPaths.begin` and `authPaths.callback`). The router class attaches them in the `create()` method and invokes methods from the `shopifyApi` module to hook into Shopify's OAuth flow.

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

## OAuth flow with Shopify in depth

Let's take a more fine-grained look at the OAuth flow with Shopify.

When a Shopify merchant clicks to install the client's Shopify app they initialize the OAuth flow by sending a GET request to the "begin" route of the Express middleware. 

This request contains within query parameters
- the `shop` name
- a `timestamp`
- the origin `host` making the request Base64 encoded
- and an `HMAC` signature.

```bash
# initial begin request
GET https://client-app/auth/shopify?hmac=907db6f3797d1c99d1f27fee7cd0a6ade97a46166ea9dfa2ba3fa388ab25fc08&host=YWRtaW4uc2hvcGlmeS5jb20vc3RvcmUvdmVyc29sLXRlc3Qz&shop=versol-test3.myshopify.com&timestamp=1735333786
```

The client app needs to verify the request being sent by the user and it does so by removing the hmac query parameter, passing the remaining parameters as a string along with the app's client ID through the HMAC-SHA256 hash function, and comparing the result of the hash function to the value of the hmac query parameter.

If the request is valid, the client app generates a redirect URL to the Shopify's app consent screen, where the user will be asked to approve the app for access to their storefront data.

```bash
# example consent screen redirect URL
GET https://admin.shopify.com/store/versol-test3/oauth/authorize?client_id=abc123456789&scope=read_products%2Creader_orders&redirect_uri=https%3A%2F%2Fexample.com%2Fauth%2Fshopify%2Fcallback&state=832550206527322
```

The consent screen redirect URL contains in query parameters
- the `client_id` identifying the app
- the API `scope`s the app is requesting access to
- the `redirect_uri` to which the user is sent after authorizing the app
- a `state` parameter unique for each authorization request which will be used for verification after the auth code is received

The client app also sets a signed cookie to the value of the `state` parameter. The consent screen redirect URL generation and setting the cookie is managed by the `shopifyApi` module at `this._shopify.auth.begin`.

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

The merchant must authorize the app's use of their storefront data before OAuth flow can proceed.
![Shopify app consent screen](/images/1_authenticating_shopify/shopify_consent_screen.png)

After the merchant authorizes the client app, Shopify redirects the merchant to the `redirect_uri` that was passed earlier with the temporary authorization `code` passed as a query parameter. This `redirect_uri` is the "callback" route of the client app and starts the second phase of the OAuth flow. 

```bash
# request to the callback route containing the temporary code
 GET https://client-app/auth/shopify/callback?code=80d4a45a44702ea84e59d5020f2f787f&hmac=dfd80e53fbe29af6d4f4fbb94eb74a4338a64d179e1cdc0b51aff74b1056b125&host=YWRtaW4uc2hvcGlmeS5jb20vc3RvcmUvdmVyc29sLXRlc3Qz&shop=versol-test3.myshopify.com&state=705654466948045&timestamp=1735337350
```

The client app again needs to verify the request being sent by the user.

The client app
- checks checks the `state` parameter is the same value that was originally sent.
- checks that the signed cookie is present and matches the `state` parameter.
- verifies the `hmac` parameter using the HMAC-SHA256 hash function.
- verfies the `host` parameter after base64 decoding is a valid Shopify shop origin that ends in `.myshopify.com`.

If all checks pass, the client app can now exchange the temporary `code` for a permenant access token by sending a request to the shop's access token endpoint.

```bash
POST https://merchants-shop.myshopify.com/admin/oauth/access_token?client_id=abc123456789&client_secret=987654321xyz&code=80d4a45a44702ea84e59d5020f2f787f
```

Verification and token exchange are managed by the `shopifyApi` module at `this._shopify.auth.callback` at the middleware's callback handler.

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

Here is the full OAuth flow with Shopify.

![OAuth flow with authorization code grant](/images/1_authenticating_shopify/oauth_flow.png)

---

## Dependency inversion

The library allows for passing access token management behavior to the middleware, decoupling it from a specific storage implementation.

The dependency inversion principle states that high-level modules should not depend on low-level modules. Both should depend on abstractions. But what does this mean? In this case the low-level module is the mechanism for access token storage and retrieval. The high-level module is the middleware itself.

The library defines the shape for how a session storage object should look and in doing so defines a contract for the middleware to interact with. A custom storage implementation can conform to this contract and the middleware need not know or care about the implementation details of session storage.

Dependency inversion is useful here because the client's needs for access token storage are likely to be specific to their architecture. In addition to allowing the library to be flexible, dependency inversion has the benefit of allowing the library to be easily tested. By creating an abstraction around low-level data access, and passing a fake session storage object, we can get to a point where the majority of our tests are fast, in-memory unit tests. We can spead up iteration speeds and get feedback early and often.

![dependency inversion](/images/1_authenticating_shopify/dependency_inversion_1.png)

---

See the full code in GitHub and the library now published on the npm registry.

https://github.com/msolorio/shopify-auth-express-middleware
https://www.npmjs.com/package/@versollabs/shopify-auth-express-middleware

<!-- 
TODO: Talk about NarrowedShopifyObject type
- Dependency inversint principle
- Allows for in memory testing of the library
Type Narrowing ensures that our fake and the real shopify api object conform to the same specification.
 -->

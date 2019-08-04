<h1 align="center"></h1>

<div align="center">
  <a href="http://nestjs.com/" target="_blank">
    <img src="https://nestjs.com/img/logo_text.svg" width="150" alt="Nest Logo" />
  </a>
</div>

<h3 align="center">Decorators for Managing Cookies with NestJS</h3>

<div align="center">
   <img src="https://img.shields.io/badge/license-MIT-brightgreen.svg" alt="License" />
  <img src="https://badge.fury.io/js/%40nestjsplus%2Fcookies.svg" alt="npm version" height="18">
  <a href="https://nestjs.com" target="_blank">
    <img src="https://img.shields.io/badge/built%20with-NestJs-red.svg" alt="Built with NestJS">
  </a>
</div>

### Installation

```bash
npm install @nestjsplus/cookies
```

### Motivation
NestJS doesn't currently have decorators for getting and setting cookies.  While it's not
too hard to read cookies, it's convenient to have a decorator to do so.

```typescript
@Post('login')
login(@Cookies() cookies) {
  console.log('Got cookies:', cookies);
}
```

Setting cookies is a little less straightforward.  You either need to utilize the platform-specific
response (`res`) object, or write an interceptor. The former is pretty straightforward, though
takes a non-Nest-like imperative style.  The latter puts you into [manual response mode](https://docs.nestjs.com/controllers#routing),
meaning you can no longer rely on features like `@Render()`, `@HttpCode()` or [interceptors that modify the response](https://docs.nestjs.com/interceptors#response-mapping), and making testing harder (you'll have to mock the response
object, etc.).  The `@SetCookies()` decorator from this package wraps an interceptor
in a declarative decorator that solves these issues.

Collectively, the `@Cookies()`, `@SignedCookies()`, `@SetCookies()` and `@ClearCookies()` decorators in this package
provide a convenient set of features that make it easier to manage cookies in a standard and declarative way,
and minimize boilerplate code.

### Importing the Decorators
Import the decorators, just as you would other Nest decorators, in the controllers
that use them.  Use `import { ...decorators... } from @nestjsplus/cookies` as shown
below:

```typescript
import { Controller, Get } from '@nestjs/common';
import { AppService } from './app.service';
import { Cookies, SignedCookies } from '@nestjsplus/cookies';

@Controller()
export class AppController {
...
```

### Reading Cookies
As mentioned, reading cookies requires the `cookie-parser` package to be installed.
Reading **signed cookies** requires that the `CookieParser` be configured with a
[signing secret](https://github.com/expressjs/cookie-parser#cookieparsersecret-options) as
shown in the `bootstrap()` function above.

#### Regular (not signed) Cookies
Use the `@Cookies()` route parameter decorator to get "regular" cookies.
```typescript
@Get('get')
get(@Cookies() cookies): string {
  console.log('cookies: ', cookies);
  return this.appService.getHello();
}
```

This will bind an array of **all** (not signed) cookies to the `cookies` parameter.
See [below](#accessing-specific-named-cookies) to access a named cookie.

#### Signed Cookies
Use the `@SignedCookies()` route parameter decorator to get signed cookies.
```typescript
@Get('getSigned')
getSigned(@SignedCookies() cookies) {
  console.log('signed cookies: ', cookies);
}
```

As with `@Cookies()`, this will bind an array of **all** signed cookies to the `cookies`
parameter.  Access individual signed cookies [as described below](#accessing-specific-named-cookies).

#### Accessing Specific (Named) Cookies
Pass the name of a specific cookie in the `@Cookies()` or `@SignedCookies()` decorator
to access a specific cookie:

```typescript
get(@SignedCookies('cookie1') cookie1)
```

### Setting Cookies
Use the `@SetCookies()` route handler method decorator to set cookies.

Here's the API:
```typescript
@SetCookies(
  options?: CookieOptions,
  cookies?: CookieSettings | CookieSettings[]
)
```

Here's how it works:
- Set default [cookie options](#cookieoptions) by passing a
`CookieOptions` object. Options set on individual cookies,
if provided, override these defaults.
- For "static" cookies, where the cookie name and/or value are known at compile time,
you can set them in the `@SetCookies()` decorator by passing a [CookieSettings](#cookie-settings)
object.  For example:
```typescript
@SetCookies({name: 'cookie1', value: 'cookie 1 value'})
@Get('set')
set() {
  ...
}
```
You can mix and match `CookieOptions` and `CookieSettings` as needed:
```typescript
@SetCookies({httpOnly: true},
 [
   {name: 'cookie1', value: 'cookie 1 value'},
   {name: 'cookie2', value: 'cookie 2 value', {httpOnly: false}}
 ]
)
```
As a result of the above, `cookie1` will be set as `HttpOnly`, but `cookie2` will not.

- For "dynamic"cookies, where the cookie name and/or value are computed at run-time,
you can provide the cookie name/value pairs to be set when the
route handler method runs.  Provide these values by passing them on the `req._cookies`
array property.  Of course if you are using this method, you must access the `request` object,
so you must bind `@Request()` to a route parameter.  For example:
```typescript
set(<at>Request() req) {
  const cookie1Value = 'chocoloate chip';
  req._cookies = [
    {
      name: 'cookie1',
      value: cookie1Value,
      options: {
        signed: true,
        sameSite: true,
      },
    },
    { name: 'cookie2', value: 'oatmeal raisin' },
  ];
  ...
```
- The route handler otherwise proceeds as normal. It can return values, and it can
use other route handler method decorators (such as `@Render()`) and other route
parameter decorators (such as `@Headers()`, `@Query()`).

#### Cookie Settings
- As shown above, each cookie has the shape:
```typescript
interface CookieSettings {
  /**
   * name of the cookie.
   */
  name: string;
  /**
   * value of the cookie.
   */
  value?: string;
  /**
   * cookie options.
   */
  options?: CookieOptions;
}
```
- If `options` are provided for a cookie, they completely replace any options
specified in the `@SetCookies()` decorator.  If omitted for a cookie, they default
to options specified on the `@SetCookies()` decorator, or [Express's default cookie settings](https://expressjs.com/en/api.html#res.cookie)
if none were set.

#### CookieOptions
Cookie options may be set at the method level (`@SetCookies()`), providing a set of
defaults, or for individual cookies. In either case, they have the following shape:
```typescript
interface CookieOptions {
  /**
   * Domain name for the cookie.
   */
  domain?: string;
  /**
   * 	A synchronous function used for cookie value encoding. Defaults to encodeURIComponent.
   */
  encode?: (val: string) => string;
  /**
   * Expiry date of the cookie in GMT. If not specified or set to 0, creates a session cookie.
   */
  expires?: Date;
  /**
   * Flags the cookie to be accessible only by the web server.
   */
  httpOnly?: boolean;
  /**
   * Convenient option for setting the expiry time relative to the current time in milliseconds.
   */
  maxAge?: number;
  /**
   * Path for the cookie. Defaults to “/”.
   */
  path?: string;
  /**
   * Marks the cookie to be used with HTTPS only.
   */
  secure?: boolean;
  /**
   * Indicates if the cookie should be signed.
   */
  signed?: boolean;
  /**
   * Value of the “SameSite” Set-Cookie attribute. More information at
   * https://tools.ietf.org/html/draft-ietf-httpbis-cookie-same-site-00#section-4.1.1.
   */
  sameSite?: boolean | string;
}
```
#### Example
Setting cookies isn't hard!  See [a full example here in the test folder](https://github.com/nestjsplus/cookies/blob/master/test/src/app.controller.ts).

### Clearing (deleting) Cookies

Delete cookies in one of two ways:
1. Use `@SetCookies()` and pass in **only** the cookie name (leave the value property
off of the object).
2. Use `@ClearCookies()`, passing in a comma separated list of cookies to clear.
```typescript
@ClearCookies('cookie1', 'cookie2')
@Get('kill')
@Render('clear')
kill() {
  return { message: 'cookies killed!' };
}
```

### Restrictions
#### Express Only
These decorators currently only work with Nest applications running on `@platform-express`.  Fastify support is not
currently available.

#### Cookie Parser
Note that reading cookies depends on the standard Express [cookie-parser]() package.  Be sure to install it
and configure it in your app.  For example:

```bash
npm install cookie-parser
```
and in your `main.ts` file:
```typescript
import { NestFactory } from '@nestjs/core';
import { NestExpressApplication } from '@nestjs/platform-express';
import { AppModule } from './app.module';

import * as CookieParser from 'cookie-parser';

async function bootstrap() {
  const app = await NestFactory.create<NestExpressApplication>(AppModule);

  app.use(CookieParser('secret'));

  await app.listen(3000);
}
bootstrap();
```

#### Decorators Can't Access `this`
Note that decorators have access to the `class` (Controller), but not the instance.  This means that, for example,
if you want to pass a variable to a `SetCookies()` decorator, you should pass a variable set in the outer scope of
the file (e.g., a `const` above the controller class definition), as opposed to a property on the controller class.

See [the controller in the test folder](https://github.com/nestjsplus/cookies/blob/master/test/src/app.controller.ts) for an example.

## Change Log

See [Changelog](CHANGELOG.md) for more information.

## Contributing

Contributions welcome! See [Contributing](CONTRIBUTING.md).

## Author

- **John Biundo (Y Prospect on [Discord](https://discord.gg/G7Qnnhy))**

## License

Licensed under the MIT License - see the [LICENSE](LICENSE) file for details.
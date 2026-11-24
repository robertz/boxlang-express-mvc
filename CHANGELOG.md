# Changelog

**0.3.0**
- Flash: `event.getFlash()` and a `flash` variable in views, kept in the
  BoxExpress session.
- `this.prehandler_only` / `_except`, the same for `posthandler` and
  `aroundhandler`, and `aroundHandler( event, targetAction,
  eventArguments, rc, prc )`.
- Routes: `get()`, `post()`, `put()`, `patch()`, `delete()`, `group()`,
  `toView()`, `toRedirect()`, `apiResources()`, and `resources()` with a
  pattern and a handler for nesting.
- A POST with `_method=PUT`, `PATCH` or `DELETE` in its body is routed
  as that method. `_method` no longer appears in `rc`.
- Event: `paramValue()`, `getOnly()`, `getExcept()`, `populateModel()`,
  `isAjax()`, `getHTTPContent()`.
- `onMissingAction` and `onInvalidHTTPMethod` on a handler.
- Interceptors: an `interceptors` setting, `event.announce()` and
  `mvc.announce()`.
- **Changed:** `aroundHandler`, `onMissingAction` and
  `onInvalidHTTPMethod` are reserved names and can't be actions.
- `examples/todo` uses flash for its messages and
  `this.prehandler_only`; `examples/starter` uses `get()` and `toView()`.

**0.2.0**
- `this.allowedMethods` on a handler limits an action to the listed
  HTTP methods on every route, the convention route included. A wrong
  method gets a 405 with an `Allow` header.
- **Changed:** an action is now a public function whose first argument
  is named `event`. Other public functions are no longer routable.
- **Changed:** `relocate()` throws `BoxExpressMvc.InvalidRelocation`
  for `//host` and `/\host`, which a browser reads as another site.
- Fixed: `buildLink()` used a param name as a regular expression.
- `examples/todo` adds security headers, a rate limit and a cap on the
  list; its Dockerfile pins the base image by digest. `examples/starter`
  defaults to `production`.
- CI: read-only token, no stored checkout credentials, actions and the
  CommandBox image pinned; Dependabot keeps the pins current.
- `examples/todo`: a todo list with sessions, CSRF protection and a
  Dockerfile. `examples/starter` is the former `examples/` app.
- A GitHub Actions workflow runs the specs on every push.
- `examples/todo` installs `bx-esapi` and uses
  `encodeForHTMLAttribute()` in attributes. The docs say to install it
  in every app.

**0.1.0**
- First version: `boxExpressMvc()`, handlers with `( event, rc, prc )`
  actions, `preHandler` / `postHandler`, the route DSL (`route`,
  `withAction`, `resources`, named routes, the convention route), `.bxm`
  views and layouts, and handler wiring from a `services` struct or a
  WireBox injector.

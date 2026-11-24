# Changelog

**Unreleased**
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

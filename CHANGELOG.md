# Changelog

**Unreleased**
- `examples/todo`: a todo list with sessions, CSRF protection and a
  Dockerfile. `examples/starter` is the former `examples/` app.
- A GitHub Actions workflow runs the specs on every push.
- Docs: `encodeForHTML()` is the only encoder in BoxLang itself.

**0.1.0**
- First version: `boxExpressMvc()`, handlers with `( event, rc, prc )`
  actions, `preHandler` / `postHandler`, the route DSL (`route`,
  `withAction`, `resources`, named routes, the convention route), `.bxm`
  views and layouts, and handler wiring from a `services` struct or a
  WireBox injector.

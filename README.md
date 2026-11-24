# boxlang-express-mvc

ColdBox-style handlers, routes, views and layouts for
[BoxExpress](https://github.com/robertz/boxlang-express).

It is **familiar, not compatible**. A ColdBox developer will recognise
`function index( event, rc, prc )`, the `route().to()` DSL and
`views/` + `layouts/`. Existing ColdBox apps, ColdBox modules and CBWIRE
do not run on it. Coming from ColdBox? Read
[docs/coming-from-coldbox.md](docs/coming-from-coldbox.md) first.

Everything here compiles to plain BoxExpress routes when the app starts.
Nothing is scanned or looked up on disk per request.

Needs BoxExpress 0.2.27 or later and BoxLang 1.15 or later. The test
suite runs on BoxLang 1.15 through 1.19 and in the `boxlang:cli` Docker
image.

## Quick start

```
myapp/
  server.bxs
  boxlang_modules/        boxexpress, boxexpressmvc and bx-esapi
  config/Router.bx
  handlers/items.bx
  views/items/index.bxm
  layouts/Main.bxm
```

```js
// server.bxs — run with `boxlang server.bxs` from this directory
app = boxExpress()
app.use( boxExpressUrlencoded() )

mvc = boxExpressMvc( {
	handlers: expandPath( "./handlers" ),
	views: expandPath( "./views" ),
	layouts: expandPath( "./layouts" ),
	routes: expandPath( "./config/Router.bx" )
} )
app.use( mvc.router() )

app.use( ( err, req, res, next ) => res.status( 500 ).send( "Something went wrong." ) )
app.listen( 3000 )
```

```js
// config/Router.bx
class {
	function configure() {
		route( "/", "items.index" )
		route( "/items/:id(\d+)" ).as( "itemDetail" ).to( "items.show" )
	}
}
```

```js
// handlers/items.bx
class {
	function index( event, rc, prc ) {
		prc.items = [ { id: 1, name: "Anvil" } ]
		// renders views/items/index.bxm inside layouts/Main.bxm
	}

	function show( event, rc, prc ) {
		event.renderData( type = "json", data = { id: rc.id } )
	}
}
```

```html
<!-- views/items/index.bxm -->
<bx:output>
<bx:loop array="#prc.items#" item="item">
	<a href="#event.buildLink( "itemDetail", { id: item.id } )#">#encodeForHTML( item.name )#</a>
</bx:loop>
</bx:output>
```

```html
<!-- layouts/Main.bxm -->
<bx:output>
<html><body>#view()#</body></html>
</bx:output>
```

Two complete apps are in [examples/](examples/): `starter`, a rebuild of
the ColdBox starter template, and `todo`, a todo list with forms, a
service, a partial, sessions with CSRF protection, and a Dockerfile.

## Handlers are shared between requests

**One instance of each handler serves every request, and requests run
at the same time on separate threads.** Never keep anything about the
current request in a handler's `variables` scope:

```js
function show( event, rc, prc ) {
	variables.item = service.find( rc.id )   // WRONG: another request overwrites it
	prc.item = service.find( rc.id )         // right: prc belongs to this request
}
```

Per-request data goes in `rc`, `prc` or a `var`. Anything a handler
holds on to (a service, a cache) must be safe to call from many threads.
The route table and handler registry are built before `listen()` and
never change afterwards.

## Settings

| Key | |
|---|---|
| `handlers` | Required. Directory of handler classes. `handlers/admin/users.bx` is the handler `admin.users`. |
| `views` | Directory of `.bxm` views. Without it an action has to send its own response. |
| `layouts` | Directory of `.bxm` layouts. |
| `defaultLayout` | Layout name. Defaults to `Main` when `layouts/Main.bxm` exists, otherwise no layout. |
| `routes` | Path of a routes class (below). Routes can also be added in code with `mvc.route()` and `mvc.resources()`. |
| `services` | A struct. A handler property named after one of its keys is set to that value at startup. |
| `injector` | A WireBox injector. `injector.autowire()` is called on each handler at startup. |

`mvc.getHandlers()` returns each handler's name with its action names,
for a listing page or a startup log.

`mvc.router()` returns an ordinary BoxExpress router, so it mounts
anywhere (`app.use( "/admin", mvc.router() )`) and middleware goes in
front of it the usual way. Calling it ends route registration; a route
added later throws.

## Actions

An action is a public function on a handler whose first argument is
named `event`: `function name( event, rc, prc )`. Any other public
function is a helper and can't be routed to, though `private` says so
more clearly.

- **`rc`** is a new struct for each request holding `req.query`, then
  `req.body` (when a body parser produced a struct), then `req.params`.
  Later ones win, so a client can't override a route param. A key sent
  more than once is an array.
- **`prc`** is `res.locals`, so middleware can fill it before the action
  runs.
- **`event`** wraps the request and response:

| | |
|---|---|
| `getValue( name, defaultValue )`, `setValue( name, value )`, `valueExists( name )` | Read and write `rc`. A missing value with no default is `""`. |
| `getCollection()`, `getPrivateCollection()` | `rc` and `prc`. |
| `setView( view, layout )`, `setLayout( name )`, `noLayout()`, `noRender()` | Choose what is rendered. `noRender()` ends the response with no body. |
| `renderData( type, data, statusCode, contentType )` | Sends now. `type` is `json` (default), `html`, `text` or `xml`. JSON takes a plain struct or array; the others take a string. |
| `relocate( to, queryString, statusCode )` | Redirects. `to` is a path (starts with one `/`), a URL with a scheme, a route name or a `handler.action`. |
| `buildLink( to, params )` | The URL of a route name or `handler.action`. `params` fill `:placeholders`; the rest become the query string. |
| `setHTTPHeader( name, value )`, `setStatusCode( code )` | |
| `getHTTPMethod()`, `getHTTPHeader( name, defaultValue )` | |
| `getCurrentEvent()`, `getCurrentHandler()`, `getCurrentAction()` | |
| `getRequest()`, `getResponse()` | The BoxExpress `req` and `res`, for anything not covered here. |

After the action, if nothing has been sent, the view is rendered:
the one named with `setView()`, or `views/{handler}/{action}.bxm`
(`views/admin/users/list.bxm` for `admin.users.list`).

`renderData()` and `relocate()` send the response immediately. Headers
can't be changed after that, and a second response throws, as in
BoxExpress.

A handler may also declare `preHandler( event, rc, prc, action )` and
`postHandler( event, rc, prc, action )`. The order is `preHandler`,
action, `postHandler`, view, layout. **If `preHandler` sends a response
(a `relocate()` to the login page, say), the action and `postHandler`
are skipped.**

An error thrown anywhere in that sequence goes to the app's error
handler (`app.use( ( err, req, res, next ) => … )`).

### `this.allowedMethods`

```js
class {

	this.allowedMethods = { delete: "POST,DELETE", save: "POST" }

	function delete( event, rc, prc ) { … }

}
```

A request for a listed action with any other method gets a `405` and an
`Allow` header, whichever route it came in on. `HEAD` counts as `GET`.
An action that isn't listed answers every method. A name that isn't an
action, or a method that doesn't exist, throws at startup.

**List every action that changes data.** `route().to()` and the
convention route answer every method, and `boxExpressCsrf()` does not
check a `GET`, so without this a link or an image tag on another site
can run the action for a signed-in visitor.

### Redirecting to something a visitor sent

`relocate()` refuses `//host` and `/\host`, which look like paths but
send a browser to another site. It does follow a full URL, so a value
such as `rc.returnTo` still needs checking before it is passed in:
accept it only if it starts with a single `/`.

## Routes

```js
class {
	function configure() {
		route( "/items/:id" ).as( "itemDetail" ).to( "items.show" )
		route( "/login" ).withAction( { GET: "form", POST: "doLogin" } ).toHandler( "auth" )
		route( "/about", "pages.about" )
		resources( "photos" )
		route( "/health", ( req, res ) => res.send( "ok" ) )
		route( "/:handler/:action?" ).end()
	}
}
```

- `to( "handler.action" )` answers every HTTP method. Use `withAction()`
  to tie methods to actions; other methods then get a 404.
- `as( name )` names a route for `buildLink()` and `relocate()`.
- A function as the second argument is a plain BoxExpress handler.
- A route pointing at a handler or action that doesn't exist throws at
  startup, as does a `route()` never finished with `to()`,
  `toHandler()` or `end()`.

Paths are BoxExpress paths: `:name`, a trailing `:name?`, a trailing `*`
or `*name`, and constraints such as `:id(\d+)`. An optional or wildcard
segment anywhere but the end can't be matched and throws at startup,
naming the route. The first matching route wins, so put specific routes
above general ones.

### `resources( "photos" )`

| Method | Path | Action |
|---|---|---|
| GET | `/photos` | `index` |
| GET | `/photos/new` | `new` |
| POST | `/photos` | `create` |
| GET | `/photos/:id` | `show` |
| GET | `/photos/:id/edit` | `edit` |
| PUT, PATCH | `/photos/:id` | `update` |
| DELETE | `/photos/:id` | `delete` |

The handler needs all seven actions. HTML forms can only send GET and
POST, and there is no `_method` override; put a small middleware in
front if you need one:

```js
app.use( ( req, res, next ) => {
	var override = req.body._method ?: ""
	if ( req.method == "POST" && isSimpleValue( override ) && listFindNoCase( "PUT,PATCH,DELETE", override ) ) {
		req.method = uCase( override )
	}
	next()
} )
```

### The convention route

`route( "/:handler/:action?" ).end()` serves any action of any handler
at `/handler/action`, with `index` as the default action. It is opt-in
and always registered after every other route.

**It answers every HTTP method, so a GET can reach an action that
changes data.** List those actions in
[`this.allowedMethods`](#thisallowedmethods), or leave this route out.
Only actions are reachable; `init`, `preHandler`, `postHandler`, private
functions and public functions without an `event` argument answer 404,
as does an unknown handler or action. A nested
handler is reached with its dotted name: `/admin.users/list`.

## Views and layouts

Views and layouts are `.bxm` files. Each sees `rc`, `prc` and `event`.
`#…#` is only evaluated inside `<bx:output>`.

**Output is not escaped.** Wrap anything that came from a user in the
encoder for where it lands:

```html
<bx:output>
<h1>#encodeForHTML( prc.item.name )#</h1>
<input name="name" value="#encodeForHTMLAttribute( prc.item.name )#">
</bx:output>
```

Install [`bx-esapi`](https://forgebox.io/view/bx-esapi) in every app:

```bash
install-bx-module boxlang-express boxlang-express-mvc bx-esapi --local
```

Only `encodeForHTML()` is part of BoxLang itself.
`encodeForHTMLAttribute()`, `encodeForURL()`, `encodeForJavaScript()`
and the rest come from `bx-esapi`, and a view that calls one without
the module fails with "Function not found".

In a layout, `#view()#` is the rendered view. Anywhere,
`#view( "partials/nav", { active: "items" } )#` renders another view
inline; that view reads what it was passed as `args.active`.

The view runs before the layout, so a view can set `prc.title` for the
layout to use.

These views are separate from BoxExpress's own `res.render()`:
`app.set( "views" )`, `app.set( "view layout" )` and `partial()` don't
apply here, and `res.render()` still works in plain-function routes.

Edits to `.bxm` files show up without a restart. Handlers and the routes
file are loaded once; restart, or use BoxExpress's
`app.set( "reloadOnChange", true )`, which restarts the server when a
handler, model or the routes file changes.

## Services

Either way, handlers are wired once at startup.

```js
// A plain struct: properties named after a key are filled in.
svc = { itemService: new services.ItemService() }
mvc = boxExpressMvc( { handlers: …, services: svc } )

// handlers/items.bx
class {
	property name="itemService";
	function index( event, rc, prc ) { prc.items = itemService.list() }
}
```

```js
// WireBox (install it yourself: `box install wirebox`)
injector = new wirebox.system.ioc.Injector()
injector.getBinder().mapDirectory( packagePath = "models" )
mvc = boxExpressMvc( { handlers: …, injector: injector } )

// handlers/items.bx
class {
	property name="itemService" inject="ItemService";
}
```

Tested with WireBox 8.3, including in the `boxlang:cli` Docker image. It
adds about a second and a half to startup.

## Testing

Use BoxExpress's `app.inject()`; no port is needed:

```js
var res = app.inject( { method: "POST", url: "/items/new", form: { name: "Lamp" } } )
expect( res.statusCode ).toBe( 302 )
```

To run this module's own suite: `boxlang setup-tests.bxs` once (it
expects a BoxExpress checkout at `../boxlang-express`, or pass its
path), then `boxlang run-tests.bxs`.

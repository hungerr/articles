## Upgrade to React Router v6

### Upgrade all `Switch` elements to `Routes`

- `Route path` and `Link to` are relative
- Routes are chosen based on the best match instead of being traversed in order
- Routes may be nested in one place instead of being spread out in different components.

### Relative Routes and Links
```JAVASCRIPT
import {
  BrowserRouter,
  Routes,
  Route,
  Link
} from "react-router-dom";

function App() {
  return (
    <BrowserRouter>
      <Routes>
        <Route path="/" element={<Home />} />
        <Route path="users/*" element={<Users />} />
      </Routes>
    </BrowserRouter>
  );
}

function Users() {
  return (
    <div>
      <nav>
        <Link to="me">My Profile</Link>
      </nav>

      <Routes>
        <Route path=":id" element={<UserProfile />} />
        <Route path="me" element={<OwnUserProfile />} />
      </Routes>
    </div>
  );
}
```

- `Route path` and `Link to` are relative. This means that they automatically build on the parent route's path and URL so you don't have to manually interpolate `match.url` or `match.path`
- `Route exact` is gone. Instead, routes with descendant routes (defined in other components) use a trailing `*` in their path to indicate they match deeply
- You may put your routes in whatever order you wish and the router will automatically detect the best route for the current URL. This prevents bugs due to manually putting routes in the wrong order in a `Switch`
- You may have also noticed that all `Route children` from the v5 app changed to `Route element`

### Advantages of `Route element`
传递`props`
```javascript
// Ah, nice and simple API. And it's just like the <Suspense> API!
// Nothing more to learn here.
<Route path=":userId" element={<Profile />} />

// But wait, how do I pass custom props to the <Profile>
// element? Oh ya, it's just an element. Easy.
<Route path=":userId" element={<Profile animate={true} />} />

// Ok, but how do I access the router's data, like the URL params
// or the current location?
function Profile({ animate }) {
  let params = useParams();
  let location = useLocation();
}

// But what about components deep in the tree?
function DeepComponent() {
  // oh right, same as anywhere else
  let navigate = useNavigate();
}

// Aaaaaaaaand we're done here.
```

nesting routes:

```javascript
// This is a React Router v6 app
import {
  BrowserRouter,
  Routes,
  Route,
  Link,
  Outlet
} from "react-router-dom";

function App() {
  return (
    <BrowserRouter>
      <Routes>
        <Route path="/" element={<Home />} />
        <Route path="users" element={<Users />}>
          <Route path="me" element={<OwnUserProfile />} />
          <Route path=":id" element={<UserProfile />} />
        </Route>
      </Routes>
    </BrowserRouter>
  );
}

function Users() {
  return (
    <div>
      <nav>
        <Link to="me">My Profile</Link>
      </nav>

      <Outlet />
    </div>
  );
}
```
We didn't need a trailing `*` on `<Route path="users">` this time because when the routes are defined in one spot the router is able to see all your nested routes.

When using a nested config, routes with `children` should render an `<Outlet>` in order to render their child routes. This makes it easy to render layouts with nested UI.

### Note on `<Route path>` patterns

React Router v6 uses a simplified path format. `<Route path>` in v6 supports only 2 kinds of placeholders: dynamic `:id-style` params and `*` wildcards. A `*` wildcard may be used only at the `end` of a path, `not in the middle`.

All of the following are valid route paths in v6:
```javascript
/groups
/groups/admin
/users/:id
/users/:id/messages
/files/*
/files/:id/*
```
The following RegExp-style route paths are not valid in v6:

```javascript
/users/:id?
/tweets/:id(\d+)
/files/*/cat.jpg
/files-*
```

If you were using `<Route sensitive>` you should move it to its containing `<Routes caseSensitive>` prop. Either all routes in a `<Routes>` element are case-sensitive or they are not.

One other thing to notice is that all path matching in v6 `ignores` the `/` on the URL. In fact, `<Route strict>` has been removed and has no effect in v6. This does not mean that you can't use trailing slashes if you need to. Your app can decide to use trailing slashes or not, you just can't render two different UIs client-side at `<Route path="edit">` and `<Route path="edit/">`. You can still render two different UIs at those URLs (though we wouldn't recommend it), but you'll have to do it server-side.

### Note on <Link to> values

In v5, a `<Link to>` value that does not begin with `/` was ambiguous; it depends on what the current URL is. For example, if the current URL is `/users`, a v5 `<Link to="me">` would render a `<a href="/me">`. However, if the current URL has a `/`, like `/users/`, the same `<Link to="me">` would render `<a href="/users/me">`. This makes it difficult to predict how links will behave

React Router v6 fixes this ambiguity. In v6, a `<Link to="me">` will always render the same `<a href>`, regardless of the current URL.

For example, a `<Link to="me">` that is rendered inside a `<Route path="users">`will always render a link to `/users/me`, regardless of whether or not the current URL has a `/`.

相对路径:
```javascript
function App() {
  return (
    <Routes>
      <Route path="users" element={<Users />}>
        <Route path=":id" element={<UserProfile />} />
      </Route>
    </Routes>
  );
}

function Users() {
  return (
    <div>
      <h2>
        {/* This links to /users - the current route */}
        <Link to=".">Users</Link>
      </h2>

      <ul>
        {users.map(user => (
          <li>
            {/* This links to /users/:id - the child route */}
            <Link to={user.id}>{user.name}</Link>
          </li>
        ))}
      </ul>
    </div>
  );
}

function UserProfile() {
  return (
    <div>
      <h2>
        {/* This links to /users - the parent route */}
        <Link to="..">All Users</Link>
      </h2>

      <h2>
        {/* This links to /users/:id - the current route */}
        <Link to=".">User Profile</Link>
      </h2>

      <h2>
        {/* This links to /users/mj - a "sibling" route */}
        <Link to="../mj">MJ</Link>
      </h2>
    </div>
  );
}
```
make `..` operate on routes instead of URL segments, but it's a huge help when working with `*` routes
```JAVASCRIPT
function App() {
  return (
    <Routes>
      <Route path=":userId">
        <Route path="messages" element={<UserMessages />} />
        <Route
          path="files/*"
          element={
            // This links to /:userId/messages, no matter
            // how many segments were matched by the *
            <Link to="../messages" />
          }
        />
      </Route>
    </Routes>
  );
}
```

### Use `useRoutes` instead of `react-router-config`
```JAVASCRIPT
function App() {
  let element = useRoutes([
    // These are the same as the props you provide to <Route>
    { path: "/", element: <Home /> },
    { path: "dashboard", element: <Dashboard /> },
    {
      path: "invoices",
      element: <Invoices />,
      // Nested routes use a children property, which is also
      // the same as <Route>
      children: [
        { path: ":id", element: <Invoice /> },
        { path: "sent", element: <SentInvoices /> }
      ]
    },
    // Not found routes work as you'd expect
    { path: "*", element: <NotFound /> }
  ]);

  // The returned element will render the entire element
  // hierarchy with all the appropriate context it needs
  return element;
}
```

### Use useNavigate instead of useHistory
```JAVASCRIPT
// This is a React Router v5 app
import { useHistory } from "react-router-dom";

function App() {
  let history = useHistory();
  function handleClick() {
    history.push("/home");
  }
  return (
    <div>
      <button onClick={handleClick}>go home</button>
    </div>
  );
}

// This is a React Router v6 app
import { useNavigate } from "react-router-dom";

function App() {
  let navigate = useNavigate();
  function handleClick() {
    navigate("/home");
  }
  return (
    <div>
      <button onClick={handleClick}>go home</button>
    </div>
  );
}
```

If you need to `replace` the current location instead of `push` a new one onto the history stack, use `navigate(to, { replace: true })`. If you need state, use `navigate(to, { state })`. You can think of `the first argument` to navigate as your `<Link to>` and `the other arguments` as `the replace and state props`.

If you prefer to use a declarative API for navigation (ala v5's `Redirect `component), v6 provides a Navigate component. Use it like:
```JAVASCRIPT
import { Navigate } from "react-router-dom";

function App() {
  return <Navigate to="/home" replace state={state} />;
}
```
Note: Be aware that the v5 `<Redirect />` uses `replace` logic by default (you may change it via push prop), on the other hand, the v6 `<Navigate />` uses `push` logic by default and you may change it via `replace prop`.
```JAVASCRIPT
// Change this:
<Redirect to="about" />
<Redirect to="home" push />

// to this:
<Navigate to="about" replace />
<Navigate to="home" />
```
If you're currently using `go`, `goBack` or `goForward` from `useHistory` to navigate backwards and forwards, you should also replace these with navigate with a numerical argument indicating where to move the pointer in the history stack. For example, here is some code using v5's useHistory hook:
```JAVASCRIPT
// This is a React Router v5 app
import { useHistory } from "react-router-dom";

function App() {
  const { go, goBack, goForward } = useHistory();

  return (
    <>
      <button onClick={() => go(-2)}>
        Go 2 pages back
      </button>
      <button onClick={goBack}>Go back</button>
      <button onClick={goForward}>Go forward</button>
      <button onClick={() => go(2)}>
        Go 2 pages forward
      </button>
    </>
  );
}
```
Here is the equivalent app with v6:
```JAVASCRIPT
// This is a React Router v6 app
import { useNavigate } from "react-router-dom";

function App() {
  const navigate = useNavigate();

  return (
    <>
      <button onClick={() => navigate(-2)}>
        Go 2 pages back
      </button>
      <button onClick={() => navigate(-1)}>Go back</button>
      <button onClick={() => navigate(1)}>
        Go forward
      </button>
      <button onClick={() => navigate(2)}>
        Go 2 pages forward
      </button>
    </>
  );
}
```
Again, one of the main reasons we are moving from using the history API directly to the navigate API is to provide better compatibility with React suspense. React Router v6 uses the useTransition hook at the root of your component hierarchy. This lets us provide a smoother experience when user interaction needs to interrupt a pending route transition, for example when they click a link to another route while a previously-clicked link is still loading. The navigate API is aware of the internal pending transition state and will do a REPLACE instead of a PUSH onto the history stack, so the user doesn't end up with pages in their history that never actually loaded.

Note: The `<Redirect>` element from v5 is no longer supported as part of your route config (inside a `<Routes>`). This is due to upcoming changes in React that make it unsafe to alter the state of the router during the initial render. If you need to redirect immediately, you can either a) do it on your server (probably the best solution) or b) render a `<Navigate>` element in your route component. However, recognize that the navigation will happen in a useEffect.

Aside from suspense compatibility, `navigate`, like `Link`, supports `relative navigation`. For example:
```JAVASCRIPT
// assuming we are at `/stuff`
function SomeForm() {
  let navigate = useNavigate();
  return (
    <form
      onSubmit={async event => {
        let newRecord = await saveDataFromForm(
          event.target
        );
        // you can build up the URL yourself
        navigate(`/stuff/${newRecord.id}`);
        // or navigate relative, just like Link
        navigate(`${newRecord.id}`);
      }}
    >
      {/* ... */}
    </form>
  );
}
```

### Remove `<Link>` component prop

### Rename `<NavLink exact>` to <NavLink end>

### Remove `activeClassName` and `activeStyle` props from `<NavLink />`

```JAVASCRIPT
<NavLink
  to="/messages"
- style={{ color: 'blue' }}
- activeStyle={{ color: 'green' }}
+ style={({ isActive }) => ({ color: isActive ? 'green' : 'blue' })}
>
  Messages
</NavLink>
<NavLink
  to="/messages"
- className="nav-link"
- activeClassName="activated"
+ className={({ isActive }) => "nav-link" + (isActive ? " activated" : "")}
>
  Messages
</NavLink>
```

### Get StaticRouter from react-router-dom/server

```JAVASCRIPT
// change
import { StaticRouter } from "react-router-dom";
// to
import { StaticRouter } from "react-router-dom/server";
```
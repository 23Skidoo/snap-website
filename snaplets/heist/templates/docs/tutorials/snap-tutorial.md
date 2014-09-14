## Table of Contents

* [Quick Start](#quick-start)
* [Components of the Snap Framework](#components-of-the-snap-framework)
* [Hello, Snap!](#hello-snap)
* [Extended Example: Web Poll](#extended-example-web-poll)
    * [Interlude: A brief `Control.Lens` tutorial](#interlude-a-brief-control.lens-tutorial)
    * [Web poll example, continued](#web-poll-example-continued)

## Quick Start

If you haven't already done so, first go to the [download](/download) page to
find out how to install Snap.  The installation generates an executable called
`snap` that you can use to get started with a basic Snap project.  The following
instructions assume that the `snap` executable is on the system path.

To set up a new Snap project, run the following commands:

~~~~~~ {.shell}
$ mkdir myproject
$ cd myproject
$ snap init barebones
~~~~~~

This will generate a bare-bones project with the following directory structure:

    myproject/
      |-- myproject.cabal
      |-- src/
          |-- Main.hs

The file `src/Main.hs` contains the `Main` module for this project, and
`myproject.cabal` is the package description for Haskell's
[Cabal](http://haskell.org/cabal) build tool.  When you build this project with
`cabal build`, an executable is created in `dist/build/myproject/` called
`myproject`.  To build and run the example application, run the following
commands:

~~~~~~ {.shell}
$ cabal build
$ ./dist/build/myproject/myproject -p 8000
~~~~~~

Or, alternatively, just use `cabal run`, which has the same effect:

~~~~~~ {.shell}
$ cabal run -- -p 8000
~~~~~~

Now point your web browser to [http://localhost:8000/](http://localhost:8000/).
You should see the following response:

![](/media/img/tutorials/hello-world.png)

You can use `snap init -h` to see help about other project templates that are
available.

## Components of the Snap Framework

Before we dive into writing our first Snap web application, let's do a quick
overview of the parts of the Snap framework. Currently Snap is divided into the
following components:

  - [`snap-core`](http://hackage.haskell.org/package/snap-core) is the core of
  Snap.  It defines an API for interfacing with web servers and includes type
  definitions and all code that is server-agnostic.  This API is on the same
  level of abstraction as Java Servlets and is the focus of this tutorial.

  - [`snap-server`](http://hackage.haskell.org/package/snap-server) is an HTTP
  server library that supports the interface defined in `snap-core`.

  - [`heist`](http://hackage.haskell.org/package/heist) is the HTML templating
  library. You do not need it to use the above two libraries but you are
  certainly welcome to.

  - [`snap`](http://hackage.haskell.org/package/snap) is a library that builds
  on the above three prackages and provides higher-level abstractions for
  building complex websites.

  - [`snap-templates`](http://hackage.haskell.org/package/snap-templates) -
  contains the `snap` executable which can generate several different skeleton
  projects to get you started.

## Hello, Snap!

We'll now look at what's inside the barebones application created by `snap init`
and describe basic use of Snap's web server API.  If you understand servlets and
Haskell, most of this section should be very self-explanatory. Even if you
don't, have no fear!  This tutorial only expects that you know a little bit of
Haskell.

`snap init barebones` creates a single file in the `src` directory, Main.hs.
Here's the important code in Main.hs:

~~~~~~ {.haskell}
main :: IO ()
main = quickHttpServe site

site :: Snap ()
site =
    ifTop (writeBS "hello world") <|>
    route [ ("foo", writeBS "bar")
          , ("echo/:echoparam", echoHandler)
          ] <|>
    dir "static" (serveDirectory ".")

echoHandler :: Snap ()
echoHandler = do
    param <- getParam "echoparam"
    maybe (writeBS "must specify echo/param in URL")
          writeBS param
~~~~~~

The behavior of this code can be summarized with the following rules:

1. If the user requested the site's root page (`http://mysite.com/`), then
return a page containing the string `"hello world"`.
2. If the user requested `/foo`, then return `"bar"`.
3. If the user requested `/echo/xyz`, then return `"xyz"`.
4. If the request URL begins with `/static/`, then look for files on disk
matching the rest of the path and serve them.
5. If none of these match, then Snap returns a 404 page.

Let's go over each of the Snap API functions used here.

### [`dir`](http://hackage.haskell.org/package/snap-core/docs/Snap-Core.html#v:dir)

`dir` runs its action only if the request path starts with the specified
directory.  You can combine successive `dir` calls to match more than one
subdirectory into the path.

### [`ifTop`](http://hackage.haskell.org/package/snap-core/docs/Snap-Core.html#v:ifTop)

`ifTop` only executes its argument if the client requested the root URL.  Use
this for your home page.  It can also be combined with the `dir` function to
define your index handler for a certain URL directory.

### [`writeBS`](http://hackage.haskell.org/package/snap-core/docs/Snap-Core.html#v:writeBS)

`writeBS` appends a strict ByteString to the response being constructed.  Snap
also provides an analogous function `writeLBS` for lazy ByteStrings.  You can
also use the functions `writeText` and `writeLazyText` if you use Data.Text
instead of ByteString.

### [`<|>`](http://hackage.haskell.org/package/base/docs/Control-Applicative.html#v:-60--124--62-)

If you're not familiar with Haskell, you may be wondering about the `<|>`.  It
is simply a binary operator that evaluates its first argument, and if it
failed, evaluates the second.  If the first argument succeeded, then it stops
without evaluating the second argument.

The `site` function uses `<|>` to connect three different functions that guard
what page gets rendered.  First, the `ifTop` function runs.  This function
succeeds if the requested URL is `http://site.com/`.  If that happens, then
Snap sends a response of "hello world".  Otherwise the `route` function is
executed.

You could build your whole site using `<|>` to connect different handlers
together, but this kind of routing is expensive because the amount of time it
takes to construct a request scales linearly with the number of handlers in
your site.  For large sites, this could be quite noticeable.  To remedy this,
Snap also provides you with the `route` function that routes requests based on the
request path in `O(log n)` time:

### [`route`](http://hackage.haskell.org/package/snap-core/docs/Snap-Core.html#v:route)

`route` takes a list of (route, handler) tuples and succeeds returning the
result of the associated handler of the route that matches.  If no route
matches, then `route` fails and execution is passed on to `serveDirectory`.

In a real application, you will want to use `route` for almost everything.  We
didn't do it that way in this example because we wanted to demonstrate more of
the API.  For example, instead of

~~~~~~ {.haskell}
... <|>
dir "static" (serveDirectory ".")
~~~~~~

we could have used

~~~~~~ {.haskell}
route [ ("static", serveDirectory "."),
        ...
      ]
~~~~~~

### [`getParam`](http://hackage.haskell.org/package/snap-core/docs/Snap-Core.html#v:getParam)

`getParam` retrieves a GET or POST parameter from the request.  In this
example, the `route` function binds a captured portion of the URL to the
parameter `echoParam` so the associated handler can make easy use of it.
`echoHandler` checks to see whether a parameter was passed and returns the
value or an error message if it didn't exist.


## Extended Example: Web Poll

Printing "Hello, world!" and echoing output is fun, but usually web applications
are more complicated. We will showcase more advanced features of Snap such as
templates, snaplets and dynamic recompilation using a web poll application as an
example. To begin, run the following commands:

~~~~~~ {.shell}
$ mkdir myproject
$ cd myproject
$ snap init simple-poll-1
~~~~~~

This will generate an example application with the following directory structure:

    myproject/
      |-- simple-poll.cabal
      |-- src/
          |-- Application.hs
          |-- Main.hs
          |-- Site.hs

Now type `cabal run` and visit `localhost:8000` in your browser. You should see
a voting form:

![](/media/img/tutorials/simple-poll.png)

Choose any alternative you want and press "Submit". The result will look like
this:

![](/media/img/tutorials/simple-poll-result.png)

Let's now look at the code. The file `Main.hs` contains the `main` function and
some boilerplate code to make dynamic recompilation work (more on that
[later](#dynamic-recompilation)). The code can look a bit intimidating at first,
but you can think of it as equivalent to this line from the previous example:

~~~~~~ {.haskell}
main :: IO ()
main = quickHttpServe site
~~~~~~

You won't actually need to touch the code in `Main.hs`, so we'll skip delving
deeper into it for the purposes of this tutorial.  Later, you're welcome to take
a look yourself - `Main.hs` is amply commented.

Next, we have `Application.hs`, which looks like this (ignoring the imports):

~~~~~~ {.haskell}
type MapRef = IORef (M.Map B.ByteString Int)

data App = App
    { _mapRef :: MapRef
    }

makeLenses ''App

type AppHandler a = Handler App App a
~~~~~~

The main difference between this simple poll application and the previous
example is that the poll application is structured as a *snaplet*. Snaplets are
composable web applications - we will learn how to actually compose them
[later](#snaplets-in-more-detail), when we'll examine the snaplet concept in
more detail.  For the time being you can think of a snaplet as some kind of
state -- a counter, a database connection pool, or a collection of other
snaplets -- plus a bunch of functions for initializing and modifying this state.
The latter often involves reacting to HTTP requests, so snaplets can include
route mappings and request handlers.

In this example, the snaplet state is represented by the `App` record.  It
contains a single key-value container (of type `Map ByteString Int`) that stores
the number of votes (represented as an integer) corresponding to each poll
choice (represented as a `ByteString`).  We have to wrap this container inside a
mutable reference (`IORef`) because we want the map to be shared among different
requests.  Remember that because Haskell is a functional language, all data is
by default immutable.  The Snap web server processes each request in its own
green thread, which means that each request receives a separate copy of the
state (in this case, the `App` record).  Modifications to that state only affect
the local thread that generates a single response - since the state (the `App`
record) is immutable, to modify it is to replace it with a new copy, which does
not affect the requests executing in other threads.  So we have to use a mutable
reference (`IORef` or `MVar`) to implement global application state shared among
multiple threads.

Finally, `AppHandler` is the type of the application's request handlers. It
provides access to `MonadSnap` functionality (meaning that we can use functions
like `writeBS` that we saw earlier) as well as access to the application state
(`AppHandler` is an instance of the `MonadState` class). We will soon see
examples of its usage.  You may also wonder why the `App` parameter is repeated
twice - this will be explained in a [later section](#snaplets-in-more-detail),
when we'll look at the snaplet concept in more detail.

You probably noticed that the field names inside the `App` record begin with an
underscore. This is because we use
[lenses](http://hackage.haskell.org/package/lens) to access the inner fields of
the application state record.  Lenses make it much easier to work with nested
data structures -- standard Haskell records are a bit of a pain in this regard.
Remember that snaplets are meant to be composable, which means that in large
Snap applications built of many snaplets there will be a lot of nesting.  So
lenses come very handy; let's look at them in more detail.

### Interlude: A brief `Control.Lens` tutorial

A lens, notated  as follows:

~~~~~~ {.haskell}
SimpleLens a b
~~~~~~

is conceptually a "getter" and a "setter" rolled up into one.  The `lens` library
provides the following functions:

~~~~~~ {.haskell}
view :: (SimpleLens a b) -> a -> b
set  :: (SimpleLens a b) -> b -> a -> a
over :: (SimpleLens a b) -> (b -> b) -> a -> a
~~~~~~

which allow you to get, set, and modify a value of type `b` within the context
of type `a`.  The `lens` package comes with a Template Haskell function called
`makeLenses`, which auto-magically defines a lens for every record field having
a name beginning with an underscore.  In the `App` example above, `makeLenses`
defined the following lens:


~~~~~~ {.haskell}
mapRef :: SimpleLens App MapRef
~~~~~~

The coolest thing about `lens` lenses is that they *compose* using the `(.)`
operator.  If the `Foo` type had a field of type `Quux` within it with a lens
`quux :: SimpleLens Foo Quux`, then you could create a lens of type `SimpleLens
App Quux` by composition:

~~~~~~ {.haskell}
data Foo = Foo { _quux :: Quux }
makeLenses ''Foo

appQuuxLens :: SimpleLens App Quux
appQuuxLens = foo . quux
~~~~~~

Lens composition is very similar to function composition except it works in the
opposite direction (think Java-style `System.out.println` ordering) and it gives
you a composed getter and setter at the same time.

### Web Poll example, continued

Now let's turn to `Site.hs`, which contains the bulk of the application's
code. The only symbol exported from this module is `app`, the snaplet
initializer, which the code in `Main.hs` uses to start the application.  It
looks like this:

~~~~~~ {.haskell}
app :: SnapletInit App App
app = makeSnaplet "poll" "A simple poll application." Nothing $ do
  r <- liftIO (newIORef M.empty)
  addRoutes routes
  wrapSite (ifTop poll <|>)
  return $ App r

routes :: [(ByteString, AppHandler ())]
routes = [ ("",        error404)
         , ("results", method POST postResults)
         , ("results", method GET  showResults)
         ]
~~~~~~

### [`makeSnaplet`](http://hackage.haskell.org/package/snap-core/docs/Snap-Core.html#v:method)

TODO

### [`addRoutes`](http://hackage.haskell.org/package/snap-core/docs/Snap-Core.html#v:method)

TODO

### [`method`](http://hackage.haskell.org/package/snap-core/docs/Snap-Core.html#v:method)

TODO

The `error404` handler is pretty simple. It just sets the response status code
to 404 and appends an error message to the response. The
[`modifyResponse`](http://hackage.haskell.org/package/snap-core/docs/Snap-Core.html#v:modifyResponse)
function takes a function

~~~~~~ {.haskell}
error404 :: MonadSnap m => m ()
error404 = do modifyResponse $ setResponseStatus 404 "File Not Found"
              writeBS "File Not Found"
~~~~~~

Likewise, the `poll` handler just writes out the HTML form with poll questions using
the `writeBS` function which we've already seen before:

~~~~~~ {.haskell}
poll :: AppHandler ()
poll = writeBS $ B.concat [
  "<!DOCTYPE html>",
  ...
  ]
~~~~~~

postResults TODO

~~~~~~ {.haskell}
pollChoices :: [ByteString]
pollChoices = ["Real World Haskell"
              ,"Learn You a Haskell"
              ,"Beginning Haskell"
              ,"Programming Haskell"]

postResults :: AppHandler ()
postResults = do
  r   <- gets (view mapRef)
  par <- getParam "answer"
  let isValidKey k = k `elem` pollChoices
      key          = fromMaybe "???" $ mfilter isValidKey par
  liftIO $ atomicModifyIORef r (M.insertWith (+) key 1)
  showResults
~~~~~~

showResults TODO

~~~~~~ {.haskell}
packInt :: Int -> ByteString
packInt = B8.pack . show

showResults :: AppHandler ()
showResults = do
  r            <- gets (view mapRef)
  m            <- liftIO $ readIORef r
  let getVal  k = packInt $ M.findWithDefault 0 k m
  writeBS $ B.concat [
    "<!DOCTYPE html>",
    ...
    "<table>",
    "<tr><td>Real World Haskell</td><td>",  getVal "Real World Haskell",
    "</td></tr>",
    "<tr><td>Learn You a Haskell</td><td>", getVal "Learn You a Haskell",
    "</td></tr>",
    "<tr><td>Beginning Haskell</td><td>",   getVal "Beginning Haskell",
    "</td></tr>",
    "<tr><td>Programming Haskell</td><td>", getVal "Programming Haskell",
    ...
    ]

~~~~~~

## Dynamic recompilation

To activate dynamic recompilation in your project, rebuild your application with
"cabal clean; cabal configure -fdevelopment; cabal build".  This won't work with
the barebones project that we created above.  You have to create your project
with `snap init` instead.

TODO

## Replacing inline HTML with Heist templates

TODO

## Snaplets in more detail

Old:

## Snaplets

A snaplet is a composable web application.  Snaplets allow you to build
self-contained pieces of functionality and glue them together to make larger
applications.  Here are some of the things provided by the snaplet API:

  - Infrastructure for application state/environment

  - Snaplet initialization, reload, and cleanup

  - Management of filesystem data and automatic snaplet installation

  - Unified config file infrastructure

One example might be a wiki snaplet.  It would be distributed as a haskell
package that would be installed with `cabal` and would probably include code,
config files, HTML templates, stylesheets, JavaScript, images, etc.  The
snaplet's code would provide the necessary API to let your application
interact seamlessly with the wiki functionality.  When you run your
application for the first time, all of the wiki snaplet's filesystem resources
will automatically be copied into the appropriate places.  Then you will
immediately be able to customize the wiki to fit your needs by editing config
files, providing your own stylesheets, etc.  We will discuss this in more
detail later.

A snaplet can represent anything from backend Haskell infrastructure with no
user facing functionality to a small widget like a chat box that goes in the
corner of a web page to an entire standalone website like a blog or forum.
The possibilities are endless.  A snaplet is a web application, and web
applications are snaplets.  This means that using snaplets and writing
snaplets are almost the same thing, and it's trivial to drop a whole website
into another one.

We're really excited about the possibilities available with snaplets.  In
fact, Snap already ships with snaplets for sessions, authentication, and
templating (with Heist),  This gives you useful functionality out of the box,
and jump starts your own snaplet development by demonstrating some useful
design patterns.  So without further ado, let's get started.

## Snaplet Overview

The heart of the snaplets infrastructure is state management.  Most nontrivial
pieces of a web app need some kind of state or environment data.  Components
that do not need any kind of state or environment are probably more
appropriate as a standalone library than as a snaplet.

Before we continue, we must clarify an important point.  The Snap web server
processes each request in its own green thread.  This means that each request
will receive a separate copy of the state defined by your application and
snaplets, and modifications to that state only affect the local thread that
generates a single response.  From now on, when we talk about state this is
what we are talking about.  If you need global application state, you have to
use a thread-safe construct such as an MVar or IORef.

This post is written in literate Haskell.  It uses a small external module
called Part2 that is [available
here](https://github.com/snapframework/snap/blob/master/project_template/tutorial/src/Part2.lhs).
You can also install the full code in the current directory with the command
`snap init tutorial`.  First we need to get imports out of the way.

~~~~~~ {.haskell}
{-# LANGUAGE TemplateHaskell #-}
{-# LANGUAGE OverloadedStrings #-}

module Main where

import           Control.Lens.TH
import           Data.IORef
import qualified Data.ByteString.Char8 as B
import           Data.Maybe
import           Snap
import           Snap.Snaplet.Heist
import           Part2
~~~~~~

We start our application by defining a data structure to hold the state.  This
data structure includes the state of all snaplets (wrapped in a Snaplet) used
by our application as well as any other state we might want.

~~~~~~ {.haskell}
data App = App
    { _heist       :: Snaplet (Heist App)
    , _foo         :: Snaplet Foo
    , _bar         :: Snaplet Bar
    , _companyName :: IORef B.ByteString
    }

makeLenses ''App
~~~~~~

The field names begin with an underscore because of some more complicated
things going on under the hood.  However, all you need to know right now is
that you should prefix things with an underscore and then call `makeLenses`.
This lets you use the names without an underscore in the rest of your
application.

The next thing we need to do is define an initializer.

~~~~~~ {.haskell}
appInit :: SnapletInit App App
appInit = makeSnaplet "myapp" "My example application" Nothing $ do
    hs <- nestSnaplet "heist" heist $ heistInit "templates"
    fs <- nestSnaplet "foo" foo $ fooInit
    bs <- nestSnaplet "" bar $ nameSnaplet "newname" $ barInit foo
    addRoutes [ ("/hello", writeText "hello world")
              , ("/fooname", with foo namePage)
              , ("/barname", with bar namePage)
              , ("/company", companyHandler)
              ]
    wrapSite (<|> heistServe)
    ref <- liftIO $ newIORef "fooCorp"
    return $ App hs fs bs ref
~~~~~~

For now don't worry about all the details of this code.  We'll work through the
individual pieces one at a time.  The basic idea here is that to initialize an
application, we first initialize each of the snaplets, add some routes, run a
function wrapping all the routes, and return the resulting state data
structure.  This example demonstrates the use of a few of the most common
snaplet functions.

# nestSnaplet

All calls to child snaplet initializer functions must be wrapped in a call to
nestSnaplet.  The first parameter is a URL path segment that is used to prefix
all routes defined by the snaplet.  This lets you ensure that there will be no
problems with duplicate routes defined in different snaplets.  If the foo
snaplet defines a route `/foopage`, then in the above example, that page will
be available at `/foo/foopage`.  Sometimes though, you might want a snaplet's
routes to be available at the top level.  To do that, just pass an empty string
to nestSnaplet as shown above with the bar snaplet.

In our example above, the bar snaplet does something that needs to know about
the foo snaplet.  Maybe foo is a database snaplet and bar wants to store or
read something.  In order to make that happen, it needs to have a "handle" to
the snaplet.  Our handles are whatever field names we used in the App data
structure minus the initial underscore character.  They are automatically
generated by the `makeLenses` function.  For now it's sufficient to think of
them as a getter and a setter combined (to use an OO metaphor).

The second parameter to nestSnaplet is the lens to the snaplet you're nesting.
In order to place a piece into the puzzle, you need to know where it goes.

# nameSnaplet

The author of a snaplet defines a default name for the snaplet in the first
argument to the makeSnaplet function.  This name is used for the snaplet's
directory in the filesystem.  If you don't want to use the default name, you
can override it with the `nameSnaplet` function.  Also, if you want to have two
instances of the same snaplet, then you will need to use `nameSnaplet` to give
at least one of them a unique name.

# addRoutes

The `addRoutes` function is how an application (or snaplet) defines its
routes.  Under the hood the snaplet infrastructure merges all the routes from
all snaplets, prepends prefixes from `nestSnaplet` calls, and passes the list
to Snap's
[route](http://hackage.haskell.org/package/snap-core/docs/Snap-Core.html#v:route)
function.

A route is a tuple of a URL and a handler function that will be called when
the URL is requested.  Handler is a wrapper around the Snap monad that handles
the snaplet's infrastructure.  During initialization, snaplets use the
`Initializer` monad.  During runtime, they use the `Handler` monad.  We'll
discuss `Handler` in more detail later.  If you're familiar with Snap's old
extension system, you can think of it as roughly equivalent to the Application
monad.  It has a `MonadState` instance that lets you access and modify the
current snaplet's state, and a `MonadSnap` instance providing the
request-processing functions defined in Snap.Types.

# wrapSite

`wrapSite` allows you to apply an arbitrary `Handler` transformation to
the top-level handler.  This is useful if you want to do some generic
processing at the beginning or end of every request.  For instance, a session
snaplet might use it to touch a session activity token before routing happens.
It could also be used to implement custom logging.  The example above uses it
to define heistServe (provided by the Heist snaplet) as the default handler to
be tried if no other handler matched.  This may seem like an easy way to define
routes, but if you string them all together in this way each handler will be
evaluated sequentially and you'll get O(n) time complexity, whereas routes
defined with `addRoutes` have O(log n) time complexity.  Therefore, in a
real-world application you would probably want to have `("", heistServe)` in
the list passed to `addRoutes`.

# with

The last unfamiliar function in the example is `with`.  Here it accompanies a
call to the function `namePage`.  `namePage` is a simple example handler and
looks like this.

~~~~~~ {.haskell}
namePage :: Handler b v ()
namePage = do
    mname <- getSnapletName
    writeText $ fromMaybe "This shouldn't happen" mname
~~~~~~

This function is a generic handler that gets the name of the current snaplet
and writes it into the response with the `writeText` function defined by the
snap-core project.  The type variables 'b' and 'v' indicate that this function
will work in any snaplet with any base application.  The 'with' function is
used to run `namePage` in the context of the snaplets foo and bar for the
corresponding routes.

# Site Reloading

Snaplet Initializers serve dual purpose as both initializers and reloaders.
Reloads are triggered by a special handler that is bound to the
`/admin/reload` route.  This handler re-runs the site initializer and if it is
successful, loads the newly generated in-memory state.  To prevent denial of
service attacks, the reload route is only accessible from localhost.

If there are any errors during reload, you would naturally want to see them in
the HTTP response returned by the server.  However, when these same
initializers are run when you first start your app, you will want to see
status messages printed to the console.  To make this possible we provide the
`printInfo` function.  You should use it to output any informational messages
generated by your initializers.  If you print directly to standard output or
standard error, then those messages will not be available in your browser when
you reload the site.

# Working with state

`Handler b v` has a `MonadState v` instance.  This means that you can access all
your snaplet state through the get, put, gets, and modify functions that are
probably familiar from the
[`State` monad](https://hackage.haskell.org/package/mtl/docs/Control-Monad-State-Lazy.html#t:State).
In our example application we demonstrate this with `companyHandler`.

~~~~~~ {.haskell}
companyHandler :: Handler App App ()
companyHandler = method GET getter <|> method POST setter
  where
    getter = do
        nameRef <- gets _companyName
        name <- liftIO $ readIORef nameRef
        writeBS name
    setter = do
        mname <- getParam "name"
        nameRef <- gets _companyName
        liftIO $ maybe (return ()) (writeIORef nameRef) mname
        getter
~~~~~~

If you set a GET request to `/company`, you'll get the string "fooCorp" back.
If you send a POST request, it will set the IORef held in the `_companyName`
field in the `App` data structure to the value of the `name` field.  Then it
calls the getter to return that value back to you so you can see it was
actually changed.  Again, remember that this change only persists across
requests because we used an IORef.  If `_companyName` was just a plain string
and we had used modify, the changed result would only be visible in the rest
of the processing for that request.

## The Heist Snaplet

The astute reader might ask why there is no `with heist` in front of the call
to `heistServe`.  And indeed, that would normally be the case.  But we decided
that an application will never need more than one instance of a Heist snaplet.
So we provided a type class called `HasHeist` that allows an application to
define the global reference to its Heist snaplet by writing a `HasHeist`
instance.  In this example we define the instance as follows:

~~~~~~ {.haskell}
instance HasHeist App where heistLens = subSnaplet heist
~~~~~~

Now all we need is a simple main function to serve our application.

~~~~~~ {.haskell}
main :: IO ()
main = serveSnaplet defaultConfig appInit
~~~~~~

This completes a full working application.  We did leave out a little dummy
code for the Foo and Bar snaplets.  This code is included in Part2.hs.  For
more information look in our [API
documentation](http://hackage.haskell.org/package/snap), specifically the
Snap.Snaplet module.  No really, that wasn't a joke.  The API docs are written
as prose.  They should be very easy to read, while having the benefit of
including all the actual type signatures.

## Filesystem Data and Automatic Installation

Some snaplets will have data stored in the filesystem that should be installed
into the directory of any project that uses it.  Here's an example of what a
snaplet filesystem layout might look like:

    foosnaplet/
      |-- *devel.cfg*
      |-- db.cfg
      |-- public/
          |-- stylesheets/
          |-- images/
          |-- js/
      |-- *snaplets/*
          |-- *heist/*
              |-- templates/
          |-- subsnaplet1/
          |-- subsnaplet2/

Only the starred items are actually enforced by current code, but we want to
establish the others as a convention.  The file devel.cfg is automatically
read by the snaplet infrastructure.  It is available to you via the
`getSnapletUserConfig` function.  Config files use the format defined by Bryan
O'Sullivan's excellent [configurator
package](http://hackage.haskell.org/package/configurator).  In this example,
the user has chosen to put db config items in a separate file and use
configurator's import functionality to include it in devel.cfg.  If
foosnaplet uses `nestSnaplet` or `embedSnaplet` to include any other snaplets,
then filesystem data defined by those snaplets will be included in
subdirectories under the `snaplets/` directory.

So how do you tell the snaplet infrastructure that your snaplet has filesystem
data that should be installed?  Look at the definition of appInit above.  The
third argument to the makeSnaplet function is where we specify the filesystem
directory that should be installed.  That argument has the type `Maybe (IO
FilePath)`.  In this case we used `Nothing` because our simple example doesn't
have any filesystem data.  As an example, let's say you are creating a snaplet
called killerapp that will be distributed as a hackage project called
snaplet-killerapp.  Your project directory structure will look something like
this:

    snaplet-killerapp/
      |-- resources/
      |-- snaplet-killerapp.cabal
      |-- src/

All of the files and directories listed above under foosnaplet/ will be in
resources/.  Somewhere in the code you will define an initializer for the
snaplet that will look like this:

~~~~~~ {.haskell}
killerInit = makeSnaplet "killerapp" "42" (Just dataDir) $ do
~~~~~~

The primary function of Cabal is to install code.  But it has the ability to
install data files and provides a function called `getDataDir` for retrieving
the location of these files.  Since it returns a different result depending on
what machine you're using, the third argument to `makeSnaplet` has to be `Maybe
(IO FilePath)` instead of the more natural pure version.  To make things more
organized, we use the convention of putting all your snaplet's data files in a
subdirectory called resources.  So we need to create a small function that
appends `/resources` to the result of `getDataDir`.

~~~~~~ {.haskell}
import Paths_snaplet_killerapp
dataDir = liftM (++"/resources") getDataDir
~~~~~~

If our project is named snaplet-killerapp, the `getDataDir` function is
defined in the module Paths_snaplet_killerapp, which we have to import.  To
make everything work, you have to tell Cabal about your data files by
including a section like the following in snaplet-killerapp.cabal:

    data-files:
      resources/devel.cfg,
      resources/public/stylesheets/style.css,
      resources/snaplets/heist/templates/page.tpl

Now whenever your snaplet is used, its filesystem data will be automagically
copied into the local project that is using it, whenever the application is
run and it sees that the snaplet's directory does not already exist.  If the
user upgrades to a new version of the snaplet and the new version made changes
to the filesystem resources, those resources will NOT be automatically copied
in by default.  Resource installation *only* happens when the `snaplets/foo`
directory does not exist.  If you want to get the latest version of the
filesystem resources, remove the `snaplets/foo` directory, and restart your
app.

## What Now?

We hope we've whetted your appetite for using Snap. From here on out you should
take a look at the [API
documentation](http://hackage.haskell.org/package/snap-core),
which explains many of the concepts and functions here in further detail.

You can also come hang out in
[`#snapframework`](http://webchat.freenode.net/?channels=snapframework&uio=d4)
on [`freenode`](http://freenode.net/) IRC.

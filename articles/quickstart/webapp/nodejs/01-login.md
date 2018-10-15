---
title: Login
description: This tutorial demonstrates how to add user login, logout, and profile to a Node.js Express application.
budicon: 448
topics:
  - quickstarts
  - webapp
  - login
  - user profile
  - logout
  - nodejs
contentType: tutorial
useCase: quickstart
github:
    path: 01-Login
---
<%= include('../_includes/_getting_started', { library: 'Node.js', callback: 'http://localhost:3000/callback' }) %>

## Configure Node.js to use Auth0

### Install the dependencies

To get started, install the following dependencies.

* [passport](http://www.passportjs.org/) - an authentication middleware for Node.js
* [passport-auth0](https://github.com/auth0/passport-auth0) - an Auth0 authentication strategy for Passport
* [express-session](https://www.npmjs.com/package/express-session) - a middleware to manage sessions

```bash
# installation with npm
npm install passport passport-auth0 express-session --save
```

### Configure express-session

In `app.js`, include the `express-session` module and configure it. The `secret` parameter is a secret string that is used to sign the session ID cookie. Please use a custom value.

```js
// app.js

var session = require('express-session');

//session-related stuff
var sess = {
 secret: 'CHANGE THIS SECRET',
 cookie: {},
 resave: false,
 saveUninitialized: true
};

if (app.get('env') === 'production') {
 sess.cookie.secure = true; // serve secure cookies, requires https
}

app.use(session(sess));
```

### Configure Passport with the application settings

In `app.js`, include the `passport` and `passport0-auth0` modules, and configure Passport to use a new instance of `Auth0Strategy` with your Auth0 application settings. Use `passport.initialize()` and `passport.session()` to initialize Passport with persistent login sessions.

```js
// app.js
var Auth0Strategy = require('passport-auth0'),
    passport = require('passport');

//passport-auth0
var strategy = new Auth0Strategy({
  domain: '${account.namespace}',
  clientID: '${account.clientId}',
  clientSecret: 'YOUR_CLIENT_SECRET', // Replace this with the client secret for your app
  callbackURL: 'http://localhost:3000/callback',
  state: true
 },
 function(accessToken, refreshToken, extraParams, profile, done) {
   // accessToken is the token to call Auth0 API (not needed in the most cases)
   // extraParams.id_token has the JSON Web Token
   // profile has all the information from the user
   return done(null, profile);
 }
);

passport.use(strategy);

app.use(passport.initialize());
app.use(passport.session());
```
Please make sure these last two commands are in your code after the application of the express middleware (`app.use(session(sess)`).

### Storing and retrieving user data from the session

In a typical web application, the credentials used to authenticate a user are only transmitted during the login request. If authentication succeeds, a session is established and maintained via a cookie set in the user's browser. Each subsequent request does not contain credentials, but rather the unique cookie that identifies the session.

To support login sessions, Passport serializes and deserializes user instances to and from the session. Optionally, you may want to serialize only a subset to reduce the footprint, i.e., `user.id`.

```js
//app.js
passport.serializeUser(function(user, done) {
  done(null, user);
});

passport.deserializeUser(function(user, done) {
  done(null, user);
});
```

## Implement login, user profile and logout

### Add the routes

In this example, we implement the following routes:

* `/login` triggers the authentication by calling Passport's `authenticate` method. The user is then redirected to the login page as required.
* `/callback`is the route the user is returned to by Auth0 after authenticating. It redirects the user to the profile page (`/user`).
* `/user` displays the user's profile.
* `/logout` closes the local user session and redirects the user again to the root index `/`.

In the authentication step, make sure to pass the `scope` parameter with values `openid email profile` to access email and the other attributes stored in the user profile. This is needed to display the user's information on the profile page.

The `/user` route (the user's profile) should only be accessible if the user is logged in. A middleware `protected` is created for this purpose. To have full access to the user profile on `userProfile`, stringify the `user` object.

:::note
This tutorial implements logout by closing the local user session. After logging out, the user's session in the Auth0 authentication server is still open. For other implementations, please refer to the [logout documentation](/logout). 
:::

Create a new router `routes/index.js`:

```js
// routes/index.js

const express = require('express');
const passport = require('passport');
const router = express.Router();

/* GET home page. */
router.get('/', function(req, res, next) {
  res.render('index');
});

router.get('/login', passport.authenticate('auth0', {
  scope: 'openid email profile'
}), function(req, res) {
  res.redirect("/");
});

router.get('/logout', function(req, res) {
  req.logout();
  res.redirect('/');
});

router.get('/callback',
  passport.authenticate('auth0', {
    failureRedirect: '/failure'
  }),
  function(req, res) {
    const returnTo = req.session.returnTo;
    delete req.session.returnTo;
    res.redirect(returnTo || '/');
  }
);

router.get('/failure', function(req, res) {
  var error = req.flash("error");
  var error_description = req.flash("error_description");
  req.logout();
  res.render('failure', {
    error: error[0],
    error_description: error_description[0],
  });
});

module.exports = router;
```

Create a middleware to ensure a user is logged in in order to access a route:
```js
// lib/middleware/protected.js

module.exports = function() {
  return function protected(req, res, next) {
    if (req.user) { return next(); }
    req.session.returnTo = req.originalUrl;
    res.redirect('/login');
  }
};
```

Create a router for the user profile and use the protected middleware to secure the profile page:

```js
// routes/user.js

const express = require('express');
const protected = require('../lib/middleware/protected');
const router = express.Router();

/* GET user profile. */
router.get('/', protected(), function(req, res, next) {
  res.render('user', {
    user: req.user ,
    userProfile: JSON.stringify(req.user, null, '  ')
  });
});

module.exports = router;
```

Include the new router in your app, by requiring the file in the top section of your `app.js`:

```js
// app.js

var indexRouter = require('./routes/index');
var authRouter = require('./routes/auth');

//..
//..

app.use('/', indexRouter);
//..
app.use('/', authRouter);
//..
```

### Implement a middleware to make the user available in views

In the views and layouts, it is often necessary to conditionally render content depending on if a user is logged in or not. Other times, the user object might be necessary in order to customize the view.

Follow the steps below to create a middleware for this purpose.

```js
// lib/middleware/userInViews.js

module.exports = function() {
  return function(req, res, next) {
    res.locals.user = req.user;
    next();
  };
}
```

In `app.js`, require the middleware at the top and use the middleware to make it available in the views. 

```js
// app.js

//...
const userInViews = require('./lib/middleware/userInViews');
//...


app.use(userInViews());

//...
// Handle auth failure error messages
app.use(function(req, res, next) {	app.use(function(req, res, next) {
```


### Implement navigation links 

In your views, use the middleware defined in the previous step to determine which navigational links to render, depending on whether the user is logged in or not.

Add these navigation links to the application layout `views/layout.pug`.

```pug
// views/layout.pug

  body
    // ...
    a(href="/") Home
    if locals.user
      a(href="/user") Profile
      a(href="/logout") Log Out
    else
      a(href="/login") Log In
    
    // ...
    block content
```

### Implement the user profile view 

Create a `views/user.pug` template. Present the information by accessing the `user` object or the pre-populated `userProfile` variable, which contains the stringified user profile.

```pug
// views/user.pug

extends layout

block content
  br
  img(src=user.picture)
  p Welcome #{user.nickname}!
  br
  h2 User Profile
  pre #{userProfile}
```

## See it in action

Start your app and point your browser to [http://localhost:3000](http://localhost:3000). Follow the **Log In** link to log in or sign up to your Auth0 tenant. Upon successful login or signup, you should be redirected to the user's profile page.

![login page](/media/articles/web/hosted-login.png)

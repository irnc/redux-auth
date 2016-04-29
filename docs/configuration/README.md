# Configuration

This is the most difficult step, but only because configuring a [redux][redux] app is inherently difficult.

The following example assumes that you are familiar with redux, and that you know how to [create a store][redux-create-store]. Also keep in mind that this is a really basic example that does not even include routing. See the [demo app][redux-auth-demo] for a more complete example.

This example assumes a directory structure that looks like this:

~~~
src/
  app.js
  client.js
  server.js
~~~

##### config shared by both client and server
~~~js
// app.js
import React from "react";
import {Provider} from "react-redux";
import {configure, authStateReducer, AuthGlobals} from "redux-auth";
import {createStore, compose, applyMiddleware, combineReducers} from "redux";
import {AuthGlobals} from "redux-auth"

class App extends React.Component {
  render() {
    return (
      <div>
        <AuthGlobals />
        {this.props.children}
      </div>
    );
  }
}

// create your main reducer
const reducer = combineReducers({
  auth: authStateReducer,
  // ... add your own reducers here
});

// create your app's store.
// note that thunk is required to use redux-auth
const store = compose(
  applyMiddleware(thunk),
  // ... add additional middleware here (router, etc.)
)(createStore)(reducer);

// a single function can be used for both client and server-side rendering.
// when run from the server, this function will need to know the cookies and
// url of the current request. also be sure to set `isServer` to true.
export function renderApp({cookies, isServer, currentLocation} = {}) {
  // configure redux-auth BEFORE rendering the page
  store.dispatch(configure(
    // use the FULL PATH to your API
    {apiUrl: "http://api.catfancy.com"},
    {isServer, cookies, currentLocation}
  )).then(({redirectPath, blank} = {}) => {
    if (blank) {
      // if `blank` is true, this is an OAuth redirect and should not
      // be rendered
      return <noscript />;
    } else {
      return (
        <Provider store={store} key="provider">
          <App />
        </Provider>
      );
    }
  });
}
~~~

##### server-side rendering configuration
~~~js
// server.js
import qs from "query-string";
import {renderToString} from "react-dom/server";
import { renderApp } from "./app";

// render the main app component into an html page
function getMarkup(appComponent) {
  var markup = renderToString(appComponent)

  return `<!doctype html>
    <html>
      <head>
        <title>Redux Auth – Isomorphic Example</title>
      </head>
    <body>
      <div id="react-root">${markup}</div>
      <script src="/path/to/my/scripts.js"></script>
    </body>
  </html>`;
}

// this function will differ depending on the serverside framework that
// you decide to use (express, hapi, etc.). The following example uses hapi
server.ext("onPreResponse", (request, reply) => {
  var query = qs.stringify(request.query);
  var currentLocation = request.path + (query.length ? "?" + query : "");
  var cookies = request.headers.cookies;

  renderApp({
    isServer: true,
    cookies,
    currentLocation
  }).then(appComponent => {
    reply(getMarkup(appComponent));
  });
}
~~~

##### client side rendering configuration

~~~js
// client.js
import React from "react";
import ReactDOM from "react-dom";
import { renderApp } from "./app";

const reactRoot = window.document.getElementById("react-root");
renderApp().then(appComponent => {
  ReactDOM.render(appComponent, reactRoot);
});
~~~

**Note:** be sure to include the [`AuthGlobals`](#authglobals) component at the top level of your application. This means **outside** of your `Routes` if you're using something like [react-router][react-router].

See below for the complete list of configuration options.

# Multiple user types

This plugin allows for the use of multiple user authentication endpoint configurations. The following example assumes that the API supports two user classes, `User` and `EvilUser`. The following examples assume that `User` authentication routes are mounted at `/auth,` and the `EvilUser` authentication routes are mounted at `evil_user_auth`.

### Multiple user type configuration

When using a single user type, you will pass a single object to the `configure` method as shown in the following example.

~~~js
store.dispatch(configure({
  apiUrl: 'https://devise-token-auth.dev'
}));
~~~

When using multiple user types, you will instead pass an array of configurations, as shown in the following example.

~~~javascript
store.dispatch(configure([
  {
    default: {
      apiUrl: 'https://devise-token-auth.dev'
    }
  }, {
    evilUser: {
      apiUrl:                  'https://devise-token-auth.dev',
      signOutUrl:              '/evil_user_auth/sign_out',
      emailSignInPath:         '/evil_user_auth/sign_in',
      emailRegistrationPath:   '/evil_user_auth',
      accountUpdatePath:       '/evil_user_auth',
      accountDeletePath:       '/evil_user_auth',
      passwordResetPath:       '/evil_user_auth/password',
      passwordUpdatePath:      '/evil_user_auth/password',
      tokenValidationPath:     '/evil_user_auth/validate_token',
      authProviderPaths: {
        github:    '/evil_user_auth/github',
        facebook:  '/evil_user_auth/facebook',
        google:    '/evil_user_auth/google_oauth2'
      }
    }
  }
], {
  isServer,
  cookies,
  currentLocation
));
~~~

### Using multiple endpoints with redux-auth components

All components accept an `endpoint` attribute that will determine which of the endpoint configurations the component should use. For forms that are used by non-authenticated users users (`EmailSignInForm`, `OAuthSignInButton`, etc.), the first configuration in the endpoint config will be used as the default if this value is not provided. For forms used by authenticated users (`SignOutButton`, `UpdatePassword`, etc.), the current user's endpoint will be used by default if this value is not provided.

The following example assumes a configuration where two endpoints have been defined, `default` and `auth`:

##### Component example:
~~~js
import { EmailSignInForm } from "redux-auth";

// within render method
<EmailSignInForm endpoint="alt" />
~~~

--

### Endpoint config options

This is the complete list of options that can be passed to the endpoint config.

~~~js
import { configure } from "redux-auth";

// ... configure store, routes, etc... //

store.dispatch(configure({
  apiUrl:                  'https://devise-token-auth.dev',
  signOutPath:             '/evil_user_auth/sign_out',
  emailSignInPath:         '/evil_user_auth/sign_in',
  emailRegistrationPath:   '/evil_user_auth',
  accountUpdatePath:       '/evil_user_auth',
  accountDeletePath:       '/evil_user_auth',
  passwordResetPath:       '/evil_user_auth/password',
  passwordUpdatePath:      '/evil_user_auth/password',
  tokenValidationPath:     '/evil_user_auth/validate_token',
  authProviderPaths: {
    github:    '/evil_user_auth/github',
    facebook:  '/evil_user_auth/facebook',
    google:    '/evil_user_auth/google_oauth2'
  }
}).then(// ... render your app ... //
~~~

#### apiUrl
###### string
The base route to your api. Each of the following paths will be relative to this URL. Authentication headers will only be added to requests with this value as the base URL.

--

#### tokenValidationPath
###### string
Path (relative to `apiUrl`) to validate authentication tokens. [Read more](#token-validation-flow).

--

#### signOutPath
###### string
Path (relative to `apiUrl`) to de-authenticate the current user. This will destroy the user's token both server-side and client-side.

--

#### authProviderPaths
###### object
An object containing paths to auth endpoints. Keys should be the names of the providers, and the values should be the auth paths relative to the `apiUrl`. [Read more](#oauth-2-authentication-flow).

--

#### emailRegistrationPath
###### string
Path (relative to `apiUrl`) for submitting new email registrations. [Read more](#email-registration-flow).

--

#### accountUpdatePath
###### string
Path (relative to `apiUrl`) for submitting account update requests.

--

#### accountDeletePath
###### string
Path (relative to `apiUrl`) for submitting account deletion requests.

--

#### emailSignInPath
###### string
Path (relative to `apiUrl`) for signing in to an existing email account.

--

#### passwordResetPath
###### string
Path (relative to `apiUrl`) for requesting password reset emails.

--

#### passwordUpdatePath
###### string
Path (relative to `apiUrl`) for submitting new passwords for authenticated users.

--

# passport-ldapauth

[Passport](http://passportjs.org/) authentication strategy against LDAP server. This module is a Passport strategy wrapper for [ldapauth-fork](https://github.com/vesse/node-ldapauth-fork). There are two ways to authenticate with LDAP - 

1. Bind-Search-Bind - Binding with an admin user and searching for the user trying to login to get his DN and use that to login
2. Direct Bind - Use a predefined DN pattern in which the user principal (username) is filled in to obtain a complete DN and bind with that

This fork attempts to provide an authentication similar to `Direct Bind`, but actually doing `Bind-Search-Bind` with the user's credentials rather than an admin credential. LDAP Auth needs to be enhanced to support pure direct bind. Once that is done, then this will become real direct bind. 

## Install

```
npm install passport-ldapauth
```

## Status

[![Build Status](https://travis-ci.org/vesse/passport-ldapauth.png)](https://travis-ci.org/vesse/passport-ldapauth)
[![Dependency Status](https://gemnasium.com/vesse/passport-ldapauth.png)](https://gemnasium.com/vesse/passport-ldapauth)

## Usage

### Configure strategy

```javascript
var LdapStrategy = require('passport-ldapauth');

passport.use(new LdapStrategy({
    server: {
      url: 'ldap://localhost:389',
      ...
    }
  }));
```

* `server`: LDAP settings. These are passed directly to [ldapauth-fork](https://github.com/vesse/node-ldapauth-fork). See its documentation for all available options.
    * `url`: e.g. `ldap://localhost:389`
    * `bindDn`: e.g. `cn='root'`
    * `bindCredentials`: Password for bindDn
    * `searchBase`: e.g. `o=users,o=example.com`
    * `searchFilter`:  LDAP search filter, e.g. `(uid={{username}})`. Use literal `{{username}}` to have the given username used in the search.
    * `searchAttributes`: Optional array of attributes to fetch from LDAP server, e.g. `['displayName', 'mail']`. Defaults to `undefined`, i.e. fetch all attributes
    * `tlsOptions`: Optional object with options accepted by Node.js [tls](http://nodejs.org/api/tls.html#tls_tls_connect_options_callback) module.
* `usernameField`: Field name where the username is found, defaults to _username_
* `passwordField`: Field name where the password is found, defaults to _password_
* `useDirectBind`: Use direct bind instead of Bind-Search-Bind
* `passReqToCallback`: When `true`, `req` is the first argument to the verify callback (default: `false`):

        passport.use(new LdapStrategy(..., function(req, user, done) {
            ...
            done(null, user);
          }
        ));

Note: you can pass a function instead of an object as `options`, see the [example below](#options-as-function)

### Authenticate requests

Use `passport.authenticate()`, specifying the `'ldapauth'` strategy, to authenticate requests.

#### `authenticate()` options

In addition to [default authentication options](http://passportjs.org/guide/authenticate/) the following options are available for `passport.authenticate()`:

 * `badRequestMessage`  flash message for missing username/password (default: 'Missing credentials')
 * `invalidCredentials`  flash message for `InvalidCredentialsError`, `NoSuchObjectError`, and `/no such user/i` LDAP errors (default: 'Invalid username/password')
 * `userNotFound`  flash message when LDAP returns no error but also no user (default: 'Invalid username/password')
 * `constraintViolation`  flash message when user account is locked (default: 'Exceeded password retry limit, account locked')

## Express example

```javascript
var express      = require('express'),
    passport     = require('passport'),
    bodyParser   = require('body-parser'),
    LdapStrategy = require('passport-ldapauth');

var OPTS = {
  server: {
    url: 'ldap://localhost:389',
    bindDn: 'cn=root',
    bindCredentials: 'secret',
    searchBase: 'ou=passport-ldapauth',
    searchFilter: '(uid={{username}})'
  }
};

var app = express();

passport.use(new LdapStrategy(OPTS));

app.use(bodyParser.json());
app.use(bodyParser.urlencoded({extended: false}));
app.use(passport.initialize());

app.post('/login', passport.authenticate('ldapauth', {session: false}), function(req, res) {
  res.send({status: 'ok'});
});

app.listen(8080);
```

### Active Directory over SSL example

Simple example config for connecting over `ldaps://` to a server requiring some internal CA certificate (often the case in corporations using Windows AD).

```javascript
var fs = require('fs');

var opts = {
  server: {
    url: 'ldaps://ad.corporate.com:636',
    bindDn: 'cn=non-person,ou=system,dc=corp,dc=corporate,dc=com',
    bindCredentials: 'secret',
    searchBase: 'dc=corp,dc=corporate,dc=com',
    searchFilter: '(&(objectcategory=person)(objectclass=user)(|(samaccountname={{username}})(mail={{username}})))',
    searchAttributes: ['displayName', 'mail'],
    tlsOptions: {
      ca: [
        fs.readFileSync('/path/to/root_ca_cert.crt')
      ]
    }
  }
};
...
```

<a name="options-as-function"></a>
## Asynchronous configuration retrieval

Instead of providing a static configuration object, you can pass a function as `options` that will take care of fetching the configuration. It will be called with the and a callback function having the standard `(err, result)` signature. Notice that the provided function will be called on every authenticate request.

```javascript
var getLDAPConfiguration = function(req, callback) {
  // Fetching things from database or whatever
  process.nextTick(function() {
    var opts = {
      server: {
        url: 'ldap://localhost:389',
        bindDn: 'cn=root',
        bindCredentials: 'secret',
        searchBase: 'ou=passport-ldapauth',
        searchFilter: '(uid={{username}})'
      }
    };

    callback(null, opts);
  });
};

var LdapStrategy = require('passport-ldapauth');

passport.use(new LdapStrategy(getLDAPConfiguration,
  function(user, done) {
    ...
    return done(null, user);
  }
));
```

## License

MIT

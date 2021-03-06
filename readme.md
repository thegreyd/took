Command line token manager for OpenID Connect. Supports authorization
flow and direct access (resource owner password credentials grant).

# What does it do?

The main purpose of took is to maintain access tokens for API
invocations and refresh them as needed. Once things are set up, you
can run:

```
   took token myapi myuser
```
and it should either print out the token for myuser to call myapi,
or take you through authentication and then print the token. If the
token is expired and if there is a refresh token, it should get you
a new token without any further interaction. Once you have a valid
token, you can do:

```
   curl -H "Authorization: Bearer `took token myapi myuser`" http://myapi
```
or
```
   curl -H `took token -e myapi myuser`" http://myapi
```


# Setup

Run

```
  took setup
```

If this is the first time took is run, this will ask you if you want
to keep your configuration and tokens encrypted on disk. Once you
decide whether you want to do that or not and enter your password if
you do, the setup command will take you through the setup of an OIDC
authorization server based on a known server profile.

 * Enter the name of the server profile corresponding to the server you want to authenticate with
 * Assign a name to this authentication configuration
 * You need to enter the following options to create a new authentication configuration:
    * client id
    * client secret
    * callback url (not required for password grants)
    * whether the client will use password grants or not

After entering all the information, you can run:

```
  took token confName userName
```

This will take you through authentication, and will print out your token.

Server profile information is stored in /etc/took.yaml. You can add
more server profiles by editing that file.

Setup command is only useful to add a new authentication configuration
based on a server profile. To add an authentication configuration, use
the "took add" command.

## Add new authentication server using the "add" command

If your authentication server is not listed as one of the known server
profiles (defined in /etc/took.yaml), then you have to use this
method.

```
  took add oidc <options>
```


Required options for oidc:
 * -n Name of the configuration. This is the 'myapi' parameter in the above examples
 * -c Client ID
 * -s Client secret
 * -u Server URL, including domain, excluding protocol specific paths


The following sets up a configuration called 'prod' using OIDC authorization flow:
```
  took add oidc -n prod -c 12345 -s abcdef -u https://myserver/realms/myrealm -b http://callback
```

Then, when you run
```
  took token prod myuser
```
It will ask you to visit a URL. That URL will authenticate the user, and redirect to the
callback URL, 'http://callback'. Copy this URL, and paste it to the command line, and it should print out a new token.

## Direct Access Grants Flow

Took supports direct access grants. In this flow, took asks username and password, and sends 
them to the authentication server.

```
  took add oidc -n prod-direct -c 12345 -s abcdef -u https://myserver/realms/myrealm -p
```
To use this, the authentication server must be configured to support 
direct access grants flow for this client.


# Multiple users 

Took can maintain tokens for multiple users. If username is omitted, the last username will be used:

```
  took token myapi user1
  <token for user1>

  took token myapi
  <token for user1>

  took token myapi user2
  <token for user2>

  took token myapi
  <token for user2>
```

# (In)security

Took can be run in one of three different security modes:

## With Encrypted Configuration and Tokens

Took stores authentication server credentials, access tokens and
refresh tokens in ~/.took.yaml. This file is created with owner
read/write mode, so you might think this is secure enough. However, if
you are not comfortable storing plaintext credentials on disk, you
have the option to encrypt them with a password. When you run took for
the first time, it'll take you through the steps to encrypt the
configuration file. If however you did not want to encrypt then, and
you want to encrypt now, run:

```
  took encrypt
```

This will ask you a password to encrypt the configuration file.

Once the configuration file is encrypted, you have to decrypt it to
use it.

```
  took decrypt -t 10m
```

This will ask the encryption password, and if the password is correct,
start the decryption server with an idle timeout of 10 minutes. The
server will stop after 10 minutes of inactivity. If you do not specify
-t flag, the default is 10 minutes. You may specify a 0 timeout, which
will start a server that will never terminate until the current
terminal session logs out.

Warning: Took does not store your password. If you forget it, there is
no way to recover it.

## With Plaintext Configuration and Tokens

When took asks you whether you want to encrypt the configuration or
not, answer "N", and it will not ask you for a decryption password
again. Authentication service credentials, access tokens, and refresh
tokens will be stored as plaintext in ~/.took.yaml. If this makes you
uncomfortable, you can run

```
  took encrypt
```

to encrypt your configuration file.

## Insecure mode

Took requires you use https:// URLs for your servers, and it validates
the server certificates. If you do not want to validate certificates,
or if you want to call http:// server URLs, you have to run took as
took-insecure. You can copy the took executable with that name, or
create a symlink.

```
  ln -s took took-insecure
```

When run as took-insecure, you can use the -k flag to disable
certificate validation. Also, took will not complain if you make calls
to http:// servers. 

# Hack: Bypassing the server login page

It might be possible to describe the authentication form used by your server, so took can emulate
what the browser does to authenticate a user. When you go to the login page with the browser,
inspect the HTML page, and identify the forms and input fields. For instance, my server has the following
form:
```
<form id="kc-form-login" class="form-horizontal" onsubmit="login.disabled = true; return true;" 
action="https://sso.someserver/auth/realms/myrealm/login-actions/authenticate?code=QWI1Bmwm0&amp;execution=bca7381b-65b-4196-936c-7f8941f121&amp;client_id=security-admin-console&amp;tab_id=uRub-YYUVuk" method="post">
  <div class="form-group">
    <div class="col-xs-12 col-sm-12 col-md-4 col-lg-3">
       <label for="username" class="control-label">Username or email</label>
    </div>
    <div class="col-xs-12 col-sm-12 col-md-8 col-lg-9">
      <input tabindex="1" id="username" class="form-control" name="username" value="" type="text" autofocus autocomplete="off" />
     </div>
  </div>
  <div class="form-group">
    <div class="col-xs-12 col-sm-12 col-md-4 col-lg-3">
       <label for="password" class="control-label">Password</label>
    </div>
    <div class="col-xs-12 col-sm-12 col-md-8 col-lg-9">
      <input tabindex="2" id="password" class="form-control" name="password" type="password" autocomplete="off" />
   </div>

```
This HTML page has a form with id="kc-form-login", containing two input fields: username and password. 
You can define this structure with the -F flag:

```
took add oidc -n myapi -s 123 -b http://callback -c abc -u https://myserver \
 -F '{"id":"kc-form-login","usernameField":"username","passwordField":"password","fields":[{"input":"username","prompt":"User name"},\
    {"input":"password","prompt":"Password","password":true}]}'
```

When a new token is requested, took will ask for the username and password fields, submit the HTML
form, and get the tokens.

# Code Organization

Took is designed as a generic front-end for multiple authentication protocols. Protocol implementations
should be under proto/, and included in main.go. When included, protocol implementation registers its
own command line handlers. Currently there is only OIDC.

These are the directories:

 * cmd/: Files under this package contain all the entry points for commands. There should be no
   dependency from this package to protocol implementations under proto/.
 * proto/: This package contains a protocol registry, and utility functions common to all protocols.
   In particular, HTTP utilities in this package should be used for all HTTP calls, because they look
   at the secure flag and turn off certificate validation
 * proto/oidc: This is the OIDC implementation. When included, this implementation registers command line
   commands, and registers itself to the registry. 
   * cfg.go: Contains the ServerProfile struct, and the code to merge default configs to user configs
   * cmd.go: Contains command line commands. The setup wizard is also here.
   * htmlform.go: Contains the parsing code that reads a login web page,parses login fields, and asks those
     fields in the command line.
   * protocol.go: Contains the implementation of 'token' command
   * refresh.go: Token refresh logic
   * serverinfo.go: Contains the code to get auth server information (part of oidc spec)
   * validate.go: Contains token validation code
 * crypta/: This package deals with encrypting/decrypting the tokens file.
   * crypta.go: Contains the encryption/decryption implementation.
   * proto.go: Defines a simple rpc protocol 
   * rpc/: This package contains an rpc server and rpc client
   When took detects that the token file is encrypted, it asks for the user password, and starts
   another instance of took with an RPC server listening to a domain socket under $HOME. This
   RPC server contains all the ecnryption/decryption functions unlocked using the user's password.

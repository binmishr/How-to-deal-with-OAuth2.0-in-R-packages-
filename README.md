# How-to-deal-with-OAuth2.0-in-R-packages

What is OAuth 2.0?

OAuth 2.0 is the most recent standard of OAuth, coming after OAuth 1.0. But you were probably not looking for this kind of answer. 😸

OAuth 2.0 is a way for a resource server (say, Twitter API) to grant access to (part of) your account to an app (say, a Twitter alternative app, or rtweet) on your behalf. It is safer than previous industry standards, see the video below.

In practice, with OAuth 2.0 there is a dance i.e. a series of requests and redirects between the server and the third-party app, launched by you when starting to install an app, and validating by you as one of the steps is your clicking something like “Yes, I grant access to this and that right on my Twitter account, to the app Twitter Lite”. Aaron Parecki (featured in the video above) has a great write-up on simplified OAuth 2.0.

To use things on the server one needs an access token (a string, e.g. e46b7faf296e3f0624e6240a6efafe3dfb17b92ae0089c7e51952934b60749f2). The whole point of the OAuth 2.0 dance is to get such an access token. Often you will also get a refresh token (similarly a string, e.g. 3bdeb3bd19a3093674f4e7ba6e1be1706ab1f16af9ebf3b79a67a807c622b650). The refresh token allows to get issued a new access token without any interactivity i.e. no need for the user to click. A common validity period for an access token would be 24 hours, for a refresh token one year, and often the refresh token is re-usable.

Now while this all makes perfect sense when you are developing an actual third-party app, for interactive usage… when all you want is an R package that can be used in scripts running on a server, it’s much less clear.
How do you use OAuth 2.0 in R?

When writing an R package wrapping an API using OAuth 2.0 you’ll need the user to grant access to an “app”, which will allow to create an access token and a refresh token. The access token will then often be passed to the API in a header when making requests, whilst the refresh token would be posted in a query string when the access token needs to be renewed.

Your problem is: how do I imitate a third-party app? Thankfully for you, in most cases the complexity can be handled by the httr package. For other cases, or if you want to e.g. only use curl, you will have to get creative. 😉

How does the OAuth 2.0 dance and usage happens with httr?
OAuth endpoint

    You create an OAuth endpoint (via httr::oauth_endpoint()) which is an object holding the URLs to which requests need to be made to request and refresh an access token. Or you use one of the built-in httr::oauth_endpoints() to popular APIs.

How do you know what URLs to feed httr::oauth_endpoint()? By reading the API docs and looking at examples. 😁 E.g. compare ORCID API docs and rorcid source code.
OAuth app

    You create an OAuth app (via httr::oauth_app()) which is an object holding the name, secret and ID of your third-party app. What third-party app?! Say you are using rtweet. You need to register an app in your Twitter developer account e.g. “My Fictional App”, you’ll get a secret and ID in return. There is no actual app (pfiew!) but Twitter now thinks there is one. The app info will be used by httr when requesting a token, and the user (you) will grant access to their account to “My Fictional App”. The server e.g. Twitter will ask you for a callback URL which needs to be http://localhost:1410/, that as we’ll see later httr will listen to to catch tokens produced by the server.

In an R package, if possible (depending on the API rules), it is nice to have a built-in OAuth app for quick experiments by the users; and the way for users to specify their own app’s information (Bring Your Own App).

In the gargle docs there is a nice discussion of the security risk of a built-in OAuth app, by Jenny Bryan:

    “Package maintainers might want to build this app in as a fallback, possibly taking some measures to obfuscate the client ID and secret and limit its use to your package.

        Note that three-legged OAuth always requires the involvement of a user, so the word “secret” here can be somewhat confusing. It is not a secret in the same sense as a password or token. But you probably still want to store it in an opaque way, so that someone else cannot easily “borrow” it and present an OAuth consent screen that impersonates your package.”

OAuth token creation and caching

    Lastly, you’d call httr::oauth2.0_token() that will open a browser window where the user can grant access to the third-party app. This step therefore needs interactivity. Its output is an httr token object containing among other things the access token and the refresh token if it was given.

Interestingly there is still no app involved but httr starts a web server using httpuv. This way when the server sends the access token to http://localhost:1410/, httr is listening and gets it.

Interactively, if no parameter / option is set for the cache argument of httr::oauth2.0_token(), the user will be asked by httr whether they want to cache the token to a file. As a package developer, ideally you’d surface options in your function setting up authentication with these concerns in mind:

    security (if the token is cached to the current project, is it gitignored?);
    user-friendliness (not caching the token means interactive authentication every time the package is used!);
    flexibility (some users will have one token but some might have several if they are juggling several, say, Twitter accounts).

In rtweet the token is by default saved to the home directory and the path to it is saved as an environment variable. More easily, you could save the token to an app directory.
OAuth usage

    With httr you can directly pass the token to httr::VERB() via the config parameter. httr also automatically refreshes your access token if the token object contains a valid refresh token.
    If using another HTTP R client you will need to extract the access token and pass it as the header the API docs require.

What are your OAuth 2.0 secret credentials?

Your secrets are

    the access token;
    the refresh token;
    the file caching them on disk (.httr-oauth or another filename). The R object in it is called a token but it contains two actual tokens (the access token and refresh token). 😵

Secret OAuth 2.0 httr token file

Often as you are not storing HTTP requests you will only need to worry about the file.

If you need to use your OAuth 2.0 R package / script on a server, you will need a cached httr token file. If your server content (GitHub repo?) is public, you will have to encrypt the file and to store a text-version of the encryption key/passwords as e.g. GitHub repo secret if you use GitHub Actions. Read the documentation of the continuous integration / cloud service you are using to find out how secrets are protected and how you can use them in your builds.

See gargle vignette about securely managing tokens. gargle is directly useful if you are wrapping a Google API, but reading this vignette is useful no matter what API you are wrapping, for the clear explanations of challenges and solutions.

The approach is:

    Create your OAuth token locally, either outside of your package/project folder, or inside of it if you really want to, but gitignored and Rbuildignored.
    For encrypting you need some sort of password. You will want to save it securely as text in your user-level .Renviron and in your GitHub repo secrets (or equivalent secret place for other CI services). E.g. create a key via sodium_key <- sodium::keygen() and get its text equivalent via sodium::bin2hex(sodium_key). E.g. the latter command might give you e46b7faf296e3f0624e6240a6efafe3dfb17b92ae0089c7e51952934b60749f2 and you would save this in .Renviron with a line such as the one below.

GIVEN_SERVICE_OAUTH_PWD="e46b7faf296e3f0624e6240a6efafe3dfb17b92ae0089c7e51952934b60749f2"

    Encrypt the OAuth token using e.g. the user-friendly cyphr package. Save the code for this and for the step before in a file e.g. inst/secrets.R for when you need to re-create a token as even refresh tokens expire.

Example of a script creating and encrypting an OAuth token (for the Meetup API).

token_path <- testthat::test_path(".meetup_token.rds") # or just token_path <- ".meetup_token.rds"
use_build_ignore(token_path)
use_git_ignore(token_path)

# the function below calls httr::oauth2.0_token()
meetupr::meetup_auth(
  token = NULL,
  cache = TRUE,
  set_renv = FALSE,
  token_path = token_path
)

# sodium_key <- sodium::keygen()
# create the environment variable "GIVEN_SERVICE_OAUTH_PWD" and set it to sodium::bin2hex(sodium_key))

# get key from environment variable
key <- cyphr::key_sodium(sodium::hex2bin(Sys.getenv("GIVEN_SERVICE_OAUTH_PWD")))
cyphr::encrypt_file(
  token_path,
  key = key,
  dest = testthat::test_path("secret.rds")
)

    In your script using the token, you would have code like below.

key <- cyphr::key_sodium(sodium::hex2bin(Sys.getenv("GIVEN_SERVICE_OAUTH_PWD")))
temptoken <- tempfile(fileext = ".rds")
cyphr::decrypt_file(
  testthat::test_path("secret.rds"),
  key = key,
  dest = temptoken
)

All of this only works if your code automatically renews the access token, and if the refresh token is re-usable and valid for a long enough time. After one year or so, depending on the API, you will have to re-create your httr token file because the refresh token will have expired.
Secret access token, secret refresh token

Now if you are storing your HTTP interactions to disk e.g. if using the vcr package for HTTP testing, you will need to know how the secrets are used in HTTP requests. Often,

    access tokens are passed in the Authorization header;
    refresh tokens are passed in a query string.

Make sure you never publish a file with unedited tokens! E.g. for vcr read the security docs.
How do you test OAuth 2.0 with R?

If you want to test how your package handles different OAuth 2.0 scenarios, you might be interested in the OAuth 2.0 apps provided by the webfakes package.

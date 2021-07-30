# OData authentication code flow

We went through this flow together in [Exploring SAP Graph together - Part 4](https://www.youtube.com/watch?v=4Dhg5WCQUNo). For context, have a look at the [Protocol Flow](https://datatracker.ietf.org/doc/html/rfc6749#section-1.2) and [Refresh Token](https://datatracker.ietf.org/doc/html/rfc6749#section-1.5) flow diagrams in the [OAuth RFC (6749)](https://datatracker.ietf.org/doc/html/rfc6749).

These steps are deliberately and extremely manual so that you can understand the flow.

## Step 0 - set up credentials

Request a set of credentials from the SAP Graph team, as described in this blog post: [Part 3: Use SAP Graph securely with real data - authentication](https://blogs.sap.com/2021/06/25/part-3-use-sap-graph-securely-with-real-data-authentication/). You should receive a `credentials.json` file. Store this securely, for example in your home directory:

```shell
chmod 400 credentials.json
mv credentials.json ~/
```

In this directory (where this document and the scripts are stored), create a symbolic link to this `credentials.json` file, and call it `key.json`. This is so that the [`skv`](https://github.com/qmacro/dotfiles/tree/master/scripts/skv) utility can be used in a simple way to retrieve values from it:

```shell
ln -s key.json ~/credentials.json
```

## Step 1 - request authorization grant

First, a request for an authorization code is made to the resource owner. The `1_request_auth_grant` file contains the construction of an appropriate URL for this, based on values in the `credentials.json` file. Open this URL in a browser, authenticate yourself if requested, and you should be redirected to a URL that looks like this:

```
http://localhost/?code=XXX
```

where `XXX` represents the authorization code.

Here's one way of opening the URL in a browser; I use Brave in incognito mode, to ensure I get a chance to authenticate using the standard identity provider flow, rather than automatically via client certificates that are installed on my machine:

```shell
brave-cli open "$(./1_request_auth_grant)" -i
```

> Before you try this, be ready to perform the next step, because the lifetime of the authorisation code is short, and if you take more than a few seconds to request the access token with the code you receive, you may receive an "invalid code" error.

## Step 2 - request access token

The authorization code now needs to be exchanged for an access token. Copy the code from the URL in your browser (i.e. whatever `XXX` is in `http://localhost/?code=XXX`) and use the `curl` invocation in `2_request_access_token`, specifying the code when you call it, like this:

```shell
./2_request_access_token XXX
```

This will make the request to the Authorization Server, sending it the code, and asking for an access token in return. The response will be in JSON representation, and is saved to a file `token.json`.

If successful, not only will an access token be sent in the response, but supporting values too (take a look inside the file to find out), such as:

|Property|Description|
|-|-|
|`access_token`|The access token itself|
|`token_type`|The type of token - it should be "bearer"|
|`refresh_token`|What to use when you need to refresh the access_token|
|`expires_in`|How long the access token is valid, in seconds (43200 seconds is 12 hours)|
|`scope`|The scope of access represented in the token|

At this point you have an access token that you can use to authenticate HTTP requests to protected resources. Specify the token in your HTTP requests, in an Authorization header, like this:

```
Authorization: Bearer ACCESSTOKEN
```

where `ACCCESSTOKEN` is your access token (it may be quite a long value, make sure you include all of it).

## Step 3 - refresh access token

At some point your HTTP requests will start to fail, when the access token expires. At this point you can request a fresh access token, using the refresh token you got in the previous step.

Use the `curl` command in `3_refresh_access_token` to do this; the values constructed in the URL assume you have the `token.json` file from the previous step.

```shell
./3_refresh_access_token
```

If successful, the response will be another set of JSON properties; this time, so that you can compare them, these are saved (by `3_refresh_access_token`) into a file called `refreshed.json`. You should then use the value of the `access_token` from this response until it expires again, and then repeat this step.

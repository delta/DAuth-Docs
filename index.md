---
layout: default
title: Home
---

# Contents
* [What is Dauth?](#what-is-dauth)
* [Getting Started](#getting-started)
* [Authorisation Code Flow](#authorisation-code-flow)
* [Endpoints](#endpoints)
* [Quick guide to integrating DAuth API](#quick-guide-to-integrating-dauth-api)

# What is DAuth?
DAuth is an OAuth2.0 based SSO (Single Sign On) for NITT students. It serves to replace existing webmail based login and redundant registration. It is more secure and easier to integrate with your services and use. 
To know more about Oauth 2.0, click [here](https://oauth.net/2/).

# Getting Started
DAuth lets the application developers securely get access to users' data without users having to share their passwords.  

<center><img src="images/getting-started.png" alt="drawing" width="50%" /></center>   

These are 3 simple steps to integrate DAuth with your applications.
1. [Create a DAuth account](https://auth.delta.nitt.edu/register) with your webmail address. 
2. Register your application (client).
3. Integrate DAuth API in your application. 

# Authorisation Code Flow

Below is the high-level flow of how your application can authenticate users and access DAuth information.   

<center><img src="images/code-flow.png" alt="drawing" width="50%" /></center>   

### Client Registration

Your application (the client) making protected resource requests on behalf of the user (resource owner) or authenticating the user has to be signed up before. So, after creating a DAuth account, navigate to the Clients section and register your application there.

Homepage URL - This is the homepage URL of your application.  
Example: `www.example.com/`  
Callback URL - This is where the user(resource owner) will be redirected back after an authorization request is made.   
Example: `www.example.com/callback/`  

### Client Credentials

A registered client will get some credentials. They include: 
* Client ID 
* Client secret - A unique secret key 
* redirectURI - Callback url that you’ve registered the application with

# Endpoints

### Authorize Endpoint
The user has to give consent for your application to use their DAuth information.The authorization endpoint is used to interact with the user and obtain an authorization grant.  

```
GET /authorize HTTP/1.1
Host: auth.delta.nitt.edu
Request Parameters:
    Required: true
    ParamType: Query
    DataType: String
```   

| Parameter | Description |
| --- | --- |
| client_id | Obtained during client registration. |
| redirect_uri | This should typically be the callback url provided during client registration. | 
| response_type | It tells the authorization server which grant to execute. Use `response_type=code` for authorization code. |
| grant_type | Use `grant_type=authorization_code` for authorization grant flow. |
| state | Used for security purposes. It is returned  back to the application as part of the redirect_uri. | 
| scope | Scopes have been defined in [bottom](#scopes). |
| nonce | It is a client generated string. It will be returned in the token and hence the client can validate the token. |

Example: 
```
client_id=qwdsfgwrTHNHRMYUKTILY&redirect_uri=https%3A%2F%2Fexample.com%2Fcallback%2F&response_type=code&grant_type=authorization_code&state=sdafsdghb&scope=email+openid+profile&nonce=bscsbascbadcsbasccabs
```   

If the user authorizes, upon success, the user is redirected back to redirect_uri with authorization code and state as query parameters.

### Token

The token endpoint is used by the client to obtain an access token by presenting its authorization grant. The user should be unaware of this request (backchannel communication).

```
POST /api/oauth/token HTTP/1.1
Host: auth.delta.nitt.edu
Content-Type: application/x-www-form-urlencoded
Request Parameters:
    Required: true
    ParamType: Query
    DataType: String
```    

| Parameter | Description |
| --- | --- |
| client_id | Obtained during client registration. |
| client_secret | Obtained during client registration. | 
| grant_type | Use `grant_type=authorization_code` for authorization grant flow. |
| code | Authorization code obtained through `/authorize` call. |
| redirect_uri | The callback_url client has registered with. |

Example: 
```
client_id=qwdsfgwrTHNHRMYUKTILY&client_secret=csadvfbgnrwmywtkulifjrknjvnjrnlrnjvlnfvnflv&grant_type=authorization_code&code=f65dbf63a96650e689ef9f800a63ed67177ebe45&redirect_uri=https%3A%2F%2Fexample.com%2Fcallback%2F
```  
  
On success, the response would contain  

| Parameter | Description |
| --- | --- |
| access_token |The token for accessing user resources |
| state | The state sent during authorize call is returned back |
| id_token | This token is sent if the initial `/authorize` call made involves openid in scope param |

### Key

```
GET /api/oauth/oidc/key HTTP/1.1
Host: auth.delta.nitt.edu
```
This route returns Jwks used to verify id_token.

### Resource api(user scope)

Request format:
```
POST /api/resources/user HTTP/1.1
Host: auth.delta.nitt.edu
Authorization: Bearer 02f71255ee95a04d6066de3969bb52f466c90572
```   

Response : 
```
user object (i.e the resources)
```


The access token is to be sent as a bearer token in headers(authorization header). The resources authorized by the user can be finally accessed.

### Scopes

The authorization and token endpoints allow the client to specify the scope of the access request using the `scope` request parameter.  

Accepted Scopes
1. openid
  - Specifying open_id scope sends id_token in the response of `/api/token` call.
2. email
  - Includes email of user in id_token
3. profile
  - Includes name in id_token
4. user  
  - Gives user object.

# Quick guide to integrating DAuth API 

1. Create a DAuth account and register the client.
2. Integrate the UI.   
    <center><img src="images/login-with-dauth.png" alt="drawing" width="50%" /></center>  

3. Once the user clicks the button, make a call to `/authorize` endpoint with the query parameters mentioned above.
4. The user will be redirected back to callback_url you’ve provided during client registration and gets code (authorization code) and state as query parameters.
5. Make a backchannel post request to `/api/oauth/token` using query parameters mentioned above.
6. Response will be token (authorization) and id_token (authentication). id_token will be sent back only if oidc scope is added in step 3.
7. Finally, with that token, do a POST request to `/api/resources/user` to get the user details. The token must be sent as a bearer token in headers (authorization header).

# Sample Application
Here's a toy Flask app to illustrate how to implement DAuth integration:
```python
from urllib.parse import urlencode

import requests
from flask import Flask, redirect, render_template, request

app = Flask(__name__)

CLIENT_ID = "<YOUR_CLIENT_ID>"
CLIENT_SECRET = "<YOUR_CLIENT_SECRET>"

# Note the callback URL must match EXACTLY with
# the one registrered. Even an extra trailing slash
# may cause a 400 bad request
CALLBACK_URL = "http://localhost:5000/callback"


@app.route("/")
def index():
    # login.html simply contains a a form with
    # a "Login with DAuth" button and action as login
    return render_template("login.html")


@app.route("/login", methods=["POST"])
def login():
    host = "https://auth.delta.nitt.edu/authorize"
    params = {
        "client_id": CLIENT_ID,
        "redirect_uri": CALLBACK_URL,
        "response_type": "code",
        "grant_type": "authorization_code",
        "state": "mystate",
        "scope": "email openid profile user",
        "nonce": "mynonce",
    }

    url = f"{host}?{urlencode(params)}"
    return redirect(url)


@app.route("/callback")
def callback():
    code = request.args.get("code")

    # Backchannel request, user doesn't know about this
    res = requests.post(
        "https://auth.delta.nitt.edu/api/oauth/token",
        data={
            "client_id": CLIENT_ID,
            "client_secret": CLIENT_SECRET,
            "code": code,
            "grant_type": "authorization_code",
            "redirect_uri": CALLBACK_URL,
        },
    )

    access_token = res.json().get("access_token")
    res = requests.post(
        "https://auth.delta.nitt.edu/api/resources/user",
        headers={"Authorization": f"Bearer {access_token}"},
    )

    # Contains information about the user
    # Eg:
    # {
    #   "batch": "BTech Third Year",
    #   "createdAt": "2022-12-23T03:24:07.959Z",
    #   "email": "106969420@nitt.edu",
    #   "gender": "MALE",
    #   "id": 7728,
    #   "name": "Joe Mama",
    #   "phoneNumber": "+915555577777",
    #   "updatedAt": "2024-07-16T17:00:09.628Z"
    # }
    user = res.json()

    return user


if __name__ == "__main__":
    app.run(debug=True)

```
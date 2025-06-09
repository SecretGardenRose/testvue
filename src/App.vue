<template>
  <div id="app">
    <h1>HELLOHELLO World from Vue on AWS Amplify!</h1>
  </div>




<!-- views/home.ejs -->
<!DOCTYPE html>
<html>
<head>
    <title>Amazon Cognito authentication with Node example</title>
</head>
<body>
<div>
    <h1>Amazon Cognito User Pool Demo</h1>

    <% if (isAuthenticated) { %>
        <div>
            <h2>Welcome, <%= userInfo.username || userInfo.email %></h2>
            <p>Here are some attributes you can use as a developer:</p>
            <p><%= JSON.stringify(userInfo, null, 4) %></p>
        </div>
        <a href="/logout">Logout</a>
    <% } else { %>
        <p>Please log in to continue</p>
        <a href="/login">Login</a>
    <% } %>
</div>
</body>
</html>


</template>

<script>
export default {
  name: "App",
};

const express = require('express');
const session = require('express-session');
const { Issuer, generators } = require('openid-client');
const app = express();

let client;
// Initialize OpenID Client
async function initializeClient() {
    const issuer = await Issuer.discover('https://cognito-idp.us-east-1.amazonaws.com/us-east-1_HI4d6CkYL');
    client = new issuer.Client({
        client_id: '4j36h21ol88r867aqonocdq3hj',
        client_secret: '<client secret>',
        redirect_uris: ['https://main.d27cr8ho8wdzxu.amplifyapp.com'],
        response_types: ['code']
    });
};
initializeClient().catch(console.error);

app.use(session({
    secret: 'some secret',
    resave: false,
    saveUninitialized: false
}));

const checkAuth = (req, res, next) => {
    if (!req.session.userInfo) {
        req.isAuthenticated = false;
    } else {
        req.isAuthenticated = true;
    }
    next();
};

app.get('/', checkAuth, (req, res) => {
    res.render('home', {
        isAuthenticated: req.isAuthenticated,
        userInfo: req.session.userInfo
    });
});


app.get('/login', (req, res) => {
    const nonce = generators.nonce();
    const state = generators.state();

    req.session.nonce = nonce;
    req.session.state = state;

    const authUrl = client.authorizationUrl({
        scope: 'phone openid email',
        state: state,
        nonce: nonce,
    });

    res.redirect(authUrl);
});

// Helper function to get the path from the URL. Example: "http://localhost/hello" returns "/hello"
function getPathFromURL(urlString) {
    try {
        const url = new URL(urlString);
        return url.pathname;
    } catch (error) {
        console.error('Invalid URL:', error);
        return null;
    }
}

app.get(getPathFromURL('https://main.d27cr8ho8wdzxu.amplifyapp.com'), async (req, res) => {
    try {
        const params = client.callbackParams(req);
        const tokenSet = await client.callback(
            'https://main.d27cr8ho8wdzxu.amplifyapp.com',
            params,
            {
                nonce: req.session.nonce,
                state: req.session.state
            }
        );

        const userInfo = await client.userinfo(tokenSet.access_token);
        req.session.userInfo = userInfo;

        res.redirect('/');
    } catch (err) {
        console.error('Callback error:', err);
        res.redirect('/');
    }
});


// Logout route
app.get('/logout', (req, res) => {
    req.session.destroy();
    const logoutUrl = `https://<user pool domain>/logout?client_id=4j36h21ol88r867aqonocdq3hj&logout_uri=<logout uri>`;
    res.redirect(logoutUrl);
});

app.set('view engine', 'ejs');

</script>

<style>
#app {
  text-align: center;
  margin-top: 50px;
  font-family: Avenir, Helvetica, Arial, sans-serif;
}
</style>


+++
date = "2013-10-03"
title = "Handling OAuth2 with NodeJS and AngularJS: Passport to the rescue"
+++

<p>
I've recently been working on a small application in AngularJS which will be eventually destined for mobile platforms. Now, I've been wanting to get a decent log-in flow for Node happening for a while now; it's the kind of boilerplate I'm likely to use in a lot of different things. And of course, I want to be able to use the OAuth2 log-in services provided by Facebook, Google et al. A lot of my previous experiments with node and/or angular didn't require this, and as I try to be prudent with my time it's always been on the back burner. Until now, that is...
</p>
<p>
Ever since fiddling around with the Asana API I've become a fan of the whole REST idea. Send a GET request to a server with some parameters; get a simple answer back. Send that GET request again? Get pretty much the same thing back. Simple.
</p>
<p>
This creates certain issues when dealing with AngularJS. For a start, the REST API and the client are decoupled, so this creates complications when dealing with tokens/sessions and authorisation flow. Chances are, the client and API are hosted on different domains, so we can throw cross-server requests issues in, too. None of this is insurmountable and I will attempt to highlight the steps I've taken to get over this as we come to them.
</p>
<p>
Lets look at the components to my solution. We are using:-
</p></br>
<p>
1. Angular-JS on the client, with AngularJS test web-server.js pointed at localhost:3000
</p>
<p>
2. Our Node server pointed at localhost:8000.
</p>
<p>
3. Our Node server is running the express framework and the Passport middleware. In particular, the examples will use the passport-google-oauth module.
</p></br>
<p>
I'll take the time now to present the authorisation flow with a short story:
</p>
<p></br>
1. Our intrepid hero (the user) will navigate to our angular client. Our angular client check it's cookie store to see if any session data exists. It does not, so the user is present with a log in page.
</p>
<p>
</p>
<p>
2. Our hero decides to log in with google, and clicks the 'log in with Google' button.
</p>
<p>
3. This sends a request to the API server, which responds with a url to redirect the user to the standard Google log in page. Our client dutifully redirects to this url.
</p>
<p>
4. Our hero enters the username and password, and hits 'log-in' on the Google page.
</p>
<p>
5. Google validates our users credentials and sends a response to the API server along with an access token.
</p>
<p>
6. Our API server stores this token with the user data, and redirects back to the client, sending the access token along in the response.
</p>
<p>
7. Now, back at the client, we store the token in our session header. To log out, we destroy the token. However, any requests we make to the API, we send the token to identify ourselves.
</p></br>
<p>
<p>
You can get the code here -
You'll need to set up a mongo database to handle user data - <a href="https://github.com/matt-tyler/oauth2-passport-angular" title="Github Repo ">https://github.com/matt-tyler/oauth2-passport-angular</a>. You can work out the schema I used by having a look through the models folder. This particular demo application was holding information about the users housing settlement data, hence the references to 'settlements' here and there in the code. Naturally, feel free to chop and change things to suit.
</p>

Lets look at some server code step-by-step. Some of this is going to be explaining different elements of Express.js
</p>

<pre><code>
var express = require('express')
	,routes = require('./routes')
	,UserHandler = require('./handlers/UserHandler')
	,AuthHandler = require('./handlers/AuthHandler')
	,passport = require('passport')
	,mongoose = require('mongoose')

var app = express();

var google_strategy = require('passport-google-oauth').OAuth2Strategy;

app.configure(function() {
	app.use(express.logger('dev'));
	app.use(express.json());
	app.use(express.urlencoded());
	app.use(express.methodOverride());
	app.use(express.cookieParser());
	app.use(passport.initialize());
	app.use(app.router);
	app.use(express.static(__dirname + '/public'));
});
</code></pre>

<p>
Firstly, I'm defining some simple imports. Of these, a few are third party libraries. In particular, they are express, passport and mongoose. routes exports a function that will we use to set up our routes later, and authhandler and userhandler are used to set up how will we handle our routes (ie redirects etc).
</p>
<p>
The line
</p>

<pre><code>
var app = express();
</code></pre>

<p>
Gives us our express variable in which to configure our server.
</p>
<p>
When it comes to authentication, the next line is where things start to get interesting. Passport.js has a concept of 'strategies'. These define how we handle authentication. This is so we can define several different means of handling authentication - such as oauth2, oauth1, openid, local etc. We could define several strategies and design routes and handles for all them; so we could choose to login with facebook, google, username/password etc. Decoupling authentication in like this allows contributers to help out with passport.js development by creating their own authentication strategies for third parties. In this instance, we are defining our google-strategy from the passport-google-oauth library - which, I might add, is distinct from passport.js and you will need to install alongside it in order to use it.
</p>
<p>
Finally we have some standard configurations to setup. These are to enable various middlewares in our application. Most of these are Connect.js middleware.
</p>
<p>
line by line
</p>
<pre><code>
app.use(express.logger('dev')); // 
app.use(express.json());
app.use(express.urlencoded());
app.use(express.methodOverride());
app.use(express.cookieParser());
app.use(passport.initialize());
app.use(express.static(__dirname + '/public'));
app.use(app.router);

</code></pre>

<p>
First we set up our express logger into development mode. This provides our response status with coloured output.
</p>
<p>
The next two enable some body parsing middleware. In particular we enable json and urlencoded. These parse our request bodies and place them into the object req.body, which we access at various points in our server application.
</p>
<p>
methodOverride is used to simulate DELETE and PUT requests with POST verbs. Going into this now will derail this tutorial, but if you are interested, the following resources may enlighten you. This is a particularly important aspect of REST implementation using HTTP verbs so I would definitely recommend doing some further reading.
</p>
<p>
<a href="http://stackoverflow.com/questions/4573305/rest-api-why-use-put-delete-post-get/4573320#4573320" title="REST API - Why use put-delete-post-get">REST API - Why use put-delete-post-get</a></br>
<a href="http://stackoverflow.com/questions/630453/put-vs-post-in-rest" title="Put vs Post in REST">Put vs Post in REST</a></br>
<a href="http://jcalcote.wordpress.com/2008/10/16/put-or-post-the-rest-of-the-story/" title="Put or Post the rest of the story">Put or Post the rest of the story</a>
</p>
<p>
Next we initialise passport and set up a static directory from which to serve static files. The latter is a 'nice-to-have' that I won't go into nor really use in these examples. Finally we set-up the router. You must initialise passport BEFORE initialising the router. Indeed, you should always set up the router last. This should make perfect sense, because we are setting up route middleware which essentially parses requests to the server and sets up a nice req object that we provided in our route handler - and we'd obviously need to do that before setting the router up.
</p>
<p>
Now lets imagine our we are using two servers on different domains; one to host the client, one to host the REST API. This means sending any kind of request from our client to the API is going to be a cross domain request. To enable this, we need to do some setting up in the server.
</p>
<pre><code>
var allowCrossDomain = function(req, res, next) {
    res.header('Access-Control-Allow-Origin', '*');
    res.header('Access-Control-Allow-Methods', 'GET,PUT,POST,DELETE');
    res.header('Access-Control-Allow-Headers', 'Content-Type, Authorization');
      
    if ('OPTIONS' == req.method) {
    	res.send(200);
    }
    else {
    	next();
    }
}; 
</code></pre>
<p>
First, we say that we're going allow requests from origins outside the domain. Next, we specify what methods will allow our clients to make. We are also going to allow authorization and content type headers to be requested.
</p>
<p>
Lastly, we need to ensure options if an OPTION verb is sent, that we respond to it with 200, representing success. All cross origin requests will be preceded by an options request - which tells the client if the request will be allowed. The Options request is this scenario is called a preflight request is a lingering artifact of a time when cross domain request were rare and feared for security reasons. 
</p>
<p>
<a href="http://www.bennadel.com/blog/2327-Cross-Origin-Resource-Sharing-CORS-AJAX-Requests-Between-jQuery-And-Node-js.htm">Cross Origin Resource Sharing CORS AJAX requests between jQuery and NodeJS</a></br>
<a href="http://stackoverflow.com/questions/19223047/xmlhttprequest-is-not-set-by-using-setrequestheader/19223205#19223205">XML-HTTP request is not set by using set-request-header</a></br>
<a href="http://stackoverflow.com/questions/15381105/cors-what-is-the-motivation-behind-introducing-preflight-requests">CORS What is the motivation behind introducing preflight requests</a></br>
<p>
To makes thing easier for ourselves, lets enable some basic error handling that will dump a stack trace to the console when we start our server in development mode.
</p>
<pre><code>
app.configure('development', function() {
	app.use(express.errorHandler({dumpExceptions: true, showStack: true}));
	console.log("Starting in development mode");
});
</code></pre>
<p>
I'm using mongoose to handle calls to mongodb for storing user details. Obviously you could use whatever backend you like, but I'm including for posterity.
</p>
<pre><code>
mongoose.connect('mongodb://localhost/bestwest');

var db = mongoose.connection;

db.on('error',console.error.bind(console, 'connection error:'));
db.once('open', function callback() {
	console.log("Connected to db");
});
</code></pre>

<p>
Now for some more passport focussed stuff.
</p>

<pre><code>
passport.use(new google_strategy({
    clientID: '442704851010.apps.googleusercontent.com',
    clientSecret: 'uRQL8HQ7zmf_yyfDoyUL1_eZ',
    callbackURL: 'http://devbox.example.com:3000/auth/google/callback'
  },
  function(accessToken, refreshToken, profile, done) {
	UserDB.findOne({email: profile._json.email},function(err,usr) {
		usr.token = accessToken;	
		usr.save(function(err,usr,num) {
			if(err)	{
				console.log('error saving token');
			}
		});
		process.nextTick(function() {
			return done(null,profile);
		});
	});
  }
));
</code></pre>

<p>
Ok, here we are telling passport we are using the Google strategy we defined earlier, and configuring it at the same time. From passport docs, we need to provide four things.
</p></br>
<p>
1. Our client ID, that you get when you register an application with Google.
</p>
<p>
2. Our client secret, which we get from the same place we got the client ID.
</p>
<p>
3. A callback URL, this should be the url that you want Google to redirect to you when you have logged. Note: This (in this example anyway) will NOT be the page on the client you want to redirect to after logging in, but the endpoint that you want Google to return your credentials to. We want our API server to handle this, so we are redirecting to the API server.
</p>
<p>
4. Finally a function that you want called when you are returned. In this function you will do any access token handling you might want to do, including storing the token into the appropriate users record in the database, which is exactly what I am doing. 
</p>
</br>
<p>
We also want to use process.nextTick to ensure we add our token into the database so that we can retrieve it later in our sign-in callback (which we will see shortly). If you do not do this, you might find that due to node's asynchronous nature that you attempted to return the token before it is stored in the database.
</p>
<p>
This is the standard set-up for using OAuth2 with passport - Facebook login is implemented in a similar way.
</p>
<pre><code>
var handlers = {
	user: new UserHandler(),
	auth: new AuthHandler()
};

//auth_routes.setup(app,passport);
routes.setup(app,handlers);

app.listen(3000);
console.log('Listening on port 3000');
</code></pre>
<p>
Finally, I set up some route handlers, and start listening on port 3000. 
</p>
<p>
Lets have a look at the auth handler, shall we?
</p>
<pre><code>
var AuthHandler = function() {
	this.googleSignIn = googleSignIn;
	this.googleSignInCallback = googleSignInCallback;
}
</code></pre>

<p>
Ok, we define a couple of functions on our auth handler, a call to sign-in, and a callback endpoint.
</p>
<p>
Let's take a look at those functions -
</p>
<pre><code>
function googleSignIn(req, res, next) {
	passport = req._passport.instance;
	
	passport.authenticate('google',{scope: 'https://www.googleapis.com/auth/userinfo.email'});

};
</code></pre>

<p>
We make a call to passport.authenticate which puts everything in motion. To do this, we pass 'google', which lets passport know we authenticating with Google. 
</p>
<p>
We also provide the 'scope' variable, which is a Google specific thing which tells Google what information about the user we want access to. There are a whole list of scopes, which determine what parts of the Google api we can access. When the user attempts to log in, they will be told what your application wants access to before they agree to sign in.
</p>
<p>
As for the callback function...
</p>
<pre><code>
function googleSignInCallback(req, res, next) {
	passport = req._passport.instance;
	passport.authenticate('google',function(err, user, info) {
		if(err) {
			return next(err);
		}
		if(!user) {
			return res.redirect('http://localhost:8000');
		}
		UserDB.findOne({email: user._json.email},function(err,usr) {
			res.writeHead(302, {
				'Location': 'http://localhost:8000/#/settlements?token=' + usr.token + '&user=' + usr.email
			});
			res.end();
		});
	})(req,res,next);
};
</code></pre>
<p>
Much like in the previous function, we get our passport instance and call our authenticate function, only this time we have our callback function do something. First we check for any errors that might have occurred. If not, let's make sure a user was returned. If there wasn't, lets just redirect back to whatever login page they came from. But if we did, it's time to check if this user exists in our database. If they do exist, redirect the user to the relevant page with their user name and token in the url query string.
</p>
<p>
Your angular client can simply strip the token out of the query string and store it in a cookie. Whenever you want to request information from the API server, you send this cookie to the server in whatever means you may desire, but most likely as either a query string or a header parameter. I shouldn't have to tell you, but it would be sensible to use https for these transactions.
</p>
<p>
Now that you are able to store a token, and return it to the user, you can use this secure your endpoints. As an example;
</p>
<pre><code>
function handleGetUserRequest(req,res) {
	user.findOne().where('token').equals(req.query.token).exec(function( err, user) {
		console.log(err);
		console.log(user);
		if(err) {
			console.log(err);
			return res.send(500,err);
		}
		if(!user){
			return res.send(401,"User Not Authenticated");
		}
		if(user) {
			return user;
		}
	});
};
</code></pre>
<p>
Here we check to ensure that the sent token matches a particular resource, and then return the relevant details back.
</p>
<p>
This should hopefully give you a good idea about how to deal with the oauth2 passport mechanism when dealing with an API server and AngularJS.
</p>
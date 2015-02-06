This is a modification to the MEAN stack generated by the [Yeoman](http://yeoman.io/) [MEANjs generator](https://github.com/meanjs/generator-meanjs) version 0.1.12 which was provided by [MEANjs.org](http://meanjs.org/). This modification adds local token authentication capabilities as an alternative to local cookie based authentication using sessions. 

## Welcome
Welcome to Whitehall Technologies LLC's open source project adding token based local authentication to the base MEANjs.org deployment of the MEAN.js stack. Please visit our [site](http://www.whitehall.io) for more info and follow us on [Twitter](https://twitter.com/castlewhitehall) and [GitHub](https://github.com/castlewhitehall) for live updates.

## Token Authentication
Token authentication is an alternative to cookie based authentication and is has a number of benefits.

* Cross-domain / CORS: cookies + CORS don't play well across different domains. A token-based approach allows you to make AJAX calls to any server, on any domain because you use an HTTP header to transmit the user information.

* Stateless (a.k.a. Server side scalability): there is no need to keep a session store, the token is a self-contanined entity that conveys all the user information. The rest of the state lives in cookies or local storage on the client side.

* CDN: you can serve all the assets of your app from a CDN (e.g. javascript, HTML, images, etc.), and your server side is just the API.

* Decoupling: you are not tied to a particular authentication scheme. The token might be generated anywhere, hence your API can be called from anywhere with a single way of authenticating those calls.

* Mobile ready: when you start working on a native platform (iOS, Android, Windows 8, etc.) cookies are not ideal when consuming a secure API (you have to deal with cookie containers). Adopting a token-based approach simplifies this a lot.

(Taken from Alberto Pose's [article](https://auth0.com/blog/2014/01/07/angularjs-authentication-with-cookies-vs-token/))

## Build Instructions

This guide assumes you have a MongoDB running locally on port 27017. If not, follow these [instructions](http://docs.mongodb.org/manual/tutorial/install-mongodb-on-ubuntu/).

* Update the local package list and download Git and Node.js 

```bash
sudo apt-get update
sudo apt-get install -y git
sudo apt-get install -y nodejs
```

* Create a symbolic link between nodejs and node (some older programs that use node call it through 'node' instead of 'nodejs')

```bash
sudo ln -s /usr/bin/nodejs /usr/bin/node
```

* Install Grunt Task Runner

```bash
npm install -g grunt-cli
```

* Clone the project and navigate into it
```bash
git clone https://github.com/castlewhitehall/meanjs-with-token-auth.git
cd meanjs-with-token-auth/
```

* Install Node packages
```bash
npm install
```

## Deployment and Use
* Run the following to deploy the server using nodemon. If files are changed, then it will automatically restart.
```bash
grunt
```

* tests
```
grunt test
```

## Additions and Modifications

* Include jwt-simple where needed
```javascript
var jwt = require('jwt-simple');
var secret = 'keepitquiet';
```


* Modified User model to include tokens (user.server.model.js)
```javascript
var UserSchema = new Schema({

  ...

	/* For user login */
	loginToken: {
		type: String
	},
	loginExpires: {
		type: Date
	},

  ...

});
```

* Created a new Passport Strategy for local token-based registration(local.js)
```javascript
passport.use('local-token', new LocalStrategy({
	usernameField: 'username',
	passwordField: 'password'	
},
function(username, password, done) {

	User.findOne({
		username: username
	}, function(err, user) {
		if (err) {
			console.log(err);
			return done(err);
		}
		if (!user) {
			return done(null, false, {
				message: 'Unknown user or invalid password'
			});
		}
		if (!user.authenticate(password)) {
			return done(null, false, {
				message: 'Unknown user or invalid password'
			});
		}

		// token expire time
		var expireTime = Date.now() + (2 * 60 * 60 * 1000); // 2 hours from now

		// generate login token
		var tokenPayload = { 
			username: user.username, 
			loginExpires: expireTime
		};

		var loginToken = jwt.encode(tokenPayload, secret);

		// add token and exp date to user object
		user.loginToken = loginToken;
		user.loginExpires = expireTime;

		// save user object to update database
		user.save(function(err) {
			if(err){
				done(err);
			} else {
				done(null, user);
			}
		});
	});
}
));
```

* Created a new middlewear that requires a login token (users.authorization.server.controller.js)
```javascript
/**
 * Require login token routing middleware
 */
exports.requiresLoginToken = function(req, res, next) {
	// check for login token here
	var loginToken = req.body.loginToken;

	// query DB for the user corresponding to the token and act accordingly
	User.findOne({
		loginToken: loginToken,
		loginExpires: {
			$gt: Date.now()
		}
	}, function(err, user){
		if(!user){
			return res.status(401).send({
				message: 'Token is incorrect or has expired. Please login again'
			});
		}
		if(err){
			return res.status(500).send({
				message: 'There was an internal server error processing your login token'
			});
		}

		// bind user object to request and continue
		req.user = user;
		next();
	});
};
```

* Modified the example Article CRUD routes to use the new token middlewear (articles.server.routes.js)
```javascript
module.exports = function(app) {
	// Article Routes
	app.route('/articles')
		.get(articles.list)
		.post(users.requiresLoginToken, articles.create); // authenticate using new requiresLoginToken middlewear

	app.route('/articles/:articleId')
		.get(articles.read)
		.put(users.requiresLoginToken, articles.hasAuthorization, articles.update)		// authenticate using new requiresLoginToken middlewear
		.delete(users.requiresLoginToken, articles.hasAuthorization, articles.delete);	// authenticate using new requiresLoginToken middlewear

	// Finish by binding the article middleware
	app.param('articleId', articles.articleByID);
};
```


## Credits
Inspired by and built upon the great work of [MEANjs.org](http://meanjs.org/)
The MEAN name was coined by [Valeri Karpov](http://blog.mongodb.org/post/49262866911/the-mean-stack-mongodb-expressjs-angularjs-and)

## License
(The MIT License)

Permission is hereby granted, free of charge, to any person obtaining
a copy of this software and associated documentation files (the
'Software'), to deal in the Software without restriction, including
without limitation the rights to use, copy, modify, merge, publish,
distribute, sublicense, and/or sell copies of the Software, and to
permit persons to whom the Software is furnished to do so, subject to
the following conditions:

The above copyright notice and this permission notice shall be
included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED 'AS IS', WITHOUT WARRANTY OF ANY KIND,
EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF
MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT.
IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY
CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT,
TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE
SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.

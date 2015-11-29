title: Build a gnarly todolist with Meteor.js
date: 2015-10-26 16:50:15
tags: [Meteor, Todo]
---

So I've gone through how to build a barebones, functional, fullstack application with meteor in [How to build an app using Meteor.js](http://connorleech.ghost.io/how-to-build-an-app-using-meteor-js/). That said, it is not impressive. Last week I went through Eventedmind's excellent course on how to [Build a multi page app with iron meteor](https://www.eventedmind.com/classes/build-a-multi-page-app-with-iron-meteor-6737880d). It is a bit like how to build a barebones blog, complete with authentication.

<!-- more -->
The [DEMO](http://js-meteor-todos.meteor.com/) is here. I used meteor's built in deployment. I had [some trouble](https://github.com/iron-meteor/iron-cli/issues/153) deploying the app to heroku. Suggestions welcome!


Rather than copying the eventedmind tutorial I'm going to point out some cool pieces of the [CODE](https://github.com/jasonshark/meteor-todos).

### First off
![imma let you finish](http://media.giphy.com/media/4P1nnTxsFtsje/giphy.gif)

The app is structured using the Iron command line tool. Iron is like yeoman but for meteor apps. If that's not helpful ignore it. To handle routing with meteor you will most likely use Iron Router. This is iron scaffolding. It's an opinionated, structured way to build your meteor apps. To build an empty skeleton you go like:

```
$ npm install -g iron-meteor
$ git clone <repo_url>
$ cd meteor-todos
$ iron
```

Next we'll add helpful packages and remove some crappy ones.

```
$ iron add less
$ iron add bootstrap
$ iron add accounts-ui
$ iron add accounts-meteor-developer
$ iron add momentjs:moment
$ iron remove autopublish
$ iron remove insecure
```

The accounts-ui and accounts-meteor-developer abstract away all authentication headache. There are also accounts-facebook, accounts-google, accounts-twitter packages. The last two come by default and we want to turn them off. Insecure allows you to write to the database from the console (don't want that) and autopublish makes all data accessible. The code from the tutorial uses best practices.

### Structure

All the app's code is in the [app](https://github.com/jasonshark/meteor-todos/tree/master/app) folder. This is exactly the code that'd be generated by running `meteor create my_app_name`. If you want to run any `meteor` commands cd into the app folder and you're good to go. (That's exactly how I deployed [`$ cd app $ meteor deploy`]).

The main folders within app to worry about are `client`, `lib` and `server`. As specified in the [documentation](http://docs.meteor.com/#/basic/filestructure) client only runs on client, server only on server and lib runs on both.


### lib

I'll start with lib cause that's where our routes are defined. They look like: 

```
Router.configure({
  layoutTemplate: 'MasterLayout',
  loadingTemplate: 'Loading',
  notFoundTemplate: 'NotFound'
});

Router.route('/', {
  name: 'home',
  controller: 'HomeController',
  action: 'action',
  where: 'client'
});

Router.route('/todos/:_id', {
	name: 'todos.detail',
	controller: 'TodosController',
	action: 'detail',
	where: 'client'
});

Router.route('/todos/:_id/edit', {
	name: 'todos.edit',
	controller: 'TodosController',
	action: 'edit',
	where: 'client'
});

Router.route('/users/:_id', {
  name: 'users.detail',
  controller: 'UsersController',
  action: 'detail',
  where: 'client'
});
```

In my previous post on meteor we did not use controllers, now we do. The controller looks like this:

```
TodosController = RouteController.extend({
  subscriptions: function () {
    this.subscribe('todoDetail', this.params._id);
  },

  // set data context for controller
  data: function () {
    return Todos.findOne({_id: this.params._id});
  },

  detail: function(){
    this.render('TodosDetail', {});
  },

  edit: function(){
    // reactive state variable saying we're in edit mode
    this.state.set('isEditing', true);
    
    this.render('TodosDetail');
  }
});
```

And finally we have a collection in `app/collections/todos.js`:

```
Todos = new Mongo.Collection('todos');

// if server define security rules
// server code and code inside methods are not affected by allow and deny
// these rules only apply when insert, update, and remove are called from untrusted client code

if (Meteor.isServer) {
  // first argument is id of logged in user. (null if not logged in)
  Todos.allow({
    // can do anythin if you own the document
    insert: function (userId, doc) {
      return userId === doc.userId;
    },

    update: function (userId, doc, fieldNames, modifier) {
      return userId === doc.userId;
    },

    remove: function (userId, doc) {
      return userId === doc.userId;
    }
  });

  // The deny method lets you selectively override your allow rules
  // every deny callback must return false for the database change to happen
  Todos.deny({
    insert: function (userId, doc) {
      return false;
    },

    update: function (userId, doc, fieldNames, modifier) {
      return false;
    },

    remove: function (userId, doc) {
      return false;
    }
  });
}
```

That's kind of a lot of code, but for what it does it aint.

![](https://40.media.tumblr.com/f0c2c2079b4978c2b99a46dc5f2edbc5/tumblr_mi3cxcm9w21rj4nx4o1_540.jpg)

The todo collection creates a MongoDB collection in the database that is accessible in the global `Todos` variable on the client and the server. The allow and deny callbacks handle our database permissions and security.


### Server

The coolest thing on the server is the [code](https://github.com/jasonshark/meteor-todos/blob/master/app/server/publish.js) that publishes out our data:

```
/**
 * Meteor.publish('items', function (param1, param2) {
 *  this.ready();
 * });
 */

var allUsersCursor = Meteor.users.find({}, { fields: { profile: 1 }});
var getCursorForUser = function(id){
	return Meteor.users.find({_id: id}, {fields: { profile: 1 }})
};

Meteor.publish('todos', function () {
	// no data published if you're not logged in
	if(!this.userId) return this.ready();

	// only allow people to see their own todos
	// this is currently logged in user
	return Todos.find({userId: this.userId});
});

Meteor.publish('todoDetail', function (id) {
	if(!this.userId) return this.ready();

	var todo = Todos.findOne({ _id:id });
	// get cursors for user who owns todo and todo itself
	// fields specifies what to make available
	return [
		getCursorForUser(todo.userId),
		Todos.find({_id: id}),
		Comments.find({todoId: id}, { sort: {createdAt: 1}})
	];
});

Meteor.publish('users', function (/* args */) {
	if(!this.userId) return this.ready();

	// publish all users but specify which fields to make available
	return allUsersCursor;
});

Meteor.publish('user', function (userId) {
	if(!this.userId) return this.ready();

	// publish user data and their todos
  return [
  	getCursorForUser(userId),
  	Todos.find({userId: userId})
  ];
});
```

In meteor you specify on the server what data to make accessible via publish functions. On the client you subscribe to that data. We handle our subscriptions in the controller. So scroll up and check out the Controller code. You'll see:

```
 subscriptions: function () {
    this.subscribe('todoDetail', this.params._id);
  },

  // set data context for controller
  data: function () {
    return Todos.findOne({_id: this.params._id});
  },
```

So we publish todoDetail on the server and then subscribe to it in the controller. This separates where data is accessible in your app. The data value is what specifically is available in your templates. Check out [this guide to data contexts](https://www.discovermeteor.com/blog/a-guide-to-meteor-templates-data-contexts/). The difference between data contexts and subscriptions confused me for a bit. We'll see how it works on the...


### client

We genereate templates using `$ iron g:template todos/todos_list` and commands like that. You can [see](https://github.com/jasonshark/meteor-todos/tree/master/app/client/templates/todos) the app is way broken up and every html file has a javascript file associated with it. This is where data context comes in. Each template has a name and associated `helpers` and `events` functions. Helpers essentially use the data context to show the data. Events are where you handle form submission, clicks, hovers etc using basically straight up jQuery. No directives or anything. Here's the code for showing a todos detail and being able to edit it:

**app/client/templates/todos/todos\_detail/todos_detail.js**
```
Template.TodosDetail.events({
	// submit edit todo form
	'submit form.edit-todo': function(e, tmpl){
		e.preventDefault();

		var subject = tmpl.find('input[name=subject]').value;
		var description = tmpl.find('[name=description]').value;
		var id = this._id;

		Todos.update({_id: id}, {
			$set: {
				subject: subject,
				description: description,
				updatedAt: new Date
			}
		});

		// reroute and pass data context
		Router.go('todos.detail', {_id: id})
	}
});

Template.TodosDetail.helpers({
	isMyTodo: function(){
		return this.userId === Meteor.userId();
	},
	todoOwner: function(){
		// data context is the todo
		var todo = this;
		return Meteor.users.findOne({_id: todo.userId});
	}
});
```

So specify events for the template with jquery selectors as the values, like `submit form.edit-todo` then a function that gets the template and event as arguments. We write to the database using the global `Todos` collection. `update()` and `$set` are from MongoDB. Meteor uses mongodb so get familiar. The reason we can do this update is because we allow it when we define the todos collection. In the [todos controller](https://github.com/jasonshark/meteor-todos/blob/master/app/lib/controllers/todos_controller.js) (shown in the lib directory) we set the `data` value to be the specific todo whose id is a param from the url bar.

todos controller sets the data context:
```
data: function () {
    return Todos.findOne({_id: this.params._id});
  },
```

So in our template helper `this` is the data context that we specified in the controller. It is kind of crazy and easy to get confused. That's why we give it a variable name. We be humans, not machines.

The code I'm talking about where we use the data context from the controller:
```
	todoOwner: function(){
		// data context is the todo
		var todo = this;
		return Meteor.users.findOne({_id: todo.userId});
	}
```


Have a look through the code. I recommend the eventedmind screencast, that's how I built this thang. If you have more questions hit me up on [twitter](https://twitter.com/cleechtech).

![](/content/images/2015/05/good_luck.gif)


### Other helpful bits

**Authentication**

`{% raw %}{{> loginButons }}{% endraw %}` in the template helps to get auth configured.

Programmatically [configure](http://docs.meteor.com/#/full/meteor_loginwithexternalservice) OAuth provider: `$ iron add service-configuration`

**Startup**

`server/bootstrap.js` is where any code goes that needs to run when we startup the server. Access environment variables from `config/development/env.sh` by using `process.env['variable_name']`

**Database**

Activate a mongo shell for the current database with `$ iron mongo`. What user records look like in database:

```
$ meteor mongo
MongoDB shell version: 2.6.7
connecting to: 127.0.0.1:3001/meteor

meteor:PRIMARY> show collections
	meteor_accounts_loginServiceConfiguration
	meteor_oauth_pendingCredentials
	system.indexes
	users
meteor:PRIMARY> db.meteor_accounts_loginServiceConfiguration.find().pretty()
	{
		"_id" : "MJRdreKXQ8NCPnhH7",
		"service" : "meteor-developer",
		"clientId" : "Dyv7PX6SWqQXhLCxZ",
		"secret" : "hDePgZTKMMw6ksGKbGj3gC75TwXRpkhECh",
		"loginStyle" : "popup"
	}
```

The above is what meteor uses to register our users. The below is what user records look like.

```
meteor:PRIMARY> db.users.find().pretty()
	{
		"_id" : "kj8iq7dpzbMGseDPy",
		"createdAt" : ISODate("2015-05-21T21:59:46.840Z"),
		"services" : {
			"meteor-developer" : {
				"accessToken" : "EzYgczXRvYRR6AyMG",
				"expiresAt" : NaN,
				"username" : "connorleech",
				"emails" : [
					{
						"address" : "connorleech@gmail.com",
						"primary" : true,
						"verified" : true
					}
				],
				"id" : "zkS6ewa9nyYcPAgah"
			},
			"resume" : {
				"loginTokens" : [ ]
			}
		},
		"profile" : {
			"name" : "connorleech"
		}
	}
```

Access users in the browser console by typing: `Meteor.users.find().fetch()`. 

`.find()` returns a cursor. `.fetch()` turns the cursor into an array


security rules on `lib/collections/todos.js` prevent unauthorized writes to database.

**iron commands**

Add a publish function: `$ iron g:publish todos`
Create collection in mongo database: `$ iron g:collection todos`

Subscribe to publications within your controller.


### Heroku

I tried to deploy with this:

```
$ heroku login
$ heroku git:remote -a <name-of-heroku-app>
$ heroku config:set BUILDPACK_URL=https://github.com/lirbank/meteor-buildpack-horse.git
$ heroku config:set ROOT_URL=https://<yourapp>.herokuapp.com
$ git push heroku master
$ heroku open
```

I think something was wrong with my environment variables. The provided meteor deployment platform works for me for now.


Hope this is helpful! yeah I'm on [twitter](https://twitter.com/cleechtech)
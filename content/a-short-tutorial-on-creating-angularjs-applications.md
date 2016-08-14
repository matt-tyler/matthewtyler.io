+++
date = "2013-05-17"
title = "A short tutorial on creating AngularJS Applications"
+++
AngularJS is fast proving itself to be the new-hotness when it comes to javascript frameworks. I've been using it over the last few months and I must say, I've never had so much fun doing web stuff. In this post, I'm going to take you over building a small application. We're going to fudge some stuff in this post to get a general application going, and then, over the next few posts, we're break down the application into a more modular layout. We'll also cover how to test the application with angularjs tests. I'm going to avoid talking about the backend, and restrict myself to talking about AngularJS.

The application to build is a simple. We want to display a table with the first row being a list of names. The rest of the columns will represent dates. If we click in the date column, it should change colours (ie to show whether the person was present, sick, or not present). Basically, it's a simple attendance register. 

You can see a video of it below -

<iframe width="560" height="315" src="http://www.youtube.com/embed/o7Ugp-kauWc" frameborder="0" allowfullscreen></iframe>

You can get the code for the full application at it's github repo <a href="https://github.com/matt-tyler/fieldtracker">https://github.com/matt-tyler/fieldtracker</a>

Firstly; some points

<li>The backend server will be written in nodejs - you can grab this along with the rest of the code at the above link.</li>
<li>We will use the excellent skeleton project angular-express-seed which you can get at <a href="https://github.com/btford/angular-express-seed ">https://github.com/btford/angular-express-seed</a></li>
<li>Our backend database will be provided by MongoDB</li>
<li>Our basic data structure is list of json objects consisting of a name, and a history field containing the particular week and the status of the week. This is not an array but a flat object.
<pre><code>
{
	name: "some name",
	history: {
		1: "sufficient",
		2: "insufficient",
		...
	}
}
</code></pre>
</li>

Although AngularJS recommends it's $resource service for interacting with restful resources, we will use the Restangular library, which is a fair bit simpler to use. Restangular is available from <a href="https://github.com/mgonto/restangular">https://github.com/mgonto/restangular</a>

Our API consists of few commands

'/api/records' to return all records

'/api/name/:name/week/:week/status/:status' which updates a particular records' week's status

'/api/page/:pagenum' to save what page we are currently looking at

'/api/page' to get whatever the last page was saved

The latter two API requests are just to ensure that when reload the page, we are returned to the last page that was looked at. I will point out that we have no way of identifying who was looking at the page, so if somebody else has loaded the page since we last checked it, whatever they were looking at last is what will be bought up. You could obviously extend the functionality to include this case, but I have left it out.

Lets hop into the code 

<pre><code>
extends layout

block body
  .navbar.navbar-fixed-top
    .navbar-inner
      .container-fluid
        a.brand(href='#') Field Tracker
        ul.nav
  .container-fluid
    div(ng-view)

  script(src='js/lib/angular/angular.js')
  script(src='js/lib/angular/angular-resource.js')
  script(src='js/app.js')
  script(src='js/services.js')
  script(src='js/controllers.js')
  script(src='js/filters.js')
  script(src='js/directives.js')
  script(src='js/lib/jquery/jquery-1.9.1.min.js')
  script(src='bootstrap/js/bootstrap.min.js')
  script(src='js/lib/restangular/restangular.js')
  script(src='js/lib/underscore/underscore.js')
</code></pre>

Our index page is extremely simple - we define our navigation bar (not really needed in this example) and define an element called ng-view. Ng-view acts a container from which views are populated in (more on where those come from later). At the end of the file, we define the scripts that angular needs to operate. The 'extends layout' line is a jade command, which is including a layout file which is including the content in our <head> tag; namely our title, meta data, css etc. 

The user defined files of that lot, are the app.js, controller.js, service.js, and filter.js files. app.js supplies routing logic, and associates controllers with views. This file allows us to watch the urls requested in the browser bar, and inject the request in the ng-view directive (btw, you can only have one ng-view directive in your application).  Our controller.js file contains our controllers, which generally associate our models with data sources and provide them to the template. Service.js provides, funnily enough, services that our controllers may use - say you wanted to share data between controllers, you might do that with a service. Finally filters by and large are concerned with formatting and sorting data.

Our app.js file contains the following lines;

<pre><code>
// Declare app level module which depends on filters, and services
angular.module('myApp', ['myApp.filters', 'myApp.services', 'myApp.directives','restangular']).
  config(['$routeProvider', '$locationProvider', function($routeProvider, $locationProvider) {
    $routeProvider.when('/view1', {templateUrl: 'partial/1', controller: fieldCtrl});
    $routeProvider.otherwise({redirectTo: '/view1'});
    $locationProvider.html5Mode(true);
  }]);
</code></pre>

It's easy enough to guess what this is doing; we define the urls we are iterested in, and it routes the requests to the appropriate view and controller. You can define what controller to use directly in your view templates, but generally I prefer to do it this way. Controllers can be nested though, so you are not restricted to just one controller in one template. Pay particular attention to the arguments in angular.module. Our first one defines the name of our application, in this case'myApp'. For an angular application to work it must have an ng-app='MyAppName' attribute/element someone on the page that denotes the root of the application. This is used to auto-bootstrap the application. You can do this bootstrapping manually, but it's outside the scope of this post.

The second argument is an array of dependencies. The MyApp depends on all these items, which themselves are angular modules - which likely have their own dependencies. AngularJS wants you to package your components as organised modules, so do what it says.

Our controller.js file is where things begin to get interesting.

<pre><code>
function fieldCtrl($scope,Restangular) {

  var states = ['sufficient','insufficient','absent']

  $scope.changeState = function(index,week) {

    var state = states.indexOf($scope.records[index]["history"][week]);

    if(state + 1  < states.length) {
      $scope.records[index]["history"][week] = states[state+1];
    }
    else {
      $scope.records[index]["history"][week] = states[0];
    }
    var name = $scope.records[index].name
    var status = $scope.records[index]["history"][week];

    Restangular.one("api/name",name).one("week",week).one("status",status).put();
  }

  Restangular.all('api/records').getList().then(function (accounts) {
      $scope.records = accounts;
  });

  $scope.range = [];
  for(var i = 1; i < 54; i++){
    $scope.range.push(i);
  }

  Restangular.one('api/page','').get().then(function (response) {
    $scope.from = parseInt(response.page);
    $scope.listnum = 15;
  });

  $scope.nextPage = function() {
    $scope.from = $scope.from + 15;
    if($scope.from > 38) {
      $scope.from = 38;
    }
    Restangular.one('api/page',$scope.from).put();
  
  $scope.prevPage = function() {
    $scope.from = $scope.from - 15;
    if($scope.from < 0) {
      $scope.from = 0;
    }
    Restangular.one('api/page',$scope.from).put();
  };
};
</code></pre>

Ok, so earlier we defined a route which redirected to a view and a corresponding controller. This is that controller and you'll notice it has two arguments, scope and restangular. Firstly, the latter is a reference to the restangular service - which is a simple library for performing REST commands. By providing it to the controller, we can use it through the wonders of Angular's dependency injection system. The first argument, scope, is a big top of AngularJS and one you wil become use to seeing as you create Angular applications. The scope, essentially, acts a big pit of data which provides two way data binding in the application. Once I define a variable in the scope, I can reference it in the view and have it automatically update values in the view as they change. The reverse is true as well, if I change something in a text box on the view that is binded to a scope variable, the scope variable will update as well. This simple two way data binding is the best part of AngularJS.

In our controller;

- We define some states which indicate what colour the entry should be.
- We define a method changeState() - which defines what state we change to upon a click. In our case, we cycle through to the next state and wrap around when we reach the last state. We also make a call to our REST Api to update the database.
- We define to other functions, nextPage() and prevPage() which provide some simple paging for the table. We use this to determine what section of our attendence register that we want to display.
- We have another call to Restangular.all(), which we use to pull all the records from the database on initialisation.
- Defining a range variable, which determines the number of entries in our table

Now lets take a look at the view we marry to the controller

<pre><code>

.row-fluid
  .span12
    table
      thead
        tr
          th(class='invisible')
          th(offset='1',colspan='2',ng-click='prevPage()')
            button(width='100%') <<<
          th(colspan='11') Field Time Tracking
          th(colspan='2', ng-click='nextPage()')
            button(width='100%') >>>
        tr
          th Name
          th(ng-repeat="week in range", ng-show="$index >= from && $index < (from + listnum )",width='25px') {{week}}
      tbody
        tr(ng-repeat="record in records")
          td(width='150px') {{record.name}}
          td(ng-repeat="week in range", ng-show="$index >= from && $index < (from+listnum)",ng-class="record.history[week]",ng-click="changeState($parent.$index,week)")

</code></pre>

Now, there are a few things to notice here. The first is the ng-click='...' attributes. These angular directives fire off the functions that they defined as on a mouse click, similar to the onlick events you are no doubt used to if you're familiar with javascript. Then there is ng-show. This helpful little directive will show the element it is defined on if and only if the expression given to it evaluates as true. In this case, whether the $index is between the values that we are defined in our scope back in the controller. Our prevPage, and nextPage functions, move the window of visible entries as they are clicked. But where does $index come from?

The final interesting bit is the ng-repeat directive, which is probably one of the most used angular directives. Given a '... in ...'s ' statement, it will iterate through the list, spitting out repeats for each tag in the array, list etc. In our case, we are generating every column, and every row that we require. The $index variable is a special variable that is provided from the scope of the ng-repeat directive, which lets us to determine the index of the generated tag. Through this, we can determine whether we should be showing the current field (through ng-show, remember?).

A note using binding in the view; AngularJS will automatically parse expressions into their values if they placed inside tags. However, when placed outside they require double curly braces {{}} to tell angular to resolve the expression.

That's all for now. If you'd like to learn more about AngularJS, egghead.io is fantastic source of videos on the framework. Otherwise, the AngularJS book from O'Reilly Media is also a great source of information.

In the next post, I'll introduce the best aspect of AngularJS, custom directives.
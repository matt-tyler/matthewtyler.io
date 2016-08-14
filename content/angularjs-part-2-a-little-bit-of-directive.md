+++
date = "2013-09-03"
title = "Angular.js Part 2: A little bit of directive"
+++

Hi guys,

I've finally got around to my next post in my series of angular posts.

In this instance, we're going to convert the fieldtracker app we made last week into a self-contained directive. An directive (in the angularjs world) is a way of defining a self-contained component that is ideally reusable. Now, what I'm going to show isn't going to be the best way of doing things - but it will get you moving in the right direction to using directives. There will be some follow up resources at the end of the post.

As a note, directives are also the preferred way of doing DOM manipulation, so if you need to use jquery for anything its preferred that you do in this in the directive, and not in the controller. The reason for this is simple - it's not guaranteed the DOM has been prepared when the controller code runs, so controller based DOM manipulation may fail.

I'll get the code up on github when I have a moment.

edit: Had a moment - code is here <a href="https://github.com/matt-tyler/fieldtracker/tree/fieldtracker-directive">https://github.com/matt-tyler/fieldtracker/tree/fieldtracker-directive</a>

The first thing you need to do is create a directive.js file that will hold our directive code, and then include it in script tags in your index. If you were following last time, I have

index.jade

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

and the corresponding layout.jade file:

<pre><code>
html(ng-app="myApp")
  head
    meta(charset='utf8')
    base(href='/')
    title Fieldtracker-Demo
    link(rel='stylesheet', href='/bootstrap/css/bootstrap.min.css')
    link(rel='stylesheet', href='/bootstrap/css/bootstrap-responsive.min.css')
    link(rel='stylesheet', href='/css/app.css')
  body
    block body
</code></pre>

Simple enough. Let's move onto our directive.js file. First, we need to define a module. An angular module contains a set of dependent components, be they directives, filters, services etc. You might remember that we defined a module in the previous tutorial 'myApp' which we used with ng-app (check back up there in the layout.jade file, you'll see it) to bootstrap the angular application and provide the app dependencies. Indeed, our app.js file looks this-

<pre><code>
angular.module('myApp', ['myApp.filters', 'myApp.services', 'myApp.directives','restangular']).
  config(['$routeProvider', '$locationProvider', function($routeProvider, $locationProvider) {
    $routeProvider.when('/view1', {templateUrl: 'partial/1'});
    $routeProvider.otherwise({redirectTo: '/view1'});
    $locationProvider.html5Mode(true);
  }]);
</code></pre>

Here, we can see we are defining a module myApp, which depends on myApp.* - which themselves are all modules. It helps to think of an angular application as a set of component modules, which we provide to other modules ad nauseum until they eventually hit our main app module.

But alas, I've gone off the topic of directives.

Let's define our directives module as such;

<pre><code>
var fieldtrackerDirective = angular.module('myApp.directives', []).
  directive('appVersion', ['version', function(version) {
    return function(scope, elm, attrs) {
      elm.text(version);
    };
  }]);
</code></pre>

Here we define a module 'myApp.directives'. The empty square brackets indicate that we have no dependencies. As an aside, if we gave a dependency that did not exist in our project because we have not included it, angularjs would report a dependency not found error to the console. Feel free to try it out!

We have define one directive inline with our module definition called appVersion - this simply depends on the version api provided in angularjs to return version information about angularjs, which it then uses to return inside the text element of the directive. ie, if somewhere on my page I did 

<pre><code>
span(app version)
</code></pre>

I would expect it to print the angularjs version information.

Our fieldtracker is going to be relatively simple. Lets look at the directive definition

<pre><code>
fieldtrackerDirective.directive('fieldtracker',function() {
    return {
      restrict: 'EA',
      replace: true,
      controller: 'fieldtrackerCtrl',
      scope: {},
      templateUrl: 'partial/3',
      compile: function(tElement,tAttrs){
        return function link(scope,iElement,iAttrs,ctrl) {
        }
      }
    }
  })
</code></pre>

We define the directive much like we did previously with appVersion. This time though, our return statement is a little more complex. It should be noted that there are a boatload of ways to define directives, and it's likely as you look over different angular projects you'll see a lot of different methods. In this instance, we a directly returning what angularjs calls a 'directive definition object' which is essentially the DNA of our directive. We return an object that consists of several different fields.

The following is information on the ones we can see in this particular directive.

restrict - this determines how declare our directive when used in our views. There are 4 main types, element, attribute, class, and comment. The latter one is rarely used. As an example of each type is declared as thus-

<pre><code>
E = &lt;fieldtracker&gt;&lt;/fieldtracker&gt;&lt;fieldtracker&gt;&lt;/fieldtracker&gt;
A = &lt;div fieldtracker&gt;&lt;/div&gt;
C = &lt;div class=&#39;fieldtracker&#39;&gt;&lt;/div&gt;
M = &lt;!-- directive: fieldtracker exp --&gt;
</code></pre>

You can choose to 'restrict' you directive to only work when declared as any or all of them. What you pick will depend on personal circumstance. I have noted that specific kinds of DOM manipulation will complain if you do not have a root element. In this instance, you can get around this by using class and attribute type directives.

replace - This determines whether the output of the directive replaces the the tag in which it was defined, or gets set as an element underneath it

example:
if false -
<pre><code>
	&lt;div some-directive&gt; 
		&lt;!-- some directive contents --&gt;
	&lt;/div&gt;
</pre>
if true - 
<pre>
	&lt;!-- some directive contents --&gt;
</code></pre>

controller - This is a way of passing in a controller for the directive to use. If methods are defined on the controller, they can be called from the linking function. Likewise, the scope of the controller will act on the scope of the directive. You can of course, do what you would in a controller in the linking function of the directive (in fact, many do), but this is perhaps a better way of decoupling the listeners from the view. You could also share the controller logic with multiple directives, if neccesary.

scope - This is where things can get a bit tricky. If you do not include this variable, the scope of the directive will be the same as it's parent scope (ie the scope of wherever you defined the directive). Otherwise, the as soon as it's declared the behaviour will be to create for the directive it's own scope. This directive scope is a child of the scope on which the directive was defined. Generally, I'd consider it good practice to declare a scope for a directive, as this decouples it from it's parent scope and it promotes component reuse. You can however, still access the parent scope by scope.parent.

There are a few other points about scope that need to be mentioned. By adding variables to the scope, you declare that you expect these to be passed in as attributes to the directive. Imagine we had-

<pre><code>
	&lt;div some-directive field1=&#39;something&#39; field2=&#39;something&#39; field3=&#39;something&#39;&gt;
	&lt;/div&gt;
</code></pre>

we would then expect something like the following definition in directive definition object:

<pre><code>
	scope: {
		field1: '=',
		field2: '@',
		field3: '&'
	}
</code></pre>

The values of those fields are significant.

- = - implies double binding. If the value changes in the directive or from where it was passed in either, it will be reflected to the other variable. So if you had defined some variable on the parent scope and passed it to the directive, if the directive changes it's value at some point that change will be reflected back.

- @ - or 'string' binding essentially passes the string value of the variable to the directive. Note this essentially means there is no two way binding occurring here. The parent and the parent alone is capable of changing the value passed to the directive. Note that unlike the previous type, if you want binding you need to explicitly state it with curly brackets ie '{{field2}}' and not 'field2' as printed in the example, because this will literally assume field2 is the value that you want to pass in.

- & - is expression scope and is used a bit less than the other two. It is a way of passing a function to the directive so that it evaluates on the parent scope. For example, say were building a calculator application and enter a number on in a text box. Clicking a button on the calculator increments the number, but you want this number passed back up to the parent scope - or you want the parent scope to define what function is called when a particular button is pressed. This is a manner in which you could achieve this functionality.

In my fieldtracker directive I haven't used any of these, but I thought it would be a good idea to cover.

templateUrl - This is a way of passing in a template to the directive by (obviously) url. In mine I'm passing a partial which will passed into the element of the directive on compile/link. By doing this, you don't have to explicitly create each DOM element in the compile phase of the directive. You can similarly use the 'template' parameter if you feel like explicitly writing out the template inline with the directive.

Ok now on to the meaty bit of directives, the dreaded compile/link phase.

The first phase, compile, deals with transforming the template DOM. A lot of directives won't and don't need to do this. ngRepeat and ngView are directives that do, if you are looking for examples. 

A compile function is required any time a change in the model results in a change of DOM structure. A good example is using an ngRepeat directive with a list, and the list suddenly loses or gains an item. You expect this change to reflected in the ngRepeat list. 

But, we expect to have a reasonably amount of performance when doing this. Let's imagine we had other directives as child elements of the ngRepeat tag. If we added another element to our list, these would all need to be compiled and relinked again. In ngRepeats case, it prevents the compilation process from descending down into elements, and instead provides a link function, which can copied and provided whenever a new element is added to the list.

Which brings us to the link function. The link function allows use to register listeners on specific DOM elements, as well as copy content from the DOM into the scope. You often find controller-type logic in the linking function.

Don't use this as the definitive guide to the compile/link phase. I highly recommend looking at code samples around the web and reading various articles on it. It's very hard to get a good grasp on this without reading code and experimenting!

One last note, is that compile function always returns either a linking function, or an object with attributes labelled pre and post (which are themselves linking functions). If you do not have a compile function, you can omit it and simply have a 'link' attribute on the directive definition object. Pre and Post, determine at what stage of the linking process you want your code to execute at. Postlinking is the most common, and the main difference between them is that it is unsafe to do DOM manipulation in the prelink function.

In my code, I haven't used anything in the compile/link phase, although I could place the controller code inside the link function and it would still retain it's functionality.

There is only two more things we need to do. First we move our template code to partial/3, as defined in the directive.

<pre><code>
table
  thead
    tr
 style="font-family: Andale Mono, Lucida Console, Monaco, fixed, monospace; color: #000000; background-color: #eee;font-size: 12px;border: 1px dashed #999999;line-height: 14px;padding: 5px; overflow: auto; width: 100%"      th(class='invisible')
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

Then we add our controller code into a seperate controller, as defined in our directive, into a controller defined on the directive module.

<pre><code>
fieldtrackerDirective.controller('fieldtrackerCtrl',function($scope,Restangular) {

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
    };

    $scope.prevPage = function() {
      $scope.from = $scope.from - 15;
      if($scope.from < 0) {
        $scope.from = 0;
      }
      Restangular.one('api/page',$scope.from).put();
    };
});
</code></pre>

Now all we have to do is use the below code and our directive will function just as normal!

<pre><code>
    div(fieldtracker)
</code></pre>

Obviously, there is nothing too 'reusable' about this example. All it does is let me (very easily) put the fieldtracker widget with the same backend anywhere I want it. Even if I placed two on the same page, changing one would not update the other, as each has its own instance of the controller. You could potentially implement shared functionally a few different ways, but essentially you'd need a way to inform other fieldtrackers that what they are displaying needs to change. 

What things could we change to make fieldtracker more flexible? We could
- Give the fieldtracker an attribute so we can define it like <div fieldtracker api="/my/awesome/api"></div> and pass the api attribute to the controller, so we can change what REST endpoint to use

- Replace the controller with a singleton service, so all instances are updated appropriately when a change is made on one of them. This could be done a few different ways; by $watch-ing the variable on the service in the directive and updating the appropriate entry when the variable changes (or refreshing the whole directive, your call!). Perhaps we use a websocket and update the view when the server changes the backend (pub/sub) ?

- Modify the controller slightly so that rather having the functions defined on the scope, they are returned as set of functions on the controller, then setting them up on the scope in the linking function. This could potentially make the controller reusable in another (yet unwritten) directive.

To wrap this up, when writing directives you essentially need to consider -

- What is the template of the directive going to be?
- How am I going to pass data into this directive? by attribute, by service?
- Do I need a controller? and am I going to have sub-directives that are going to need access to this controller?
- What attributes do I need to watch for changes?
- Do I need/want to transclude any content?
- Am I going to need to do any Jquery style DOM manipulation?

Writing directives aren't a whole lot different then writing views and wiring up controllers, so there is not anything to fear.

As always, if you want more information on directives, the following are fantastic sources of information;

<a href="http://eggehead,io" title="Egghead.io">egghead.io</a>
<a href="http://liamkaufman.com/blog/2013/05/13/understanding-angularjs-directives-part1-ng-repeat-and-compile/" title="Understanding Compile/Link Separation with ng-repeat">Understanding Compile/Link Separation with ng-repeat</a>
<a href="http://onehungrymind.com/angularjs-dynamic-templates/" title="One Hungry Mind - Dynamic Templates">One Hungry Mind - Dynamic Templates</a>
<a href="http://docs.angularjs.org/guide/directive" title="AngularJS Official Directive Documentation">AngularJS Official Directive Documentation</a>

A shout out to the AngularUI team - the accordion directive is a good example of a directive of average difficulty.

<a href="https://github.com/angular-ui/bootstrap/blob/master/src/accordion/accordion.js" title="Angular-UI accordion source code">Angular-UI accordion source code</a>


Cheers!
>==**Disclaimer:** In this post I am not trying to preach the best/only way to  build a scalable AngularJS apps. There are many other approaches, this one happens to be the result of hours of research, reading, trial and error and some experience.==

When learning new programming languages or frameworks I love to see practical/real world examples. It makes me better understand the technology and how people are actually using it.
So I will use as case study a web app called [TheFitStep](http://thefitstep.com) recently launched by my company [Atiiv](http://www.atiiv.com).

To give you some context before diving in some code examples, TheFitStep is a  **Single Page Application** (SPA) built with **AngularJS** which allows people to search personal trainers in their area and contact them, on the other hand personal trainers can create a "CV", promote themselves and have available some specific marketing tools.

In this article I'm going to talk about something that unfortunately for some reason is kept in some secrecy, and that is **Application Architecture**. Here is what I am going to cover:

* Folder structure;
* Modules and sub-modules;
* Handling environments.
* Browserify and gulp scripts

So let's get started.

### #1 Planning

Planning should be the first thing to do before write any code. Sure you can't predict everything but with time and experience you get better at it and that's how your application architecture get's better every time. You learn with your mistakes right? Well I hope so.

In TheFitStep we have two main different areas.

**Public area** - where everyone can search for a personal trainer and view their profiles.

**Private area** - where personal trainers need to register and login in order to build their CV.

### #Folder Structure

So we came up with this folder structure:
```
- TheFitStep
    - assets
        - js
        - scss
    - public
        - css
        - js
        - img
        - views
```

#### Breaking down things
Inside the `assets>js` folder we have:

```
- Config
   dev.json
   production.json
   staging.json
- Modules
  - Admin
     - Account
     - Profile
         - Controllers
         - Directives
         - Filters
         index.js
         ProfileRoutes.js
         ProfileRun.js
     // (etc)
  - Authentication
  - Authorization
  - Core
  - Finder
      - Home
      - Profile
      - Search
      - StaticPages
      index.js
  - Registration
  - Services
app-bootstrap.js
app.js
```
Wow so many folders! Maybe yes but each one have only one or two things inside.
Lets break them even further.

The **config** folder simply have .json files with our API URL for the different environments (more on that later).

If we open the **Profile** folder there is a Controllers, Directives and Filters subfolders, these off-course are related only to the profile feature.
We like to keep our controllers as small as possible, so don't get surprised to see a `EditProfilesController.js`, `ListProfilesController.js` and so on. Each one have one and only one job.
This is a bit different from mostly server-side frameworks where in a ProfilesController you could have all this methods (show, store, update etc), in AngularJS a controller gets mudded pretty faster then in Laravel, Rails etc.

**Authentication** and **Authorization** are obvious so won't get in much details.

In the **Core** is where we place everything that is used in both private and public areas. Under it we have again folders for Controllers, Directives, etc.

The **Finder** is the actual public area and more of the same, folders for the features and sub-folders for Controllers and so on.

All the interaction with the API is made through Factories but for the sake of readability we named it **Services**.
So here you find files like: `ProfileService.js`, `AuthenticationService.js`, basically all the app resources.


### #Modules and Submodules

We are using [browserify](http://browserify.org/) to help us decoupling our app dividing it in modules and submodules.
Please understand by modules I'm referring to things like: Admin, Authentication, Authorization, Core, Finder etc, and submodules...well those are basically features.

Note that in every single one of them there is a `index.js`. This file is used to import everything related to the submodule or module.

Confused? Let me show you an example.

Remember the folder structure from before, so here is the `index.js`from the **Admin Module:**

```
'use strict';

require('angular');
require('angular-ui-router');
require('Modules/Admin/Profile/index.js');
require('Modules/Admin/Education/index.js');
require('Modules/Admin/Experience/index.js');
require('Modules/Admin/Certificates/index.js');
require('Modules/Admin/Account/index.js');

module.exports = angular.module('thefitstep.module.administration',
  [
    'ui.router',
    // All the administration submodules
    'thefitstep.module.administration.profile',
    'thefitstep.module.administration.education',
    'thefitstep.module.administration.experience',
    'thefitstep.module.administration.certificates',
    'thefitstep.module.administration.account'
  ]);
```
So using the node.js power that browserify gives to us, we require all the Admin submodules and bind them to the parent module.
This way every time your client asks you for one more feature you just have to import it here.

At Atiiv we choose the following name convention: `<app_name>.module.<module_name>.<submodule_name>` this is entirely up to you, whatever it sounds best.

Now lets take a look at one submodule `index.js` file:

```
'use strict';

require('angular');
require('angular-ui-router');

module.exports = angular.module('thefitstep.module.administration.profile',
  [
    'ui.router',
  ]);

// Routes file
require('Modules/Admin/Profile/Profile_Routes.js');

// Controllers
require('Modules/Admin/Profile/Controllers/EditProfilesController.js');
(...)
```
Can you guess which submodules is this? If you said Profile then you are right.

Each submodule have their routes file, instead of having a huge and extremely confusing general file it is much better to break things down.
Again another example:
```
'use strict';

// gets access to the parent module
var profileModule = require('Modules/Admin/Profile/index.js');

var routes = [
  '$stateProvider',
  '$urlRouterProvider',
  '$locationProvider',
  function($stateProvider, $urlRouterProvider,  $locationProvider){

  $urlRouterProvider.otherwise('/404');
  $stateProvider.state('admin.edit-profile', {
    url: '/profile/edit',
    templateUrl: 'views/admin/edit-profile.tpl.html',
    controller: 'EditProfileController',
    access: {
      requiresLogin: true,
      requiredPermissions: ['trainer'],
      permissionType: 'AtLeastOne'
    },
    resolve: {
      getMyProfile: ['ProfileService', function(ProfileService){
        var success = function(response) { return response.data.data.profile.data; }
        var error = function(response) { return response.data.data; }
        return ProfileService.me().then(success, error);
      }]
    }
  })
  (...)

  $locationProvider.html5Mode(true);
}];


profileModule.config(routes);
```

To make it more clear an example of a controller is a good idea:

```
'use strict';
var profileModule = require('Modules/Admin/Profile/index.js');

var EditProfileController = [
  '$scope',
   // this is injected from the resolve method with the current user Profile
  'getMyProfile',
  'ProfileService',
  'Flash',
  function($scope, getMyProfile, ProfileService, Flash) {
        $scope.profile = {};
      $scope.profile = getMyProfile;


    $scope.editProfile = function(profile) {
          var success = function(response) {
              var message = 'Profile successfully updated';
              Flash.create('success', message);
        };

        var error = function(response) {
              $scope.errors = response.data.error.fields;
        };

    ProfileService.updateProfile(id, profile).then(success, error);
  }

}];

profileModule.controller('EditProfileController', EditProfileController);
```
See there isn't much going on in the controllers. It just get the data from the view and delegates it to the Service.

Finally a peek on the ProfileService:

```
'use strict';
var servicesModule = require('Modules/Services/index.js');

var ProfileService = ['$http', 'API_URL', function($http, API_URL) {

  return {
    me: function() {
      return $http.get(API_URL + '/me?include=profile.country.regions,profile.cities');
    },

    updateProfile: function(id, profile) {
      return $http.put(API_URL + '/users/' + id + '/profile', profile);
    },

   (...)
  };
}];

servicesModule.factory('ProfileService', ProfileService);
```
In the **app-bootstrap.js** all the modules are required keeping it very clean.
```
'use strict';

require('Modules/Core/index.js');
require('Modules/Services/index.js');
require('Modules/Finder/index.js');
require('Modules/Authentication/index.js');
require('Modules/Authorization/index.js');
require('Modules/Registration/index.js');
require('Modules/Admin/index.js');
```

Finally in our main **app.js** file we just do this:

```
'use strict';
require('angular');
require('app-bootstrap.js');

(function() {
  var app = angular.module('thefitstep',
    [
      // All the parent modules
      'thefitstep.module.core',
      'thefitstep.module.services',
      'thefitstep.module.finder',
      'thefitstep.module.authentication',
      'thefitstep.module.authorization',
      'thefitstep.module.registration',
      'thefitstep.module.administration',
    ]);
})();
```

### #Handling Environments
AngularJS doesn't support different environments out of the box, so I wondered if it was actually possible to accomplish this. It turns out that it is, but with the help of [gulp](http://gulpjs.com/) or [grunt](http://gruntjs.com/).
In our app one of the things I wanted to change deppending on the environment was the API URL.
Here is how I solved the problem.

Inside of the `config` folder create some json files with your environments names for example: `dev.json`, `production.js` and `staging.json`.
Here you just have to put your environments variables like so:
```
// dev.json

{
  "API_URL": "http://api.example.app:8000/v1"
}
```
```
// production

{
  "API_URL": "http://api.example.com/v1"
}
```
So far so good. Pretty basic stuff right?

Once you setup your variables the next step is to create a module with constants, where it's values will be placeholders so we can run a gulp script and replace those with the real values deppending on the environment.

```
// constants.js

'use strict';
require('angular');

module.exports = angular.module('thefitstep.module.core.constants', []).constant('API_URL', '@@API_URL');
```

Now in your deploy script you just need to run the following command and *voilà*.

`gulp replace --env <env_name>`

And that's the gulp script that makes all this possible:
```
var gulp      = require('gulp'),
    args      = require('yargs').argv,
    replace     = require('gulp-replace-task'),
    rename      = require('gulp-rename'),
    fs        = require('fs');

gulp.task('replace', function () {

  var env = args.env || 'dev';
  var filename = env + '.json';
  var settings = JSON.parse(fs.readFileSync('./assets/js/config/' + filename, 'utf8'));

  gulp.src('./assets/js/Modules/Core/Constants/constants.js')
    .pipe(replace({
    patterns: [
      {
        match: 'API_URL',
        replacement: settings.API_URL
      }
    ]
  }))
  .pipe(rename('app-constants.js'))
  .pipe(gulp.dest('./assets/js/Modules/Core/Constants'));
});
```
What this will do is check the file with the environment name you passed get all the key, value pairs of the json and match with constant names, creating a new file with those same values.

There is a lot more to cover on this topic but hopefully you can have some idea from where to start.
Divide your app in modules and submodules, keep your controllers small, make sure when possible they only do one job and use directives when you have the same functionality on multiple parts of your application.
These are the basics to build a good and scalable app using AngularJS.

Tweet me [@hfalucas](https://twitter.com/hfalucas), with your opinions/suggestions or any idea you guys have to improve our app architecture as well.


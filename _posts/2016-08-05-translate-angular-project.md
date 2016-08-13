---
layout: post
title: "Translate Angular Project"
date: 2016-08-12 14:02:03
image: '/assets/img/'
description: How to setup language translation on AngularJS project
main-class: 'web'
color:
tags:
- angularjs
- translation
categories: web
twitter_text:
introduction:
---

If you are googling to find how to internalizate your angularjs, you would find `angular-translate` or `angular-gettext` mostly. The first reason I choose `angular-translate` is because this has setting for reading translation data through json file. `angular-gettext` is using .po format and they have own tool to edit, but I don't want to use other specific tool for editing. Because I have to ask for translation to other team, and it is a bugging job to guide how to work on with this. Editing text or json is more simple, and this is why I made this decision.

You can find more details about `angular-translate` on https://angular-translate.github.io

![Screenshot](/assets/post_img/translate_angular_project/main_page.png)

Add library file with bower for setup...

{% highlight shell %}
{% raw %}
bower install angular-translate
{% endraw %}
{% endhighlight %}

and add path and module for this. Module name is `pascalprecht.translate`.

{% highlight html %}
{% raw %}
...
<script src="bower_components/angular-translate/angular-translate.js"></script>
<script>
  var app = angular.module('appName', ['pascalprecht.translate']);
</script>
...
{% endraw %}
{% endhighlight %}

Now go on for simple translation.

{% highlight html %}
{% raw %}
<h1>{{ 'TITLE' | translate }}</h1>
...
<script>
  var app = angular.module('appName', ['pascalprecht.translate']);
  app.config(['$translateProvider', function ($translateProvider) {
    $translateProvider.translations('en', {
      'TITLE': 'Hello',
    });

    $translateProvider.translations('de', {
      'TITLE': 'Hallo',
    });

  }]);

  app.controller('appCtrl'['$scope','$translate', function($scope, $translate){
    $translate.use('en');
  }]);
</script>
{% endraw %}
{% endhighlight %}

Translation data is being defined with module `$translateProvider`. After config is setup, you can select your target language by using `$translate.use('your-target-language')` on your controller.

This will be enough if you have only few of strings to handle. But if you have more resource to translate, and want to manage list on seperate file, you need more things to do.

Get several libraries with bower again...

{% highlight shell %}
{% raw %}
bower install angular-translate-loader-static-files
bower install angular-translate-storage-local
bower install angular-translate-storage-cookie
{% endraw %}
{% endhighlight %}

and add path for these.

{% highlight html %}
{% raw %}
...
<script src="bower_components/angular-translate-loader-static-files/angular-translate-loader-static-files.js"></script>
<script src="bower_components/angular-translate-storage-local/angular-translate-storage-local.js"></script>
<script src="bower_components/angular-translate-storage-cookie/angular-translate-storage-cookie.js"></script>
...
{% endraw %}
{% endhighlight %}

Now you need to call your JSON list. First, as you know, you cannot inject `$http` service inside configuration on AngularJS. There are some of detour for this, but it is not recommended. Thankfully, there are prebuild module `translateProvider.useStaticFilesLoader` for this.
Change your config file like this:

{% highlight javascript %}
{% raw %}
...
var app = angular.module('appName', ['pascalprecht.translate']);
app.config(
    function ($translateProvider) {

        $translateProvider
            .useStaticFilesLoader({
                prefix: 'json/file/path/',
                suffix: '.json'
            })
            .preferredLanguage('de')
            .fallbackLanguage(['en']).useLocalStorage();
});
...
{% endraw %}
{% endhighlight %}

file: json/file/path/en.json
{% highlight json %}
{% raw %}
{
  "TITLE": "Hello"
}
{% endraw %}
{% endhighlight %}

file: json/file/path/de.json
{% highlight json %}
{% raw %}
{
  "TITLE": "Hallo"
}
{% endraw %}
{% endhighlight %}

This is it. Now this will work just same as first configurations.

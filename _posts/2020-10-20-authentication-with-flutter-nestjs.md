---
layout: post
title: Making authentication logic for Flutter + Nestjs
date:  2020-10-21 
description: 
tags:
- web
- flutter
- dart
- nestjs
- nodejs
- typescript
permalink: 

---


Few days ago, I've planned to design some side project, which is based on mobile application. This is the record about how I've implement authentication logic through application side to back-end side.

I've once wrote about authentication with golang, but in this time I've decided to use nodejs. Still yet golang was not handy to me, and because I've changed my position on work, it requires much time to get used on new work(and studying new programming language).

Moreover, I've decided use Flutter, which I've never used before, so couldn't have much time to assign a time for get used to golang.


## Why Flutter?
If the performance is not a high priority for application, it's inconvenient to develop apps for iOS and Android separately. Luckly there are several ways to develop application for iOS/Android in same code base.

![image](/assets/post_img/authentication-with-flutter-nestjs/flutter-main-page.png)

There are other frameworks(Xamarin, React-Native) to make this done, but lots of performance benchmark(it's pretty close to native!), growth of community, and support from Google make me to choose Flutter. Also, there were similarities with 'component' based development at front-end.

One sad thing(maybe good thing) is there are no other choice than using `Dart` for development, but thought it worths to go on with this.


## Nestjs

// TODO:




## Authentication logic for Flutter
These are the logic for basic authentication.

1. Open application
2. Check state of authentication with stored JWT
   1. If JWT is available, show main page
   2. If not, move to login page
3. In 2-2, login with ID/PW, and store JWT to local storage


You can just call API for each process and check state separately, but let's try to use 'provider pattern' to keep state in unified logic.


### Define API
First let's define APIs to get/post login informations from server.

```dart
import 'dart:async';
import 'dart:convert';
import 'dart:io';

// flutter
import 'package:http/http.dart' as http;
import 'package:logging/logging.dart';
import 'package:flutter_secure_storage/flutter_secure_storage.dart';

// local
import 'package:app/services/api_response.dart';
import 'package:app/services/api_error.dart';
import 'package:app/models/user.dart';

String _baseUrl = "base-url-to-call-api";

Future<ApiResponse> authenticateUser(String username, String password) async {
  ApiResponse _apiResponse = new ApiResponse();
  final _storage = new FlutterSecureStorage();

  try {
    final resp = await http.post('${_baseUrl}api/auth/login', body: {
      'username': username,
      'password': password
    });
    
    if (resp.statusCode == 200 || resp.statusCode == 201) {
      Map mapped= json.decode(resp.body);
      String token = mapped["access_token"];
      await _storage.write(key: "accessToken", value: token);

      String tok = await _storage.read(key: "accessToken");
      return await getProfile();
      
    } else {
      _apiResponse.ApiError = ApiError.fromJson(json.decode(resp.body));
    }
  } catch (e) {
    _apiResponse.ApiError = ApiError(error: "Unknown server error. Please retry");
  }

  return _apiResponse;
}

Future<ApiResponse> getProfile() async {
  final _storage = new FlutterSecureStorage();
  final token = await _storage.read(key: 'accessToken');
  ApiResponse _apiResponse = new ApiResponse();

  try {
    final profile = await http.get(
      '${_baseUrl}api/users/profile', 
      headers: {HttpHeaders.authorizationHeader: 'Bearer ${token}'}
    );

    if (profile.statusCode == 200 || profile.statusCode == 201) {
      _apiResponse.Data = Users.fromJson(json.decode(profile.body));
    } else {
      _apiResponse.ApiError = ApiError.fromJson(json.decode(profile.body));
    }
  } on Exception {
    _apiResponse.ApiError = ApiError(error: "Unknown server error. Please retry");
  }

  return _apiResponse;
}
```

I've used wrapper class for response handler(ApiResponse, ApiError). About this, you can find more in [here](https://mundanecode.com/posts/flutter-restapi-login/). It is one kind of method to keep common rule for response handling.  This post will only focus on managing login state.

There are 2 APIs here:
> - ${_baseUrl}api/auth/login -> login with ID/PW
> - ${_baseUrl}api/users/profile -> get profile with token stored in device. It is to check user is logged in or not.




### with provider pattern
Provider pattern is one of design pattern, to manage application state. In this pattern, states are being managed in separate module, and views which requires these information will check states from this module. If you're familiar with `react-redux` or `vuex`, you can understand more quickly.

For implementation, you need to add `provider` module.


```yaml
dependencies:
  ...
  provider: ^4.3.2
```

First, let's define 'state' of login. Basically, you can think of 'loading', 'success', 'fail',　but you can divide into more segments if needed.
```dart
enum LoginState { Loading, Success, Fail }
```

Now will make provider class. For this, you need to extend `ChangeNotifier`. This is a simple class included in the Flutter SDK which provides change notification to its listeners. By extending this, user can subscribe to its change.

```dart
class AuthProviders with ChangeNotifier {
  
  LoginState _state = LoginState.Fail;
  LoginState get state => _state;

  AuthProviders() {
    this._state = LoginState.Loading;

    getProfile().then((resp) {
      ApiResponse response = resp;
      if (response != null && response.Data != null) {
        _log.info('success auth api call');
        this._state = LoginState.Success;
      } else {
        _log.info('fail auth api call');
        this._state = LoginState.Fail;
      }

      notifyListeners();

    }).catchError((onError) {
      this._state = LoginState.Fail;
      notifyListeners();
    });
  }

  Future<bool> login(String id, String password) async {
    this._state = LoginState.Loading;
    notifyListeners();

    try {
        
      await authenticateUser(id, password);
    } catch (e) {
      this._state = LoginState.Fail;
      
    }

    notifyListeners();
  }
}
```

First, you can see line `LoginState _state = LoginState.Fail`, for initiating state. Init value is 'fail' because it didn't started(or you can make state 'Init' for this).

When user try to call API to login or check state, it will change state as `Loading` until it gets the result(Success/Fail).



## JWT inside Nestjs
Nestjs offers various features for web server development as module, and so as [authentication](https://docs.nestjs.com/security/authentication). I'm able to check whether user is authenticated or not by checking value inside 'Bearer Token' is valid one. It can be checked by parsing header with code, but Nestjs offers smarter way with `passport.js` module.

In Nestjs, it offers `Guards` class annotation. As you can expect by the name, it makes whether a given request will be handled by the route handler or not, depending on certain conditions (like permissions, roles, ACLs, etc.) present at run-time. This is pretty useful function for authentication logic.

First setup scaffold with [Nest CLI](https://docs.nestjs.com/first-steps), install additional modules for this:

```
$ npm install --save passport passport-jwt @nestjs/passport @nestjs/jwt
$ npm install --save-dev @types/passport-jwt
```

`@nestjs/passport` is extention for Nestjs, which will offer extendable module for authentication.





## Reference
* <https://flutter.dev/docs>
* <https://mundanecode.com/posts/flutter-restapi-login/>
* <https://docs.nestjs.com/>
* <https://docs.nestjs.com/security/authentication>


---
layout: post
title: "Unit testing in Flutter: http requests"
date: 2019-03-17
categories: flutter
---

![test-http-requests](/contents/images/test-http-requests/header.png)

With this article I start a new series about testing with Flutter, where I’ll be presenting how to test the main actors in a typical app: models, providers, blocs and widgets.

In this first article I want to demonstrate how to unit test a provider used in a Flutter application to create users in a backend service via http.

Let’s imagine we have a Flutter application with a sign in screen where the user introduces a username and password and presses a button to register. We have also a custom backend that deals with the user creation and authentication tokens.

The following diagram illustrates the sequence of events happening and our different use cases.

![test-http-requests](/contents/images/test-http-requests/sequence.png)

The catch here is that I will always try to do a post call first. If I get a 409 status code in the response it means the user already exists and so I will do a patch instead (to update a timestamp or something similar).

## What code do I want to test?

The code that I use here has been simplified a lot so that I can abstract from complex authentication flows but I can still use it to showcase some testing scenarios.

I’ll be using a class `UserResponse` which represents the user information returned from the backend.

```dart
class UserResponse {
  final String id;
  final String activationCode;

  UserResponse.fromJson(Map<String, dynamic> json)
      : id = json['id'],
        activationCode = json['activationCode'];
}
```

The class `CertHttpProvider` is used to deal with certifications, and will be used as an injected dependency in `IdentityProvider`, which will be our **S**ubject **U**nder **T**est. As we should always depend on abstractions and not on implementations, I have also defined an abstract `HttpProvider`, implemented by `CertHttpProvider`, but that will be the one used by our sut.

```dart
abstract class HttpProvider {
  Future<Response> post(String url,
      {Map<String, String> headers, dynamic body});

  Future<Response> patch(String url,
      {Map<String, String> headers, dynamic body});

  void close();
}

class CertHttpProvider implements HttpProvider {
  // some code that deals with certifications
  // not relevant for our purposes, just provides an implementation
  // for the methods defined in our abstract class HttpProvider
}
```

Finally, `IdentityProvider` exposes only a `create()` method which eventually will return a `User` or will throw an exception.

```dart
class IdentityProvider {
  final String api = 'someapi';

  final HttpProvider http;

  IdentityProvider({@required this.http});

  Future<User> signIn(String id) async {
    try {
      final body = { 'id': id };

      final userResponse = await _createUser(body);

      final tokenResponse = await _getToken(userResponse);

      final user = User(
          uid: userResponse.id,
          token: tokenResponse.token,
          refreshToken: tokenResponse.refreshToken);

      return user;
    } catch (err) {
      throw Exception('Could not create user');
    }
  }

  Future<UserResponse> _createUser(dynamic body) async {
    var response = await http.post('$api/users', body: json.encode(body));
    if (response.statusCode == 409) {
      response = await http.patch('$api/users', body: json.encode(body));
    }

    return response.statusCode == 201
        ? UserResponse.fromJson(json.decode(response.body))
        : throw Exception(response.body);
  }

  Future<TokenResponse> _getToken(UserResponse userResponse) {
    // get a token from an oauth server using the data returned from our api
  }
}
```

The `create()` method does as we designed in our previous diagram. If the `POST` call returns a 409 a `PATCH` will be sent. If there is any other error, the execution will be aborted and an exception thrown.

Again, remember that the code has been simplified. In real life we would use different models for the response and the identity and a more complex and robust authentication flow.

## Test cases

We are going to focus the tests in our `IdentityProvider`. Testing our models it’s fairly simple but the provider is where our core logic resides and we want to make sure we have every case covered. The three different scenarios to test are:

- when the user does not exist, `create()` returns a `User`.
- when the user exists, `create()` returns a `User`.
- when there is an unexpected error from the backend, `create()` throws an exception.

## Mocking and faking

As we want to unit test our `IdentityProvider` we need to mock our backend and use some fake data as responses.

As my `IdentityProvider` gets the `HttpProvider` as an injected dependency it’s very easy to use a mock `HttpProvider` in my tests, because I don’t want to do real requests here. I could manually mock my `HttpProvider` but I’m going to use [mockito](https://pub.dartlang.org/packages/mockito) instead; I just need to create a new class that will extend `Mock` (provided by [mockito](https://pub.dartlang.org/packages/mockito)) and implements the class that I want to mock, and [mockito](https://pub.dartlang.org/packages/mockito) will generate a mock for each method in the class.

```dart
const fakeUserResponse = {
  "id": "123",
  "activationCode": "111111",
};

const fakeTokenResponse = {"access_token": "123456", "refresh_token": "123"};

class MockHttpProvider extends Mock implements HttpProvider {}

void main() {
  MockHttpProvider http;
  IdentityProvider sut;

  final user = User(
      uid: fakeUserResponse['id'],
      token: fakeTokenResponse['access_token'],
      refreshToken: fakeTokenResponse['refresh_token']);

  setUp(() {
    // I use this method to initialize my environment for the tests
    // basically api values and such
    buildTestEnvironment();
    http = MockHttpProvider();
    sut = IdentityProvider(http: http);
  });

  tearDown(() {
    http.close();
    clearInteractions(http);
    reset(http);
    http = null;
    sut = null;
  });
}
```

`FakeUserResponse` is the data that I’m going to use as response for a successful call to our mocked backend.

The variable `expected` will be used in our tests to verify that the result of our calls is what we really expect. We won’t use this in every test, but as this will be returned in two of our tests we can define it as final and share it by both of them.

Finally, `setUp()` and `tearDown()` are used to prepare and clear the environment before each test in our suite. Basically, what we are doing in this case is initializing our sut and closing the client connection.

## Helper functions

Thanks to [mockito](https://pub.dartlang.org/packages/mockito) I have mocked implementations of `post()` and `path()`, which I can now use to spy my calls and return whatever I need in each test. Let’s define two helper functions that will be very handy in all our tests:

```dart
main() {

    ...

    void stubPost(String url, Response response) {
      when(http.post(argThat(startsWith(url)), body: anyNamed('body')))
          .thenAnswer((_) async => response);
    }

    void stubPatch(String url, Response response) {
      when(http.patch(argThat(startsWith(url)), body: anyNamed('body')))
          .thenAnswer((_) async => response);
    }

    ...
}
```

For `stubPost()`, we want that whenever a `POST` call is made to a url starting with the param provided, we will get the response also passed as a parameter. For `stubPatch()` it’s exactly the same.

## Test user creation when the user does not exist

```dart
  test('create returns a user if user does not exist', () async {
    final expected = user;

    stubPost(sut.api, Response(json.encode(fakeUserResponse), 201));

    stubPost(sut.oauthApi, Response(json.encode(fakeTokenResponse), 201));

    final result = await sut.signIn();
    expect(result, expected);
  });
```

Easy one, when the user does not exist, our `POST` will return a 201, and our second call to retrieve the token will also return a 201.

## Test user creation when the user already exists

```dart
test('create returns a user if user already exists', () async {
    final expected = user;

    stubPost(sut.api, Response('user already exists', 409));
    stubPatch(sut.api, Response(json.encode(fakeUserResponse), 201));

    stubPost(sut.oauthApi, Response(json.encode(fakeTokenResponse), 201));

    final result = await sut.signIn();
    expect(result, expected);
});
```

In this case, the first `POST` will return a 409, and the `PATCH` a 201 with the proper user. Finally, the `POST` to retrieve the token will return a 201.

We then execute our `signIn()` method, calls will be intercepted and we will get our user with the successful `PATCH` call.

## Test user creation when there is an unexpected error

```dart
test('create throws an exception if there is any error', () async {
    stubPost(sut.api, Response('unknown error', 500));

    expect(sut.signIn(), throwsException);
});
```

Also an easy test, we just need to return a 500 status code. This will not trigger a `PATCH` but will just throw an Exception instead.

That’s the complete example of my test class, including the imports that I need:

```dart
import 'dart:convert';

import 'package:flutter_test/flutter_test.dart';
import 'package:http/http.dart' show Response;
import 'package:mockito/mockito.dart';

import 'package:app/src/models/user.dart';
import 'package:app/src/providers/http_provider.dart';
import 'package:app/src/providers/identity_provider.dart';

// buildTestEnvironment()
import '../../config.dart';

const fakeUserResponse = {
  "id": "123",
  "activationCode": "111111",
};

const fakeTokenResponse = {"access_token": "123456", "refresh_token": "123"};

class MockHttpProvider extends Mock implements HttpProvider {}

void main() {
  MockHttpProvider http;
  IdentityProvider sut;

  final user = User(
      uid: fakeUserResponse['id'],
      token: fakeTokenResponse['access_token'],
      refreshToken: fakeTokenResponse['refresh_token']);

  void stubPost(String url, Response response) {
    when(http.post(argThat(startsWith(url)), body: anyNamed('body')))
        .thenAnswer((_) async => response);
  }

  void stubPatch(String url, Response response) {
    when(http.patch(argThat(startsWith(url)), body: anyNamed('body')))
        .thenAnswer((_) async => response);
  }

  setUp(() {
    buildTestEnvironment();
    http = MockHttpProvider();
    sut = IdentityProvider(http: http);
  });

  tearDown(() {
    http.close();
    clearInteractions(http);
    reset(http);
    http = null;
    sut = null;
  });

  test('create returns a new user if user does not exist', () async {
    final expected = user;

    stubPost(sut.api, Response(json.encode(fakeUserResponse), 201));

    stubPost(sut.oauthApi, Response(json.encode(fakeTokenResponse), 201));

    final result = await sut.signIn();
    expect(result, expected);
  });

  test('create returns an existing user if user already exists', () async {
    final expected = user;

    stubPost(sut.api, Response('user already exists', 409));
    stubPatch(sut.api, Response(json.encode(fakeUserResponse), 201));

    stubPost(sut.oauthApi, Response(json.encode(fakeTokenResponse), 201));

    final result = await sut.signIn();
    expect(result, expected);
  });

  test('create throws an exception if there is any error', () async {
    stubPost(sut.api, Response('unknown error', 500));

    expect(sut.signIn(), throwsException);
  });
}
```

That’s all, you can see how with some minimal boilerplate we can write very lean tests to verify our test cases. Again, remember I simplified quite a bit the code to try to focus on the tests, so please feel free to write in the comments if you want some further explanations.

Thanks for reading!

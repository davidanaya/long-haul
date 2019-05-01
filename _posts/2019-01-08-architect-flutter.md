---
layout: post
title: "How to architect a small to medium Flutter application"
date: 2019-01-08
categories: flutter
---

In a [previous article](https://davidanaya.io/bloc), I showed how to apply the BLoC pattern to a small Flutter application. Even though this is a very powerful pattern that scales really well, the truth is that it adds some complexity that might not be desirable when building simple applications.

Here I’ll use a very basic chat application to show an alternative way to organize our apps that works really well when we don’t really need the benefits of blocs.

## What are we building?

I want to use a simple application but that still has some complexity so that we can see that even when there is a business logic this alternative is still valid and works well, so I created a very basic chat application using Firebase as our backend.

![monday-morning-dialog](/contents/images/architect-flutter/morning-dialog.png)

The app would allow multiple users to interact with it at the same time, so this is kind of a chat room, and by using Firebase we don’t need to worry about implementing our own backend with web sockets; every time a user sends some text, the conversation will be updated for all the users.

## How are we building it?

The are different ways to design this application, each of them with their own advantages and disadvantages.

- The simplest solution would be to create a single stateful widget in charge of everything. This widget would be in charge of both the UI and the communication with Firebase. This wouldn’t scale very well though, since in case we wanted to reuse anything somewhere else in the app we would need to duplicate the code.

![simple-not-scalable](/contents/images/architect-flutter/simple.png)

- The most scalable solution would be using BLoC pattern. In this case, we could have a widget in charge of the UI only, and we would create a bloc to deal with the state, in this case messages. By doing this we could access this bloc from anywhere in the app very easily without the need to duplicate any code. This is extremely useful, but there is some boilerplate as we already saw in a previous article.

![bloc-very-scalable](/contents/images/architect-flutter/bloc.png)

- Somewhere in between would be my preferred choice for this use case scenario. We are not reusing this logic anywhere else in the application nor do we have multiple states, but I don’t really want to tie my logic to the view. Let’s use a service instead to deal with the state and inject it into our view widget.

![simple-yet-scalable](/contents/images/architect-flutter/scalable.png)

## Enough talking, let’s see some code

I’m far more interested in the architecture than the implementation details in this article, so I won’t go in much detail into the code, but feel free to ask in the comments section and I’ll be happy to answer.

First, I’m going to add an extra class to the mix, Message, which will model the messages in the chat. We will use the `fromMap` constructor to load data from Firebase and `toSnapshot` to serialize our model to `json` to store in Firebase. The integration is fairly easy and I’m using the (cloud_firestore library)[https://pub.dartlang.org/packages/cloud_firestore] to connect my application to my database.

```dart
import 'package:intl/intl.dart';

import 'package:cloud_firestore/cloud_firestore.dart';

class Message {
  DocumentReference reference;
  final DateTime date;
  final String text;
  final String author;

  final _timeFormatter = DateFormat.jm();

  Message(this.text, this.author) : date = DateTime.now();

  Message.fromMap(Map<String, dynamic> map, {this.reference})
      : author = map['author'],
        text = map['text'],
        date = DateTime.parse(map['date']);

  Message.fromSnapshot(DocumentSnapshot snapshot)
      : this.fromMap(snapshot.data, reference: snapshot.reference);

  Map<String, dynamic> toSnapshot() =>
      {'text': text, 'author': author, 'date': date.toIso8601String()};

  String get time => _timeFormatter.format(date);
}
```

This is how my database will look for this example.

![firebase](/contents/images/architect-flutter/firebase-collection.png)

The link between my model and the widgets that will render the chat page will be the `ChatService` as you can see in my diagram.

```dart
import 'dart:async';

import 'package:basic_chat_flutter_workshop/src/models/message.dart';

abstract class ChatService {
  Future<void> addMessage(Message message);

  Stream<List<Message>> get messages$;
}
```

I’m basically exposing 2 methods here, the first one will be used to add a new message to the database. So, when I type something in the application and I press the `Send` button, I will use this method.

The second one is a stream with the whole list of messages in the database. Every time a new message is added, this stream will emit a new value with the updated list of messages.

As you can see, this is an abstract class, not a real implementation. We will see why it’s much better to do it this way shortly. Let’s see how we implement this service for Firebase.

```dart
import 'dart:async';

import 'package:cloud_firestore/cloud_firestore.dart';

import 'package:basic_chat_flutter_workshop/src/models/message.dart';
import 'package:basic_chat_flutter_workshop/src/services/chat_service.dart';

class ChatFirebase extends ChatService {
  final Stream<QuerySnapshot> _snapshots$;
  final CollectionReference _collection;

  ChatFirebase()
      : _snapshots$ = Firestore.instance.collection('messages').snapshots(),
        _collection = Firestore.instance.collection('messages');

  @override
  Future<void> addMessage(Message message) {
    return _collection.document().setData(message.toSnapshot());
  }

  @override
  Stream<List<Message>> get messages$ =>
      _snapshots$.map((snapshot) => snapshot.documents.reversed
          .map((document) => Message.fromSnapshot(document))
          .toList());
}
```

Again, without going into much detail, we get a reference to a Firebase instance and store the stream of snapshots for our collection and also the proper collection. Then we use the first one as a reactive stream and map it to create proper message models from the data. We use the second one to add a new message into the collection.

Now we just need the view. We could use a single widget, but I think it’s better to separate the page from the components widgets; even if we don’t reuse them, we get much better readability.

```dart
import 'package:flutter/material.dart';

import 'package:basic_chat_flutter_workshop/src/services/chat_service.dart';
import 'package:basic_chat_flutter_workshop/src/widgets/chat_input.dart';
import 'package:basic_chat_flutter_workshop/src/widgets/chat_list.dart';
import 'package:basic_chat_flutter_workshop/src/models/message.dart';

class ChatPage extends StatelessWidget {
  final bkgColor = Colors.blueGrey[100];

  final String _username;
  final ChatService _chatService;

  ChatPage(this._chatService, this._username);

  @override
  Widget build(BuildContext context) {
    return Container(color: bkgColor, child: _buildChat());
  }

  Widget _buildChat() {
    return Column(
      mainAxisAlignment: MainAxisAlignment.spaceBetween,
      children: <Widget>[
        _buildChatList(),
        ChatInput(_username, onSubmit),
      ],
    );
  }

  Widget _buildChatList() {
    return Expanded(
      child: StreamBuilder<List<Message>>(
          stream: _chatService.messages$,
          builder: (context, snapshot) => (snapshot.hasData)
              ? ChatList(snapshot.data, _username, backgroundgColor: bkgColor)
              : Center(child: CircularProgressIndicator())),
    );
  }

  void onSubmit(Message message) {
    _chatService.addMessage(message);
  }
}
```

By using a stream to retrieve the messages from Firebase we can now use `StreamBuilder` in the `_buildChatList()` function and make sure that the list of messages will be rebuilt every time the stream emits a new value.

We pass the `onSubmit()` function as a parameter in the `ChatInput` constructor, and it will be executed when the user presses the `Send` button.

I won’t be showing the code for the `ChatInput` or the `ChatList` here as it’s basically styling and I want to keep this article simple.

The last step will be glueing all these widgets and the service. We could do this anywhere, but I like to do it in the main file, at least with small applications like this.

```dart
import 'package:flutter/material.dart';

import 'package:basic_chat_flutter_workshop/src/services/chat_service.dart';
import 'package:basic_chat_flutter_workshop/src/pages/chat_page.dart';
import 'package:basic_chat_flutter_workshop/src/pages/login_page.dart';
import 'package:basic_chat_flutter_workshop/src/services/chat_firebase.dart';

void main() {
  var chatService = ChatFirebase();

  runApp(MyApp(chatService));
}

class MyApp extends StatelessWidget {
  final ChatService chatService;

  MyApp(this.chatService);

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Flutter Workshop',
      theme: ThemeData(
        primarySwatch: Colors.blue,
      ),
      home: MyHomePage(chatService),
    );
  }
}

class MyHomePage extends StatefulWidget {
  final ChatService chatService;

  MyHomePage(this.chatService);

  @override
  _MyHomePageState createState() => _MyHomePageState();
}

class _MyHomePageState extends State<MyHomePage> {
  final String title = 'Welcome to MyFChat';
  String _username;
  String _title;

  @override
  void initState() {
    super.initState();
    _title = title;
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text(_title),
        actions: _buildAppBarActions(),
      ),
      body: Center(
        child: _username != null
            ? ChatPage(widget.chatService, _username)
            : LoginPage(_onLog),
      ),
    );
  }

  void _onLog(String username) {
    setState(() {
      _username = username;
      _title = _username != null ? '$title, $_username' : title;
    });
  }

  List<Widget> _buildAppBarActions() {
    return _username == null
        ? []
        : [
            IconButton(
              icon: Icon(Icons.exit_to_app),
              onPressed: () => _onLog(null),
            )
          ];
  }
}
```

We create an instance of our `ChatFirebase` service and pass it down to the `MyHomePage` widget and eventually to the `ChatPage` widget as a `ChatService`.

In the final architecture we can see the business logic and the data decoupled from the view, which is what we also get with the BLoC pattern.

![final-architecture](/contents/images/architect-flutter/final-architecture.png)

## Why is this approach better in this case?

For big applications where we want maximum flexibility and scalability, I’d still go for the BLoC pattern, but for small to medium size applications I think this approach is very interesting.

- Keep boilerplate to a minimum while still having good readability and scalability.

- Injecting abstractions (`ChatService`) instead of implementations (Firebase) will us to unit test our widgets in isolation, by using fake implementations. Also, we can provide different implementations (Firebase, SQLite,…) and use one or the other without modifying our widgets.

- Upgrading to a BLoC pattern in the future is very easy as we will be using our services as they are, but instead of injecting them in the widgets we will do it in the blocs.
- In case we don’t like to initialize all our services in the main method, we could do it in the pages where they are used and use those pages as our key widgets in the application.

That’s all, feel free to ask for clarifications in the comments or if you want me to go in further detail on some of the topics here in a next article. Thanks for reading!

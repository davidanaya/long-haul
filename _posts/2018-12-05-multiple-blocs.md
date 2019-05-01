---
layout: post
title: "Multiple BLoCs in Flutter and how to communicate between them"
date: 2018-12-05
categories: flutter
---

In [my previous article](https://www.davidanaya.io/bloc) I briefly explained what BLoC pattern is and how to transform an application to use this architectural design, using a basic counter application that we get when we create a new Flutter project as an example.

In this article I want to continue with the same application and showcase how to separate our business logic into different blocs and also how to communicate them.

## How can I separate my logic in different blocs?

I hope by now you can see the power behind this architecture, but you can probably see some flaws as well. With the initial design I can hold my state inside a widget and clearly separate my logic by domain, but does having this bloc mean that I have to put all my state in the same class/bloc? Of course not, we can still create different blocs.

Imagine I want to display a different counter now, but in this case the increment will be done by 2, so every time we press the button we will increment our initial counter by 1 and the second counter by 2, something like this.

![double-counter](/contents/images/multiple-blocs/double-counter.gif)

Pretty basic, but again you can extrapolate this into a different use case where instead of two counters we could have different sets of data with their own sets of related functions.

So, let’s create our new bloc class, `EvenCounterBloc`, which in this case will be pretty similar to the original `CounterBloc`.

```dart
import 'dart:async';

import 'package:rxdart/rxdart.dart';
import 'package:rxdart/subjects.dart';

class EvenCounterBloc {
  int _counter = 0;

  final _counter$ = BehaviorSubject<int>(seedValue: 0);
  final _incrementController = StreamController<void>();

  EvenCounterBloc() {
    _incrementController.stream.listen((void _) {
      _counter = _counter + 2;
      _counter$.add(_counter);
    });
  }

  Sink<void> get increment => _incrementController.sink;

  Stream<int> get counter$ => _counter$.stream;

  void dispose() {
    _incrementController.close();
    _counter$.close();
  }
}
```

Now, instead of exposing the `CounterBloc` directly with our `BlocProvider`, let’s create an extra bloc, `AppBloc`, which can be used as a gatekeeper and orchestrator.

```dart
import 'package:flutter_counter_bloc/bloc/counter_bloc.dart';
import 'package:flutter_counter_bloc/bloc/even_counter_bloc.dart';

class AppBloc {
  CounterBloc _counter;
  EvenCounterBloc _evenCounter;

  AppBloc()
      : _counter = CounterBloc(),
        _evenCounter = EvenCounterBloc();

  CounterBloc get counterBloc => _counter;
  EvenCounterBloc get evenCounterBloc => _evenCounter;
}
```

As you can see, it’s a very simple class, with a couple of getters to access our blocs.

And finally, our widget will use both blocs to display the counters.

```dart
class MyHomePage extends StatelessWidget {
  MyHomePage({Key key, this.title}) : super(key: key);
  final String title;

  @override
  Widget build(BuildContext context) {
    final bloc = BlocProvider.of(context).counterBloc;
    final evenBloc = BlocProvider.of(context).evenCounterBloc;
    return Scaffold(
      appBar: AppBar(
        title: Text(title),
      ),
      body: Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: <Widget>[
            StreamBuilder(
                stream: evenBloc.counter$,
                builder: (context, snapshot) => snapshot.hasData
                    ? Text('${snapshot.data}',
                        style: Theme.of(context).textTheme.display1)
                    : CircularProgressIndicator()),
            StreamBuilder(
                stream: bloc.counter$,
                builder: (context, snapshot) => snapshot.hasData
                    ? Text('${snapshot.data}',
                        style: Theme.of(context).textTheme.display1)
                    : CircularProgressIndicator()),
          ],
        ),
      ),
      floatingActionButton: FloatingActionButton(
        onPressed: () {
          bloc.increment.add(null);
          evenBloc.increment.add(null);
        },
        tooltip: 'Increment',
        child: Icon(Icons.add),
      ),
    );
  }
}
```

We just added a new `StreamBuilder` subscribed to the new counter in `EvenCounterBloc` and used the sink to increment it when pressing the button.

In this case, we are treating both counters as totally independent, but what if we wanted the even counter to be based on the original counter? That is, what if the even counter had a dependency on the original one?

## BLoC dependencies

There are different solutions to achieve this. We could inject the first bloc as a dependency of the second bloc when we create them, but then that makes the second bloc aware of this dependency. We could also try to come up with a `SharedBloc` where we could keep the state that is shared among different domains, but that feels a bit clunky.

Instead, we can use the power of streams to interconnect different blocs. If every bloc exposes a set of streams as outputs and a set of sinks as inputs, why not use those to connect different blocs as a sort of pipe? Let’s go for it.

First off, we can remove the increment from the even bloc in the widget as it was before.

```dart
floatingActionButton: FloatingActionButton(
        onPressed: () => bloc.increment.add(null),
        tooltip: 'Increment',
        child: Icon(Icons.add),
      ),
```

Now, what we want is for our EvenCounterBloc to depend on the original counter, so that every time this counter is incremented, our even counter is updated. Let’s do that in our AppBloc.

```dart
import 'package:flutter_counter_bloc/bloc/counter_bloc.dart';
import 'package:flutter_counter_bloc/bloc/even_counter_bloc.dart';

class AppBloc {
  CounterBloc _counter;
  EvenCounterBloc _evenCounter;

  AppBloc() {
    _counter = CounterBloc();
    _evenCounter = EvenCounterBloc();
    _counter.counter$.listen(_evenCounter.increment.add);
  }

  CounterBloc get counterBloc => _counter;
  EvenCounterBloc get evenCounterBloc => _evenCounter;
}
```

In line 11 we are doing exactly that. We pipe our output (stream) from `CounterBloc` to the input (sink) of our `EvenCounterBloc`. Every time a new value is emitter by `_counter.counter$`, this value will be passed along as the parameter for `_evenCounter.increment.add()` function.

Now we just need to adjust our `EvenCounterBloc`.

```dart
import 'dart:async';

import 'package:rxdart/rxdart.dart';
import 'package:rxdart/subjects.dart';

class EvenCounterBloc {
  final _counter$ = BehaviorSubject<int>(seedValue: 0);
  final _incrementController = StreamController<int>();

  EvenCounterBloc() {
    _incrementController.stream.listen((int value) => _counter$.add(value * 2));
  }

  Sink<int> get increment => _incrementController.sink;

  Stream<int> get counter$ => _counter$.stream;

  void dispose() {
    _incrementController.close();
    _counter$.close();
  }
}
```

Our sink now requires an int value, and whenever we get it, we multiply it by 2 and emit the result. No need to keep an internal property with the counter as it now depends on the value received in our sink.

Arguably, we could now rename our increment getter to something more meaningful, and you can imagine that we could do far more complex things here, but the whole point is that we don’t need to deal with dependencies in the bloc itself. Every piece of the state is now clean and isolated, and we rely on the `AppBloc` as some sort of blocs manager to sort out the dependencies for us in a single point.

You can check the complete code in my (GitHub repository)[https://github.com/davidanaya/flutter-counter-bloc]. Thanks for reading!

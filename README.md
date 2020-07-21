# Nuvigator

[![CircleCI](https://circleci.com/gh/nubank/nuvigator/tree/master.svg?style=svg)](https://circleci.com/gh/nubank/nuvigator/tree/master)
[![Pub](https://img.shields.io/pub/v/nuvigator.svg)](https://pub.dartlang.org/packages/nuvigator)

Routing and Navigation package.


This package aims to provide a powerful routing abstraction over Flutter Navigators. Using a most declarative and concise
approach you will be able to model complex navigation flows without needing to worry about several tricky behaviours
that the framework will handle for you.

Below you will find a description of the main components of this frameworks,
and how you can use they in your project.

# Main Concepts

## Basic usage

```dart
import 'package:flutter/widgets.dart';
import 'package:nuvigator/nuvigator.dart';

part 'main_router.g.dart';

@NuRouter()
class MainRouter extends Router {

  @NuRoute(deepLink: 'myapp://myRoute')
  ScreenRoute myRoute() => ScreenRoute(
    builder: (context) => MyScreenWidget(),
  );

  @override
  Map<RouteDef, ScreenRouteBuilder> get screensMap  => _$screensMap;
}

class MyApp extends StatelessWidget {

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      builder: Nuvigator(
        router: MainRouter(),
        screenType: materialScreenType,
        initialRoute: 'myapp://myRoute',
      ), 
    );
  }
}

void main() {
  runApp(MyApp());
}

```

To generated required files you can run:
```bash
flutter pub run build_runner build --delete-conflicting-outputs
```

To start a watcher that will re-create the files at each change you can also use:
```bash
flutter pub run build_runner watch --delete-conflicting-outputs
```

## Nuvigator

`Nuvigator` is a custom `Navigator`, it behaves just like a normal `Navigator`, but with several custom improvements and
features. It is engineered specially to work under nested scenarios, where you can have several Flows one inside another,
it will event ensure that your Hero transitions work well under those scenarios. `Nuvigator` will be your main interface 
to interact with navigation, and  it can be easily fetched from the context, just like a normal `Navigator`, ie: `Nuvigator.of(context)`.

Nuvigator includes several extra methods that can be used, for example the `Nuvigator.of(context).openDeepLink()` methods
that will try to open the desired deepLink. Or event the `Nuvigator.of(context).closeFlow()` that will try to close a nested
Nuvigator.

Each `Nuvigator` should have a `Router`. The `Router` acts as the routing controller. While the `Nuvigator` will
be responsible for visualization, widget render and state keeping.

## Router and Routes

Defining a new Router will probably be the what you will do the most when working with Nuvigator, so understanding how
to do it properly is important. A Router class is a class that should extend a `Router`. To have it's code generated you should
also annotate it with `@NuRouter`.
 
Inside your Router class you can define your Routes, each Route will be represented by a method annotated with the
`@NuRoute` annotation. Those methods should return a `ScreenRoute`, and can receive any number of **named** arguments,
those will be used as the Arguments that should/can be passed to your route when navigating to it.

Example:
```dart
@NuRouter()
class MyCustomRouter extends Router {
  
  @NuRoute(deepLink: 'myapp://firstScreen')
  ScreenRoute firstScreen({String argumentHere}) => ScreenRoute(
    builder: (_) => MyFirstScreen(),
  );
  
  @NuRoute(deepLink: 'myapp://secondScreen/:name')
  ScreenRoute secondScreen({String name}) => ScreenRoute(
    builder: (context) => MySecondScreen(userName: name)
  );
  
  @override
  Map<RouteDef, ScreenRouteBuilder> get screensMap  => _$screensMap; // Will be created by generated code

}
```

When using the `NuRoute` annotation you will need to provide a `routePath` for this Route, this is the string that
represents the "address" of your Route. This is also referred as DeepLink, and it follows the format of an URL path pattern.
You can use path parameters to extract those and make available to the Route, to declare a path parameter you can use `:`
before the parameter name, just like: `routeName/:parameterHere`. In addition to that query parameters passed when
trying to open a new DeepLink will also be made available and can be extracted into the method parameters. 

Inside the Router class you have access to the local `nuvigator` that can be used to do navigation operations, and also
to the generated extension methods to navigate between the screens of your router.

### Router Prefix

Inside you Router you can define a common prefix to use for all of your Routes declared in it. To do this
just override the `deepLinkPrefix` getter.

Example:
```dart
@NuRouter()
class MyCustomRouter extends Router {
  
  String get deepLinkPrefix => 'prefixHere/';

  @NuRoute(deepLink: 'firstScreen')
  ScreenRoute firstScreen({String argumentHere}) => ScreenRoute(
    builder: (_) => MyFirstScreen(),
  );
  
  @NuRoute(deepLink: 'secondScreen/:name')
  ScreenRoute secondScreen({@required String name}) => ScreenRoute(
    builder: (context) => MySecondScreen(userName: name)
  );
}
```

### ScreenRoute

Envelopes a widget that is going to be presented as a Screen by a Nuvigator. The Screen contains a ScreenBuilder 
function, and some attributes, like screenType, that are going to be used to generate a route properly.

When creating a `ScreenRoute` it can receive a optional generic type that will be considered as the type of the value
to be returned by this Route when popped.

Screen Options:

- builder
- screenType
- wrapper

### Merging/Grouping Routers

In large applications it's common to have many Routes and even having them split among several packages, to better support
this Nuvigator has the concept of Grouped Routers. This is basically a way to merge several Routers into one final big Router
that will be used by the application entry point. Grouping Routers does **NOT** stabilises a nesting or parent/children
relationship, instead it just acts as a "merge" function for your Routes. You can think it as a way of merging or grouping Routers.

To declare that a Router should be merged into another one, you can use the `@NuRouter` annotation in a property containing 
the Router that will include the Routes of the second Router.

Example:
```dart
@NuRouter()
class MyMain extends Router {
  @NuRouter()
  FirstRouter firstRouter = FirstRouter();

  @NuRouter()
  SecondRouter secondRouter = SecondRouter();
  
  @override
  List<Router> get routers => _$routers; // Generated
}
```
 obs :Remember to include the override with the generated `_$routers` field.
 
This way all the routes declared in both `FirstRouter` and `SecondRouter` will be available to the `MainRouter`. Note that
however, the `deepLinkPrefix` declared in the `MainRouter` do take effect for the Routes declared in the other Routers, and 
of course, after that they will respect the owns Router `deepLinkPrefix`.

### Nested Nuvigators

The concept of nested Nuvigators comes from the need of including self contained autonomous flow inside the App. Having a 
nested Flow is nothing more than rendering another Nuvigator inside a regular route.

```dart
  @NuRoute(deepLink: 'myFlow/:id')
  ScreenRoute myCustomFlow({@required String id}) => ScreenRoute(
    builder: (context) => Nuvigator(
      router: FlowRouter(),
      initialRoute: FlowRoutes.firstScreen,
    ),
  );
```

Nuvigator includes a great support for this kind of pattern, for both runtime and also development experience and capabilities.
One notable feature for example is the `nuvigator.closeFlow()` method, that allows you to close a whole nested flow easily.

### Wrappers

It's common to require your screen to be presented in a context where some specific information is available, one usual way of doing it
is to use inherited widgets, providers, etc... All of those methods rely on wrapping your widget inside another one. To help
with this task Nuvigator has special extension points where you can provide Wrappers functions to wrap your screens. 

You can provide an override to the `screensWrapper` getter in your Router, this method should return a function that will
be called to wrap each one of the screens presented by this Router. Note that the function is going to be executed every time
a new screen is going to be showed, and not once per Router, so if you wish to shared a value between all of your screen be sure
to keep track of the instance somewhere, instead of creating a new instance every time inside this function.
Also, `screensWrapper` are composed over Routers when they are grouped, this means that wrappers present in routers that 
contains Grouped Routers are going to be applied too (in addition to the wrapper defined in the merged Router).

Example:
```dart
@NuRouter()
class MyMainRouter extends Router {
  @NuRouter()
  FirstRouter firstRouter = FirstRouter();

  @NuRouter()
  SecondRouter secondRouter = SecondRouter();
  
  @override
  WrapperFn get screensWrapper => (BuildContext context, Widget child) {
    return ChangeNotifierProvider<SamplesBloc>.value(
      value: SamplesBloc(),
      child: child,
    );
  };

  @override
  List<Router> get routers => _$routers; // Generated
}
``` 

In this scenario above, the `MyMainRouter.screensWrapper` is going to wrap the wrapper of `FirstRouter` or `SecondRouter`
: `MyMainRouter.screensWrapper(FirstRouter.screensWrapper(..screen..))`. This mean that the wrapper present in `FristRouter`
of `SecondRouter` will have any information added by the `MyMainRouter` wrapper available to be used in their
respective wrappers.

## Code Generators

You probably noted in the examples above that we have methods that will be created by the Nuvigator generator. So while
they don't exists you can just make they return `null` or leave un-implemented. 

Before running the generator we recommend being sure that each Router is in a separated file, and also make sure that you
have added the `part 'my_custom_router.g.dart';` directive and the required imports (`package:nuvigator/nuvigator.dart` 
and `package:flutter/widgets.dart`) in your router file. 

After running the generator (`flutter pub run build_runner build --delete-conflicting-outputs`), you will notice that 
each router file will have it's part of file created. Now you can complete the `screensMap` and `routersList` functions
with the generated: `_$myScreensMap(this);` and `_$samplesRoutersList(this);`. Generated code will usually follow this pattern
of stripping out the `Router` part of your Router class and using the rest of the name for generated code.

Generated code includes the following features:

- Navigation extension methods to Router
- Typed Arguments classes
- Implementation Methods

### Typed Argument Classes

Those are classes representing the arguments each route can receive. They include parse methods to extract itself from
the BuildContext. eg:

```dart
class SecondArgs {
  SecondArgs({@required this.testId});

  final String testId;

  static SecondArgs parse(Map<String, Object> args) {
    ...
  }

  static SecondArgs of(BuildContext context) {
    ...
  }
}
```

### Navigation Extensions

It's a helper class that contains a bunch of typed navigation methods to navigate through your App. Each declared `@NuRoute`
will have several methods created, each for one of the existing push methods. You can also configure which methods you want
to generate using the `@NuRoute.pushMethods` parameter.

```dart
final router = Router.of<SamplesRouter>(context);
await router.sampleOneRouter.toScreenOne(testId: 'From Home');
await router.toSecond(testId: 'From Home');
```

## Custom Routes/Transitions

Nuvigator uses the concept of `ScreenType` on top of Routes. A `ScreenType` is basically a class that knows how to transform
a builder function into a Flutter `Route`. Nuvigator ships with the default Flutter `Material` and `Cupertino` screenTypes,
but this is a extensible mechanism and it's very easy to create your own `ScreenTypes`. 

You can take a look at how Nuvigator implements the `Material` screenType [here](lib/src/screen_types/material_screen_type.dart).
Basically you should create a class that extends `ScreenType` and implement the `toRoute` method, and that's all! One pro tip
about using custom Routes is: when creating your custom Flutter `Route` class, be sure use the mixin `NuvigatorPageRoute`, it will
add some behavior override, that will make your Screen handle correctly leadingIcons in AppBars, and other default Flutter stuff in
nested Nuvigators.

We also provide some helpers to build simpler Route transitions, you can take a look at `NuvigatorRouteBuilder` and `ScreenTypeBuilder`
classes.

## Dynamic API

If you with to use Nuvigator without code generation features, you can rely on defining things dynamically.
Providing a Router instance with the public methods implemented is enough to make it work properly with Nuvigator, this
way you can implement any custom logic you want in your Router.

If you just want to construct Routers dynamically you can use the helpers:
 - `mergeRouters` that will return a Router with all Routers passed as argument Grouped into it.
 - `RouteHandler` will return a Router that handles just one Route, you need to provide just the DeepLink and a screenRoute.
 - `GenericRouter` is a class that can receive in the constructor arguments to create a simple router.

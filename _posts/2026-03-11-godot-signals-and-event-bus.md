---
title: "Decoupling systems: Godot signals using event bus"
description: "🚌 la la la"
categories: [Coding]
tags: [godot, godot signals, eda]
---


- [Intro](#intro)
  - [🎵 Door SFX example](#-door-sfx-example)
- [Problem: Subscriber needs a direct access to publisher](#problem-subscriber-needs-a-direct-access-to-publisher)
- [Solution: Signal scopes](#solution-signal-scopes)
  - [🚪 Object scope](#-object-scope)
  - [🌎 Global scope](#-global-scope)
- [Global scope in details](#global-scope-in-details)
  - [Implementation via bus event](#implementation-via-bus-event)
  - [All the dangers](#all-the-dangers)
  - [Comparison to Event-Driven Architecture](#comparison-to-event-driven-architecture)
  - [DDD comment](#ddd-comment)
- [Solution: Connecting signals in UI](#solution-connecting-signals-in-ui)
- [Random commentary](#random-commentary)
  - [DIP comment](#dip-comment)
  - [Local Event Bus](#local-event-bus)
  - [🤷‍♂️ Why using signals at all](#️-why-using-signals-at-all)

> WIP
{: .prompt-danger }
<!-- lint fight -->
> Official docs: [link](https://docs.godotengine.org/en/stable/classes/class_signal.html)
> Official tutorial: [link](https://docs.godotengine.org/en/stable/getting_started/step_by_step/signals.html)
{: .prompt-book }

## Intro

> For a more theoretical perspective on treating Godot signals as EDA events, check out my previous post.
> This guide can be read on its own.
{: .prompt-info }

Godot signals are a powerful tool for decoupling systems in your project.

How official docs describe them:
> Signals are a delegation mechanism built into Godot that allows one game object to react to a change in another ***without them referencing one another***.

However, a zero-reference promise isn't fully realized in examples.

In this post I discuss:

- Direct reference problem and how it can be solved via event bus.
- What new dangers would appear with new approach.
- Further EDA comparisons and some other tech notes, like DDD commentary.

### 🎵 Door SFX example

#### Simple direct call

{% include shared/door_example_direct_call.md %}

#### Decoupling using Godot's signals

{% include shared/door_example_signal_version.md %}

> In EDA terms, the system emitting the event is the Publisher, and the system listening is the Subscriber.
> I will be using these terms. For more details, see [link].
> In our setup, Door publishes and SFXSystem subscribes.
{: .prompt-info }

## Problem: Subscriber needs a direct access to publisher

Event-Driven Architecture typically handles interactions between fully decoupled systems.
But in our example a subscriber still holds a direct reference to the publisher:

```gdscript
class_name SFXSystem

var door: Door # <-- here!

func _ready():
	door.door_opened.connect(_on_door_opened)
```

In case of a scene where both door and its dependent systems are together, it's fine.

But imagine another scenario:

A Achievements service tracks how many doors the player opens. Naturally, this service doesn't have references to every door in the game, and it probably doesn't even know the Door class exists.

This situation is closer to typical EDA: achievement service is independent from the door class, but they need to communicate.
And we discussed here so much, how signals implement event-driven relationships. What can be done?

## Solution: Signal scopes

To solve this, I distinguish between two structural approaches which I called "signal scopes": object scope and global scope.
The door example falls into the object scope, so we'll break that pattern down first.

### 🚪 Object scope

This is the most common way to use signals.
It’s the approach shown in the official Godot tutorial and used by built-in nodes.

Object-scoped signals are attributes of a specific object. This means that a specific class (`Door`) has a signal (`door_opened`) which might be emitted on its state change (i.e door goes from `closed` state to `open`).

This scope naturally represents **one-to-one/one-to-many** relationship between the publisher and subscriber:

- Multiple Door instances each have their own independent door_opened signals. The same applies if they inherit from a BaseDoor class.
- The number of subscribers can vary: An SFXSystem might subscribe to play a sound, while a VFXSystem subscribes to trigger a dust effect, etc.

This scope is used when subscriber has a relation to the publisher which can be 'described' in your code or tree hierarchy (subscriber will be accessing the the publisher's signal in order to connect to it).

Let's add another use case besides the door example: consider a UI where pressing an "Options" button opens a submenu. The button's identity matters (it's not the "Exit" or "New Game" button). The subscriber (e.g., OptionSubmenuLoader) needs to connect to that exact button's signal in order to open the sub-menu. They are both probably a part of the same UI options menu.

### 🌎 Global scope

This scope comes into play, when you don't have a direct access between the publisher and subscriber (e.g. they do not belong to the same scene).

Instead of belonging to a specific instance, global signals are declared in a dedicated third-party class that is independent from publisher or subscriber.

These signals typically represent a *many-to-one/many-to-many relationships*, since multiple different publishers can emit the same event.

Button example variation: Pressing any button triggers a generic click sound. The sound player doesn't care which specific button was pressed (Options, New Game, or Exit) or which menu it belongs to; it might not even be tied to the UI logic at all.

Achievements example: Every door simply emits a global door_opened signal. Achievements service subscribes to only this one event. It doesn't care which door has been opened (and doesn't know that Door class exists).

This setup is closer to typical Event-Driven Architecture where components are independent and separated.

## Global scope in details

### Implementation via bus event

Godot features a concept called Autoloads, which acts as the engine's version of the Singleton pattern. An Autoload is initialized once and remains globally accessible from anywhere in your code. We can create an Autoload named GlobalSignals that strictly holds our global signal definitions:

```gdscript
signal door_opened
signal button_pressed
```

Returning to our SteamAchievements example, the door's code will look like this:

```gdscript
class_name Door

signal door_opened

func open_door():
	# ...
	GlobalSignals().door_opened.emit() # <- global-scoped signal
	door_opened.emit() # <- object-scoped signal.
```

This can be seen as a simple implementation of the [event bus pattern](https://www.enterpriseintegrationpatterns.com/patterns/messaging/MessageBus.html)

> Notice that the object-scoped door_opened signal is still there. SFXSystem still needs it: in case of subscribing to the global `GlobalSignal.door_opened`, one opened door would trigger opening sound on every existing door in the level
{: .prompt-mug }
<!-- lint fight -->
> The same idea is described in the official tutorial [comment](https://github.com/godotengine/godot-docs-user-notes/discussions/5#discussioncomment-8124099) by [**samuelfine**](https://github.com/samuelfine).
Given the number of reactions it got, I guess that people really liked this approach.
{: .prompt-info }

### All the dangers

Global scoped signals come with caveats that require careful handling.

#### Ruins architecture

The biggest trap of global signals is that they are too easy to use. You might be tempted to skip designing proper abstraction layers, function interfaces, or dependency injection  and just throw a global signal at the problem. Object-scoped signals inherently require some structural relationship between components, but global signals let you bypass that discipline entirely.

Here we just add four lines of code:

  - GlobalSignals:
    - `signal another_signal(any_data)`
  - SystemBob:  
    - `GlobalSignal.another_signal.connect(_on_another_signal)`
    - `func _on_another_signal(any_data): pass`
  - SystemAlice:
    - `GlobalSignal.another_signal.emit(["hi", "bob"])`

> In EDA, new event type typically requires a lot of efforts: you think about event protocols, event routing, event payload, configuring publisher and subscriber, etc.
{: .prompt-mug }

#### One-to-many pitfall

Consider a scenario where enemies stop attacking when the player dies. Emitting a global player_died signal is fast and effective, especially since enemies and the player usually lack direct references to each other. It seems logical if there is only one main character. However, the problem arises if you ever add split-screen or multiplayer: a single player's death will instantly freeze every enemy on the map.

Of course, adding a second main character involves a massive game redesign anyway, so this specific example is extreme. But idea is that using the global scope is fine for a singleton publisher, but you must be absolutely certain that publisher is a true singleton by nature, and that a second instance won't be required later.

#### Broadcasting pitfall

With global signals, it is common to include a payload since the publisher and subscriber lack direct references to one another.

Imagine an analytics manager tracking which menu buttons a player presses most frequently. To facilitate this, every button could emit a global button_pressed signal containing its unique ID as a payload.

This doesn't scale well. Every single button press across the entire game will trigger the OptionSubmenuLoader. This forces the subscriber to constantly filter irrelevant data, leading to wasted performance and a convoluted signal topology.

> Essentially, we gave up on the natural "Event routing" ability of the signals and force a broadcast model: every subscriber receives every signal and must manually filter the information. This is not necessary a bad design, if your aware of the pros and cons.
{: .prompt-mug }

### Comparison to Event-Driven Architecture

***Event bus***

In EDA it is typical that components does not know about each other but have a "middle man" to spoke to. This third layer is usually a **Event Bus** or **Event Broker** variation.

Global scoped signals resemble this model: publisher and subscriber uses a GlobalSignals autoload to communicate.

***Event Routing***

Common middleman feature is called **Event Routing**. Our bus has zero logic, yet signals provide this by design. I discussed it here.

***Pulling after receiving event***

In EDA it is common for subscriber to make a direct request to publisher after receiving an event in order to better understand the context of it. This is a known trade-off of the "thin" events that don't contain much data (in case of fat events we still can't be sure that new client subscriber would be satisfied by that).

In global-scoped signals there is no analogy for that. Practical application of this may be more carefully designed global signal's payload.

> Ironically, object-scoped relationship can apply this: since subscriber has a direct access to publisher, it can use some publisher's getters after receiving the signal.
{: .prompt-mug }

### DDD comment

Object s - inside one bounded context. Given the direct access of the subscriber to publisher, it is more likely that o s means inside one [aggregate](https://martinfowler.com/bliki/DDD_Aggregate.html).

Global s - between different bounded contexts. Components are fully independent.

**entity** object is likely to use its own signals (object scope 🚪) for domain processes, while global signals are also can be used..

**value** object uses only global scoped 🌎 signals.

You may try to differentiate your signals as domain and integrated events. I haven't thought about it much.

## Solution: Connecting signals in UI

Godot has a feature of connecting the signals via UI, which means that `connect` api is not called in the code.
Tutorials describes it [here](https://docs.godotengine.org/en/stable/getting_started/step_by_step/signals.html#connecting-a-signal-in-the-editor).

Probably that's why docs state that signals help components interact "without referencing one another"

It is a valid "UI programming level" solution, but it has several problems:

- UI connection makes it implicit: you don't know how code works outside the engine. This mean that in external IDE or repo storage (github, gitlab) such function handlers would appear unused.
  - This is important while working in VSCode. I describe it in another post.
- Commits would not reflect the difference in code (other files will be probably changed, as this connection data should be stored somewhere).
- Code refactoring may silently break the connection. With explicit connection, actions like renaming signal or moving the handler will result in a compilation error.
- Doesn't work if you create or instantiate nodes during runtime:
- > necessary when you create nodes or instantiate scenes inside of a script ([link](https://docs.godotengine.org/en/stable/getting_started/step_by_step/signals.html#connecting-a-signal-via-code)).

My opinion is that it's useful for a quick prototyping and learning, but once the things "are settled", it's better to make the connection in code (using appropriate scope we discussed). In case of a big project with several developers and lots of VCS usage, I consider UI connections very dangerous.

## Random commentary

Some related thoughts that I didn't know where to fit

### DIP comment

I talked a lot about advantages of the dependency reversal signals provide.

In our canonic example, **SFX system** started to depend on *Door*, but scenarios where **Door** has **SFX system** as a dependency are common and valid.

Consider another generic **SFX system**: it doesn't know about doors, but is good at playing sounds from some library (e.g. it maps signal name to sound type using some predefined global map).

One way to maintain it is using signals with dependency injection:

- **Door** injects its signal while initializing **SFX system**.
- Door still does not call `SFXSystem.play_sound` directly and does not really care about **SFXService** after its initialization.
- **SFX system** can be attached to every interactive object as long as it is ready to initialise sfx system with some signal.

### Local Event Bus

Imagine that the door has many signals (like `door_closed`, `door_locked` etc) and many systems depending on them. Then **DoorSignalContainer** can be created. Any system like **SFXSystem** will use this injected container in order to do its own thing. It is similar to global scope we discussed but this time the bus is not global, and have an item (a door) scope.

### 🤷‍♂️ Why using signals at all

It may seem, that signal approach is not necessary when the two components have a direct access to each other (object scope).
While this is true on a small scale, we still has all the advantages of the decoupled system. I listed main reasons here.

Another argument might be that a game app is a big monolith (at least an indie game without multiplayer features): you don't have a network connection and independent components which inherently are run on different machines. Every part can be accessed directly using global tree structure, singletons, Godot's `groups` or low level API. But this fact only increases the advantages of decoupling approaches, and event-like signals are a good candidate to apply.

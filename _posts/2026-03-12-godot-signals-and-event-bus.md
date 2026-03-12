---
title: "Godot signals and Event Bus"
description: "🚌 How to use Godot signals using the Event Bus pattern, and common pitfalls to be aware of."
categories: [Coding]
tags: [godot, godot signals, eda]
---


- [Intro](#intro)
  - [🎵 Door SFX example](#-door-sfx-example)
- [Problem: Subscriber needs a direct access to publisher](#problem-subscriber-needs-a-direct-access-to-publisher)
- [Solution: Signal scopes](#solution-signal-scopes)
  - [🚪 Object scope](#-object-scope)
  - [🌎 Global scope](#-global-scope)
- [About Global Scope](#about-global-scope)
  - [Implementation via event bus](#implementation-via-event-bus)
  - [All the dangers](#all-the-dangers)
  - [Comparison to Event-Driven Architecture](#comparison-to-event-driven-architecture)
  - [DDD comment](#ddd-comment)
- [Connecting signals in UI](#connecting-signals-in-ui)
- [Random commentary](#random-commentary)
  - [DIP comment](#dip-comment)
  - [Local Event Bus](#local-event-bus)
  - [🤷‍♂️ Why use signals at all](#️-why-use-signals-at-all)

> Official docs: [link](https://docs.godotengine.org/en/stable/classes/class_signal.html)
> Official tutorial: [link](https://docs.godotengine.org/en/stable/getting_started/step_by_step/signals.html)
{: .prompt-book }
<!-- lint fight -->
> I discussed how Event-Driven Architecture (EDA) can be applied to Godot signals [here]({% post_url 2026-03-06-godot-signals-and-eda %}).
{: .prompt-book }

![alt text](/assets/img/posts/godot_signal_bus/bus.png)

## Intro

Godot signals are a powerful tool for decoupling systems in your project.

How official docs describe them:
> Signals are a delegation mechanism built into Godot that allows one game object to react to a change in another ***without them referencing one another***.

However, this zero-reference promise isn't fully realized in examples.

In this post I discuss:

- A reference problem and how it can be solved via event bus.
- New dangers that appear with a new approach.
- Further EDA comparisons and some other technical commentary.

### 🎵 Door SFX example

#### Simple direct call

{% include shared/door_example_direct_call.md %}

#### Decoupling using Godot's signals

{% include shared/door_example_signal_version.md %}

> In EDA terms, the system that emits an event is called **Publisher**, and the system listening to events - **Subscriber**.
> I [will be using]({% post_url 2026-03-06-godot-signals-and-eda %}#event-related-terms-we-will-borrow) these terms.
{: .prompt-info }

## Problem: Subscriber needs a direct access to publisher

EDA typically handles interactions between fully decoupled systems.
But in our example a subscriber still holds a direct reference to the publisher:

```gdscript
class_name SFXSystem

var door: Door # <-- here!

func _ready():
	door.door_opened.connect(_on_door_opened)
```

If the door and its dependent systems are in the same scene, this tight coupling is fine.

But imagine another scenario:

An achievements service tracks how many doors the player opens during playthrough. Naturally, this service doesn't have references to every door in the game, and it probably doesn't even know that the `Door` class exists.

This situation is closer to typical EDA: the achievements service is independent of the door class, but they still need to communicate. How do we establish this connection?

## Solution: Signal scopes

To solve this, let's distinguish between two structural approaches which I call 'signal scopes': object scope and global scope.
The door example falls into the object scope, so we'll break that pattern down first.

### 🚪 Object scope

![alt text](/assets/img/posts/godot_signal_bus/image-2.png)

It’s the approach shown in the official Godot tutorial and used by built-in nodes.

Object-scoped signals are attributes of a specific object. This means that a specific class (`Door`) has a signal (`door_opened`) which might be emitted on its state change.

This scope naturally represents **one-to-one/one-to-many** relationship between the publisher and subscriber:

- Multiple `Door` instances each have their own independent `door_opened` signals.
- The number of subscribers can vary: A `DoorSFXSystem` might subscribe to play a sound, while a `VFXSystem` subscribes to trigger a dust effect, etc.

This scope is used when the subscriber has a relation to the publisher which can be described in your code or tree hierarchy (subscriber will be accessing the publisher's signal in order to connect to it).

***Button example***

Consider a UI where pressing an "Options" button opens a submenu. The button's identity matters (it's not the "Exit" or "New Game" button). The subscriber (e.g., `OptionSubmenuLoader`) needs to connect to that exact button's signal in order to open the sub-menu. They are both probably a part of the same UI options menu.

### 🌎 Global scope

![alt text](/assets/img/posts/godot_signal_bus/image-3.png)

This scope comes into play when you don't have direct access between the publisher and subscriber (e.g. they are not a part of the same scene).

Instead of belonging to a specific instance, global-scoped signals are declared in a separate class that is independent from publisher or subscriber.

These signals typically represent **many-to-one/many-to-many relationships**, since multiple different publishers can emit the same event.

This setup is closer to typical EDA where components are independent and separated.

***Button example variation***

Pressing any button triggers a generic click sound. The sound player doesn't care which specific button was pressed (Options, New Game, or Exit) or which menu it belongs to; it might not even be tied to the UI logic at all.

***Achievements example***

Every door simply emits a global `door_opened` signal. Achievements service subscribes to only this one event. It doesn't care which door has been opened (and doesn't know that `Door` class exists).

## About Global Scope

### Implementation via event bus

Godot [features **Autoloads**](https://docs.godotengine.org/en/stable/tutorials/scripting/singletons_autoload.html#singletons-autoload), which act as the engine's version of the Singleton pattern: it is initialized once and remains globally accessible from anywhere in your code.

We can create an Autoload named `GlobalSignals` that holds our global-scoped signals:

```gdscript
signal door_opened
signal button_pressed
```

Returning to our achievements example, the door's code will look like this:

```gdscript
class_name Door

signal door_opened

func open_door():
	# ...
	GlobalSignals.door_opened.emit() # <- global-scoped signal
	door_opened.emit() # <- object-scoped signal
```

This can be seen as an implementation of the [event bus pattern](https://www.enterpriseintegrationpatterns.com/patterns/messaging/MessageBus.html)

> Notice that `DoorSFXSystem` still needs the object-scoped signal: in case of subscribing to the global `GlobalSignal.door_opened`, one opened door would trigger opening sound on every existing door in the level
{: .prompt-mug }
<!-- lint fight -->
> The same idea is described in the official tutorial [comment](https://github.com/godotengine/godot-docs-user-notes/discussions/5#discussioncomment-8124099) by [**samuelfine**](https://github.com/samuelfine).
Given the number of reactions it got, I think there is demand for such an approach.
{: .prompt-info }

### All the dangers

> I use **global-scoped signals** and **global signals** interchangeably. I also sometimes refer to **object-scoped signals** as **local signals**. These are just terms I came up with, but I think the semantics is intuitive.
> {: .prompt-info }

Global-scoped signals come with caveats that require careful handling.

#### Ruins architecture

![alt text](/assets/img/posts/godot_signal_bus/image-4.png)

The biggest trap of global signals is that they are too easy to use. Developer might be tempted not to design the components relationships (e.g, abstraction layers, function interfaces, SOLID patterns) and just throw a global signal at the problem. We can just add four lines of code:

  - GlobalSignals:
    - `signal another_signal(any_data)`
  - SystemBob:  
    - `GlobalSignal.another_signal.connect(_on_another_signal)`
    - `func _on_another_signal(any_data): pass`
  - SystemAlice:
    - `GlobalSignal.another_signal.emit(["hi", "bob"])`

In contrast, object-scoped signals still inherently require some structural relationship.

> In EDA, new event type typically requires a lot of effort: you think about event protocols, event routing, event payload, configuring publisher and subscriber, etc.
{: .prompt-mug }

#### One-to-many pitfall

![alt text](/assets/img/posts/godot_signal_bus/image-5.png)

Consider a scenario where enemies stop attacking when the player dies. Emitting a global `player_died` signal would be fast and effective.

The problem arises if you ever add split-screen or multiplayer: a single player's death will freeze every enemy on the map.

Of course, adding a second main character involves a massive game redesign anyway, so this specific example is extreme. But the idea is that using the global scope is fine for a singleton publisher, but you must be certain that the publisher is a true singleton by nature, and a second instance won't be required later.

#### Broadcasting pitfall

![alt text](/assets/img/posts/godot_signal_bus/image-6.png)

> On emitting signal, you can pass additional data. These are just arguments of the function call. I call it payload, while it's [not quite the same]({% post_url 2026-03-06-godot-signals-and-eda %}#signals-are-not-data-but-may-transfer-it).
{: .prompt-mug }

With global signals, it is common to include a payload since the publisher and subscriber are independent.

Imagine an analytics system that tracks which menu buttons a player presses most frequently. Every button could emit a global `button_pressed` signal containing its unique button ID as a payload.

This is fine, but a tricky problem can emerge later: developers might start relying on this global signal for what should be object-scoped scenarios. Remember the earlier example where an "Options" button opens a submenu? The `OptionSubmenuLoader` connected directly to that specific button. Now, a developer might just use the global `button_pressed` signal instead, filtering the events to open the menu only if `button_id == "OptionsButton"`.

This doesn't scale well. Every button press across the game will trigger the `OptionSubmenuLoader`. This means that the subscriber will be constantly filtering irrelevant data, leading to wasted performance and a convoluted signal topology.

> Essentially, we gave up on the natural ["Event routing" signal ability]({% post_url 2026-03-06-godot-signals-and-eda %}#signals-imply-event-routing) and force a broadcast model: every subscriber receives every signal and manually filters the information. This is not necessarily a bad design, if you are aware of the pros and cons.
{: .prompt-mug }

### Comparison to Event-Driven Architecture

***Event bus***

In EDA it is typical that components don't know about each other but have a "middle man" to speak to (comes in many shapes: event bus, event queue, event broker, etc). Global-scoped signals resemble this model: publisher and subscriber use a `GlobalSignals` autoload to communicate.

***Event Routing***

Common middleman feature is called **Event Routing**. Our bus has zero logic, yet signals provide this by design. I discussed it [here]({% post_url 2026-03-06-godot-signals-and-eda %}#signals-imply-event-routing).

***Pulling after receiving event***

In EDA it is common for a subscriber to make a direct request to publisher after receiving an event in order to better understand the context of it. This is a known trade-off of the ["thin" events](https://www.thoughtworks.com/insights/blog/architecture/thin-events-the-lean-muscle-of-event-driven-architecture) that don't contain much data.

In global-scoped signals there is no analogy for that. It probably hints that the payload must be designed more carefully upfront.

> Ironically, object-scoped relationship can support this: since subscriber has a direct access to publisher, it can use some publisher's getters after receiving the signal.
{: .prompt-mug }

### DDD comment

Object-scoped signals - operate inside a single Bounded Context. Because the subscriber has direct access to the publisher, this usually represents communication within an [Aggregate](https://martinfowler.com/bliki/DDD_Aggregate.html).

Global-scoped signals - used to communicate between different Bounded Contexts. The systems are fully independent and unaware of each other's logic.

## Connecting signals in UI

![alt text](/assets/img/posts/godot_signal_bus/image-7.png)

Godot has a feature of connecting the signals via UI, which means that `connect` api is not called in the code.
Official tutorial describes it [here](https://docs.godotengine.org/en/stable/getting_started/step_by_step/signals.html#connecting-a-signal-in-the-editor).

This is likely why the docs state that signals help components interact "without referencing one another".

It is a valid "UI programming level" solution, but it has several problems:

- UI connection makes it implicit: you don't know how code works outside the engine. This means that in external IDE or repo storage (like github) such function handlers would appear unused. I wrote about it [here]({% post_url 2026-03-10-godot-and-vscode %}#️-godot-built-in-ui-features-that-vscode-lacks).
- Commits would not reflect the difference in code (some other resources will be changed, as this connection data should be stored somewhere).
- Code refactoring may silently break the connection. With explicit connection, actions moving the handler will result in a compilation error.
- Doesn't work if you create or instantiate nodes during runtime:
  > necessary when you create nodes or instantiate scenes inside of a script ([link](https://docs.godotengine.org/en/stable/getting_started/step_by_step/signals.html#connecting-a-signal-via-code)).

My opinion is:

- It is useful for a quick prototyping and learning, but once the things "are settled", it's better to make the connection in code (using appropriate scope we discussed).
- In a large project with multiple developers and heavy VCS usage, UI connections become a major risk.

## Random commentary

Some related thoughts that I didn't know where to fit.

### DIP comment

I've discussed the dependency reversion that signals provide here: [link]({% post_url 2026-03-06-godot-signals-and-eda %}#-door-sfx-example), [link]({% post_url 2026-03-06-godot-signals-and-eda %}#finding-eda-pattern)

In our canonical example, the SFX system depends on the door. However, scenarios where the `Door` depends on the `DoorSFXSystem` are also common and valid.

Consider a generic `SFXSystem`: it doesn't know about doors, but is good at playing sounds from a library (e.g. it maps a sound type to audio stream file using a predefined global map).

You can maintain this decoupling by combining signals with dependency injection:

- The `Door` injects its signal while initializing `SFXSystem`.
- The door still does not call `SFXSystem.play_sound` directly and ignores the sfx system entirely after initialization.
- Such sfx system can be attached to any other interactive object, provided that object initializes it with the appropriate signal.

### Local Event Bus

Imagine the door has many signals (`door_closed`, `door_locked` etc) and multiple systems depend on them. Then a `DoorSignalContainer` can be created. Dependent systems, like the SFXSystem, would use this injected container to handle their logic. It is similar to the global scope we discussed but this time the bus is local, scoped strictly to a specific item (the door).

### 🤷‍♂️ Why use signals at all

It may seem that the signal approach is not necessary when the two components have direct access to each other (object scope).
While this is true on a small scale, we still have all the advantages of the decoupled system. I listed main reasons [here]({% post_url 2026-03-06-godot-signals-and-eda %}#why-direct-call-dependency-might-be-undesired).

Another argument might be that a game app is a big monolith (at least an indie game without multiplayer features): You don't have network boundaries or independent components running on different machines. Every part can be accessed directly using global tree structure, singletons, Godot's [**Groups**](https://docs.godotengine.org/en/stable/tutorials/scripting/groups.html#groups) or low level API. But the absence of these boundaries only makes deliberate decoupling more important, not less.

---

![alt text](/assets/img/posts/godot_signal_bus/image-8.png)

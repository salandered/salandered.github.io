---
title: "Reversing the flow: Godot Signals through an EDA lens"
description: Understanding the basic usage of Godot's signals from an Event-Driven Architecture perspective.
categories: [Coding]
tags: [godot, godot signals, events, eda]
---

- [📋 About](#-about)
  - [My motivation](#my-motivation)
  - [Who is this article for?](#who-is-this-article-for)
- [☄️ Introduction to signals](#️-introduction-to-signals)
  - [🎵 Door SFX example](#-door-sfx-example)
- [🆚 Signal is not quite an event](#-signal-is-not-quite-an-event)
  - [Currently looks just like it](#currently-looks-just-like-it)
  - [Let's try to be specific, though](#lets-try-to-be-specific-though)
  - [Let's prepare](#lets-prepare)
  - [Signals are not asynchronous events](#signals-are-not-asynchronous-events)
  - [Other differences: event routing, event payload](#other-differences-event-routing-event-payload)
  - [Summary of Differences](#summary-of-differences)
- [🏛️ What software pattern we actually use](#️-what-software-pattern-we-actually-use)
  - [Observer](#observer)
  - [Finding EDA pattern](#finding-eda-pattern)
  - [Observer + Event Notification](#observer--event-notification)
- [👋 Let's wrap](#-lets-wrap)

> - Official docs: [link](https://docs.godotengine.org/en/stable/classes/class_signal.html)
> - Official tutorial: [link](https://docs.godotengine.org/en/stable/getting_started/step_by_step/signals.html)
{: .prompt-book }

## 📋 About

This blog post is about understanding the basic usage of Godot's signals from an Event-Driven Architecture perspective.

### My motivation

I have a backend development background (distributed systems, microservices, all this stuff), but never worked with game engines.
I started learning Godot less than a year ago, and for most of that time I was hesitant to use Godot signals. They felt like a scary hack that ruins architecture.
I saw that some more experienced Godot creators share the same skepticism, as well as a couple of projects where signal usage damaged the readability (in my opinion).

At the same time, Godot promotes them as one of the key features, interactions with built-in classes often imply signal usage and they are firmly integrated into the UI.
Also a game application is a monolith, which means that if we don't decouple our components, no one else will. And a signal looks like an event that can be sent... can't be that bad, right?

Well, I started using them, trying to mix documentation practices and my Event-Driven Architecture knowledge. The resemblance to EDA is sometimes metaphorical (we will see it), but Godot signal is commonly referred to as an event, and this is an interesting perspective.

Note that I only cover most basic signal usage (i.e the one described in Godot docs). It turned out to be more than enough for a couple of posts.

> I usually refer to Event-Driven Architecture as **EDA**.
{: .prompt-info }

### Who is this article for?

I wrote the key parts with such an audience in mind: "Understands software patterns, in particular EDA; noob at Godot".
But I actually explain a lot about Event-Driven Architecture itself, while also addressing common misconceptions about signals in the Godot community (from my experience).
I also explain some basic software principles along the way, as well as why signals are a powerful feature.

So overall the entry barrier is low - something here should be useful regardless of skill level.
And if you are experienced with all these concepts, you are welcome to point at my mistakes.

Also this blog post can be seen as a theoretical preparation to another post where I discuss more practical examples.

## ☄️ Introduction to signals

We will start with a basic example.

![alt text](/assets/img/posts/godot_signals/door_open.png)

### 🎵 Door SFX example

> I use GDScript in code snippets. It is illustrative and concise. It shouldn't be a problem if you are not familiar with it, but
> note that C# may [look differently](https://docs.godotengine.org/en/stable/tutorials/scripting/c_sharp/c_sharp_differences.html#doc-c-sharp-differences),
> in particular, signals are implemented via delegates.
{: .prompt-info }

#### Simple direct call

Imagine we have a `Door` class that represents an interactive object in game and a `DoorSFXSystem` class whose responsibility is playing different door sounds.
If a player opens the door, we want to play a creaking sound.

This is a basic object relationship:

```gdscript
class_name DoorSFXSystem

func play_sound():
	# playing sound
```

```gdscript
class_name Door

var sfx_system: DoorSFXSystem

func open_door():
	# some opening logic
	sfx_system.play_sound()
```

> We are interested in code relationships, but for a full picture note that In Godot it would also mean that we have
> a Door [scene](https://docs.godotengine.org/en/stable/getting_started/introduction/key_concepts_overview.html#scenes),
> where `Door`'s code is root node and `DoorSFXSystem`'s script is a child node.
{: .prompt-mug }

#### Why direct call dependency might be undesired

We can see that the door depends on its sfx system. Why this can be an issue?

The first thing is that any change to `DoorSFXSystem` implementation would require the changes on the door side.
The same problem arises if we want to completely replace `DoorSFXSystem` class with, let's say, one generic `PropSFXSystem` which covers all interactive items in game.

> It is common to solve this via [DIP principle](https://en.wikipedia.org/wiki/Dependency_inversion_principle)
> (adding `SFXSystem` interface, injecting `DoorSFXSystem` implementation while initializing the door, etc).
> Here this is not our point of interest.
{: .prompt-mug }

Besides that, we need to deal with initialization, validation, and runtime safety measures:

  1. Door's `sfx_system` variable should be initialized (I didn't cover that in code). It can be a part of the door's constructor, high level manager or Godot-specific features (like `@ready` and `@export`), but either way we need to maintain this logic.
  1. This assignment should be validated (is `sfx_system` ready to use?)
  1. During runtime door also needs to keep an eye on it (is `sfx_system` still ready to use? maybe someone deleted it?).

Even with all this in place, new challenges arise:

- What if we want a silent door (no sfx system at all)?
- What if on door opening we also want another VFX system to create dust effect, achievement system to count how many doors a player has opened, and many others?
- What if we want to detach and add sfx system dynamically during the `Door`'s object life cycle?

#### How event approach solves this via decoupling

> For now, I discuss the 'event' definition, not EDA as a whole. If you are unfamiliar with it, don't worry: different related concepts will be gradually revealed throughout the article. The essence of it that we care about is described here via the door example.
{: .prompt-tip }

One way to address this design issues is to use Event-Driven Architecture (EDA) and [**Event**](https://en.wikipedia.org/wiki/Event_(computing)) concept in particular (also known as **Message** or **Notification**, depending on the context):
> In computing, an event is a detectable occurrence or change in state that the system is designed to monitor,
> such as user input, hardware interrupt, system notification, or change in data or conditions. When associated with an event handler, an event triggers a response.

This might sound scary, let's keep only key parts:
> An event is a *detectable occurrence* or *change in state*. When *associated* with an *event handler*, an event *triggers* a response.

How it applies to our door?

*"Change of state"* is that the door was opened (probably went from `closed` state to `opened`). Event represents this change.
*"Detectable occurrence"* usually means that this event is being sent (or "emitted"): this is how we know that it actually happened.
*"Event handler"* is a function of the sfx system which plays a sound, and *"associated"* means that somehow we connect our event to this *"handler"* function.
This operation is commonly called "subscribe", while, as we will see, Godot uses exactly "connect".

All together:

- Door has and may send an event about it being opened.
- Sfx system subscribes to this event during the initialization.
- During runtime sfx system would play the sound on receiving such an event.

This means that the door not only does not have a dependency on sfx system, but ***it has no idea such system exists***.

The only question is how to do this in our code.
We don't want to implement some specific EDA pattern in Godot, this would require not just a post, but a couple of books (of a questionable value).
Luckily Godot has us covered with signals.

#### Decoupling using Godot's signals

How official docs describe them:
> Signals are a delegation mechanism built into Godot that allows one game object to react to a change in another *without them referencing one another*.

This is exactly what we need!

Signals have two main functions: `emit` and `connect`.
Now `Door` has a signal as its attribute, and `DoorSFXSystem` *connects* to this signal.
The door would *emit* it instead of the direct call to sfx system.

```gdscript
class_name Door

signal door_opened

func open_door():
	# some opening logic
	door_opened.emit()
```

```gdscript
class_name DoorSFXSystem

var door: Door

func _ready():
	door.door_opened.connect(_on_door_opened)

func _on_door_opened():
	# play sound
```

We did exactly the same decoupling that was described via events.

📍 Let's pinpoint the key difference between the direct call and the signal:
the dependency between `Door` and `SFXSystem` objects ***has been reversed***: sfx system now depends on the door.
We will be referring to this fact many times.

Also note that I renamed `play_sound` function to `_on_door_opened`. This is optional and does not affect the code flow, but serves two purposes:

 - prefix `_` accents that the function is now private. This is no longer a public API that the `Door` (or any other class) can call.
 - `on_door_opened` semantics follows Godot naming convention: "_on_node_name_signal_name". Godot calls such functions **callbacks**

> How this addresses previously raised questions?
>
> - Silent door: sfx system may ignore the signal, or be completely deleted from the door scene.
> - Many dependent systems: every one would subscribe to door's event just like sfx system did.
> - Dynamic attachment of sfx system: we can add and delete sfx system's node (notice that on adding we need to perform the subscribe operation)
>
> Door's implementation stays intact in any of these cases, it just doesn't care.
>
> An attentive reader might notice, that logic on sfx system side still needs an access to the door and hence some additional logic. In my next blog post I will describe signal usage with no direct dependencies at all. But for now there are some wins at this part as well:
>
> - Only validation on initialization is needed. During runtime, the system doesn't check that its door is valid.
> - Dependent system needs to maintain only door dependency. While on the door side we were risking to maintain dozens of different systems (sfx, vfx, etc).
{: .prompt-mug }

## 🆚 Signal is not quite an event

> Term "signal" has [different well-known](https://en.wikipedia.org/wiki/Signal_(IPC)) connotations.
> I always refer to [Godot's signal](https://docs.godotengine.org/en/stable/classes/class_signal.html) in this article.
{: .prompt-info }

### Currently looks just like it

We saw that the signal represents a state change ("door has been opened"), how it solved the typical problem which requires events, and it solved it in an "EDA fashion":
Sfx system subscribed to door's signal; door sent this signal; sfx system logic was triggered by it.

We can also see that official tutorial starts with using such words:
> In this lesson, we will look at **signals**. They are **messages** that nodes emit when something specific happens to them, like a button being pressed. Other nodes can connect to that signal and call a function when the **event** occurs.

Let's also explore built-in signals (after all, door example was made up). Usually they represent some fact that happened to them, the notable change: `BaseButton` has `pressed` signal; `Node` has `child_entered_tree`;  `Node3D` has `visibility_changed()`.

If we use the word "event" broadly, then a signal can clearly be described as such.

### Let's try to be specific, though

If we use the word "event" like a term...

Firstly, there is no strict definition: it depends on the area, use cases, implementations. I tried to discuss the EDA essence of it, and signals fit this.
That being said, there are common associations with events usage, especially with EDA, and they do not fit our signals.
Also we will see that technically signal and a usual event have very little in common.

> "Event" term is being used in different areas, like [hardware interruptions](https://wiki.osdev.org/Interrupt_Service_Routines) or
> [OS loops](https://learn.microsoft.com/en-us/windows/win32/winmsg/about-messages-and-message-queues).
> Same [wiki page](https://en.wikipedia.org/wiki/Event_(computing)) also contains this "*can be implemented through various mechanisms such as callbacks, message objects, signals, or interrupts*".
> Here in article I ignore such connotations and refer to "event" in EDA sense, meaning it is at least a ["bottled"](https://youtu.be/STKCRSUsyP0?si=hSBRrEPEbe6hz1_G&t=430) first class type object.
>
> This does not mean that I imply only usage in distributed systems, though.
> Components inside one application can be very well event-driven. In fact, Godot manages user inputs [via events](https://docs.godotengine.org/en/stable/classes/class_inputevent.html#class-inputevent). Another example can be seen [here](https://youtu.be/MX2PNIzxXMc?si=DmyY1OUucaGeF4cM).
{: .prompt-info }

### Let's prepare

Some preparations before delving into details.

#### Event-related terms we will borrow

Let's start with using EDA "lenses" and borrow some terms from it (later we will see that such things should be done cautiously).

It is common to call a system that emits the event a **Publisher** and system that listens to such events a **Subscriber**.

In our door example, `Door` is a **publisher** and `SFXSystem` is a **subscriber**.

#### New reference example

Relying only on the door example flattens our perspective, and I feel like a bit of an imposter.
Let's retell the example from the [official tutorial](https://docs.godotengine.org/en/stable/getting_started/step_by_step/signals.html#custom-signals).

Player character has a health attribute and on receiving the damage it emits a signal about health being changed.

```gdscript
class_name Player

signal health_changed(new_amount)

var health = 10

func take_damage(amount):
  health -= amount
  health_changed.emit(health)
```

This signal also has additional data (`new_amount`). We will talk about it later.

Subscriber can be a UI system which shows the player's health on the screen.

> We said that both event and signal represent a state change (door probably went from `closed` to `open`, while built-in button goes from `normal` to `pressed`).
> Does this example contradict this? We talk about some variable and not about player's state machine (i.e. discussing `player_died` signal).
>
> No, any class attribute describes the state of the object (basic OOP), and changing such attribute is a notable difference in a state.
> If other systems rely on this, having a dedicated event is perfectly fine.
>
> You can also think about it this way: in real game character stats are important, and we probably would have a dedicated class:
>
> ```gdscript
> class_name Health
> 
> var amount
>
> signal health_changed(new_value)
> ```
>
> This signal has the exact same meaning, but now its "state nature" is more illustrative and closer to our `door_opened`
{: .prompt-mug }

### Signals are not asynchronous events

Ok, let's explore what's actually problematic about our extensive usage of the word "event".

Event-Driven Architecture may mean [a lot of different things](https://youtu.be/STKCRSUsyP0?si=AIqlRfyynrLzspT8), but it is common to associate it with asynchronous interactions.
It is actually one of the [main selling points](https://www.ibm.com/think/topics/event-driven-architecture):
> EDAs enable systems to work independently and *process events asynchronously*.

If we talk strictly about events, they are just [objects](https://medium.com/aos-engineering-blog/what-is-an-event-anyway-651122f4f3e6), and implementation decides on how to operate them. [From wiki](https://en.wikipedia.org/wiki/Event_(computing)):
> The handler *may run synchronously*, where the execution thread is blocked until the event handler completes

But again, it is common to associate events with asynchronous "fire and forget" interaction: publisher sends an event and continues to do its own stuff.

This does not apply to Godot.

#### Signals are synchronous

![alt text](/assets/img/posts/godot_signals/signals_r_d_f_calls.png)

> I'm not a Godot expert, my knowledge about this part comes from empirical tests and rather humble C++ implementation research.
> This part may contain inaccuracies and misconceptions.
{: .prompt-warning }

Signals are synchronous and act just like a function call.

It's a common misconception to assume otherwise, and I think there are two reasons for that.

First one we already mentioned: EDA is associated with async interactions, and just by using the word "event" it becomes easier to forget that Godot's application
[is a monolith](https://docs.godotengine.org/en/stable/_images/godot-architecture-diagram.webp), and we don't deal with services which are being processed by their own host machines and send the events over the network (to be honest, the word "signal" doesn't really help with async associations either).

Secondly, documentation does not emphasize it (I presume this is common knowledge that client code runs using one thread?).
There is a funny proof of that, top links that pop up in the search:

- Reddit post that claims "I just learned that signals are completely asynchronous"
- Godot [forum thread](https://forum.godotengine.org/t/are-signals-fired-synchronically/80009) where confused people were making wrong assumptions, decoded C++ engine implementation and eventually get to the truth

So emitting a signal is a **synchronous operation** by default. The app's code (the one you wrote) is run by one thread.
When our door emits its signal, the game's execution thread stops, goes to the `DoorSFXSystem` to play a sound, and then returns to the `Door` code.

What that means for us?

#### Publisher halts

Firstly, publisher stops doing its thing on emitting the signal.
It will continue just after subscriber's finished processing this signal.

Obvious when you think about function calls, but the words 'emit' and 'signal' make it feel a bit weird (even if we forget about EDA).

#### Signals don't make a system less traceable

Event-Driven Architecture has a [well](https://www.confluent.io/learn/event-driven-architecture/#disadvantages) [known](https://www.reddit.com/r/programming/comments/1ngwj0l/comment/ne86gga/) [trade off](https://medium.com/@Iyanudavid/event-driven-architecture-is-easy-to-start-and-brutal-to-debug-at-scale-000d04cf8253): component interactions are hard to debug. Decoupling leads to separation of "cause and effect" and a failure can't be traced by a linear call stack. Well, you might have guessed it: this is not the case with our signals.

Let's debug this:

```gdscript
extends Node

signal test_signal

func _ready():
	test_signal.connect(_on_test_signal)
	run_sync_test()

func run_sync_test():
	print(">>> Step A: Before emit")
	test_signal.emit()
	print(">>> Step C: After emit")

func _on_test_signal():
	print(">>> Step B: Inside handler") # 🔴 breakpoint
```

![alt text](/assets/img/posts/godot_signals/dbg_stack_frames.png)
*We are inside `_on_test_signal` function; we have one thread and three stack frames*

Final output:

```console
>>> Step A: Before emit
>>> Step B: Inside handler
>>> Step C: After emit
```

#### Emitting many signals is fine

Another common misconception about signals is that you should not emit them in the loop (i.e on every engine frame).

From a design perspective it is valid, and ironically, the EDA analogy shows why: objects rarely change state on every tick.
But technically, since it is just a function call, it should not be a problem (goes without saying that in distributed systems this would be a technical hell).

There is an [GDQuest article](https://www.gdquest.com/tutorial/godot/best-practices/signals/) that addresses this directly:
> In gdscript, emitting a disconnected signal barely costs more than a function call.
> When connected to one node, emitting the signal plus its callback costs a little over three times slower than a direct functions call.
{: .normal-quote }
<!-- lint fight -->
> This article covers many interesting topics that the official tutorial lacks, but was written for Godot 3 (`March 30, 2021`).
> Signals were severely redesigned in Godot 4 (in particular, they became first-class type).
> I don't know what actually changed, so to be on a safe side I sadly wouldn't recommend to rely on this source.
{: .prompt-warning }

#### Can we make it asynchronous?

We can easily solve "publisher halts" problem.

Godot has a [call_deferred](https://docs.godotengine.org/en/stable/classes/class_callable.html#class-callable-method-call-deferred) feature.
It "schedules" any function call until the end of the current frame. We can use this on the publisher side.

Also on the subscriber side an [API flag](https://docs.godotengine.org/en/stable/classes/class_object.html#enum-object-connectflags) is available.
I haven't tested it but assume it will do the same but is "baked" into signal API to make it more handy.

> Note, that code will still be computed synchronously. But publisher can be described as it "fires and forgets".
{: .prompt-warning }

To make this truly asynchronous, we can use the [threading API](https://docs.godotengine.org/en/stable/tutorials/performance/using_multiple_threads.html).
I assume that implementing popular EDA interactions using threads would not require signals at all, though.
Either way, async signals are beyond the scope of this article.

#### Publisher-Subscriber pitfall

Since we talked about how words may bring undesired associations, I want to mention that "Publisher-Subscriber" may be described as a pattern on its own: [link](https://learn.microsoft.com/en-us/azure/architecture/patterns/publisher-subscriber), [link](https://en.wikipedia.org/wiki/Publish%E2%80%93subscribe_pattern). It may have different definitions, but, as we might expect by now, asynchronous communication is emphasized.

I still decided to use those words, but I want to illustrate that such "borrowing" should be made with caution.
Assume that you started using these terms in your documentation or code: a new developer who is not familiar with signals may be easily misguided.

> About alternatives:
>
> - We need words to describe "the one who emitted event" and "the one who listens to it". Emitter/Connector or Sender/Listener sound either abstract or awkward in my opinion.
> - Publisher/Subscriber are common: in official [tutorial comments](https://github.com/godotengine/godot-docs-user-notes/discussions/5) the word 'subscribe' is widely used (to be fair, "publish" is never used).
> - In distributed systems Producer/Consumer is a common pair, but it is mostly the same.
{: .prompt-mug }

### Other differences: event routing, event payload

All other differences are natural consequences of what we have already discussed and may seem like an unnecessary nitpicking. I decided to leave this part, as it might be helpful for those who are not familiar with EDA.

#### Signals imply Event Routing

Event is a container which holds information. All events are treated the same (i.e being passed between functions, stored in database, sent over the network),
and typically a subscriber is exposed to any event in the system. Publishers "broadcast" their events and subscribers pick only those that matter for them.

In order to distinguish events, event type attribute is commonly used:

```gdscript
class_name Event

var event_type
```

> Event has additional fields to describe what actually happened (like an object ID),
> as well as technical fields like `event_id` or `timestamp`. This is not important for us here so I greatly simplify the examples.
{: .prompt-mug }

A subscriber would filter events in order to find the one that it cares about: `if event.event_type == "door_opened"`

This does not scale well: if you have 100 different event types, `DoorSFXSystem` would be triggered for each of them, while actually working only with one.

Common solution to it is called [Event Routing](https://oneuptime.com/blog/post/2026-01-30-event-routing/view):
it directs the events to the appropriate subscribers, i.e removing the "filtering" logic from the subscriber's part and making it centralized.

In case of Godot signals, we don't need any of that. Subscriber connects to a specific event and its logic would be triggered only by that event.
Signal should not carry its type, and the routing feature happens naturally.  
In fact, just the opposite: we cannot "broadcast" a signal to all the subscribers *unless they explicitly chose to subscribe* to it.

In a sense, code line `door_opened.emit()` is a router itself.

But where is that "event type" information actually located? Event type is *just a name* that we give our signal variable *by convention*. It is not really used anywhere (unlike the event's attribute).

>Speaking of event that I showed, approach with an `var event_type` attribute is convenient because an event can be easily stored in database (represented as a row of table)
> or be sent over the network (serialized to JSON or similar structure).
> If we don't talk about distributed systems and events are used inside one application, we can "encode" event type into class type:
>
>```gdscript
>@abstract
>class_name Event
>```
>
>```gdscript
>class_name DoorOpened
>extends Event
>```
>
> This makes the event look closer to signal, while all the mentioned differences are still valid.
{: .prompt-mug }
<!-- lint fight -->
> Some features similar to event routing can be done via [Godot groups](https://docs.godotengine.org/en/stable/tutorials/scripting/groups.html). In particular, we can call all nodes in one group ("broadcasting all the subscribers" mechanic). I haven't researched this topic.
{: .prompt-tip }

#### Signals are not data but may transfer it

![alt text](/assets/img/posts/godot_signals/mr_x_opens_the_door.png)

As we saw, an event almost always carries some information (like `event_type`), as this data *is* the event.
Such essential data is commonly referred to as the [payload](https://en.wikipedia.org/wiki/Payload_(computing)), and I will be using this term:
> payload is the part of transmitted data that is the actual intended message

Usually payload is represented as a JSON-like structure. Our health example will look like this:

```gdscript
class_name Event
var payload: Dictionary # example: {"event_type": "HealthChanged", "new_amount": 5}
```

In case of signals, we saw that sometimes having just a signal defined is enough (`door_opened`).
But additional information may still be useful. Godot allows us to attach additional data while emitting a signal. We saw it with the "health" example, but let's do the same with the door.

```gdscript
signal door_opened(initiator: String) 

door_opened.emit("Mr. X")
```

Subscriber would need to define its callback accordingly:

```gdscript
func _on_door_opened(initiator: String):
	# ...
```

Here the parameter of the `emit` is just a way to pass it as an argument to the function  `_on_door_opened`.

An important thing here is that declaration of payload at `signal door_opened(initiator: String)` is optional: it is supported by some Godot UI features, but *does not affect the runtime behavior*.
What matters is a match between `emit` arguments and a callback (`_on_door_opened`) interface. Just like with a direct function call.

It means that strictly speaking `initiator: String` is not a part of signal's object and can only loosely be called a payload.

Also I like how [official tutorial](https://docs.godotengine.org/en/stable/getting_started/step_by_step/signals.html#custom-signals) playfully remarks:
> So it's up to you to emit the correct values.
{: .normal-quote }
<!-- lint fight -->
> Not only the cosmetic declaration, but the way callback interface is dependent on `emit` implies many potential problems, damaging our EDA decoupling. I am talking about this in depth in another post (link to come).
{: .prompt-warning }

### Summary of Differences

To conclude, let's list what we've discussed.

***Event***

- Is an object; Has explicit data; It *is* the data;
- Being passed in order to deliver this data to someone else.
- May be routed via other systems to optimize the delivery.
- Usually is passed asynchronously: publisher and subscriber are independent systems run by different threads (or processes)

***Signal***

- Is also an object, but this object represents a fancy direct call;
- By convention the name of this object represents the data change;
- May pass additional data using basic procedural language ability, but this data is not part of it.
- Can be seen as a router itself.
- By default is synchronous: publisher and subscriber are the same code run by one thread;

Now what makes them similar:

- Both represent an object's *state change*
- Are used to *categorize* different changes in state
- Publisher uses them to *notify other systems* about the change, i.e make it "public"
- Subscribers should be *configured to react* to such "changes" in advance
- During runtime subscribers *react* to publishers "changes" accordingly
- May have additional data associated with them

Which leads to similar use cases:

- Decouple the logic of publisher and subscriber
- Transfer the data between publisher and subscriber

All in all, I would say that Godot's signal *can be treated* as a natively built-in to language **[event pattern](https://en.wikipedia.org/wiki/Event_(computing))**.
This implies that we might use EDA "lenses" while looking at signals and their usage, but we should be aware of differences and don't let them mislead us.

## 🏛️ What software pattern we actually use

Can we pick a high-level pattern to describe what we do?

### Observer

Let's first discuss what the official tutorial tells you:
> Signals are Godot's version of the **observer pattern**. You can learn more about it in [Game Programming Patterns](https://gameprogrammingpatterns.com/observer.html)

Comparison with **observer** is understandable, but it can mislead: in **observer**, the publisher (**Subject**) manages its subscribers (**Observers**)
[[link_1](https://refactoring.guru/design-patterns/observer/python/example), [link_2](https://en.wikipedia.org/wiki/Observer_pattern)]:

Recreating [this article](https://gameprogrammingpatterns.com/observer.html) with our door example would look roughly like this:

```gdscript
@abstract
class_name Observer

@abstract
func on_notify(event: String)
```

```gdscript
@abstract
class_name Subject

@abstract
func notify(event: String)
```

```gdscript
class_name SFXSystem
extends Observer

func on_notify(event: String):
	if event == "door_opened":
		# <play sound>
```

```gdscript
class_name Door
extends Subject

var _observers: Array[Observer] = []

func notify(event: String):
	for observer in _observers:
		observer.on_notify(event)
```

While it can be said that dependency is inversed (using DIP), during runtime the `Door` subject is still dependent on its `SFXSystem` observer.
This is a common "trait" of the pattern, for example, you need to make sure that [observers are ok on the door's side](https://gameprogrammingpatterns.com/observer.html#destroying-subjects-and-observers).

But the first thing we [saw](#decoupling-using-godots-signals) is how signals helped to reverse the dependency between `Door` and `DoorSFXSystem` classes. **Observer** design directly contradicts it.

> I'm not saying the documentation is wrong:
>
> - It says "a *version* of the observer pattern."
> - DIP is involved, so from a design perspective, the relationship is reversed
> - I use the perspective of a developer using GDScript, ignoring the C++ implementation details
{: .prompt-mug }

On the other hand, **observer** actually perfectly captures signal's synchronous call nature (`observer.on_notify(event)` is indeed a direct call).
From the [wiki](https://en.wikipedia.org/wiki/Observer_pattern):
> <...> multiple observers can listen to a single subject, but the coupling is typically synchronous and direct

### Finding EDA pattern

Observer is one of the [Gang of Four](https://en.wikipedia.org/wiki/Design_Patterns) design patterns.
EDA was not a formulated term back then. Let's try to find something more suitable inside the EDA "toolbox".

Can we just declare our signals to be "event-driven"? But what does that mean, exactly?

Martin Fowler has described four main patterns of EDA: [201701-event-driven](https://martinfowler.com/articles/201701-event-driven.html).

He calls the first and most common one **Event Notification**:
> This happens when a *system sends event messages to notify other systems of a change in its domain*.
> A key element of event notification is that the *source system doesn't really care* much about the response.

This article is accompanied by a GOTO talk, where he emphasizes ([timestamp](https://youtu.be/STKCRSUsyP0?si=wMKupqLgTuDpShZp&t=416)):
> "the essence is reversal of dependencies"

Mission accomplished! **Event Notification**'s essence is what Observer's design lacked.
[In fact](https://en.wikipedia.org/wiki/Chekhov%27s_gun), this is the first thing I pinpointed in [introduction](#decoupling-using-godots-signals).

I will be referencing this pattern, but as always, it doesn't cover all the bases:

- Sounds too abstract, and can't be treated as a "coined" term (like "CQRS", for example).
- Has this strong "async" feeling to it, hence the majority of the examples in the article and talk are about microservices (while GUI events are mentioned as well).
Of course, this does not mean that we can't apply it synchronously in a monolith. But in my opinion the naming is more misleading than helpful for our case
(to be fair, "notify" can be used in synchronous sense, we just saw it in **Observer** pattern).

Also note that, of course, you can build any pattern that Martin Fowler described.
But basic signal usages (official tutorial, our examples) are synchronous implementation of the **Event Notification**.

### Observer + Event Notification

To sum up:

- **Observer** pattern misses the "reversal of dependencies" point but emphasizes the synchronous nature of signals, it is also widely recognizable.
- **Event Notification** gets the dependency reversal just right, but sounds too abstract and may imply asynchronous interactions.

## 👋 Let's wrap

Signals can be applied in an EDA fashion. A signal can be treated as an event, even though sometimes the differences are massive - all depends on the perspective: overall design, technical implementation, code level, data transfer ability, etc. And there is no one term that will capture all the intricacies that we deal with.

This is perfectly normal: both similarities and discrepancies with patterns and use cases that other smart people have established will help us better understand what we do with our signals,
and how it fits into the "global world".

It might also help to deal with official documentation imperfections and better evaluate information from the web.

In order not to close on such abstract notes, let's apply the latter right now.

Take a look at [this clip](https://youtu.be/MWHFV_BPqkA?si=_SInfWhNvBcTHf4R&t=384). Possible critiques:

- **Event Notification** traits are listed, not the **Observer**'s.
- There is a mention of nodes, that "react independently in their own threads". In case of the default signal usage in Godot this is not true.

Another example is this [video](https://youtu.be/b3uFSh3GASs?si=SAyHDv9h_JcHV1Qb) description (edited slightly):
>... we build a bridge between our character controller and the rest of the game world. Godot has powerful events routing system called signals, and it seems that collision with an enemy's sword is exactly the case for using them. But in a big game we'll either have zero signals or like 50, and *a system whose logic is built on 50 events can be hardly debugged or scaled* properly.

I would argue that:

- because of the *difference* between the signal and event, 50 signals should not be hard to debug.
- because of the *similarity* between the signal and event, they actually should help us to scale our system.

Also in my opinion signals are actually a good candidate for "bridges between character and the rest of the game world",
as EDA events are commonly used to connect independent components (to cross between bounded contexts, you might say). But this would be covered in my next post.

> I am saying this respectfully and want to note, that the first author created an open source [game template](https://github.com/catprisbrey/Cats-Godot4-Modular-Souls-like-Template)
> with custom CC0 assets, including SFX and third person animations (not mixamo!), which is truly unique. Speaking about second video mentioned, I learned more from [this channel](https://www.youtube.com/@PointDown) than anywhere else about game mechanics. This author also has [open source repositories](https://github.com/Gab-ani).
{: .prompt-mug }

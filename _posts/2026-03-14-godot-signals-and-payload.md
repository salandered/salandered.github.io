---
title: "Structuring Godot Signal Payloads"
description: "💼 The problem with typed signal arguments in decoupled systems, and a structured payload approach to fix it."
categories: [Coding]
tags: [godot, godot signals]
---


- [Intro](#intro)
  - [Adding more signals](#adding-more-signals)
  - [Adding signal with payload](#adding-signal-with-payload)
- [Problem: emit must match callback](#problem-emit-must-match-callback)
- [Solution: abstract structured payload](#solution-abstract-structured-payload)
- [Advantages](#advantages)
  - [Safety benefits](#safety-benefits)
  - [Scalability benefits](#scalability-benefits)
  - [Solving the signal declaration problem](#solving-the-signal-declaration-problem)
- [It comes at cost](#it-comes-at-cost)
  - [Less readability](#less-readability)
  - [Infrastructural code](#infrastructural-code)
- [Implementation tips](#implementation-tips)
  - [Gradual implementation](#gradual-implementation)
  - [Not too abstract](#not-too-abstract)
  - [Auto-serialization](#auto-serialization)
  - [It is not new](#it-is-not-new)

## Intro

![alt text](/assets/img/posts/godot_payload/image.png)

From the [docs](https://docs.godotengine.org/en/stable/classes/class_signal.html#class-signal):
> When calling emit() or Object.emit_signal(), the signal parameters can be also passed.

Passing parameters while emitting a signal is a common way to attach the additional context, but it can also undermine the component independence that signals are supposed to provide.

I call such parameters a signal's [**payload**](https://en.wikipedia.org/wiki/Payload), while this term should be used with caution. I've discussed this topic in [another post]({% post_url 2026-03-06-godot-signals-and-eda %}#signals-are-not-data-but-may-transfer-it).

Let's trace how a signal naturally evolves to carry a payload, and where things go wrong.

### Adding more signals

Basic door example is described [here]({% post_url 2026-03-12-godot-signals-and-event-bus %}#-door-sfx-example).

Imagine the door now opens and closes, and the sfx system needs to play the appropriate sound for each event:

```gdscript
class_name DoorSFXSystem

var door: Door

func _ready():
  door.door_opened.connect(_on_door_opened)
  door.door_closed.connect(_on_door_closed)

func _on_door_opened():
	pass

func _on_door_closed():
	pass
```

```gdscript
class_name Door

signal door_opened
signal door_closed

func open_door():
	door_opened.emit()

func close_door():
	door_closed.emit()
```

But what if the door can also be locked, broken, and exploded? Adding a new signal for every state doesn't scale well.

### Adding signal with payload

Let's take a look at a Godot's built-in `BaseButton`. Its `toggled(toggled_on: bool)` signal indicates that a change happened, but the actual state is being passed as an boolean argument.

We can apply the same approach to the door:

```gdscript
class_name DoorSFXSystem

var door: Door

func _ready():
	door.state_changed.connect(_on_door_state_changed)

func _on_door_state_changed(new_state: String):
	match new_state:
		"opened":
			pass
		"closed":
			pass
```

```gdscript
class_name Door

signal state_changed(new_state: String)

func open_door():
	state_changed.emit("opened")

func close_door():
	state_changed.emit("closed")
```

> The `Door`'s code hasn't really simplified.
> Usually as door states grow, it leads to implementing explicit state logic (e.g a state machine).
> Having a dedicated function to handle state changes gives you a single, clean location to emit the signal:
>
>```gdscript
>var current_state: String
>
>func _change_state(new_state: String):
>	current_state = new_state
>	state_changed.emit(current_state)
>```
>
{: .prompt-mug }

## Problem: emit must match callback

We will encounter an error if the arguments passed into emit do not match the signature of the callback (e.g. type mismatch or different amount of arguments).

This will break:

```gdscript
signal state_changed(new_state: String)

state_changed.emit("opened")
```

```gdscript
func _on_door_state_changed(a: int):
	pass
```

As we can see, changing the callback's interface requires updating the emit arguments.

In other words, **changing the subscriber's implementation forces a change in the publisher** (and vice versa). We are essentially giving up on the [main advantages principle]({% post_url 2026-03-06-godot-signals-and-eda %}#how-event-approach-solves-this-via-decoupling) signals helped us achieve: the reversing of dependencies and component independence.

## Solution: abstract structured payload

The core problem is that a signal under the hood acts as a function call.
Naturally, the arguments passed must strictly match the callback signature.

We can adopt an EDA approach to payloads (I described it [here]({% post_url 2026-03-06-godot-signals-and-eda %}#signals-are-not-data-but-may-transfer-it)). Instead of passing explicit arguments, we pass a unified, abstract structure:

```gdscript
signal state_changed(payload: Dictionary[String, String])
```

```gdscript
func _on_door_state_changed(payload: Dictionary[String, String]):
	pass
```

Here, the payload acts like a generic JSON-like structure that can transfer any data without forcing the callback signature to change.

## Advantages

### Safety benefits

The advantage of an abstract payload isn't just about achieving a clean EDA design.

Any function call requires you to pass the exact right arguments to avoid a runtime error. But with signals, components can be independent and know very little about each other (see this post). They can represent completely different parts of the game (e.g core mechanic and achievements service) with different developers assigned.

In such decoupled scenarios, a mismatch between the `emit` arguments and the callback signature is a severe vulnerability: it's easy to miss and it causes a runtime crash.

Replacing direct arguments with a structured payload object makes it more fault tolerant: if the subscriber doesn't find an expected key in the dictionary, it can simply log a warning and fall back to its default logic without breaking the game.

### Scalability benefits

Imagine you want to add new information to a signal.

With an usual argument-based approach, you need to update the signal declaration, every single `emit` call, and all connected callbacks simultaneously:

```gdscript
signal state_changed(new_state: String, old_state: String)

func open_door():
	state_changed.emit("opened", "closed")

func _on_door_state_changed(new_state: String, old_state: String):
	pass
```

With an abstract payload, you don't make any changes to the interfaces.
New logic will be described via how publishers and subscribers pack and unpack the payload. This also meant that you can update publishers and subscribers gradually to utilize this new information whenever they are ready.

>All benefits (and disadvantages) may seem unimportant because of the simple example I use. In a real project, an event might carry a complex structure containing both domain state and technical context:
>
>```gdscript
>signal state_changed(old_state: String, new_state: String, object_id: int, frame: int, log_context: Array[Variant])
>```
>
>Maintaining this signature across dozens of files might be a problem.
{: .prompt-mug }

### Solving the signal declaration problem

From the docs:
> The signal arguments show up in the editor's node dock, and Godot can use them to generate callback functions for you. However, **you can still emit any number of arguments when you emit signals**. So **it's up to you to emit the correct values**.

An important detail about GDScript is that the signal's argument declaration is optional. It serves as a hint for Godot's UI facilities, but during runtime you can emit anything. The code editor won't show any warnings either.

For example, this code is valid:

```gdscript
signal state_changed(new_state: String)

state_changed.emit(1)
```

```gdscript
func _on_door_state_changed(a: int):
	pass
```

And so is this:

```gdscript
signal state_changed(new_state: String)

state_changed.emit(1, "a", [1, 2])
```

```gdscript
func _on_door_state_changed(a, b, c):
	pass
```

It creates several problems:

- It is easy to forget to update it during refactoring.
- It misleads developers reading the code (especially outside the Godot editor, like in VSCode or on GitHub).

Implementing an abstract, generic payload solves these issues. You only need to stick to one reliable declaration throughout your architecture.

## It comes at cost

As always, introducing an abstraction layer comes with trade-offs: reduced readability, boilerplate code, and infrastructural overhead.
In the case of our structured payload this is especially noticeable.

### Less readability

The signal declaration no longer provides a clear, explicit look at the data it carries.

Compare the explicit version:

```gdscript
signal state_changed(new_state: String)
```

With the abstracted version:

```gdscript
signal state_changed(payload: Dictionary[String, String]) # during runtime: {"new_state": "opened"}
```

The callback signature loses the same clarity.

### Infrastructural code

> Packing and unpacking a structure is usually called more officially: [**serialization**](https://en.wikipedia.org/wiki/Serialization)
{: .prompt-mug }

Before emitting the signal, the publisher must pack the data:

```gdscript
state_changed.emit({"new_state": "opened"})
```

On receiving the signal, the subscriber must unpack it:

```gdscript
func _on_door_state_changed(payload: Dictionary[String, String]):
	var new_state = payload.get("new_state", "")
	if new_state == "opened":
		pass
```

This means the receiver now needs to know the exact string keys contained in the dictionary.
To prevent typos and maintain references, this usually leads to implementing explicit schema logic:

```gdscript
const NEW_STATE_KEY := "new_state"
```

Furthermore, in a real game, you will likely need to transfer more than just `String` values. The payload would evolve into:

```gdscript
signal state_changed(payload: Dictionary[String, Variant])
```

This also forces the subscriber to validate the variable types expected within the dictionary.

The list goes on: the amount of utils, helpers and different dictionary operations would be significantly increased throughout the project.

## Implementation tips

### Gradual implementation

You don't need to rewrite your entire codebase. This pattern can be tested between two core components in your application to see how it scales and feels in practice.

### Not too abstract

You also don't need to take the most abstract approach from the start.

Using `payload: Dictionary[String, Variant]` is just an example. Your system might rely on a specific structure that all components are already familiar with. For example, if you use events for a state machine, the payload structure could look like this:

```gdscript
class_name SignalPayload

var new_state: String
var additional_data: Dictionary

```

The majority of components will only interact with the `new_state` attribute, requiring no validation. Meanwhile, `additional_data` provides the flexibility to experiment with new features and cover edge cases without breaking the signature.

### Auto-serialization

Some objects in your game may be frequently passed through signals. Consider a `HitData` class:

```gdscript
class_name HitData

var damage: int
var weapon_id: String
```

If you notice this object is being fired across multiple combat mechanics, you can add the packing/unpacking logic directly into the class:

```gdscript
func to_dict() -> Dictionary:
	return {
		"damage": damage,
		"weapon_id": weapon_id
	}

static func from_dict(data: Dictionary) -> HitData:
	var hit = HitData.new()
	hit.damage = data.get("damage", 0)
	hit.weapon_id = data.get("weapon_id", "")
	return hit
```

Publishers and subscribers can now use these methods instead of having explicit serialization logic.

> This example illustrates an approach where passing `HitData` directly is not desired (e.g., you want to pass a copy or your payload is restricted to primitive types). Otherwise, just using `{"hit_data": hit_data_instance}` is fine.
{: .prompt-info }

### It is not new

Developers familiar with Event-Driven Architecture understand these complexities and won't be surprised by the additional effort required.

For example, the payload structure is commonly referred to as a "schema." Many languages and backend tools have popular libraries to help maintain them (handling the serialization and validation).

Also, in distributed systems, publishers and subscribers often declare an integration protocol using specifications like [AsyncAPI](https://www.asyncapi.com/en) (similar to OpenAPI for HTTP).

This means you can leverage established practices rather than inventing everything from scratch.

---

![alt text](/assets/img/posts/godot_payload/image-1.png)

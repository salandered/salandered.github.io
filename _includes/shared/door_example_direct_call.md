Imagine we have a `Door` class that represents an interactive object in game, and a `DoorSFXSystem` class whose responsibility is playing different door sounds.
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

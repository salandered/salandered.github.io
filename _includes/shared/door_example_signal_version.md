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

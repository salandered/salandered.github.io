---
title: Hello World
categories: [General]
tags: [first-post]
---


# Hello blog

hey
**hi**
_test_


##


### Auto disabling

Godot has a node which disables the process based on the camera frustum. See official docs for [VisibleOnScreenNotifier3D](https://docs.godotengine.org/en/stable/classes/class_visibleonscreennotifier3d.html).

Many objects in project use this node, sometimes with additional QoL features like auto sizing.

<details>

<summary>🔽 Expand to see screenshots 🔽</summary>
<br>
Fire node (used for torches):

![alt text](__images/vosn/fire_usual.png)

VisibleOnScreenNotifier3D cube around the fire node:

![alt text](images/vosn/fire_vosn.png)

Fire node settings: we can set cube size or auto calculate it using the omni light radius (part of the fire scene).

![alt text](images/vosn/fire_vosn_settings.png)

Similar with node that makes the parent swaying: we can set the cube size or auto calculate it using parent's AABB parameters:

![alt text](images/vosn/sway_vosn_settings.png)

</details>

## 🥧 Light baking

Project uses light baking for some specific scene.
Not all usages of it lead to any measurable performance gain (at least I haven't spotted it in profilers): sometimes it is used only for aesthetic or 🧪 scientific reasons.

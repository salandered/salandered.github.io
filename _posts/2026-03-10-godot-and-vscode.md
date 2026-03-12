---
title: "Using VSCode with Godot: Pros, Cons and Tips"
description: "🔷 I share my experience using both Godot's built-in editor and VSCode. I cover pros, cons, tips, and pitfalls."
categories: [Coding]
tags: [godot]
---

- [Short overview](#short-overview)
  - [Pros for Godot Script Editor](#pros-for-godot-script-editor)
  - [Pros for VSCode](#pros-for-vscode)
- [Details](#details)
  - [🌉 Pros and Cons of Godot-Tools](#-pros-and-cons-of-godot-tools)
  - [🔠 Code formatting](#-code-formatting)
  - [↔️ Project-Wide Renaming](#️-project-wide-renaming)
  - [🎨 IDE Extensions and Godot Addons](#-ide-extensions-and-godot-addons)
  - [🌿 Version control](#-version-control)
  - [🖥️ Godot built-in UI features that VSCode lacks](#️-godot-built-in-ui-features-that-vscode-lacks)
  - [📂 Working with file system](#-working-with-file-system)
- [VSCode extensions for working with Godot](#vscode-extensions-for-working-with-godot)
- [VSCode troubleshooting](#vscode-troubleshooting)

> Information currently only applies if you use **GDScript** for your projects.
{: .prompt-info }
<!-- lint fight -->
> Post will be updated based on feedback and new tools that I try.
{: .prompt-bolt }

## Short overview

> Some info can be applied to any IDE, but I focus on VSCode. I haven't tried [Rider](https://www.jetbrains.com/lp/rider-godot/) yet.
{: .prompt-mug }

### Pros for Godot Script Editor

Working inside Godot keeps everything in one place:

- Code is not separated from essential windows like **Inspector** or **Scene**.
- Built-in UI integration like signal connection or autocompletion icons hints. See [details](#️-godot-built-in-ui-features-that-vscode-lacks).
- Ability to make changes to project filesystem using **FileSystem** window. See [details](#-working-with-file-system).

Other notes:

- Direct access to addons from the [asset library](https://godotengine.org/asset-library/asset). See [details](#-ide-extensions-and-godot-addons)
- You don't need to manage other technologies. VSCode not only means a new app, but also an additional layer (godot-tools) between your code and engine which naturally comes with additional complexity. See [details](#-pros-and-cons-of-godot-tools).

### Pros for VSCode

Godot is a game engine and not an IDE, so it naturally lacks features:

- Tab management like split view or tab pins.
- Ability to work with textual representation of essential Godot files like `.tscn` or `.tres`
- Ability to work with files which Godot does not support (e.g. blender `.py` scripts). See [details](#-ide-extensions-and-godot-addons).
- Deep integration with version control system. See [details](#-version-control).

VSCode (via **godot-tools** extension) also provides:

- Project wide renaming. See [details](#️-project-wide-renaming)
- Code formatting. See [details](#-code-formatting)

VSCode also has a huge extensions ecosystem. See [details](#-ide-extensions-and-godot-addons) and
[recommended list for VSCode](#vscode-extensions-for-working-with-godot).

## Details

### 🌉 Pros and Cons of Godot-Tools

> Godot-tools is a necessary extension which makes VSCode interact with Godot Engine via LSP connection.
> When I mention VSCode, I am referring to VSCode/godot-tool pair.
{: .prompt-info }

The connection and running/debugging features are very robust, but it still adds a layer between you and the engine, which comes with bugs, possible sync issues and additional latency.

Also **godot-tools** lags behind engine releases. This is noticeable when new GDScript syntax features are introduced (while it does not happen often).

At the same time, it provides additional important features, see below.

### 🔠 Code formatting

> UPD: New semi-official [formatter](https://www.gdquest.com/library/gdscript_formatter/) has been developed. I haven't tried it yet.
{: .prompt-info }

I haven't found a way to do that inside Godot (without addons), so this is a big advantage for VSCode/Godot-tool pair.

But note, that godot-tools supports only the subset of  [the official style guide](https://docs.godotengine.org/en/stable/tutorials/scripting/gdscript/gdscript_styleguide.html#gdscript-style-guide). Examples:

- Does not enforce 2 empty lines padding between functions (uses 1 line)
- Ignores indentations like [this](https://docs.godotengine.org/en/stable/tutorials/scripting/gdscript/gdscript_styleguide.html#indentation) (leaves "bad" indentations unchanged).

Also it might add additional bugs. Examples:

- Recently [started to add](https://github.com/godotengine/godot-vscode-plugin/issues/972) a redundant space after `self`: `call_func(self )`
- Struggles with `@abstract` keyword ([link](https://github.com/godotengine/godot-vscode-plugin/issues/984), [link](https://github.com/godotengine/godot-vscode-plugin/issues/962#issuecomment-3905152966)).

It happened only once for me, but formatting actually may *break* the code:

```gdscript
@abstract class _Base:
	@abstract func hey()

class Derived	extends _Base:
	func hey():
		pass
```

Is formatted as:

```gdscript
class Derivedextends_Base: # <- problem
	func hey():
		pass
```

Fix:

```gdscript
class Derived \
	extends _Base:
	func hey():
		pass
```

### ↔️ Project-Wide Renaming

> I haven't found an equivalent for this feature in the built-in Godot editor, and I don't know how godot-tools performs it technically. I describe the basic usage and some empirical tests.
{: .prompt-info }

**Godot-tools** supports project wide renaming of entity names (like variables, classes and functions). This can be done via **F2** key or **Rename Symbol** context option.

Scope is generally respected: renaming function parameter from `a` to `b` does not mean that all `a` will be renamed across the whole project.

But mistakes are also common. Notable example is that when you rename a function, **all mentions of this name inside string values and comments** will be renamed. This can be both handy and catastrophic:

```gdscript
# TestClass.waiting() helps you to wait
# "waiting for godot" is a tragicomedy play first published in 1952

const default_state = "waiting"
const second_state = "waiting for godot"
const third_state = "awaiting"

func waiting():
	pass
```

Let's rename `func waiting()` to `async_waiting` using **F2**.

✔️ - expected change. ❌ - false positive.

```gdscript
# TestClass.async_waiting() helps you to wait ✔️
# "async_waiting for godot" is a tragicomedy play first published in 1952 ❌

const default_state = "async_waiting" # ❌ (probably)
const second_state = "async_waiting for godot" # ❌
const third_state = "awaiting" # ✔️ (only this line hasn't changed)

func async_waiting(): # ✔️
	pass
```

It makes me think that it depends on pattern matching besides the language semantics.

To conclude:

- Very useful for things like renaming base classes
- Pretty good for narrow scopes (local variable) and for big and unambiguous names (`awake_knight_artorias`).
- Renaming generic and short names may cause dozens of unrelated false positives across the project.
- False negatives also occur.

The best tip is to always verify the diff changes before committing.

### 🎨 IDE Extensions and Godot Addons

> Honorable mention: a Godot [addon](https://godotengine.org/asset-library/asset/2206) which makes built-in editor closer to IDE. I haven't tried it.
{: .prompt-bolt }

The IDE has a big library of actively maintained extensions or plugins. For example:

- You write project docs and want to add a markdown linter or spell checker
- You work in a team and want additional VCS features like git blame or git graph
- You want using some additional tech stack like Docker containers

Sometimes you can find alternatives in Godot asset library, but there are usually niche and could be abandoned. [link](https://github.com/turboseb/godot-git-bash-here) [link](https://github.com/brunogbrito/Godot-Hunspell) [link](https://github.com/phnix-dev/markdown-book)

On the other hand, it provides popular GDScript specific tools, that external IDE would not have.  [link](https://godotengine.org/asset-library/asset/4612) [link](https://github.com/Lcbx/GdScript2All)

Main takeaway is that using IDE adds a lot of possibilities, while all native addons can still be used.

### 🌿 Version control

I would recommend using external IDE for big projects.

Godot has an [official plugin](https://github.com/godotengine/godot-git-plugin), but it's not actively maintained.

### 🖥️ Godot built-in UI features that VSCode lacks

Godot UI allows you to drag-and-drop a node to your code script as an [`@onready` variable](https://docs.godotengine.org/en/stable/tutorials/scripting/gdscript/gdscript_basics.html#doc-gdscript-onready-annotation).
Same trick can be done with files and probably some other UI windows.

Besides that, built-in editor shows many useful UI hints.

- Icons for different types, while performing completion
- If method is overridden for both built-in and custom methods
- If function is connected to signal in UI. This one is important: **callbacks that were connected using UI look like dead code in external IDE.**

![alt text](/assets/img/posts/godot_ide/icon-hints.png){: width="700"}
*type icons*

![alt text](/assets/img/posts/godot_ide/hint-function-overridden.png)
*overridden built-in function*

![alt text](/assets/img/posts/godot_ide/hint-function-overriden-2.png)
*overridden custom function*

![alt text](/assets/img/posts/godot_ide/sig_ui_1.png)
*signal is connected*

This is not an exhaustive list, and with each new engine release new possibilities appear:
[link](https://godotengine.org/releases/4.5/#drag-and-drop-resources-in-scripts-to-preload-by-uid-instead-of-by-path)
[huge-link](https://github.com/godotengine/godot/pulls?q=is%3Apr+106341+107273+109458+108079+112174+82212+112729+112187+108342+103257+)

Using VSCode will cut you off from all these conveniences.

### 📂 Working with file system

Any changes to the file system should be done [only](https://docs.godotengine.org/en/stable/tutorials/scripting/filesystem.html#drawbacks) via Godot UI FileSystem view.
> Never move assets from outside Godot, or dependencies will have to be fixed manually

Changing file structure via IDE would lead to broken dependencies, scripts being unattached from nodes, incorrect UIDs and sometimes even a file duplication.

This could make VSCode usage cumbersome and potentially dangerous (accidentally moving a file).

## VSCode extensions for working with Godot

🦾 - recommended; ✍️ - QoL

Godot extensions:

- 🦾 [godot-tools](https://github.com/godotengine/godot-vscode-plugin).
	- **This one is necessary**. Integration with Godot Engine via LSP, debugger, formatter
- 🦾 [godot-files](https://marketplace.visualstudio.com/items?itemName=alfish.godot-files)
- ✍️ [godot-tab-formatter](https://marketplace.visualstudio.com/items?itemName=justburntpixels.godot-tab-formatter)
- ✍️ [godot theme](https://marketplace.visualstudio.com/items?itemName=JamesSauer.gdscript-theme)

Other:

- 🦾 [vscode-highlight](https://marketplace.visualstudio.com/items?itemName=fabiospampinato.vscode-highlight)
  - helps to quickly extend godot-tools code highlighting abilities or solve some quirks (like with `@abstract` keyword)
- 🦾 [error lens](https://marketplace.visualstudio.com/items?itemName=usernamehw.errorlens)
  - displays inline errors and warnings directly in the code
- ✍️ [Colorful folders](https://marketplace.visualstudio.com/items?itemName=VisbyDev.folder-path-color)
  - if you want to synchronize your fancy colorful folders between Godot and VSCode
- ✍️ [Copy Path (Unix Style)](https://marketplace.visualstudio.com/items?itemName=baincd.copy-path-unixstyle)
  - recommended for Windows for matching Godot's forward-slash path format

## VSCode troubleshooting

When updating or moving `godot.exe` **don't forget to change** `godotTools.editorPath.godot4` in VSCode settings.

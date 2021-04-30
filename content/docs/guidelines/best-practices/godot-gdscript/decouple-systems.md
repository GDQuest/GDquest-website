+++
author = "nathan"
date = "2020-01-29T08:11:34+02:00"
description = "This guide covers some best practices to write solid GDScript code, to keep a sane code-base when developing projects of growing sizes."
title = "Best Practices: Godot GDScript - Decouple Systems"
menuTitle = "Decouple Systems"
+++

This chapter looks at how to create and manage decoupled systems in Godot and take advantage of the game engine tools for building **reusable** and **independent** building blocks for our projects.

Decoupled systems:

1. **Can be tested on their own**.
1. Are **black boxes** that are fed data and produce some output. They don't care where they get the data from.
1. **Don't know anything about the outside world**.

This makes them **independent** and **reusable** across projects. This also makes them **compose**. We can create complex systems using **aggregation** by adding several decoupled nodes to a scene and using them from a parent node. This means you can remove any of the nodes below the parent and the scene will still work without errors.

For an example of this, see the Player in our [Godot Platformer 2D project](//github.com/GDQuest/godot-platformer-2d).

{{< figure src="../images/aggregation-player-scene-tree.png" >}}

{{% notice note %}}
In Object-Oriented Programming, **aggregation** is a type of **association** between objects. In the example above, the `Player` object uses other objects such as the `Hook` or the `CameraRig` but these can still work even if we remove the association with the `Player` node.
{{% /notice %}}

### Every building-block scene should run by itself without errors

**If you save a branch as an independent scene, it should run without any errors on its own**.

Godot relies on its node tree. It's a recursive data structure such that we can pick any node in the tree with all of its children and this is a tree itself. You can consider each branch of the scene tree as an independent scene.

{{< figure
    src="../images/scene_tree.png"
    caption="Screenshot of a real system scene tree layout from [OpenRPG](//github.com/GDQuest/godot-open-rpg)" >}}

In the example above, you can view each node as a separate scene, be it `Board` or `QuestSystem`.

If we save `QuestSystem` using `Save Branch as Scene` and run it locally by pressing <kbd>F6</kbd>, **it should run without any error** even if it doesn't have the same behaviour as when run as part of the `Game` scene.

{{< youtube WLYCgar9fyQ >}}

You should **never have direct references to specific objects from another system**. Instead, you should rely on a parent node to route information and let the systems interconnect via signals. We'll discuss more about signal handling in the next section: [the event bus pattern]({{< ref "event-bus.md" >}}).

In the example above, the `Game` node has a script attached to it. This script sends information from one system to another, while the `QuestSystem` or `DialogSystem` have no knowledge about any other system other than themselves for example.

### Use signals to coordinate time-dependent interactions

Godot's signals use the [Observer pattern](//gameprogrammingpatterns.com/observer.html). It's a useful tool that allows one node to react to a change in another without storing it or having a direct reference to it. Sometimes direct function calls aren't the right way to interact with objects.

We also can't always predict when an event will occur. Perhaps an animation has a random or varied duration for example. Signals allow us to react to the animation right when it finishes, regardless of its duration.

**Rely on signals when orchestrating time-dependent interactions.**

#### More tips about independent scenes ####

Here are a few ideas that could improve code maintainability and overall structure:

  - As discussed before, each scene should run locally using <kbd>F6</kbd> without error.
  - Break up complex functions into smaller functions (ideally up to 10-15 lines of code and no more) and give them descriptive names.
  - Don't store references to systems within other systems.
  - Never alter state from a function that returns a concrete type (i.e. functions that don't return `-> void`).

  Lets look at some examples of these. Take the following:

```gdscript
# Game.gd

...

func _unhandled_input(event: InputEvent) -> void:
  var move_direction := Utils.get_direction(event)
  if event.is_action_pressed("tap"):
    party_command({tap_position = board.get_global_mouse_position()})
  elif not dialog_system.is_open and move_direction != Vector2():
    party_command({move_direction = move_direction})


func party_command(msg: Dictionary = {}) -> void:
  var leader := party.get_member(0)
  if leader == null:
    return

  var path := prepare_path(leader, msg)
  party_walk(leader, path)

  var destination := get_party_destination(path)
  if destination != Vector2():
    emit_signal("party_walk_started", {"to": destination})


func prepare_path(leader: Actor, msg: Dictionary = {}) -> Array:
  var path := []
  match msg:
    {"tap_position": var tap_position}:
      path = board.get_point_path(leader.position, tap_position)
    {"move_direction": var move_direction}:
      if move_direction in board.path_finder.possible_directions:
        var from: Vector2 = leader.position
        var to := from + Utils.to_px(move_direction, board.path_finder.map.cell_size)
        if not board.path_finder.map.world_to_map(to) in board.path_finder.map.obstacles:
          path.push_back(from)
          path.push_back(to)
  return path


func party_walk(leader: Actor, path: Array) -> void:
  if leader.is_walking:
    return

  for member in party.get_members():
    if member != leader:
      path = [member.position] + path
      path.pop_back()
    member.walk(path)


func get_party_destination(path: Array) -> Vector2:
  var destination: Vector2 = path[path.size() - 1] if not path.empty() else Vector2()
  return destination
```

We can read through `party_command` without looking at the implementation of other functions because we already have a good idea about what it does. We **divided up the implementation into smaller functions** and gave expressive names to these functions.

The implementation for checking if the walk command can be issued is found the parent node `Game.gd`. Because the verification depends on the `Board` system and the `Party` object is independent, we have a sibling relationship. The validation step depends on both `Party` and `Board` and that's why it's taken care of in the level of the top `Game` node. Otherwise we'd have to introduce an interdependence between `Party` and `Board` which would tightly couple these systems.

Note how we could have had `Party` set up in such a way as to store a reference to `Board` and do all the validation checks on `path` inside of `Party`, but **this would mean that it is tightly coupled with `Board`** which is something we want to avoid.

The `prepare_path` and `get_party_destination` functions each return a concrete type: `-> Array` and `-> Vector2` respectively. Their job is to return these calculations and nothing more. **They don't alter the state in any other way**, which makes them "pure". Impure functions are much harder to track because they can change state in ways that isn't easily seen.

Now, this isn't a hard rule. For example, we may want to return a `bool` value from within a function upon the successful execution of a behaviour.

The `party_walk` function could be rewritten like this:

```gdscript
func party_command(msg: Dictionary = {}) -> void:
  var leader := party.get_member(0)
  if leader == null:
    return

  var path := prepare_path(leader, msg)
  if party_walk(leader, path):
    var destination := get_party_destination(path)
    emit_signal("party_walk_started", {"to": destination})

...

func party_walk(leader: Actor, path: Array) -> bool:
  if leader.is_walking:
    return false

  for member in party.get_members():
    if member != leader:
      path = [member.position] + path
      path.pop_back()
    member.walk(path)
  return true
```

Sometimes returning `bool` values like this in order to validate a successful execution is a decent idea.

_Keep in mind these are just guidelines!_

### Know when to break the guidelines

There are cases when you can get away with a simpler way to manage scene nodes. If the scene is intended to be one building block that's supposed to be self-contained, then we can think of a "contract" that the scene will have to honor. In this case we can use the `owner` node property to access the root node of the scene from any level. This can greatly simplify interactions within the scene. We use this approach throughout our implementation of the [State Machine system](//gameprogrammingpatterns.com/state.html) from our 2D Platformer demo:

```gdscript
extends State


var last_checkpoint: Area2D = null


func _on_Player_animation_finished(anim_name: String) -> void:
	_state_machine.transition_to('Spawn', {last_checkpoint = last_checkpoint})


func enter(msg: Dictionary = {}) -> void:
	last_checkpoint = msg.last_checkpoint
	owner.camera_rig.is_active = true
	owner.skin.play("die")
	owner.skin.connect("animation_finished", self, "_on_Player_animation_finished")


func exit() -> void:
	owner.skin.disconnect("animation_finished", self, "_on_Player_animation_finished")
```

We use the `owner` property to directly access the root node in the `Die` state. The `owner` in this case is the `Player` node.

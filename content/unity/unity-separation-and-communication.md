+++
title = "Decoupling & Communication in Unity"
date = 2023-01-29
[extra]
author = "@andystopia"
+++

## Background

As a bit of a disclaimer, I don't do a lot of game dev myself, and am mostly leaving this here as a guide to my future self so that I don't forget.


What is decoupling? It is essentially the procedure of separating implementation from behavior. It is the process of abstraction. 

I could explain more, but let's dive into an example.

Let's say that we have a game, where players and enemies have an `Inventory` and there's things on the ground that can be picked up.

We could of course, for every object that could be picked up, model a way for that object to be picked up individually, but this is tedious and sacrifices time without even the benefit of flexibility. 

So let's do something else instead. 

Let's say that when our player collides with our pickable, they try to place it in their inventory. 

To do this, let's think about what actions need to be performed. 

Unity already gives us `OnCollideEnter`, for free, so let's just use that and when we collide, we can see if there's a component that's pickable on the item. That's the basic premise, but how do we know that an object is pickable, without *constraining* how we write code at all. 

To do this, we can write what's called an interface. An interface defines what behaviors/methods are required to be implemented for a class to fully satisfy a certain behavior.  A class can later implement interfaces (like class extension, but the methods must be implemented entirely not overridden).

```cs
interface IPickable { 
	/**
	* Creates / shows the game object in the scene
	* 
	* @param position the position to place the pickable
	* @returns a game object that could be picked up later
	*/
	GameObject drop(Vector3 position);

	/**
	 * Hides / deletes the game object from the scene
	 */
	void pickup();
}
```

Note the "I" in front of `IPickable`. This is just by convention, C# devs generally write the letter "I" in front of the name of any interface. That convention is adopted here.


As an example of away to fulfill this behavior, but not the only way, it would be possible to use a prefab and `Instantiate` prefabs places, removing them or hiding them when they are picked up, but the actual mechanism doesn't matter. 


Note that for most games, pickup probably shouldn't be `void`, but rather some other interface or base class (probably some extension of `ScriptableObject`), this provides a way to display them in the inventory. [Brakey's](https://www.youtube.com/watch?v=aPXvoWVabPY) has a great, relevant video on `ScriptableObject`s, that could probably contextualize this.

Depending on our use case, our inventory class can essentially be a `List<IPickable?>` (prefilled with null where useful), or it can can be a `List` of whatever the pickup method returns. Either way could certainly work, and it just simply depends on implementation requirements.

For our player collision method, we'd want at least the following code:

```cs
void OnCollisionEnter(Collision collision) { 
	var pickable = TryGetComponent<IPickable>(collision.gameObject);
	if (pickable != null) {
		pickable.pickup();
	}
}
```

That will pick up our component, however, you will probably want to do do some sort of 1. `inventory.InsertPickable(pickable)`, *or*, 2. `inventory.PickedUp(pickable.pickup())`, depending on what the design requires. 2 will probably be used for most games moreso than 1.


For pickable, we could simply define a component for the most general case, with no special behaviors

```cs
class Pickable: MonoBehavior, IPickable { 
	/**
	* Creates / shows the game object in the scene
	* 
	* @param position the position to place the pickable
	* @returns a game object that could be picked up later
	*/
	GameObject drop(Vector3 position) { 
		// instantiate a new prefab at vector3 position
		this.transform.position = position;
		SetActive(true);
	}

	/**
	 * Hides / deletes the game object from the scene
	 */
	void pickup() { 
		SetActive(false);
	}
}
```

Now we can add the Pickable to any prefab that we would like by dragging it and dropping it in the Unity Scene Editor, and that will make it so player could collect that prefab instance and also drop it. Hopefully that last sentence carries a little weight, because it means we can essentially just mark prefabs as pickable in the Unity Scene Editor, without any changes to the codebase. If we want *special behavior* for some other prefab, we don't need to special case it in here. We can write a new class, similar to this one, but change out the body of the methods with the behavior we want, and not have to change anything about the code, and that `MonoBehavior` could be dropped onto any prefab. The player collision code cares that our class implements `IPickable` not `Pickable`, so we aren't dependent on the last code block at all. Once you're comfortable with this style, you could easily write, without any if statements, other behaviors such as pickables that never run out, pickables that can be picked up 3 times, etc. Having all your changes be additive makes it easy for other programmers to come along, and implement a new pickable, they don't need to understand how you wrote your code, they just need to implement the interface.

Whew! That was a heck of a background. We did gloss over a few aspects though, which are much harder, communication. Where did our inventory come from? How do we re-use the inventory code for enemies and players? How do we access it? What if Unity didn't provide us with something like `OnCollisionEnter` or `TryGetComponent`? What if we want a cutscene to play when we pick up a specific pickable? This was my biggest stumbling block in Unity: communication.

## Why Communicate?

We can certainly just request the type of the object that we want as a serialized field in our classes and invoke methods on that. Why shouldn't we do that? 

1. Sometimes it's okay, *but* most times, it *isn't*. However, if it's one simple thing that will probably not need to be changed, I wouldn't bother.
2. Lack of Flexibility. Let's say I start a game and I as a `Player` has a `Dog` instance in a serialize field and I can pet that dog for a slight happiness buff. Let's say later on, I want a cat instead, now it's time to refactor. I don't like refactoring, because it requires refactoring + whatever else I wanted to be doing.
3. Doesn't fit all use cases: Let's say my player could freeze bullets midair. Let's say there was a *lot* of bullets in the game. Now my player needs to hold a `List<Bullet>` and update that list depending on what bullets are in the scene. Also I need to iterate over them, whenever I freeze time. This is a repetitive pattern and annoying to get right. Oh wait, I actually want to freeze plasma rays too!
4. No official support for interfaces. Everything will be constrained by some base class. A base class  `Pettable` is restrictive for the `Dog` example, because our `Dog` might also extend something like `Mammal`, and not all `Mammal`s are `Pettable`, but we can't extend both `Mammal` and `Pettable`.


Often times, other communication patterns simply fit better, and allow us to "lift up" more game design choices into Unity Scene Editor itself, where non-programmers can make changes

## Same Object Communication

Perhaps the simplest form of communication, simply one component talking to other components on the same game object.

While it is possible to implement most entity behaviors using simply one monobehavior, this is often inflexible, and can lead to classes that take a lot to understand because they implement a lot of distinct pieces of logic. There is a concept called the [Single Responsibility Principle](https://en.wikipedia.org/wiki/Single-responsibility_principle). Every class implemented would be preferably implemented to this standard, and when possible, it should be adhered to. 

Separating out game logic into different components promotes re-use and makes reading code and implementing new features a little easier. To communicate between components, define interfaces between them for communication, that way, it's simple to change out one component for another.

For instance our main character entity could have 
 * `Player`
 * `Inventory`

Let's say our player class facilites movement and picking things up. What we probably want for our inventory is an IPlayerInventory interface, which Inventory can implement. This allows us to substitute out our inventories if we should want to do something like that in the future. We can always request components by an interface they implement by calling `GetComponent<IPlayerInventory>`. Our IPlayerInventory should at least probably support adding and dropping pickables, as well as perhaps some way to request all the items in the inventory so we can display them required as methods.

As an example, we could have an an interface,

```cs
interface IPlayerInventory<T> { 
	IEnumerable<T> RetrieveItems();
	void Pickup(T pickable);
	void Drop(T pickable);
}
```

Note that from the case discussed in the background `IPickable` is the type that satisfies T for an inventory containing pickables. 

Now in our `Player:  MonoBehaviour`

```cs
class Player: MonoBehaviour {
	private IPlayerInventory<IPickable> inventory;
	void Awake() { 
		inventory = GetComponent<IPlayerInventory<IPickable>>();
	}

	void OnCollisionEnter(Collision collision) { 
		var pickable = TryGetComponent<IPickable>(collision.gameObject);
		if (pickable != null) {
			pickable.pickup();
			inventory.Pickup(pickable);
		}
	}
}
```

Note that in this case, we can clearly see that the player doesn't require a specific inventory, it just requires one that meets the bounds required above. We can then implement an inventory that adheres to the `IPlayerInterface<IPickable>`. This style of coding clearly establishes a separation of concerns. 

Hopefully at this point, it is clear how we could swap out other implementations for inventory. 


## Peer to Peer Driven Communication

The ability for one object to reflect the state of another object, such as a hotbar reflecting an inventory is often a necessary feature for games to implement. 

As of right now, I would probably just have a hotbar component that receives an `[SerializeField] private Inventory inventory` instance and calls it's public methods directly, but if you know a good way to decouple these behaviors without using an external package, please PR!


## Multiple-Producer, Multiple-Consumer Communication

### Definitions

I invented the terms "restrictive" and "unrestrictive" to describe modes of event sending where communication is limited to those who have an instance of a certain event channel (former), and event systems where just broad listening is possible "unrestrictive".

"peer" here refers to "`GameObject`s".

### Restrictive Events

The essential premise is what I've heard other Unity devs online call the "`ScriptableObject` Channel", where scriptable object instances made in Unity are provided to Unity `MonoBehaviours`, and then messages are passed through the channel where listeners react to those events. 

There are two kinds of these as well, lazy and sequential. Pull from queue to know what's available(lazy), and react to events the moment they happen(sequential). 

So for instance, I could have an `InventoryUpdateChannel` called `invChannel` and I can say `invChannel.publish(new InventoryUpdate(7, potatoes))`. When "potatoes" are  in slot 7, and a listener can pick this up and can then update the hotbar.

Here's how the approach breaks down, 

* In the GUI
	* Create instance of message channel scriptable object for any specific channel type
	* Pass instance to receiving `MonoBehaviours`
* In the code
	* Receive instance of channel (1 line)
	* Publish events (1 line)
	* Subscript events (1 line)

This is in contrast to method calling directly calling one peer method (1 line for method call + 1 line one serialize field or worse `FindObjectOfType` = 2 lines), or if many things need to be called, it will add an additional two lines, either every time, or inside a method (now 5 lines). The channel based approach is more flexible to use in the editor, is more flexible to implement, and is shorter than method calls directly, if there are multiple listeners, but does come at a slight performance hit.


A simple way to do event based peer to peer messaging would be to follow an implementation like my own, [RadioActive](https://github.com/andystopia/RadioActive). It is sequential. It is possible to use as is, or as inspiration to write a different system. It keeps concerns separate across messaging channels and takes decent advantage of the C# type system.


### Unrestrictive Events

Sometimes, it's desirable be able to fire off, arbitrary events with perhaps a few instances associated with that, there is a package called [MessageKit](https://github.com/prime31/MessageKit). I haven't used it myself, but for most things I would argue that it is a better fit than many other solutions posted here for simpler projects. For more complex projects, longer stories, more elements, etc, I have a feeling this package will feel quite limited and would recommend writing something using `ScriptableObject`s



# Writing your own EventManager in Unity

Events help you keep objects independent of other objects in your application, making for a better object oriented design. The downside of events is that it can be difficult to follow the execution of your program, making it difficult to debug. In this section we describe how you could design an EventManager that enables you to create many invokers and listeners. Unity implements UnityEvents as a class you can inherit from to create an event:
``` c#
using System.Collections;
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.Events;

public class EnemyHitEvent : UnityEvent<int>{}
```
Now the event called ```EnemyHitEvent``` inherits the methods ```AddListener``` and ```Invoke```. The ```AddListener``` adds a delegate, a function, to the event that is called when the event is triggered. The ```Invoke``` method can be called to trigger the event. The ```EnemyHitEvent``` above is a ```UnityEvent<int>```, meaning it can call delegate function with one int parameter.

Lets say that in our imaginairy game we have the ```EnemyHitEvent``` that is triggered when the player hits an enemy. One part of the game that might be interested in this event is the Head-up Display (HUD), which displays the players score and other information. We could implement this in the following way:
``` c#
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

public class HUD : MonoBehaviour
{
    // Start is called before the first frame update
    void Start()
    {
        EventManager.AddListener(EventName.EnemyHitEvent, HandleEnemyHitEvent);   
    }

    void HandleEnemyHitEvent(int i)
	{
        Debug.Log("Enemy hit, nice! " + i);
	}
}
```
Here the HUD adds a listener to the EventManager, giving it the name of the event ```EventName.EnemyHitEvent``` and a delegate method ```HandleEnemyHitEvent```, which is a function implemented below. This function is called when the event is invoked. Here ```EventName``` is an ```enum``` that holds all the event names supported by the EventHandler. It looks something like this:
``` c#
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

public enum EventName
{
	EnemyHitEvent,
	EnemyDeathEvent
}
```
The example ```enum``` above now supports two events, one where the enemy in our imaginairy game gets hit and one where it is destroyed. The ```enum``` file above is a nice way for multiple classes to know which events are implemented by the EventManager.

Now we need gameobjects to be able to invoke the ```EnemyHitEvent```. Lets say we have an enemy in the game with the following ```EnemySimpleBehaviour``` script attached to it:
``` c#
using System.Collections;
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.Events;

public class EnemySimpleBehaviour : IntEventInvoker
{
    // Start is called before the first frame update
    void Start()
    {
		unityEvents.Add(EventName.EnemyHitEvent, new EnemyHitEvent());
		EventManager.AddInvoker(EventName.EnemyHitEvent, this);

	}

	private void OnCollisionEnter(Collision collision)
	{
		if(collision.gameObject.tag == "Bullet")
		{
			unityEvents[EventName.EnemyHitEvent].Invoke(10);
			Destroy(collision.gameObject);
		}
	}

}
```
At start the enemy will add an ```EnemyHitEvent``` to its list of events ```unityEvents``` and it will add itself in the EventManager as an invoker of the ```EnemyHitEvent``` event. When the enemy is hit by a bullet the ```OnCollisionEnter``` method is called and the event is invoked by calling ```unityEvents[EventName.EnemyHitEvent].Invoke(10)```. This ```EnemySimpleBehaviour``` has the unityEvents list because it inherits from ```IntEventInvoker```. This class inplements the unityEvents list and also an ```AddListener``` method as can be seen here:
``` c#
using System.Collections;
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.Events;

public class IntEventInvoker : MonoBehaviour
{

    protected Dictionary<EventName, UnityEvent<int>> 
        unityEvents = new Dictionary<EventName, UnityEvent<int>>();

    public void AddListener(EventName eventName, UnityAction<int> listener)
	{
		if (unityEvents.ContainsKey(eventName))
		{
			unityEvents[eventName].AddListener(listener);
		}
	}
}

```

What ties all this together is ofcourse the EventManager. This static class implements two dictionaries that hold all the invokers and the listeners. The keys of the dictionaries are the event names as defined in ```enum EventName``` and the values are a list of ```IntEventInvoker```s and ```UnityAction<int>```s. Here the ```UnityAction``` class is implemented by Unity and are delegate methods that can be called when an event is invoked. The ```EventManager``` implements four methods and a static contructor. The ```Initialize``` method fills the lists with the event names and empty lists as keys and values respectively. The other three methods lets you add an invoker, a listener or lets you remove an invoker. The ```AddInvoker``` and ```AddListener``` are implemented so that it does not matter in which order an invoker and listener are implemented. When an invoker is added, it searches through all listeners to see if there is a listener that needs to be added to it and the other way around. 

The full EventManager is implemented like this:
``` c#
using System;
using System.Collections;
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.Events;

public static class EventManager
{

	static Dictionary<EventName, List<IntEventInvoker>>
		invokers = new Dictionary<EventName, List<IntEventInvoker>>();

	static Dictionary<EventName, List<UnityAction<int>>>
		listeners = new Dictionary<EventName, List<UnityAction<int>>> ();

	static EventManager()
	{
		Initialize();
	}

	public static void Initialize()
	{
		foreach (EventName name in Enum.GetValues(typeof(EventName)))
		{
			if (!invokers.ContainsKey(name))
			{
				invokers.Add(name, new List<IntEventInvoker>());
				listeners.Add(name, new List<UnityAction<int>>());
			}
			else
			{
				invokers[name].Clear();
				listeners[name].Clear();
			}
		}
	}

	public static void AddInvoker(EventName eventName, IntEventInvoker invoker)
	{
		foreach( UnityAction<int> listener in listeners[eventName])
		{
			invoker.AddListener(eventName, listener);
		}
		invokers[eventName].Add(invoker);
	}

	public static void AddListener(EventName eventName, UnityAction<int> listener)
	{
		foreach (IntEventInvoker invoker in invokers[eventName])
		{
			invoker.AddListener(eventName, listener);
		}
		listeners[eventName].Add(listener);
	}

	public static void RemoveInvoker(EventName eventName, IntEventInvoker invoker)
	{
		invokers[eventName].Remove(invoker);
	}
}
```

This is already quite a flexible EventManager and it only misses one more functionality you might want. Now it only implements events that have delegates that receive one ```int``` argument. You might want to have events with different numbers and types of arguments. These can be implemented in a similar way as in this example.

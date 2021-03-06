---
title: Coordinate systems in Unity
description: 
author: 
ms.author: alexturn
ms.date: 2/28/2018
ms.topic: article
keywords: 
---



# Coordinate systems in Unity

Windows Mixed Reality supports apps across a wide range of [experience scales](coordinate-systems.md), from orientation-only and seated-scale apps up through room-scale apps. On HoloLens, you can go further and build world-scale apps that let users walk beyond 5 meters, exploring an entire floor of a building and beyond.

Your first step in building a mixed reality experience in Unity is to determine which [experience scale](coordinate-systems.md) your app will target.
> [!NOTE]
> 


This article has been updated for the final shipping Unity 2017.2 API shapes:
* If you are using Unity 5.6, you will see an older version of these APIs under the UnityEngine.VR namespace rather than UnityEngine.XR. Beyond the namespace change, there are other minor breaking API changes between Unity 5.6 and Unity 2017.2 that Unity's script updater will fix for you when moving to 2017.2.
* If you are using an earlier beta build of Unity 2017.2, you will see these APIs under UnityEngine.XR as expected, but you may see some differences from what is described below, as the initial 2017.2 beta builds contain an older version of the API shape.

## Building an orientation-only or seated-scale experience

**Namespace:** *UnityEngine.XR*\
 **Type:** *XRDevice*

To build an **orientation-only** or **seated-scale experience**, you must set Unity to the Stationary tracking space type. This sets Unity's world coordinate system to track the [stationary frame of reference](coordinate-systems.md#spatial-coordinate-systems). In the Stationary tracking mode, content placed in the editor just in front of the camera's default location (forward is -Z) will appear in front of the user when the app launches.

```
XRDevice.SetTrackingSpaceType(TrackingSpaceType.Stationary);
```

**Namespace:** *UnityEngine.XR*\
 **Type:** *InputTracking*

For a pure **orientation-only experience** such as a 360-degree video viewer (where positional head updates would ruin the illusion), you can then set [XR.InputTracking.disablePositionalTracking](https://docs.unity3d.com/2017.2/Documentation/ScriptReference/XR.InputTracking-disablePositionalTracking.html) to true:

```
InputTracking.disablePositionalTracking = true;
```

For a **seated-scale experience**, to let the user later recenter the seated origin, you can call the [XR.InputTracking.Recenter](https://docs.unity3d.com/2017.2/Documentation/ScriptReference/XR.InputTracking.Recenter.html) method:

```
InputTracking.Recenter();
```

## Building a standing-scale or room-scale experience

**Namespace:** *UnityEngine.XR*\
 **Type:** *XRDevice*

For a **standing-scale** or **room-scale experience**, you'll need to place content relative to the floor. You reason about the user's floor using the **[spatial stage](coordinate-systems.md#spatial-coordinate-systems)**, which represents the user's defined floor-level origin and optional room boundary, set up during first run.

To ensure that Unity is operating with its world coordinate system at floor-level, you can set Unity to the RoomScale tracking space type, and ensure that the set succeeds:

```
if (XRDevice.SetTrackingSpaceType(TrackingSpaceType.RoomScale))
{
    // RoomScale mode was set successfully.  App can now assume that y=0 in Unity world coordinate represents the floor.
}
else
{
    // RoomScale mode was not set successfully.  App cannot make assumptions about where the floor plane is.
}
```
* If SetTrackingSpaceType returns true, Unity has successfully switched its world coordinate system to track the [stage frame of reference](coordinate-systems.md#spatial-coordinate-systems).
* If SetTrackingSpaceType returns false, Unity was unable to switch to the stage frame of reference, likely because the user has not set up even a floor in their environment. This is not common, but can happen if the stage was set up in a different room and the device was moved to the current room without the user setting up a new stage.

Once your app successfully sets the RoomScale tracking space type, content placed on the y=0 plane will appear on the floor. The origin at (0, 0, 0) will be the specific place on the floor where the user stood during room setup, with -Z representing the forward direction they were facing during setup.

**Namespace:** *UnityEngine.Experimental.XR*\
 **Type:** *Boundary*

In script code, you can then call the TryGetGeometry method on your the UnityEngine.Experimental.XR.Boundary type to get a boundary polygon, specifying a boundary type of TrackedArea. If the user defined a boundary (you get back a list of vertices), you know it is safe to deliver a **room-scale experience** to the user, where they can walk around the scene you create.

Note that the system will automatically render the boundary when the user approaches it. Your app does not need to use this polygon to render the boundary itself. However, you may choose to lay out your scene objects using this boundary polygon, to ensure the user can physically reach those objects without teleporting:

```
var vertices = new List<Vector3>();
if (UnityEngine.Experimental.XR.Boundary.TryGetGeometry(vertices, Boundary.Type.TrackedArea))
{
    // Lay out your app's content within the boundary polygon, to ensure that users can reach it without teleporting.
}
```

## Building a world-scale experience

**Namespace:** *UnityEngine.XR.WSA*\
 **Type:** *WorldAnchor*

For true **world-scale experiences** on HoloLens that let users wander beyond 5 meters, you'll need new techniques beyond those used for room-scale experiences. One key technique you'll use is to create a [spatial anchor](coordinate-systems.md#spatial-anchors) to lock a cluster of holograms precisely in place in the physical world, regardless of how far the user has roamed, and then [find those holograms again in later sessions](coordinate-systems.md#spatial-anchor-persistence).

In Unity, you create a spatial anchor by adding the **WorldAnchor** Unity component to a GameObject.

### Adding a World Anchor

To add a world anchor, call AddComponent<WorldAnchor>() on the game object with the transform you want to anchor in the real world.

```
WorldAnchor anchor = gameObject.AddComponent<WorldAnchor>();
```

That's it! This game object will now be anchored to its current location in the physical world - you may see its Unity world coordinates adjust slightly over time to ensure that physical alignment. Use [persistence](persistence-in-unity.md) to find this anchored location again in a future app session.

### Removing a World Anchor

If you no longer want the GameObject locked to a physical world location and don't intend on moving it this frame, then you can just call Destroy on the World Anchor component.

```
Destroy(gameObject.GetComponent<WorldAnchor>());
```

If you want to move the GameObject this frame, you need to call DestroyImmediate instead.

```
DestroyImmediate(gameObject.GetComponent<WorldAnchor>());
```

### Moving a World Anchored GameObject

GameObject's cannot be moved while a World Anchor is on it. If you need to move the GameObject this frame, you need to:
1. DestroyImmedaite the World Anchor component
2. Move the GameObject
3. Add a new World Anchor component to the GameObject.

```
DestroyImmediate(gameObject.GetComponent<WorldAnchor>());
gameObject.transform.position = new Vector3(0, 0, 2);
WorldAnchor anchor = gameObject.AddComponent<WorldAnchor>();
```

### Handling Locatability Changes

A WorldAnchor may not be locatable in the physical world at a point in time. If that occurs, Unity will not be updating the transform of the anchored object. This also can change while an app is running. Failure to handle the change in locatability will cause the object to not appear in the correct physical location in the world.

To be notified about locatability changes:
1. Subscribe to the OnTrackingChanged event
2. Handle the event

The **OnTrackingChanged** event will be called whenever the underlying spatial anchor changes between a state of being locatable vs. not being locatable.

```
anchor.OnTrackingChanged += Anchor_OnTrackingChanged;
```

Then handle the event:

```
private void Anchor_OnTrackingChanged(WorldAnchor self, bool located)
{
       // This simply activates/deactivates this object and all children when tracking changes
    self.gameObject.SetActiveRecursively(located);
}
```

Sometimes anchors are located immediately. In this case, this isLocated property of the anchor will be set to true when AddComponent<WorldAnchor>() returns. As a result, the OnTrackingChanged event will not be triggered. A clean pattern would be to call your OnTrackingChanged handler with the initial IsLocated state after attaching an anchor.

```
Anchor_OnTrackingChanged(anchor, anchor.isLocated);
```

## See Also
* [Experience scales](coordinate-systems.md)
* [Spatial stage](coordinate-systems.md#spatial-coordinate-systems)
* [Spatial anchors](spatial-anchors.md)
* [Persistence in Unity](persistence-in-unity.md)
* [Tracking loss in Unity](tracking-loss-in-unity.md)
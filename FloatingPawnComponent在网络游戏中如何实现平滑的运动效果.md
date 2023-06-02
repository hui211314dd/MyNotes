[讨论链接](https://forums.unrealengine.com/t/what-is-the-best-way-to-smooth-player-movement-for-network-hiccups-interp-ease-no-physics/368904)

Zak Middleton (Engine Programmer @ Epic Games)：

Hi Matt!

Well there are a few approaches, but I’m going to suggest one that I think will be pretty straightforward and fairly robust in most situations.

First of all, I’m assuming you’re talking about smoothing the location of other players you see when playing, not your own player. Those are two different problems really (and the first is the easier to solve).

First of all, you’ll want to be able to detect whether the client has been moved by a network update. In blueprints this is a little messy with Blueprint-only projects right now as we do not have a callback to tell you the network update happened. The most reliable way to do this is probably have your own replicated location, rotation, and possibly velocity set by the server each update, and catch the RepNotify event for that struct on the client (In blueprints select “RepNotify” under “Replication” for your variable). When the client detects that the network location from the server arrives, this is your “new” server location. You can then choose how to use this new location.

One approach (the simplest) would be to turn off the “Replicate Movement” flag on the actor, and do the interpolation yourself to the server location on the client. Turning off this flag means the server won’t do the normal replicate and change of location for the client that happens automatically. Then you could do an interpolation in the tick for the client from current position to the server replicated position, however you like (EaseIn, Lerp, etc). (By the way, make sure you are factoring in the Alpha or DeltaTime parameters to your Ease or Lerp functions!)

Alternatively, you can leave replicated movement on (meaning the root component will teleport to the server location when it’s updated), but interpolate another part of your pawn to that location instead. For instance, say you have a sphere collision root, and an attached mesh. You could let the server teleport the sphere, and update the relative location of the mesh to “undo” the server change, essentially leaving it in the spot in the world. In code this looks like this (I’ve simplified it to make it more readable I hope, even if you aren’t a programmer):

```C++
// The mesh doesn't move, but the collision root does so we have a new offset.
FVector NewToOldVector = (OldLocation - NewLocation);
float Distance = NewToOldVector.Size();
if (Distance > MaxSmoothNetUpdateDistance)
{
	// We would move too far, so make the offset zero (warp to the root)
	MeshTranslationOffset = FVector::ZeroVector;
}
else
{
	MeshTranslationOffset = MeshTranslationOffset + NewToOldVector;	
}
```

Then each tick you decay the relative offset of the mesh towards zero, meaning you try to move it back to the root position. This is similar to how the engine code does this for Characters (UCharacterMovementComponent::SmoothCorrection figures out MeshTranslationOffset based on the change in location, and SmoothClientPosition_Interpolate decays it toward zero with some simple code (again simplified):

```C++
if (DeltaSeconds < SmoothLocationTime) // SmoothLocationTime is about 0125 secs
{
	// Slowly decay translation offset
	MeshTranslationOffset = MeshTranslationOffset * (1.f - DeltaSeconds / SmoothLocationTime);
}
else
{
	MeshTranslationOffset = FVector::ZeroVector;
}
		
FVector NewRelTranslation = GetComponentToWorld()InverseTransformDirection(MeshTranslationOffset) + DefaultMeshOffset;
Mesh->SetRelativeLocation(NewRelTranslation);
```

The advantage here is that your are moving the invisible gameplay-relevant collision to exactly match the server collision as soon as possible, but slowly moving the visual representation (the mesh) over time to reduce choppiness.

Both approaches mentioned above should be compatible with continuing to update locations based on a velocity during Tick, which will further maintain the smooth look on clients.

Hope this helps!

(Note: in code, detecting the network change in location is much easier, you override PostNetReceiveLocationAndRotation() and you can record the change in location, knowing this came from a network update).
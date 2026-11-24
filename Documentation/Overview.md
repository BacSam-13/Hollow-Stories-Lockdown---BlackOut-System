# Event-Driven Lighting System

Let's break down the problem.

Inside *Hollow Stories: Lockdown*, there are multiple Blueprints that act as light sources. We needed a centralized system capable of modifying these lights in response to specific gameplay events, without having to implement the event detection and modification logic individually in every Blueprint.

To achieve this, we developed two main components:

- **BP_Blackout:** Responsible for listening to gameplay events, identifying matching Event Queries, and notifying the corresponding Actor Components.
- **AC_Light:** Responsible for handling the actual modifications to the light sources.

Let's explore how each component works.

# Event Listener (BP_Blackout)

First, we created a Blueprint called `BP_Blackout`, designed to listen for gameplay events. Whenever an event matches a specific Event Query, the Blueprint notifies the corresponding Actor Component, called `AC_Light`, within the specified class of Blueprints.

To achieve this, we used the following variables:

```text
ActorType Class_BP_Light
PSLightData LightDataArray
```

`PSLightData` is a custom struct designed to store an Event Query and other properties that determine how the lights should be modified.

### PSLightData Structure

Before exploring the event logic, let's take a look at the properties stored in `LightDataArray`:

```text
Bool ModifyLights // Determines whether the lights should be modified and whether the AC_Light Update function should be called.
Bool LightsOn // Determines whether the light intensity should be set to zero or restored to its original value.
SoundWave AudioAsset // Sound played when the event is first detected.
```

An important detail is that `ModifyLights` and `AudioAsset` work independently. This means an event can trigger its associated audio without necessarily modifying any lights.

### Event Detection

The first thing we do when the game starts is check whether the events have already been broadcast and stored in the SaveGame Data.

```text
BEGIN PLAY:
    TagContainer = SaveGameData.BroadcastedTags
    F1(TagContainer, true)
```

`F1` is a function designed to look for matches between the Event Queries stored in `LightDataArray` and the Gameplay Tags contained in `TagContainer`.

If a match is found, we call an event that handles the corresponding light properties.

This event also receives a variable called `AlreadyDone`, which determines whether the event was already registered in the SaveGame Data when `BeginPlay` started, or whether it has just been broadcast.

This distinction allows us to avoid playing the associated audio again when loading a game in which the event had already occurred.

```text
FUNCTION F1(TagContainer, AlreadyDone):
    FOR EACH LightData IN LightDataArray:
        // If the EventQuery was already sent
        IF Container TagContainer MATCHES LightData.EventQuery:
            Call EventListened(AlreadyDone, LightData)
```

### EventListened

Finally, let's explore the event responsible for interacting with the Actor Components inside the Blueprints belonging to the `Class_BP_Light` class.

```text
EventListened(AlreadyDone, LightData):
    FOR EACH Actor in GetAllActorsOfClass(Class_BP_Light):
        IF LightData.ModifyLights:
            Actor.AC_Light.Update(LightData)
        END IF
    IF NOT AlreadyDone: // We only play the audio if it´s the first time the event is listened
        PLAY LightData.AudioAsset
```

As shown above, `BP_Blackout` does not directly modify the light sources. Instead, it identifies the relevant actors and delegates the modifications to their respective `AC_Light` components.

# Actor Component (AC_Light)

This Actor Component is attached to every Blueprint that acts as a light source. Its purpose is to handle all light-related modifications requested by `BP_Blackout`, keeping the lighting logic independent from the event detection system.

First, let's explore the variables that the Actor Component gathers on its own:

```text
ActorType BP_Light // The Blueprint that contains AC_Light.
LightComponentsArray LightComponents // An array containing all light components within BP_Light.
```

### Update Function

Earlier, we mentioned the `Update` function inside `AC_Light`. This function is responsible for executing the modifications in `BP_Light` according to the information received through `LightData`.

Let's take a look at how it works:

```text
FUNCTION Update(LightData):
    FOR EACH LightC IN LightComponents:
        IF LightData.LightsOn:
            SET LightC.Intensity = OG_Intensities[LightC.Index]
        ELSE:
            SET LightC.Intensity = 0
        END IF

        SET LightC.Visibility = LightData.LightsOn
    END FOR

    FOR EACH StaticMesh IN BP_Light:
        // MI_ON and MI_OFF are materials defined in AC_Light for the Static Meshes.
        IF LightData.LightsOn:
            SET StaticMesh.Material = MI_ON
        ELSE:
            SET StaticMesh.Material = MI_OFF
        END IF
```

The `Update` function handles two different aspects of the light sources:

1. **Light Components:** Their intensity and visibility are modified according to `LightData.LightsOn`. When the lights are turned off, their intensity is set to zero. When turned on, their original intensity is restored using `OG_Intensities`.
2. **Static Meshes:** Their materials are switched between `MI_ON` and `MI_OFF`, providing a visual representation of the current lighting state.

This allows the system to modify both the actual illumination and the visual appearance of the objects associated with each light source.

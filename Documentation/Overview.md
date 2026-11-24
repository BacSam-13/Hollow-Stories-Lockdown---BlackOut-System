Let´s break down the main issue.
Inside "Hollow Stories: Lockdown", there ara many Blueprints that works as Light sources.

# Event Listener (BP_Blackout)
First, we created a Blueprint, let´s call it BP_Blackout, that is designed to listen for events, and if they match with
specific EventQuerys, the Blueprint will call the correspondent Actor Component, called "AC_Light", within an already specified class of Blueprints.
In order to achieve this, we used this variables for the Blueprint.
```text
ActorType Class_BP_Light
PSLightData LightDataArray
```
Where PSLightData is a struct we made to hold an EventQuery, and other propterties about how to modify the lights.

Now, the first thing we do once the game starts inside this blueprint, is to check if the events were already send, and stored in the SaveGameData:
```text
BEGIN PLAY:
    TagContainer = SaveGameData.BroadcastedTags
    F1(TagContainer, true)
```
F1 is a function designed to look for matches between the EventQuery in LightDataArray, and the Tag Container.
If there is a match, we call an event, that will work with the LightProperties, and it´s approach to the issue, will depend on
the variable "AlreadyDone", that tells us if the event was just sent, or if it was already on the SaveGameData once BeginPlay started.

```text
FUNCTION F1(TagContainer, AlreadyDone):
    FOR EACH LightData IN LightDataArray:
        // If the EventQuery was already sent
        IF Container TagContainer MATCHES LightData.EventQuery:
            Call EventListened(AlreadyDone, LightData)
```

Finally, let´s see the event that will interact with the ActorComponents within the Blueprints that belong to the class Class_BP_Light.
But first, let´s mention the properties inside LightDataArray that will give us the information abput how to modify the light:

```text
Bool ModifyLights // Tells us if we are indeed going to  modify the lights, and call a function, called "Update", inside the ActorComponent
Bool LightsOn // Indicates if the lights are going to be turned on/off by the end of the moficiation. This works with the Material inside each ligt.
SoundWave AudioAsset // The audio taht will be played once the event is listened
```

```text
EventListened(AlreadyDone, LightData):
    FOR EACH Actor in GetAllActorsOfClass(Class_BP_Light):
        IF LightData.ModifyLights:
            Actor.AC_Light.Update(LightData)
        END IF
    IF NOT AlreadyDone: // We only play the audio if it´s the first time the event is listened
        PLAY LightData.AudioAsset
```

# Actor Component (AC_Light)
This Actor Component is placed within every Blueprint that works as a light source , and it is intended to be afected by BP_Blackout.
First of all, let´s explore the variables that the Actor Component gathers on it´s own:

```text
ActorType BP_Light // The Blueprint where AC_Light is within
LightComponentsArray LightComponents // An array of all the light components within BP_Light
FloatArray OG_Intensities // Array to save the Intensities of LightComponents
```

Now, we mentioned a function called "Update" erly on BP_Blacout. This function is intended to execute all the modifications in the BP_Light,
according to the LightData. Let´s see how the functions inside AC_Light works:
```text
FUNCTION Update(LightData):
    FOR EACH LightC IN LightComponents:
        IF NOT LightData.LightsOn:
            SET LightC.Intensity = 0
        END IF
        SET LightC.Visibilirty = LightsOn
    END FOR

    FOR EACH StaticMesh IN BP_Light:
        // MI_ON and MI_OFF are material for the statich mesh, defined in AC_Light
        IF LightsOn:
            SET StaticMesh.Material = MI_ON
        ELSE:
            SET StaticMesh.Material = MI_OFF
        END ELSE
        END IF    
```

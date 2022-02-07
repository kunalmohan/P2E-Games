# P2E Doss Games

Intro
-----

Doss is the world's first platform where developers and creators can build and launch their own Play2Earn games. It is currently in the alpha stage, so creators would need scripting knowledge; but eventually, it will be a no-code experience.

  

The infra to ship the economics of the games to the blockchain main net is under development and we will very soon leverage that to the open-source community as well.

  

How to share new unique game ideas
----------------------------------

1.  You can share your unique game ideas as a new issue in the repository.
2.  We have an issue template attached that will make your life easier 🙂 .
3.  For making your game ideas more visible attach the `gameIdea` tag, additionally, you can attach more tags like `shooting` , `racing` , `treasure-hunt` for adding more context!

  

How to publish your games
-------------------------

Publishing your own Ply to Earn Game on Doss currently requires some scripting and programming knowledge. We are working on making it more streamlined and easier for non-programmers to publish their own game.

  

Meanwhile, if you know little bits of C# (or similar OOP language), voila! 90 percent of your work is done.

  

In order to add a new game on the Doss platform, you simply need to implement the MiniGameBase class, details of which are provided in the following sections.

  

For the in-game assets, you can use the variety of assets available in the <DOSS Asset Store/Folder> for the games you create. Each asset has some properties configured that can be sent and received upon collision with other in-game assets.

  

To make it easier to understand, we have also added a sample game in <Examples> section with a complete walkthrough.

  

  

#### GameBase class structure:

  

```
public class MiniGameBase
{
//This dictionary holds the asset_id and the corresponding smart object. This is later used to retrieve the smart objects corresponding to the entity for triggering appropriate actions when the entities collide.
    public Dictionary<string, SmartObjectInteractionBase> smartObjects = new Dictionary<string, SmartObjectInteractionBase>();


//Initiate the placement of assets and broadcast it to other players.
    public virtual void PlaceAssets(){}
    
// Initiates what inventory should be equipped by the player when the game starts.
    public virtual void EquipInventory(){}
    
// Unequips all the inventory.
    public virtual void UnequipInventory(){}
    
// Call this function if you want to enable the primary button and pass the appropriate onclick function argument.
    public void EnablePrimaryButton(string buttonText, Action onClick){}

//This is called when photon room properties are updated. Handle states for the multiplayer events such as creating assets and updating asset properties, etc. The `start_game` property is available in the change properties whenever the master client starts the game. 
    public virtual void OnRoomPropertiesUpdate(ExitGames.Client.Photon.Hashtable propertiesThatChanged){}

// This is called whenever any player properties are updated.
    public virtual void OnPlayerPropertiesUpdate(Photon.Realtime.Player targetPlayer, ExitGames.Client.Photon.Hashtable changedProps){}


// This is called on every frame. You can use this to update local properties. Any parameter which needs to be handled on each frame. For example, whenever the player dies, it is updated via this function.
    public virtual void GameUpdate(){}

// You can add additional logic that you would like to end your game.
    public virtual void GameClearObject(){}

// Call this function to create game entities like trees, cars, gems, etc.
    public EntityData AddEntity(short id, string bundle, string asset, Vector3 position, float3 scale, bool undoRedoTask = false,
        byte _colorOverlay = 0x0, byte _physicsProperties = 0xE3, byte _additionalProperties = 0xF){}

}
```

  

### Initiating Doss

1.  Clone the project.
2.  Make a new C# file in the `/Games` directory folder.
3.  Use the given template to build your own game.

####   

### Publish the game

1.  Open a pull request and fill out the template
2.  Your game is in the pipeline!

  

Sample code
-----------

```
using System.Collections.Generic;
using UnityEngine;
using Photon.Pun;
using Photon.Realtime;
using Unity.Entities;
using Unity.Mathematics;
using System;
using System.Linq;
using Newtonsoft.Json.Linq;
using Unity.Transforms;

public class Shooting: MiniGameBase
{
    List<Tuple<string, string>> bullets = new List<Tuple<string, string>>();
    List<Tuple<string, string>> obstacles = new List<Tuple<string, string>>();

    string bulletAsset = "";
    string bulletBundle = "";

    public Shooting(Dictionary<string, object> gameAssetInput) : base(gameAssetInput)
    {
        foreach (var gameAsset in gameAssetInput)
        {
            if (gameAsset.Key.Contains("obstacle")) obstacles.Add((Tuple<string, string>)(gameAsset.Value));
            if (gameAsset.Key.Contains("bullet"))
            {
                var t = (Tuple<string, string>)(gameAsset.Value);
                bulletAsset = t.Item2;
                bulletBundle = t.Item1;

                smartObjects["asset_100001"] = AssetRegistry.GetSmartObjectFromData(bulletAsset, new SmartObjectData { id = 100001, sendingProperties = 3, damageSend = 10, receivingProperties = (int)MiniGameUtils.Properties.POINT });
            }
        }
    }

    public void ResetPlayerProperties()
    {
        GameManager.instance.SwitchToFppView();
        if (entityManager.HasComponent<SmartObjectData>(GameManager.instance.playerEntity))
        {
            entityManager.SetComponentData<SmartObjectData>(GameManager.instance.playerEntity, new SmartObjectData { id = 10000 });
        }
        else
        {
            entityManager.AddComponentData(GameManager.instance.playerEntity, new SmartObjectData { id = 10000 });
        }

        smartObjects["asset_10000"] = AssetRegistry.GetSmartObject("player", GameManager.instance.playerEntity);

        Player player = PhotonNetwork.LocalPlayer;
        PhotonNetwork.LocalPlayer.SetCustomProperties(new ExitGames.Client.Photon.Hashtable { { "score", 0 } });
    }

    public override void PlaceAssets()
    {
        var assetHashtable = new ExitGames.Client.Photon.Hashtable { };
        //
        List<Vector3> assetPlaceholders = PlaceholderManager.instance.Placeholders.GetRandomItems(20);
        short id = 0;
        foreach (Vector3 assetPosition in assetPlaceholders)
        {
            if (obstacles.Count > 0)
            {
                var obstacleTup = obstacles[UnityEngine.Random.Range(0, obstacles.Count)];
                var temp = AddEntity(id, obstacleTup.Item1, obstacleTup.Item2, assetPosition, new float3(1, 1, 1), false);
                id++;
                assetHashtable.Add(temp.key, JsonUtility.ToJson(new PhotonSmartObjectDetails(smartObjects[temp.key], obstacleTup.Item2, obstacleTup.Item1, assetPosition, temp.rotation)));
            }
        }

        PhotonNetwork.CurrentRoom.SetCustomProperties(assetHashtable);

    }

    public override void OnRoomPropertiesUpdate(ExitGames.Client.Photon.Hashtable propertiesThatChanged)
    {

        if (propertiesThatChanged.TryGetValue("start_game", out object status))
        {
            ResetPlayerProperties();
            EquipInventory();
            EnablePrimaryButton("", ShootBullet);
            CountdownScreen.instance.gameObject.SetActive(true);
        }

        foreach (var prop in propertiesThatChanged)
        {
            if (((string)prop.Key).Contains("asset_") && prop.Value != null)
            {
                var temp = new PhotonSmartObjectDetails();
                JsonUtility.FromJsonOverwrite(prop.Value.ToString(), temp);
                if (!smartObjects.ContainsKey((string)prop.Key))
                {
                    if (!String.IsNullOrEmpty(temp.asset) && !String.IsNullOrEmpty(temp.bundle)) CreateAssetGame(temp);
                }
                else
                {
                    smartObjects[(string)prop.Key].onRoomPropertiesUpdate(temp);
                }
            }
        }
    }

    private void CreateAssetGame(PhotonSmartObjectDetails obj)
    {
        var assetPosition = new float3(obj.posx, obj.posy, obj.posz);
        AddEntity((short)obj.id, obj.bundle, obj.asset, assetPosition, new float3(1, 1, 1), false);
    }

    public void ShootBullet()
    {
        if (bulletAsset == "" || bulletBundle == "") return;
        ShootingManager.instance.Shoot(bulletAsset, bulletBundle);
    }

    public override void GameClearObject()
    {
        UnequipInventory();
        GameManager.instance.SwitchToTppView();
    }

    public override void EquipInventory()
    {
        base.EquipInventory();
        GameplayUIManager.instance.OnSendEquipGun();
    }

    public override void UnequipInventory()
    {
        base.UnequipInventory();
        GameplayUIManager.instance.OnSendUnequipGun();
    }
}
```

  

Contact us
----------

You can contact us for any queries at [tanmay@doss.games](mailto:tannay@doss.games) / [vishal@doss.games](mailto:vishal@doss.games) / [aniket@doss.games](mailto:aniket@doss.games)
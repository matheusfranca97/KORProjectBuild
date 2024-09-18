
# KORProtocolGame - Unity WebGL Integration

This repository contains the code for **KORProtocolGame**, a Unity-based rhythm game using WebGL to interact with the browser via JavaScript.

## Communication Between Unity WebGL and Browser

### 1. JSLib File

A custom JSLib file enables Unity to communicate with the browser. Below is the JSLib used for sending data from Unity to the parent window and notifying the browser.

```javascript
mergeInto(LibraryManager.library, {
    SendDataToParent: function(data) {
        var jsonString = UTF8ToString(data);
        console.log("JSON string received:", jsonString);

        // Parse the JSON string and log the parsed object
        try {
            var jsonData = JSON.parse(jsonString);
            console.log("Parsed JSON data:", jsonData);
        } catch (e) {
            console.error("Failed to parse JSON:", e);
        }

        // Send the parsed JSON data to the parent window
        window.parent.postMessage(jsonString, "*");
    }
});
```

This JSLib defines two functions:

- `SendDataToParent`: Receives data from Unity, processes it (e.g., logs it), and sends it to the parent window using postMessage.

### 2. Sending Score from Unity to JavaScript

In Unity, we use the `SendScoreToParent` method to send score data from Unity to the browser. Below is an example:

```csharp
[System.Serializable]
public class ScoreData
{
    public float score;
}

public class ScoreManager : MonoBehaviour
{
#if UNITY_WEBGL
    [DllImport("__Internal")]
    private static extern void SendDataToParent(string data);
#endif

    private bool hasSentScore = false;
    private ScoreManager scoreManager;

    private void SendScoreToParent()
    {
        Debug.Log("Sending score to parent.");
#if UNITY_WEBGL
        var scoreData = new ScoreData { score = scoreManager.CurrentScore };
        string jsonString = JsonUtility.ToJson(scoreData);

        // Send the score data to the parent window
        SendDataToParent(jsonString);
        hasSentScore = true;
#endif
    }
}
```

### 3. Handling JSON Data and Volume from Browser in Unity

The `BackendCommunicationManager` class handles receiving JSON data and volume data from the browser. It also uses a fallback debug JSON file in the Unity Editor when running the game outside of WebGL.

```csharp
using UnityEngine;
using System.Collections;

public class BackendCommunicationManager : MonoBehaviour
{
    [SerializeField] TextAsset debugJson;

    public void Start()
    {
        // If not running in WebGL, always use the debug JSON
#if UNITY_EDITOR
        Debug.Log("Not WebGL, using debug JSON.");
        ReceivePlayerData(debugJson.text);
#endif
    }

    public void ReceivePlayerData(string json)
    {
        if (string.IsNullOrEmpty(json))
        {
            Debug.LogWarning("No JSON data received, using debug JSON instead.");
            json = debugJson.text;  // Use debugJson as fallback
        }

        Debug.Log("Received JSON: " + json);
        StartCoroutine(WaitForBeatManager(json));
    }

    private IEnumerator WaitForBeatManager(string json)
    {
        while (BeatManager.Instance == null)
        {
            Debug.Log("Waiting for BeatManager to be initialized...");
            yield return null;  // Wait one frame before checking again
        }

        BeatManager.Instance.LoadJsonData(json);
    }
}
```

### 4. JSON Metadata for the Game

Within the repository, there is a JSON file containing the metadata for each track. The system receives a JSON with this same model to play the track. It automatically sets the following information based on the JSON:

- Track name
- Artist name
- Track duration
- Beat detection

The Unity system uses this metadata to play the correct track and synchronize the game with the beats of the song.

Here’s an example of how to send this JSON data from JavaScript to Unity:

```javascript
fetch('9726142b.json') 
    .then(response => response.json())
    .then(data => {
        const jsonString = JSON.stringify(data);
        unityInstance.SendMessage('BackendCommunicationManager', 'ReceivePlayerData', jsonString);
    })
    .catch(error => {
        console.error("Failed to load the JSON file:", error);
    });
```

A collection of notes I am making about Unity NGO to help myself in the future, and other developers who might find this.

# NetworkVariable

### Sync
These are "automatically synchronized" for late joiners, but they do not make it clear that "OnValueChanged" is **NOT called on spawn.**
You end up needing to call the method you have subscribed in OnNetworkSpawn.
Like this:
```c#
public override void OnNetworkSpawn()
{
    myNetworkVar.OnValueChanged += OnMyNetworkVarChanged;
    OnMyNetworkVarChanged(myNetworkVar.Value, myNetworkVar.Value);
}
```

### Update rate
NetworkVariables are synced at the set tickrate

# RPC (Remote Procedure Call)

### Update rate
RPCs are called **immediately**. They are not synced to the tickrate.
You can subscribe to the tickrate yourself:
```c#
NetworkManager.NetworkTickSystem.Tick += Tick;

private void Tick() {
  if (!IsOwner) return;

  // Call your RPCs here that you want synced to the Tick
}
```

### Send RPC to every OTHER client
You can specify which clients to send an RPC to with the optional arg [ClientRpcParams](https://docs-multiplayer.unity3d.com/netcode/current/advanced-topics/message-system/rpc-params/)

However, making a new ClientRpcParams object every time you want to send an RPC is slow.
Whenever a client joins I build a Dictionary that contains a ClientRpcParams for every client, and this ClientRpcParams simply contains a list of every other client id.
```c#
private static Dictionary<ulong, ClientRpcParams> _clientRpcParamsDict; 

public override void OnNetworkSpawn()
{
    NetworkManager.Singleton.OnServerStarted += Init;
}

private void Init()
{
    if (!IsServer)
    {
        return;
    }

    NetworkManager.Singleton.OnClientConnectedCallback += OnPlayerJoin;
    NetworkManager.Singleton.OnClientDisconnectCallback += OnPlayerLeave;
}

private void OnPlayerJoin(ulong clientID)
{
    BuildDictionary();
}

private void OnPlayerLeave(ulong clientID)
{
    BuildDictionary();
}

/// <summary>
/// Build a dictionary that has each client ID mapped to a ClientRpcParams with every *other* client ID in it
/// </summary>
private void BuildDictionary()
{
    _clientRpcParamsDict = new Dictionary<ulong, ClientRpcParams>();

    foreach(ulong currentId in NetworkManager.Singleton.ConnectedClientsIds)
    {
        // Get every player id except the current id
        List<ulong> otherIdsList = NetworkManager.Singleton.ConnectedClientsIds.Where(id => id != currentId).ToList();

        ClientRpcParams clientRpcParams = new ClientRpcParams
        {
            Send = new ClientRpcSendParams
            {
                TargetClientIds = otherIdsList
            }
        };

        _clientRpcParamsDict.Add(currentId, clientRpcParams);
    }
}

public bool IsClientInDictionary(ulong clientID)
{
    return _clientRpcParamsDict.ContainsKey(clientID);
}

public ClientRpcParams GetClientRpcParamsOfAllOtherClients(ulong clientID)
{
    return _clientRpcParamsDict[clientID];
}
```

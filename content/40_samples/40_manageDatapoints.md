# Manage datapoints and subscriptions

```
NOTE:
Sample are written in C# and uses nuget package RestSharp for convenience.
baseUrl for local access : http://<ipaddress>
baseUrl for cloud access : https://api.electrification.ability.abb/buildings/openbos/apiproxy/v1/gateway/<edgeid>
```

## The different type of datapoint 

When trying to read/write fieldbus value or trying to create alarm/trend/schedule on fieldbus value you will refer to the identification of datapoint.
The datapoint identification can be :
  - Datapoint instance from database
  - Protocol argument based
  - Network organization datapoint (bus datapoint)

### Datapoint instance
The datapoint instance, correspond to a datapoint attached to a space instance or asset instance. It is identified by its unique database identifier.

```json
  {
      "id": "my_unique_identifier", // Network from where datapoint will be read
      "type": "datapointinstance" // Indicates the datapoint is entirely described by its protocol argument
  }
```

### Protocol argument based
The datapoint from protocol arguments, this is a datapoint entirely described by the protocol arguments of what must be read on the fieldbus. The protocol arguments depend of the targetted fieldbus network .

```json
  {
      "networkId": "{{bacnetNetworkId}}", // Network from where datapoint will be read
      "type": "protocolargumentonly", // Indicates the datapoint is entirely described by its protocol argument
      "protocolArguments":
      { 
          // Fieldbus specific protocol arguments
      }
  }
```

Protocol arguments for BACNet property are:
```json
  {
    "deviceid": 1,                // BACNet device instance
    "objecttype": "analog-value", // BACNet object type
    "objectinstance": 1,          // BACNet object instance
    "propertyidentifier": "present-value"// BACNet property identifier
  }
```

Protocol arguments for ModBus property are:
```json
{
  "deviceaddress":"192.68.1.1", // ModBus IP address
  "deviceport":502,             // Ip port
  "deviceslave":0,              // Slave Id
  "protocol":"TCP",             // TCP/UDP
  "datapointaddress":11,        // Register or Coil address
  "datapointtype":"coil",       // coil, discrete, inputRegister, holdingRegister
  "valueformat":"bit"           // 
}
```

|Datapoint Type| Value format|
|-|-|
|coil, discrete| bit|
|inputRegister, holdingRegister| int16, int16rev, uint16, uint16rev, int32, int32sb, int32rev, int32revsb, uint32, uint32sb, uint32rev, uint32revsb, float32, float32sb, float32rev, float32revsb, float64, float64rev, string, stringrev, int64, int64rev, uint64, uint64rev|
|


### Network organization datapoint (bus datapoint)
The datapoint from network organization, this is a datapoint that has been scanned or declared in the fieldbus network organization, but which is not yet linked to any asset or spaces.

```json
  {
      "type": "busdatapoint", // Indicates the datapoint is available in a network organization
      "networkId": "{{bacnetNetworkId}}", // Network to which belongs the bus datapoint
      "busDatapointId": "bus_datapoint_id_read_from_the_network_organization"
  }
```

### How to read a DataPoint ?

This sample shows how to read a single datapoint instance value  from openBOS&reg;.
To read the value from a specific DataPoint use the route<br/>
`POST : {{baseUrl}}/api/v1/ontology/datapointinstance/livedata/extended/read`

As a parameter you give an array of datapoint identifiers to read

Example for a datapoint identifier corresponding to a datapoint instance:
```csharp
    var readRequest = new RestRequest($"{baseUrl}/api/v1/ontology/datapointinstance/livedata/extended/read", Method.Post);
    readRequest.RequestFormat = DataFormat.Json;
    readRequest.OnBeforeDeserialization = resp => { resp.ContentType = "application/json; charset=utf-8"; };
    List<dynamic> read = new List<dynamic>
    { // List of datapoint identifier to be read
        new
        {
            datapointClientId = "client_unique_identifier", // Unique identifier by datapoint (will be find back in the response)
            identifier= new {
              datapointInstanceId = "datapoint_unique_identifier", // The id of the datapoint instance in the database
              type = "datapointinstance" // Indicates the datapoint is entirely described by its protocol argument
            }
        }
    };
    readRequest.AddBody(read);
    var readResponse = client.Execute<dynamic>(readRequest).Data;
```

Example for a datapoint identifier corresponding to a protocol argument datapoint:
```csharp
    var readRequest = new RestRequest($"{baseUrl}/api/v1/ontology/datapointinstance/livedata/extended/read", Method.Post);
    readRequest.RequestFormat = DataFormat.Json;
    readRequest.OnBeforeDeserialization = resp => { resp.ContentType = "application/json; charset=utf-8"; };
    List<dynamic> read = new List<dynamic>
    {
        new
        {
          datapointClientId: "unique_identifier", // Unique identifier by datapoint (will be find back in the response)
          identifier: new {
            type= "protocolargumentsonly",
            networkId= "{{bacnetNetworkId}}", // Database identifier of the network to read on
            protocolArguments= new { // protocol arguments of the fieldbus datapoint
              deviceid= 1, // BACNet device instance
              objectinstance= 1, // BACNet object instance
              objecttype= "analog-value", // BACNet object Type
              propertyidentifier= "present-value" // BACNet property Identifier
            }
          }
        }
    };
    readRequest.AddBody(read);
    var readResponse = client.Execute<dynamic>(readRequest).Data;
```

Example for a datapoint identifier corresponding to network organization datapoint:
```csharp
    var readRequest = new RestRequest($"{baseUrl}/api/v1/ontology/datapointinstance/livedata/extended/read", Method.Post);
    readRequest.RequestFormat = DataFormat.Json;
    readRequest.OnBeforeDeserialization = resp => { resp.ContentType = "application/json; charset=utf-8"; };
    List<dynamic> read = new List<dynamic>
    {
        new
        {
          datapointClientId= "unique_identifier", // id of the response
          identifier: new {
            type= "busdatapoint",
            networkId= "{{bacnetNetworkId}}", // Database identifier of the network to read on
            busDatapointId= "{{busdatapointId}}" // Database identifier of the fieldbus datapoint in the network organization
          }
        }
    };
    readRequest.AddBody(read);
    var readResponse = client.Execute<dynamic>(readRequest).Data;
```

Result value will be an array of read result DatapointInstanceValueDTO[]
```json
  [  
    {
      "id": "datapointClientId from request",
      "value": "the value of the datapoint",
      "quality": "quality of the datapoint (none / good / bad)",
      "innerError": "error occured while reading a datapoint",
      "timeStamp": "date time of when datapoint has been read"
    }
  ]
```

## How to write a DataPoint value?

This sample will show how to write a value on a specific datapoint.

A datapoint is uniquely identified by its unique id.
To write a value to a specific DataPoint use the route<br/>
`POST : /api/v1/ontology/datapointinstance/livedata/extended/write`<br/>
As a parameter you give an array of DatapointInstanceExtendedWriteDTO[] containing the value that will be written for each DataPoint.

A single item would look as follow:
```json
  {
    "datapointClientId": "unique identifier of the datapoint",
    "value": "value to be written",
    "identifier": {
      // A datapoint identifier as explained previously
    } 
  }
```

 Example:
```json
[
  {
    "datapointClientId" : "bob",
    "identifier": {
      "type": "datapointinstance",
      "datapointInstanceId": "{{knowndatapoint}}"
    },
    "value": 3
  }
]
```

```csharp
    var writeRequest = new RestRequest($"{baseUrl}/api/v1/ontology/datapointinstance/livedata/extended/write", Method.POST);
    writeRequest.RequestFormat = DataFormat.Json;
    writeRequest.OnBeforeDeserialization = resp => { resp.ContentType = "application/json; charset=utf-8"; };
    List<dynamic> writeBody = new List<dynamic>
    {
        new
        {
          datapointClientId= "bob",
          identifier= {
            type= "datapointinstance",
            datapointInstanceId= "{{knowndatapoint}}"
          },
          value= 3
        }
    };
    writeRequest.AddBody(writeBody);
    var writeResponse = client.Execute<dynamic>(writeRequest).Data; // result of type DatapointInstanceExtendedWriteResultDTO[]
```

The result value will be an array of DatapointInstanceWriteResultDTO.
```json
[
  {
    "id": "unique datapointClientId",
    "errorCode": "error code of the write operation",
    "innerError": "error occured while reading a datapoint",
  }
]
```

## How do I subscribe to any datapoint value change?

This sample explains how to create a subscription to be notified of value change on a specific datapoint.
The sample uses SignalR technology.

### Using local access on the Building edge

```csharp

    // Define callback where the value event will arrive
    Action<JsonElement> receiveNotificationHandler = (JsonElement data) =>
    {
      // Output the datapoint value change
      Console.WriteLine(data);
    };

    // Initialize the SignalR hub connection between the client application and the openBOS&reg;
    var url = "{baseUrl}/subscription/";
    var connection = new HubConnectionBuilder()
        .WithUrl(url, options =>
        {
            options.AccessTokenProvider = () => Task.FromResult(bearerToken);
        })
        .WithAutomaticReconnect()
        .AddNewtonsoftJsonProtocol()
        .Build();

    // Attach the previous defined callback to the reception of the signalr event
    connection.On<JsonElement>("Notification", receiveNotificationHandler);

    // Starts notification channel
    await connection.StartAsync();

    // Keep in memory the connection id of the channel taht will be used to identify where subscription must relay the event
    string connectionId = await connection.InvokeAsync<string>("ConnectionId");

    // Subscribe to specific datapoint instance value
    var subRequest = new RestRequest($"{baseUrl}/api/v1/ontology/datapointinstance/livedata/extended/subscribe", Method.POST, DataFormat.Json);
    dynamic subBody =
        new
        {
            keepAllChanges = true, // all changes
            minSendTime = 0, // minimum time between sending event
            maxSendTime = 0, // maximum time between sending event
            connectionId = connectionId
        };
    subRequest.AddBody((object)subBody);
    var subscriptionResponse = client.Execute<JsonNode>(subRequest);

    // Add datapoint to the subscription
    var subRequest = new RestRequest($"{baseUrl}/api/v1/ontology/datapointinstance/livedata/extended/subscribe/{subscriptionResponse.Data["id"]}", Method.POST, DataFormat.Json);
    dynamic subBody =
        new
        {
            ignoreErrors = true, 
            items = new dynamic[] {
              {
                datapointClientId= "bob",
                scanRate= 30000,
                deadband= 0,
                datapoint= {
                    type= "datapointinstance",
                    datapointInstanceId= "{{knowndatapoint}}"
                }              
              }
            }
        };
    subRequest.AddBody((object)subBody);
    var addDatapointResponse = client.Execute<JsonNode>(subRequest);

    // Value change event will be relayed in the callback method when a value will change

```


When using access from cloud:

```csharp

  // Define callback where the value event will arrive
  Action<JsonElement> receiveNotificationHandler = (JsonElement data) =>
  {
    // Output the datapoint value change
    Console.WriteLine(data);
  };

  var deviceid = "<mybuildingedgedeviceid>";

    // Initialize the SignalR hub connection between the client application and the openBOS&reg;
  var url = $"https://api.electrification.ability.abb/buildings/openbos/apiproxy/v1/events?device={deviceid}";
  var connection = new HubConnectionBuilder()
      .WithUrl(url, options =>
      {
          options.AccessTokenProvider = () => Task.FromResult(bearerToken);
      })
      .WithAutomaticReconnect()
      .Build();

    // Attach the previous defined callback to the reception of the signalr event
    connection.On(deviceid, receiveNotificationHandler);
    // Starts notification channel
    await connection.StartAsync();
    // Keep in memory the connection id of the channel that will be used to identify where subscription must relay the event
    string connectionId = connection.ConnectionId;

    var subRequest = new RestRequest($"{baseUrl}/api/v1/ontology/datapointinstance/livedata/extended/subscribe", Method.POST, DataFormat.Json);
    subRequest.OnBeforeDeserialization = resp => { resp.ContentType = "application/json; charset=utf-8"; };
    dynamic subBody =
        new
        {
            keepAllChanges = true, // all changes
            minSendTime = 0, // minimum time between sending event
            maxSendTime = 0, // maximum time between sending event
            connectionId = connectionId, // connectionId retrieved after the connection is build
        };
    subRequest.AddBody((object)subBody);
    var subscriptionResponse = client.Execute<JsonNode>(subRequest);

    // Add datapoint to the subscription
    var subRequest = new RestRequest($"{baseUrl}/api/v1/ontology/datapointinstance/livedata/extended/subscribe/{subscriptionResponse.Data["id"]}", Method.POST, DataFormat.Json);
    subRequest.OnBeforeDeserialization = resp => { resp.ContentType = "application/json; charset=utf-8"; };
    dynamic subBody =
        new
        {
            ignoreErrors = true, 
            items = new dynamic[] {
              {
                datapointClientId= "bob",
                scanRate= 30000,
                deadband= 0,
                datapoint= {
                    type= "datapointinstance",
                    datapointInstanceId= "{{knowndatapoint}}"
                }              
              }
            }
        };
    subRequest.AddBody((object)subBody);
    var addDatapointResponse = client.Execute<JsonNode>(subRequest);

    // Value change event will be relayed in the callback method when a value will change
```
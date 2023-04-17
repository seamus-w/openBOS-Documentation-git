# Manage alarms

This example reads the complete of alarm active in the property

```csharp
    // Set Authentication token
    var client = new RestSharp.RestClient();
    client.AddDefaultHeader("Authorization", $"Bearer {bearerToken}");

    Console.WriteLine("Reading current alarms");
    dynamic liveAlarms = client.Execute<dynamic>(new RestRequest($"{baseUrl}/api/v1/ontology/livealarms")).Data;

    foreach (var liveAlarm in liveAlarms)
    {
        Console.WriteLine($"Timestamp : {liveAlarm["timeStamp"]}");
        Console.WriteLine($"DataPointName : {liveAlarm["dataPointName"]}");
        Console.WriteLine($"AlarmInstanceName : {liveAlarm["alarmInstanceName"]}");
        Console.WriteLine($"Active: {liveAlarm["active"]}");
        Console.WriteLine($"Closed: {liveAlarm["closed"]}");
        Console.WriteLine($"Acked : {liveAlarm["acked"]}");
    }
```

This example reads the complete of alarm active that are active inside a specific zone

```csharp
    // Set Authentication token
    var client = new RestSharp.RestClient();
    client.AddDefaultHeader("Authorization", $"Bearer {bearerToken}");

    dynamic floors = client.Execute<dynamic>(new RestRequest($"{baseUrl}/api/v1/ontology/zone?filter=Tags contains \"bos:floor\"")).Data;

    Console.WriteLine("Reading current alarms active floor by floor");
    foreach (var floor in floors){
        dynamic liveAlarms = client.Execute<dynamic>(new RestRequest($"{baseUrl}/api/v1/ontology/livealarms/zone/{floor["id"]}")).Data;

        foreach (var liveAlarm in liveAlarms)
        {
            Console.WriteLine($"Timestamp : {liveAlarm["timeStamp"]}");
            Console.WriteLine($"DataPointName : {liveAlarm["dataPointName"]}");
            Console.WriteLine($"AlarmInstanceName : {liveAlarm["alarmInstanceName"]}");
            Console.WriteLine($"Active: {liveAlarm["active"]}");
            Console.WriteLine($"Closed: {liveAlarm["closed"]}");
            Console.WriteLine($"Acked : {liveAlarm["acked"]}");
        }
    }
```


## How to subscribe to live alarms ?

This sample explains how to create a subscription to be notified of alarm change on the complete property. The sample uses SignalR technology.

### Using local access on the Building edge.

```csharp

    // Define callback where the alarms event will arrive
    Action<JsonElement> receiveNotificationHandler = (JsonElement data) =>
    {
      // Output the alarm change
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

    // Keep in memory the connection id of the channel that will be used to identify where subscription must relay the event
    string connectionId = await connection.InvokeAsync<string>("ConnectionId");

    // Subscribe to alarm change
    var subRequest = new RestRequest($"{baseUrl}/api/v1/ontology/alarm/subscription", Method.POST, DataFormat.Json);
    dynamic subBody =
        new
        {
            connectionId = connectionId // connectionId retrieved after the connection is build
        };
    subRequest.AddJsonBody(subBody);
    var subscriptionResponse = client.Execute<dynamic>(subRequest);

    // Value change event will be relayed in the callback method when a new alarm state will change

```


### When using access from cloud:

```csharp

  // Define callback where the value event will arrive
  Action<JsonElement> receiveNotificationHandler = (JsonElement data) =>
  {
    // Output the alarm change
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

    var subRequest = new RestRequest($"{baseUrl}/api/v1/ontology/alarm/subscription", Method.Post);
    // Subscribe to alarm change
    subRequest.OnBeforeDeserialization = resp => { resp.ContentType = "application/json; charset=utf-8"; };
    dynamic subBody =
        new
        {
            connectionId = connectionId, // connectionId retrieved after the connection is build
        };
    subRequest.AddBody(subBody);
    var subscriptionResponse = client.Execute<dynamic>(subRequest);

    // Alarm event will be relayed in the callback method when aa alarm state will change
```

## How to create an alarm instance ?

Here is an example on to create an alarm that will trigger when the fieldbus value is higher than 1.

```csharp
    // Set Authentication token
    var client = new RestSharp.RestClient();
    client.AddDefaultHeader("Authorization", $"Bearer {bearerToken}");

    // Subscribe to alarm change
    var subRequest = new RestRequest($"{baseUrl}/api/v1/core/alarm", Method.Post);
            dynamic subBody =
                new
                {
                    frequency = 300000, // polling frequency
                    datapoint = new
                    {
                        // An object representing the datapoint identifier that will be used for the alarm
                    },
                    deadband = 0,  // Value deadband
                    hysteresis = 0, // Alarm hysteresis
                    needAcknowledge = false,// Indicates whether the alarm needs to be acknowledged to disappear
                    triggers = new dynamic[] { // Indicates the algorithm that will trigger the alarm
                    new {
                        name= "", // Name of the trigger
                        severity= "High", // Severity of the alarm 
                        type= "AlarmTriggerValueDTO", 
                        description= "This is my alarm message",
                        threshold= "1"
                    }
                }
                };
    subRequest.AddBody(subBody);
    var subscriptionResponse = client.Execute<JsonObject>(subRequest);

```

## How to update an alarm instance ?

To update an alarm instance you must get the alarm, change the properties you want to change and use the PUT method to update
```csharp
    // Set Authentication token
    var client = new RestSharp.RestClient();
    client.AddDefaultHeader("Authorization", $"Bearer {bearerToken}");

    var subRequest = new RestRequest($"{baseUrl}/api/v1/core/alarm{alarmId}", Method.Get);
    var alarmDetail = client.Execute<JsonObject>(subRequest).Data;
    alarmDetail["triggers"][0]["threshold"] = 10;
    var subRequest = new RestRequest($"{baseUrl}/api/v1/core/alarm/{alarmId}", Method.Put);
    subRequest.AddBody(alarmDetail);
    var subscriptionResponse = client.Execute<dynamic>(subRequest);

```

## List of possible triggers

### AlarmTriggerDigitalOFFDTO
Object is as follow:
```json
    {
        "name"= "", // Name of the trigger
        "severity"= "High", // Severity of the alarm 
        "type"= "alarmtriggerdigitaloffdto", 
        "description"= "This is my alarm message"
    }
```

### AlarmTriggerDigitalONDTO
Object is as follow:
```json
    {
        "name"= "", // Name of the trigger
        "severity"= "High", // Severity of the alarm 
        "type"= "alarmtriggerdigitalondto", 
        "description"= "This is my alarm message"
    }
```

### AlarmTriggerHiDTO

Object is as follow:
```json
    {
        "name"= "", // Name of the trigger
        "severity"= "High", // Severity of the alarm 
        "type"= "alarmtriggerhidto", 
        "description"= "This is my alarm message",
        "threshold"= "1"
    }
```

### AlarmTriggerHiHiDTO
Object is as follow:
```json
    {
        "name"= "", // Name of the trigger
        "severity"= "High", // Severity of the alarm 
        "type"= "alarmtriggerhihidto", 
        "description"= "This is my alarm message",
        "threshold"= "1"
    }
```

### AlarmTriggerLoDTO
Object is as follow:
```json
    {
        "name"= "", // Name of the trigger
        "severity"= "High", // Severity of the alarm 
        "type"= "alarmtriggerlodto", 
        "description"= "This is my alarm message",
        "threshold"= "1"
    }
```

### AlarmTriggerLoLoDTO
Object is as follow:
```json
    {
        "name"= "", // Name of the trigger
        "severity"= "High", // Severity of the alarm 
        "type"= "alarmtriggerlolodto", 
        "description"= "This is my alarm message",
        "threshold"= "1"
    }
```

### AlarmTriggerNetworkDTO
Object is as follow:
```json
    {
        "name"= "", // Name of the trigger
        "severity"= "High", // Severity of the alarm 
        "type"= "alarmtriggernotvaluedto", 
        "description"= "This is my alarm message"
    }
```

### AlarmTriggerNotValueDTO
Object is as follow:
```json
    {
        "name"= "", // Name of the trigger
        "severity"= "High", // Severity of the alarm 
        "type"= "alarmtriggernotvaluedto", 
        "description"= "This is my alarm message",
        "threshold"= "1"
    }
```
### AlarmTriggerInBandDTO
Object is as follow:
```json
    {
        "index"= 1 or 2, // You can create 2 triggers with inband
        "name"= "", // Name of the trigger
        "severity"= "High", // Severity of the alarm 
        "type"= "alarmtriggeroutofbanddto", 
        "description"= "This is my alarm message",
        "thresholdLower"= "1", // Low band level
        "thresholdUpper"= "10", // High band level
    }
```

### AlarmTriggerOutOfBandDTO
Object is as follow:
```json
    {
        "name"= "", // Name of the trigger
        "severity"= "High", // Severity of the alarm 
        "type"= "alarmtriggeroutofbanddto", 
        "description"= "This is my alarm message",
        "thresholdLower"= "1", // Low band level
        "thresholdUpper"= "10", // High band level
    }
```

### AlarmTriggerValueDTO
Object is as follow:
```json
    {
        "name"= "", // Name of the trigger
        "severity"= "High", // Severity of the alarm 
        "type"= "alarmtriggervaluedto", 
        "description"= "This is my alarm message",
        "threshold"= "1"
    }
```


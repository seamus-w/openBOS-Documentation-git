# Manage users

```
NOTE:
Sample are written in C# and uses nuget package RestSharp for convenience.
baseUrl for local access : http://<ipaddress>
baseUrl for cloud access : `https://api.electrification.ability.abb/buildings/openbos/apiproxy/v1/gateway/<edgeid>`
```
## Postman link

You can find other samples in the Postman collection
[![Run in Postman](https://run.pstmn.io/button.svg)](https://app.getpostman.com/run-collection/14996509-f2ab8b96-9c38-4825-ab6f-7022e954deda?action=collection%2Ffork&collection-url=entityId%3D14996509-f2ab8b96-9c38-4825-ab6f-7022e954deda%26entityType%3Dcollection%26workspaceId%3Dea90c3d1-21af-4177-8e72-f21b5ed12326)

## How to display property users? 

This example collects the users that are configured as "owner" in the project.
For each owner the sample displays the list of AccessRights the user is allowed and the list of roles to which the user is related to.

```csharp
    // Set Authentication token
    var client = new RestSharp.RestClient();
    client.AddDefaultHeader("Authorization", $"Bearer {bearerToken}");

    // Collect users with role owner (UserInfoDTO[])
    dynamic owners = client.Execute<JsonArray>(new RestRequest($"{baseUrl}/api/v1/ontology/user?type=Owner")).Data;

    // Display the rights of the first owner user (UserDTO)
    dynamic userDetail = client.Execute<JsonObject>(new RestRequest($"{baseUrl}/api/v1/ontology/user/{owners[0]["id"]}")).Data;
    // Display user AccessRights
    Console.WriteLine("AccessRights are : " + userDetail["accessRights"]);
    // Display user roles
    void DisplayRoleParent(string roleId)
    {
        if (string.IsNullOrEmpty(roleId)) return;
        dynamic roleDetail = client.Execute<JsonObject>(new RestRequest($"{baseUrl}/api/v1/ontology/role/{roleId}")).Data; // (RoleDTO)
        Console.WriteLine("Role Name : " + roleDetail["name"]);
        DisplayRoleParent((string)roleDetail["parentId"]);
    }
    foreach (JsonNode role in userDetail["roleIds"])
    {
        DisplayRoleParent((string)role);
    }

```

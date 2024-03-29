# Azure Functions for CRUD operations with Azure CosmosDB

See the source code for this example in these repos:

https://github.com/luiscoco/MicroServices-AzureFunction_CosmosDB-CreateItem

https://github.com/luiscoco/MicroServices-AzureFunction_CosmosDB-ReadItem

https://github.com/luiscoco/MicroServices-AzureFunction_CosmosDB-UpdateItem

https://github.com/luiscoco/MicroServices-AzureFunction_CosmosDB-DeleteItem

## 1. Prerequisite

Create Azure CosmosDB account, database and collection

e navigate to **Azure CosmosDB** service and we click in this option to create a new account:

![image](https://github.com/luiscoco/MicroServices_dotNET8_CRUD_WebAPI-AzureCosmosDB/assets/32194879/3822db85-173d-411d-8383-7f97d07c05f4)

We click on **Azure CosmosDB account** button

![image](https://github.com/luiscoco/MicroServices_dotNET8_CRUD_WebAPI-AzureCosmosDB/assets/32194879/c1080788-bfc0-445e-9084-ce0f0772044a)

Now we select the option **Azure Cosmos DB for NoSQL**, and we press the **create** button

![image](https://github.com/luiscoco/MicroServices_dotNET8_CRUD_WebAPI-AzureCosmosDB/assets/32194879/69182e3f-9493-41a8-ab6b-92308a9bdddf)

In following screen we input the required data for creating the service

We create a new **ResourceGroup name**: myRG

We set the **account name**: mycosmosdbluis1974

We choose the service **location**: France Central

Capacity mode: **serverless**

![image](https://github.com/luiscoco/MicroServices_dotNET8_CRUD_WebAPI-AzureCosmosDB/assets/32194879/ed5a1407-0303-44b3-b528-7a7fc1398d80)

We navigate to the **Data Explorer** page and we create a **New Database** and a **New Container**

We first create a **New Database**. We input the **DatabaseId**: ToDoList

![image](https://github.com/luiscoco/MicroServices_dotNET8_CRUD_WebAPI-AzureCosmosDB/assets/32194879/978b2f67-01c3-4711-95fd-e0eeb75a99c1)

We also create a **New Container**: Items

![image](https://github.com/luiscoco/MicroServices_dotNET8_CRUD_WebAPI-AzureCosmosDB/assets/32194879/10e1d766-9c53-4ab0-9624-52f257340480)

![image](https://github.com/luiscoco/MicroServices-AzureFunction_CosmosDB-CRUD_operations/assets/32194879/a7f40ff9-af30-4d1d-b04a-55fe66d7b4bb)

We insert the new items in the Azure CosmosDB

This is the **new item** json file:

```json
{
  "id": "1",
  "name": "Sample Item",
  "category": "Sample Category"
}
```

We click in **Items** and then **New Item**

Then we copy and paste the json content and press **Save** button

![image](https://github.com/luiscoco/MicroServices-AzureFunction_CosmosDB-CRUD_operations/assets/32194879/08809426-85e4-4e75-a938-6d779989b218)

## 2. Create Azure Functions Projects

We install the Project Templates with dotnet

```
dotnet new --install Microsoft.Azure.Functions.Worker.ProjectTemplates
```

First, we create a new Azure Functions project targeting .NET 8

```
dotnet new func -n CosmosDbCrudFunctions --Framework net8.0
cd CosmosDbCrudFunctions
```

## 3. Add NuGet Package for Cosmos DB

![image](https://github.com/luiscoco/MicroServices-AzureFunction_CosmosDB-CreateItem/assets/32194879/c384c247-fbcf-4c9d-866d-ee7a147bf0f5)

## 4. Implement each CRUD Operation in one Azure Function

### 4.1. Create a new Item

You will need to add a new class for each operation or implement them in a single class based on your preference

Here's an example of how to implement a simple Create operation

Similar patterns can be followed for Read, Update, and Delete operations.

**CreateFunction.cs**

```csharp
using Microsoft.Azure.Functions.Worker;
using Microsoft.Azure.Functions.Worker.Http;
using Microsoft.Extensions.Logging;
using Newtonsoft.Json;
using System.IO;
using System.Threading.Tasks;
using Microsoft.Azure.Cosmos;
using System;

namespace CosmosDbCrudFunctions
{
    public static class CreateFunction
    {
        private static readonly string EndpointUri = Environment.GetEnvironmentVariable("CosmosDbEndpointUri");
        private static readonly string PrimaryKey = Environment.GetEnvironmentVariable("CosmosDbPrimaryKey");
        private static readonly string DatabaseName = "ToDoList";
        private static readonly string ContainerName = "Items";
        private static CosmosClient cosmosClient = new CosmosClient(EndpointUri, PrimaryKey);

        [Function("CreateItem")]
        public static async Task<HttpResponseData> CreateItem(
            [HttpTrigger(AuthorizationLevel.Function, "post", Route = null)] HttpRequestData req,
            FunctionContext executionContext)
        {
            var logger = executionContext.GetLogger("CreateItem");
            logger.LogInformation("C# HTTP trigger function processed a request.");

            string requestBody = await new StreamReader(req.Body).ReadToEndAsync();
            dynamic data = JsonConvert.DeserializeObject(requestBody);

            Container container = cosmosClient.GetContainer(DatabaseName, ContainerName);
            await container.CreateItemAsync(data, new PartitionKey(data.id.ToString()));

            var response = req.CreateResponse(System.Net.HttpStatusCode.OK);
            await response.WriteStringAsync("Item created successfully");

            return response;
        }
    }
}
```

### 4.2. Read Operation: Retrieve an Item by ID

```csharp
using Microsoft.Azure.Functions.Worker;
using Microsoft.Azure.Functions.Worker.Http;
using Microsoft.Extensions.Logging;
using Newtonsoft.Json;
using System.IO;
using System.Threading.Tasks;
using Microsoft.Azure.Cosmos;
using System;

namespace CosmosDbCrudFunctions
{
    public static class ReadFunction
    {
        private static readonly string EndpointUri = Environment.GetEnvironmentVariable("CosmosDbEndpointUri");
        private static readonly string PrimaryKey = Environment.GetEnvironmentVariable("CosmosDbPrimaryKey");
        private static readonly string DatabaseName = "ToDoList";
        private static readonly string ContainerName = "Items";
        private static CosmosClient cosmosClient = new CosmosClient(EndpointUri, PrimaryKey);

        [Function("ReadItem")]
        public static async Task<HttpResponseData> ReadItem(
            [HttpTrigger(AuthorizationLevel.Function, "get", Route = "items/{id}")] HttpRequestData req,
            string id,
            FunctionContext executionContext)
        {
            var logger = executionContext.GetLogger("ReadItem");
            logger.LogInformation($"Reading item with ID: {id}");

            Container container = cosmosClient.GetContainer(DatabaseName, ContainerName);
            ItemResponse<dynamic> responseItem;
            try
            {
                responseItem = await container.ReadItemAsync<dynamic>(id, new PartitionKey(id));
            }
            catch (CosmosException ex) when (ex.StatusCode == System.Net.HttpStatusCode.NotFound)
            {
                var notFoundResponse = req.CreateResponse(System.Net.HttpStatusCode.NotFound);
                await notFoundResponse.WriteStringAsync($"Item with ID: {id} not found.");
                return notFoundResponse;
            }

            var okResponse = req.CreateResponse(System.Net.HttpStatusCode.OK);
            okResponse.Headers.Add("Content-Type", "application/json; charset=utf-8");

            // Convert the object to a JSON string.
            string jsonResponse = JsonConvert.SerializeObject(responseItem.Resource);

            // Use the static type for the body content.
            await okResponse.WriteStringAsync(jsonResponse);
            return okResponse;
        }
    }
}
```

### 4.3. Update Operation: Update an Item by ID

```csharp
using Microsoft.Azure.Functions.Worker;
using Microsoft.Azure.Functions.Worker.Http;
using Microsoft.Extensions.Logging;
using Newtonsoft.Json;
using System.IO;
using System.Threading.Tasks;
using Microsoft.Azure.Cosmos;
using System;

namespace CosmosDbCrudFunctions
{
    public static class UpdateFunction
    {
        private static readonly string EndpointUri = Environment.GetEnvironmentVariable("CosmosDbEndpointUri");
        private static readonly string PrimaryKey = Environment.GetEnvironmentVariable("CosmosDbPrimaryKey");
        private static readonly string DatabaseName = "ToDoList";
        private static readonly string ContainerName = "Items";
        private static CosmosClient cosmosClient = new CosmosClient(EndpointUri, PrimaryKey);

        [Function("UpdateItem")]
        public static async Task<HttpResponseData> UpdateItem(
            [HttpTrigger(AuthorizationLevel.Function, "put", Route = "items/{id}")] HttpRequestData req,
            string id,
            FunctionContext executionContext)
        {
            var logger = executionContext.GetLogger("UpdateItem");
            logger.LogInformation($"Updating item with ID: {id}");

            string requestBody = await new StreamReader(req.Body).ReadToEndAsync();
            dynamic data = JsonConvert.DeserializeObject(requestBody);
            data.id = id; // Ensure the ID is set correctly for the item to be updated

            Container container = cosmosClient.GetContainer(DatabaseName, ContainerName);
            var responseItem = await container.UpsertItemAsync(data, new PartitionKey(id));

            var okResponse = req.CreateResponse(System.Net.HttpStatusCode.OK);
            okResponse.Headers.Add("Content-Type", "application/json; charset=utf-8");

            // Convert the object to a JSON string.
            string jsonResponse = JsonConvert.SerializeObject(responseItem.Resource);

            // Use the static type for the body content.
            await okResponse.WriteStringAsync(jsonResponse);
            return okResponse;
        }
    }
}
```

### 4.4. Delete Operation: Delete an Item by ID

```csharp
using Microsoft.Azure.Functions.Worker;
using Microsoft.Azure.Functions.Worker.Http;
using Microsoft.Extensions.Logging;
using System.Threading.Tasks;
using Microsoft.Azure.Cosmos;
using System;

namespace CosmosDbCrudFunctions
{
    public static class DeleteFunction
    {
        private static readonly string EndpointUri = Environment.GetEnvironmentVariable("CosmosDbEndpointUri");
        private static readonly string PrimaryKey = Environment.GetEnvironmentVariable("CosmosDbPrimaryKey");
        private static readonly string DatabaseName = "ToDoList";
        private static readonly string ContainerName = "Items";
        private static CosmosClient cosmosClient = new CosmosClient(EndpointUri, PrimaryKey);

        [Function("DeleteItem")]
        public static async Task<HttpResponseData> DeleteItem(
            [HttpTrigger(AuthorizationLevel.Function, "delete", Route = "items/{id}")] HttpRequestData req,
            string id,
            FunctionContext executionContext)
        {
            var logger = executionContext.GetLogger("DeleteItem");
            logger.LogInformation($"Deleting item with ID: {id}");

            Container container = cosmosClient.GetContainer(DatabaseName, ContainerName);

            try
            {
                // Attempt to delete the item. If it does not exist, a CosmosException with status code 404 is thrown.
                await container.DeleteItemAsync<object>(id, new PartitionKey(id));
                var okResponse = req.CreateResponse(System.Net.HttpStatusCode.OK);
                okResponse.Headers.Add("Content-Type", "application/json; charset=utf-8");
                await okResponse.WriteStringAsync($"Successfully deleted item with ID: {id}");
                return okResponse;
            }
            catch (CosmosException ex) when (ex.StatusCode == System.Net.HttpStatusCode.NotFound)
            {
                var notFoundResponse = req.CreateResponse(System.Net.HttpStatusCode.NotFound);
                notFoundResponse.Headers.Add("Content-Type", "application/json; charset=utf-8");
                await notFoundResponse.WriteStringAsync($"Item with ID: {id} not found.");
                return notFoundResponse;
            }
        }
    }
}
```

## 5. Configure Local Settings

In your **local.settings.json** file, add your Cosmos DB connection string as follows

```json
{
  "IsEncrypted": false,
  "Values": {
    "AzureWebJobsStorage": "UseDevelopmentStorage=true",
    "FUNCTIONS_WORKER_RUNTIME": "dotnet-isolated",
    "CosmosDbEndpointUri": "",
    "CosmosDbPrimaryKey": ""
  }
}
```

We get the CosmosDB URI and the PrimaryKey from the Azure Portal

![image](https://github.com/luiscoco/MicroServices-AzureFunction_CosmosDB-CreateItem/assets/32194879/6739542b-3080-4667-aace-146cc02a8ae0)

## 6. Run and test the Azure Function with Postman

We run and test the Create Azure Function and we get this output

We test the function with **Postman** 

![image](https://github.com/luiscoco/MicroServices-AzureFunction_CosmosDB-CRUD_operations/assets/32194879/fdf67985-7b38-4ac5-831b-fe9641f05521)

We first send a **POST** request to create a new item

http://localhost:7104/api/CreateItem

![image](https://github.com/luiscoco/MicroServices-AzureFunction_CosmosDB-CRUD_operations/assets/32194879/739cf8f8-1491-42fd-a7c4-ba7970b9111d)

After we send a **GET** by ID request to retrive one item from the database

http://localhost:7124/api/items/77

![image](https://github.com/luiscoco/MicroServices-AzureFunction_CosmosDB-CreateItem/assets/32194879/8e02de8f-26b7-49d5-906d-4559cc7fd170)

We also send a **PUT** request to modify an existing item in the database

http://localhost:7278/api/items/88

![image](https://github.com/luiscoco/MicroServices-AzureFunction_CosmosDB-CreateItem/assets/32194879/7dd8b019-d1a9-4423-98fd-d30e8e8020a1)

Finally, we send a **DELETE** request to delete an item in the databse

 http://localhost:7164/api/items/88

![image](https://github.com/luiscoco/MicroServices-AzureFunction_CosmosDB-CreateItem/assets/32194879/727fe9b1-294a-4688-816c-379a4366f739)


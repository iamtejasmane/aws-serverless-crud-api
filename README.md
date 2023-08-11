# Building AWS Serverless CRUD APIs

This is AWS serverless CRUD API project. In this project, we will create a serverless API that creates, reads, updates, and deletes items from a DynamoDB table. DynamoDB is a fully managed NoSQL database service that provides fast and predictable performance with seamless scalability.

![](/img/arch.png)

When you invoke your HTTP API, API Gateway routes the request to your Lambda function. The Lambda function interacts with DynamoDB, and returns a response to the API Gateway. The API Gateway then returns a response to you.

## Step1: Create a DynamoDB table

You use a DynamoDB table to store data for your API.

Each item has a unique ID, which we use as the partition key for the table.

### To create a DynamoDB table

1. Open the DynamoDB console at https://console.aws.amazon.com/dynamodb/.
2. Click on **Create table**.

   ![](/img/dynamodb-1.png)

3. For **Table name**, enter `http-crud-api`.
4. For **Partition key**, enter `id`.

   ![](/img/dynamodb-2.png)

5. CLick **Create table**.

## Step 2: Create a Lambda Function

Create a Lambda function for the backend of your API. This Lambda function creates, reads, updates, and deletes items from DynamoDB.

The function uses events from API Gateway to determine how to interact with DynamoDB. For simplicity this tutorial uses a single Lambda function. As a best practice, you should create separate functions for each route.

### To create a Lambda function

1. Sign in to the Lambda console at https://console.aws.amazon.com/lambda.
2. Click on **Create function**.
3. For **Function name**, enter `http-crud-api-lambda`.

   ![](/img/function-1.png)

4. Under **Permissions** choose **Change default execution role**.
5. Select **Create a new role from AWS policy templates**.
6. For **Role name**, enter `http-crud-api-project-role`.

   ![](/img/function-2.png)

7. For **Policy templates**, choose `Simple microservice permissions`. This policy grants the Lambda function permission to interact with DynamoDB.

8. Click on **Create function**.

9. On the console's code editor, replace the code with the following code. and Click on **Deploy** to update your function.

```javascript
import { DynamoDBClient } from "@aws-sdk/client-dynamodb"
import {
  DynamoDBDocumentClient,
  ScanCommand,
  PutCommand,
  GetCommand,
  DeleteCommand,
} from "@aws-sdk/lib-dynamodb"

const client = new DynamoDBClient({})

const dynamo = DynamoDBDocumentClient.from(client)

const tableName = "http-crud-api"

export const handler = async (event, context) => {
  let body
  let statusCode = 200
  const headers = {
    "Content-Type": "application/json",
  }

  try {
    switch (event.routeKey) {
      case "DELETE /items/{id}":
        await dynamo.send(
          new DeleteCommand({
            TableName: tableName,
            Key: {
              id: event.pathParameters.id,
            },
          })
        )
        body = `Deleted item ${event.pathParameters.id}`
        break
      case "GET /items/{id}":
        body = await dynamo.send(
          new GetCommand({
            TableName: tableName,
            Key: {
              id: event.pathParameters.id,
            },
          })
        )
        body = body.Item
        break
      case "GET /items":
        body = await dynamo.send(new ScanCommand({ TableName: tableName }))
        body = body.Items
        break
      case "PUT /items":
        let requestJSON = JSON.parse(event.body)
        await dynamo.send(
          new PutCommand({
            TableName: tableName,
            Item: {
              id: requestJSON.id,
              price: requestJSON.price,
              name: requestJSON.name,
            },
          })
        )
        body = `Put item ${requestJSON.id}`
        break
      default:
        throw new Error(`Unsupported route: "${event.routeKey}"`)
    }
  } catch (err) {
    statusCode = 400
    body = err.message
  } finally {
    body = JSON.stringify(body)
  }

  return {
    statusCode,
    body,
    headers,
  }
}
```

## Step 3: Create an HTTP API

The HTTP API provides an HTTP endpoint for your Lambda function. In this step, you create an empty API. In the following steps, you configure routes and integrations to connect your API and your Lambda function.

### To create an HTTP API

1. Sign in to the API Gateway console at https://console.aws.amazon.com/apigateway.

2. Click on the **Create API**, and then for **HTTP API**, choose **Build**.

   ![](/img/api-gateway-1.png)

3. For **API name**, enter `http-crud-api`.

   ![](/img/api-gateway-2.png)

4. Click on **Next**.

   ![](/img/api-gateway-3.png)

5. For **Configure routes**, Click on **Next** to skip route creation. You create routes later.

   ![](/img/api-gateway-4.png)

6. Review the stage that API Gateway creates for you, and then choose **Next**.

   ![](/img/api-gateway-5.png)

7. Click on **Create**.

## Step 4: Create routes

Routes are a way to send incoming API requests to backend resources. Routes consist of two parts: an HTTP method and a resource path, for example, GET /items. For this example API, we create four routes:

- `GET /items/{id}`

- `GET /items`

- `PUT /items`

- `DELETE /items/{id}`

### To create routes

1. Sign in to the API Gateway console at https://console.aws.amazon.com/apigateway.

2. Click on your API.
3. Choose **Routes**.
4. Choose **Create**.
5. For **Method**, choose **GET**.
6. For the path, enter `/items/{id}`. The `{id}` at the end of the path is a path parameter that API Gateway retrieves from the request path when a client makes a request.

   ![](/img/route-1.png)

7. Choose **Create**.

8. Repeat steps 4-7 for `GET /items`, `DELETE /items/{id}`, and `PUT /items`.

   ![](/img/route-2.png)

## Step 5: Create an integration

You create an integration to connect a route to backend resources. For this example API, you create one Lambda integration that you use for all routes.

### To create an integration

1. Sign in to the API Gateway console at https://console.aws.amazon.com/apigateway.

2. Choose your API.

3. Choose **Integrations**.

   ![](/img/integration-1.png)

4. Choose **Manage integrations** and then choose **Create**.

   ![](/img/integration-2.png)

5. Skip **Attach this integration to a route**. You complete that in a later step.

6. For **Integration type**, choose **Lambda function**.

7. For **Lambda function**, enter `http-crud-api-lambda`.

   ![](/img/integration-3.png)

8. Click on **Create**.

## Step 6: Attach your integration to routes

1. Sign in to the API Gateway console at https://console.aws.amazon.com/apigateway.
2. Choose your API.
3. Choose **Integrations**.
4. Choose a **route**.
5. Under **Choose an existing integration**, choose `http-crud-api-lambda`.
6. Click on **Attach integration**.

   ![](/img/integration-4.png)

7. Repeat steps 4-6 for all routes.
   All routes show that an AWS Lambda integration is attached.

   ![](/img/integration-5.png)

## Step 7: Test your API

### To get the URL to invoke your API

1. Sign in to the API Gateway console at https://console.aws.amazon.com/apigateway.
2. Choose your API.
3. Note your API's invoke URL. It appears under **Invoke URL** on the Details page.
4. Copy your API's invoke URL.

   The full URL looks like `https://abcdef123.execute-api.us-west-2.amazonaws.com`.

### To create or update an item

- Use the following command to create or update an item. The command includes a request body with the item's ID, price, and name.
  ```bash
  curl -X "PUT" -H "Content-Type: application/json" -d "{\"id\": \"123\", \"price\": 12345, \"name\": \"myitem\"}" https://abcdef123.execute-api.us-west-2.amazonaws.com/items
  ```

### To get all items

- Use the following command to list all items.
  ```bash
  curl https://abcdef123.execute-api.us-west-2.amazonaws.com/items
  ```

### To get an item

- Use the following command to get an item by its ID.
  ```
  curl https://abcdef123.execute-api.us-west-2.amazonaws.com/items/123
  ```

### To delete an item

1. Use the following command to delete an item.
   ```bash
   curl -X "DELETE" https://abcdef123.execute-api.us-west-2.amazonaws.com/items/123
   ```
2. Get all items to verify that the item was deleted.
   ```
   curl https://abcdef123.execute-api.us-west-2.amazonaws.com/items
   ```

### Congratulation! You have completed the project.

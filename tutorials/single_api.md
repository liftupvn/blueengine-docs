# Request processing with single engine

After the engines are up and running, the next step is to request them to process some input data by calling REST APIs. There are two ways:

- Implement your own APIs in the BlueEdge monorepo. This is not recommended.
- Calling the generic APIs, define your own DAG, required engines, as well as the input data. You can write your own API to call those APIs and use the engines as-a-service.

## Generic API to call single engine

In many cases, all you need is one single engine to process everything, so we provide a simple API for that case: `api/v1/process-single-engine`. With the API you will:

- Define the engine you want use, including its ID and IO channels
- Define the input data to send to the engine
- List the engine output channels you are interested. The data that are written to them will be stored into the database for later use.
- Optionally register webhook callbacks to receive the results based on events.
- Other configurations.

### Request processing

To create new processing `session` with the target ID, call the API as the following example:

```
POST http://192.168.80.186:5000/api/v1/process-single-engine
```

```json
Body:
{
  "engineId": "jW7qGV9dyuBYy5kYpcMHm", // Must match Engine's ID
  "inputs": { // Must match the engine's input channels.
    "Control": { // Channel Name
      "type": "JSON", // Channel data type
      "payload": { // Data input to the "Control" channel
        "fileUrls": [
            "https://stimg.cardekho.com/images/carexteriorimages/630x420/Lamborghini/Urus/4418/Lamborghini-Urus-V8/1621927166506/front-left-side-47.jpg",
            "https://boatsandjetskis.co.uk/wp-content/uploads/2020/08/Volvo-XC40-white-scaled-1.jpg"
        ],
        "accessKey": "123431",
        "secretKey": "asdfadsf",
        "targetFolder": "claim_id",
        "bucket": "insurance_data_bucket"
      }
    }
  },
  "outputs": { // Must match the engine outputs
    "Results": {
      "type": "JSON", // Must match as in the engine's config
      "interested": true // Output to this channel will be saved into the database
    },
    "Status": {
      "type": "JSON",
      "interested": true
    }
  },
  "webhooks": [ // Webhook registrations
    {
        "statusTypes": ["finish", "sessionStatusChanged"], // Listen to these events
        "callbackUrl": "https://engine.aicycle.ai/api/v1/utils/test-webhook",
        "callbackHeader": {
            "Authorization": "Bearer afkdjaf...."
        },
        "resultChan": "Results"
    }
  ],
  "waitTimeSecond": 15 // Wait for finish, or 15s max
}
```

> For webhook registration details, see the [webhook registration doc]()

Response:

```json
{
    "status": "success",
    "message": "",
    "data": {
        "sessionId": "session-process-single-engine-0-1635242482.3139913-HUWxQmER0lI3BuztWjW7V", // Use this session id to get the results later 
        "process": {
            "status": "finished", // Status of the session
            "total_pending": 0,
            "total_queuing": 0
        },
        "results": { // The results if available
            "Results": [
                ... // Data write to Results channel
            ],
            "Status": [
                {
                    "type": "finish"
                }
            ]
        }
    }
}
```

### Get the results

If you specify webhook registrations, when an event, e.g. `finish`, occurs, a POST request with the result data in the body will be sent to the webhook's `callbackUrl`.

To call API to get the results, use the following API:

```
GET http://192.168.80.186:5000/api/v1/process-single-engine/::sessionId?channel=Results
```

Where `::sessionId` is the sessionId responded in the previous POST request.

Response is similar to the response of the POST request, but only the requested channel is responded.

```json
{
    "status": "success",
    "message": "",
    "data": {
        "sessionId": "session-process-single-engine-0-1635242482.3139913-HUWxQmER0lI3BuztWjW7V", // Use this session id to get the results later 
        "process": {
            "status": "finished", // Status of the session
            "total_pending": 0,
            "total_queuing": 0
        },
        "results": { // The results if available
            "Results": [
                ... // Data write to Results channel
            ]
        }
    }
}
```

## Generic API to call multi-engine DAG

See [dag_api](tutorials/dag_api.md)

## API Authorization

Currently APIs are all open. We will support API Key authorization later.
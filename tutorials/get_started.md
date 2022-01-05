# Getting started: Hello world engine!

This doc describes how to create a super simple engine with the BlueEngine SDK, and call it with a simple API. With the knowledge provided, you can easily create much bigger AI engines and deploy it to the cloud in no time :D

## Setup

Follow the [install instruction](tutorials/install.md) to install the SDK.

Then, clone the tutorial repo: https://github.com/liftupvn/blueengine-tutorial.

## Write the processing function

Create an example folder called hello-world, then create the `engine.py` file as the entrypoint of the engine.

```bash
cd ~/path/to/your/workplace
mkdir -p hello-world-blueengine
cd hello-world-blueengine

# create the engine file 
touch engine.py
```

Open the file. Here, we are going to create an *engine instance*, *a processing function*, and connect that function to the *IO channels*, so it can receive data and produce some results.

```python
# engine.py
from blueengine import BlueEngine

def run_process(context, input):
    # will be defined later

if __name__ == "__main__":
    # Make a simple engine that will works with BlueEdge system
    engine = BlueEngine()
    
    # add the processing function
    engine.add_step('hello world',  # register a process
                    process_func=run_process,  # processing function
                    input_channels=['Control'], # connect to 'Control' input channel
                    output_channels=['Result', 'Status']) # connect to 'Result' and 'Status' output channels
    # run the engine
    engine.start()
```
What we are doing here is:

- Create an `BlueEngine` instance by calling `engine = BlueEngine()`
- Add the `run_process` function to the engine with `engine.add_step()`, and connect it to the `Control` input and `Result` and `Status` output channels.
- Start the engine with `engine.start()`

The run_process function is defined as follow:

```python
def run_process(context, input):
    # data from the Message channel
    message = input.get('Control', None)
    
    if message is None:
      return
    
    # assuming the message will be:
    # {
    #   'data': 'Hello'
    # }
    if message['data'] == 'Hello':
        return {
            'Result': {
                'data': 'World' # this will be sent to `Result` channel
            },
            'Status': {
                'type': 'finish', # tell the server the session is finished
            }
        }
```

What is happening is:

- The function will be called every time there are any input data on one of the registered channels. 
- The input argument is a dict that contains the input data from different input channels. Use `input.get('ChannelName', default_value)` to get the data. **Don’t use `input['ChannelName']`!** Later you will see when we have multiple input channels, the channels that don’t have data won’t be presented in the dict’s keys. Thus, getting the data like that can cause `KeyError` exception. Here the input channel is `Control`.

> `Control` is a special input channel that is added to the engine by default, which can be used by the management system to send control commands to the engine, like `stop`. You can use the `Control` channel to send any other `JSON` data as well.
- The function does some processing, then send results to the channels:
  - Send result to user-defined channel `Result`, in `JSON` format.
  - Send status update finish to `Status` channel. Status is a special channel that is examined by the management server to update the session’s status. Here, we simply tell the session is finished. More on this later.

> Same as `Control`, `Status` is another special output channel that is listened by the system to check the status of the process. E.g, if you produce a `JSON` message `{'type': 'finish'}` to `Status` channel, the internal management system of the engine, as well as the cloud management services, will understand the engine's process is done, and will make appropriate reactions and send the callback. Message with `type` `error` indicates that the session is failing. Custom `type` can be used as well, useful for custom webhook callback on certain events.

Full code is at: https://github.com/liftupvn/blueengine-tutorial/blob/master/1.hello-world/engine.py 

## Add the engine config files

Next, you must define the engine's configurations, by define a `config.yml` file. Run:

```bash
mkdir -p configs
nano configs/engine.yml
```

Then paste the following content to the file:

```yaml
ENGINE:
  ID: 'unique-nano-id'
  NAME: 'simple hello world engine'
INPUTS:
  NAMES: ['Control']
  TYPES: ['JSON']
OUTPUTS:
  NAMES: ['Result', 'Status']
  TYPES: ['JSON', 'JSON']
```

Save the file. Here, there are a few things to notice:

- The engine has a unique `ID`, which is used to find and assign an engine to process input. For now, simply go to [Online Nanoid Generator](nanoid.dev) and generate your `ID` ;D We will implement the engine management app later to produce this config.
- `INPUTS` and `OUTPUTS` fields define the IO channels of the engine, which we connected our function to in the previous step. 
- `NAMES` define the channels' name. `TYPES` define the channels' data types. Here, we only use `JSON`, but there are also other formats for images and videos IO as well.
> Mind the positions of the values, they must match.

## Connect to the BlueEdge system.

In order to let the engine connect to the network system and communicate with other parts, we need the definitions for the define the following config data on `configs/local.yml` file:

```yaml
# configs/local.yml

SYSTEM:
  ENVIRONMENT: 'local'
  MONGODB_DB_NAME: 'blueeye_db_dev'
  REDIS_ADDRESS: '192.168.80.186'
  REDIS_PORT: 6380
  KAFKA_BROKERS: ['192.168.80.186:9092']
  MONGODB_CONNECTION_STR: 'mongodb://192.168.80.25:27017'
  MANAGER_BROADCAST_CHANNEL: 'MNG_JZp0px0DalZ7je5kKYlZgddd'
  ENGINE_ATTENDANCE_CHANNEL: 'ATTENDANCE_gPKhENDvX5Zit-SAD3pmXddd'
  SESSION_MANAGE_REQUEST_CHANNEL: 'MANAGE-q-PAH6U_NwE1Uej-BljeSddd'
  USAGE_MANAGE_REQUEST_CHANNEL: 'USAGE-q-PAH6U_NwE1Uej-BljeSddd'
  ENGINE_STATUS_UPDATE_CHANNEL: 'STATUS-gGY7Ot-TAc8aAtfwypz3Fddd'
  TRACING:
    ENABLE: False
```

For systems that are deployed on Kubernetes, this can be deployed as a Secret.

When start the engine, set the `BLUEEDGE_COMM_CONFIG` environment to the file `configs/local.yml`

```bash
export BLUEEDGE_COMM_CONFIG=configs/local.yml
python3 engine.py
```

## Test the engine
That’s it! Now start the engine, and we are ready to send the first message to it!

```bash
export BLUEEDGE_COMM_CONFIG=configs/local.yml
python engine.py
```

Now, run this `curl`, or use `Postman`, to send a test request:

```json
// body.json file
{
  "engineId": "unique-nano-id",
  "inputs": {
    "Control": {
      "type": "JSON",
      "payload": {
          "data": "Hello"
      }
    }
  },
  "outputs": {
    "Result": {
      "type": "JSON",
      "interested": true
    },
    "Status": {
      "type": "JSON",
      "interested": true
    }
  },
  "resultChannels": ["Result"]
}
```

```bash
# make the request

curl --header "Content-Type: application/json" \
  --request POST \
  --data @body.json \
  http://192.168.80.186:5000/api/v1/process-single-engine
```

The response will be something like this:

```json
{
    "status": "success",
    "message": "",
    "data": {
        "sessionId": "session-process-single-engine-0-1634124519.3001285-XHRoeyWr9laz5pisha2GO",
        "process": {
            "status": "pending",
            "current_position": 1,
            "total_pending": 2,
            "total_queuing": 0
        }
    }
}
```

For now all you need to care about is the `sessionId`: We will use it to get the results and status of the process. 

> The above `api/v1/process-single-engine` API is a general purpose API that can be used to test processing with a single engine. You define the engine definition, as well as the data to send to the engine's input channels in the request's body. For testing a workflow that contains multiple engines, you can use `api/v1/process-workflow`. We also encourage you to create your own API and service instead of using those APIs, since they don't have any authentication or data validation.

## See the output results

With the `sessionId` from the previous step, call this to see the session status and results:

```bash
# make the request

curl --location --request GET 'https://engine.aicycle.ai/api/v1/process-single-engine/session-process-single-engine-0-1634124519.3001285-XHRoeyWr9laz5pisha2GO' \
--header 'Content-Type: application/json'
```

Response:

```json
{
    "status": "success",
    "message": "",
    "data": {
        "process": {
            "status": "finished",
            "total_pending": 0,
            "total_queuing": 0
        },
        "Status": [
            {
                "type": "finish"
            }
        ],
        "Result": [
            {
                "data": "World"
            }
        ]
    }
}
```
The `data.process.status` value is the status of the session, which can be `pending`, `queuing`, `processing`, `stopping`, `finished`, `failed`, and `canceled`. The rest of the response is the output of the engine.

And that is it! Congratulation! You have created the first engine :D There are also several other steps, including create Docker file, Kubernetes deployment file, but they are fairly simple and you can use some of our existing templates.

Finally, good luck and happy building engines!

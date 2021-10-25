# Working with Context

If you want to:

- Initialize some resources and reuse it again and again for all of the incoming sessions, e.g an AI model
- Store persisted data for each session, e.g user initialize data that is used to process each video frame.

Then `Context` is what you need.

## Intro 

**Context** is what is provided as the first argument in the processing functions. Each `Process` will have a separated `Context` instance.

```python

def process_fun(context, input):
    """
    context: current Context of this process
    input: dict of input data for each channel
    """
```

A `Context` object contains:

- `init_data`: store the initial data (if provided) for each `Process`, which is created only once when the `Process` start, and can be reused for all incoming requests.
- `start_data`: store the initial data for each `session`, which is created once for each session. You can use it to store the user's inputs, results from previous runs,...

## Guide

In order to write data into context's `init_data` and `start_data`, define the `init_func` and `start_func`, respectively, and pass them into the `add_step` method. If you need to release the data when finish using them, e.g. close or delete files, define `stop_func` and `terminate_func`, which will be called when the session is finished, and when the `Process` is finished, respectively.

```python

from blueengine import BlueEngine, Context

def init_processor(context: Context):
    # Called once at the start of each process
    model = init_model()
    other_data = get_init_data
    return model, other_data # A tuple, or anything you want

def start_processor(context: Context):
    # Called once for each session
    user_data = {
        'confident_thresh': 0.5,
        'file': None,
        'other': 'other stuffs'
    }
    
    return user_data # A dict, or anything you want

def run_process(context: Context, input):
    # Called anytime there is a new input
    model, other_data = context.init_data
    user_data = context.start_data

    # update the start data if needed
    user_data['confident_thresh'] = input['Control']['conf']
    user_data['file'] = open('user_file.tmp', 'r')
    ...

def stop_processor(context: Context):
    # Called when the session finish
    user_data = context.start_data
    if user_data['file'] != None:
        user_data['file'].close()

def terminate_processor(context: Context):
    # Called when the process is terminated
    model, other_data = context.init_data
    
    release_model(model)


if __name__ == "__main__":
    engine = BlueEngine()

    engine.add_step('example',
                    init_func=init_processor,
                    start_func=start_processor,
                    process_func=run_process,
                    stop_func=stop_processor,
                    terminate_func=terminate_processor,
                    input_channels=['Control'],
                    output_channels=['Result', 'Status'])
```
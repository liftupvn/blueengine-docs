# Build engine with multiple steps processing pipeline

One of the most advanced feature of `BlueEngine SDK` is allowing you to easily build an engine with multiple processing steps that run in parallel. This feature especially useful when you want to boost the total speed of the engine. For example, a *face recognition* engine can will need to run the following steps:

- Download image file
- Run faces detection
- Run face landmarks detection
- Align faces
- Run face recognition
- Write results

Traditionally, the above steps will run continuously in a single python process. And the when the next step runs, the previous step will stay idle, contribute nothing to the total performance. Therefore it's much better to be able to run those steps in parallel.

The most popular approach is using `multiprocessing` library to run each of the step in a seperated `Process`. And this is exactly what we are doing inside `BlueEngine SDK`. We just make your life a bit easier.

Internally, each of the step you add with the `add_step` function is run inside a `Process`, and communicate with others and with the remaining system using [ZeroMQ](https://zeromq.org/).

## Guide

In order to create a multi-steps processing pipeline, you can follow the steps:

- Design the required processing step. The general rule of thumb is seperate if two steps are both can be slow. E.g download, run AI model, and upload results, can be seperated into 3 parallel steps, so they won't block each other.
- Define the processing functions for each step.
- Write unit tests if possible.
- Use `add_step` function to add the above functions into the pipeline.

```python

def run_step_one(context, input):

    ...

def run_step_two(context, input):

    ...

if __name__ == '__main__':
    engine = BlueEngine()
    engine.add_step('step one',
                    process_func=run_step_one,
                    input_channels=['Control'],
                    output_channels=['Status', 'Statistic'])
    engine.add_step('step two',
                    process_func=run_step_two,
                    input_channels=[], # will take the forward data from previous step
                    output_channels=['Status', 'Result', 'Statistic'])

    engine.start()
```

## Forward data between steps

The steps is designed to run in waterfall fashion. The previous step will forward data to the next step. As you already see from the [first tutorial](get_started.md), the step write data to the output channels by `return` or `yield` to the channel name, like `Status` or `Result`. To push data to the later step, the current step can `yield` to `forward` channel instead.

```python

def run_step_one(context, input):
    # Do some processing
    ...
    
    yield {
        'forward': (image, numpy_array, json_data), # no need to be Json serializable
        'Status': { # also update something to the
            'type': 'message',
            'message': 'finish step 1'
        }
    }
    ...
    yield {
        'forward'
    }


def run_step_two(context, input):
    # Do some processing

    fwd_data = input.get('forward', None)
    if 'forward' in input:
        if 
        image, numpy_array, json_data = 
    ...
    
    yield {
        'Result': {
            'result': some_results # must be Json serializable
        }, 
        'Status': { # also update something to the
            'type': 'finish',
        }
    }

```

> Never yield `{'type': 'finish'}` message to `Status` channel in the middle process, since it will tell the engine to stop immediately. 
# Best practices

## 1. Mind the imports

Always remember that each of the processing steps will run in a different [Process](https://docs.python.org/3/library/multiprocessing.html#multiprocessing.Process), some libraries may not work as expected if you import them at the top of the file, especially for CUDA related libraries.

So instead of doing this:

```python
from cuda_lib import CudaModel

def init_processor(context):
    model = CudaModel() # This can have an CUDA exception

    return model
```

Do this:

```python

def init_processor(context):
    from cuda_lib import CudaModel
    model = CudaModel() # This should run fine

    return model
```

## 2. Yielding to Status channel

`Status` is the special output channel which is listened by many components of the management system:

- Internal engine management system.
- Session management system
- Webhook callback system

At anytime, if you write to `Status` channel a message `{'type': 'finish'}` or `{'type': 'error'}`, the whole process will be stopped immediately. So:

- Never yield `{'type': 'finish'}` from a middle step in a multi-step processing pipeline. Instead, forward a finish signal to the next step so it can know the previous step is finished and should finish up remaining works as well.

Don't:

```python
def step_one(context, input):
    ...
    yield {
        'Status': {
            'type': 'finish'
        },
        'forward': some_data # this will not be sent, the process is stopped immediately
    }

def step_two(context, input):
    ... # may miss something because the previous step finished the whole process
```

Do:

```python
def step_one(context, input):
    ...
    yield {
        'forward': some_data 
    }
    yield {
        'forward': 'FINISH' # I'm done.
    }

def step_two(context, input):
    fwd_data = input.get('forward', None)
    if fwd_data == 'FINISH':
        ...
        yield {
            'Status': {
                'type': 'finish' # finished
            }
        }
    
```

- If there is any error, yield `{'type': 'error'}` to `Status` channel immediately, so that the whole engine's processes as well as the whole system will be stopped.

```python
def run_something(context, input):
    try:
        ...
    except:
        yield {
            'Status': {
                'type': 'error', # this will cause the whole session to stop, even on other engines
                'message': f'Error: {traceback.format_exc()}' # provide details about the exception
            }
        }
```

## 3. Create and cleaning temporary files

In a process, you often create one or many temporally folders and files. After finish processing the session, you should always clean those files. The best place to clean them is inside the `stop_func` function. The rule-of-thumb is, the process which create the files should be the one clean the files. So do:

```python 
def start_process(context): # start_func
    return {
        'temp_files': []
    }

def process_data(context, input): # process_func
    ...
    a_file = create_files()

    context.start_data['temp_files'].append(a_file) # save the file info
    ...

def stop_process(context): # stop_func, called when the session finish
    for f in context.start_data['temp_files']:
        os.remove(f)
```

There is also another simpler way! When a new session come in, the engine's internal process automatically create a temp folder for that session, which has the name of `client_id`, and will be deleted when the session finish. You can get the folder name with `context.client_id`.

```python

def process_data(context, input): # process_func
    ...
    tmp_folder = context.client_id

    a_file_path = os.path.join(tmp_folder, a_file_name) # the whole tmp_folder will be deleted when the session finish
    ...

```




# cfmr-python

This is a python library for writing cloud-function map/reduce (a.k.a [cfmr](https://github.com/floodfx/cfmr)) jobs.

See [cfmr](https://github.com/floodfx/cfmr) project repository for more general information.

## Use
When writing a [cfmr](https://github.com/floodfx/cfmr) job, you just implement the `map` and `reduce` functions and then delegate the coordination to the framework.

#### Framework Hooks
There are 3 points of integration with the `cfmr` framework:
* `emitter`
* `mapper`
* `reducer`

### `emitter`
`emitter` is a framework object to which you should use to write your output `key`/`value` pairs.  Data written to the `emitter` is partitioned by the `key`.  

`emitter` has one function that you should use, namely the `emit` function which just takes a `key` and a `value`.  

`emitter.emit(key, value)`

Both keys and values can be binary data (e.g. `bytes`) and should be handled by the framework.  Otherwise, their type should support serialization via `json.dumps`.

### `mapper`
Use the `mapper` to delegate handling getting data from your input paths to your `map` function.  Your mapper lambda function should look something like:

```python
from cfmr import mapper

# your map function
def map(key, value, emitter):
  # your map logic
  # emit data to the reducer using the emitter. i.e. emitter.emit(myKey, myValue)
  return None # your map output or None

# the aws lambda entrypoint
def handle(event, context):
  # which delegates to the cfmr mapper
  # note the "map" function is passed into the mapper.handle function
  mapper.handle(event, context, map)
```  

##### Implementing `map`
As you can see above `map` should have the following function signature:

```python
# your map function
def map(key, value, emitter):
  # your map logic
  # emit data to the reducer using the emitter. i.e. emitter.emit(myKey, myValue)
  return None # your map output or None
```
`map` is automatically called by the cfmr framework and is passed:
* `key` - the path of the input data
* `value` - the bytes in the input data
* `emitter` - a framework helper to which you should write output for the reducer (discussed above)

### `reducer`
Use the `reducer` to delegate handling getting data from your partitioned `mapper` output into your `reduce` function.  Your `reducer` lambda function should look something like:

```python
from cfmr import reducer

# your reduce function
def reduce(key, values, emitter):
  # your reduce logic
  # emit data from the reducer using the emitter. i.e. emitter.emit(myKey, myValue)
  return None # your reduce output or None

def handle(event, context):
    reducer.handle(event, context, reduce)
```  

##### Implementing `reduce`
As you can see above `reduce` should have the following function signature:

```python
# your reduce function
def reduce(key, values, emitter):
  # your reduce logic
  # emit data from the reducer using the emitter. i.e. emitter.emit(myKey, myValue)
  return None # your reduce output or None
```
`reduce` is automatically called by the cfmr framework and is passed:
* `key` - the bytes or value of the partition key
* `values` - a collection of values in the same type as outputted by the `mapper` for this partition key
* `emitter` - a framework helper to which you should write output from the reducer (again discussed above)

### Example
Here is the `mapper` and `reducer` implementations for the canonical example of wordcount:

#### `mapper`
```python
import logging
from cfmr import mapper

logger = logging.getLogger()
logger.setLevel(logging.DEBUG)

def map(key, value, emitter):
    logger.debug(f"map: key:{key}, val:{value}")
    mapResult = []
    for word in value.decode('utf-8').split():
        emitter.emit(word, 1)
        mapResult.append([word, 1])
    logger.debug(f"mapResult:{mapResult}")
    return mapResult

def handle(event, ctx):
    logger.debug(f"event:{event}")
    mapper.handle(event, ctx, map)
```
#### `reducer`
```python
import logging
from cfmr import reducer

logger = logging.getLogger()
logger.setLevel(logging.DEBUG)

def reduce(key, values, emitter):
    logger.debug(f"map: key:{key}, vals:{values}")
    total = 0
    for value in values:
        total += value
    emitter.emit(key, total)
    logger.debug(f"reduceResult:{total}")
    return total

def handle(event, ctx):
    logger.debug(f"event:{event}")
    logger.debug(f"ctx:{ctx}")
    reducer.handle(event, ctx, reduce)
```

### Other Notes
Build & Uploading to PyPi
* run `python setup.py sdist` to build
* run `twine upload dist/*` to deploy
* wait (up to) 5 minutes for new version to show up in `pip`

# Save/Load for DL Estimators

Contributors: fkiraly, achieveordie, aurumnpegasus

## Contents

[TOC]


## Overview

### Problem Statement

We would like to support two workflows in sktime:

1. serialization to in-memory object and deserialization
2. serialization to single file and deserialization

Initially proposed in sktime issues [#3128](https://github.com/alan-turing-institute/sktime/pull/3128) and [#3336](https://github.com/alan-turing-institute/sktime/pull/3336).


### The proposed solution

The solution below introduces:

* an object interface points for estimators, `save`
* a loose method `load`
* a file format specification `skt` for serialized estimator objects

which are counterparts and universal.

Concrete class or intermediate class specific logic is implemented in:

* `save`, a class method
* `load_from_path` and `load_from_serial`, class methods called by `load`

while no class specific logic is present in `load`.

## Problem to solve

### Requirements

* the workflow should satisfy the strategy pattern: serialize/deserialize should work the same for all estimators, irrespective of type
* in particular, it should follow the same interface for "classical" statistical estimators, ML estimators, and deep learning estimators,
  without special assumptions and workflows for special estimators or estimator classes (e.g., deep learning)
* loading should not require the user to know the contents of the deserialized object/file

### Current implementation and workflow

Currently, no dedicated serialization and deserialization interfaces exist in `sktime`.

Serialization and deserialization is based on `pickle`, which uses default `__getstate__` and `__setstate__` dunders.

This works for most `sktime` estimators, but fails for unpickleable objects and the new deep learning estimators.

The intended user journey is as follows:

```python
import pickle

# 1. create model and acquire state
vecm = VECM()
vecm.fit(y_train, fh=fh)
# and possibly further state change

# 2. serialization to a pickle
save_output = pickle.dumps(vecm)
# potentially: manually save to file

# 3. deserialization from a pickle
# potentially: manually load from file 
model = pickle.loads(save_output)

# 4. use of deserialized object
model.predict(fh=fh)
# potentially further method calls or state change
# state change does not change state of pickle
```

### Problems of the current soultion

The current solution does not satisfy the following requirements:

* this fails for deep learning estimators
* control whether save/load is from file or memory object is manual

However, it does satisfy the strategy pattern - in all case where it does not fail.


## Proposed solution

### User journey design

The design implements the natural "save/load" user journey.

We pay particular attention to:

* the user does not need to have knowledge about contents of a serial object they load
* the user journey is generic, and does not depend on the particular estimator

The user journey exists in two minimally different versions, one for in-memory and one for file location.

#### User journey: serialize/deserialize in-memory

```python
from sktime import load

# 1. create model and acquire state
vecm = VECM()
vecm.fit(y_train, fh=fh)
# and possibly further state change

# 2. serialization to in-memory object
memory_serial = vecm.save()

# 3. deserialization from in-memory object
model = load(memory_serial)

# 4. use of deserialized object
model.predict(fh=fh)
# potentially further method calls or state change
# state change does not change state of serial object
```

#### User journey: serialize/deserialize to file

```python
from sktime import load

# 1. create model and acquire state
vecm = VECM()
vecm.fit(y_train, fh=fh)
# and possibly further state change

# 2. serialization to in-memory object
file_location = "my/file/location"
vecm.save(file_location)
# creates file at file_location
# the file is in skt format, see below

# clear memory

from sktime import load

# 3. deserialization from in-memory object
file_location = "my/file/location"
model = load(memory_serial)

# 4. use of deserialized object
model.predict(fh=fh)
# potentially further method calls or state change
# state change does not change state of serial object
```

### skt file format

All files are stored in the `skt` format, with specification as follows.

`skt` is a zip archive, by default uncompressed (but optionally with compression).

It must contain the following files:

* `object` - a serialization of the estimator class, using `pickle`, `marshal`, or `cloudpickle`.
  The serialization should be saved in ASCII encoding.
* `object_type` - one of `pickle`, `marshal`, or `cloudpickle`
* `requirements` - optional, a requirements file with a PEP 508 specifier string in ASCII encoding.
  If present, this will typically contain `python` and `sktime` version.
* any number of other files and folders, as required by `load_from_file` method in the deserialization of `object`


### Code design: estimator methods

The following methods are added to `BaseObject` as abstract with a default:

saving: `save(self, file=None) -> Optional`

`save` is an *object* method, and serializes the host object.

`file` can be in one of the python file location specifier formats.

If an argument is provided, it must point to a valid file location, and the return is `None`.
`self` is serialized to the file location, in `skt` format.

If no argument is provided, an in-memory serialized object is returned from `self`.


loading: `load_from_serial(obj) -> BaseObject`, `load_from_path(obj) -> BaseObject`

`load_from_serial` and `load_from_path` are *class methods* and implement default deserialization.

`load_from_path` takes a reference to an `skt` file, `load_from_serial` takes an in-memory serialized objcet.

The returns are the deserialized objects, newly constructed.


The above methods, `save`, `load_from_serial`, `load_from_path` are considered abstract with a default.

I.e., implementers of classes that require custom serialization should override these functions.


### Code design: load function

We further propose to add a generic `load` method.

The key requirement is that no class specific logic is present in `load`.

`load` does the following:

* if the argument is in-memory, it unpacks the in-memory serialization to obtain the class
  if the serialized object, and calls its `load_from_serial` to produce the deserialized object.
* if the argument is a file location, it must point to an `skt` file.
  In this case, `load` unpacks the file, obtains the class from the `object` file in the archive,
  and calls its `load_from_file`, on the file,  to produce the deserialized object.


### Dealing with deep learning models

Currently, `sktime` deep learning models are based on `keras`/`tensorflow`.

The strategy to deal with such models would be as follows:

1. an intermediate base class for such deep learning models (`tensorflow` based) is introduced
2. that intermediate base class overrides `save`, `load_from_file`, `load_from_serial` with
  functionality using `keras`/`tensorflow` specific serialization
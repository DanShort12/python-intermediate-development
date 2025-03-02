---
title: "Persistence"
teaching: 25
exercises: 25
questions:
- "How can we store and transfer structured data?"
- "How can we make it easier to substitute new components into our software?"
objectives:
- "Describe how the environment in which software is used may constrain its design."
- "Identify common components of multi-layer software projects."
- "Define serialization and deserialization."
- "Store and retrieve structured data using an appropriate format."
- "Define what is meant by a contract in the context of Object Oriented design."
- "Explain the benefits of contracts and implement software components which fulfill them."
keypoints:
- "Planning software projects in advance can save a lot of effort later - even a partial plan is better than no plan at all."
- "The environment in which users run our software has an effect on many design choices we might make."
- "By breaking down our software into components with a single responsibility, we avoid having to rewrite it all when requirements change."
- "These components can be as small as a single function, or be a software package in their own right."
- "When writing software used for research, requirements *always* change."
---

## Introduction

Our patient data system so far can read in some data, process it, and display it to people.
What's missing?

Well, at the moment, if we wanted to add a new patient or perform a new observation, we would have to edit the input CSV file by hand.
We might not want our staff to have to manage their patients by making changes to the data by hand, but rather provide the ability to do this through the software.
That way we can perform any necessary validation (e.g. inflammation measurements must be a number) or transformation before the data gets accepted.

If we want to bring in this data, modify it somehow, and save it back to a file, all using our existing MVC architecture pattern, we'll need to:

- Write some code to perform data import / export (**persistence**)
- Add some views we can use to modify the data
- Link it all together in the controller

## Serialization and Serializers

The process of converting data from an object to and from storable formats is often called **serialization** and **deserialization** and is handled by a **serializer**.
Serialization is the process of exporting our structured data to a usually text-based format for easy storage or transfer, while deserialization is the opposite.
We're going to be making a serialiser for our patient data, but since there are many different formats we might eventually want to use to store the data, we'll also make sure it's possible to add alternative serializers later and swap between them.
So let's start by creating a base class to represent the concept of a serializer for our patient data - then we can specialise this to make serializers for different formats by inheriting from this base class.

By creating a base class we provide a contract that any kind of patient serializer must satisfy.
If we create some alternative serializers for different data formats, we know that we will be able to use them all in exactly the same way.
This technique is part of an approach called **design by contract**.

We'll call our base class `PatientSerializer`.

~~~ python
# file: inflammation/serializers.py

from inflammation import models


class PatientSerializer:
    model = models.Patient

    @classmethod
    def serialize(cls, instances):
        raise NotImplementedError

    @classmethod
    def save(cls, instances, path):
        raise NotImplementedError

    @classmethod
    def deserialize(cls, data):
        raise NotImplementedError

    @classmethod
    def load(cls, path):
        raise NotImplementedError
~~~
{: .language-python}

Our serializer base class has two pairs of classmethods, one to serialize (save) the data and one to deserialize (load) it.
We're not actually going to implement any of them quite yet as this is just a template for how our real serializers should look, so we'll raise `NotImplementedError` to make this clear if anyone tries to use this class directly.
The reason we've used classmethods is that we don't need to be able to pass any data in using the `__init__` method, as we'll be passing the data to be serialized directly to the `save` function.

There are many different formats we could use to store our data, but a good one is **JSON** (JavaScript Object Notation).
This format comes originally from JavaScript, but is now one of the most widely used serialization formats for exchange or storage of structured data, used across most common programming languages.

Data in JSON format is structured using nested **arrays** (very similar to Python lists) and **objects** (very similar to Python dictionaries).
For example, we're going to try to use this format to store data about our patients:

~~~ json
[
    {
        "name": "Alice",
        "observations": [
            {
                "day": 0,
                "value": 3
            },
            {
                "day": 1,
                "value": 4
            }
        ]
    },
    {
        "name": "Bob",
        "observations": [
            {
                "day": 0,
                "value": 10
            }
        ]
    }
]
~~~
{: .language-json}

Compared to the CSV format, this gives us much more flexibility to describe complex structured data.
If we wanted to represent this data in CSV format, the most natural way would be to have two separate files: one with each row representing a patient, the other with each row representing an observation.
We'd then need to use a unique identifier to link each observation record to the relevant patient.
This is how relational databases work, but it would be quite complicated to manage this ourselves with CSVs.

Now if we're going to follow TDD (Test Driven Development), we should write some test code.
Our JSON serializer should be able to save and load our patient data to and from a JSON file, so for our test we could try these save-load steps and check that the result is the same as the data we started with.
Again you might need to change these examples slightly to get them to fit with how you chose to implement your `Patient` class.

~~~ python
# file: tests/test_serializers.py

from inflammation import models, serializers

def test_patients_json_serializer():
    # Create some test data
    patients = [
        models.Patient('Alice', [models.Observation(i, i + 1) for i in range(3)]),
        models.Patient('Bob', [models.Observation(i, 2 * i) for i in range(3)]),
    ]

    # Save and reload the data
    output_file = 'patients.json'
    serializers.PatientJSONSerializer.save(patients, output_file)
    patients_new = serializers.PatientJSONSerializer.load(output_file)

    # Check that we've got the same data back
    for patient_new, patient in zip(patients_new, patients):
        assert patient_new.name == patient.name

        for obs_new, obs in zip(patient_new.observations, patient.observations):
            assert obs_new.day == obs.day
            assert obs_new.value == obs.value
~~~
{: .language-python}

Here we set up some patient data, which we save to a file named `patients.json`.
We then load the data from this file and check that the results match the input.

With our test, we know what the correct behaviour looks like - now it's time to implement it.
For this, we'll use one of Python's built-in libraries.
Among other more complex features, the `json` library provides functions for converting between Python data structures and JSON formatted text files.
Our test also didn't specify what the structure of our output data should be, so we need to make that decision here  - we'll use the format we used as JSON example earlier.

~~~ python
# file: inflammation/serializers.py

import json

class PatientSerializer:
    model = models.Patient

    @classmethod
    def serialize(cls, instances):
        return [{
            'name': instance.name,
            'observations': instance.observations,
        } for instance in instances]

    @classmethod
    def deserialize(cls, data):
        return [cls.model(**d) for d in data]


class PatientJSONSerializer(PatientSerializer):
    @classmethod
    def save(cls, instances, path):
        with open(path, 'w') as jsonfile:
            json.dump(cls.serialize(instances), jsonfile)

    @classmethod
    def load(cls, path):
        with open(path) as jsonfile:
            data = json.load(jsonfile)

        return cls.deserialize(data)
~~~
{: .language-python}

For our `save` / `serialize` methods, since the JSON format is similar to nested Python lists and dictionaries, it makes sense as a first step to convert the data from our `Patient` class into a dictionary - we do this for each patient using a list comprehension.
Then we can pass this to the `json.dump` function to save it to a file.

As we might expect, the `load` / `deserialize` methods are the opposite of this.
Here we need to first read the data from our input file, then convert it to instances of our `Patient` class.
The `**` syntax here may be unfamiliar to you - this is the **dictionary unpacking operator**.
The dictionary unpacking operator can be used when calling a function (like a class `__init__` method) and passes the items in the dictionary as named arguments to the function.
The name of each argument passed is the dictionary key, the value of the argument is the dictionary value.

When we run the tests however, we should get an error:

~~~
FAILED tests/test_serializers.py::test_patients_json_serializer - TypeError: Object of type Observation is not JSON serializable
~~~

This means that our patient serializer almost works, but we need to write a serializer for our observation model as well!

Since this new serializer isn't a type of `PatientSerializer`, we need to inherit from a new base class which holds the design that's shared between `PatientSerializer` and `ObservationSerializer`.
Since we don't actually need to save the observation data to a file independently, we won't worry about implementing the `save` and `load` methods for the `Observation` model.

~~~ python
# file: inflammation/serializers.py

from inflammation import models


class Serializer:
    @classmethod
    def serialize(cls, instances):
        raise NotImplementedError

    @classmethod
    def save(cls, instances, path):
        raise NotImplementedError

    @classmethod
    def deserialize(cls, data):
        raise NotImplementedError

    @classmethod
    def load(cls, path):
        raise NotImplementedError


class ObservationSerializer(Serializer):
    model = models.Observation

    @classmethod
    def serialize(cls, instances):
        return [{
            'day': instance.day,
            'value': instance.value,
        } for instance in instances]

    @classmethod
    def deserialize(cls, data):
        return [cls.model(**d) for d in data]

...
~~~
{: .language-python}

Now we can link this up to the `PatientSerializer` and our test should finally pass.

~~~ python
# file: inflammation/serializers.py
...

class PatientSerializer(Serializer):
    model = models.Patient

    @classmethod
    def serialize(cls, instances):
        return [{
            'name': instance.name,
            'observations': ObservationSerializer.serialize(instance.observations),
        } for instance in instances]

    @classmethod
    def deserialize(cls, data):
        instances = []

        for item in data:
            item['observations'] = ObservationSerializer.deserialize(item.pop('observations'))
            instances.append(cls.model(**item))

        return instances

...
~~~
{: .language-python}

> ## Linking it All Together
> We've now got some code which we can use to save and load our patient data, but we've not yet linked it up so people can use it.
>
> Just like we did with the `display_patient` view in the previous section, try adding some views to work with our patient data using the JSON serializer.
> When you do this, think about the design of the command line interface - what arguments will you need to get from the user, what output should they receive back?
{: .challenge}

> ## Equality Testing
>
> When we wrote our serializer test, we said we wanted to check that the data coming out was the same as our input data, but we actually compared just parts of the data, rather than just using `assert patients_new == patients`.
>
> The reason for this is that, by default, `==` comparing two instances of a class tests whether they're stored at the same location in memory, rather than just whether they contain the same data.
>
> Add some code to the `Patient` and `Observation` classes, so that we get the expected result when we do `assert patients_new == patients`.
> When you have this comparison working, update the serializer test to use this instead.
>
> **Hint:** The method Python uses to check for equality of two instances of a class is called `__eq__` and takes the arguments `self` (as all normal methods do) and `other`.
{: .challenge}

## Design Early

By taking time to design our software for extensibility, we can save ourselves a lot of time later when requirements change.
The sooner we do this the better - ideally we should have a rough design sketched out for our software before we write a single line of code.
This design should be based around the structure of the problem we're trying to solve: what are the concepts we need to represent and what are the relationships between them.
Who will be using our software and how will the interact with it?

Software is often designed and built as part of a team, so in the next section we'll be looking at how to manage the process of software development.

> ## Advanced Challenge: Abstract Base Classes
>
> Since our `Serializer` class is designed not to be directly usable and its methods raise `NotImplementedError`, it ideally should be an abstract base class.
> An abstract base class is one which is intended to be used only by creating subclasses of it and can mark some or all of its methods as requiring implementation in the new subclass.
>
> Using Python's documentation on the [abc module](https://docs.python.org/3/library/abc.html), convert the `Serializer` class into an ABC.
>
> **Hint:** The only component that needs to be changed is `Serializer` - this should not require any changes to the other classes.
>
> **Hint:** The abc module documentation refers to metaclasses - don't worry about these.
> A metaclass is a template for creating a class (classes are instances of a metaclass), just like a class is a template for creating objects (objects are instances of a class), but this isn't necessary to understand if you're just using them to create your own abstract base classes.
{: .challenge}

> ## Advanced Challenge: CSV Serialization
>
> Try implementing an alternative serializer, using the CSV format instead of JSON.
>
> **Hint:** Python also has a module for handling CSVs - see the documentation for the [csv module](https://docs.python.org/3/library/csv.html).
> This module provides a CSV reader and writer which are a bit more flexible, but slower for purely numeric data, than the ones we've seen previously as part of NumPy.
>
> Can you think of any cases when a CSV might not be a suitable format to hold our patient data?
{: .challenge}

{% include links.md %}

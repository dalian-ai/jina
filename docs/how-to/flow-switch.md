(flow-switch)=
# How to build switches in a Flow

````{admonition} Requirements
:class: note
To follow along with this How-To, you need Jina 3.x or higher. TODO specify version.
````

In this tutorial you will gain a deeper insight into the Flow's {ref}`filter condition feature<flow-filter>`.

In a nutshell, this feature allows every Executor in a Flow do define a filter that can only be passed by Documents that
fulfill a specified `condition`.

````{admonition} See Also
:class: seealso

If you are not yet familiar with the basics of the DocArray query language (TODO link there) and how it is used to create
{ref}`filters <flow-filter>`, we recommend that you read the linked documentation pages first.
````

Here you will learn where and how to use this feature, and how you can build *switches* into your Flow.

## Why do you need a switch?

{ref}`As you know <flow-complex-topologies>`, Jina Flows can define complex topologies that include multiple Executors,
both in sequence and on parallel branches.

A simple Flow with parallel branches could be defined like so:

```python
from jina import Flow

f = (
    Flow()
    .add(name='start_exec')
    .add(name='exec1', needs='start_exec')
    .add(name='exec2', needs='start_exec')
    .needs_all()
)
f.plot()
```

```{figure} simple-parallel-flow.svg
:width: 70%
:align: center
Flow with two parallel branches.
```

A topology like this means that the Documents do not get processed by `exec1` before they go to `exec2`, and vice vers.
However, all Documents still go through all branches and all Executors.

In some scenarios, you might not want all Documents to be processed by all Executors, for example when you index Documents
representing different kinds of data, like text and images.

One way to achieve this differentiation is to use different Executor endpoints:

````{tab} Main app

```{code-block} python
---
emphasize-lines: 19, 23
---
from docarray import DocumentArray, Document
from jina import Flow
import numpy as np


f = (
    Flow()
    .add(name='start_exec')
    .add(name='ImageIndexer', uses=ImageIndexer, needs='start_exec')
    .add(name='TextIndexer', uses=TextIndexer, needs='start_exec')
    .needs_all()
)

text_data = DocumentArray([Document(text='hey there!') for _ in range(2)])
image_data = DocumentArray(
    [Document(tensor=np.random.rand(16, 16)) for _ in range(2)]
)  # dummy images
with f:
    embedded_texts = f.post(inputs=text_data, on='/index-text')
    print(embedded_texts[:, 'tags'])
    print(embedded_texts.embeddings)
    print('---------')
    embedded_images = f.post(inputs=image_data, on='/index-images')
    print(embedded_images[:, 'tags'])
    print(embedded_images.embeddings)
```
````

````{tab} Define Executors

```{code-block} python
---
emphasize-lines: 7, 15
---
from docarray import DocumentArray, Document
from jina import Executor, requests
import numpy as np


class TextIndexer(Executor):
    @requests(on='/index-text')
    def index(self, docs: DocumentArray, **kwargs):
        docs.embeddings = np.random.rand(len(docs), 3)  # dummy embeddings
        for doc in docs:
            doc.tags['embedded_by'] = 'textIndexer'


class ImageIndexer(Executor):
    @requests(on='/index-images')
    def index(self, docs: DocumentArray, **kwargs):
        docs.embeddings = np.random.rand(len(docs), 3)  # dummy embeddings
        for doc in docs:
            doc.tags['embedded_by'] = 'imageIndexer'
```
````

```terminal
[{'embedded_by': 'textIndexer'}, {'embedded_by': 'textIndexer'}]
[[0.37511057 0.14902827 0.31666838]
 [0.18466062 0.17823904 0.20046065]]
---------
[{'embedded_by': 'imageIndexer'}, {'embedded_by': 'imageIndexer'}]
[[0.37511057 0.14902827 0.31666838]
 [0.18466062 0.17823904 0.20046065]]
```

As you can see, the image data was only processed by the `ImageIndexer`, and the text data was only processed by the `TextIndexer`.

However, there are two problems with this approach:
- All Documents are sent to *all* Executor microservices, where they are ignored if the endpoint does not match. This creates unnecessary networking overhead.
- Sometimes you can't easily control the endpoints of all Executors, for example when you are using the {ref}`Jina Hub <jina-hub>` or {ref}`external Executors <external-executor>`.

To solve these problems, you can leverage filter condition to easily build a switch into your Flow.

## Define the filter conditions

In a Jina Flow, you can use the DocArray query language (TODO link to docarray docs) to specify a filter condition for every
Executor.

To do this, you pass a condition to the `condition` parameter in `flow.add()`:

```python
from jina import Flow

f = Flow().add(condition={'tags__key': {'$eq': 5}})
```

The, Documents that do not satisfy the `condition` will not reach the associated Executor.

In the use case where you are trying to separate Documents according to the data modality they hold, you need to choose
a condition accordingly.
To enable this, you first need to add a fitting `tag` to each of you Documents.

````{admonition} Note
:class: note
For the particular use case of identifying whether a field in a Document is set or not, you need this pre-processing step
of tagging each Document.
For many other use cases this is not necessary, as they can be directly handled by the query language itself. TODO link to ql docs in docaray
````

```python
from docarray import DocumentArray, Document
import numpy as np

# dummy data
text_data = [Document(text='hey there!') for _ in range(2)]
image_data = [Document(tensor=np.random.rand(16, 16)) for _ in range(2)]
data = DocumentArray(text_data + image_data)

# tag the data
for doc in data:
    if doc.text:
        doc.tags['has_text'] = 1
    if doc.tensor is not None:
        doc.tags['has_tensor'] = 1
print(data[:, 'tags'])
```

```console
[{'has_text': 1}, {'has_text': 1}, {'has_tensor': 1}, {'has_tensor': 1}]
```

Now you can define your conditions:

```python
# define filter conditions
text_condition = {'tags__has_text': {'$eq': 1}}
tensor_condition = {'tags__has_tensor': {'$eq': 1}}
```

These conditions specify that only Documents that have been tagged for a specific data modality can pass the filter.

### Try the filters outside the Flow

You can try the conditions directly on your data, outside the Flow:

```python
filtered_text_data = data.find(text_condition)
filtered_image_data = data.find(tensor_condition)

print(filtered_text_data.texts)  # print text
print('---')
print(filtered_image_data.tensors[:, :2, :2])  # print four pixels of each image
```
```console
['hey there!', 'hey there!']
---
[[[0.50535537 0.50538128]
  [0.40446746 0.34972967]]

 [[0.04222604 0.70102327]
  [0.12079661 0.65313938]]]
```

As you can see, each filter selects the Documents that contain the desired data fields.
That's exactly what you want for your filter!

## Build the Flow

Finally, you can assemble your Flow using these conditions and the Indexers from above.
This time there is no need to choose different Executor endpoints for the Indexers, since the filters take care of the
switch logic.

````{tab} Flow

```{code-block} python
---
emphasize-lines: 10, 16
---
from jina import Flow

f = (
    Flow()
    .add(name='start_exec')
    .add(
        name='ImageIndexer',
        uses=ImageIndexer,
        needs='start_exec',
        condition=tensor_condition,
    )
    .add(
        name='TextIndexer',
        uses=TextIndexer,
        needs='start_exec',
        condition=text_condition,
    )
    .needs_all()
)
```
````

````{tab} Executors

```{code-block} python
---
emphasize-lines: 7, 15
---
from docarray import DocumentArray, Document
from jina import Executor, requests
import numpy as np


class TextIndexer(Executor):
    @requests(on='/index')
    def index(self, docs: DocumentArray, **kwargs):
        docs.embeddings = np.random.rand(len(docs), 3)  # dummy embeddings
        for doc in docs:
            doc.tags['embedded_by'] = 'textIndexer'


class ImageIndexer(Executor):
    @requests(on='/index')
    def index(self, docs: DocumentArray, **kwargs):
        docs.embeddings = np.random.rand(len(docs), 3)  # dummy embeddings
        for doc in docs:
            doc.tags['embedded_by'] = 'imageIndexer'
```
````

Here you simply tell each Executor to filter its inputs according to the conditions that you specified above:
Text and image data should be treated separately.

## Index the data

If you now send your data to the Flow, you expect that only your image data will be sent to `ImageIndexer`, and only your
text data will be sent to `TextIndexer`.

Let's take a look:

```python
with f:
    embedded_docs = f.post(on='/index', inputs=data)
print(embedded_docs[:, 'tags'])
```

```console
[{'has_text': 1.0, 'embedded_by': 'textIndexer'},
{'has_text': 1.0, 'embedded_by': 'textIndexer'},
{'has_tensor': 1.0, 'embedded_by': 'imageIndexer'},
{'has_tensor': 1.0, 'embedded_by': 'imageIndexer'}]
```

And indeed, that's exactly what happens!

The Documents that have a text were processed by `TextIndexer`, and the Documents that contain an image were processed by `ImageIndexer`.
And remember, with this solution the Documents don't even get *sent* to the incorrect Executor.

## What's next

Now that you know how to use filter conditions to build switches, you can leverage this feature to build all kinds of business logic.

You could differentiate between more than just two different data modalities, you could direct requests based on the Client they come from,
ignore Documents that don't meet certain quality criteria, or route data to specialized Executors based on what the data
itself looks like. Your imagination is the limit!

## See also

- {ref}`How to run your Flow with Docker Compose <docker-compose>`
- {ref}`How to deploy your Flow on Kubernetes <kubernetes>`
- {ref}`How to scale Executors in your Flow <scale-out>`
---
layout: post
title: Managing Machine Learning Dependencies In Distributed Systems
---

<p align="center">
<img src="https://github.com/colethienes/colethienes.github.io/raw/master/images/octopus.png?sanitize=true" width="100%">
</p>

Recently, I was struggling with dependency issues while building [NBoost](https://github.com/koursaros-ai/nboost). One of the important features of NBoost is the portability of the models; we wanted the platform to be agnostic between many different types of finetuned models: Tensorflow, Pytorch, Transformers, whatever. But I really wanted NBoost to be packageable and pip-installable as well! This meant that all of the python classes needed to be added to the package. But **machine learning dependencies are HEAVY**; you shouldn't have to import pytorch (which is a heavy dependency) if you want to use a tensorflow model, and vice versa. Users should be able to cherry-pick the dependencies they need. The solution wasn't very complicated, but it worked well for our use case, so I thought I'd share it. Here it is:

### 1. Add the dependencies in `extras_require`

Nboost comes in four flavors: `nboost` (the base package), `nboost[tf]` (w/ tensorflow), `nboost[torch]` (w/ pytorch), and `nboost[all]` (with both). Our [setup.py](https://github.com/koursaros-ai/nboost/blob/master/setup.py) is set up so that users can pip install any of these options depending on their model:

```python
from setuptools import setup

setup(
    # ...
    extras_require={
        'torch': ['torch', 'transformers'],
        'tf': ['tensorflow==1.15', 'sentencepiece'],
        'all': ['torch', 'tensorflow==1.15', 'transformers'],
    },
    # ...
)
``` 

### 2. Create a model mapping in the `__init__.py`

In the [base of our package](https://github.com/koursaros-ai/nboost/blob/master/nboost/__init__.py) I added a mapping that reveals which module each model class can be found.

```python
# component => class => module
CLASS_MAP = {
    'protocol': {
        'TestProtocol': 'test',
        'ESProtocol': 'es'
    },
    'model': {
        'TestModel': 'test',
        'TransformersModel': 'transformers',
        'BertModel': 'bert_model',
        'AlbertModel': 'albert_model'
    }
}
```

This way, when a user types in `nboost --model_dir BertModel`, NBoost knows that it should only import the `bert_model` module.

### 3. Import the classes dynamically from the cli

In our [command line entrypoint](https://github.com/koursaros-ai/nboost/blob/master/nboost/cli/__init__.py), I added an `import_class` function that takes "model" (the module) and "BertModel" (the class) as arguments and returns that class (but doesn't import anything else!).

```python
import importlib, CLASS_MAP

def import_class(module: str, name: str):
    """Dynamically import class from a module in the CLASS_MAP. This is used
    to manage dependencies within nboost. For example, you don't necessarily
    want to import pytorch models everytime you boot up tensorflow..."""
    if name not in CLASS_MAP[module]:
        raise ImportError('Cannot locate %s with name "%s"' % (module, name))

    file = 'nboost.%s.%s' % (module, CLASS_MAP[module][name])
    return getattr(importlib.import_module(file), name)
```

And that's it! In NBoost, this model class gets handed to the proxy so that the proxy can initialize the model. Good luck!
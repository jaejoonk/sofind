# actapack
Simple, extendable framework for loading ACT products from disk.

## Dependencies
Currently:
* from `simonsobs`: `pixell`

All other dependencies (e.g. `numpy` etc.) are required by packages listed here, especially by `pixell`.

## Installation
Clone this repo and `cd` to `/path/to/actapack/`:
```
$ pip install .
```
or 
```
$ pip install -e .
```
to see changes to source code automatically updated in your environment.

## Setup
All users must create a file `.actapack_config.yaml` in their system's `HOME` directory. This file encodes the location on their system of products implemented in the `actapack` package. It also groups a set of these locations under data model "names," such as `dr6_default` (see `actapack/configs/datamodels`). This way, different data models may have their products in different locations on the system. 

To facilitate setup, we have provided some `.actapack_config.yaml` files for common public systems, such as `NERSC`, in the `actapack_configs` folder for users to copy. However, we recommend reading this section regardless to understand these files in case changes are necessary, or if you'd like to install `actapack` on your laptop, for instance.

We'll start with a basic example. Let's assume you wish to interact with the `dr6_default` data model. Let's also assume that `actapack` currently implements (see `actapack/products`) map and beam products in `maps.py` and `beams.py`, along with possibly other products in different modules as well. Then, your `.actapack_config.yaml` file might look like this:
```
dr6_default:
    maps:
        default_path: "/path/to/default/maps/on/this/system/"
        pwv_split_path: "/path/to/pwv_split/maps/on/this/system/"
    beams:
        default_path: "/path/to/default/beams/on/this/system/"
```
First, we must have a block for the data model (`dr6_default`) we wish to use. Under this block, we must have "product"-level blocks for each product implementation (e.g. `maps`, `beams`) we wish to interact with. The name of these "product" blocks must match the module in which the product is implemented (e.g. `maps` for `maps.py`). Finally, within each "product" block, we may have several "subproducts." These "subproducts" share the same code interface (again, in `maps.py`), but may just be different "kinds" of maps. The name of each "subproduct" is indicated by the words before `_path`. The system path for that product/subproduct is then listed. All the possible files for `pwv_split` maps should be in the directory `"/path/to/pwv_split/maps/on/this/system/"`.

A few notes:
* A user's `.actapack_config.yaml` may have several different "data model" blocks, so that they can select from their desired data model at runtime.
* It is *not* necessary to have a "product" block for every product in a data model. In this example, if you omit the `beams` block, then a call to load a beam from disk (for any beam "subproduct") would raise an error, but the other products, like `maps`, would be unaffected. Thus, users do not need to make any other changes to `actapack` or their setup if some products do not exist on their system.  
* It is *not* necessary to have a "subproduct" path for every subproduct in a product. In this example, if you omit the `pwv_split_path` from the `maps` block, then a call map methods with the keyword argument `subproduct='pwv_split'` would raise an error without affecting other subproducts. Thus, users do not need to make any other changes to `actapack` or their setup if some subproducts do not exist on their system.  
* By convention, the default "subproduct" passed to the methods in each product implementation (e.g. `read_map` in `maps.py`) is called "default". Thus, most "product" blocks in a `.actapack_config.yaml` file will have, at minimum, a `default_path`. However, there is nothing special about this name as far as whether it must be present or may be omitted from the `.actapack_config.yaml` file.

## Usage
All you need in your code is the following (e.g. for the `dr6_default` data model):
```
from actapack import DataModel

dm = DataModel.from_config('dr6_default')

my_default_map = dm.read_map(...)
my_pwv_split_map = dm.read_map(..., subproduct='pwv_split')
my_beam_fn = dm.get_beam_fn(...)
```

## If you would like to contribute a product to `actapack`
There are three steps:
1. Create a new product module in the `actapack/products` directory.
    * The module should only contain a subclass of the `actapack.products.products.Product` class.
    * There is a set prescription your subclass implementation must follow. To make it easy, a template of this implementation (for a product called `HotDog`) can be copied from `actapack.products.__init__.py`. You should *only* modify the class name and the exposed methods (not the class declaration or `__init__` method). Note the template has more detail on how to implement your product class. You can also look at e.g. `actpack.products.maps.Map` for inspiration.
2. Make sure your product is imported directly by the `actapack.products` package. For instance, if your module was named `hotdogs.py`, then add this line to `actapack.products.__init__.py`:

    ```
    from .hotdogs import *
    ```
3. Add a config (or multiple configs if you have multiple product versions etc.) to `actapack/configs/products/{module_name}/`. Following the hotdog example, there is a config file `hotdog_example.yaml` in `actapack/configs/products/hotdogs/`.
    * This must be a `.yaml` file.
    * This should contain any information to help get filenames for your product or load them from disk, such as a template filename. For instance, given a set of keyword arguments `array='pa6', freq='f090', num_splits=8, condiment='mustard'`, the `hotdog_file_template` string in `hotdog_example.yaml` would format to `pa6_f090_8way_mustard.txt` (the actual formatting would occur in your `HotDog` class's `get_hotdog_fn` method).
    * If your template filename requires additional keywords for a given `qid` than are present in any `qids` configuration files, those keywords would need to be added here. For instance, the `dr6_default_qids.yaml` file only contains `array`, `freq`, `patch`, and `daynight` keywords. The `num_splits` keyword required for the `hotdog_file_template` is thus added for each qid directly in `hotdog_example.yaml`.
    
Please commit and push your contribution to a new branch and open a PR on github! If you are updating an old config, please include it under a new file altogether so that historical products may still be loaded at runtime.
    
## If you would like to contribute a data model to `actapack`
There are one (maybe two) steps:
1. Create a new data model config in `actapack/configs/datamodels/`.
    * Like the `.actapack_config.yaml` file, this config must have a block for each product this data model will load. Note, also like the `.actapack_config.yaml` file, it is not necessary to have a block for every product in `actapack`, only those that will be functional in this data model.
    * The same goes for subproducts of a product -- include only those that will be functional in this data model. To add a subproduct, include an entry under the associated product block like `{subproduct_name}_dict`, where the value is a product `.yaml` config file in `actapack/configs/products/{module_name}`. 
    * As in the `.actapack_config.yaml` file, you should include a `default_dict` if there is an obvious "default" subproduct. 
    * This config file must also have an entry for a `qids_dict` (at the top-level), indicating one of the qid config files under `actapack/configs/qids/`.
2. Only if one of the included qid config files are not sufficient for your needs, you'll need to add another one with your qids (or add your qids to what's already there).

Please commit and push your contribution to a new branch and open a PR on github! If you are updating an old config, please include it under a new file altogether so that historical products may still be loaded at runtime.

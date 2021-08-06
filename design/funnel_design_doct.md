# Funnel Design Document

###### tags: `funnel-dev`

## Table of Contents
* [Introduction](#Introduction)
* [Design Considerations](#Design-Considerations)
    * [Assumptions and Dependencies](#Assumptions-and-Dependencies)
* [System-Overview](#System-Overview)
* [Design-Considerations](#Design-Considerations)
    * [Assumptions and Dependencies](#Assumptions-and-Dependencies)
    * [General Constraints](#General-Constraints)
    * [Goals and Guidelines](#Goals-and-Guidelines)
    * [Development Methods](#Development-Methods)
* [Architecture Strategies](#Architecture-Strategies)
* [System Architecture](#System-Architecture)
* [Detailed Systems Design](#Detailed-Systems-Design)
* [Glossary](#Glossary)
* [Appendix](#Appendix)

## Introduction 

`funnel` provides a way of **extending** an `intake-esm` catalog, providing a means of building diagnostics workflows, including both***routine*** and ***curiosity driven*** diagnonostics. Typically, data operators are developed separately from the data access api, which leads to a large amount of duplicate code, and struggles when reproducing the **workflow** within diagnostics frameworks. This package couples the data access and data operators, such that one can use these `funnel-collections` to build extensible diagnostics frameworks, around a set of common operators and and derived variables.

--- 


## System Overview
The `funnel` package takes in `intake-esm` catalogs, which represent a dataset comprised of files either on disk or in the cloud, and extends these catalogs to include modified datasets which have some set of functions applied to them. 

A "real-world" example of this would be CESM output within a diagnostics framework. The CESM data can be assembled into a data catalog using [ecgtools](https://github.com/NCAR/ecgtools), which abstracts the data access, converting from just files on disk, a more unified data access api. 

Diagnostic workflows require applying some set of **operators**, for example yearly averages or global averages, to the data. `Funnel` combines both the data access, and application of these operators within the framework, enabling essentially **extensions** of the data catalog, where one can query the catalog and this package returns the queried variables with some set of operators applied.



--- 

## Design Considerations

This package should really be thought of as an **extension** of `intake-esm` catalogs. There are three main components of functionality when designing this package:

* Applying "operators" to datasets within an `intake-esm` catalog
* Defining derived variables which can be computed from base variables within the catalog
* Cache output, providing a means to check to see if computations have been cached before

While designing this package, is is important to keep in mind what should fall within `intake-esm` versus what consitutes putting into this separate package.

We should also consider what falls within the scope of this package, and what the major engineering + user api challenges are, and what falls within ***"implementation details"***


### Assumptions and Dependencies
* One is working with `intake-esm` catalogs as the data api
* We cache the last operator in a sequence
* Operators consume and produce `xarray.Datasets`
* The actual operators and derived variables are defined **outside** of this package. `Funnel` provides the **framework** for diagnostics packages to plug into + build off of

### General Constraints
Something to keep in mind, in general, is balancing the following:
* Achieving functionality quickly and having people test it
* Thoughtfully designing this package to ensure it is well-tested, maintainable, and fits well within the diagnostics tech stack

While these two are neccessarily separate from each other, it is important to take both into consideration with designing and implementing this package. 

We should also think about how this fits in the generic diagnostics framework, and what this will look like when implemented into "diagnostics packages". 

### Goals and Guidelines
* Provide an extensible method of extending `intake-esm` catalogs, such that one can query for variables and operators are applied within a single api
* Test out the functionality + user api within key stakeholders developing the next generation of CESM diagnostics packages
* Make this package general enough to work with both **model** and **observational** datasets such that one could use this a model-ob comparison framework
* Collaborate with the outside community (outside of NCAR) once this package is well-deveopled and tested to collaborate on general diagnostic workflows

### Development Methods

So far, we have been reproducing the prototype Matt put together in [this repo](https://github.com/matt-long/ocean-metabolisms/tree/main/notebooks/funnel), with the reproduced, modified version found in the [esds-funnel repository](https://github.com/NCAR/esds-funnel)

We have since had a meeting, and decided to construct a robust design document, outlining exactly what falls within this package, what falls within `intake-esm`, and how best to move forward with development.

We will consider using the "4 + 1" software architecture model [example](https://docs.google.com/presentation/d/1hVCuxtm0f8TzDqsy2kC5nnjqNt3rsz26qgcMQOnM0RY/edit?usp=sharing), keeping in mind the following:
* Logical View - step by step high level
* Process view - more detailed data flow - step by step
* Development view - grouping steps into well-defined packages with defined functionality
* Physical view - look at the physical hardware what is contained within what

We will adopt an **agile** development framework, focusing on achieving functionality quickly, with robust tests. Once we development new components, we will test this package with the group signed up to be beta testers.

---

## Architecture Strategies

Here is a [powerpoint presentation](https://docs.google.com/presentation/d/1npBbBSWJyYNK1ePv-1R_fzeRDSxP03X7jhBZbN4KKCU/edit?usp=sharing) which includes the four main views described above.

_Strategies that will be used: https://techbeacon.com/app-dev-testing/top-5-software-architecture-patterns-how-make-right-choice_

---

## System Architecture
_An overview of how the functionality and responsibilities of funnel are divided and assigned to sub-systems_

- Collection
    - Data access via `intake-esm`
    - Allow user to input a query of the catalog
- Apply_Operators
    - Functions assume working with `xarray.Datasets`
    - Some sort of way to map operators with datasets (ex. ```Collection.map(calc_pco2)```)
    - Coupled with the Collection class
    - Have a way of knowing about the underlying catalog (ex. `query_dependent_operators`)
- Derived Variables
    - Decorator to define + plug into this interface
    - Collect a `derived_variable_database`
- Caching
    - Implementation detail for now
        - SQL database
        - Directory of YAML files
    - Have a way for people to share this database
    - Key- parallel writes and reads since many people will be using

---


## Detailed System Design
_Describe components and subcomponents_

- `Collection`
    - `to_dataset_dict()` - similar api to `intake-esm`, dictionary of datasets 
    - `apply_operators()` - way to map operators to the catalog
    - `save()` - save the modified, extended catalog
- `Derive`
    - `@register_derived_variable(variable_name='', dependent_variables=[''])`
    - `Derived_Variable_Registry`
- `Cache`
    - `cache_store` - base way of defining directory to cached output
    - `base_db`
        - High-level, in-memory representation of the cached output (`pd.Dataframe`)
    - `SQLdb` - SQL-lite database of cached output
        - `.get(key)` - gets the cached object (dataset) using key
        - `.put(key)` - saves the cached object (dataset) using key
    - `YAMLdb`
        - `.get(key)` - gets the cached object (dataset) using key
        - `.put(key)` - saves the cached object (dataset) using key

---

## Glossary
* Collection
    * the high level api which combines an `intake-esm` catalog with set of operators
* Derived Variable
    * A variable which can be computed from assets within a given catalog
    * Assume working with xarray dataset which contains the `dependent_variables`
* Diagnostics
    * A process which allows one to evaluate model performance using a set of tools, such as the Python stack
    * Typically comparing models with other models, or models with observations
    * Key to gathering insight about whether models are performing well
* Framework
    * Overarching workflow which encapsulates data acccess, data operations, and visualization
* Operators
    * Functions with consume and produce `xarray.Datasets`
    * Typically result in data reduction
    * Can come from wide variety of external packages (ex. geocat-comp, pop-tools)


---
## Appendix

- From Deepak

```python=
Collection["pCO2"] = Collection.map(calc_pco2)  # "derived" var + cached
Collection.map(geocat_comp.monthly_climatology) -> Collection
```

- [CMIP6 Pre-processing](https://cmip6-preprocessing.readthedocs.io/en/latest/?badge=latest)
- Funnel predecessor design doc: https://hackmd.io/GAaKFCgbSkW0pvpCFqi3eA

- [Slideshow from funnel dev meeting 8/4/21](https://docs.google.com/presentation/d/1xwbn39ks3FdNza-sl2rVcUzZyYcQQdE7Qgc9mY9YeQ0/edit?usp=sharing)

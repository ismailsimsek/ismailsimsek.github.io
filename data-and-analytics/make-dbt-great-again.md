---
layout: default
title: Make dbt great again
parent: "Data and Analytics Blog Post"
nav_order: 3
---

# Make dbt great again

dbt is a popular solution for batch data processing in data analytics. While it operates on an open-core model, which can sometimes limit the inclusion of community features in the open-source version. no worries opendbt is here to solve it. opendbt offers a fully open-source package to address these concerns. OpenDBT builds upon dbt-core, adding valuable features without altering its core code.

How? Here’s a step-by-step example. We’ll demonstrate how to use OpenDBT to activate custom adapters and execute Python models locally within dbt. The key feature we’ll use is OpenDBT’s ability to “registering custom adapter classes”.

## The problem

Using dbt core, one could do data Transform step inside the data part-form using dbt SQL, or Python (spark, snowpark). pretty much doing Doing T of ELT however its not possible to do ET Extract and load. for eaxmple extracting data from web api and Loading it to Data platform. the very fist step of data pipeline. This task usually done with python coding outside of dbt, which hides data lineage and dataflow dependencies.

## Extract Load with opendbt, using dbt-core and python

![dbt python execution](https://github.com/memiiso/opendbt/blob/623913e5ef11febb6b75aab8e74bdb903492141e/docs/assets/dbt-local-python.png?raw=true)

External data imports with opendbt

In this post we will see how to do Extract and Load with dbt using opendbt. with the solution entire data flow is done with dbt. That means full data dependencies end to end defined by dbt and documented.

While dbt core is powerful for data transformation (the ‘T’ in ELT), it’s limited to E and L step. It cannot handle the initial Extraction and Loading (the ‘E’ and ‘L’) of data, such as fetching data from web APIs and loading it into a data platform.

Typically, these early steps are handled outside of dbt, often with Python scripts. This approach can obscure data lineage and dependencies.

In this post, we’ll explore how to use OpenDBT to perform both extraction and loading within dbt. By doing so, we can maintain end-to-end data flow visibility and dependency tracking, ensuring a more robust and transparent data pipeline.

## Using your own customized Adapter

#### Step 1: Create a Custom Adapter

We’ll start by extending an existing adapter (in this case, DuckDBAdapter) and customizing it to meet our specific needs. We'll add a new method to the adapter, making it accessible to dbt Jinja templates using the @available decorator. This method will be used to execute dbt Python models later on. see full adapter code

```python
class DuckDBAdapterV2Custom(DuckDBAdapter):

     @available 
     def submit_local_python_job(self, parsed_model: Dict, compiled_code: str): 
# python metdond to run local pyhon code, dbt pyhn models
         model_unique_id = parsed_model.get('unique_id') 
         __py_code = f""" 
{compiled_code}

# NOTE this is local python execution so session is None
model(dbt=dbtObj(None), session=None)
"""
with tempfile.NamedTemporaryFile(suffix=f'__{model_unique_id}.py', delete=False) as fp:
fp.write(__py_code.encode('utf-8'))
fp.close()
print(f"Created temp py file {fp.name}")
Utils.runcommand(command=['python', fp.name])
```

#### Step 2: Registering the Custom Adapter
OpenDBT allows you to define custom adapters using the dbt_custom_adapter variable. To activate our custom adapter, add the following configuration to your dbt_project.yml file:

```yaml
vars:
  dbt_custom_adapter: opendbt.examples.DuckDBAdapterV2Custom
```

#### Step 3: Creating a dbt Macro to Execute Python Code

Next, we’ll create a dbt macro named execute_python.sql. This macro will pass the compiled Python code to the new method we added to our custom adapter. see full macro code

```python
{% raw %}{% materialization executepython, supported_languages=['python']%}
{%- set identifier = model['alias'] -%}
{%- set language = model['language'] -%}
{% set grant_config = config.get('grants') %}

{%- set old_relation = adapter.get_relation(database=database, schema=schema, identifier=identifier) -%}
{%- set target_relation = api.Relation.create(identifier=identifier,
schema=schema,
database=database, type='table') -%}
{{ run_hooks(pre_hooks) }}
{% call noop_statement(name='main', message='Executed Python', code=compiled_code, rows_affected=-1, res=None) %}
{%- set res = adapter.submit_local_python_job(model, compiled_code) -%}
{% endcall %}
{{ run_hooks(post_hooks) }}
{% set should_revoke = should_revoke(old_relation, full_refresh_mode=True) %}
{% do apply_grants(target_relation, grant_config, should_revoke=should_revoke) %}
{% do persist_docs(target_relation, model) %}
{{ return({'relations': [target_relation]}) }}

{% endmaterialization %}{% endraw %}
```

#### Step 4: Executing Local Python Models

Ready to Run! With everything configured, let’s create and run Python a model locally, eliminating the need for Spark or Snowpark.

##### Create Your Python Model:

Save your model code in a file named models/my_local_python_model.py. In this model, we’ll simply print the Python version and platform information to demonstrate local execution.

```python
import os
import platform

def print_info():
_str = f"name:{os.name}, system:{platform.system()} release:{platform.release()}"
_str += f"\npython version:{platform.python_version()}, dbt:{version.__version__}"
print(_str)

def model(dbt, session):
dbt.config(materialized="executepython")
print("==================================================")
print("========IM LOCALLY EXECUTED PYTHON MODEL==========")
print("==================================================")
print_info()
print("==================================================")
print("===============MAKE DBT GREAT AGAIN===============")
print("==================================================")
return None
```

#### Step 5: Execute the Python Model:

```python
dp = OpenDbtProject(project_dir='/dbt/project', profiles_dir='/dbt/project')
dp.run(command="run", args=['--select', 'my_local_python_model'])
```

The output should look something like this:

```python
09:57:11  Running with dbt=1.8.7
09:57:11  Registered adapter: duckdb=1.8.4
09:57:12  Unable to do partial parsing because config vars, config profile, or config target have changed
09:57:13  Found 4 models, 4 data tests, 418 macros
09:57:13  
09:57:15  Concurrency: 1 threads (target='dev')
09:57:15  
09:57:15  1 of 1 START python executepython model main.my_executepython_dbt_model ........ [RUN]
Created temp py file /var/folders/3l/n5dbz15s68592fk76c31hth8ffmnng/T/tmp4l6hufwl__model.dbttest.my_executepython_dbt_model.py
==================================================
========IM LOCALLY EXECUTED PYTHON MODEL==========
==================================================
name:posix, system:Darwin release:23.5.0
python version:3.9.19, dbt:1.8.7
==================================================
===============MAKE DBT GREAT AGAIN===============
==================================================
09:57:15  1 of 1 OK created python executepython model main.my_executepython_dbt_model ... [Executed Python in 0.51s]
09:57:15  
09:57:15  Finished running 1 executepython model in 0 hours 0 minutes and 2.53 seconds (2.53s).
09:57:15  
09:57:15  Completed successfully
09:57:15  
09:57:15  Done. PASS=1 WARN=0 ERROR=0 SKIP=0 TOTAL=1
```

Here’s a breakdown of the execution flow:

![dbt python execution flow](https://github.com/memiiso/opendbt/blob/main/docs/assets/dbt-custom-adapter-python.png?raw=true)

The full code, test cases, documentation and additional features are available on GitHub.

## Wrap-Up and Contributions

The project completely open-source, using the Apache 2.0 license. opendbt still is a young project and there are things to improve. Please feel free to test it, give feedback, open feature requests or send pull requests. You can see more examples and start experimenting with opendbt using github project

For Part 2 please see: Streamline Your Data Pipelines with opendbt: Leverage dbt and dlt for Effortless End-to-End ELT Workflows

_[Originally published on medium](https://medium.com/@ismail-simsek/make-dbt-great-again-ec34f3b661f5)_

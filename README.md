# MLP Product Walkthrough Catboost

## Authors

- Development Lead - Jamie McCann jamie.mccann@trustbit.tech




## Summary

This project is generated from the ML Platform Blueprint

TODO: Please put the summary here.




# Quickstart 

To set up this project, make sure you have a python and virtualenv installed. Then:
```bash

# create a new virtual environment
# this environment will keep this project and its dependencies safely isolated
virtualenv -p `which python3` venv

# activate this virtual environment in the terminal
source venv/bin/activate

# install python package into this environment in editable mode
# e.g. prediction and serving packages goes into overall dependencies section
# of pyproject.toml and are installed with:
pip install --editable '.'
```

From here you could open the folder in PyChart, VS Code.




# Architecture

Below you find the structure of the `src/mlp_product_walkthrough_catboost` folder:

- `infra` - cross-cutting infrastructure concerns like authentication, logging or vertex path mapping
- `model` - Model code including loading, saving, data processing, training and predicting methods
- `onprem` - on-the-premises server module, including API and/or Kafka endpoint
- `pipeline` - core module for the pipeline functionality: Model's methods orchestration, packaging and quality gating
- `serve` - allows to serve the model with an API
- `tools` - additional tools like smoke and load testing (ONPREM only)
- `version.py` - initially, it accesses on-the-premisses APP_VERSION argument value. Version can be also automatically bumped from the CD pipeline and hardcoded into the VERSION variable

Train (i.e. pipeline) module can use code from the predict (i.e. serving) module. **Predict module can not use any code from the train.**

This is because training requires more dependencies and services, which could become a problem when the model has to be packaged for the production.

This isn't a hard rule for the experiment phase. However, be aware that operations people will need to split the code as the project matures. So we are suggesting to keep things separate from the step_start.




# Dependencies

Dependencies for the `predict` (i.e. serving) code go into file `requirements-serve.txt`. They are always installed.

```bash
pip install --editable .
# or
pip install -r requirements-serve.txt
```

Dependencies for the `train` (i.e. pipeline) code go into file `requirements-pipeline.txt`. They are installed only when the `pipeline` component is specified (e.g. for local development or when training pipelines are deployed and executed on the server).

```bash
pip install --editable '.[pipeline]'
# or
pip install -r requirements-pipeline.txt
```

Dependencies for the `onprem` (i.e. production) code go into file `requirements-onprem.txt`. They are installed only when the `onprem` component is specified (e.g. for on-the-premisses deployment with **Tekton**).

```bash
pip install --editable '.[onprem]'
# or
pip install -r requirements-onprem.txt
```




# Structured Logging

To enable structured logging, set environment variable `DS_STRUCTURED_LOG=1`. This will switch output from loguru and FAST API to the JSON format, one JSON per line.

Note, that behavior of structured logging and schema should be consistent across all projects derived from [ML Platform](https://github.com/WALTER-GROUP/ml-platform/wiki) Product Blueprint.

Here is the log line sample, formatted for readability:

```json
{
    "message": "Uvicorn running on http://127.0.0.1:8000 (Press CTRL+C to quit)",
    "log.level": "INFO",
    "labels": {},
    "log.origin.file.name": "/Users/rinat/proj/planning-algorithm/planning/venv/lib/python3.9/site-packages/uvicorn/server.py",
    "log.origin.file.line": "207",
    "log.origin.function": "_log_started_message"
}
```

Loguru lets you [contextualize your log messages](https://loguru.readthedocs.io/en/stable/overview.html?highlight=extra#structured-logging-as-needed) via:

```python
# log message context
logger.info("loaded passwords from file {file}...", file=str(basic_auth))
# setting context
with logger.contextualize(task=task_id):
    do_something()
    logger.info("End of task")
```

Extra context information is preserved and and passed into `labels` dictionary in the output.

```json
{
    "message": "loaded passwords from file /Users/rinat/proj/planning-algorithm/planning/src/serve/secret.txt...",
    "log.level": "INFO",
    "labels": {
        "file": "/Users/rinat/proj/planning-algorithm/planning/src/serve/secret.txt"
    }
}
```

Severity is mapped from `loguru` methods to these in JSON output:

- `logger.trace` and `logger.debug`: `DEBUG` - debug or trace information.
- `logger.info`: `INFO` - routine information, such as ongoing status or performance.
- `logger.success`: `NOTICE` - normal but significant events, such as start up, shut down, or a configuration change.
- `logger.warning`: `WARNING` - warning events might cause problems.
- `logger.error`: `ERROR` - error events are likely to cause problems.
- `logger.critical`: `CRITICAL` - critical events cause more severe problems or outages.

Severity levels are consistent with logging levels defined in [Google Operations Suite - Logging](https://cloud.google.com/logging/docs/reference/v2/rest/v2/LogEntry#LogSeverity), making it easier to switch between onprem and cloud deployments.




# Model Train Tool

**Experiment** blueprint includes [command-line tool](https://github.com/WALTER-GROUP/ml-platform/wiki/Command-line-tools) to:

- generate yaml components for Vertex AI pipeline execution
- run the whole pipeline at once
- run selected steps of your pipeline, such as data loading, training and evaluation.

This is just a sample of how to structure training and model persistence. You don't have to use that.

This tool is installed automatically in your environment. Just type `mypipeline --help` to step_start exploring it.

```bash
> mypipeline --help
Usage: mypipeline [OPTIONS] COMMAND [ARGS]...

  Command-line entry point to the model training pipeline 

Options:
  --executor-context TEXT  Kubeflow execution context
  --wgs-log TEXT           stackdriver, vertex, json or default
  --help                   Show this message and exit.

Commands:
  components                Generate yaml components from cli commands...
  run-all-locally           Not a component.
  step-deploy-event         Publish a deployment event to Google PubSub
  step-eval                 Kubeflow step to evaluate a model against...
  step-init                 Load dataset into pipeline folders
  step-load                 Load dataset into pipeline folders
  step-model-batch-predict  Run a batch prediction job on Vertex AI using...
  step-model-upload         Upload a sub-model to Vertex AI under a...
  step-train                Train the model and save to a folder
```

The source code is in [src/mlp_product_walkthrough_catboost/pipeline/cli.py](src/mlp_product_walkthrough_catboost/pipeline/cli.py)




# Model Serving (API)

The product blueprint includes a model serving API that uses [FastAPI](https://fastapi.tiangolo.com). It is similar to Flask, except comes with OpenAPI integration out-of-the-box.

It is also a command-line tool (see [src/mlp_product_walkthrough_catboost/serve/server.py](src/mlp_product_walkthrough_catboost/serve/server.py)). To step_start it, execute `myserve`:

```bash
> myserve
2022-10-20 00:28:07.271 | INFO     | mlp_product_walkthrough_catboost.serve.server:serve:135 - Init model server from local/model at 127.0.0.1:8080
2022-10-20 00:28:07.274 | INFO     | mlp_product_walkthrough_catboost.serve.server:serve:138 - Loading model from <...>/local/model
2022-10-20 00:28:07.282 | DEBUG    | mlp_product_walkthrough_catboost.serve.server:serve:142 - Model loaded
2022-10-20 00:28:07.285 | INFO     | mlp_product_walkthrough_catboost.serve.server:serve:147 - Starting web server on 8080 port 127.0.0.1 to serve /predict
2022-10-20 00:28:07.422 | INFO     | uvicorn.server:serve:75 - Started server process [33512]
2022-10-20 00:28:07.424 | INFO     | uvicorn.lifespan.on:startup:47 - Waiting for application startup.
2022-10-20 00:28:07.426 | INFO     | uvicorn.lifespan.on:startup:61 - Application startup complete.
2022-10-20 00:28:07.428 | INFO     | uvicorn.server:_log_started_message:207 - Uvicorn running on http://127.0.0.1:8080 (Press CTRL+C to quit)
```

Then, open [http://127.0.0.1:8080](http://127.0.0.1:8080) in your browser. It will display an interactive documentation for the model API.

You can alternatively open [http://127.0.0.1:8080/redoc](http://127.0.0.1:8080/redoc) for another UI flavour.




# ML Platform integration

To run checks, make sure you have the [latest mlops tool](https://github.com/WALTER-GROUP/ml-platform/wiki/mlops)! Then, execute in the root folder of the repository `mlops check`:

```bash
❯ mlops check
mlops cli version: 1.0.0
Running check without explanations. Use --explain switch to show more detail
13 checks loaded
v   ProjectNameIsRequired - OK
v   ReadmeIsRequired - OK
v   DescriptionIsRequired - OK
v   LicenseIsRequired - OK
v   AuthorIsRequired - OK
v   SrcFolderIsRequired - OK
v   SrcIsNotModule - OK
v   TestsFolderIsRequired - OK
v   DontUseAssert - OK
v   DontUsePickle - OK
v   UseLoguruForLogging - OK
v   PyFilesAreValidPythonCode - OK
v   NoGenericNamesInSrc - OK
```




# Unit Tests

You can ran unit tests from the command-line interface:

```bash
❯ pytest tests
==================================== test session starts =====================================
platform darwin -- Python 3.9.16, pytest-7.4.0, pluggy-1.2.0
rootdir: /path/to/mlp-product-blueprint
configfile: pyproject.toml
testpaths: tests
plugins: anyio-3.7.1
collected 28 items                                                                           

tests/test_infra_auth.py ..                                                            [  7%]
tests/test_model.py ......                                                             [ 28%]
tests/test_onprem.py .....                                                             [ 46%]
tests/test_pipeline.py ...............                                                 [100%]

===================================== 28 passed in 1.62s =====================================
```




## How to distribute as a package

Before you can build wheels and sdists for your project, you’ll need to install the build package:
```bash
python3 -m pip install build
```

To create a source distribution:

```bash
python3 -m build --sdist
```

Then you can publish it to a package repository. Get in touch with Trustbit to help you in setting it up.
# mlp-product-walkthrough-catboost
# mlp-product-walkthrough-catboost

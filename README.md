# Conductor Python

Software Development Kit for Netflix Conductor, written on and providing support for Python.

## Quick Guide

1. Create a virtual environment
    ```shell
    $ virtualenv conductor
    $ source conductor/bin/activate
    $ python3 -m pip list
    Package    Version
    ---------- -------
    pip        22.0.3
    setuptools 60.6.0
    wheel      0.37.1
    ```
1. Install latest version of `conductor-python` from pypi
    ```shell
    $ python3 -m pip install conductor-python
    Collecting conductor-python
    Collecting certifi>=14.05.14
    Collecting urllib3>=1.15.1
    Requirement already satisfied: setuptools>=21.0.0 in ./conductor/lib/python3.8/site-packages (from conductor-python) (60.6.0)
    Collecting six>=1.10
    Installing collected packages: certifi, urllib3, six, conductor-python
    Successfully installed certifi-2021.10.8 conductor-python-1.0.7 six-1.16.0 urllib3-1.26.8
    ```
2. Create a worker capable of executing a `Task`. Example:
    ```python
    from conductor.client.http.models.task import Task
    from conductor.client.http.models.task_result import TaskResult
    from conductor.client.http.models.task_result_status import TaskResultStatus
    from conductor.client.worker.worker_interface import WorkerInterface


    class SimplePythonWorker(WorkerInterface):
        def execute(self, task: Task) -> TaskResult:
            task_result = self.get_task_result_from_task(task)
            task_result.add_output_data('key', 'value')
            task_result.status = TaskResultStatus.COMPLETED
            return task_result
    ```
    * The `add_output_data` is the most relevant part, since you can store information in a dictionary, which will be sent within `TaskResult` as your execution response to Conductor
3. Create a main method to start polling tasks to execute with your worker. Example:
    ```python
    from conductor.client.automator.task_handler import TaskHandler
    from conductor.client.configuration.configuration import Configuration
    from conductor.client.configuration.settings.authentication_settings import AuthenticationSettings
    from conductor.client.worker.sample.faulty_execution_worker import FaultyExecutionWorker
    from conductor.client.worker.sample.simple_python_worker import SimplePythonWorker


    def main():
        configuration = Configuration(
            base_url='https://play.orkes.io',
            debug=True,
        )
        task_definition_name = 'python_task_example'
        workers = [
            FaultyExecutionWorker(task_definition_name),
            SimplePythonWorker(task_definition_name)
        ]
        with TaskHandler(workers, configuration) as task_handler:
            task_handler.start_processes()
            task_handler.join_processes()


    if __name__ == '__main__':
        main()
    ```
    * This example contains two workers, each with a different execution method, capable of running the same `task_definition_name`
    * From `Configuration`, you can also:
      * Add `AuthenticationSettings`, like:
        ```
        authentication_settings=AuthenticationSettings(
            key_id='id',
            key_secret='secret'
        )
        ```
      * Add `MetricsSettings`, like:
        ```
        metrics_settings=MetricsSettings(
            directory='.',
            file_name='metrics.log', 
            update_interval=0.1
        )
        ```
4. Now that you have implemented the example, you can start the Conductor server locally:
      1. Clone [Netflix Conductor repository](https://github.com/Netflix/conductor):
          ```shell
          $ git clone https://github.com/Netflix/conductor.git
          $ cd conductor/
          ```
      2. Start the Conductor server:
          ```shell
          /conductor$ ./gradlew bootRun
          ```
      3. Start Conductor UI:
          ```shell
          /conductor$ cd ui/
          /conductor/ui$ yarn install
          /conductor/ui$ yarn run start
          ```
      You should be able to access:
      * Conductor API:
        * http://localhost:8080/swagger-ui/index.html
      * Conductor UI:
        * http://localhost:5000
5. Create a `Task` within `Conductor`. Example:
    ```shell
    $ curl -X 'POST' \
        'http://localhost:8080/api/metadata/taskdefs' \
        -H 'accept: */*' \
        -H 'Content-Type: application/json' \
        -d '[
        {
          "name": "python_task_example",
          "description": "Python task example",
          "retryCount": 3,
          "retryLogic": "FIXED",
          "retryDelaySeconds": 10,
          "timeoutSeconds": 300,
          "timeoutPolicy": "TIME_OUT_WF",
          "responseTimeoutSeconds": 180,
          "ownerEmail": "example@example.com"
        }
      ]'
    ```
6. Create a `Workflow` within `Conductor`. Example:
    ```shell
    $ curl -X 'POST' \
        'http://localhost:8080/api/metadata/workflow' \
        -H 'accept: */*' \
        -H 'Content-Type: application/json' \
        -d '{
        "createTime": 1634021619147,
        "updateTime": 1630694890267,
        "name": "workflow_with_python_task_example",
        "description": "Workflow with Python Task example",
        "version": 1,
        "tasks": [
          {
            "name": "python_task_example",
            "taskReferenceName": "python_task_example_ref_1",
            "inputParameters": {},
            "type": "SIMPLE"
          }
        ],
        "inputParameters": [],
        "outputParameters": {
          "workerOutput": "${python_task_example_ref_1.output}"
        },
        "schemaVersion": 2,
        "restartable": true,
        "ownerEmail": "example@example.com",
        "timeoutPolicy": "ALERT_ONLY",
        "timeoutSeconds": 0
      }'
    ```
7. Start a new workflow:
    ```shell
    $ curl -X 'POST' \
        'http://localhost:8080/api/workflow/workflow_with_python_task_example' \
        -H 'accept: text/plain' \
        -H 'Content-Type: application/json' \
        -d '{}'
    ```
    You should receive a *Workflow ID* at the *Response body*
    * *Workflow ID* example: `8ff0bc06-4413-4c94-b27a-b3210412a914`
    
    Now you must be able to see its execution through the UI.
    * Example: `http://localhost:5000/execution/8ff0bc06-4413-4c94-b27a-b3210412a914`
8. Run your Python file with the `main` method

## Unit Tests

### Simple validation

```shell
/conductor-python/src$ python3 -m unittest -v
test_execute_task (tst.automator.test_task_runner.TestTaskRunner) ... ok
test_execute_task_with_faulty_execution_worker (tst.automator.test_task_runner.TestTaskRunner) ... ok
test_execute_task_with_invalid_task (tst.automator.test_task_runner.TestTaskRunner) ... ok

----------------------------------------------------------------------
Ran 3 tests in 0.001s

OK
```

### Run with code coverage

```shell
/conductor-python/src$ python3 -m coverage run --source=conductor/ -m unittest
```

Report:

```shell
/conductor-python/src$ python3 -m coverage report
```

Visual coverage results:

```shell
/conductor-python/src$ python3 -m coverage html
```

## C/C++ Support

### C++

1. Export your C++ functions as `extern "C"`:
   * C++ function example (sum two integers)
        ```cpp
        #include <iostream>

        extern "C" int32_t get_sum(const int32_t A, const int32_t B) {
            return A + B; 
        }
        ```
2. Compile and share its library:
   * C++ file name: `simple_cpp_lib.cpp`
   * Library output name goal: `lib.so`
        ```bash
        $ g++ -c -fPIC simple_cpp_lib.cpp -o simple_cpp_lib.o
        $ g++ -shared -Wl,-install_name,lib.so -o lib.so simple_cpp_lib.o
        ```
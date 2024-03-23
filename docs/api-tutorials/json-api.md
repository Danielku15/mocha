Mocha provides a JSON API through the command line. The primary use case for this mode is the VS Code extension but it can also be used to build automations or extensions for other IDEs.

## High level concept

The idea behind this mode is similar to the `watch` mode in which Mocha keeps running and executes tests whenever modifications are done on the watched files and directories.

In the JSON API mode Mocha will also keep track of any modifications to the available tests and allows running of these tests on demand.

The API consists of two main functional areas:

- Keeping track and interacting with the test model.
- Executing individual tests and test suites and obtaining the results.

## Starting the JSON API mode

The JSON API mode of Mocha is started by either passing the `--json-api` argument or by setting the environment variable `MOCHA_JSON_API=true` for the process.

## The protocol

As mentioend above the STDIN and STDOUT of the process are used to exchange information. Each message must be terminated with a newline character `\n`. Platform Specific line endings like `\r\n` work too thanks to using the [Node.js `readline` API](https://nodejs.org/api/readline.html).

Each message uses the following framing format:

`[version, messageid, messagebody]`

- `version` - (int, required) is a integer version of the message.
- `messageid` - (int, required) is a string indicating the message of the protocol.
- `messagebody` - (any, optional) is the message specific body as defined in this protocol spec.

Note: While the API itself is versionized, some message bodies might be using Mocha specific types like the Suite and Test descriptors. Applications might query in advance the Mocha version via `version request` message to decide on version specific behaviors.

### General

#### JSON API Error

**Channel:** `STDOUT`

**Description:** This message is sent in case there was a JSON API related error.

**Message Version:** 1

**Message ID:** `api error`

**Message Body:**

```json
{
  "code": "ERROR CODE",
  "text": "Error description"
}
```

- `code` - (string, mandatory) An API specific error code depending on the error.
- `text` - (string, mandatory) A human readable description of the error.

**Example**

`[1,'api error',{"code","UNKNOWN_MESSAGE","text":"Mocha does not understand the message id 'TEST'"}]`

#### Mocha Version Request

**Channel:** `STDIN`

**Description:** Request the Mocha Version to be replied as `version response`

**Message Version:** 1

**Message ID:** `version request`

**Message Body:** None

#### Mocha Version Response

**Channel:** `STDOUT`

**Description:** Provides the Mocha version.

**Message Version:** 1

**Message ID:** `version response`

**Message Body:**

```json
{
  "version": "10.3.0"
}
```

- `version` - (string, required) Like https://mochajs.org/api/mocha#version

**Example**

`[1,'version response', {"version":"10.3.0"}]`

### Test Model interaction

The messages in this category allow consumers to keep track of the general test model consisting of test suites and tess in a hierarchical structure.

#### Test Model Request

**Channel:** `STDIN`

**Description:** Request the full test model to be provided. This will result in a `model reply` being sent.

**Message Version:** 1

**Message ID:** `model request`

**Message Body:**

```json
{
  "refresh": true
}
```

- `refresh` - (boolean, optional) If set to true, the whole test model will be reloaded from disk before providing it.

#### Test Model Reply

**Direction:** `STDOUT`

**Description:** The full test model as currently stored in Mocha.

**Message Version:** 1

**Message ID:** `model reply`

**Message Body:**

```json
{
  "id": "abcdefg1234568",
  "title": "Test Root",
  "root": true,
  "location": {
    "file": ".mocharc.js",
    "line": 0,
    "column": 0
  },
  "tests": [],
  "suites": []
}
```

- The body contains a Mocha version specific `Suite` object holding the information about the whole test model.
- The top level object is the root suite referencing the related Mocha configuration file as `location`.
- Any user defined test suites and tests are listed nested.

TODO: public documentation for the `Suite` and `Test` classes needed.

#### Test Model Modification

**Direction:** `STDOUT`

**Description:** Whenever Mocha detects changes in the test files and the test model is updated, this event provides the related modifications.

**Message Version:** 1

**Message ID:** `model modify`

**Message Body:**

```json
{
  "added": [
    {
      "type": "Suite",
      "parent": "abcdef1234",
      "index": 0,
      "data": {
        // Suite
      }
    }
  ],
  "removed": [],
  "changed": []
}
```

- `added` - (optional, array) Contains a list of items which were added to the model.
- `removed` - (optional, array) Contains a list of items which were removed from the model.
- `changed` - (optional, array) Contains a list of items which were changed in the model (e.g. name changed).
- Each item has following properties:
  - `type` - (required, string) The type of the item stored in `data` (e.g. `Suite` and `Test`)
  - `parent` - (required, string) The ID of the parent node (e.g. if a test was added to a suite, this is the id of the parent test suite).
  - `index` - (required, number) The index to which the item was added in the parent node (e.g. if a test was added in the middle).
  - `data` - (required, object) The item metadata itself.

### Test Execution

The messages in this category allow you to execute test runs and receive the test outputs and results.

#### Start a test run

**Channel:** `STDIN`

**Description:** Request a new test run to be executed

**Message Version:** 1

**Message ID:** `run start`

**Message Body:**

```json
{
  "run": "custom-id",
  "ids": ["suite01", "suite02", "test01", "test02"]
}
```

- `run` - (string, required) A custom ID identifying this test run. This ID will be used for any events emitted as part of this test run.
- `ids` - (string[], required) An array containing IDs of test suites and tests from the model to be executed in the run.

#### Cancel a test run.

**Channel:** `STDIN`

**Description:** Cancel a new test run or item which is currently executed

**Message Version:** 1

**Message ID:** `run cancel`

**Message Body:**

```json
{
  "run": "custom-id",
  "ids": ["suite01", "test02"],
  "force": true
}
```

- `run` - (string, required) The ID of the run which should be affected by the cancel.
- `ids` - (string[], optional) A list of IDs which should be cancelled. If no IDs are provided the whole test run will be cancelled.
- `force` - (boolean, optional) If set to true, the test run will be cancelled immediately without waiting for a graceful shutdown. This is useful if tests appear to be stuck and a clean shutdown is not working. Cannot be combined with `ids`, only whole test runs can be force cancelled.

#### Test Run Begin

**Channel:** `STDOUT`

**Description:** Marks the begin of a test run.

**Message Version:** 1

**Message ID:** `run start`

**Message Body:**

```json
{
  "run": "custom-id",
  "testCount": 100
}
```

- `run` - (string, required) The ID of the run as specified on the start.
- `testCount` - (number, required) The total number of tests which will be executed.

#### Test Run End

**Channel:** `STDOUT`

**Description:** Marks the end of a test run.

**Message Version:** 1

**Message ID:** `run end`

**Message Body:**

```json
{
  "run": "custom-id",
  "outcome": "cancelled",
  "duration": 8000
}
```

- `run` - (string, required) The ID of the run as specified on the start.
- `outcome` - (string, required) A string describing the outcome of the test run.
  - `pass` - All tests passed (cancelled or skipped tests are treated as passed).
  - `fail` - One or more tests failed.
  - `cancel` - The whole test run was cancelled.
- `duration` - (number, required) The duration of the test run in milliseconds.

#### Test Suite Begin

**Channel:** `STDOUT`

**Description:** Marks the begin of a test suite.

**Message Version:** 1

**Message ID:** `suite start`

**Message Body:**

```json
{
  "run": "custom-id",
  "id": "suite01",
  "testCount": 30
}
```

- `run` - (string, required) The ID of the run as specified on the start.
- `id` - (string, required) The ID of the test suite.
- `testCount` - (number, required) The number of tests which will be executed in this suite.

#### Test Suite End

**Channel:** `STDOUT`

**Description:** Marks the end of a test suite.

**Message Version:** 1

**Message ID:** `suite end`

**Message Body:**

```json
{
  "run": "custom-id",
  "id": "suite01",
  "outcome": "pass",
  "testCount": 30,
  "duration": 8000
}
```

- `run` - (string, required) The ID of the run as specified on the start.
- `id` - (string, required) The ID of the test suite.
- `outcome` - (string, required) A string describing the outcome of the test suite.
  - `pass` - All tests passed (cancelled or skipped tests are treated as passed).
  - `fail` - One or more tests in the suite failed.
  - `skipped` - If the whole test suite was skipped.
  - `cancel` - The test suite was cancelled.
- `testCount` - (number, required) The number of tests executed.
- `duration` - (number, required) The duration of the tests in this test suite in milliseconds.

#### Test Begin

**Channel:** `STDOUT`

**Description:** Marks the begin of a test.

**Message Version:** 1

**Message ID:** `test start`

**Message Body:**

```json
{
  "run": "custom-id",
  "id": "test01"
}
```

- `run` - (string, required) The ID of the run as specified on the start.
- `id` - (string, required) The ID of the test.

#### Test End

**Channel:** `STDOUT`

**Description:** Marks the end of a test.

**Message Version:** 1

**Message ID:** `test end`

**Message Body:**

```json
{
  "run": "custom-id",
  "id": "test01",
  "outcome": "pass",
  "error": {
    "type": "TypeError",
    "message": "A type error occured.",
    "stack": "",
    "cause": {}
  },
  "duration": 8000
}
```

- `run` - (string, required) The ID of the run as specified on the start.
- `id` - (string, required) The ID of the test.
- `outcome` - (string, required) A string describing the outcome of the test.
  - `pass` - The test passed.
  - `fail` - The test failed.
  - `skipped` - The test was skipped.
  - `cancel` - The test was cancelled.
- `error` - (Error, optional) A serialized `Error` which caused the test to fail.
- `duration` - (number, required) The duration of the test in milliseconds.

#### Test Console Output

**Channel:** `STDOUT`

**Description:** An output was received on `STDOUT` or `STDERR` during the test execution. Due to features like parallel tests, outputs cannot be always related to a dedicated test or test suite. It is up to the receiver to decide how to attach and treat those outputs. Note that output might be buffered or not received line-wise. This message should be treated as a chunk of data as received on the respective channel.

**Message Version:** 1

**Message ID:** `test output`

**Message Body:**

```json
{
  "run": "custom-id",
  "channel": "stderr",
  "text": ""
}
```

- `run` - (string, required) The ID of the run as specified on the start.
- `channel` - (string, required) Whether the output was received on `stderr` or `stdout`.
- `text` - (string, required) The raw text as received on the output.

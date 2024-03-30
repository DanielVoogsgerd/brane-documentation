# POSIX reasoner

The POSIX policy reasoner is based on POSIX file permissions. It aims to be a simple and accessible alternative to the
default eFLINT reasoner. This simplicity should make it easy to try out Brane. However, if you need to be able
to express complex rules/policies, then stick with the default eFLINT reasoner.

## Introduction

The POSIX reasoner relies on the POSIX file permissions of the files of a dataset. The reasoner uses the POSIX file
permissions (read, write, execute) set for the `Owner`, `Group`, and `Others` classes on the dataset's files. In
addition to these files, there is a simple POSIX policy file that has to be set up. This policy file will map/link a
global username to a user identifier and group identifiers of a user and groups on the machine where the dataset is
stored. The global username here could for example be `Amy`. `Amy` in this case would be a data scientist that wants to
execute a Brane workflow (the global name is specified in the workflow).

Let's take a look at the POSIX policy file below. `hospital_a` defines the Brane location. This location is used to
determine what mappings will be used. Next up is the global username mapping discussed earlier. Here we see that `Amy`
maps onto user ID `1000` and group IDs `101`, `102`, and `103`.

```yaml
# example-posix-policy.yml
description: This file contains the policy for the hospital_a dataset.
version: 0.1.0
version_description: The policy is used to define the users and groups that have access to the dataset.
content:
  - reasoner: posix
    reasoner_version: 0.0.1
    content:
      hospital_a:
        user_map:
          Amy:
            uid: 1000
            gids:
              - 101
              - 102
              - 103
          Ben:
            uid: 1001
            gids:
              - 101
```

Aside from the file permissions and the policy file, one more file is required. This is the standard dataset identifier
file which is also used for the eFLINT reasoner. As is shown below this file defines the files that the dataset consists
of.

```yaml
# hospital_a/dataset_1.yml
name: hospital_a_dataset_1
owners: null
description: A test dataset.
created: 1970-01-01T00:00:00Z
access:
  localhost: !file
    path: ./data.txt
```

Now we're ready to discuss the full process. Given a workflow the reasoner will first gather all datasets that are being
read from and written to in that workflow. Then using the Brane location (e.g., `hospital_a`) it will map the workflow's
global user (e.g., `Amy`) to the user ID and group IDs. Finally, the datasets that are accessed their dataset identifier
files are read to figure out which files those datasets consist of. The reasoner subsequently reads out the POSIX
permissions of these files and checks if the user ID, group IDs (or Others) provide the required read/write/execute
access. Thus, in essence the POSIX reasoner verifies whether the global user (`Amy`) has sufficient permissions by
mapping to and then using a user ID and group IDs on the local machine.

## Tutorial

This tutorial uses the `policy-reasoner` and `policy-reasoner-gui` to demonstrate the new POSIX policy reasoner.

### Prerequisites


Create a `brane` directory to store all Brane related repositories.

Clone the [policy reasoner](https://github.com/epi-project/policy-reasoner) repository.

Clone the [policy reasoner GUI](https://github.com/epi-project/policy-reasoner-gui) repository.

Clone the [eflint-server-go](https://github.com/epi-project/eflint-server-go) repository.

`cd eflint-server-go/cmd/eflint-to-json` and compile the binary using `go build`. To cross compile for macOS (M1, M2, M3), you can run `GOOS=darwin GOARCH=arm64 go build -o eflint-to-json`.

`cd policy-reasoner` and create a `.env` file with the following content:
```bash
EFLINT_TO_JSON_PATH=../eflint-server/eflint-to-json`
DATA_INDEX=tests/data/`
```

Install the `diesel_cli` using `cargo install diesel_cli --no-default-features
--feature sqlite`.

Install the `yq` command line tool using for example `brew install yq` or `sudo apt install yq`.

### Demonstration

We'll be using the example test datasets already set up in the policy reasoner repository. Navigate to `policy-reasoner/tests/data/umc_utrecht_ect/` and take a look at the `test.yml` file. This is the dataset identifier file as described in the introduction above. The `test.txt` file is the file that contains the dataset's actual data.

Under `policy-reasoner/tests/management` you'll find a `posix-policy.yml` file. `stat posix-policy.yml` will (among other metadata) show the file owner. It should be your current user account. Run `id` and note the `uid`. Now in the `posix-policy.yml` file change the `uid` in the mapping for the location `umc_utrecht_ect` and for the global user `test` to the `uid` of your own user account.

With the POSIX policy defined in `posix-policy.yml` navigate back to the root directory of the `posix-reasoner` repository and generate a JSON version of the POSIX policy file with `yq -o=json -I=0 '.' ./tests/management/posix-policy.yml > ./tests/management/posix-policy.json` (verified with `yq` version `4.43.1` on macOS).

We're now ready to add and activate the policy.

Start the POSIX policy reasoner from `policy-reasoner` with `cargo run --bin posix`.

Then from `policy-reasoner` run:
```bash
export JWT_DELIB="$(cat ./jwt_delib.json)"
export JWT_EXPERT="$(cat ./jwt_expert.json)"
```

```bash
# Create the Posix policy
curl -X POST -H "Authorization: Bearer $JWT_EXPERT" -H "Content-Type: application/json" -d "@tests/management/posix-policy.json" localhost:3030/v1/management/policies
```

```bash
# Activate the Posix policy
curl -X PUT -H "Authorization: Bearer $JWT_EXPERT" -H "Content-Type: application/json" -d '{"version": 1 }' localhost:3030/v1/management/policies/active
```


Now also start the policy reasoner GUI from `policy-reasoner-gui` with `cargo run`.

Navigate to [http://localhost:3001/deliberation](http://localhost:3001/deliberation).

From `policy-reasoner` run `cat ./jwt_delib.json` and paste the token into the prompt on the webpage.

On the webpage change the request type (in the top left) to `Validate workflow`.

Then enter the workflow below and press the `Execute` button. The verdict is shown on the right. This workflow should be allowed.
```js
#[on("surf")]
{
  let data0 := new Data { name := "umc_utrecht_ect" };

  commit_result("umc_utrecht_ect", data0);
}
```
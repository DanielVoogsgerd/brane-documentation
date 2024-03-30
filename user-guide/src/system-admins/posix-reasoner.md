# POSIX reasoner

The POSIX policy reasoner is based on POSIX file permissions. It aims to be a simple and accessible alternative to the
default eFLINT reasoner. This simplicity should make it easy to try out Brane. However, if you need to be able
to express complex rules/policies, then stick with the default eFLINT reasoner.

## Introduction

The POSIX reasoner relies on the [POSIX file
permissions](https://en.wikipedia.org/wiki/File-system_permissions#POSIX_permissions) of the files of a dataset. The
reasoner uses the POSIX file permissions (read, write, execute) set for the `Owner`, `Group`, and `Others` classes on
the dataset's files. In addition to these files, there is a simple POSIX policy file that has to be set up. This policy
file will map/link a global username to a user identifier and group identifiers of a user and groups on the machine
where the dataset is stored. The global username here could for example be `Amy`. `Amy` in this case would be a data
scientist that wants to execute a Brane workflow (the global name is specified in the workflow).

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

This tutorial uses the `policy-reasoner` and `policy-reasoner-gui` to set up and demonstrate the new POSIX policy
reasoner.

### Prerequisites

Create a `brane` directory to store all Brane related repositories.

Clone the [policy reasoner](https://github.com/epi-project/policy-reasoner) repository.

Clone the [policy reasoner GUI](https://github.com/epi-project/policy-reasoner-gui) repository.

Clone the [eflint-server-go](https://github.com/epi-project/eflint-server-go) repository.

`cd eflint-server-go/cmd/eflint-to-json` and compile the binary using `go build`. To cross compile for macOS (M1, M2, M3), you can run `GOOS=darwin GOARCH=arm64 go build -o eflint-to-json`.

`cd policy-reasoner` and create a `.env` file with the following content:
```bash
EFLINT_TO_JSON_PATH=../eflint-server/eflint-to-json
DATA_INDEX=./tests/data
```

Install the `diesel_cli` using `cargo install diesel_cli --no-default-features
--feature sqlite`.

Install the `yq` command line tool using for example `brew install yq` or `sudo apt install yq`.

### Configuration

#### Environment

This section describes the environment that is used for this tutorial.

We'll be using the example test datasets already set up in the policy reasoner repository. Navigate to
`policy-reasoner/tests/data/umc_utrecht_ect/` and take a look at the `test.yml` file. This is the dataset identifier
file as described in the introduction above. The `test.txt` file is the file that contains the dataset's actual data. Run `stat -l test.txt` to see the file's metadata. Make note of the file owner's username and the group ID.

Under `policy-reasoner/tests/management` you'll find a `posix-policy.yml` file. This is the POSIX policy which maps the
global username (e.g., `test`) to a local user ID and group IDs. If the file owner in the previous step was your own
username, then use the `id` command to see the `uid` of your user account. If you want a global user to have access to
the file, then the POSIX mapping should map from their global username to either the `uid` of the file owner or to a
group ID of the file owner.

#### Updating the POSIX policy

This section defines how the POSIX policy can be adjusted, loaded into the reasoner, and activated. It consists of 3
steps.

**0. Set up environment**

For the commands below you'll need the following environment variables. From `policy-reasoner` run:
```bash
export JWT_DELIB="$(cat ./jwt_delib.json)"
export JWT_EXPERT="$(cat ./jwt_expert.json)"
```

Then start the POSIX policy reasoner from `policy-reasoner`.
```bash
cargo run --bin posix
```

**1. Convert POSIX policy YAML to JSON**

From the `posix-policy.yml` directory. Make any adjustments you'd like to make to the `policy-reasoner/tests/management/posix-policy.yml` file. Convert the YAML policy to JSON with the command below.

```bash
# (verified with `yq` version `4.43.1`)
yq -o=json -I=0 '.' ./tests/management/posix-policy.yml > ./tests/management/posix-policy.json
```

**2. Create/load the policy file**

```bash
curl -X POST -H "Authorization: Bearer $JWT_EXPERT" -H "Content-Type: application/json" -d "@tests/management/posix-policy.json" localhost:3030/v1/management/policies
```

Note that the response will include the version number of the policy that was just added/created. This version number is
set in the `curl` below.

**3. Activate the new policy**

As noted in the previous step, make sure to update the version number in the command below.

```bash
curl -X PUT -H "Authorization: Bearer $JWT_EXPERT" -H "Content-Type: application/json" -d '{"version": 1 }' localhost:3030/v1/management/policies/active
```

#### Validate workflows

With the policy in place we can now validate if a workflow is valid.

In addition to the already running `policy-reasoner`, start the policy reasoner GUI from `policy-reasoner-gui`.
```bash
cargo run
```

Navigate to [http://localhost:3001/deliberation](http://localhost:3001/deliberation).

From `policy-reasoner` copy and paste the token into the prompt on the webpage.
```bash
cat ./jwt_delib.json
```

On the webpage change the request type (in the top left) to `Validate workflow`.

Then enter the example workflow below and press the `Execute` button. The verdict is shown on the right. This workflow
should be allowed.
```js
#[on("surf")]
{
  let data0 := new Data { name := "umc_utrecht_ect" };

  commit_result("umc_utrecht_ect", data0);
}
```

## Demonstration

Using the same steps as described above try updating the policy file and checking whether the workflow is allowed to be
run.

```yaml
# posix-policy.yml
description: This file contains the policy for the surf dataset.
version: 0.1.0
version_description: 
content:
  - reasoner: posix
    reasoner_version: 0.0.1
    content:
      surf:
        user_map:
          test:
            uid: 1000
            gids:
              - 101
```
1. First off, use the 3 steps provided above to update the policy in the reasoner (1) convert YAML to json, 2) create
policy, 3) activate the new policy try using provided YAML file to access the dataset. This should result in a denial
(unless the file owners uid is 1000 or the gid is 101):

2. Run `stat` on the file to discover the file owner (e.g., 502), Make sure `surf` is the location and the `test` is the
   user to refer to in the policy file. Use the `owner` from the stat command and insert it into the `uid` field of
test. In this way, you are telling the reasoner that this Brane user has the same `uid` as the owner of the file. Now,
update the policy and validate that the policy provides access.
```yaml
# Add the uid of the file owner here
uid: 502
```

3. You can also use the group id, which can be determined from the stat command, instead of the `uid`. Try changing the
   `uid` back to the original (or a completely different one) and add the `gid` of the group from the stat to the `gids`
   in the user of the YAML file and update the policy once more. Validate that the policy provides access, now it should
   be through the `gid` field rather than the `uid`.
```yaml
test:
  # Change the uid back (or to a different uid)
  uid: 1000
  gids:
    - 101
    # Add the gid here
    - 20
```

4. Global access can be modified by altering the permissions of the owner group via the `chmod command` (changing file
   permissions). Run
```bash
# Add read and write permissions to the others class
chmod o+rw
# Remove read and write permissions from the others class
chmod o-rw
```

This gives the others group read and write permissions to the dataset. Now run the 3 steps above to re-upload the policy
and validate the workflow, the policy should provide access, this time through the others group.

5. Now you can give other Brane users access to the dataset in the YAML file, by either having them in the same group
   (recommended) or giving them the id of the owner.
```yaml
test:
  # Uid of the file owner of the file
  uid: 502
  gids:
    - 101
    # Gid of the file owner of the file 
    - 20
new_user:
  # Add the uid of the file owner here if new_user should have the same permissions as the file owner 
  uid: 1000
  gids:
    # Add the gid of the file owner to the gids property to give new_user the same permissions as the owner of the file
    - 20
```
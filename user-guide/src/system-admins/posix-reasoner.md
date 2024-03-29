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

TODO: Practical. Step by step. Provide example use case, including simple workflow, mapping, etc. How to enable the
reasoner to run in Posix mode instead of eFLINT. Potentially take inspiration from this from the existing documentation.

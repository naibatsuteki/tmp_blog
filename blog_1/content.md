# How to not lose Terraform features using Terragrunt

The goal is to create a robust and scalable framework, allowing us to apply CI/CD practices to the infrastructure created with Terraform in an easy and efficient way.

The core goals:

1. Semantic versioning
2. CI/CD Driven
3. CI/CD Platform agnostic
4. Support local development
5. Support multiple environments.
6. Support environments parametrization
7. Account/Region agnostic

## Goals

Arcu odio ut sem nulla pharetra diam sit amet nisl. Proin libero nunc consequat interdum varius sit amet mattis vulputate.

### CI/CD Driven

![CI/CD](image.png)

## Terraform Overview

TODO Lorem ipsum dolor sit amet, consectetur adipiscing elit, sed do eiusmod tempor incididunt ut labore et dolore magna aliqua. Senectus et netus et malesuada fames ac. Scelerisque in dictum non consectetur a. Pretium viverra suspendisse potenti nullam. Eget arcu dictum varius duis at consectetur lorem donec massa. A lacus vestibulum sed arcu non odio. Purus sit amet luctus venenatis lectus magna. Arcu odio ut sem nulla pharetra diam sit amet nisl. Proin libero nunc consequat interdum varius sit amet mattis vulputate. In ornare quam viverra orci. Massa id neque aliquam vestibulum morbi blandit. Malesuada bibendum arcu vitae elementum curabitur vitae. Viverra adipiscing at in tellus integer feugiat scelerisque.

## Terragrunt Overview

TODO Lorem ipsum dolor sit amet, consectetur adipiscing elit, sed do eiusmod tempor incididunt ut labore et dolore magna aliqua. Adipiscing diam donec adipiscing tristique risus nec feugiat. Suscipit adipiscing bibendum est ultricies integer quis. Augue eget arcu dictum varius duis at consectetur lorem donec. Facilisi morbi tempus iaculis urna id volutpat lacus. Nisl condimentum id venenatis a. Congue nisi vitae suscipit tellus. Amet consectetur adipiscing elit ut aliquam purus. Consectetur libero id faucibus nisl tincidunt eget nullam non nisi. Diam in arcu cursus euismod quis viverra. Dui faucibus in ornare quam viverra orci sagittis eu volutpat. Urna porttitor rhoncus dolor purus. Pulvinar etiam non quam lacus suspendisse faucibus interdum posuere. Scelerisque viverra mauris in aliquam sem fringilla ut. Vel quam elementum pulvinar etiam non quam lacus suspendisse. Dolor sed viverra ipsum nunc aliquet bibendum enim. Non diam phasellus vestibulum lorem sed risus ultricies tristique. Consectetur libero id faucibus nisl.

## Standard Terraform/Terragrunt Approach

If you will follow [Terragrunt Quick Start Guide](https://terragrunt.gruntwork.io/docs/getting-started/quick-start/) you probably end with file structure similar to this.

```sh
# infrastructure-live
├── terragrunt.hcl
├── prod
│   ├── app
│   │   └── terragrunt.hcl
│   ├── mysql
│   │   └── terragrunt.hcl
│   └── vpc
│       └── terragrunt.hcl
├── qa
│   ├── app
│   │   └── terragrunt.hcl
│   ├── mysql
│   │   └── terragrunt.hcl
│   └── vpc
│       └── terragrunt.hcl
└── stage
    ├── app
    │   └── terragrunt.hcl
    ├── mysql
    │   └── terragrunt.hcl
    └── vpc
        └── terragrunt.hcl
```

The root terragrunt file will probably contain provider and remote state configuration:

```terraform
remote_state {
  backend = "s3"
  generate = {
    path      = "backend.tf"
    if_exists = "overwrite_terragrunt"
  }
  config = {
    bucket = "my-terraform-state"

    key = "${path_relative_to_include()}/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "my-lock-table"
  }
}

generate "provider" {
  path = "provider.tf"
  if_exists = "overwrite_terragrunt"
  contents = <<EOF
provider "aws" {
  assume_role {
    role_arn = "arn:aws:iam::0123456789:role/terragrunt"
  }
}
EOF
}
```
in the series
Each of the leaf terragrunt files will contain a module declaration.

```terraform
# infrastructure-live/stage/app/terragrunt.hcl
terraform {
  source = "github.com:foo/infrastructure-modules.git//app?ref=v0.0.1"
}
inputs = {
  instance_count = 3
  instance_type  = "t2.micro"
}
```

When I started my journey with Terragrunt this approach worked fine when the number of the environment was well known and infrastructure has been build from 5-10 modules. It solves multiple terraform problems especially reducing redundancy in the provider and remote state configuration and provides a very confined way to execute terraform commands for multiple modules.

But when environment infrastructure start getting more complex and number of the environments increase some problems start to appear. To begin with promotions by copping files, the configurations of environments are usually slightly different, and after each promotion is necessary to manually correct some values, in addition, you need to repeat this process for each module, which produces a high chance of making mistake.
Furthermore, you can't answer in a simple way the question: "What version of the infrastructure is currently on the `stage` environment?" or "Are versions on the `stage` and `prod` the same?" To answer this question you need to check the version of every module. ... The next issue is lack of the support for some features introduced in the newer version of the terraform like `moved`, `import` or even lack of the support for standard Terraform features like `data` block can be a pain in the neck.

## Present modified version

To address mentioned issues we need slightly modify previous structure. First reduce the number of the infrastructure definition to single instance which solve both the promotion and the versioning issue.

```sh
├── terragrunt.hcl
└── infrastructure
    ├── app
    │   └── terragrunt.hcl
    ├── mysql
    │   └── terragrunt.hcl
    └── vpc
        └── terragrunt.hcl
```

 Let's explain versioning first. We have now a single instance of each module and all environments use the same base so we can simply add a version tag eg.: `0.0.1` and this will contain versions of all modules. We can now create infrastructure artifact and use this as blueprint for as many environments as we want which solve the Promotion issue. For example, we can easily create a temporal environment and test the infrastructure upgrade process.

Unfortunately, this approach generates problems with standard remote state generation because we have conflicted keys for the terraform remote states. The fix for this issue is relatively simple to solve by introducing a few environment variables.

```hcl
locals {
  env_name                          = lower(get_env("INF_ENV_NAME"))
  aws_region                        = lower(get_env("INF_ENV_REGION"))
  tfstate_s3_bucket_name            = lower(get_env("INF_TFSTATE_S3_BUCKET_NAME"))
  tfstate_s3_bucket_region          = lower(get_env("INF_TFSTATE_S3_BUCKET_REGION"))
}

remote_state {
  backend = "s3"

  generate = {
    path      = "backend.tf"
    if_exists = "overwrite_terragrunt"
  }

  config = {
    bucket         = local.tfstate_s3_bucket_name
    dynamodb_table = local.tfstate_s3_bucket_name # Note: For simplicity the name of the DynamoDb table is the same as s3 bucket name
    encrypt      = true
    key          = lower("${local.env_name}/${path_relative_to_include()}/terraform.tfstate")
    session_name = "terraform"
    region       = local.tfstate_s3_bucket_region
  }
}

generate "provider" {
  path      = "provider.tf"
  if_exists = "overwrite_terragrunt"

  contents = <<EOF
    provider "aws" {
        region  = "${local.aws_region}"
    }
  EOF
}
```

After solving the backend configuration issue we need also to provide a method to pass values to differentiate environments.
For this purpose, we can use environment variables and configuration files.

```hcl
include "root" {
  path   = find_in_parent_folders()
  expose = true
}

include "terraform" {
  path = find_in_parent_folders("_common/terraform.hcl")
}

locals {
  json_config =jsondecode(file(find_in_parent_folders("${include.root.locals.env_name}.json")))
}

terraform {
  source = "github.com:foo/infrastructure-modules.git//app?ref=v0.0.1"
}

inputs = {
  instance_count = local.json_config.app.instance_count
  instance_type  = local.json_config.app.instance_type
  tags = {
    Environment=include.root.locals.env_name
  }
}
```

Now we can simply define as many environments as we want without changing the code base.

**Note:** Before interacting with infrastructure in different environments we need to remove `.terragrunt_cache` by executing `<rm with find command here>`.

```sh
├── terragrunt.hcl
└── infrastructure
    ├── app
    │   ├── main.tf
    │   └── terragrunt.hcl
    ├── mysql
    │   ├── main.tf
    │   └── terragrunt.hcl
    └── vpc
        ├── main.tf
        └── terragrunt.hcl
```

The difference between environments needs to be passed via environment variables or by configuration files. For more complex scenarios I recommend using configuration files but in this example, we use environment variables for bre

WHY???

- Access all the latest terraform functionalities
  - Import block
- Allow terragrunt to manage more than one module
  - Data resources
  - Multiple module blocks
  - Single resources

Provide example of the move block, import block

Discuss pros and cons of this approach.

Please review it Michał

BLOG: Requirments:

- Blog Content
  - H1-H6 support
  - Bold
  - Italic
  - Ordered list
  - Unordered list
  - External links
  - Internal links
  - TOC
  - Images
  - Optional, but embedded videos would be nice
  - Code fragments with the possibility to copy them using one click
  - Core Languages Syntax Highlighting
    - Python
    - HCL
    - Bash
    - Major programming languages blocks in general
  - Separation lines
- Functional
  - Possibility to create series
  - Buttons redirecting directly to the next and previous blog
  - **IF WE COULD WRITE DIRECTLY IN MARKDOWN AND UPLOAD/PUBLISH IT FROM SUCH A SOURCE TO THE WEBSITE IT WOULD BE GREAT – HIGHLY DESIRED FEATURE**

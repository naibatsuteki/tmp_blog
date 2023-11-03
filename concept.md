<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->
# Series topic

Create multi accounts/regions/environments infrastructure fast.

## Core tools

- AWS (Public Cloud)
- Terraform/OpenTofu (Main IaC tools)
- Terragrunt (Deployments orchestrator)

## Base version (Single account, Multi environment)

- Single account
- Single region
- Multiple environments

### Infrastructure

- Budget alerts - Singleton
- S3 Bucket for the zip based lambda artifacts - Singleton
- ECR for the docker based lambda artifacts - Singleton
- Lambda Docker based
- Lambda Zip based
- Dynamo DB
- API Gateway

## Middle version (Single/Multi account Multi environments)

## Advanced version

## Blogs Series order

1. Introduction to terraform patterns: (Intro)
   1. Monolithic
   2. Modular
   3. Something between
2. Development deployment strategies (Intro)
   1. Tree base configuration
   2. Single repository
   3. Multi repository
   4. External tool driven
3. Singleton resources deployment (Base version)
4. Environment resource deployments (Base version) <!-- Consider connect 3 and 4 in one blog post -->
5. Improve security:
   1. Replace passing values from terraform state file to AWS SecretManager
   2. Use Multiple More Granular roles
   3. Add TopLevel AWS Secret Manager instance
   4. Add Environment Level AWS Secret Manager instance
6. asd

**Notes:**:

- Consider connect `Development deployment strategies` and `Environment resource deployments` into single blog post
- Keep all modules in one git repository. Simplify CI/CD auth and CQ checks
- I need remember to introduce terraform and the terragrunt. (Where)


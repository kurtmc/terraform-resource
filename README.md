## Terraform Concourse Resource

**Warning:** This is still in pre-alpha mode, so features will change without notice. The initial stable release should be out soon!

A Concourse resource that allows jobs to modify IaaS resources via Terraform.
Useful for creating reproducible testing and production environments. No more snowflakes!

See what's in progress on the [Trello board](https://trello.com/b/s06sLNwc/terraform-resource).

## Source Configuration

### `storage`

* `storage.driver`: *Optional. Default `s3`.* The driver used to store the Terraform state file. Currently `s3` is the only supported driver.

#### `s3` Driver

* `storage.bucket`: *Required.* The S3 bucket used to store the state files.

* `storage.bucket_path`: *Required.* The S3 path used to store state files, e.g. `terraform-ci/`.

* `storage.access_key_id`: *Required.* The AWS access key used to access the bucket.

* `storage.secret_access_key`: *Required.* The AWS secret key used to access the bucket.

* `storage.endpoint`: *Optional.* The endpoint for an s3-compatible blobstore (e.g. Ceph).
  * **Note:** By default, the resource will use S3 signing version v2 if an endpoint is specified as many non-S3 blobstores do not support v4.
    Opt into v4 signing by setting `storage.use_signing_v4: true`.

### `terraform`

Terraform configuration options can be specified under `source.terraform` and/or `put.params.terraform`.
Options from these two locations will be merged, with fields under `put.params.terraform` taking precedence.

* `terraform.source`: *Required.* The location of the Terraform module to apply.
These can be local paths, URLs, GitHub repos, and more.
See [Terraform Sources](https://www.terraform.io/docs/modules/sources.html) for more examples.

* `terraform.destroy_on_failure`: *Optional. Default `false`.* If true, the resource will run `terraform destroy` if `terraform apply` returns an error.

* `terraform.vars`: *Optional.* A collection of Terraform input variables.
These are typically used to specify credentials or override default module values.
See [Terraform Input Variables](https://www.terraform.io/intro/getting-started/variables.html) for more details.

### Example

**Note:** Declaring custom resources under `resource_types` requires Concourse 1.0 or newer.

```yaml
resource_types:
- name: terraform
  type: docker-image
  source:
    repository: ljfranklin/terraform-resource

resources:
  - name: terraform
    type: terraform
    source:
      storage:
        bucket: mybucket
        bucket_path: terraform-ci/
        access_key_id: {{storage_access_key}}
        secret_access_key: {{storage_secret_key}}
      terraform:
        # the '//' indicates a sub-directory in a git repo
        source: github.com/ljfranklin/terraform-resource//fixtures/aws
        vars:
          access_key: {{environment_access_key}}
          secret_key: {{environment_secret_key}}
          tag_name: concourse
```

## Behavior

This resource should always be used with the `put` action rather than a `get`.
This ensures the output always reflects the current state of the IaaS and allows management of multiple environments as shown below.

Depending on the context, the `put` step will perform one of three actions:

**Create:**
If the state file does not exist, `put` will create all the IaaS resources specified by `terraform.source`.
It then uploads the resulting state file using the configured `storage` driver.

**Update:**
If the state file already exists, `put` will update the IaaS resources to match the desired state specified in `terraform.source`.
It then uploads the updated state file.

**Destroy:**
If the `destroy` action is specified, `put` will destroy all IaaS resources specified in the state file.
It then deletes the state file using the configured `storage` driver.

#### Parameters

* `env_name`: *Optional.* The name of the environment to create or modify. Multiple environments can be managed with a single resource.

* `generate_random_name`: *Optional.* Generates a random `env_name` (e.g. "coffee-bee"). Useful for creating lock files.

* `env_name_file`: *Optional.* Reads the `env_name` from a specified file path. Useful for destroying environments from a lock file.

* `terraform`: *Optional.* The same Terraform configuration options described under `source.terraform` can also be specified under `put.params.terraform` with the following addition:

  * `terraform.var_file`: *Optional.* A file containing Terraform input variables.
  This file can be in YAML or JSON format.
  Terraform variables will be merged from the following locations in increasing order of precedence: `source.terraform.vars`, `put.params.terraform.vars`, and `put.params.terraform.var_file`. If a state file already exists, the outputs will be fed back in as input `vars` to subsequent `puts` with the lowest precedence.
  Finally, `env_name` is automatically passed as an input `var`.

* `action`: *Optional.* Used to indicate a destructive `put`. The only recognized value is `destroy`, create / update are the implicit defaults.

  * **Note:** You must also set `put.get_params.action` to `destroy` to ensure the task succeeds. This is a temporary workaround until Concourse adds support for `delete` as a first-class operation. See [this issue](https://github.com/concourse/concourse/issues/362) for more details.

#### Metadata file

Every `put` action creates `name` and `metadata` files as an output containing the `env_name` and [Terraform Outputs](https://www.terraform.io/intro/getting-started/outputs.html) in JSON format.

```yaml
jobs:
  - put: terraform
    params:
      env_name: e2e
      terraform:
        source: project-src/terraform
  - task: show-outputs
    config:
      platform: linux
      inputs:
        - name: terraform
      run:
        path: /bin/sh
        args:
          - -c
          - |
              echo "name: $(cat terraform/name)"
              echo "metadata: $(cat terraform/metadata)"
```

The preceding job would show a file similar to the following:

```
name: e2e
metadata: { "vpc_id": "vpc-123456", "vpc_tag_name": "concourse" }
```

#### Examples

See the [annotated pipeline](ci/pipeline.yml) for usage examples.

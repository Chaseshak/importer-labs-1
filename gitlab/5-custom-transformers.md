# Using custom transformers to customize Valet's behavior

In this lab you will build upon the `dry-run` command to override Valet's default behavior and customize the converted workflow using "custom transformers". Custom transformers can be used to:

1. Convert items that are not automatically converted.
2. Convert items that were automatically converted using different actions.
3. Convert environment variable values differently.
4. Convert references to runners to use a different runner name in Actions.

## Prerequisites

1. Followed the steps [here](./readme.md#configure-your-codespace) to set up your GitHub Codespaces environment and start your GitLab server.
2. Completed the [configure lab](./1-configure.md#configuring-credentials).
3. Completed the [dry-run lab](./4-dry-run.md).

## Perform a dry-run

You will be performing a `dry-run` command to inspect the workflow that is converted by default. Run the following command within the codespace terminal:

```bash
gh valet dry-run gitlab --output-dir tmp/dry-run --namespace valet --project terraform-example
```

The converted workflow that is generated by the above command can be seen below:

<details>
  <summary><em>Converted workflow 👇</em></summary>

```yaml
name: valet/custom-transformer
on:
  push:
  workflow_dispatch:
concurrency:
  group: "${{ github.ref }}"
  cancel-in-progress: true
jobs:
  plan:
    runs-on: ubuntu-latest
    timeout-minutes: 60
    env:
      PLAN: plan.cache
      PLAN_JSON: plan.json
    steps:
    - uses: actions/checkout@v2
      with:
        fetch-depth: 20
        lfs: true
    - run: terraform plan -out=$PLAN
    - run: terraform show --json $PLAN | convert_report > $PLAN_JSON
#     # 'artifacts.terraform' was not transformed because there is no suitable equivalent in GitHub Actions
```

</details>

_Note_: You can refer to the previous [lab](./4-dry-run.md) to learn about the fundamentals of the `dry-run` command.

## Custom transformers for an unknown step

The converted workflow above contains an `artifacts.terraform` step that was not automatically converted. Answer the following questions before writing a custom transformer:

1. What is the "identifier" of the step to customize?
    - __artifacts.terraform__

2. What is the desired Actions syntax to use instead?
    - After some research, you have determined that the following action will provide similar functionality:

      ```yaml
      - uses: actions/upload-artifact@v3
        with:
          path: VALUE_FROM_GITLAB
      ```

Now you can begin to write the custom transformer. Custom transformers use a DSL built on top of Ruby and should be defined in a file with the `.rb` file extension. You can create this file by running the following command in your codespace terminal:

```bash
touch transformers.rb && code transformers.rb
```

To build this custom transformer, you first need to inspect the `item` keyword to programmatically use the file path of the terraform json in the `actions/upload-artifact@v3` step.

To do this, you will print `item` to the console. You can achieve this by adding the following custom transformer to `transformers.rb`:

```ruby
transform "artifacts.terraform" do |item|
  puts "This is the item: #{item}"
end
```

The `transform` method can use any valid ruby syntax and should return a `Hash` that represents the YAML that should be generated for a given step. Valet will use this method to convert a step with the provided identifier and will use the `item` parameter for the original values configured in GitLab.

Now, we can perform a `dry-run` command with the `--custom-transformers` CLI option. The output of the `dry-run` command should look similar to this:

```console
$ gh valet dry-run gitlab --output-dir tmp --namespace valet --project terraform-example --custom-transformers transformers.rb
[2022-09-20 17:47:55] Logs: 'tmp/log/valet-20220920-174755.log'                 
This is the item: $PLAN_JSON                                                    
[2022-09-20 17:47:56] Output file(s):
[2022-09-20 17:47:56]   tmp/valet/terraform-example.yml
```

Now that you know the data structure of `item`, you can access the file path programmatically by editing the custom transformer to the following:

```ruby
transform "artifacts.terraform" do |item|
  {
    uses: "actions/upload-artifact@v3",
    with: {
      path: item
    }
  }
end
```

Now you can perform another `dry-run` command and use the `--custom-transformers` CLI option to provide this custom transformer. Run the following command within your codespace terminal:

```bash
gh valet dry-run gitlab --output-dir tmp/dry-run --namespace valet --project terraform-example --custom-transformers transformers.rb
```

The converted workflow that is generated by the above command will now use the custom logic for the `artifacts.terraform` step.

```diff
- #     # 'artifacts.terraform' was not transformed because there is no suitable equivalent in GitHub Actions
+ uses: actions/upload-artifact@v3
+ with:
+   path: "$PLAN_JSON"
```

## Custom transformers for environment variables

You can also use custom transformers to edit the values of environment variables in converted workflows. In this example, you will update the `PLAN_JSON` environment variable to be `custom_plan.json` instead of `plan.json`.

To do this, add the following code at the top of the `transformers.rb` file.

```ruby
env "PLAN_JSON", "custom_plan.json"
```

In this example, the first parameter to the `env` method is the environment variable name and the second is the updated value.

Now you can perform another `dry-run` command with the `--custom-transformers` CLI option.  When you open the converted workflow the `PLAN_JSON` environment variable will be set to `custom_plan.json`:

```diff
 env:
   PLAN: plan.cache
-  PLAN_JSON: plan.json
+  PLAN_JSON: custom_plan.json
```

## Custom transformers for runners

Finally, you can use custom transformers to dictate which runners the converted workflows should use. To do this, answer the following questions:

1. What is label of the runner in GitLab to update?
    - __:default__. This is a special keyword to define the default runner to use. You can optional target specific `tags` in a job.

2. What is the label of the runner in Actions to use instead?
    - __custom-runner__

With these questions answered, you can add the following code to the `transformers.rb` file:

```ruby
runner :default, "custom-runner"
```

In this example, the first parameter to the `runner` method is the GitLab label and the second is the Actions runner label.

Now you can perform another `dry-run` command with the `--custom-transformers` CLI option. When you open the converted workflow the `runs-on` statement will use the customized runner label:

```diff
runs-on:
-  - ubuntu-latest
+  - custom-runner
```

At this point, the file contents of `transformers.rb` should match this:

<details>
  <summary><em>Custom transformers 👇</em></summary>

```ruby
  runner :default, "custom-runner"

  env "PLAN_JSON", "custom_plan.json"

  transform "artifacts.terraform" do |item|
    {
      uses: "actions/upload-artifact@v2",
      with: {
        path: item
      }
    }
  end
```

</details>

That's it! Congratulations, you have overridden Valet's default behavior by customizing the conversion of:

- Unknown steps
- Environment variables

## Next lab

[Perform a production migration of a GitLab pipeline](6-migrate.md)
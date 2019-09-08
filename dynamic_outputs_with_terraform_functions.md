# Building Dynamic Outputs with Terraform Loops and Functions

We know we can define a Terraform module that produces output for *another* module to use as input. But how can we build dynamic output from a module that creates a *set* resources, and format that output _just right_ to act as input elsewhere? It's possible with the help of Terraform `for` and `zipmap` functions. Keep reading to learn more!

## The Static Approach

While building out a set of Terraform modules to define some AWS resources at The Flatiron School, we ran into an interesting challenge. We needed to be able to do the following:

* Define a set of AWS Secrets Manager resources.
* Create an AWS ECS Task Definition resource with a container definition using those secrets.

A static approach to solving this problem would look something like this.


**Step 1: Create the AWS Secrets Manager Resources**

```
resource "aws_secretsmanager_secret" "my-app-PGUSER" {
  name                    = "my-app-PGUSER"
  recovery_window_in_days = "30"
}

resource "aws_secretsmanager_secret" "my-app-PGPASSWORD" {
  name                    = "my-app-PGPASSWORD"
  recovery_window_in_days = "30"
}
```

**Step 2: Create the Container Definition**

```
resource "aws_ecs_task_definition" "app-task-definition" {
  ...

  container_definitions = <<DEFINITION
[
  {
    ...
    "secrets": [{
        "name": "PGUSER",
        "valueFrom": "${aws_secretsmanager_secret.my-app-PGUSER.arn}"
      }, {
        "name": "PGPASSWORD",
        "valueFrom": "${aws_secretsmanager_secret.my-app-PGPASSWORD.arn}"
      }
]
DEFINITION
}
```

While this approach works, is has a couple of obvious drawbacks.

**Static, instead of dynamic**

In this example, we established just two AWS Secrets Manager secrets, `my-app-PGUSER` and `my-app-PGPASSWORD` and passed the values of those secrets into our container definitions as environment variables. What happens when when our application far more secrets? Having to manually write out the resource definitions for each secret and update the container definition accordingly makes for a lot of repetitious and verbose Terraform code. It's hard to read and annoying to write. We hate it and with there was a better way!

**Concrete, instead of abstract**

What happens when we next need to create a set of resources and a container definition––maybe for a brand new application? We would have to write all this code all over again! We also hate this and wish there was a better way!

By creating a set of reusable modules, we make it easy to add more secrets as env vars in the future *and* we make it easy to create brand new task definitions with the appropriate secrets dynamically.

## Building Dynamic Resources with `for_each`

First we'll solve our repetition problem. Instead of manually writing a new AWS Secrets Manager resource definition for every secret we want to create, we'll define a reusable module that will take in an input of a list of secrets to establish. Our module will use Terraform's [`for_each`](https://www.terraform.io/docs/configuration/expressions.html) expression to iterate over that list and create a resource for each one.

We want to define a module that is called with two inputs:

* The list of application secrets, which we'll pass in as the `application_secrets` input.
* The name of the application.

This way, we can create a set of AWS Secrets Manager resources with the naming convention: `my-app-THE_SECRET`.

We'll define our module in a subdirectory, `terraform/prod-us-east-1/secrets-modules/`, and we'll start with the `main.tf` definition.

```
# terraform/prod-us-east-1/secrets-module/main.tf

resource "aws_secretsmanager_secret" "application-secret" {
  for_each                = toset(var.application_secrets)
  name                    = "${var.name}-${each.value}"
  recovery_window_in_days = "30"
}
```

Here we're using Terraform's `for_each` expression in our resource definition. This has the effect of iterating over the list of secrets we pass into our module call, made available as `var.application_secrets`, and creating a resource for each one.

Note that we use the [`toset`](https://www.terraform.io/docs/configuration/functions/toset.html) function on `var.application_secrets`. This is because `for_each` only operates on the `set` datatype, but our application secrets get passed into the module call as a list.

Also note that our resource definition won't work unless we define the variables that it wants to use:

```
# terraform/prod-us-east-1/secrets-module/variables.tf

variable "application_secrets" {
  description = "list of application secret names"
  type        = "list"
  default     = []
}

variable "name" {
  description = "the name of your application to prepend to secret names in AWS Secrets Manager"
  type        = "string"
  default     = "application name"
}
```

Now let's take a look at how we can call this module to define a specific set of application secrets.

```
# terraform/prod-us-east-1/my-app-secrets.tf

variable "my-app-application-secrets" {
  type = list(string)
  default = [
    "PGPASSWORD",
    "PGUSER"
  ]
}

module "my-app-application-secrets" {
  source              = "./secrets-module"
  name                = "my-app"
  application_secrets = var.my-app-application-secrets
}
```

Having defined a list of application secrets as the variable `"my-app-application-secrets"`, we create a module using our `secrets-module` as a source. We call this module with two inputs: the `name` and `my-app-application-secrets` list.

Now, if we run `terraform plan`, we'll see the following resources will be created:

```
 # module.my-app-application-secrets.aws_secretsmanager_secret.application-secret["PGPASSWORD"] will be created
 + resource "aws_secretsmanager_secret" "application-secret" {
     + arn                     = (known after apply)
     + id                      = (known after apply)
     + name                    = "my-app-PGPASSWORD"
     + name_prefix             = (known after apply)
     + recovery_window_in_days = 30
     + rotation_enabled        = (known after apply)
   }

 # module.my-app-application-secrets.aws_secretsmanager_secret.application-secret["PGUSER"] will be created
 + resource "aws_secretsmanager_secret" "application-secret" {
     + arn                     = (known after apply)
     + id                      = (known after apply)
     + name                    = "my-app-PGUSER"
     + name_prefix             = (known after apply)
     + recovery_window_in_days = 30
     + rotation_enabled        = (known after apply)
   }
```

This approach wins us two things:

* Now, when we want to create a new AWS Secrets Manager resource, all we need to do is add to the list defined in the `my-app-appication-secrets` variable. Much easier than writing out a whole new resource definition!
* At such time as we want to build a set of AWS Secrets Manager resources for a *new* application, we can call on our `secrets-module` module with ease. All we need to do is define a new module `my-other-app-application-secrets`, with its own variable that lists the secrets to create. Easy!

While we've solved our repetition problem when it comes to defining secrets, we *still* need to manually grab the ARNs of each secret resource in order to construct the input for our container definition.

Let's fix this by teaching our secrets module to generate output that we can pass into our container definition.

## Building Dynamic Resources with `for`

We need our secrets module to output the *exact* input our container definition requires. Before we discuss how we'll do this, let me introduce the container definition module that our setup utilizes. This post assumes that we have a container definition module already set up that expects the following input:

```
module "my-app-container-definition" {
  source          = "./container-modules"
  ...
  secrets = [
    {
      name = "PGUSER",
      valueFrom = ${aws_secretsmanager_secret.my-app-PGUSER.arn}
    },
    {
      name = "PGPASSWORD",
      valueFrom = ${aws_secretsmanager_secret.my-app-PGPASSWORD.arn}
    }
  ]
}
```

We won't focus on how this particular module was defined here. If you want to take a deeper dive into the container definition module, check out the source code that inspired us [here](https://github.com/cloudposse/terraform-aws-ecs-container-definition).

So, we need our secrets module to dynamically generate output that looks *exactly* like this:

```
[
  {
    name = "PGUSER",
    valueFrom = <arn of this secret resource>,
  {
    name = "PGPASSWORD",
    valueFrom = <arn of this secret resource>,
]
```

And so on for each secret that we input into the secrets module via the `application_secrets` input. Let's do it!

## Generating Dynamic Output with `for` and `zipmap`

We'll define an output for our secrets module:

```
# terraform/prod-us-east-1/secrets-module/output.tf

output "application-secret-mappings" {
  value = # coming soon!
}
```

Our output needs to iterate over all of the secrets we passed into the module via the `application_secrets` input, map each one to its corresponding ARN and construct a list of payloads:

```
{
  name = < secret name>,
  valueFrom = < secret ARN>
}
```

Right now, secret names and the ARNs exist as two separate lists. Our secrets names are here:

```
var.application_secrets
```

And our ARN list can be looked up within the module like this:

```
values(aws_secretsmanager_secret.application-secret)[*]["arn"]
```

We'll use Terraform's [`zipmap`](https://www.terraform.io/docs/configuration/functions/zipmap.html) function to build a map where the keys are the secret names and the values are the corresponding ARNs:

```
zipmap(
  sort(var.application_secrets),
  sort(values(aws_secretsmanager_secret.application-secret)[*]["arn"])
)
```

*Note that we use the `sort` function to ensure that the right secret is mapped to the right ARN*.

This will produce the following map:

```
{
  "PGUSER" = aws_secretsmanager_secret.application-secret["PGUSER"].arn,
  "PGPASSWORD" = aws_secretsmanager_secret.application-secret["PGPASSWORD"].arn
}
```

Now, we can iterate over this map in our output definition, with the help of the [`for`](https://www.terraform.io/docs/configuration/expressions.html#for-expressions) expression in order to collect a list of objects that are formatted correctly for our container definition.

```
# where `secrets_map` is the result of our zipmap call
[ for secret, arn in secrets_map : map("name", secret, "valueFrom", arn) ]
```

Putting it all together in our output definition:

```
output "application-secret-mappings" {
  value = [
    for secret, arn in zipmap(
      sort(var.application_secrets),
      sort(values(aws_secretsmanager_secret.application-secret)[*]["arn"])) :
      map("name", secret, "valueFrom", arn)
  ]
}
```

This will produce the following output:

```
[
  {
    name = "PGUSER",
    valueFrom = <arn of this secret resource>,
  {
    name = "PGPASSWORD",
    valueFrom = <arn of this secret resource>,
]
```

Exactly what we need for our container definition!

One last step before we can call on this output in our container definition input. We need to tell the `my-app-secrets` module to produce its own output, using the `"application-secret-mappings"` output as a source.

Back in our `my-app-secrets.tf` file:

```
# terraform/prod-us-east/my-app-secrets.tf

output "my-app-application-secrets" {
  description = "List of secrets mapped to ARNs"
  value       = module.my-app-application-secrets.application-secret-mappings
}
```

Now we're ready to use on our output when we call on a container definition module!

## Using Dynamic Outputs in the Container Definition

Remember that we have a container definition module that looks something like this:

```
module "my-app-container-definition" {
  source          = "./container-modules"
  ...
  secrets = [
    {
      name = "PGUSER",
      valueFrom = ${aws_secretsmanager_secret.my-app-PGUSER.arn}
    },
    {
      name = "PGPASSWORD",
      valueFrom = ${aws_secretsmanager_secret.my-app-PGPASSWORD.arn}
    },
  ]
}
```

And we want to replace the static list of secrets with our dynamic secrets module output. We can do so like this:

```
module "my-app-container-definition" {
  source  = "./container-modules"
  secrets = module.my-app-application-secrets.application-secret-mappings
  ...
}
```

And that's it! This approach wins us a few things:

* When we need to add new secrets to a given app's container, we _don't_ need to manually create the AWS Secrets Manager resource definition, _and_ capture the ARN _and_ update the container definition. We _just_ need to add the new secret to the list of application secrets in the `my-app-application-secrets` variable and apply our changes. This will create the resource and update the output, thereby updating the container definition to include that new secret and ARN for free!
* We've made building out new applications and their resources into a relatively quick and painless process. We can call on our secrets module again and again to dynamically create the secrets resources for any new apps we want to spin up, and use the output from those module calls in each app's corresponding container definitions with ease.

## Conclusion

Terraform's reusable modules and helpful expressions and functions allowed us to write DRY "infrastructure as code". With the help of the `for_each` expression, we were able to define a module that dynamically creates AWS Secrets Manger resources with just a few lines of code. With the help of the `for` expression and `zipmap` function, we were able to take that module one step further. We defined dynamic module output that is formatted _just right_ for our corresponding container definition input. As a result, we've given ourselves a sane and replicable approach to generating complex application infrastructure.

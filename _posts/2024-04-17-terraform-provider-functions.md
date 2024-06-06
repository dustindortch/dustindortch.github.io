---
layout: post
title: Terraform Provider Functions
permalink: /2024/04/17/terraform-provider-functions
date: 2024-04-17 17:22:00 -0500
categories:
  - DevOps
  - Terraform
  - Go
tags:
  - terraform provider
  - terraform provider funtions
  - terraform-provider-math
{% assign terraform_plan = site.static_files | where: "image", true | where: "extname", "png" | where: "basename", "2024-04-17-terraform-provider-functions-terraform-plan" %}
{% assign terraform_apply = site.static_files | where: "image", true | where: "extname", "png" | where: "basename", "2024-04-17-terraform-provider-functions-terraform-apply" %}
---

## Terraform Provider Functions

Have you ever wished for a function to exist in Terraform but it simply wasn’t there? Fret no more! Terraform 1.8 was released last week and includes the general availability of [Provider Functions](https://www.hashicorp.com/blog/terraform-1-8-improves-extensibility-with-provider-defined-functions). This feature allows you to write your own functions in Go and use them in your Terraform configuration. This is a game changer for Terraform and I’m excited to see what the community comes up with.

I decided to try this out for myself using inspiration from the [fibonacci  sequence and memoization that I wrote about with respect to Python](https://dustindortch.com/?page_id=1406). I cloned the [hashicorp/terraform-provider-scaffolding-framework repository](https://github.com/hashicorp/terraform-provider-scaffolding-framework) and got to work.

My initial intent was just to write a function called “fib” that would generate the fibonacci sequence number given an input. It took me down a rabbit hole learning even more about the type system implemented in HCL. The tutorial provided by HashiCorp walks through creating the “hashicups” provider. I loosely used it as a starting point, but I quickly lost interest because they walk you through, line by line, modifying the “template”. No thanks. I wrote a Bash script called [scaffold-terraform-provider.sh](https://gist.github.com/dustindortch/0df94248b232c9b16f757313c2690225) to handle that.

```bash
./scaffold-terraform-provider.sh math
```

I named my provider “math” to keep it generic and include additional functions within that space to use it like a library/package.

### Functions

Creating provider functions should follow the format of making a new file in “internal/provider” and naming it for the function (e.g. “fib_function.go”). I copied the example_function.go file and then proceded to modify it by searching for “ExampleFunction” and replacing with “FibFunction”.

A function is registered to the provider with a creation of a new function called “NewFibFunction” that returns the “function.Function” type:

```go
func NewFibFunction() function.Function {
  return &FibFunction{}
}
```

Then, the Metadata method needs to be called that returns the function metadata:

```go
func (f *FibFunction) Metadata(ctx context.Context, req function.MetadataRequest, resp *function.MetadataResponse) {
  resp.Name = "fib"
}
```

The Definition method is used to create your input and return types:

```go
func (f *FibFunction) Definition(ctx context.Context, req function.DefinitionRequest, resp *function.DefinitionResponse) {
  resp.Definition = function.Definition{
    Summary:             "Fibonacci sequence",
    MarkdownDescription: "Accepts a number and returns the result of the fibonacci sequence at that index.",
    Parameters: []function.Parameter{
      function.NumberParameter{
        Name: "number",
      },
    },
    Return: function.NumberReturn{},
  }
}
```

The “fib” function is rather simple because it takes a single input and returns a value. Everything uses the different types that are part of the provider framework.

To execute the function, you implement the Run method:

```go
func (f *FibFunction) Run(ctx context.Context, req function.RunRequest, resp *function.RunResponse) {
  var number *big.Float
  var result *big.Float

  // Read Terraform argument data into the variables
  resp.Error = function.ConcatFuncErrors(resp.Error, req.Arguments.Get(ctx, &number))

  fibInt, _ := number.Uint64()
  result = big.NewFloat(0).SetUint64(fib.Fib(fibInt))

  resp.Error = function.ConcatFuncErrors(resp.Error, resp.Result.Set(ctx, result))
}
```

The tricky part is that everything implements the data types from the go-cty package. So, Terraform “number” data type is implemented as a big.Float, from the “math/big” standard library package.

In order to simplify everything, I put my actual implementation of the fibonacci sequence in a separate package called “fib” and imported it into the function file (the package is in “internal/fib”). This allowed me to test the function in isolation from the provider.

```go
package fib

var cache = make(map[uint64]uint64)

func Fib(n uint64) uint64 {
  if n < 2 {
    return n
  }

  if _, ok := cache[n]; !ok {
    cache[n] = Fib(n-1) + Fib(n-2)
  }
  return cache[n]
}
```

The final requirement to register the function is to update the “internal/provider/provider.go” file:

```go
func (p *MathProvider) Functions(ctx context.Context) []func() function.Function {
  return []func() function.Function{
    NewFibFunction,
  }
}
```

To test it, I had to create a `~/.terraformrc` file to overwrite the provider path and source it locally:

```hcl
provider_installation {

  dev_overrides {
      "hashicorp.com/edu/math" = "/Users/dustindortch/go/bin"
  }

  # For all other providers, install them directly from their origin provider
  # registries as normal. If you omit this, Terraform will _only_ use
  # the dev_overrides block, and so no other providers will be available.
  direct {}
}
```

Now, I am able to build:

```bash
go install .
```

After this, I created a simple Terraform configuration to test the function:

```hcl
terraform {
  required_providers {
    math = {
      source = "hashicorp.com/edu/math"
    }
  }
}

output "sequence" {
  value = provider::math::fib(12)
}
```

For the moment of truth:

```bash
terraform plan
```

And the results:

![terraform plan]({{ terraform_plan.path}})

### Resources

To contrast the behavior between functions and resources, I also implemented a resource called “math_fib” that uses the same “fib” pacakge. The main difference is that the resource is written to state and won’t update unless there is a change that requires it to be replaced. A function does not get [directly] written to state (only if it is the input to some other resource that is written to state).

I created a new resource file “internal/provider” and naming it for the resource (e.g. “fib_resource.go”).

I am not going to go into as many details here, but following are some highlights.

A data structure must be created that includes the inputs and outputs of the resource:

```go
type FibResourceModel struct {
  ID     types.String `tfsdk:"id"`
  Number types.Int64  `tfsdk:"number"`
  Result types.Int64  `tfsdk:"result"`
}
```

The Metadata method to name the resource:

```go
func (r *FibResource) Metadata(ctx context.Context, req resource.MetadataRequest, resp *resource.MetadataResponse) {
  resp.TypeName = req.ProviderTypeName + "_fib"
}
```

A detailed schema is created with the Schema method:

```go
func (r *FibResource) Schema(ctx context.Context, req resource.SchemaRequest, resp *resource.SchemaResponse) {
  resp.Schema = schema.Schema{
    // This description is used by the documentation generator and the language server.
    MarkdownDescription: "The resource `math_fib` generates the fibonacci number from a given number.",

    Attributes: map[string]schema.Attribute{
      "number": schema.Int64Attribute{
        MarkdownDescription: "The number to calculate the fibonacci sequence",
        Required:            true,
        PlanModifiers: []planmodifier.Int64{
          int64planmodifier.RequiresReplace(),
        },
      },
      "result": schema.Int64Attribute{
        MarkdownDescription: "The result of the fibonacci number calculation.",
        Computed:            true,
        PlanModifiers: []planmodifier.Int64{
          int64planmodifier.UseStateForUnknown(),
        },
      },
      "id": schema.StringAttribute{
        Computed:            true,
        MarkdownDescription: "The string representation of the fibonacci number result.",
        PlanModifiers: []planmodifier.String{
          stringplanmodifier.UseStateForUnknown(),
        },
      },
    },
  }
}
```

The Create method does the hard work by getting the plan data, performing any necessary operations, and then writing the result to the state:

```go
func (r *FibResource) Create(ctx context.Context, req resource.CreateRequest, resp *resource.CreateResponse) {
  var data FibResourceModel

  // Read Terraform plan data into the model
  resp.Diagnostics.Append(req.Plan.Get(ctx, &data)...)

  if resp.Diagnostics.HasError() {
    return
  }

  result := int64(fib.Fib(uint64(data.Number.ValueInt64())))

  // For the purposes of this example code, hardcoding a response value to
  // save into the Terraform state.
  data.ID = types.StringValue(strconv.FormatInt(result, 10))
  data.Result = types.Int64Value(result)

  // Write logs using the tflog package
  // Documentation: https://terraform.io/plugin/log
  tflog.Info(ctx, "created a resource")

  // Save data into Terraform state
  resp.Diagnostics.Append(resp.State.Set(ctx, data)...)
}
```

The other operations are fairly generic. The one thing that I did do, otherwise, is remove the call to the Configure method because my provide isn’t communicating with any API endpoint, similar to the Random provider.

Back in the “internal/provider/provider.go” file, I added the resource to the list of resources:

```go
func (p *MathProvider) Resources(ctx context.Context) []func() resource.Resource {
  return []func() resource.Resource{
    NewFibResource,
  }
}
```

Aside from that, I generated the documentation:

```bash
go generate ./...
```

Then, it was simply (an understatement to be sure) a matter of creating my repository, committing and pushing the code, generating a GPG signing key and importing the requisite details as GitHub Secrets, and then creating a release.

The last thing I did was added my GPG public key to my Terraform Registry account and then I published.

I made a new directory to test the code:

```hcl
terraform {
  required_providers {
    math = {
      source = "dustindortch/math"
    }
  }
}

resource "math_fib" "sequence" {
  number = 9
}

output "result" {
  value = math_fib.sequence.result
}

output "sequence" {
  value = provider::math::fib(12)
}
```

The run it:

```bash
terraform init
terraform apply -auto-approve
```

![terraform apply]({{ terraform_apply.path}})

### Conclusion

Functions were not a huge hurdle to overcome. If you have any ideas about functions that you would like to see, please let me know. I am looking for a few ideas to add to my “math” provider.

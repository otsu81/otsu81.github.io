---
layout: post
title: AWS SAM Parameter Value tags for universal tagging
---

Love them or hate the, it's generally a good idea to apply [tags](https://docs.aws.amazon.com/whitepapers/latest/tagging-best-practices/what-are-tags.html) to your AWS resources. We use it primarily for cost allocation tags, but are slowly discovering more usecases. 

As your stack grows it can become a chore to remember to apply tags, or even to keep them updated. It can also be a bit annoying to have to keep track of which resources use the `Value: Key` syntax and which ones use the `- Key: key \nValue: value`. I was just shown a neat trick by a colleague (shout-out Ilir!) which really highlighted the need to actually read the docs from time to time - SAM parameter file tags, neatly tucked away under [Parameter value rules](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/serverless-sam-cli-config.html#serverless-sam-cli-config-rules). Unbeknowst to me, this is a configuration file wrapper for the AWS CLI `cloudformation create-stack --tags` [command](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/cloudformation/create-stack.html) which applies Stack-level tags to all supported resources in your stack.

Previously to achieve some kind of DRY principle for tagging people would resort to using the `Globals` section. **Don't** do this:

```yaml
Globals:                # NOT
  Function:             # THE
    Tags:               # BEST
      SomeKey: SomeVal  # WAY
```
... but it had the obvious downside of only being applied to the resource `AWS::Serverless::Function`.

By using the built-in transformation capability of SAM by simply adding `tags` to your configuration file you can ensure **all** resources that support tagging will receive it! It's a super neat way to handle cost allocation tags, versioning, easily searchable strings in the Tag Editor - the possibilities are endless!

```toml
# Example samconfig.toml
version = 0.1
[us-east-1]
[us-east-1.deploy]
[us-east-1.deploy.parameters]
stack_name = "api-services"
resolve_s3 = true
s3_prefix = "api-services"
region = "us-east-1"
confirm_changeset = false
capabilities = "CAPABILITY_IAM"
tags = "Product=\"apiServices\" Version=\"0.39\"" # <<-- THIS IS THE MAGIC BIT
```

Now, without specifying a single tag in the actual template, SAM will ensure that all resources will receive the tags. 

![](/images/sam_parameter_tags.png)

Take care of your colleagues and stay open, you never know when they will show you've missed.

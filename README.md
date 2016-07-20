# Codex

Codex brings together the years of experience and lessons learned by
Stark & Wayne engineers after designing, deploying and managing their client's
BOSH and Cloud Foundry distributed architectures.  These best practices are
gathered together here to further mature and develop these techniques.

## Software Requirements

You will need the following software on your local machine to run these
deployments:

- **[Spruce][spruce]**
- **[Genesis][genesis]**
- **[Safe][safe]**
- **[Terraform][terraform]**

On a Mac, you can install spruce, genesis, safe and terraform via Homebrew:

```
$ brew tap starkandwayne/cf
$ brew install spruce genesis safe terraform
```

Terraform comes with its own [installation instructions][tf-inst].

## Get Started

To get started with a best practices Codex deployment, you'll want to choose
your IaaS provider. We have guides for the following:

- [AWS](aws.md)
- [Azure](azure.md)
- [OpenStack](openstack.md)

## Offline Reading

The product of this repo is also available for online reading and PDF download
at [gitbook.com][gitbook].

Read more about how to use and abuse gitbook tools, plugins, etc. in our [gitbook
guide](gitbook.md).

[genesis]:   https://github.com/starkandwayne/genesis
[gitbook]:   https://www.gitbook.com/book/starkandwayne/codex/details
[spruce]:    https://github.com/geofffranks/spruce
[safe]:      https://github.com/starkandwayne/safe
[terraform]: https://www.terraform.io
[tf-inst]:   https://www.terraform.io/intro/getting-started/install.html

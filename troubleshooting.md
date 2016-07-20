## Common Errors and Solutions to them!
Here we will display all common errors, the common paths that you ended up in that position, and the ways to get around them. If you can't find the solution in the main docs, then the answer will probably be here....ONWARDS!

### Bastion / Jumpbox
* When trying to `make all` to deploy the Bastion host.
    Terraform will connect to AWS, using your Access Key and Secret
    Key, and spin up all the things it needs.  When it finishes, you
    should be left with a bunch of subnets, configured network ACLs,
    security groups, routing tables, a NAT instance (for public
    internet connectivity) and a Bastion host.

    If the `deploy` step fails with errors like:

    ```
    * aws_subnet.prod-cf-edge-1: Error creating subnet: InvalidParameterValue: Value (us-east-1a) for parameter availabilityZone is invalid. Subnets can currently only be created in the following availability zones: us-east-1c, us-east-1e, us-east-1b, us-east-1d. status code: 400, request id: 8ddbe059-0818-48c2-a936-b551cd76cdeb
    * aws_subnet.prod-infra-1: Error creating subnet: InvalidParameterValue: Value (us-east-1a) for parameter availabilityZone is invalid. Subnets can currently only be created in the following availability zones: us-east-1c, us-east-1b, us-east-1d, us-east-1e. status code: 400, request id: 876f72b2-6bda-4499-98c3-502d213635eb
    * aws_subnet.dev-infra-3: Error creating subnet: InvalidParameterValue: Value (us-east-1a) for parameter availabilityZone is invalid. Subnets can currently only be created in the following availability zones: us-east-1c, us-east-1b, us-east-1d, us-east-1e. status code: 400, request id: 66fafa81-7718-46eb-a606-e4b98e3267b9
    ```

    you should run `make destroy` to clean up, then add a line like `aws_az1 = "d"` to replace the restricted zone.
    
* No Vault or Genesis - Chances are you did not run the jumpbox script, refer to [the Bastion/Jumpbox section][1] and make sure that you remain logged in as your Jumpbox user during the rest of the deployment process. Nothing is done from the default Bastion user.

### ProtoBosh & Proto-Vault
* Error Deploying Proto-BOSH with Shield Agent Job - If you see the error below, then you are running the scripts and everything from the Bastion user, you MUST use the Jumpbox scripts/users/Vault in order for it to be nice and not throw errors at you.
```
    Command 'deploy' failed:
      Deploying:
        Building state for instance 'bosh/0':
          Rendering job templates for instance 'bosh/0':
            Rendering templates for job 'shield-agent/38e11abc09d09a2af3572c070cc9813ab640e8dd':
              Rendering template src: config/target.json.erb, dst: config/target.json:
                Rendering template src: /home/ubuntu/.bosh_init/installations/bea9b166-9d73-41bc-43b1-37f49aa20d83/tmp/bosh-init-release368081404/extracted_jobs/shield-agent/templates/config/target.json.erb, dst: /home/ubuntu/.bosh_init/installations/bea9b166-9d73-41bc-43b1-37f49aa20d83/tmp/rendered-jobs598042080/config/target.json:
                  Running ruby to render templates:
                    Running command: 'ruby /home/ubuntu/.bosh_init/installations/bea9b166-9d73-41bc-43b1-37f49aa20d83/tmp/erb-renderer386521023/erb-render.rb /home/ubuntu/.bosh_init/installations/bea9b166-9d73-41bc-43b1-37f49aa20d83/tmp/erb-renderer386521023/erb-context.json /home/ubuntu/.bosh_init/installations/bea9b166-9d73-41bc-43b1-37f49aa20d83/tmp/bosh-init-release368081404/extracted_jobs/shield-agent/templates/config/target.json.erb /home/ubuntu/.bosh_init/installations/bea9b166-9d73-41bc-43b1-37f49aa20d83/tmp/rendered-jobs598042080/config/target.json', stdout: '', stderr: '/home/ubuntu/.bosh_init/installations/bea9b166-9d73-41bc-43b1-37f49aa20d83/tmp/erb-renderer386521023/erb-render.rb:189:in `rescue in render': Error filling in template '/home/ubuntu/.bosh_init/installations/bea9b166-9d73-41bc-43b1-37f49aa20d83/tmp/bosh-init-release368081404/extracted_jobs/shield-agent/templates/config/target.json.erb' for shield-agent/0 (line 2: #<TypeError: nil is not a symbol nor a string>) (RuntimeError)
        from /home/ubuntu/.bosh_init/installations/bea9b166-9d73-41bc-43b1-37f49aa20d83/tmp/erb-renderer386521023/erb-render.rb:175:in `render'
        from /home/ubuntu/.bosh_init/installations/bea9b166-9d73-41bc-43b1-37f49aa20d83/tmp/erb-renderer386521023/erb-render.rb:200:in `<main>'
    ':
                      exit status 1
    Makefile:25: recipe for target 'deploy' failed
```

### Proto Deployments
* Vault -
* Concourse -
* SHIELD -
* Bolo -

### Prod Bosh Deployments
* CF Deployment

[1]: https://github.com/starkandwayne/codex/blob/master/aws.md#setting-up-the-bastion-host

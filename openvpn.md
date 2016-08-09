# How to Deploy OpenVPN and Setup Root Certificates with Vault

## Enabling Remote Access to Internal Services in Client Environments

In order to access the user interfaces for Concourse, SHIELD, Bolo,
and other internal-only services deployed for clients, your web browser
will need some sort of connection into the private cloud network. Some clients
may provide a VPN connection into their infrastructure/VPC, but some might not.

For the clients that provide VPN connections inside the infrastructure, use that.

If however we need to provide the client with a way in, we have the following options:

1. Use ssh-tunneling. This unfortunately has some downsides:
   a. The correct syntax for enabling the tunnels can be difficult to remember,
      especially if you have multiple SSH hosts to jump through.
   b. You might have forgotten to enable the tunnel when you originally SSH'd in,
      but half-way through your investigations, you realize you need the web UI for
      some task, and now have to start over.
   c. Managing what services are on what ports, and preventing conflicts between you
      and other users of the jumpboxes becomes very difficult.
2. Expose the internal-only services publicly. This also has downsides:
   a. Your internal-only services are now one misconfiguration away from being public.
   b. ACLs to block traffic unless originating from certain areas can be circumvented,
      allow more people to access services than you might want, and cause issues for
      remote workers.
3. Provide a VPN into the infrastructure. Hey, this one has no real downsides that I'm
   documenting, so we should chose this option!


## Enter the OpenVPN BOSH Release

OpenVPN is a relatively easy to manage VPN solution that will work on Mac, Windows, Linux,
and even mobile devices. We use it in TCP mode, on port 443, so it should be allowed through
just about any firewall that's restricting outbound connections.

To get started with it, you will need two Availability Zones in your infrastructure,
a public-facing load balancing solution, and a PKI system. For the PKI system, we recommend
Vault's PKI backend for generating the CA, server key/cert, client keys/certs, and maintaining
the Certificate Revocation List (CRL). Once the data is generated, you must store it in Vault
manually (until we add functionality into `safe` to support this directly).

Lets walk through setting up OpenVPN in AWS, using the same infrastructure built in our [Deploying
on AWS walkthrough](aws.md). Run through that plan through at least the deploying Vault stage,
and come back here to finish up the rest.

Back? Awesome. Let's create a new `openvpn-deployment` using our genesis template:

```
$ cd ~/ops
$ safe target ops
Now targeting ops at https://10.4.1.16:8200
$ bosh target proto
Target set to `aws-proto-bosh'
$ genesis new deployment --template openvpn
cloning from template https://github.com/starkandwayne/openvpn-deployment
Cloning into '/home/gfranks/ops/openvpn-deployments'...
remote: Counting objects: 47, done.
remote: Compressing objects: 100% (27/27), done.
Unpacking objects: 100% (47/47), done.
remote: Total 47 (delta 6), reused 47 (delta 6), pack-reused 0
Checking connectivity... done.
Embedding genesis script into repository
Treating 'development' version as up-to-date with 0.0.14
genesis 1.5.2 (61864a21370c)
[master 734c0d0] Initial clone of templated openvpn deployment
 3 files changed, 3686 insertions(+), 33 deletions(-)
  rewrite README.md (95%)
   create mode 100755 bin/genesis
    create mode 100644 global/README
```

And now we'll create a new AWS based site:

```
cd openvpn-deployments
$ genesis new site --template aws aws
Treating 'development' version as up-to-date with 0.0.14
Created site aws (from template aws):
/home/gfranks/ops/openvpn-deployments/aws
├── README
└── site
    ├── disk-pools.yml
    ├── jobs.yml
    ├── networks.yml
    ├── properties.yml
    ├── releases
    ├── resource-pools.yml
    ├── stemcell
    │   ├── name
    │   └── version
    └── update.yml

2 directories, 10 files

```

And lastly, the OpenVPN environment for the global infrastructure:

```
$ genesis new env aws proto
Treating 'development' version as up-to-date with 0.0.14
Created environment aws/proto:
/home/gfranks/ops/openvpn-deployments/aws/proto
├── cloudfoundry.yml
├── credentials.yml
├── director.yml
├── Makefile
├── monitoring.yml
├── name.yml
├── networking.yml
├── properties.yml
├── README
└── scaling.yml

0 directories, 10 files

```

Now let's go into that environment and try to make the manifest:

```
$ cd aws/proto
$ make deploy
Treating 'development' version as up-to-date with 0.0.14
  checking https://genesis.starkandwayne.com for details on latest stemcell bosh-aws-xen-hvm-ubuntu-trusty-go_agent
  checking https://genesis.starkandwayne.com for details on latest release openvpn
  checking https://genesis.starkandwayne.com for details on release toolbelt/3.2.10
8 error(s) detected:
 - $.meta.azs.z1: What availability zone should your openvpn_z1 VMs be in?
 - $.meta.azs.z2: What availability zone should your openvpn_z2 VMs be in?
 - $.meta.certs.ca: What is the CA cert used to sign the server/client certs?
 - $.meta.certs.crl: Provide the current CRL pem listing what certs have been revoked by the CA
 - $.meta.certs.server: Provide the cert to be used by the OpenVPN server
 - $.meta.certs.server_key: Provide the private key used by the OpenVPN server for its certificate
 - $.meta.client_routes: What networks should be advertised to clients over the VPN? (format NET_ADDR MASK)
 - $.networks.openvpn_z1.subnets: Specify your openvpn subnet
 - $.networks.openvpn_z2.subnets: Specify your openvpn subnet


Failed to merge templates; bailing...
Makefile:25: recipe for target 'deploy' failed
make: *** [deploy] Error 3
```

### Vault Setup

Ok, time to start filling in some data. We know we need certs and keys for OpenVPN, and we'll
be using Vault to store them, so we can fill in the Vault data now (and worry about generating
the keys in a couple minutes). Also, lets solve the client routes. We will want to push a route
to VPN clients to allow them to attempt to access anything in our VPC (10.4.0.0/16).

```
$ cat properties.yml
---
meta:
  certs:
    ca: (( vault meta.vault_prefix "/ca:cert" ))
    crl: (( vault meta.vault_prefix "/crl:pem" ))
    dh: (( vault meta.vault_prefix "/dh:pem" ))
    server: (( vault meta.vault_prefix "/server:cert" ))
    server_key: (( vault meta.vault_prefix "/server:key" ))
  client_routes:
      - 10.4.0.0 255.255.0.0
```

**NOTE** we need to specify the routes as `NET_ADDR NETMASK`, not via CIDR notation, as OpenVPN
does not work when CIDR is specified.

### Network Configuration

Now we can work on the networking configuration for OpenVPN. We need to place OpenVPN in two AZs,
and stick an ELB in front of them. Since the ELB will be public, we have the added requirement
that the subnets used by the VMs of OpenVPN will be publicly accessible.

Essentially this boils down to needing a routing table in AWS that is hooked up to an Internet
Gateway. If you used the Codex terraform configs to create all the subnets in your VPC, this
has been taken care of you, and you only need to create your ELB, and consult the [Network Plan](network.md)

When creating the ELB in the AWS console, ensure that it is running using the TCP load balancer
protocol (not HTTP, HTTPS, or SSL - our TLS termination will happen inside OpenVPN), on port 443,
with an instance protocol of TCP, and an instance port of 443. Place it on the subnets for
your OpenVPN servers (according to the [Network Plan](network.md), 10.4.4.0/25 and 10.4.4.128/25).

You do not need to assign any instances at this time, BOSH will handle that assignment for us.
When prompted for a security group, assign the `openvpn` security group (or create one that
only allows TCP port 443 inbound from 0.0.0.0/0).

With the ELB now created, we can now fill in our network config:

```
$ cat networking.yml
---
meta:
  azs:
    z1: us-west-2a
    z2: us-west-2b
  elbs: [openvpn] # <--- change this to be the name of the ELB you created

networks:
  - name: openvpn_z1
    type: manual
    subnets:
    - range: 10.4.4.0/25
      gateway: 10.4.4.1
      cloud_properties:
        subnet: subnet-XXXXXX # <--------- you'll want to change this
        security_groups: [wide-open]
      dns: [10.4.0.2]
      reserved:
        - 10.4.4.2   - 10.4.4.10  # everything before our /28
      static:
        - 10.4.4.11 - 10.4.4.80  # first 16 IPs
  - name: openvpn_z2
    type: manual
    subnets:
    - range: 10.4.4.128/25
      gateway: 10.4.4.129
      cloud_properties:
        subnet: subnet-XXXXXXX # <--------- you'll want to change this
        security_groups: [wide-open]
      dns: [10.4.0.2]
      reserved:
        - 10.4.4.130   - 10.4.4.140
      static:
        - 10.4.4.141 - 10.4.4.200  # first 16 IPs
```

### Initialze Public Key Infrastructure

Using Vault and `safe` we can begin the process to initialize our Public Key
Infrastructure (PKI).

```
$ safe vault mount pki # loads the pki backend into vault
Successfully mounted 'pki' at 'pki'!
$ safe vault mount-tune -max-lease-ttl=87600h pki # sets max lease ttl to 10 years for the pki backend
Successfully tuned mount 'pki'!
$ safe vault write pki/root/generate/internal common_name=myvault.com ttl=87600h # change the common_name to the name for your CA for this client
<output redacted for brevity>

# Update the configuration for Vault so that it can point to itself when telling people
# how to connect to grab the CA cert or CRL.
$ VAULT_ADDRESS="https://10.4.1.16:8200"
$ safe vault write pki/config/urls issuing_certificates="${VAULT_ADDRESS}/v1/pki/ca" crl_distribution_points="${VAULT_ADDRESS}/v1/pki/crl"
Success! Data written to: pki/ca/urls

# Add our openvpn role:
$ vault write pki/roles/openvpn allowed_domains="openvpn" allow_subdomains="true" max_ttl="87600h"
Success! Data written to: pki/roles/openvpn
```

Now, let's grab the CA and initial CRL (will be empty, but is required nonetheless). The data will be stuck in Vault momentarily:

```
$ curl ${VAULT_ADDRESS}/v1/pki/ca/pem -k
OUTPUT REDACTED
$ curl ${VAULT_ADDRESS}/v1/pki/crl/pem -k
OUTPUT REDACTED
```
### Setup Diffie-Hellman

The [Diffie-Hellman][dh] key-exchange is a secure way to exchange keys.

We will need to generate our DH params via `openssl`, and store it in dh.pem (for now, we'll stick
it in Vault momentarily):

```
$ $ openssl dhparam -out dh.pem 2048
Generating DH parameters, 2048 bit long safe prime, generator 2
This is going to take a long time
......................................+.........................................................................+.....................................+..............................................................................+............+...........................
(output shortened for sanity)
```

## Generate Certificates

Time to generate certificates. Take the private key, and cert values, and store them in
the correct area of the `secret` backend in Vault. **NOTE** the private keys are not stored
after creation, unless you put the data somewhere, so you will need to re-issue the cert
if you lose them. Manually storing in Vault, by creating a yaml file, and importing it to Vault
via `safe` is recommended:

```
$ safe vault pki/issue/openvpn common_name="server.openvpn"
Key             	Value
---             	-----
lease_id        	pki/issue/openvpn/d5c7a33b-68e6-d0f6-3503-6bf3a771a06d
lease_duration  	2591999
lease_renewable 	false
certificate     	REDACTED
issuing_ca      	REDACTED
private_key     	REDACTED
private_key_type	rsa
serial_number   	67:f4:6d:65:4d:0e:b8:7a:fa:be:f0:d5:f5:ae:86:59:e2:4e:e7:96
```

### Repeat on Each Environment

Do this for each of the client certificates needing to be generated as well. Those do not
need to be stored in Vault, but should be securely transferred to each user to be connecting
to the VPN (one cert per user). You *should* stick the serial number in some database (Vault
might be easiest), associated with each user, should you need to revoke it down the road,
as Vault requires the serial number of the certificate to revoke it, and add to the CRL.

To get all of the multi-line certificate/key/pem data into Vault, I recommend
`spruce json | safe import`:

```
$ cat certs.yml
---
secret/aws/proto/openvpn/server:
  cert: |
  REDACTED CERTIFICATE
  key: |
  REDACTED PRIVATE KEY
secret/aws/proto/openvpn/ca:
  cert: |
  REDACTED CA CERTIFICATE
secret/aws/proto/openvpn/crl:
  pem: |
  REDACTED CRL PEM DATA
secret/aws/proto/openvpn/dh:
  pem: |
  REDACTED DH PARAMS PEM DATA
$ spruce json certs.yml | safe import
```

### Cleanup Temporary Files

With that data in vault, you should clean up any temporary files that you used to store
certs/keys prior to placing in Vault:

```
rm certs.yml
rm dh.pem
```

### Deploy OpenVPN

Now, we can finally deploy!

```
make deploy
Treating 'development' version as up-to-date with 0.0.14
  checking https://genesis.starkandwayne.com for details on stemcell bosh-aws-xen-hvm-ubuntu-trusty-go_agent/3262.4
  checking https://genesis.starkandwayne.com for details on release openvpn/2.1.2
  checking https://genesis.starkandwayne.com for details on release toolbelt/3.2.10
  checking https://genesis.starkandwayne.com for details on stemcell bosh-aws-xen-hvm-ubuntu-trusty-go_agent/3262.4
  checking https://genesis.starkandwayne.com for details on release openvpn/2.1.2
  checking https://genesis.starkandwayne.com for details on release toolbelt/3.2.10
Acting as user 'admin' on 'aws-proto-bosh'
Checking whether release openvpn/2.1.2 already exists...YES
Acting as user 'admin' on 'aws-proto-bosh'
Checking whether release toolbelt/3.2.10 already exists...YES
Acting as user 'admin' on 'aws-proto-bosh'
Checking if stemcell already exists...
Yes
Acting as user 'admin' on 'aws-proto-bosh'
Checking if stemcell already exists...
Yes
Acting as user 'admin' on deployment 'aws-proto-openvpn' on 'aws-proto-bosh'
Getting deployment properties from director...

Detecting deployment changes
----------------------------
resource_pools:
- name: openvpn_z1
  stemcell:
    sha1: 58b80c916ad523defea9e661045b7fc700a9ec4f
    url: https://bosh.io/d/stemcells/bosh-aws-xen-hvm-ubuntu-trusty-go_agent?v=3262.4
- name: openvpn_z2
  stemcell:
    sha1: 58b80c916ad523defea9e661045b7fc700a9ec4f
    url: https://bosh.io/d/stemcells/bosh-aws-xen-hvm-ubuntu-trusty-go_agent?v=3262.4
Please review all changes carefully

Deploying
---------
Are you sure you want to deploy? (type 'yes' to continue): yes

Director task 66
  Started preparing deployment > Preparing deployment. Done (00:00:00)

  Started preparing package compilation > Finding packages to compile. Done (00:00:00)

...

Task 66 done

Started		2016-07-27 21:26:04 UTC
Finished	2016-07-27 21:26:04 UTC
Duration	00:00:00

Deployed `aws-proto-openvpn' to `aws-proto-bosh'
```

## Client VPN Configuration

End users should use the following `openvpn` configuration in their clients:

```
client
dev tun
proto tcp
remote YOUR.ELB.HOSTNAME 443
resolv-retry infinite
nobind
persist-key
persist-tun
ca ca.crt
cert vpn.crt
key vpn.key
comp-lzo
verb 3
```

They  will need to be given their private key, and certificate, as well as the CA certificate,
and those placed in the same directory as their configuration file.

### Revoking VPN Access for users

If a users access needs to be revoked, you can use Vault to revoke the certificate, retrieve,
retrieve the new CRL from Vault, store the updated CRL into Vault, and redeploy OpenVPN to via
BOSH:

```
# revoke:
$ safe curl POST pki/revoke '{"serial_number":"<certificate serial number>"}'
$ curl https://10.4.1.16:8200/pki/crl/pem -k
$ cat crl.yml
---
secret/aws/proto/openvpn/crl:
  pem: |
    CRL PEM GOES HERE
$ spruce json crl.yml | safe import
$ rm crl.yml
$ make deploy
```

## Future Notes

We will most likely make a lot of the PKI interaction much easier in the future, since as is, it's
quite manual and not likely to be sustainable. Look for new features built-into `safe` for this,
that may render this documentation out of date.

[dh]: (https://en.wikipedia.org/wiki/Diffie%E2%80%93Hellman_key_exchange)

---
title: On-demand Service Broker Documentation
owner: London Services Enablement
---


# Creating a PCF OpsMan Tile
This documents the process for deploying a ODB with a service in a single tile, on a AWS installation of Ops Manager 1.8. We have built a reference [Kafka title](https://github.com/pivotal-cf-experimental/example-kafka-on-demand-tile).

- [Requirements](#requirements)
- [Deploying OpsMan to AWS](#deploying)
- [Building a tile](#building)
- [Non exhaustive Accessors Reference](#accessors)


<a id="requirements"></a>
## Requirements
Pre ODB, opsman controls the IP allocation of the private network that it lives in. So when using the ODB within a tile, you need to create two subnetworks: one to host the tile VMs and one where the service VMs will be placed.

<a id="deploying"></a>
## Deploying OpsMan to AWS

1. Follow the default OpsMan deployment [docs](https://docs.pivotal.io/pivotalcf/customizing/cloudform-template.html), but with these modifications:
  1. Create a self-signed wildcard SSL certificate for a domain you control: This will usually be `*.some-subdomain.cf-app.com`, and upload it (along with the associated private key) to AWS. Instructions [here](http://docs.aws.amazon.com/ElasticLoadBalancing/latest/DeveloperGuide/ssl-server-cert.html#create-cert).
  1. Download the CloudFormation JSON and save it in the opsman directory.
  1. Run the CloudFormation stack, saving any pertinent inputs (e.g BOSH DB credentials) you type into the web console into the opsman directory for safe keeping (e.g. in `info.txt`).
  1. launch an instance of the AMI. If possible, use an elastic IP so that we can always keep the same DNS record even if we recreate the VM. Failing that, auto-assign a public IP.
  1. create a DNS record for `pcf.<the domain you made a wildcard cert for earlier>`. To use the earlier example, the record will be for `pcf.some-subdomain.cf-app.com`. It should point to the public IP of the ops man VM.
1. Keep following the docs to log into ops man (save the credentials).
1. Configure the ops manager director (BOSH) tile.
1. Click "Apply Changes", and steal the BOSH init manifest for future reference. `scp -i private_key.pem ubuntu@opsmanIP:/var/tempest/workspaces/default/deployments/bosh.yml bosh.yml`

Notes:

1. The ELBs created by CloudFormation are both for CF, not ops man. One of them will be configured with your wildcard certificate. This takes the place of HAProxy in AWS PCF deployments, and is therefore not used until you deploy the ERT tile.
1. To target the bosh director from the OpsMan VM: `bosh --ca-cert /var/tempest/workspaces/default/root_ca_certificate target 10.0.16.10`

<a id="building"></a>
## Building a tile
Follow the default build your own [Product tile documentation](https://docs.pivotal.io/partners/deploying-with-ops-man-tile.html#build-your-own), enhance the handcraft.yml with the accessors listed below. To access the `$self` accessors, the `service-broker` flag has to be set on the handcraft.

<a id="accessors"></a>
## Non exhaustive Accessors Reference
#### director
Used to provide fields relating to the BOSH director installation present.

| Accessor                  |                                                                       Description                                                                        |
|:--------------------------|:--------------------------------------------------------------------------------------------------------------------------------------------------------:|
| $director.deployment\_ip  |                                                                The director's IP address                                                                 |
| $director.ca\_public\_key | The director's root ca certificate. Related: [how to configure SSL certificates for the ODB](operating.html#ssl-certificates). |

For example

```yaml
bosh:
  url: https://(( $director.deployment_ip )):25555
  root_ca_cert: (( $director.ca_public_key ))
```

#### self

Used to provide fields that belong to the specific tile (in this case, the broker tile).

| Accessor                  |                           Description                           |
|:--------------------------|:---------------------------------------------------------------:|
| $self.uaa\_client_name    |  UAA client name, that can authenticate with the bosh director  |
| $self.uaa\_client\_secret | UAA client secret, that can authenticate with the bosh director |
| $self.service\_network    |     Service network configured for the on-demand instances      |


The service network has to be created manually. Create a subnet on AWS and then add it to the director. In the director tile, under Create Networks > ADD network > fill in the subnet/vpc details.

`$self` accessors are enabled by setting `service_broker: true` at the top level of handcraft.yml. Please note that, at the time of writing this, setting `service_broker: true` will cause a redeployment of the bosh director when installing or uninstalling the tile.

For example

```yaml
authentication:
  uaa:
    url: https://(( $director.deployment_ip )):8443
    client_id: (( $self.uaa_client_name ))
    client_secret: (( $self.uaa_client_secret ))
```

For more accessors you can see the [ops-manager-example product](https://github.com/pivotal-cf-experimental/ops-manager-example)
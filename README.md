# Bosh-Lite on Softlayer

## Prereq's

- [Bosh-Deployment](https://github.com/cloudfoundry/bosh-deployment)
- [CF-Deployment](https://github.com/cloudfoundry/cf-deployment)
- Softlayer Account with the correct rights to create VMs and upload stuff
- [SL VPN](https://www.softlayer.com/VPN-Access)

## Deploy the Director

0. **Connect to SL VPN!**

1. **Deploy Director**

Navigate to the `bosh-deployment` repository. 

```
bosh2 create-env bosh.yml \
  -o jumpbox-user.yml \
  -o softlayer/cpi-dynamic.yml \
  -o bosh-lite.yml \
  -o bosh-lite-runc.yml \
  -o <bosh-lite-softlayer>/operations/softlayer/add-hosts-entry.yml \
  --vars-store creds.yml \
  -v director_name=<your-choice> \
  -v sl_vm_name_prefix=<your-choice> \
  -v sl_vm_domain=softlayer.com \
  -v sl_username=<your-sl-user> \
  -v sl_api_key=<your-sl-api-key> \
  -v sl_datacenter=<datacenter> \
  -v sl_vlan_public=<public-vlan> \
  -v sl_vlan_private=<private-vlan> \
  -v internal_ip=<sl_vm_name_prefix>.<sl_vm_domain>
```
*NOTE: The order for the ops files is important, as it ensures that the correct cpi is used for the director (`warden_cpi`)*

_add-hosts-entry.yml_ is required to enable bosh-ssh. (TODO: PR this into bosh-deployment `softlayer/`)

2. Set Alias

`$ bosh -e <internal_ip> --ca-cert <(bosh2 int ./creds.yml --path /director_ssl/ca) alias-env <your-alias>`

3. Bosh Auth

`$ export BOSH_CLIENT=admin  && export BOSH_CLIENT_SECRET=`bosh2 int ./creds.yml --path /admin_password`

or

`$ bosh login --environment <bosh-alias> --client=admin --client-secret=`bosh2 int ./creds.yml --path /admin_password`

---

## Deploy CF

Navigate to the `cf-deployment` repo on your machine. 

1. Apply Cloud Config 

`bosh -e <your-alias> update-cloud-config ./iaas-support/bosh-lite/cloud-config.yml`

2. Upload the right stemcell

e.g.

`bosh -e <your-alias> upload-stemcell https://bosh.io/d/stemcells/bosh-warden-boshlite-ubuntu-trusty-go_agent\?v\=3541.5`

3. Deploy CF

```
bosh2 -e <your-alias> deploy -d cf ./cf-deployment.yml \
    --vars-store deployment-vars.yml \
    -o ./operations/use-compiled-releases.yml \
    -o ./operations/bosh-lite.yml \
    -o ./operations/experimental/use-bosh-dns.yml \
    -o <softlayer-bosh-lite>/operations/cf-deployment/add-dns-entry.yml \
    -v system_domain=<your-dynamic-dns-entry> \
    --no-redact
```

*For the system_domain we added a dyn-dns entry at [changeip.com](http://changeip.com) or you can use [nip.io](http://nip.io/)


## Target CF with CF-CLI

1. Set API Endpoint

`cf api <system-domain>`

2. Auth

Navigate to `cf-deployment`

`$ cf auth admin $(bosh2 int ./deployment-vars.yml --path /cf_admin_password)`

## Bosh SSH

Navigate to `bosh-deployment`

1. Generate `jumpbox.key` 

`$ bosh int creds.yml --path /jumpbox_ssh/private_key > jumpbox.key`

2. chmod the key

`$ chmod 0600 jumpbox.key`

3. Setup socks5 ssh tunnel

`$ ssh -4 -D 5000 -fNC jumpbox@<internal_ip> -i ./jumpbox.key`

4. ssh to a component

`$ BOSH_ALL_PROXY=socks5://localhost:5000 bosh2 -e <your-alias> -d cf ssh <instance>`

(*note that <internal_ip> is the property you defined when used bosh `create-env` command)

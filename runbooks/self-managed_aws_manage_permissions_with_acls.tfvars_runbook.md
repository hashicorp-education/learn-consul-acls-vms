
# Manage permissions with Access Control Lists (ACLs)

This is a solution runbook for the scenario deployed.


## Prerequisites

Login to the Bastion Host

```
$ ssh -i certs/id_rsa.pem admin@`terraform output -raw ip_bastion`
#...
admin@bastion:~$
```


### Configure CLI to interact with Consul

Configure your bastion host to communicate with your Consul environment using the two dynamically generated environment variable files.

```shell-session
$ source "/home/admin/assets/scenario/env-scenario.env" && \
  source "/home/admin/assets/scenario/env-consul.env"
```

That will produce no output.

After loading the needed variables, verify you can connect to your Consul 
datacenter.

```shell-session
$ consul members
```

That will produce an output similar to the following.

```plaintext
Node                  Address          Status  Type    Build   Protocol  DC   Partition  Segment
consul-server-0       10.0.4.165:8301  alive   server  1.17.0  2         dc1  default    <all>
gateway-api-0         10.0.4.10:8301   alive   client  1.17.0  2         dc1  default    <default>
hashicups-api-0       10.0.4.157:8301  alive   client  1.17.0  2         dc1  default    <default>
hashicups-db-0        10.0.4.154:8301  alive   client  1.17.0  2         dc1  default    <default>
hashicups-frontend-0  10.0.4.29:8301   alive   client  1.17.0  2         dc1  default    <default>
hashicups-nginx-0     10.0.4.213:8301  alive   client  1.17.0  2         dc1  default    <default>
```


## Create ACL token for ACL management

To manage ACL a management token is required. It is recommended to create a new Consul management token instead of reusing the bootstrap token.

```shell-session
$ consul acl token create \
  -description="SVC HashiCups API token" \
  --format json \
  -policy-name="global-management" | tee /home/admin/assets/scenario/conf/secrets/acl-token-management.json
```

That will produce an output similar to the following.

```json
{
    "CreateIndex": 200,
    "ModifyIndex": 200,
    "AccessorID": "29d755ef-adde-3910-e4c1-00219eb506a7",
    "SecretID": "0f079e83-c48f-105b-d232-f4971c753aeb",
    "Description": "SVC HashiCups API token",
    "Policies": [
        {
            "ID": "00000000-0000-0000-0000-000000000001",
            "Name": "global-management"
        }
    ],
    "Local": false,
    "CreateTime": "2023-11-18T15:23:53.212182917Z",
    "Hash": "UfTdHatx/OxAw8z+/8suSKQrY6MgpC3CutkuZZAzPC4="
}
```


Use the newly generated token to perform the remaining operations.

```shell-session
$ export CONSUL_HTTP_TOKEN=`cat /home/admin/assets/scenario/conf/secrets/acl-token-management.json | jq -r ".SecretID"`
```

That will produce no output.

## Create ACL configuration for Consul agent

Consul provides a fine grained set of permissions to manage your Consul nodes and services. 
In this scenario you are going to create ACLs tokens to 
register a new Consul client node in a Consul datacenter and to 
register a service in Consul service mesh.

For this purpose the scenario comes with an empty VM, named `hashicups-api-1` that will be used for the test.

When configuring a Consul agent, there is a section in the configuration that permits you to specify ACL tokens to be used by the agent.

```hcl

...
acl {
  tokens {
    agent  = "<Consul agent token>"
    default  = "<Consul default token>"
    config_file_service_registration = "<Consul service registration token>"
  }
}
...
```


The different tokens are used for different operation performed by the Consul agent:
- `agent` - Used for clients and servers to perform internal operations. This token must at least have write access to the node name it will register as in order to set any of the node-level information in the catalog.
- `default` - Used for the default operations when `agent` token is not provided and for the endpoints that do not permit a token being passed. This will be used for the DNS interface of the Consul agent.
- `config_file_service_registration` - Specifies the ACL token the agent uses to register services and checks. This token needs write permission to register all services and checks defined in this agent's configuration.

### Generate `default` token

The scenario already contains a policy used to generate the default token for the other nodes that is already suitable for the node default token.
The policy is named `acl-policy-dns`.


```shell-session
$ consul acl policy read -name acl-policy-dns
```

That will produce an output similar to the following.

```hcl
ID:           3222bd38-77a8-8428-333a-332d3f751f60
Name:         acl-policy-dns
Description:  Policy for DNS endpoints
Datacenters:  
Rules:
# -----------------------------+
# acl-policy-dns.hcl           |
# -----------------------------+

node_prefix "" {
  policy = "read"
}
service_prefix "" {
  policy = "read"
}
# only needed if using prepared queries
query_prefix "" {
  policy = "read"
}
# needed for prometheus metrics
agent_prefix ""
{
  policy = "read"
}
```


The policy guarantees read access to all services and nodes to permit DNS query to work. It also includes read permissions for prepared queries and agent metrics endpoints.

Generate a new token using the `acl-policy-dns` policy.

```shell-session
$ consul acl token create \
  -description="hashicups-api-1 default token" \
  --format json \
  -policy-name="acl-policy-dns" | tee /home/admin/assets/scenario/conf/secrets/acl-token-default-hashicups-api-1.json
```

That will produce an output similar to the following.

```json
{
    "CreateIndex": 201,
    "ModifyIndex": 201,
    "AccessorID": "19e1edef-c841-6635-78b7-3352765eb5b2",
    "SecretID": "35783382-2990-9118-4c25-e5ca72ed4bc4",
    "Description": "hashicups-api-1 default token",
    "Policies": [
        {
            "ID": "3222bd38-77a8-8428-333a-332d3f751f60",
            "Name": "acl-policy-dns"
        }
    ],
    "Local": false,
    "CreateTime": "2023-11-18T15:23:53.442894184Z",
    "Hash": "CR2G+SUUNCQUIQ6pXb9zkajkUOnnxEulm16DCxOnsHY="
}
```


### Generate `agent` token

This token must at least have write access to the node name it will register as in order to set any of the node-level information in the catalog.


Consul provides a convenient shortcut to generate `agent` tokens for Consul nodes. You can use a `node-identity`.
Node identities enable you to quickly construct policies for nodes, rather than manually creating identical polices for each node.

```hcl

# Allow the agent to register its own node in the Catalog and update its network coordinates
node "<node name>" {
  policy = "write"
}

# Allows the agent to detect and diff services registered to itself. This is used during
# anti-entropy to reconcile difference between the agents knowledge of registered
# services and checks in comparison with what is known in the Catalog.
service_prefix "" {
  policy = "read"
}
```


Generate a new token for `hashicups-api-1` using `node-identity`

```shell-session
$ consul acl token create \
  -description="hashicups-api-1 agent token" \
  --format json \
  -node-identity="hashicups-api-1:dc1" | tee /home/admin/assets/scenario/conf/secrets/acl-token-agent-hashicups-api-1.json
```

That will produce an output similar to the following.

```json
{
    "CreateIndex": 202,
    "ModifyIndex": 202,
    "AccessorID": "9ec3ec73-dd51-fb13-c0d2-6f5119fbc7f8",
    "SecretID": "5090b814-59b5-42e6-449b-b53a92204b99",
    "Description": "hashicups-api-1 agent token",
    "NodeIdentities": [
        {
            "NodeName": "hashicups-api-1",
            "Datacenter": "dc1"
        }
    ],
    "Local": false,
    "CreateTime": "2023-11-18T15:23:53.526037103Z",
    "Hash": "dn0S8locXJeqcWcmqHURzQDKSO4UZKwV49QO0YCCbv8="
}
```


### Generate `config_file_service_registration` token

This token needs write permission to register all services and checks defined in this agent's configuration.

Consul provides a convenient shortcut to generate `config_file_service_registration` tokens for Consul nodes. You can use a `service-identity`.
Service identities enable you to quickly construct policies for services, rather than creating identical polices for each service.

```hcl

# Allow the service and its sidecar proxy to register into the catalog.
service "<service name>" {
    policy = "write"
}
service "<service name>-sidecar-proxy" {
    policy = "write"
}

# Allow for any potential upstreams to be resolved.
service_prefix "" {
    policy = "read"
}
node_prefix "" {
    policy = "read"
}

```


Generate a new token for `hashicups-api-1` using `node-identity`.

```shell-session
$ consul acl token create \
  -description="hashicups-api-1 config_file_service_registration token" \
  --format json \
  -service-identity="hashicups-api:dc1" | tee /home/admin/assets/scenario/conf/secrets/acl-token-svc-hashicups-api-1.json
```

That will produce an output similar to the following.

```json
{
    "CreateIndex": 203,
    "ModifyIndex": 203,
    "AccessorID": "f70efe29-dc66-9e6f-99dc-770877a21b0b",
    "SecretID": "e2e5cf5f-aefa-26cb-ae02-fa86a8c52e21",
    "Description": "hashicups-api-1 config_file_service_registration token",
    "ServiceIdentities": [
        {
            "ServiceName": "hashicups-api",
            "Datacenters": [
                "dc1"
            ]
        }
    ],
    "Local": false,
    "CreateTime": "2023-11-18T15:23:53.608311215Z",
    "Hash": "WCtk+YLUNIm5VR0zLykM9oo5TG4xdBbwAIBb3fSZ2qA="
}
```


Verify all tokens are now created and the files are present in the `assets/` folder.

```shell-session
$ ls ~/assets/scenario/conf/secrets/ | grep hashicups-api-1
```

That will produce an output similar to the following.

```plaintext
acl-token-agent-hashicups-api-1.json
acl-token-default-hashicups-api-1.json
acl-token-svc-hashicups-api-1.json
```


### Generate Consul agent ACL configuration section

With the three tokens you created, generate the Consul agent ACL configuration.

```shell-session
$ tee /home/admin/assets/scenario/conf/hashicups-api-1/agent-acl-tokens.hcl > /dev/null << EOF
acl {
  tokens {
    agent  = "`cat /home/admin/assets/scenario/conf/secrets/acl-token-agent-hashicups-api-1.json | jq -r ".SecretID"`"
    default  = "`cat /home/admin/assets/scenario/conf/secrets/acl-token-default-hashicups-api-1.json | jq -r ".SecretID"`"
    config_file_service_registration = "`cat /home/admin/assets/scenario/conf/secrets/acl-token-svc-hashicups-api-1.json | jq -r ".SecretID"`"
  }
}
EOF

```

That will produce no output.

## Generate hashicups-api service configuration

Generate `hashicups-api` service configuration.

```shell-session
$ tee /home/admin/assets/scenario/conf/hashicups-api-1/svc-hashicups-api.hcl > /dev/null << EOF
## -----------------------------
## svc-hashicups-api.hcl
## -----------------------------
service {
  name = "hashicups-api"
  id = "hashicups-api-1"
  tags = [ "inst_1" ]
  port = 8081

  token = "`cat /home/admin/assets/scenario/conf/secrets/acl-token-svc-hashicups-api-1.json | jq -r ".SecretID"`"

  connect {
    sidecar_service {
        proxy {
          upstreams = [
            {
              destination_name = "hashicups-db"
              local_bind_port = 5432
            }
          ]
        }
     }
  }

  checks =[
  {
  id =  "check-hashicups-api.public.http",
  name = "hashicups-api.public  HTTP status check",
  service_id = "hashicups-api-1",
  http  = "http://localhost:8081/health",
  interval = "5s",
  timeout = "5s"
  },
  {
    id =  "check-hashicups-api.public",
    name = "hashicups-api.public status check",
    service_id = "hashicups-api-1",
    tcp  = "localhost:8081",
    interval = "5s",
    timeout = "5s"
  },
  {
    id =  "check-hashicups-api.product",
    name = "hashicups-api.product status check",
    service_id = "hashicups-api-1",
    tcp  = "localhost:9090",
    interval = "5s",
    timeout = "5s"
  },
  {
    id =  "check-hashicups-api.payments",
    name = "hashicups-api.payments status check",
    service_id = "hashicups-api-1",
    tcp  = "localhost:8080",
    interval = "5s",
    timeout = "5s"
  }]
}
EOF

```

That will produce no output.

## Start Consul on hashicups-api-1

The scenario contains a basic Consul configuration for the node.

```shell-session
$ ls /home/admin/assets/scenario/conf/hashicups-api-1
```

That will produce an output similar to the following.

```plaintext
agent-acl-tokens.hcl
agent-gossip-encryption.hcl
consul-agent-ca.pem
consul.hcl
svc-hashicups-api.hcl
```


Copy the configuration on the remote node.

```shell-session
$ scp -r -i /home/admin/certs/id_rsa /home/admin/assets/scenario/conf/hashicups-api-1/* admin@hashicups-api-1:/etc/consul.d/
```

That will produce no output.

Login to `hashicups-api-1` from the bastion host.

```
$ ssh -i certs/id_rsa hashicups-api-1
#..
admin@hashicups-api-1:~
```


Verify the files got copied correctly on the VM.

```shell-session
$ ls /etc/consul.d
```

That will produce an output similar to the following.

```hcl
agent-acl-tokens.hcl
agent-gossip-encryption.hcl
consul-agent-ca.pem
consul.hcl
svc-hashicups-api.hcl
```


Start Consul agent.

```
$ 
/usr/bin/consul agent \
  -retry-join=consul \
  -log-file=/tmp/consul-client \
  -config-dir=/etc/consul.d > /tmp/consul-client.log 2>&1 &
```


```shell-session
$ export _agent_token=`cat /etc/consul.d/agent-acl-tokens.hcl | grep -Po "(?<=registration = \")[^\"]+(?=\")"` && echo ${_agent_token}
```

That will produce an output similar to the following.

```plaintext
e2e5cf5f-aefa-26cb-ae02-fa86a8c52e21
```


Start Envoy sidecar proxy for hashicups-api service.

```
$ 
/usr/bin/consul connect envoy \
  -token=${_agent_token} \
  -envoy-binary /usr/bin/envoy \
  -sidecar-for hashicups-api-1 > /tmp/sidecar-proxy.log 2>&1 &
```


Start `hashicups-api` service.

```shell-session
$ ~/start_service.sh local
```

That will produce an output similar to the following.

```plaintext
e78546bada0f9f3edd8aa5caef3a1c6e55afdb32870f89145128b03baab60b3e
d9e98f4f56031c2a76242e7de7a19c263fae5011531abd36fc0b84cba5229fea
7b3113bd565da8e0322df4f4afddd6369c9ccbf0ef30129c38372e8789308ad8
```


To continue with the tutorial, exit the ssh session to return to the bastion host.

```
$ exit
logout
Connection to hashicups-api-1 closed.
admin@bastion:~$
```


## Verify service registration

Consul CLI

```shell-session
$ consul catalog services -tags
```

That will produce an output similar to the following.

```plaintext
consul                                
gateway-api                           
hashicups-api                         inst_0,inst_1
hashicups-api-sidecar-proxy           inst_0,inst_1
hashicups-db                          inst_0
hashicups-db-sidecar-proxy            inst_0
hashicups-frontend                    inst_0
hashicups-frontend-sidecar-proxy      inst_0
hashicups-nginx                       inst_0
hashicups-nginx-sidecar-proxy         inst_0
```


Consul API

```shell-session
$ curl --silent \
   --header "X-Consul-Token: $CONSUL_HTTP_TOKEN" \
   --connect-to server.${CONSUL_DATACENTER}.${CONSUL_DOMAIN}:8443:consul-server-0:8443 \
   --cacert ${CONSUL_CACERT} \
   https://server.${CONSUL_DATACENTER}.${CONSUL_DOMAIN}:8443/v1/catalog/service/hashicups-api | jq -r .
```

That will produce an output similar to the following.

```json
[
  {
    "ID": "550cbada-8aba-b389-9ce8-1386e701ba4c",
    "Node": "hashicups-api-0",
    "Address": "10.0.4.157",
    "Datacenter": "dc1",
    "TaggedAddresses": {
      "lan": "10.0.4.157",
      "lan_ipv4": "10.0.4.157",
      "wan": "10.0.4.157",
      "wan_ipv4": "10.0.4.157"
    },
    "NodeMeta": {
      "consul-network-segment": "",
      "consul-version": "1.17.0"
    },
    "ServiceKind": "",
    "ServiceID": "hashicups-api-0",
    "ServiceName": "hashicups-api",
    "ServiceTags": [
      "inst_0"
    ],
    "ServiceAddress": "",
    "ServiceWeights": {
      "Passing": 1,
      "Warning": 1
    },
    "ServiceMeta": {},
    "ServicePort": 8081,
    "ServiceSocketPath": "",
    "ServiceEnableTagOverride": false,
    "ServiceProxy": {
      "Mode": "",
      "MeshGateway": {},
      "Expose": {}
    },
    "ServiceConnect": {},
    "ServiceLocality": null,
    "CreateIndex": 76,
    "ModifyIndex": 76
  },
  {
    "ID": "63a36c13-ff4c-c1f3-dcd0-c08b81ad0897",
    "Node": "hashicups-api-1",
    "Address": "10.0.4.47",
    "Datacenter": "dc1",
    "TaggedAddresses": {
      "lan": "10.0.4.47",
      "lan_ipv4": "10.0.4.47",
      "wan": "10.0.4.47",
      "wan_ipv4": "10.0.4.47"
    },
    "NodeMeta": {
      "consul-network-segment": "",
      "consul-version": "1.17.0"
    },
    "ServiceKind": "",
    "ServiceID": "hashicups-api-1",
    "ServiceName": "hashicups-api",
    "ServiceTags": [
      "inst_1"
    ],
    "ServiceAddress": "",
    "ServiceWeights": {
      "Passing": 1,
      "Warning": 1
    },
    "ServiceMeta": {},
    "ServicePort": 8081,
    "ServiceSocketPath": "",
    "ServiceEnableTagOverride": false,
    "ServiceProxy": {
      "Mode": "",
      "MeshGateway": {},
      "Expose": {}
    },
    "ServiceConnect": {},
    "ServiceLocality": null,
    "CreateIndex": 211,
    "ModifyIndex": 211
  }
]
```


Using `dig` command

```shell-session
$ dig @consul-server-0 -p 8600 hashicups-api.service.dc1.consul
```

That will produce an output similar to the following.

```plaintext

; <<>> DiG 9.16.44-Debian <<>> @consul-server-0 -p 8600 hashicups-api.service.dc1.consul
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 25610
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 2, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;hashicups-api.service.dc1.consul. IN	A

;; ANSWER SECTION:
hashicups-api.service.dc1.consul. 0 IN	A	10.0.4.47
hashicups-api.service.dc1.consul. 0 IN	A	10.0.4.157

;; Query time: 0 msec
;; SERVER: 10.0.4.165#8600(10.0.4.165)
;; WHEN: Sat Nov 18 15:29:24 UTC 2023
;; MSG SIZE  rcvd: 93
```


Notice the output reports two instances for the service.

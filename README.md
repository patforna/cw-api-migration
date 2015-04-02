# API Migration Strategy

## Assumptions

In the interest of completing the task in the given timeframe, I've made a number of assumptions which are listed below:

**How do we identify users as v1/v2?**<br/>
Using a HTTP header (i.e. `X-API-VERSION`).

**How does the version header get set?**<br/>
Out of scope. By some other component (e.g. another service, vulcand middleware, etc.)

**How do we identify new/existing users?**<br/>
All we need to know is if we're processing a v1 or v2 request.

**How do we authenticate users?**<br/>
Out of scope. Could be done by verifying a signed header, token, cookie, etc.

**What if incoming request can't be authenticated?**<br/>
Out of scope. Return 400 Bad Request.

**What if incoming request doesn't include the version header?**<br/>
Out of scope. Return 400 Bad Request.

**Do we need to migrate data?**<br/>
No data migration necessary, because API verison share common data store (as per diagram).

## Approach

API v1 and v2 will be live at the same time. Some component (not implemented) then authenticates incoming requests, decides if the should use v1 or v2 and sets the `X-API-VERSION` header accordingly. Based on the value of this header, a request is then routed to the appropriate backend version.

The approach is the same for all three phases. The only thing that changes is the logic in the component responsible for deciding the value of the version header.

### Phase I - Fraction of new users

* Identify user as v1 or v2 and set `X-API-VERSION` header (not implemented).
* Route to v1 or v2 as appropriate.

### Phase II - All new users

* Set `X-API-VERSION` to v2 for "all new" users (not implemented).
* Route to v1 or v2 as appropriate.

### Phase III - All users

* Remove component which was required for setting version header.
* Route all traffic to v2.
* Remove v1.

## Architecture

The solution uses CoreOs, Docker, Etcd, Fleet and Vulcand as the basic infrastructure. Unlike shown in the original diagram, Vulcand will also be deployed onto CoreOs and use the same Etcd cluster.

![Architecture](/docs/arch.png)

## Environment Setup

In order to test the setup, I've created a Vagrant file, which spins up a CoreOs cluster of three machines. We then use this environment to deploy Vulcand and the two versions of the API, which are provided as Docker containers and can be deployed using Fleet.

### Prerequisites
* Vagrant 1.6.3 or greater (tested with 1.7.2)
* Some VM provider (tested with VirtualBox 4.3.20)
* etcdctl (tested with 0.4.6)
* fleetctl (tested with 0.9.1)


### CoreOs / Vagrant

Start up the CoreOs cluster:

```shell
$ cd vagrant
$ vagrant up
```

Verify that all three machines have started:

```shell
$ vagrant status
Current machine states:

core-01                   running (virtualbox)
core-02                   running (virtualbox)
core-03                   running (virtualbox)
```

The IPs assigned to the servers will be `172.17.8.10x` and Etcd should be running on each machine on port `4001`.

```shell
$ curl -w "\n" -L http://172.17.8.101:4001/version
{"releaseVersion":"0.4.7","internalVersion":"1"}
```

### Deploy API apps

The API is a simple Ruby/Sinatra app (see [GitHub](https://github.com/patforna/cw-api)) and automatically provided as Docker image via [DockerHub](https://registry.hub.docker.com/u/patforna/cw-api/). Additionally, this repo contains the unit files for deploying the apps using Fleet.

Launch two instances of each API version:

```shell
$ cd units
$ fleetctl --endpoint http://172.17.8.101:4001 submit api_v1@.service
$ fleetctl --endpoint http://172.17.8.101:4001 start api_v1@1
$ fleetctl --endpoint http://172.17.8.101:4001 start api_v1@2

$ fleetctl --endpoint http://172.17.8.101:4001 submit api_v2@.service
$ fleetctl --endpoint http://172.17.8.101:4001 start api_v2@1
$ fleetctl --endpoint http://172.17.8.101:4001 start api_v2@2
```

Verify that the services have started:

```shell
$ fleetctl --endpoint http://172.17.8.101:4001 list-units
UNIT                    MACHINE                         ACTIVE  SUB
api_v1@1.service        d714e7d6.../172.17.8.102        active  running
api_v1@2.service        ef0b0e00.../172.17.8.103        active  running
api_v2@1.service        ef0b0e00.../172.17.8.103        active  running
api_v2@2.service        d714e7d6.../172.17.8.102        active  running
```

And that they actually work (your IPs may differ):

```shell
$ curl -w "\n" http://172.17.8.102:4567
V1.OUTPUT

$ curl -w "\n" http://172.17.8.103:4568
V2.OUTPUT
```

### Deploy Vulcand

Deployment of Vulcand is the same as the process described above for deploying the API apps:

```shell
$ fleetctl --endpoint http://172.17.8.101:4001 submit vulcand@.service
$ fleetctl --endpoint http://172.17.8.101:4001 start vulcand@1
```

Verify it has started:

```shell
$ fleetctl --endpoint http://172.17.8.101:4001 list-units
UNIT                    MACHINE                         ACTIVE  SUB
...
vulcand@1.service       148b37e3.../172.17.8.101        active  running
```

#### Configure Vulcand

Configuration of Vulcand is done via Etcd.

##### API V1 backend and servers

```shell
$ etcdctl --peers http://172.17.8.101:4001 set /vulcand/backends/b1/backend '{"Type": "http"}'
```

##### API V2 backend and servers

```shell
$ etcdctl --peers http://172.17.8.101:4001 set /vulcand/backends/b2/backend '{"Type": "http"}'
```

##### Frontends

```shell
$ etcdctl --peers http://172.17.8.101:4001 set /vulcand/frontends/f1/frontend '{"Type": "http", "BackendId": "b1", "Route": "Header(`X-API-VERSION`, `1`)"}'
$ etcdctl --peers http://172.17.8.101:4001 set /vulcand/frontends/f2/frontend '{"Type": "http", "BackendId": "b2", "Route": "Header(`X-API-VERSION`, `2`)"}'
```

### Test

If all has gone well, you should now be able to route requests to the correct API servers, based on the version header you include in the request (your IP may differ - depending on which server Vulcand has been deployed):

```shell
$ curl -w "\n" --header "X-API-VERSION: 1" http://172.17.8.101
V1.OUTPUT

$ curl -w "\n" --header "X-API-VERSION: 2" http://172.17.8.101
V2.OUTPUT
```

### Notes

This is just a proof of concept and there's quite a bit of outstanding work that would have to happen in order to productionise this, such as:

* Set up in Cloud or own DataCentre
* DNS
* Write component that decides if users are v1 or v2 (service, vulcand middleware, etc.)
* Authenticate users and requests
* Ensure consistent experience by always routing same users to same version (Phase I)
* More than one vulcand instance
* Automatic registration of Vulcand IPs in DNS
* Automatic registration of API IPs in Vulcand
* Release pipeline
* etc.

#### Pros/Cons

* Pro: approach is highly scalable, available
* Con: lots of new technologies; complexity.
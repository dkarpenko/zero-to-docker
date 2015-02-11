# Zero-to-Docker: Baby step onto Private Docker Registry
After building you first Docker containers you looking for a place place to store and share docker containers. Thins page should help you when realize that using [hub.docker.io](hub.docker.io) is not enough for you.

http://blog.octo.com/en/docker-registry-first-steps/

## Running Docker Registry
[Official Registry Server for Docker](https://github.com/docker/docker-registry) is a web application which can hosted on any server capable to run python applications.
### Running Docker Registry as a Docker container
The fastest way to get running:
* install docker
* run the registry: `docker run -p 5000:5000 registry`

All possible configuration options available at official [Docker Registry page](https://github.com/docker/docker-registry#quick-start). You can configure backend for storing containers, mirroring, caching, search engine for `GET /v1/search` [endpoint](http://docs.docker.com/reference/api/docker-io_api/#search)
### Manual configuration of server and application
The best resource describing fully manual configuration is an [tutorial published on DigitalOcean](https://www.digitalocean.com/community/tutorials/how-to-set-up-a-private-docker-registry-on-ubuntu-14-04)
This tutorial describes setup on Ubuntu 14.04 but may be used for other Debian-based distributions.
Topics covered in this publications are:
* environment pre-configuration
* registry installation
* adding Authorization (with Nginx)
* generating and adding SSL certificate.

Also this article can be really helpful for you, if you want to automate setup of registry with any tool (Chef, Puppet or even Docker container).
### Configuring Docker Registry with Chef
if you decided just to install docker on instance it's more preferable to use (Chef cookbook)[https://github.com/bflad/chef-docker].
Cookbook supports installation on a variety of platforms and provides recipes for managing containers (pull/push form private registry, tagging containers, ...).
This cookbook can really help you not even to setup Docker Registry, but automate some routine related to containers management on your Dev/QA instances (pull, stop, start, kill).
```ruby
# Login to private registry
docker_registry 'https://docker-registry.example.com/' do
  username 'shipper'
  password 'iloveshipping'
end

# Pull tagged image
docker_image 'apps/crowsnest' do
  tag 'not-latest'
end

# Run container
docker_container 'crowsnest'

# Save current timestamp
timestamp = Time.new.strftime('%Y%m%d%H%M')

# Commit container changes
docker_container 'crowsnest' do
  repository 'apps'
  tag timestamp
  action :commit
end

# Push image
docker_image 'crowsnest' do
  repository 'apps'
  tag timestamp
  action :push
end
```

### Configuring Docker Registry with Ansible
In the same manner as you configure Docker Registry using Chef, you can use Ansible playbook [nextrevision/ansible-docker-registry](https://github.com/nextrevision/ansible-docker-registry).
It allows you to setup registry, configure SSL and storage (local or S3) for containers.
Also Ansible can be used for managing Docker containers; some details on this available on their [official page](http://www.ansible.com/docker).


### Testing connection to Docker Registry
To test that the registry is up you can simply call:
```bash
curl <docker_registry_host>
 # depending on your installation options
 curl <docker_registry_host>:<port>
```

## Preparing client environments to work with private Docker Registry (insecure-registry)
More probably that initially you've configured your private Docker Registry without SSL. In this case you have to make some actions to be able to work with the registry.

#### Configuring MacOS and Windows

```bash
boot2docker init
boot2docker up
boot2docker ssh
echo 'EXTRA_ARGS="--insecure-registry <docker_registry_host>:<port>"' | sudo tee -a /var/lib/boot2docker/profile
sudo /etc/init.d/docker restart
```

### Configuring Ubuntu
```bash
sudo vim /etc/default/docker
export DOCKER_OPTS="$DOCKER_OPTS --insecure-registry=<docker_registry_host>:<port>"
sudo service docker restart
```
All details you can find on the section dedicated to [insecure-registry](https://github.com/boot2docker/boot2docker#insecure-registry).

## Publishing image
 Let's assume that `<image_id>` is and ID of Docker image you see via `docker images` command.


 ```bash
 $ docker images
 REPOSITORY                          TAG                 IMAGE ID            CREATED             VIRTUAL SIZE
 marvambass/nginx-registry-proxy     latest              6aec975ff87a        4 weeks ago         101.4 MB
 dockerfile/elasticsearch            latest              f024714f6b3e        5 weeks ago         751.3 MB
 dockerfile/java                     latest              e5b3ce7f0b79        5 weeks ago         705.7 MB
 dockerfile/java                     oracle-java7        6191d06ead72        5 weeks ago         717.4 MB
 dockerfile/java                     oracle-java8        68916afd53e5        5 weeks ago         750.8 MB
 dockerfile/mongodb                  latest              0980bbd7909c        5 weeks ago         702.8 MB

 ```

```bash
docker tag <image_id> <docker_registry_host>/<container_name>:<container_version>
docker push 172.22.8.70:5000/<container_name>
```
or for publishing with more specific container
```bash
docker tag 6aec975ff87a <docker_registry_host>/my-container:0.1-SNAPSHOT
docker push <docker_registry_host>/my-new-container
```

## Pulling images
To pull the latest version of container in general
```bash
docker pull <docker_registry_host>/<container_name>
```
or for pulling specific container
```bash
docker pull <docker_registry_host>/my-container
```

To pull particular version of container in general
```bash
docker pull <docker_registry_host>/<container_name>:<container_version>
# more specific example
docker pull 172.22.8.70:5000/my-container:0.1-SNAPSHOT
```


## Searching for images
To list all images in registry call
```
curl -X GET http://<registry_host>:5000/v1/search
```

You can specify what images you are searching for by passing additional parameter:
```
curl -X GET http://<registry_host>:5000/v1/search?q=centos
```
Also, Docker client can be used to [search for images](https://docs.docker.com/reference/commandline/cli/#search) in your private registry:
```
docker search <registry_host>:5000/centos
```

## Advanced
### Dogestry

After configuring your local docker client and server machines to work with private ('insecured') Docker Registry you may think about leaving this idea aside. But before that you should try [Dogestry](https://github.com/dogestry/dogestry) - simple tool which mimics the work of the Docker Registry.

```
This is a simple client you can run where you run docker - and you don't need a registry - it talks directly to s3 (for example). docker save/load is the mechanism used.
```

Push the hipache image to the S3 bucket ops-goodies located in us-west-2:
```bash
dogestry push s3://ops-goodies/?region=us-west-2 hipache
```
Pull the hipache image and tag from S3 bucket ops-goodies:
```bash
dogestry pull s3://ops-goodies/docker-repo/?region=us-west-2 hipache
```
An amazing tool!
### Secure Docker Registry
#### SSH Tunneling

```bash
# Run the registry on the server, allow only localhost connection
docker run -p 127.0.0.1:5000:5000 registry

# On the client, setup ssh tunneling
ssh -N -L 5000:localhost:5000 user@server
```
[Stackoverflow answer](http://stackoverflow.com/a/25136952)

#### Nginx proxy
It's a common practice to use Docker containers to add additional functionality even over Docker Registry itself. With [marvambass/nginx-registry-proxy](https://github.com/MarvAmBass/docker-nginx-registry-proxy) you can easily add Nginx reverse proxy with SSL and Basic Auth over Docker Registry.

There is an information in this repository on generating self-signed SSL certificates and adding user accounts which will help you make your Docker Registry secured in a few minutes.

```bash
docker run -d -p 443:443 \
-v $PATH\_TO\_YOUR/external:/etc/nginx/external \
--link registry:registry --name nginx-registry-proxy \
marvambass/nginx-registry-proxy
```
Don't forget to [login](https://docs.docker.com/reference/commandline/cli/#search) to Docker Registry after setting up Basic Auth.

##### Authentication
As Nginx is a very customizable server it can be very easy to integrate it with [LDAP](https://github.com/kvspb/nginx-auth-ldap) ([more details](http://www.allgoodbits.org/articles/view/29)), [SPNEGO, GSSAPI, and Kerberos](https://github.com/fintler/nginx-mod-auth-kerb) authentication systems.

#### Docker index
[Docker Index](https://github.com/ekristen/docker-index) is an API based tool for managing access control to Docker Registry, managing account which will be used in `docker login` command, registering hooks for image pushes.

```
The purpose of this project is to provide the docker masses with an authenticated docker index for their own docker registry that has real repository access controls instead of just using htaccess on a reverse proxy.

Technically now called the Docker Hub (by Docker), this is an independently developed version using the Docker Hub and Registry Spec and throw a tiny bit of reverse engineering.

This is a functioning Docker Index (Docker Hub) that can be run independent of the Docker Registry software.

This project does not have a front-end UI for administration or for viewing of repositories and images, to be able to view this information you will have to use the command line tool that was written to work with it.
```

#### Web interface to Docker Registry
Because of [API based Docker architecture](http://docs.docker.io/en/latest/api/docker_remote_api/), there is a lot of web UI applications for Docker Registry.

##### docker-registry-web

One of the good examples of Web UI over Docker Registry is [atcol/docker-registry-ui(https://github.com/atc-/docker-registry-web)

```
A web UI for easy private/local Docker Registry integration. Allows you to browse, delete and search for images as well as register multiple registries for large installations.
```

To run it, you just need to specify path to http Docker Registry API:
```
docker run -p 8080:8080 -e REG1=http://registry_host.name:5000/v1/ atcol/docker-registry-ui
```

### Storage backends
Depending on you reliability/availability and amount of containers you are going to store, you can use different storage types. Docker Registry provides different options for this purpose:
* local: stores data on the local filesystem
* s3: stores data in an [AWS S3](http://aws.amazon.com/s3/) bucket
* ceph-s3: stores data in a [Ceph cluster](http://ceph.com/docs/master/rados/) via a [Ceph Object Gateway](http://ceph.com/docs/master/radosgw/), using the S3 API
* azureblob: stores data in an [Microsoft Azure Blob Storage](http://azure.microsoft.com/en-us/services/storage/)
* gcs: stores data in [Google cloud storage](https://cloud.google.com/storage/)
* swift: stores data in [OpenStack Swift](http://docs.openstack.org/developer/swift/)
* glance: stores data in [OpenStack Glance](http://docs.openstack.org/developer/glance/), with a fallback to local storage
* glance-swift: stores data in OpenStack Glance, with a fallback to Swift
* elliptics: stores data in [Elliptics](http://reverbrain.com/elliptics/) key/value storage


### Registry API
There are a few documents describing an API of the Docker Hub and Registry:
* [Docker Hub API](https://docs.docker.com/reference/api/docker-io_api/)
* [The Docker Hub and the Registry spec](https://docs.docker.com/reference/api/hub_registry_spec/)

### JFrog Artifactory
With such features, already implemented, as LDAP and SAML integration, ACLs, Artifactory can be a good option for 'corporate' customers. And, of course, it implements all Docker API.
http://www.jfrog.com/article/docker/
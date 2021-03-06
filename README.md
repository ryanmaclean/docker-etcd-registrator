# docker-etcd-registrator

Docker service registrator for etcd (and CoreOS).
The very end of `sidekick.service`

* [SkyDNS](https://github.com/skynetservices/skydns) support
* [Vulcanproxy](http://vulcanproxy.com) support
* Startup synchronization: bring etcd up to date 
 * Add already running containers
 * Remove stopped but registred container
* Realtime: Listening for docker events
* Registers all ports
 * defined via `EXPOSE` in the `Dockerfile`
 * exposed via `-p` commandline argument
* Supports secured etcd
* Service config using ENV
* Written in Javascript
* for (but not limited to) CoreOS, see [fleet-unit-files](https://github.com/psi-4ward/docker-etcd-registrator/tree/master/fleet-unit-files)

*(thanks to [gliderlabs/registrator](https://github.com/gliderlabs/registrator) for the some ideas)*

### TODO / Planned

* Configuration using commandline arguments
* Improve docu

## Install &amp; Config

* You need NodeJS >= 0.12.x and NPM; Should also run with IO.JS
* For now its only possible to configure docker-etcd-registrator using environment variables
* Make sure the app can read/write to `DOCKER_HOST` (default: `/var/run/docker.sock`)

```shell
sudo npm install -g docker-etcd-registrator

DEBUG=docker,skydns,service \
  ETCD_ENDPOINTS=http://10.1.0.1:4001,http://10.1.0.2:4001 \
  docker-etcd-registrator
```

**Docker**

```shell
docker run --rm \
  --name docker-etcd-registrator \
  -v /etc/ssl/etcd:/etc/ssl/etcd \
  -v /var/run/docker.sock:/var/run/docker.sock \
  --env DEBUG=docker,skydns,vulcand,container \
  --env HOSTNAME=`hostname` \
  --env ETCD_ENDPOINTS=https://10.1.0.1:4001,https://10.1.0.2:4001 \
  --env ETCD_CAFILE=/etc/ssl/etcd/ca-authority.pem \
  --env ETCD_CERTFILE=/etc/ssl/etcd/certificate.crt \
  --env ETCD_KEYFILE=/etc/ssl/etcd/key.key \
  psitrax/docker-etcd-registrator
```

**Manual:**

```shell
git clone https://github.com/psi-4ward/docker-etcd-registrator.git
cd docker-etcd-registrator
npm install
ETCD_ENDPOINTS=http://10.1.0.1:4001 node app.js
```

### Config parameters

All params are optional

* `HOSTNAME`: Hostname of the system
* `REGISTER=public`: Register only Ports which binds to the host interface (docker -p)
* `REGISTER_PUBLIC_IP=10.0.1.1`: IP if Hostbinding dont specify any (docker -p 80:80 instead of docker -p 10.0.1.1:80:80)
* `SKYDNS_ETCD_PREFIX`: `/skydns/local/skydns`
* `VULCAND_ETCD_PREFIX`: `/skydns/local/skydns`
<br>
* `DOCKER_HOST`: `/var/run/docker.sock` or `tcp://localhost:2376`
* `DOCKER_TLS_VERIFY` from docker-modem
* `DOCKER_CERT_PATH`: Directory containing `ca.pem`, `cert.pem`, `key.pem` (filenames hardcoded) 
<br>
* `ETCD_ENDPOINTS`: `http://127.0.0.1:4001`
* `ETCD_CAFILE`
* `ETCD_CERTFILE`
* `ETCD_KEYFILE`

### Debug
Enable debugging using `DEBUG` env var: `DEBUG=docker,skydns,service node app.js`

flag       | description
-----------|-----------------------------
 *         | print every debug message |
 docker    | docker related messages   |
 conteiner | container-inspect => service transformation |
 skydns    | skydns etcd data population | 
 vulcand   | skydns etcd data population | 
 modem     | raw docker socket messages | 


## Service Discovery Configration

* Use env vars to configure a specific container / service
* Everything is optional
* Name is received from `SERVICE_NAME` or `--name` or the container ID
* Services with `SERVICE_IGNORE` are not observed

```
$ docker run -d --name mariadb \
    -e "SERVICE_NAME=mysql" \
    -e "SERVICE_TAGS=database,customers" \
    mariadb
```

### Multiple Services per Container

You can specify a service identified by a given port `SERVICE_<PORT>_<FLAG>`:
```
$ docker run -p 80:80 -p 443:443 -p 9000:9000 \
    -e "SERVICE_80_NAME=http-proxy" \
    -e "SERVICE_443_NAME=https-proxy" \
    -e "SERVICE_9000_IGNORE=yes" \
    docker/image
```

### Vulcand
Use `SERVICE_[PORT_]VULCAND_(BE|FE)_` formatted env vars to generate etcd values for Vulcanproxy.
Per default registrator will not generate any vulcand frontend or backend.

In general the `SERVICE_VULCAND_FE_k1_k2_k3=value` style would result in a JSON structure like: `{"k1": {"k2": {"k3": "value"} } }`

Generate a vulcand-backend of type http using the defaults for every port but 9000: 
```shell
$ docker run -p 80:80 -p 443:443 -p 9000:9000 \
    -e "SERVICE_NAME=websrv" \
    -e "SERVICE_VULCAND_BE_Type=http" \
    -e "SERVICE_9000_IGNORE=yes" \
    docker/image
```

Defining more FE/BE settings
```shell
$ docker run -p 3000:3000 -p 22:22 \
    -e "SERVICE_22_IGNORE=yes" \
    -e "SERVICE_3000_NAME=microservice" \
    -e "SERVICE_3000_VULCAND_BE_Type=http" \
    -e "SERVICE_3000_VULCAND_BE_Settings_Timeouts_Read=10s" \
    -e "SERVICE_3000_VULCAND_BE_Settings_KeepAlive_MaxIdleConnsPerHost=20" \
    -e "SERVICE_3000_VULCAND_FE_Type=https" \
    -e "SERVICE_3000_VULCAND_FE_Route=Host('ms.example.com')" \
    -e "SERVICE_3000_VULCAND_FE_Settings_Limits_MaxBodyBytes=4048" \
    docker/image
```


## Authors

* Christoph Wiechert



## License

  [MIT](LICENSE)
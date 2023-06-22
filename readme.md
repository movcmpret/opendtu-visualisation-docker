# openDTU visualisation docker setup 
![grafana](https://github.com/movcmpret/opendtu-visualisation-docker/assets/43320190/4a0e25f8-e559-4233-9262-66fcad97d611)

This docker setup contains all services needed in order to receive, provide, store and visualize data coming from your openDTU device.

## Infrastructure

- [mosquitto](https://mosquitto.org/) **MQTT broker:** receives data via MQTT
- [telegraf](https://github.com/influxdata/telegraf) **data collection agent**: subscribes to all configured topics in your mosquitto service and stores them in the influx database.
- [influxDB](https://www.influxdata.com/) **high speed database**: Keeps you data persistent and structured. Grafana data source.
- [grafana](https://grafana.com/) **analytics platform**: 

## Prerequisites

- Linux server (raspberry pi, vps, etc.); not tested on windows
- `docker` and `docker-compose`
- `openssl`

**Nice to have but not necessary:** 
- public ip/domain with A-record

## Relevant files

- mosquitto_config
- - mosquitto.conf
- mosquitto_tls
- - chain.pem
- - privkey.pem
- - cert.pem
- telegraf
- - telegraf.conf
- grafana.ini

## Starte here

Clone this repository.

### .env file

Execute `cp .env.example .env` to create an environment file for `docker-compose`.
You might change any value. If you want to access grafana directly on your host and 
don't want to deliver it via reverse proxy you might change `GRAFANA_PORT` to fit
your needs.

### Mosquitto

#### server, ports, auth

In order to connect to your broker, you need to bind the port to your
host machine and alter your firewall. The services inside the docker network don't
need TLS, which is why there are 2 listeners. One for public access with TLS (`1883`) 
and one used within the docker network (`1884`).

mosquitto.conf
```ini
#public
listener 1883
persistence true
[...]
#private / local network
listener 1884 0.0.0.0
```

In the `docker-compose` file the port `1883` is exposed and bound to the host machine.
If you have a firewall (in my case iptables) you need to add an allow rule on 
port 1883/TCP to you input chain.

```bash
iptables -A INPUT -p tcp --dport 1883 -j ACCEPT -m comment --comment "MOSQUITTO"
```

#### TLS

Mosquitto's traffic can be secured via TLS. I recommend to configure it, however 
in some cases e.g. in local are networks, tls might not be used. In this case you would
comment out `cafile`, `keyfile`, `certfile` in `mosquitto.conf` to disable TLS.

I will explain two ways of using TLS encryption. 

**Self-signed certificates** 

If you don't have a domain you should go for this one. You will create your own
certificate, which will be used for TLS.

1.  `cd mosquitto_tls`
2. `openssl genrsa -des3 -out ca.key 2048` and enter some passphrase
3. `openssl req -new -x509 -days 9000 -key ca.key -out chain.pem`
4. `openssl genrsa -out server.key 2048`
5. `openssl req -new -out privkey.pem -key server.key`
6. `openssl x509 -req -in privkey.pem -CA chain.pem -CAkey ca.key -CAcreateserial -out cert.pem -days 3600`

You will need all `*.pem` files for the mosquitto configuration. You should keep the other files somewhere safe.


**Letsencrypt certificates**

In case you have a domain I recommend to use letsencrypt/certbot to obtain tls certificates.
You will need a configured certbot for this step. 

`sudo certbot certonly --standalone -d grafana.example.com`

Copy or symlink the files in `/etc/letsencrypt/live/grafana.example.com` to `./mosquitto_tls`

**User config**

The default login is _admin:admin_ To change or create an account please use `mosquitto_passwd`
inside the container (after build and start):

```bash
docker compose exec mosquitto mosquitto_passwd -b /mosquitto/config/password.txt admin supersecretpassword1234
```

### Telegraf

If you changed your mosquitto auth data make sure to configure `username` 
and `password` in your `telegraf.conf`.  

If you want to have weather data available you need to create a free account 
at [OpenWeatherMap](https://openweathermap.org/) to obtain an API Key. Replace
each `YOURAPIKEY` in the default telegraf config file `telegraf.conf`.

Set `YOUR_CITY_ID`, latitude and longitude in this config as well.

### InfluxDB

You don't need to configure anything here :)

### Grafana

In `grafana.ini` replace `domain = grafana.example.com` to your domain/ip address.


### Build and start

Execute `docker-compose up -d` to start the container and detach you terminal.

The grafana service is bound to port `GRAFANA_PORT`. It's easier to access it directly,
but I recommend to use a reverse proxy.

The default login is `admin:admin`. You are forced to set a new password immediately 
after your first login.



## Credits

https://github.com/Setcover/smarthome/blob/main/telegraf/telegraf.conf

https://github.com/vvatelot/mosquitto-docker-compose

http://www.steves-internet-guide.com/mosquitto-tls/

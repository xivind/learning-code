# Background
The code in this repo is meant for experimenting and learning basic programming, with emphasis on containers, messagebrokers, data visualisation and git as a bonus. Some of the sources that are used are [strava.com](https://www.strava.com), [yr.no](https://www.yr.no) and [nilu.no](https://www.nilu.no). InfluxDB is used to store the data, Grafana to visualize the data and Mosquitto to move data across the containers. The scripts that interacts with the data sources are all python. Everything runs in docker containers. The code is tested on Raspberry Pi 4b.

- **The code is for testing purposes only, so security is not configured**
- **All comments, suggestions and pull requests to improve the code is very welcome, see open issues..**
- **This is a learning effort, so expect many strange choices and errors in the code, waiting to be corrected**

> The instructions in this readme file is a little rough around the edges, but will be improved over time...
# InfluxDB
Create the InfluxDB container with the command below. This is the container that runst the actualt InfluxDB server. To persist the data create a docker volume first. And also update the path to the default InfluxDB config file, `influxdb.conf`.
```
docker run -d \  
--name=influxdb \  
-e TZ=Europe/Stockholm \  
--mount type=bind,source=/<path on host>/influxdb.conf,target=/etc/influxdb/influxdb.conf \  
--mount type=volume,source=<docker volume on host>,target=/var/lib/influxdb \  
-p 8086:8086 \  
--restart unless-stopped \  
influxdb:1.8.6 -config /etc/influxdb/influxdb.conf
```
With the commands below, build the image and create the container that will run the python script that writes to InfluxDB. The python script listens to mqtt for messages with data to write. See the Strava container below for an example of how the messages must be formatted. Create a separeate container for each data source, e.g. one for Strava, one for Yr etc. The commands must be run from the directory that contains `Dockerfile` and `requirements.txt`  

`docker build -t <name of image> .`

```
docker run -d \  
--name=<name of container> \  
-e TZ=Europe/Stockholm \  
--restart unless-stopped \  
<name of image> \  
./receive-timeseries.py \  
--mqttHost <mosquitto host> \  
--mqttPort 1883 \  
--mqttKeepalive 60 \  
--mqttTopic <mosquitto topic> \  
--influxHost <influxdb host> \  
--influxPort 8086 \  
--influxUser <influxdb user> \  
--influxPassword <influxdb password> \  
--influxDatabase <influxdb database> \  
--influxMeasurement <influxdb measurement>
```
# Strava
Create the container that retrieves data from Strava with the command below. This container only retrieves data from Strava and sends it to the mqtt broker. It does not need to persist data. The commands must be run from the directory that contains `Dockerfile` and `requirements.txt`. The script here is quite rudimentary in regards to the handling of data. Benji Knights Johnson has a better approach, using Pandas, that is described [in an article on medium](https://medium.com/swlh/using-python-to-connect-to-stravas-api-and-analyse-your-activities-dummies-guide-5f49727aac86).

`docker build -t strava .`

```
docker run -d \  
--name=strava \  
-e TZ=Europe/Stockholm \  
-v /<path on host to directory containing file with tokens>:/secrets \  
--restart unless-stopped \  
strava \  
./send-strava.py \  
--debug no \  
--tokens /secrets/<name of filen with tokens> \  
--mqttHost <mosquitto host> \  
--mqttPort 1883 \  
--mqttTopic <mosquitto topic> \  
--mqttClientID <mosquitto clientID>
```

Strava requires authentication with OAuth2. It looks complicated, but is fairly straight forward to configure if you follow [Stravas step-by-step guide](https://developers.strava.com/docs/getting-started/#oauth). In the script we are using here, the tokens are handled through a json file, which must look as following.

`{"clientId": "", "clientSecret": "", "accessToken": "", "refreshToken": ""}`
# Fitbit
Create the container that retrieves data from Fitbit with the command below. This container only retrieves data from Fitbit and sends it to the mqtt broker. It does not need to persist data. The commands must be run from the directory that contains `Dockerfile` and `requirements.txt`

`docker build -t fitbit .`

```
docker run -d \  
--name=fitbit \  
-e TZ=Europe/Stockholm \  
-v /<path on host to directory containing file with tokens>:/secrets \  
--restart unless-stopped \  
fitbit \  
./send-fitbit.py \  
--debug no \  
--tokens /secrets/<name of filen with tokens> \  
--mqttHost <mosquitto host> \  
--mqttPort 1883 \  
--mqttTopic <mosquitto topic> \  
--mqttClientID <mosquitto clientID>
```

Fitbit requires authentication with OAuth2. In the script we are using here, the tokens are handled through a json file, which must look as following. 

`{"AccToken": "", "RefToken": "", "ClientId": "", "ClientSecret": ""}`

[Thanks to Pauls Geek Dad Blog](https://pdwhomeautomation.blogspot.com/2016/01/fitbit-api-access-using-oauth20-and.html) for pointing us in the right direction with python and OAuth2 also.

# Nilu
Create the container that retrieves data from [nilu.no](https://www.nilu.no) with the command below. This container only retrieves data from [nilu.no](https://www.nilu.no) and sends it to the mqtt broker. It does not need to persist data. The commands must be run from the directory that contains `Dockerfile` and `requirements.txt`

`docker build -t nilu .`

```
docker run -d \  
--name=nilu \  
-e TZ=Europe/Stockholm \  
--restart unless-stopped \ 
nilu \  
./send_nilu.py \  
--debug no \  
--url <URl to Nilus API>
--user_agent <email address of entity using the API> \  
--mqtt_host <mosquitto host> \  
--mqtt_port 1883 \  
--mqtt_topic <mosquitto topic> \  
--mqtt_client_id <mosquitto clientID>
```

# Yr
Create the container that retrieves data from [yr.no](https://www.yr.no) with the command below. This container only retrieves data from [yr.no](https://www.yr.no) and sends it to the mqtt broker. It does not need to persist data. The commands must be run from the directory that contains `Dockerfile` and `requirements.txt` . Update the `urls.json` file Yr resources as required.

`docker build -t yr .`

```
docker run -d \`  
--name=yr \`  
-e TZ=Europe/Stockholm \`  
--restart unless-stopped \` 
yr \`  
./send_yr.py \`  
--debug no \`  
--url_file <path and filename of json file with Yr resources>`
--user_agent <email address of entity using the API> \`  
--mqtt_host <mosquitto host> \`  
--mqtt_port 1883 \`  
--mqtt_topic <mosquitto topic> \`  
--mqtt_client_id <mosquitto clientID>`
```
# Grafana
Once InfluxDB is set up and receives data over Mosquitto from the containers that retrieve data from Strava, Fitbit, Nilu and Yr, the data can easily be visualised in Grafana. Just set up [InfluxDB as datasource](https://grafana.com/docs/grafana/latest/datasources/add-a-data-source/) and create the diagrams that suits you. Some examples below 

Basic visualisation of data from Strava
![data from Strava](/grafana/strava.png)

Basic visualisation of data from Fitbit
![data from Fitbit](/grafana/fitbit.png)

Basic visualisation of data from Nilu and Yr
![data from Nilu and Yr](/grafana/climate.png)
# Blinkt
This scipt uses the [ledstrip](https://shop.pimoroni.com/products/blinkt) from [Pimoroni](https://shop.pimoroni.com/) to display system status. To make it run in the background, create a service file with the content below and place it in `/etc/systemd/system` Use `sudo systemctl enable <name of service file>` to make it run on system startup
>`[Unit]`  
`Description=Control status LEDs for RPI4s`  
`After=network.target`  
`[Service]`  
`ExecStart=/usr/bin/python3 -u <full path to python script>`  
`rpi4-statusleds.py`  
`WorkingDirectory=<full path to directory that contains the python script>`  
`StandardOutput=inherit`  
`StandardError=inherit`  
`Restart=always`  
`User=<name of user to run the script>`  
`[Install]`  
`WantedBy=multi-user.target`  

The Pimoroni Blinkt in use at a Raspberry Pi4b with the [Argon40 Argon ONE M.2 Case](https://www.argon40.com/argon-one-m-2-case-for-raspberry-pi-4.html)
![Pimoroni Blinkt](/blinkt/IMG_20210722_000358.jpg)

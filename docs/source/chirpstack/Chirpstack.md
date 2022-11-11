# Chirpstack

Chirpstack is an open-source LoRaWAN Network Server stack that provides the essential components to build user-friendly web interfaces for device management. Chirpstack APIs are provided for integration purposes, allowing LoRaWAN stack to be easily integrated with applications.


Chirpstack is composed of three modules that work together for a client to be able to communicate with end-devices. These operate as follows.

<center>

![image cs](./../assets/img/cs-arq.svg)

</center>

<center>

Chirpstack Arquitecture - Image taken from [Chirpstack Website](https://www.chirpstack.io/docs/architecture.html)

</center>



<!------------------------------------ Gateway Bridge ------------------------------------------------------>


## Gateway Bridge


The Chirpstack Gateway Bridge (CGB) is the module of Chirpstack LoRaWAN stack that converts the Packet Forwarder message payloads into a serialized data format (JSON or Protobuf) and publishes it in a Message Queuing Telemetry Transport (MQTT) server, to which the Network Server subscribes. The message will be published with a topic that indicates the source (by default the gateway MAC number and event type: up, down, status). This module can also be configured to support different types of packet forwarders.




### Requirements

- Any MQTT broker for instance [Mosquitto](https://mosquitto.org/download/)

```bash
sudo apt-get install mosquitto
```

If you wanna pub or sub to mqtt you will need to install a pub/sub package (optional: used for debugging).

```bash
sudo apt-get install mosquitto-clients
```


### Instalation

```bash
sudo apt install chirpstack-gateway-bridge
```

### Configurations

The configurations for the gateway bridge will be located at `/etc/chirpstack-gateway-bridge/chirpstack-gateway-bridge.toml`

By default the gateway bridge will listen to port 1700 and will encode it with json and sent it to port localhost:1883 (MQTT default).

If you want to do add new message formats that arent `stats|uplinks|downlinks|execs|acks` you can add them in the configuration file. The topic format for up/downlinks is usually `gateway/[gateway_id]/event/[event_type]`. You can check more about the formats and events in [here](https://www.chirpstack.io/docs/chirpstack-gateway-bridge/payloads/events.html).

You can also change the detail of the logs, it goes from 0 to 5, where 5 is the one with the most details and mostly used for debugging. This is achieved by after `[general]` write `[log_level=number]`.

### Logs and Debugging

```bash
journalctl -u chirpstack-gateway-bridge -f -n 50
```

To track if the gateway bridge is sending packets at mqtt you can use `$mosquitto_sub -h <ipaddress> -t <topic>`. By default the topic that you wanna use is `gateway/#/event/up` which will track all the uplinks sent from the gateway bridge to the network server. To check any kid of package you can just use `gateway/#`. For more information on debugging you can check [here](https://www.chirpstack.io/project/guides/connect-gateway/).


If you wanna track if the gateway is sending packets towards your server, you can use `tcpdump`.


If the ChirpStack Gateway Bridge is installed on the gateway itself, then the following command must be executed on the gateway itself:


```bash
sudo tcpdump -AUq -i lo port 1700
```
If the ChirpStack Gateway Bridge is installed on a server, then you must execute the following command (either on the gateway or on the receiving server):


```bash
sudo tcpdump -AUq port 1700
```

more details can be foun in [here](https://www.chirpstack.io/project/guides/connect-gateway/#tcpdump).


<!------------------------------------ NETWORK SERVER ------------------------------------------------------>
## Network Server

Once the message has been published in the MQTT server, the Chirpstack Network Server (subscribing the topic) will read the message and confirm if it is a duplicate message. The duplication of LoRa frames can happen because a LoRa frame is broadcasted, allowing multiple gateways to read the frame for delivery to the CNS. In this case the CNS will keep the frame from the gateway that has the best connection with the device. The CNS also remembers gateway-device associations for downlink purposes. After dealing with frame duplication it will communicate with the Application Server through google Remote Procedure Call (gRPC).



### Requirements

- Any MQTT broker for instance [Mosquitto](https://mosquitto.org/download/)

```bash
sudo apt-get install mosquitto
```

If you wanna pub or sub to mqtt you will need to install a pub/sub package (optional: used for debugging).

```bash
sudo apt-get install mosquitto-clients
```


- Database: by default uses PSQL.

```
sudo apt install postgresql
```

In the SQL console:

```SQL
-- create role for authentication
create role chirpstack with login password 'chirpstack';

-- create database
create database chirpstack with owner chirpstack;

-- change to chirpstack database
\c chirpstack

-- create pg_trgm extension
create extension pg_trgm;

-- exit psql
\q
```

- Redis

```
sudo apt install redis-server
```

### Instalation

```bash
sudo apt install chirpstack-gateway-bridge
```

### Configurations

The configurations for the gateway bridge will be located at `/etc/chirpstack−network−server/chirpstack−network−server.toml`



First thing it needs to be here is to set up your database in the line `dns=postgres://user:password@hostname/database?sslmode=disable`.


In here you can configure the **delays** related information such as **deduplication_delay** which is the interval for how long the network server waits for duplication packets and **get_downlink_data_delay** which is how long the network server waits to send the downlink after receiving the uplink. 




Can also easily add **ADR plugins** which are alternate versions to the default ADR.
To do that just add the location of your built GO plugin example.

```GO
// Handle handles the ADR and returns the (new) parameters.
func (h *Handler) Handle(req adr.HandleRequest) (adr.HandleResponse, error) {
	resp := adr.HandleResponse{
		DR:           req.DR,
		TxPowerIndex: req.TxPowerIndex,
		NbTrans:      req.NbTrans,
	}

// if ADR is disabled, return current values
	if !req.ADR {
	    return resp, nil
	}
    
// gets info related to the last package received
	up := req.UplinkHistory[len(req.UplinkHistory)-1]

// calculates the nstep value using SNR
	snrM := up.MaxSNR
	snrMargin := snrM - req.RequiredSNRForDR - req.InstallationMargin
	nStep := int(snrMargin / 3)

    
// gets the SF and BW value corresponding to the datarate received
	sf, bw := get_sf_bw_val(req.DR)
	tx := up.TXPowerIndex
    
// dummy data example
// if TXPOWER index is 0 then increase to 2 just for demonstration
	if up.TXPowerIndex > 1 {
// get the values from the solution table generated with RL Agent
		newsf, newtx := RLAGENT(sf, tx, nStep)
		
		log.Info("New SF: ", newsf, " NewTX: ", newtx)

// calculates new datarate based on the new datarate and new bw
		newdr := get_dr_val(newsf, bw)
		
// if new datarate is higher than the max dr of the device,
// it sets the value of the datarate to max datarate
		if newdr > req.MaxDR {
			resp.DR = req.MaxDR
		} else {
			resp.DR = newdr
		}
		resp.TxPowerIndex = newtx
	} else {
		resp.TxPowerIndex = 2
	}

	// resp.DR = 3
	return resp, nil
}   
```

```bash
go build projectdir
```

You can also change the detail of the logs, it goes from 0 to 5, where 5 is the one with the most details and mostly used for debugging. This is achieved by after `[general]` write `[log_level=number]`.

More details about configuration of the network server can be found in [here](https://www.chirpstack.io/network-server/install/config/).

### Logs and Debugging

```bash
journalctl -u chirpstack-network-server -fn 50
```

This allows to check if the Network Server is communicating with the Application Server.



## Application Server

This is the last module of Chirpstack that is responsible for the join-requests and encrypts the payloads of the application. It also provides a web interface for users where they can control users where they can control the features mentioned in the Network Server and a few more. It also offers a restful API and allows sending post whenever it receives information from a device to a user application.




### Requirements

- Any MQTT broker for instance [Mosquitto](https://mosquitto.org/download/)

```bash
sudo apt-get install mosquitto
```

If you wanna pub or sub to mqtt you will need to install a pub/sub package (optional: used for debugging).

```bash
sudo apt-get install mosquitto-clients
```


- Database: by default uses PSQL.

```
sudo apt install postgresql
```

In the SQL console:

```SQL
sudo -u postgres psql

-- create the chirpstack_ns user with password 'dbpassword'
create role chirpstack_ns with login password 'dbpassword';

-- create the chirpstack_ns database
create database chirpstack_ns with owner chirpstack_ns;

-- exit the prompt
\q
```

- Redis

```
sudo apt install redis-server
```

### Instalation

```bash
sudo apt install chirpstack-application-server
```

### Configurations
The settings in this section are more related to association with other applications like Influxdb, AWS, etc.
The App Server provides an API port, which is set by default to port 8003, and the API provides information available on the Network Server, which was mentioned earlier, for example subscribing to the frames received by a certain device as long as it has permission to do so.
    
There are not many settings in this section. This is due to the fact that the Application Serv\er mostly works as a server with a web interface that allows you to easily manage the Network Server, allowing you to create gateways, devices, profiles and all other content mentioned in the Network Server.

It also offers the possibility to integrate the Chirpstack with other applications, depending on your needs. To do so, different steps have to be followed depending on the application. It is in this process that you can integrate an application created by the user, and for that, you just have to select the ip:port of the app's server.

### Debugging and Logging

```
journalctl -u chirpstack-application-server -fn 50
```

This allows to check if the Application Server is successfully sending and receiving packets. 


## API

The API allows to easily control chirpstack Network Server using your own Application and it is quite simple to use. Only requires the user to create a key in the application server. The key is a JWT Token that is originated by the key in the Application Server config file in conjuction with the user account information.

More information in [here](https://www.chirpstack.io/application-server/api/).


JWT Token: https://jwt.io/

Http Examples: https://www.chirpstack.io/application-server/api/go-examples/

Enqueue Downlink Example with curl:

```bash
curl -X POST --header 'Content-Type: application/json' --header 'Accept: application/json' --header 'Grpc-Metadata-Authorization: Bearer <API TOKEN>' -d '{ \ 
   "deviceQueueItem": { \ 
     "confirmed": false, \ 
     "data": "AQID", \ 
     "fPort": 10 \ 
   } \ 
 }' 'http://localhost:8080/api/devices/0101010101010101/queue'
```




<script src="./../_static/js/required.js"></script> 


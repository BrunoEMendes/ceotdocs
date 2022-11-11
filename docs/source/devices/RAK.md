# Gateway RAK 7244 

The internet connection setup is easy to make, it can be easily done by following the setup on [rakwireless website](https://docs.rakwireless.com/Product-Categories/WisGate/RAK7244/Quickstart/#accessing-your-gateway).


After the setup the only required setting that needs to be done is to reroute your packets towards your server. For that you simply need to:
1) connect to the gateway using ssh or any other mechanism
2) use the command `$sudo gateway-config`
3) select 'Setup RAK Gateway Channel Plan' and pick which service you are using (Chirpstack/loraserver or TTN/TTI).


Do not forget to turn off the Chirpstack services in the gateway to ease the processor work, they come installed and active by default.
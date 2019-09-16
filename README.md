# eduroam-ng
Installation and setup for [eduroam](https://www.eduroam.org) including [eduroam CAT](https://cat.eduroam.org) and [Switchboard](https://switchboard.eduroam.africa) configuration

## Setup eduroam using Freeradius
Install `freeradius` with `mysql` (or `postgresql`) support. Choose any as the case may be
```
  sudo apt update
  sudo apt install freeradius freeradius-mysql
  sudo apt install freeradius freeradius-postgresql
```

Create local users in the users file for testing purposes. Open the users file with your preferred editor (`vi` or `nano`) and add the text below at the top of the file (preferably replacing the bob line)
```
sudo nano /etc/freeradius/3.0/users
```

```
eduroam    Cleartext-Password := "eduroampass"
       Reply-Message := "Hello, %{User-Name}"
```
Now restart the freeradius service and test the newly created user. You should get an accept reply.
```
sudo systemctl restart freeradius.service
radtest eduroam eduroampass localhost 0 testing123
```
Sometimes, we may want to reject requests immediately instead, then set reject_delay = 0 in radiusd.conf file
sudo nano /etc/freeradius/3.0/radiusd.conf
```
  reject_delay = 0
```
## Setup eduroam server
Now delete all the files in sites-enabled folder. Create two files called eduroam and eduroam-inner-tunnel and paste the contents below to the corresponding files
```
sudo rm /etc/freeradius/3.0/sites-enabled/*
sudo touch /etc/freeradius/3.0/sites-enabled/eduroam
sudo nano /etc/freeradius/3.0/sites-enabled/eduroam
```
Copy the contents of [eduroam](https://github.com/ezeasorekene/eduroam-ng/blob/master/sites-enabled/eduroam) file in `sites-enabled` to the new file created
```
sudo touch /etc/freeradius/3.0/sites-enabled/eduroam-inner-tunnel
sudo nano /etc/freeradius/3.0/sites-enabled/eduroam-inner-tunnel
```
Copy the contents of [eduroam-inner-tunnel](https://github.com/ezeasorekene/eduroam-ng/blob/master/sites-enabled/eduroam-inner-tunnel) file in `sites-enabled` to the new file created


In some cases, `attr_filter.pre-proxy` removes by default `Operator-Name` and `Calling-Station-Id`, which are two very useful attributes for debugging and analytics. We ensure they don’t get removed by editing `pre-proxy` file. Open the `pre-proxy` file and add the following to the end of the file.
```
sudo nano /etc/freeradius/3.0/mods-config/attr_filter/pre-proxy
```
Append the [contents located here](https://github.com/ezeasorekene/eduroam-ng/blob/master/others/pre-proxy) towards the end of the file or make appropriate changes

## Configure proxy.conf
Let us now configure our RADIUS server to forward all users it doesn’t recognize to the federation Proxy Server for Nigeria (or any other African country). It will now serve as a Service Provider.
```
sudo mv /etc/freeradius/3.0/proxy.conf /etc/freeradius/3.0/proxy.conf.original
sudo touch /etc/freeradius/3.0/proxy.conf
sudo nano /etc/freeradius/3.0/proxy.conf
```
Generate the `proxy.conf` configuration file from the [Switchboard](https://switchboard.eduroam.africa)

*The contents of `proxy.conf` generated from the Switchboard typically looks like [this file](https://github.com/ezeasorekene/eduroam-ng/blob/master/switchboard/proxy.conf). Some unique values were stripped.*

## Configure clients
Now let us configure the clients. Again the main contents of the file should be generated and downloaded from the Switchboard. Rename, create and open the `clients.conf` file and add the generated contents.
```
sudo mv /etc/freeradius/3.0/clients.conf /etc/freeradius/3.0/clients.conf.original
sudo touch /etc/freeradius/3.0/clients.conf
sudo nano /etc/freeradius/3.0/clients.conf
```
*The contents of `proxy.conf` generated from the Switchboard typically looks like [this file](https://github.com/ezeasorekene/eduroam-ng/blob/master/switchboard/clients.conf). Some unique values were stripped.*

## Configure certificate
Let us now make the certificates used by `eduroam`. Open the `ca.cnf` file and edit the file to correspond as specified
```
sudo /etc/freeradius/3.0/certs/ca.cnf
```
**Edit the file as appropriate**
```
default_days            = 3650

[certificate_authority]
countryName             		= NG
stateOrProvinceName     	= Your State
localityName            		= Your City
organizationName        	= Your Institution
emailAddress            		= support@yourdomain.edu.ng
commonName              	= "Your Institution Certificate Authority"
```
Now build the `ca.pem` and `ca.der` files
```
cd /etc/freeradius/3.0/certs/
sudo make ca.pem
sudo make ca.der
```
Edit the `server.cnf` file to correspond with the values edited in `ca.cnf` as shown below
```
sudo nano /etc/freeradius/3.0/certs/server.cnf
```
```
default_days            = 3650

[server]
countryName             		= NG
stateOrProvinceName     	= Your State
localityName            		= Your City
organizationName        	= Your Institution
emailAddress            		= support@yourdomain.edu.ng
commonName              	= "Your Organization eduroam Server Certificate"
```
Then build the `server.pem`
```
cd /etc/freeradius/3.0/certs/
sudo make server.pem
```
We need to now assign the correct file ownership
```
sudo chown freerad:freerad ca.*
sudo chown freerad:freerad server.*
sudo chown freerad:freerad index.*
```
Let us configure the EAP by editing the contents of the `eap` file to correspond as [shown in this file](https://github.com/ezeasorekene/eduroam-ng/blob/master/mods-enabled/eap). Here we are telling the eap file where our certificate is located and setting different configuration to use.
```
sudo rm /etc/freeradius/3.0/mods-enabled/eap
sudo touch /etc/freeradius/3.0/mods-enabled/eap
sudo nano /etc/freeradius/3.0/mods-enabled/eap
```
## Setup testing with rad_eap_test
Let us now setup testing with rad_eap_test
```
sudo apt install libnl-genl-3-200 libpcsclite1
cd ~/
sudo apt install git
git clone https://github.com/CESNET/rad_eap_test.git
cd rad_eap_test
sudo cp bin/eapol_test /usr/local/bin/
```
Let us now run any of the following tests using any of the users specified in the users file
```
./rad_eap_test -H localhost -P 1812 -S testing123 -u eduroam -p eduroampass -m WPA-EAP -e TTLS
./rad_eap_test -H localhost -P 1812 -S testing123 -u eduroam -p eduroampass -m WPA-EAP -e PEAP
./rad_eap_test -H localhost -P 1812 -S testing123 -u eduroam -p eduroampass -m WPA-EAP -e TTLS -2 PAP
./rad_eap_test -H localhost -P 1812 -S testing123 -u eduroam -p eduroampass -m WPA-EAP -e TTLS -2 MSCHAPV2
./rad_eap_test -H localhost -P 1812 -S testing123 -u eduroam -p eduroampass -m WPA-EAP -e PEAP -2 PAP
./rad_eap_test -H localhost -P 1812 -S testing123 -u eduroam -p eduroampass -m WPA-EAP -e PEAP -2 MSCHAPV2
```

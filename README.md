# eduroam-ng
Installation and setup for eduroam (Africa) including eduroam CAT and Switchboard configuration

## How to setup eduroam using Freeradius

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

Now delete all the files in sites-enabled folder. Create two files called eduroam and eduroam-inner-tunnel and paste the contents below to the corresponding files
```
sudo rm /etc/freeradius/3.0/sites-enabled/*
sudo touch /etc/freeradius/3.0/sites-enabled/eduroam
sudo nano /etc/freeradius/3.0/sites-enabled/eduroam
```
Copy the contents of [eduroam](https://github.com/ezeasorekene/eduroam-ng/sites-enabled/eduroam) file in `sites-enabled` to the new file created
```
sudo touch /etc/freeradius/3.0/sites-enabled/eduroam-inner-tunnel
sudo nano /etc/freeradius/3.0/sites-enabled/eduroam-inner-tunnel
```
Copy the contents of [eduroam-inner-tunnel](https://github.com/ezeasorekene/eduroam-ng/sites-enabled/eduroam-inner-tunnel) file in `sites-enabled` to the new file created


In some cases, attr_filter.pre-proxy removes by default Operator-Name and Calling-Station-Id, which are two very useful attributes for debugging and analytics. We ensure they don’t get removed by editing  pre-proxy file. Open the pre-proxy file and add the following to the end of the file.
sudo nano /etc/freeradius/3.0/mods-config/attr_filter/pre-proxy

*** Append the following contents towards the end of the file or make appropriate changes ***

DEFAULT
		User-Name =* ANY,
	User-Password =* ANY,
	CHAP-Password =* ANY,
	CHAP-Challenge =* ANY,
	MS-CHAP-Challenge =* ANY,
	MS-CHAP-Response =* ANY,
	EAP-Message =* ANY,
	Message-Authenticator =* ANY,
	State =* ANY,
	NAS-IP-Address =* ANY,
	NAS-Identifier =* ANY,
	Operator-Name =* ANY,
	Calling-Station-Id =* ANY,
	Chargeable-User-Identity =* ANY,
	Proxy-State =* ANY
	Called-Station-Id =* ANY,
	Eduroam-SP-Country =* ANY,
	Operator-Name =* ANY

Let us now configure our RADIUS server to forward all users it doesn’t recognize to the federation Proxy Server for Nigeria. It will now serve as a Service Provider.
sudo mv /etc/freeradius/3.0/proxy.conf /etc/freeradius/3.0/proxy.conf.original
sudo touch /etc/freeradius/3.0/proxy.conf
sudo nano /etc/freeradius/3.0/proxy.conf

*** The contents of this file should ordinarily be generated from the Switchboard, but it typically looks like below. Some unique values were stripped. ***

#######################################################################
#
#  Proxy server configuration for NAME-OF-YOUR-ORGANIZATION
#  Generated on 2019-04-02 11:56:40 UTC
#  This entry controls the servers behaviour towards ALL other servers
#  to which it sends proxy requests.
#
proxy server {
  default_fallback = no
}

# Blackhole Routing - EAP-SIM/MNOs
realm "~\.3gppnetwork\.org$" {
  nostrip
  notrealm
}

# Your local realms - leaving them blank stops them from ever being forwarded
realm LOCAL {
}
realm NULL {
}

# Actual realms, including subdomains if applicable
realm yourdomain.edu.ng {
}
realm "~.+\.yourdomain\.edu\.ng$" {
}

############################################################
##                Switchboard/Monitor                     ##
############################################################
### Switchboard ###
home_server switchboard.ip4 {
  ipaddr                = ip-address
  proto                 = udp
  type                  = auth
  port                  = 1812
  secret                = password
  response_window       = 20
  response_timeouts     = 2
}

home_server_pool switchboard {
  type        = client-balance
  home_server = switchboard.ip4
}

realm 1ef40719-4d45-46a7-b237-0bb9e5725330.ng {
  auth_pool = switchboard
  nostrip
}


############################################################
##                        Upstream                        ##
############################################################
# Servers
home_server flr2.ngren.edu.ng.ip4 {
  ipaddr                = ip-address
  proto                 = udp
  type                  = auth+acct
  port                  = 1812,1813
  secret                = password
  response_window       = 20
  response_timeouts     = 2
  zombie_period         = 60
  status_check          = status-server
  check_interval        = 30
  check_timeout         = 6
  num_answers_to_alive  = 6
  max_outstanding       = 65536
}

home_server flr1.ngren.edu.ng.ip4 {
  ipaddr                = ip-address
  proto                 = udp
  type                  = auth+acct
  port                  = 1812,1813
  secret                = password
  response_window       = 20
  response_timeouts     = 2
  zombie_period         = 60
  status_check          = status-server
  check_interval        = 30
  check_timeout         = 6
  num_answers_to_alive  = 6
  max_outstanding       = 65536
}

home_server flr1.ngren.edu.ng.ip6 {
  ipaddr                = ipv6-address
  proto                 = udp
  type                  = auth+acct
  port                  = 1812,1813
  secret                = password
  response_window       = 20
  response_timeouts     = 2
  zombie_period         = 60
  status_check          = status-server
  check_interval        = 30
  check_timeout         = 6
  num_answers_to_alive  = 6
  max_outstanding       = 65536
}

# Pools
home_server_pool upstream {
  type        = client-balance
  home_server = flr2.ngren.edu.ng.ip4
  home_server = flr1.ngren.edu.ng.ip4
  home_server = flr1.ngren.edu.ng.ip6
}

# Default destination for unknown realms - forward to the upstream servers
# Regex version is required for f_ticks to log properly
realm "~.+$" {
  pool = upstream
  nostrip
}
realm DEFAULT {
  pool = upstream
  nostrip
}


Now let us configure the clients. Again the main contents of the file should be generated and downloaded from the Switchboard. Rename, create and open the clients.conf file and add the following contents.
sudo mv /etc/freeradius/3.0/clients.conf /etc/freeradius/3.0/clients.conf.original
sudo touch /etc/freeradius/3.0/clients.conf
sudo nano /etc/freeradius/3.0/clients.conf

*** Just like I explained earlier, the contents of this file should ordinarily be generated from the Switchboard, but it typically looks like below. Some unique values were stripped. ***

### Institutional Equipment/NAS - APs, Switches, Controllers ###
client adminblock.ip4 {
  ipv4addr                      = ip-address
  proto                         = udp
  secret                        = password
  require_message_authenticator = yes
  nas_type                      = other
}


### Localhost ###
client localhost.ip4 {
  ipv4addr                      = 127.0.0.1/32
  proto                         = udp
  secret                        = password
  require_message_authenticator = yes
  nas_type                      = other
}

client localhost.ip6 {
  ipv6addr                      = ::1/128
  proto                         = udp
  secret                        = password
  require_message_authenticator = yes
  nas_type                      = other
}
client localhost.public.ip4 {
  ipv4addr                      = ip-address
  proto                         = udp
  secret                        = password
  require_message_authenticator = yes
  nas_type                      = other
}

### Upstream ###
client flr2.ngren.edu.ng.ip4 {
  ipv4addr                      = ip-address
  proto                         = udp
  secret                        = password
  require_message_authenticator = yes
  nas_type                      = other
  shortname                     = ng-flr_flr2.ngren.edu.ng.ip4
}

client flr1.ngren.edu.ng.ip4 {
  ipv4addr                      = ip-address
  proto                         = udp
  secret                        = password
  require_message_authenticator = yes
  nas_type                      = other
  shortname                     = ng-flr_flr1.ngren.edu.ng.ip4
}

client flr1.ngren.edu.ng.ip6 {
  ipv6addr                      = ipv6-address
  proto                         = udp
  secret                        = password
  require_message_authenticator = yes
  nas_type                      = other
  shortname                     = ng-flr_flr1.ngren.edu.ng.ip4
}


### eduroam Switchboard ###
client switchboard.ip4 {
  ipv4addr                      = ip-address
  proto                         = udp
  secret                        = password
  require_message_authenticator = yes
  nas_type                      = other
}

### eduroam Monitoring ###
client monitoring.ip4 {
  ipv4addr                      = ip-address
  proto                         = udp
  secret                        = password
  require_message_authenticator = yes
  nas_type                      = other
}

Let us now make the certificates used by eduroam. Open the ca.cnf file and edit the file to correspond as specified
sudo /etc/freeradius/3.0/certs/ca.cnf

*** Edit the file as appropriate ***

default_days            = 3650

[certificate_authority]
countryName             		= NG
stateOrProvinceName     	= Your State
localityName            		= Your City
organizationName        	= Your Institution
emailAddress            		= support@yourdomain.edu.ng
commonName              	= "Your Institution Certificate Authority"

*** Now build the ca.pem and ca.der files ***

cd /etc/freeradius/3.0/certs/
sudo make ca.pem
sudo make ca.der

*** Edit the server.cnf file to correspond with the values edited in ca.cnf as shown below ***

sudo nano /etc/freeradius/3.0/certs/server.cnf

default_days            = 3650

[server]
countryName             		= NG
stateOrProvinceName     	= Your State
localityName            		= Your City
organizationName        	= Your Institution
emailAddress            		= support@yourdomain.edu.ng
commonName              	= "Your Organization eduroam Server Certificate"

*** Then build the server.pem ***

cd /etc/freeradius/3.0/certs/
sudo make server.pem

*** We need to now assign the correct file ownership ***

sudo chown freerad:freerad ca.*
sudo chown freerad:freerad server.*
sudo chown freerad:freerad index.*

Let us configure the EAP by editing the contents of the eap file to correspond as shown. Here we are telling the eap file where our certificate is located and setting different configuration to use.
sudo rm /etc/freeradius/3.0/mods-enabled/eap
sudo touch /etc/freeradius/3.0/mods-enabled/eap
sudo nano /etc/freeradius/3.0/mods-enabled/eap

*** We need to now assign the correct file ownership ***

eap {
	default_eap_type = peap
	timer_expire     = 60
	ignore_unknown_eap_types = no
	cisco_accounting_username_bug = no
	max_sessions = ${max_requests}

	md5 {
	}

	leap {
	}

	gtc {
		auth_type = PAP
	}

	tls-config tls-common {
		private_key_password = whatever
		private_key_file = ${certdir}/server.key
		certificate_file = ${certdir}/server.pem
		ca_file = /etc/ssl/certs/ca-certificates.crt

		dh_file = ${certdir}/dh

		ca_path = ${cadir}

		cipher_list = "DEFAULT"

		cipher_server_preference = no

		ecdh_curve = "prime256v1"

		cache {
			enable = no

			lifetime = 24 # hours

		}

		verify {
		}

		ocsp {
			enable = no

			override_cert_url = yes

			url = "http://127.0.0.1/ocsp/"

		}
	}

	tls {
		tls = tls-common

	}

	ttls {
		tls = tls-common

		default_eap_type = mschapv2
		copy_request_to_tunnel = yes
		use_tunneled_reply = no
		virtual_server = "eduroam-inner-tunnel"
	}

	peap {
		tls = tls-common
		default_eap_type = mschapv2
		copy_request_to_tunnel = yes
		use_tunneled_reply = no
		virtual_server = "eduroam-inner-tunnel"
	}
	mschapv2 {
	}
}

Let us now setup testing with rad_eap_test
sudo apt install libnl-genl-3-200 libpcsclite1
cd ~/
sudo apt install git
git clone https://github.com/CESNET/rad_eap_test.git
cd rad_eap_test
sudo cp bin/eapol_test /usr/local/bin/

*** Let us now run any of the following tests using any of the users specified in the users file ***

./rad_eap_test -H localhost -P 1812 -S testing123 -u eduroam -p eduroampass -m WPA-EAP -e TTLS
./rad_eap_test -H localhost -P 1812 -S testing123 -u eduroam -p eduroampass -m WPA-EAP -e PEAP
./rad_eap_test -H localhost -P 1812 -S testing123 -u eduroam -p eduroampass -m WPA-EAP -e TTLS -2 PAP
./rad_eap_test -H localhost -P 1812 -S testing123 -u eduroam -p eduroampass -m WPA-EAP -e TTLS -2 MSCHAPV2
./rad_eap_test -H localhost -P 1812 -S testing123 -u eduroam -p eduroampass -m WPA-EAP -e PEAP -2 PAP
./rad_eap_test -H localhost -P 1812 -S testing123 -u eduroam -p eduroampass -m WPA-EAP -e PEAP -2 MSCHAPV2

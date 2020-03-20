# Kubernetes-strongswan

## How we can manage vpn in kubernetes environment

Hi there , this project is to cover the vpn ipsec-xauth topic in a kubernetes evironment,   
the goal of this is to have the less effort possible when we have to manage users.  

Architecture   
![architecture](https://res.cloudinary.com/ethzero/image/upload/v1584731735/misc/vpn_diagram.jpg)  


Requirements:  
- Kubernetes
- Strongswan
- Microsoft Acrive Directory / openldap / freeipa etc etc LDAP (i'll use ldap instead the software name)

## WHY  
The traditional ipsec-xauth vpn with ikev1 is based on PSK  
and a client username/password , this is a problem when the credential are stored in a file  
in kubernetes update a file always mean rollout a new deploy  or create a procedure to  
make effective the changes.  
So the idea is to deploy something that doesn't need any interaction  
after the deploy and manage the clients, with the company standards,  
like password expiration, password complexity, groups attributions and so on.  




## HOW  
In order to have a fully managed services we can leverage the usage of ldap procedures (that all company has).  
Strongswan (a fork of \*swan ipsec software) could be integrated with ldap with pam.  
pam is ... well --> https://tldp.org/HOWTO/User-Authentication-HOWTO/x115.html  

So what we need in ldap ?  
We need:
- a technical user that is a low level profile that will be used only to check the users inside the ldap tree and the groups associated
- a group to associate to people who need/granted the vpn access  

in this scenario
tech users is --> *ldapbind*  
group is      --> *vpn*

here some screen related  

![ldapbind](https://res.cloudinary.com/ethzero/image/upload/c_scale,w_640/v1584730828/misc/strongswan_bind_user.png)  

![vpn group](https://res.cloudinary.com/ethzero/image/upload/c_scale,w_640/v1584730828/misc/strongswan_user_vpn_group.png)


Then... now we have to configure configure the Docker image in order to
support pam ldap.  

Dockerfile  
```
FROM debian:stretch
MAINTAINER lgirardi <l@k8s.it>


RUN apt-get -y update && apt-get -yq install \
        strongswan \
        libcharon-extra-plugins \
        iptables \
        kmod \
        libpam-ldap \
        vim

EXPOSE 500/udp 4500/udp

CMD /usr/sbin/ipsec start --nofork
```

libpam-ldap and libcharon-extra-plugins are what we need to perform this kind of integration.  

Since strongswan is not traditionally used in kubernetes , has some files that needs a configuration.  
ENV variables are the most useful to configure it,  
unfortunately the process is not able to share the env this the child process,  
so we will work with 2 concepts,  
use the configmap for all files we need to configure
use the secrets for all sensitive data we need to add

files configured:
- ipsec.conf (the strongswan main configuration)
- xauth-pam.conf (strongswan configuration to enable pam)
- attr.conf (strongswan configuration file for split-tunnel)  
*split-tunnel is when you want to move in vpn only the company subnet and use the home gateway for all the other usages*
- ipsec (pam configuration in /etc/pam.d)  

secrets:
- ipsec.secrets (file with the ipsec PSK) rif. 003-configmap.yaml   
- pam_ldap.conf (configuration used by pam module to connect to ldap) rif. 002-secrets.yaml  

*remember that all secrets files are managed using base64 encoding*  


When we have multiple files to spread in different locations we have to create some tricks,  
one is to create symlink in the Dockerfile , however we have to keep the configuration  
as much as possible agnostic from the Dockerfile.  



*volume* and *volumeMounts* can help on this topic  

```
volumeMounts:
- name: psk
  mountPath: /etc/ipsec.secrets
  subPath: psk
  readOnly: true
- name: pamldap
  mountPath: /etc/pam_ldap.conf
  subPath: pamldap
- name: strongswan-attr
  mountPath: /etc/strongswan.d/charon/attr.conf
  subPath: attr.conf
- name: strongswan-xauth-pam
  mountPath: /etc/strongswan.d/charon/xauth-pam.conf
  subPath: xauth-pam.conf
- name: strongswan-ipsec
  mountPath: /etc/pam.d/ipsec
  subPath: ipsec
- name: strongswan-ipseconf
  mountPath: /etc/ipsec.conf
  subPath: ipsec.conf
volumes:
- name: strongswan-attr
  configMap:
    name: strongswanconfigmap
    items:
    - key: attr.conf
      path: attr.conf
- name: strongswan-xauth-pam
  configMap:
    name: strongswanconfigmap
    items:
    - key: xauth-pam.conf
      path: xauth-pam.conf
- name: strongswan-ipsec
  configMap:
    name: strongswanconfigmap
    items:
    - key: ipsec
      path: ipsec
- name: strongswan-ipseconf
  configMap:
    name: strongswanconfigmap
    items:
    - key: ipsec.conf
      path: ipsec.conf
- name: psk
  secret:
    secretName: strongswan-secret
- name: pamldap
  secret:
    secretName: strongswan-secret
```


Now we have all configured, we can just run  
```kubectl apply -f deploy```

We will have soon a pod into strongswan namespace

```
# kubectl get pods -n strongswan
NAME                          READY   STATUS    RESTARTS   AGE
strongswan-77bfbb9f9f-57hmz   1/1     Running   0          22h
```

Since the service is configured with nodport we need to enable the default 500 and 4500  
in our firewall , matching the kubernetes ports 30000-32767,
in this this service are

```
ports:
- name: isakmp-udp
  protocol: UDP
  nodePort: 30500
  port: 500
  targetPort: 500
- name: ipsec-nat-t
  protocol: UDP
  nodePort: 30450
  port: 4500
  targetPort: 4500
type: NodePort
```
![firewall configuration](https://res.cloudinary.com/ethzero/image/upload/c_scale,w_640/v1584730827/misc/strongswan_firewall_nat.png)


We can configure our standard client (cisco ipsec client is enough)

![android configuration](https://res.cloudinary.com/ethzero/image/upload/c_scale,w_320/v1584735493/misc/strongswan_android_client.png)




```
# kubectl logs strongswan-77bfbb9f9f-57hmz -n strongswan
Starting strongSwan 5.5.1 IPsec [starter]...
no netkey IPsec stack detected
no KLIPS IPsec stack detected
no known IPsec stack detected, ignoring!
charon (13) started after 80 ms
00[DMN] Starting IKE charon daemon (strongSwan 5.5.1, Linux 4.15.0-70-generic, x86_64)
00[CFG] mapping attribute type split-exclpude failed
00[CFG] loading ca certificates from '/etc/ipsec.d/cacerts'
00[CFG] loading aa certificates from '/etc/ipsec.d/aacerts'
00[CFG] loading ocsp signer certificates from '/etc/ipsec.d/ocspcerts'
00[CFG] loading attribute certificates from '/etc/ipsec.d/acerts'
00[CFG] loading crls from '/etc/ipsec.d/crls'
00[CFG] loading secrets from '/etc/ipsec.secrets'
00[CFG]   loaded IKE secret for %any
00[CFG] loaded 0 RADIUS server configurations
00[CFG] HA config misses local/remote address
00[LIB] loaded plugins: charon aes rc2 sha2 sha1 md5 random nonce x509 revocation constraints pubkey pkcs1 pkcs7 pkcs8 pkcs12 pgp dnskey sshkey pem openssl fips-prf gmp agent xcbc hmac gcm attr kernel-netlink resolve socket-default connmark farp stroke updown eap-identity eap-aka eap-md5 eap-gtc eap-mschapv2 eap-radius eap-tls eap-ttls eap-tnc xauth-generic xauth-eap xauth-pam tnc-tnccs dhcp lookip error-notify certexpire led addrblock unity
00[LIB] dropped capabilities, running as uid 0, gid 0
00[JOB] spawning 16 worker threads
05[CFG] received stroke: add connection 'roadw'
05[CFG] adding virtual IP address pool 172.16.17.0/29
05[CFG] added configuration 'roadw'
08[NET] received packet: from 10.1.1.1[36312] to 10.1.1.84[500] (756 bytes)
08[ENC] parsed ID_PROT request 0 [ SA V V V V V V V V ]
08[IKE] received NAT-T (RFC 3947) vendor ID
08[IKE] received draft-ietf-ipsec-nat-t-ike-02 vendor ID
08[IKE] received draft-ietf-ipsec-nat-t-ike-02\n vendor ID
08[IKE] received draft-ietf-ipsec-nat-t-ike-00 vendor ID
08[IKE] received XAuth vendor ID
08[IKE] received Cisco Unity vendor ID
08[IKE] received FRAGMENTATION vendor ID
08[IKE] received DPD vendor ID
08[IKE] 10.1.1.1 is initiating a Main Mode IKE_SA
08[ENC] generating ID_PROT response 0 [ SA V V V V ]
08[NET] sending packet: from 10.1.1.84[500] to 10.1.1.1[36312] (160 bytes)
05[NET] received packet: from 10.1.1.1[36312] to 10.1.1.84[500] (228 bytes)
05[ENC] parsed ID_PROT request 0 [ KE No NAT-D NAT-D ]
05[IKE] local host is behind NAT, sending keep alives
05[IKE] remote host is behind NAT
05[ENC] generating ID_PROT response 0 [ KE No NAT-D NAT-D ]
05[NET] sending packet: from 10.1.1.84[500] to 10.1.1.1[36312] (244 bytes)
09[NET] received packet: from 10.1.1.1[40011] to 10.1.1.84[4500] (92 bytes)
09[ENC] parsed ID_PROT request 0 [ ID HASH ]
09[CFG] looking for XAuthInitPSK peer configs matching 10.1.1.84...10.1.1.1[100.106.113.62]
09[CFG] selected peer config "roadw"
09[ENC] generating ID_PROT response 0 [ ID HASH ]
09[NET] sending packet: from 10.1.1.84[4500] to 10.1.1.1[40011] (92 bytes)
09[ENC] generating TRANSACTION request 3276308191 [ HASH CPRQ(X_USER X_PWD) ]
09[NET] sending packet: from 10.1.1.84[4500] to 10.1.1.1[40011] (76 bytes)
11[NET] received packet: from 10.1.1.1[40011] to 10.1.1.84[4500] (108 bytes)
11[ENC] parsed TRANSACTION response 3276308191 [ HASH CPRP(X_USER X_PWD) ]
11[IKE] PAM authentication of 'lgirardi' successful
11[IKE] XAuth authentication of 'lgirardi' successful
11[ENC] generating TRANSACTION request 1380277626 [ HASH CPS(X_STATUS) ]
11[NET] sending packet: from 10.1.1.84[4500] to 10.1.1.1[40011] (76 bytes)
10[NET] received packet: from 10.1.1.1[40011] to 10.1.1.84[4500] (108 bytes)
10[ENC] parsed INFORMATIONAL_V1 request 4006980307 [ HASH N(INITIAL_CONTACT) ]
12[NET] received packet: from 10.1.1.1[40011] to 10.1.1.84[4500] (92 bytes)
12[ENC] parsed TRANSACTION response 1380277626 [ HASH CPA(X_STATUS) ]
12[IKE] IKE_SA roadw[1] established between 10.1.1.84[ETHZERO_HOME_VPN]...10.1.1.1[100.106.113.62]
12[IKE] scheduling rekeying in 86047s
12[IKE] maximum IKE_SA lifetime 86227s
```

ok now i'm connected and i can see my network

![android ip](https://res.cloudinary.com/ethzero/image/upload/c_scale,w_320/v1584730960/misc/strongswan_client_android.jpg)  

![android ping](https://res.cloudinary.com/ethzero/image/upload/c_scale,w_320/v1584736034/misc/strongswan_android_ping.jpg)   


Thats all ... here my connection

![strongswan connection](https://res.cloudinary.com/ethzero/image/upload/c_scale,w_640/v1584736328/misc/strongswan_stroke_statusall.png)

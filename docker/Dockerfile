FROM debian:stretch
MAINTAINER lgirardi <l@k8s.it>

RUN apt-get -y update && apt-get -y install \
	strongswan \
	iptables \
	kmod \
	vim 

EXPOSE 500/udp 4500/udp

CMD /usr/sbin/ipsec start --nofork

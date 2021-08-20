FROM debian:11
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

sudo docker image build -t strongswan:02 .
sudo docker run --cap-add NET_ADMIN --net=host -i -t strongswan:02 /bin/bash


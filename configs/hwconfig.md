# Hardware config

## Network 

Tinkerbell needs MAC in lowercase!

| IP              | name                 | remarks               |
| --------------- | ----------------------- | --------------------- |
| 192.168.48.0/20 | lab net                 | 4094 might be enough  |
| 192.168.48.1    | switch port 1           | dc:2c:6e:ad:fb:40, default gateway       |
| 192.168.48.10   | h2 - provision server   | hw interface 00:1e:06:45:12:99     |
| 192.168.49.0/24 | h2 - clusternet         | docker macvlan - see below  |
| 192.168.49.1    | h2 - clusternet shim    | ip to reach clusternet from h2     |
| 192.168.49.11   | node-01                 | 00:1e:06:45:0d:48     | 
| 192.168.49.12   | node-02                 | 00:1e:06:45:14:78     | 
| 192.168.49.13   | node-03                 | 00:1e:06:45:01:1e     | 

## docker macvlan

```
docker network create -d macvlan --subnet=192.168.48.0/20 \
  --ip-range=192.168.49.0/24 --aux-address 'host=192.168.49.1' \
  --gateway=192.168.48.1 -o parent=enp2s0 clusternet
``` 

## switch

UI forward `ssh aleks@glasfish -L 8080:192.168.48.1:80`



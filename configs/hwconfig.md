# Hardware config

## Network 

Tinkerbell needs MAC in lowercase!

| IP              | name                 | remarks               |
| --------------- | ----------------------- | --------------------- |
| 192.168.48.0/20 | lab net                 | 4094 might be enough  |
| 192.168.48.1    | switch port 1           | dc:2c:6e:ad:fb:40, default gateway       |
| 192.168.48.2    | master - provision server   | hw interface 00:1e:06:45:16:9d     |
| 192.168.48.11   | node-01                 | 00:1e:06:45:0d:48     | 
| 192.168.48.12   | node-02                 | 00:1e:06:45:14:78     | 
| 192.168.48.13   | node-03                 | 00:1e:06:45:01:1e     | 
| 192.168.48.14   | node-04                 | 00:1e:06:45:01:1e     | 
| 192.168.48.15   | node-05                 | 00:1e:06:45:01:1e     | 
| 192.168.48.16   | node-06                 | 00:1e:06:45:01:1e     | 

## switch

UI forward `ssh aleks@glasfish -L 8080:192.168.48.1:80`



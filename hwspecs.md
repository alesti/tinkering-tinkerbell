# Hardware specs

## Network 

Tinkerbell needs MAC in lowercase!

| IP              | name                 | remarks               |
| --------------- | -------------------- | --------------------- |
| 192.168.48.0/20 | lab net              | 1022 might be enough  |
| 192.168.48.1    | switch port 1        | dc:2c:6e:ad:fb:40, default gateway       |
| 192.168.48.10   | provision server     | 00:1e:06:45:12:99     |
| 192.168.48.11   | node1                | 00:1e:06:45:0d:48     | 
| 192.168.48.12   | node2                | 00:1e:06:45:14:78     | 
| 192.168.48.13   | node3                | 00:1e:06:45:01:1e     | 


## switch

UI forward `ssh aleks@glasfish -L 8080:192.168.48.1:80`



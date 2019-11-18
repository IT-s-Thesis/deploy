
## Set up swarm mode

Trong Docker Swarm (Swarm mode) co the tao nhieu node may chu vat ly la cac may `manager` or `worker` ket noi voi nhau tao thanh cac cluster. Thuong cac may `manager` co nhiem vu cai dat giao nhiem vu; con cac may `worker` co nhieu vu thuc thi cong viec duoc giao.

Dau tien, can tao cac may chu ao. => su dung docker-machine.
Gia su, co 1 may `manager` va 3 may `worker`
```bash
# tao may manager
> docker-machine create manager

# tao may worker
> docker-machine create worker1
> docker-machine create worker2
> docker-machine create worker3
```
Kiem tra lai cac thong tin cua cai may chu da tao:
```bash
> docker-machine ls
NAME      ACTIVE   DRIVER       STATE     URL                         SWARM   DOCKER     ERRORS
manager   -        virtualbox   Running   tcp://192.168.99.100:2376           v18.09.0   
worker1   -        virtualbox   Running   tcp://192.168.99.101:2376           v18.09.0   
worker2   -        virtualbox   Running   tcp://192.168.99.102:2376           v18.09.0   
worker3   -        virtualbox   Running   tcp://192.168.99.103:2376           v18.09.0   
```

Moi may chi tuong ung voi 1 dia chi IP. Bay gio chi quan tam den IP cua manager de bat dau tao cluster.

* SSH vào một machine manager

```bash
> docker-machine ssh manager
   ( '>')
  /) TC (\   Core is distributed with ABSOLUTELY NO WARRANTY.
 (/-_--_-\)           www.tinycorelinux.net


```

* Khoi tao Swarm cua may `manager` voi IP la 

```bash
> docker swarm init --advertise-addr 192.168.99.100
Swarm initialized: current node (rh9krtfftxhjvb25t7k6y8gxs) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join --token SWMTKN-1-0t08e9l15ta4refv1y7jms78er7iguz8wsdrdjunkitylh5wrf-63zl0jxp7retpn7ov6sequk3e 192.168.99.100:2377

*To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.*
```

* Sau khi khoi tao Swarm, Docker se tao ma token de ket noi cac may tinh voi nhau. Co 2 loai token: Mot loai danh cho may `Manager` cai con lai la may `Worker`.

* Neu muon kiem tra lai ma token su dung 1 trong 2 cau lenh sau:
```bash
# Doi voi may worker
> docker swarm join-token worker
docker swarm join --token SWMTKN-1-0t08e9l15ta4refv1y7jms78er7iguz8wsdrdjunkitylh5wrf-63zl0jxp7retpn7ov6sequk3e 192.168.99.100:2377
```

```bash
# Doi voi may manager  
> docker swarm join-token manager
docker swarm join --token SWMTKN-1-0t08e9l15ta4refv1y7jms78er7iguz8wsdrdjunkitylh5wrf-71idqhdh53pmw66xzk86iqz49 192.168.99.100:2377
```

* Sau khi co ma token, co the join cai may lai voi nhau. Bang cach ssh den tung may worker va run lenh 

```bash
> docker ssh worker1
> docker swarm join --token SWMTKN-1-0t08e9l15ta4refv1y7jms78er7iguz8wsdrdjunkitylh5wrf-63zl0jxp7retpn7ov6sequk3e 192.168.99.100:2377
```

* Kiem tra lai cac may tinh da ket noi voi nhau chua? (chi chay duoc tren may manager)
```bash
> docker node ls
ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS      ENGINE VERSION
jxlq7hwo2xle8xm3dzosnx1mi *   manager             Ready               Active              Leader              19.03.4
h6s4eigtqtn4m1f7vywbzdf9p     worker2             Ready               Active                                  18.09.9
kn45ukmgihu4kwytw3pnbhuxb     worker3             Ready               Active                                  18.09.9

```

## Source code
* Source code (Development) [tai day](https://github.com/IT-s-Thesis/main)
* Source code (Release) [tai day](https://github.com/IT-s-Thesis/deploy)

* De he thong hoat dong binh thuong, can download source tai repos nay cho cac may `Manager`
* Ssh den mot may manager trong he hong
```bash
> docker-machine ssh manager
   ( '>')
  /) TC (\   Core is distributed with ABSOLUTELY NO WARRANTY.
 (/-_--_-\)           www.tinycorelinux.net

```
* Clone Repos
```bash
>  git clone https://github.com/IT-s-Thesis/deploy.git
```

* Trong thu muc `Services` chua cac service:
    - Traefik: lam Reverse-proxy dinh tuyen cac backend trong he thong: (Odoo, django, React)
    - Web: lam nhiem vu tinh toan tra ket qua cho cac client thong qua API
    - Client: la trang web danh cho nguoi mua hang dat hang va theo doi tinh trang don hang.
    - Delivery: la trang web danh cho shipper theo doi nhiem vu giao hang cua ho.

```bash
                                   +-----------------+           +-------------+
                     deli.hostname |                 |           |             |
                   + -------------->   Django:8000   +----------->   SQLite    |
                   |               |                 |           |             |
                   |               +-----------------+           +-------------+
                   |
+---------------+  |                +-----------------+           +-------------+
|               |  | web.hostname   |                 |           |             |
| Traefik Proxy +------------------->   Odoo:8069     +----------->  PostgreSQL |
|               |  |                |                 |           |             |
+---------------+  |                +-----------------+           +-------------+
                   |
                   |                +-----------------+           +-------------+
                   |client.hostname |                 |           |             |
                   +---------------->  React:5000     +----------->   MongoDB   |
                                    |                 |           |             |
                                    +-----------------+           +-------------+
                                            X
 +-------+proxy: network+------------------+X+----------+db: network+-----------+
                                            X
                                            X
```

## Deploy
* Quy trinh thuc hien:
```bash
Traefik ---> Web ---> Client ---> Delivery
```

* Tao bien Host trong `Manager`
```bash
> export USE_HOSTNAME=$(cat /etc/hostname) 
```

* Tao service Traefik:
```bash
> docker stack deploy -c services/traefik.yml traefik
```

* Kiem tra lai
```bash
> docker stack ls
NAME                SERVICES            ORCHESTRATOR
traefik             1                   Swarm

> docker service ls
ID                  NAME                MODE                REPLICAS            IMAGE                   PORTS
u26dxnlrea0v        demo_traefik        replicated          1/1                 traefik:1.7             *:80->80/tcp, *:443->443/tcp

> docker service ps demo_traefik
ID                  NAME                 IMAGE               NODE                DESIRED STATE       CURRENT STATE           ERROR                              PORTS
tl98cqrp09y9        demo_traefik.1       traefik:1.7         manager             Running             Running 19 hours ago                                       
fpuca8cly8hq         \_ demo_traefik.1   traefik:1.7         worker2             Shutdown            Shutdown 29 hours ago                                      
ki98d9a70yyc         \_ demo_traefik.1   traefik:1.7         worker1             Shutdown            Shutdown 30 hours ago 
```
* Note: Tai Cot Node hien thi ten may chu chay service. Nhu vidu tren, cach day 30h service chay tren may `worker1` sau do chuyen qua `worker2` va dang chay tren `manager`. Dieu do cho thay, khi 1 may chu trong docker swarm gap su co hoac bi die, Swarm mode se tim kiem may chu khac de chay. Cho den khi khong tim thay may chu nao khac thay the -> he thong xay ra loi cao.


* Lam tuong tu doi voi cac services con lai.

## Testing

* Kiem tra hostname cua may `Manager`
```bash
> cat /etc/hostname
manager
```


* Sua DNS cho Client Test: (May Ubuntu18.6)
```bash
> sudo vi /etc/hosts
```
* Copy doan ma ben duoi vao file hosts cua may Ubuntu18.6
```bash
192.168.99.100  traefik.manager
192.168.99.100  web.manager
192.168.99.100  client.manager
192.168.99.100  deli.manager
```
* Note: `192.168.99.100` IP cua may Manager

* Chay ung dung tren browse
```bash
http://traefik.manager/
http://web.manager/
http://client.manager/
http://deli.manager/
```
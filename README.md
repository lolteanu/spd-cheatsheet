# NFS

## Mdadm
Create RAID 0 with 3 devices:
```
sudo mdadm --create /dev/md0 --level=0 --raid-devices=3 /dev/sdb1 /dev/sdc1 /dev/sdd1
```

Verify RAID device:
```
sudo mdadm --detail /dev/md0
```

Stop RAID:
```
sudo mdadm --stop /dev/md0
```

Clean RAID:
```
sudo mdadm --zero-superblock /dev/sdb1 /dev/sdc1 /dev/sdd1
```

Fail disk:
```
sudo mdadm /dev/md0 --fail /dev/sdd1
```

Remove disk:
```
sudo mdadm /dev/md0 --remove /dev/sdd1
```

Add disk:
```
sudo mdadm /dev/md1 --add /dev/sdb2
```

Persistant configuration
```
sudo mdadm --detail --scan | sudo tee /etc/mdadm/mdadm.conf
sudo update-initramfs -u
```

Create partition:
```
sudo fdisk /dev/md1
n
+2G
w
```

Create xfs:
```
sudo mkfs.xfs -i size=512 /dev/md1p1
```

Mount xfs:
```
sudo mkdir -p /export
echo '/dev/md1p1 /export xfs defaults 1 2' | sudo tee -a /etc/fstab
sudo mount /export
```

## Gluster

Start gluster:
```
sudo apt install glusterfs-server
sudo systemctl enable --now glusterd
```

Probe:
```
sudo gluster peer probe lab-nfs-2
sudo gluster peer status
```

Create gluster volume:
```
sudo gluster volume create gluster-vol transport tcp lab-nfs-1:/export/brick1 lab-nfs-2:/export/brick1

# Replica
sudo gluster volume create gluster-vol replica 2 transport tcp lab-nfs-1:/export/brick1 lab-nfs-2:/export/brick1

```

Network access:
```
sudo gluster volume set gluster-vol auth.allow 192.168.100.*
sudo gluster volume start gluster-vol
```

Mount gluster on host:
```
sudo apt update
sudo apt install glusterfs-client
sudo mkdir /export
sudo mount -t glusterfs lab-nfs-1:/gluster-vol /export
df -h | grep export
```

Remove gluster:
```
host:$ sudo umount /export

sudo gluster volume stop gluster-vol
sudo gluster volume delete gluster-vol

all_replicas:$ sudo rm -rf /export/brick1/
```

## DNS
```
# /etc/bind/named.conf.options

acl goodguys { 192.168.100.11; 127.0.0.1; ::1; };

options {
[...]
        dnssec-validation no;
[...]
        listen-on { 192.168.100.11; localhost; };
[...]
        allow-recursion { goodguys; };
        recursion yes;
[...]
};
```

```
# /etc/bind/named.conf.local

zone "laur.scgc.ro" {
    type master;
    file "/etc/bind/db.laur.scgc.ro"; # zone file path
    allow-transfer { 192.168.100.12; }; # slave VM IP address
};
```

```
# /etc/bind/db.laur.scgc.ro

;
; BIND data file for local loopback interface
;
$TTL    604800
@       IN      SOA     laur.scgc.ro. root.laur.scgc.ro. (
                              8         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;

; name servers - NS records
    IN      NS      ns1.laur.scgc.ro.

; name servers - A records
ns1.laur.scgc.ro.          IN      A      192.168.100.11
www.laur.scgc.ro.          IN      A      192.168.100.11


; mail exchange records
maildns                     IN      A      192.168.100.11
mailhelper                  IN      A      192.168.100.12
@    IN    MX  10    maildns
@    IN    MX  20    mailhelper
```

Check configurations
```
named-checkconf
named-checkzone laur.scgc.ro /etc/bind/db.laur.scgc.ro

host www.laur.scgc.ro localhost
host -t ns laur.scgc.ro localhost
host -t mx laur.scgc.ro localhost
host ns1.laur.scgc.ro localhost

# Recursion
non_dns:$ host google.com dns
```

[Zone transfer](https://scgc.pages.upb.ro/cloud-courses/docs/management/dns#zone-transfer)

# Certificates

Generate private key:
```
openssl genrsa -out server.key 2048
```

Generate signing request:
```
openssl req -new -key server.key -out server.csr
```

Self sign certificate:
```
openssl ca -config ca.cnf -policy signing_policy -extensions signing_req -in server.csr -out server.crt
```

Verify certificate matches key:
```
openssl x509 -in server.crt -noout -modulus | md5sum
openssl rsa -in server.key -noout -modulus | md5sum
```

Verify certificate:
```
openssl verify -CAfile ca/ca.crt server.crt
```

Monitor traffic:
```
sudo tcpdump -A -i lo port 12345
```

Unsecured server:
```
nc localhost 12345
nc -l 12345
```

Secured server:
```
openssl s_server -key server.key -cert server.crt -accept 12345
openssl s_client -CAfile ca/ca.crt -connect localhost:12345
```

## HAproxy
```
# Append to /etc/haproxy/haproxy.cfg

frontend www
        bind *:80
        default_backend realservers

backend realservers
        mode http
        balance roundrobin
        option httpchk
        http-check send meth GET

	# Cache
        http-request cache-use main
        http-response cache-store main

        server realserver-1 192.168.100.72:80 check inter 5s
        server realserver-2 192.168.100.73:80 check inter 5s

cache main
        total-max-size 256
        max-object-size 20971520
        max-age 5
```

Verify cache:
```
echo 'show cache' | sudo socat - /run/haproxy/admin.sock

httperf --server 192.168.100.71 --port 80 --num-conns 10000 --rate 1000 --timeout 5
```

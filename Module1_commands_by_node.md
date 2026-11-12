# Модуль 1 (ДЭ 09.02.06-1-2026) — команды по узлам

Источник: практикум «Сетевое и системное администрирование. Подготовка к ДЭ 09.02.06-1-2026».

## Общая адресация (пример из практикума)

| Устройство | IP-адрес | Шлюз |
|---|---|---|
| ISP (к провайдеру) | DHCP | — |
| ISP → HQ-RTR | 172.16.1.1/28 | — |
| ISP → BR-RTR | 172.16.2.1/28 | — |
| HQ-RTR (к ISP) | 172.16.1.2/28 | 172.16.1.1 |
| HQ-RTR VLAN 100 (HQ-SRV) | 192.168.100.1/27 | — |
| HQ-RTR VLAN 200 (HQ-CLI) | 192.168.200.1/24 | — |
| HQ-RTR VLAN 999 (управление) | 192.168.99.1/29 | — |
| BR-RTR (к ISP) | 172.16.2.2/28 | 172.16.2.1 |
| BR-RTR (LAN) | 192.168.0.1/28 | — |
| HQ-SRV | 192.168.100.2/27 | 192.168.100.1 |
| HQ-CLI | DHCP (192.168.200.2–254) | 192.168.200.1 |
| BR-SRV | 192.168.0.2/28 | 192.168.0.1 |
| Туннель HQ-RTR / BR-RTR | 10.10.10.1/30 / 10.10.10.2/30 | — |

Важно: имена интерфейсов (ens19, ens20, ge0, ge1 и т.д.) — из примера, на вашем стенде они могут отличаться. Сверяйте по MAC-адресам (`ip -c a`, `show port brief`).

---

## 1. ISP (ОС Альт / JeOS)

**Имя (без домена):**
```bash
hostnamectl set-hostname isp; exec bash
```

**Интерфейс в сторону провайдера — DHCP** (ens19 — пример):
```bash
mkdir -p /etc/net/ifaces/ens19
cat > /etc/net/ifaces/ens19/options <<EOF
TYPE=eth
BOOTPROTO=dhcp
EOF
```

**Интерфейсы в сторону HQ-RTR и BR-RTR — статика:**
```bash
mkdir -p /etc/net/ifaces/ens20 /etc/net/ifaces/ens21
printf 'TYPE=eth\nBOOTPROTO=static\n' > /etc/net/ifaces/ens20/options
printf 'TYPE=eth\nBOOTPROTO=static\n' > /etc/net/ifaces/ens21/options
echo "172.16.1.1/28" > /etc/net/ifaces/ens20/ipv4address
echo "172.16.2.1/28" > /etc/net/ifaces/ens21/ipv4address
systemctl restart network
```

**Включить пересылку пакетов:**
```bash
sed -i "s/net.ipv4.ip_forward = 0/net.ipv4.ip_forward = 1/g" /etc/net/sysctl.conf
systemctl restart network
sysctl net.ipv4.ip_forward
```

**NAT (динамическая трансляция портов) через iptables:**
```bash
apt-get update
apt-get install -y iptables
iptables -t nat -A POSTROUTING -s 172.16.1.0/28 -o ens19 -j MASQUERADE
iptables -t nat -A POSTROUTING -s 172.16.2.0/28 -o ens19 -j MASQUERADE
iptables-save >> /etc/sysconfig/iptables
systemctl enable --now iptables
```

**Часовой пояс:**
```bash
timedatectl set-timezone Europe/Moscow
```

**Проверка:** `ip -c a`, `ip -c r`, `ping 8.8.8.8`, `iptables -t nat -L -n -v`, `timedatectl`.

---

## 2. HQ-RTR (EcoRouterOS)

**Имя, домен, сохранение:**
```
enable
configure terminal
hostname hq-rtr
ip domain-name au-team.irpo
write memory
```

**Пользователь net_admin (максимальные привилегии):**
```
username net_admin
 password P@ssw0rd
 role admin
 exit
```

**Интерфейс к ISP + маршрут по умолчанию:**
```
interface isp
 description "ISP"
 ip address 172.16.1.2/28
 exit
ip route 0.0.0.0/0 172.16.1.1
port ge0
 service-instance ge0/isp
  encapsulation untagged
  connect ip interface isp
  exit
 exit
```

**VLAN-интерфейсы (router-on-a-stick на одном порту ge1):**
```
interface vl100
 description "VLAN 100"
 ip address 192.168.100.1/27
 exit
interface vl200
 description "VLAN 200"
 ip address 192.168.200.1/24
 exit
interface vl999
 description "VLAN 999"
 ip address 192.168.99.1/29
 exit
port ge1
 service-instance ge1/vl100
  encapsulation dot1q 100 exact
  rewrite pop 1
  connect ip interface vl100
  exit
 service-instance ge1/vl200
  encapsulation dot1q 200 exact
  rewrite pop 1
  connect ip interface vl200
  exit
 service-instance ge1/vl999
  encapsulation dot1q 999 exact
  rewrite pop 1
  connect ip interface vl999
  exit
 exit
```

**GRE-туннель до BR-RTR:**
```
interface tunnel.0
 description "GRE"
 ip address 10.10.10.1/30
 ip tunnel 172.16.1.2 172.16.2.2 mode gre
 exit
```
*(Вариант IP-in-IP: добавить `ip mtu 1400` и `mode ipip`.)*

**OSPF с MD5-паролем, только на туннеле:**
```
router ospf 1
 ospf router-id 10.10.10.1
 passive-interface default
 no passive-interface tunnel.0
 network 10.10.10.0/30 area 0
 network 192.168.100.0/27 area 0
 network 192.168.200.0/24 area 0
 network 192.168.99.0/29 area 0
 exit
interface tunnel.0
 ip ospf authentication message-digest
 ip ospf message-digest-key 1 md5 P@ssw0rd
 exit
```

**Динамический NAT в сторону ISP:**
```
interface isp
 ip nat outside
 exit
interface vl100
 ip nat inside
 exit
interface vl200
 ip nat inside
 exit
interface vl999
 ip nat inside
 exit
ip nat pool VLAN100 192.168.100.1-192.168.100.30
ip nat pool VLAN200 192.168.200.1-192.168.200.254
ip nat pool VLAN999 192.168.99.1-192.168.99.6
ip nat source dynamic inside-to-outside pool VLAN100 overload interface isp
ip nat source dynamic inside-to-outside pool VLAN200 overload interface isp
ip nat source dynamic inside-to-outside pool VLAN999 overload interface isp
```

**DHCP-сервер для HQ-CLI (VLAN 200):**
```
ip pool VLAN200 192.168.200.2-192.168.200.254
dhcp-server 1
 pool VLAN200 1
  mask 24
  gateway 192.168.200.1
  dns 192.168.100.2
  domain-name au-team.irpo
  exit
 exit
interface vl200
 dhcp-server 1
 exit
```

**Часовой пояс и сохранение:**
```
ntp timezone utc+3
write memory
```

**Проверка:** `show ip interface brief`, `show service-instance brief`, `show interface tunnel.0`, `show ip ospf neighbor`, `show ip route ospf`, `show ip nat translations`, `show running-config dhcp-server 1`, `show users localdb`, `show ntp timezone`.

---

## 3. BR-RTR (EcoRouterOS)

**Имя, домен:**
```
enable
configure terminal
hostname br-rtr
ip domain-name au-team.irpo
write memory
```

**Пользователь net_admin:**
```
username net_admin
 password P@ssw0rd
 role admin
 exit
```

**Интерфейс к ISP + маршрут по умолчанию:**
```
interface isp
 description "ISP"
 ip address 172.16.2.2/28
 exit
ip route 0.0.0.0/0 172.16.2.1
port ge0
 service-instance ge0/isp
  encapsulation untagged
  connect ip interface isp
  exit
 exit
```

**LAN в сторону BR-SRV:**
```
port ge1
 service-instance ge1/int1
  encapsulation untagged
  exit
 exit
interface int1
 description "BR-Net"
 ip address 192.168.0.1/28
 connect port ge1 service-instance ge1/int1
 exit
```

**GRE-туннель до HQ-RTR:**
```
interface tunnel.0
 description "GRE"
 ip address 10.10.10.2/30
 ip tunnel 172.16.2.2 172.16.1.2 mode gre
 exit
```
*(Вариант IP-in-IP: `ip mtu 1400` и `mode ipip`.)*

**OSPF с MD5-паролем:**
```
router ospf 1
 ospf router-id 10.10.10.2
 passive-interface default
 no passive-interface tunnel.0
 network 192.168.0.0/28 area 0
 network 10.10.10.0/30 area 0
 exit
interface tunnel.0
 ip ospf authentication message-digest
 ip ospf message-digest-key 1 md5 P@ssw0rd
 exit
```

**Динамический NAT:**
```
interface isp
 ip nat outside
 exit
interface int1
 ip nat inside
 exit
ip nat pool BR-Net 192.168.0.1-192.168.0.14
ip nat source dynamic inside-to-outside pool BR-Net overload interface isp
```

**Часовой пояс и сохранение:**
```
ntp timezone utc+3
write memory
```

**Проверка:** те же команды `show ...`, что и на HQ-RTR; связность — `ping 10.10.10.1`.

---

## 4. HQ-SW (только если коммутатор — виртуальная машина, ОС Альт + Open vSwitch)

Если отдельной ВМ-коммутатора нет — порты доступа (VLAN 100 для HQ-SRV, VLAN 200 для HQ-CLI) настраиваются на гипервизоре в параметрах сетевого адаптера ВМ (поле VLAN Tag).

```bash
apt-get install -y openvswitch
systemctl enable --now openvswitch
# для каждого интерфейса: /etc/net/ifaces/<if>/options с TYPE=eth
systemctl restart network

ovs-vsctl add-br SW
ovs-vsctl add-port SW ens3 trunk=100,200,999   # транк к HQ-RTR
ovs-vsctl add-port SW ens4 tag=100              # HQ-SRV
ovs-vsctl add-port SW ens5 tag=200              # HQ-CLI
ovs-vsctl show
```
Интерфейсы ens3/ens4/ens5 — пример, сверяйте по MAC. Часовой пояс на HQ-SW не настраивается.

---

## 5. HQ-SRV (ОС Альт Сервер)

**Имя:**
```bash
hostnamectl set-hostname hq-srv.au-team.irpo; exec bash
```

**IP-адрес (ens19 — пример имени интерфейса):**
```bash
mkdir -p /etc/net/ifaces/ens19
printf 'TYPE=eth\nBOOTPROTO=static\n' > /etc/net/ifaces/ens19/options
echo "192.168.100.2/27" > /etc/net/ifaces/ens19/ipv4address
echo "default via 192.168.100.1" > /etc/net/ifaces/ens19/ipv4route
printf 'search au-team.irpo\nnameserver 77.88.8.7\n' > /etc/net/ifaces/ens19/resolv.conf
systemctl restart network
```
*(После настройки BIND можно сменить nameserver на 127.0.0.1 / 192.168.100.2.)*

**Пользователь sshuser (UID 2026, sudo без пароля):**
```bash
useradd sshuser -u 2026
passwd sshuser                                  # ввести P@ssw0rd
usermod -aG wheel sshuser
echo "sshuser ALL=(ALL:ALL) NOPASSWD: ALL" >> /etc/sudoers
```

**Безопасный SSH:**
```bash
vim /etc/openssh/sshd_config
#   Port 2026
#   AllowUsers sshuser
#   MaxAuthTries 2
#   Banner /etc/openssh/banner
echo "Authorized access only" > /etc/openssh/banner
systemctl restart sshd
```

**DNS-сервер BIND:**
```bash
apt-get update && apt-get install -y bind bind-utils
vim /var/lib/bind/etc/options.conf
#   listen-on { any; };
#   forwarders { 77.88.8.7; 77.88.8.3; };
#   allow-query { any; };
```
Зоны — добавить в **конец** `/var/lib/bind/etc/rfc1912.conf`:
```
zone "au-team.irpo" {
    type master;
    file "au-team.irpo";
};
zone "100.168.192.in-addr.arpa" {
    type master;
    file "100.168.192.in-addr.arpa";
};
zone "200.168.192.in-addr.arpa" {
    type master;
    file "200.168.192.in-addr.arpa";
};
```
Файлы зон:
```bash
cd /var/lib/bind/etc/zone
cp empty au-team.irpo
cp empty 100.168.192.in-addr.arpa
cp empty 200.168.192.in-addr.arpa
```
Содержимое прямой зоны `au-team.irpo` (в практикуме приведено скриншотом — ниже типовой вариант; SOA/шапку из `empty` сохраните):
```
hq-rtr   IN A   192.168.100.1
br-rtr   IN A   192.168.0.1
hq-srv   IN A   192.168.100.2
hq-cli   IN A   192.168.200.2
br-srv   IN A   192.168.0.2
docker   IN A   172.16.1.1
web      IN A   172.16.2.1
```
Обратная зона `100.168.192.in-addr.arpa`:
```
1   IN PTR   hq-rtr.au-team.irpo.
2   IN PTR   hq-srv.au-team.irpo.
```
Обратная зона `200.168.192.in-addr.arpa`:
```
2   IN PTR   hq-cli.au-team.irpo.
```
Запуск:
```bash
rndc-confgen > /etc/bind/rndc.key
sed -i '6,$d' /etc/bind/rndc.key
chown -R root:named /var/lib/bind/etc/zone/*
named-checkconf
named-checkconf -z
systemctl enable --now bind.service
systemctl status bind
```

**Часовой пояс:**
```bash
timedatectl set-timezone Europe/Moscow
```

**Проверка:** `ip -c a`, `ip -c r`, `id sshuser`, `ssh -p 2026 sshuser@192.168.100.2`, `host hq-srv.au-team.irpo localhost`, `host 192.168.100.2 localhost`, `timedatectl`.

---

## 6. BR-SRV (ОС Альт Сервер)

**Имя:**
```bash
hostnamectl set-hostname br-srv.au-team.irpo; exec bash
```

**IP-адрес:**
```bash
mkdir -p /etc/net/ifaces/ens19
printf 'TYPE=eth\nBOOTPROTO=static\n' > /etc/net/ifaces/ens19/options
echo "192.168.0.2/28" > /etc/net/ifaces/ens19/ipv4address
echo "default via 192.168.0.1" > /etc/net/ifaces/ens19/ipv4route
printf 'search au-team.irpo\nnameserver 192.168.100.2\n' > /etc/net/ifaces/ens19/resolv.conf
systemctl restart network
```

**Пользователь sshuser (UID 2026, sudo без пароля):**
```bash
useradd sshuser -u 2026
passwd sshuser                                  # P@ssw0rd
usermod -aG wheel sshuser
echo "sshuser ALL=(ALL:ALL) NOPASSWD: ALL" >> /etc/sudoers
```

**Безопасный SSH:**
```bash
vim /etc/openssh/sshd_config
#   Port 2026
#   AllowUsers sshuser
#   MaxAuthTries 2
#   Banner /etc/openssh/banner
echo "Authorized access only" > /etc/openssh/banner
systemctl restart sshd
```

**Часовой пояс:**
```bash
timedatectl set-timezone Europe/Moscow
```

**Проверка:** `ping 192.168.100.2` (через туннель/OSPF), `tracepath 192.168.100.2`, `ssh -p 2026 sshuser@192.168.0.2`.

---

## 7. HQ-CLI (ОС Альт Рабочая станция)

**Имя:**
```bash
hostnamectl set-hostname hq-cli.au-team.irpo; exec bash
```

**Сеть:** клиент получает адрес по DHCP от HQ-RTR (VLAN 200) — ничего вручную настраивать не нужно, только убедиться, что интерфейс в режиме DHCP.

**Часовой пояс:**
```bash
timedatectl set-timezone Europe/Moscow
```

**Проверка:**
```bash
ip -c a                  # адрес из 192.168.200.0/24
ip -c r                  # шлюз 192.168.200.1
cat /etc/resolv.conf     # nameserver 192.168.100.2, search au-team.irpo
ping 192.168.100.2
ping ya.ru
hostname -f
timedatectl
```

---

## Порядок выполнения (рекомендуемый)

1. Имена и IP на всех узлах → 2. ISP (DHCP, forwarding, NAT) → 3. Маршрутизаторы: интерфейсы к ISP, маршрут по умолчанию, NAT → 4. Пользователи → 5. VLAN/коммутация (HQ-SW или гипервизор) → 6. SSH на серверах → 7. Туннель → 8. OSPF → 9. DHCP на HQ-RTR → 10. DNS на HQ-SRV → 11. Часовой пояс везде → 12. Отчёт (таблица адресов, сведения о коммутации, туннеле, OSPF, DHCP).

Не забывайте `write memory` на маршрутизаторах после каждого блока.

## debian8/9 openvz&kvm 替换"..."
### ssh ss bbr
```sh
echo 'deb [arch=amd64] http://ftp.us.debian.org/debian/ stable main contrib non-free' > /etc/apt/sources.list
apt update
apt install -y ca-certificates htop curl wget openssh-server vim ranger git shadowsocks-libev aria2 iptables
# key
mkdir ~/.ssh -p
cat > ~/.ssh/authorized_keys<<EOF
ssh-rsa AAAAB3NzaC1yc2E...
EOF
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
sed -i 's/^PasswordAuthentication yes/#&/' /etc/ssh/sshd_config
sed -i 's/^PubkeyAuthentication no/#&/' /etc/ssh/sshd_config
echo 'PasswordAuthentication no' >> /etc/ssh/sshd_config
echo 'PubkeyAuthentication yes' >> /etc/ssh/sshd_config
systemctl restart sshd
# ss
cat>/etc/shadowsocks-libev/config.json<<EOF
{
    "server":"0.0.0.0",
    "server_port":40000,
    "password":"...",
    "no_delay":true,
    "method":"rc4-md5",
}
EOF
systemctl restart shadowsocks-libev
systemctl status --no-pager -l shadowsocks-libev
# bbr
wget https://raw.githubusercontent.com/tkkcc/rinetd/master/rinetd -O /usr/bin/rinetd
chmod +x /usr/bin/rinetd
echo '0.0.0.0 40000 0.0.0.0 40000' > /etc/rinetd.conf
iface=$(ip -4 addr | awk '{if ($1 ~ /inet/ && $NF ~ /^[ve]/) {a=$NF}} END{print a}')
echo "/usr/bin/rinetd -f -c /etc/rinetd.conf raw $iface&" >> /etc/rc.local
chmod +x /etc/rc.local
```
> `rinetd` is same as [nanqinlang](https://github.com/tcp-nanqinlang/lkl-rinetd), better than [linhua55](https://github.com/linhua55/lkl_study) in my case

### aria2 http-server
```sh
# aria2
mkdir -p /download
mkdir -p /etc/aria2/
echo > /etc/aria2/aria2.session
echo "" >> /etc/rc.local
echo "aria2c --conf=/etc/aria2/aria2.conf -D" >> /etc/rc.local
chmod +x /etc/rc.local
cat>/etc/aria2/aria2.conf<<EOF
dir=/download
input-file=/etc/aria2/aria2.session
save-session=/etc/aria2/aria2.session
continue=true
max-concurrent-downloads=16
max-connection-per-server=16
split=16
enable-rpc=true
rpc-listen-all=true
rpc-allow-origin-all=true
rpc-listen-port=40050
rpc-secret=...
EOF
# update tracker
f=/etc/aria2/aria2.conf
list=`wget -qO- https://raw.githubusercontent.com/ngosang/trackerslist/master/trackers_all.txt|awk NF|sed ":a;N;s/\n/,/g;ta"`
if [ -z "`grep "bt-tracker" $f`" ]; then
    sed -i '$a bt-tracker='${list} $f
else
    sed -i "s@bt-tracker.*@bt-tracker=$list@g" $f
fi
# start
aria2c --conf=/etc/aria2/aria2.conf -D
pgrep aria2c
# node
curl -sL https://deb.nodesource.com/setup_11.x | bash -
apt-get install -y nodejs
# http-server
npm i -g http-server
nohup http-server /download -p 80&
echo "" >> /etc/rc.local
echo "http-server /download -p 80&" >> /etc/rc.local
```

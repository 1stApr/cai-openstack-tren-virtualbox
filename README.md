# Cài đặt Openstack Với Kolla-Ansible
- Tài liệu này hướng dẫn triển khai cài đặt Openstack trên các máy chủ Ubuntu Server 20.04 LTS
- Ví dụ trong tài liệu này được thực hiện trên máy ảo Ubuntu Server 20.04 (RAM: 8GB, Processor: 4CPU, Storage: 2x50GB)

# Tài liệu tham khảo

[Ansible](https://docs.ansible.com/)

[Docker](https://docs.docker.com/)

[Openstack Docs](https://docs.openstack.org/)


# 1: Tạo máy ảo và cài một số package cần thiết

## 1.1: Tạo máy ảo
Nên cài hệ điều hành Ubuntu Server 20.04 và cấu hình như sau:

|System|Storage|Network|
|------|-------|-------|
|<img src="/src/picture_system_motherboard.png" width="250">|<img src="/src/picture_storage01.png" width="250">|<img src="/src/picture_network01.png" width="250">|
|<img src="/src/src/picture_system_processor.png" width="250">|<img src="/src/picture_storage02.png" width="250">|<img src="/src/picture_network02.png" width="250">|
|-----|-----|<img src="/src/picture_network03.png" width="250">|

<img src="/src/picture_information.png" width="250">


## 1.2: Cài đặt một số package cần thiết

Cài đặt SSH
```
sudo apt update
apt install ssh 
```

Chỉnh timezone
```
timedatectl set-timezone Asia/Ho_Chi_Minh
```

Cài đặt sshpass
```
apt install sshpass
```

Tạo khóa ssh
```
ssh-keygen -t rsa -N '' -f ~/.ssh/id_rsa
```

Tạo phân vùng Cinder
```
apt install lvm2
```
```
pvcreate /dev/sdb
vgcreate cinder-volumes /dev/sdb
```
Cài đặt Python và các thư viện phụ thuộc
```
sudo apt install python3-dev libffi-dev gcc libssl-dev -y
```

Cài đặt pip
```
sudo apt install python3-pip -y 
```

Đảm bảo phiên bản mới nhất của pip được cài đặt
```
sudo pip3 install -U pip
```

Cài đặt Ansible
```
pip3 install "ansible==2.10.7"
```

Cài đặt Kolla-ansible
```
sudo pip3 install kolla-ansible
```

Tạo thư mục /etc/kolla 
```
sudo mkdir -p /etc/kolla
sudo chown $USER:$USER /etc/kolla
```

Copy các file globals.yml, passwords.yml vào thư mục /etc/kolla 
```
cp -r /usr/local/share/kolla-ansible/etc_examples/kolla/* /etc/kolla
```

Copy 2 file all-in-one và multinode 
```
cp /usr/local/share/kolla-ansible/ansible/inventory/* .
```

# 2: Configure

Chỉnh sửa file /etc/hosts
```
vi /etc/hosts
```
```
192.168.48.4 control01

```

Thêm các dòng sau vào trong file ansible.cfg
```
mkdir /etc/ansible/
vi /etc/ansible/ansible.cfg
```
```
[defaults]
host_key_checking=False
pipelining=True
forks=100
```

Chỉnh sửa file multinode
```
vi multinode
```
```
[all:vars]
ansible_connection=ssh
ansible_become=true
ansible_ssh_port=22
ansible_ssh_user=USER
ansible_ssh_pass=SSH_PASS
ansible_sudo_pass=SUDO_PASS

[control]
control01

[network]
control01

[monitoring]
control01

[storage]
storage01

[compute]
compute01

[deployment]
localhost  ansible_connection=local

```

Kiểm tra cấu hình multinode có đúng hay không
```
ansible -i multinode all -m ping
```

<img src="/src/picture_ping.png" width="250">

Chạy trình tạo mật khẩu ngẫu nhiên
```
kolla-genpwd
```

#### 2.4.6 Chỉnh sửa file cấu hình chính cho Kolla-Ansible

```
vi /etc/kolla/globals.yml
```
```
kolla_base_distro: "ubuntu"
kolla_install_type: "source"
openstack_release: "xena"
neutron_bridge_name: "br-ex"
network_interface: "enp0s8"
neutron_external_interface: "enp0s9"
kolla_internal_vip_address: "192.168.48.4"
enable_chrony: "yes"
enable_neutron_provider_networks: "yes"
enable_haproxy: "no"
nova_compute_virt_type: "qemu"

#Cài Block Storage-Cinder
enable_cinder: "yes"
enable_cinder_backend_lvm: "yes"
enable_cinder_backup: "no"

#Cài đặt dịch vụ giám sát các máy chủ
enable_prometheus: "yes"
enable_grafana: "yes"

#Cài một vài dịch vụ khác
enable_heat: "yes"
enable_trove: "yes"
enable_senlin: "yes"
enable_designate: "yes"
enable_cloudkitty: "yes"
```

**Kiểm tra lại cấu hình**
```
cat /etc/kolla/globals.yml | egrep -v '^#|^$'
```

# 3: Triển khai cài đặt


## 3.1: Bootstrap servers với kolla-ansible

```
kolla-ansible -i multinode bootstrap-servers
```
<img src="/src/picture_bootstrap.png" width="250">

## 3.2: Thực hiện kiểm tra trước khi triển khai cho các máy chủ:

```
kolla-ansible -i multinode prechecks
```
<img src="/src/picture_prechecks.png" width="250">

## 3.3 Tải về các Docker image:

```
kolla-ansible -i multinode pull
```
<img src="/src/picture_pull.png" width="250">

## 3.4 Cài đặt các dịch vụ

```
kolla-ansible -i multinode deploy
```
<img src="/src/picture_deploy.png" width="250">

--------------------

# 4 Sử dụng OpenStack

## 4.1 Cài OpenStack CLI client:

```
apt install python3-openstackclient -y
```

Tạo file admin-openrc:
```
kolla-ansible post-deploy
```
```
. /etc/kolla/admin-openrc.sh
cp /etc/kolla/admin-openrc.sh admin-openrc.sh
```

## 4.2 Lấy người dùng quản trị và tenant IDs

```
ADMIN_USER_ID=$(openstack user list | awk '/ admin / {print $2}')
ADMIN_PROJECT_ID=$(openstack project list | awk '/ admin / {print $2}')
ADMIN_SEC_GROUP=$(openstack security group list --project ${ADMIN_PROJECT_ID} | awk '/ default / {print $2}')
```

## 4.3 Cấu hình Sec Group

```
openstack security group rule create --ingress --ethertype IPv4 \
   --protocol icmp ${ADMIN_SEC_GROUP}
openstack security group rule create --ingress --ethertype IPv4 \
   --protocol tcp --dst-port 22 ${ADMIN_SEC_GROUP}
openstack security group rule create --ingress --ethertype IPv4 \
   --protocol tcp --dst-port 8000 ${ADMIN_SEC_GROUP}
openstack security group rule create --ingress --ethertype IPv4 \
   --protocol tcp --dst-port 8080 ${ADMIN_SEC_GROUP}
 ```
 
## 4.4 Cấu hình nova public-key và quotas

```
openstack keypair create --public-key ~/.ssh/id_rsa.pub mykey
```
```
openstack quota set --instances 20 ${ADMIN_PROJECT_ID}
openstack quota set --cores 20 ${ADMIN_PROJECT_ID}
openstack quota set --ram 48000 ${ADMIN_PROJECT_ID}
```
```
openstack flavor create --id 1 --ram 512 --disk 1 --vcpus 1 flavor01
openstack flavor create --id 2 --ram 1024 --disk 1 --vcpus 1 flavor02
openstack flavor create --id 3 --ram 2048 --disk 5 --vcpus 2 flavor03
openstack flavor create --id 4 --ram 512 --disk 0 --vcpus 1 flavor04
openstack flavor create --id 5 --ram 1024 --disk 0 --vcpus 1 flavor05
openstack flavor create --id 6 --ram 2048 --disk 0 --vcpus 2 flavor06
```

### 4.5 Tải lên OpenStack image cirros

```
wget http://download.cirros-cloud.net/0.4.0/cirros-0.4.0-x86_64-disk.img
```
```
apt install python3-glanceclient -y
```
```
glance image-create --name "cirros" \
  --file cirros-0.4.0-x86_64-disk.img \
  --disk-format qcow2 --container-format bare \
  --visibility=public
```

**Lấy mật khẩu truy cập Horizon Dashboard**

```
grep "keystone_admin" /etc/kolla/passwords.yml
```

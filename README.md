# redhat

# Radhat test


## 第二冊
* Shell scripts
    * 
* 正規表示式
    * 抓特定字串 grep
* P44 排程
    * 每n分鐘跑一次
    * 給一個指令 設定規定的排成 crontab -e
    * 

### Tuned
  * 安裝 啟動enable
  * 改預設模式 
  * 使用tuned-adm tab看參數設定
```
yum install tuned
systemctl enable --now tuned
tuned-adm active #查看目前模式
tuned-adm list
tuned-adm profile throughput-performance #修改模式
tuned-adm active
```


### ACL 同中取異
P99 群組中只有某人沒權限
* setfacl
    * -m 新增修改, -x 移除 
    * -R 以下目錄都套用 :rwX (
    * d: 設定dedault 將來建的都套用
    * --set-file=-
    * mask owner other不受影響
* getfacl

```
setfacl -m u:name:rX file  #user
setfacl -m g:name:rw file  #group
setfacl -m o::- file       #other
setfacl -m m::r file       #mask

getfacl file-A | setfacl --set-file=- file-B #複製A給B

(test)
#設定目前所有檔案路徑
setfacl -Rm g:consultants:rwX /shares/content
setfacl -Rm u:consultant1:- /shares/content
#設定將來default
setfacl -m d:g:consultants:rwx /shares/content
setfacl -m d:u:consultant1:- /shares/content
```


### SELinux
* t1:設定mode
```
getenforce  #查看目前
vim /etc/selinux/config #修改設定黨
SELINUX=permissive/enforcing #下次重開的模式
```

* t2:semanager 有一個法條錯的要改掉
    * 檢查狀態 ll -Z
    * cp 繼承目的地type
    * cp -a 複製原先設定的ACL selinux

```
#一次性修改 restorecon會被洗掉
chcon -t httpd_sys_content_t /virtual

#修法
semanage fcontext -l
restorecon -Rv /var/www/
semanage fcontext -a -t httpd_sys_content_t '/virtual(/.*)?'
restorecon -RFvv /virtual  #才會套用新的
semanage fcontext -l -C  #查看自己增加的rule

#test
#查看過去apache type
restorecon -FRvv /var/www/html  #先restore比較保險
ll -dZ /var/www/html
semanage fcontext -a -t httpd_sys_content_t '/custom(/.*)?'
restorecon -FRvv /custom

semanage fcontext -l | grep type錯誤的檔案
semanage fcontext -d /路徑/file #刪除錯誤的
restorecon -FRvv /custom
```

* SELinux booleans
    * webserver開關大鈔
    * yum install selinux-policy-doc
    * man -k '_selinux' | grep http
    * 搜尋 /EXAMPLE
    * setsebool -P httpd_enable_homedirs on 
        * -P 才會重開生效
    * 檢查: 
        * curl http://servera/~student/index.html
*  T-shoot
```
yum -y install setroubleshoot-server
grep sealert /var/log/messages
sealert -l b1c9cc8f-a953-4625-b79b-82c4f4f1fee3

#如果報告過期
sealert -a /var/log/audit/audit.log  #把歷史的log都重新處理一次
```


### 硬碟相關
* 系統碟不動
* 第二塊硬碟
    * PV
    * LV放大
    * SWAP
* 第三顆硬碟
    * VDO 會用到一顆硬碟



### Partition
* **切基本part**
    * P154 看清楚 MiB MB
    * 不要動到系統碟 直接GG

```
lsblk #先看有幾個硬碟 規劃硬碟切割
parted /dev/vda print
parted /dev/vda help

parted /dev/vdb mklabel msdos/gpt
parted /dev/vdb  #都可以用help輔助
udevadm settle #重新整理硬碟狀態
cat /proc/partitions  #查看part

#一行搞定
parted /dev/vdb mkpart primary xfs 2048s 1000MB
#format硬碟
mkfs.xfs /dev/vdb1 

mkdir /data
mount /dev/vdb1 /data

#查看UUID
blkid /dev/vdb1
vim /etc/fstab
UUID="7a20315d-ed8b-4e75-a5b6-24ff9e1f9838" /data xfs defaults 0 0

#(重要)檢查是否有問題
mount -a
systemctl daemon-reload


#刪除
parted /dev/vdb rm 1 #print確認編號
```
 
* **P170 swap partition**
    * type: inux-swap
    * swapon
    * UUID="" swap swap defaults 0 0
    
```
parted /dev/vdb print

#type:linux-swap
parted /dev/vdb mkpart myswap linux-swap 1001MB 1501MB   
parted /dev/vdb print
udevadm settle

#format swap
mkswap /dev/vdb2

#設定
vim etc/fstab
UUID=cb7f71ca-ee82-430e-ad4b-7dda12632328 swap swap defaults 0 0

systemctl daemon-reload
mount -a
swapon -a
swapon --show (-s)
udevadm settle

systemctl reboot
```


### LVM
![](https://i.imgur.com/RKkCY2r.png)


* -s 指定PE大小
* 放大LVM
    * vgdisplay 看空間還夠不夠
    * 規劃好要切多大

```
#切partition
parted /dev/vdb mklabel gpt
parted  dev/vdb mkpart primary 1MiB 257MiB
parted /dev/vdb mkpart primary 258MiB 514MiB
parted /dev/vdb set 1 lvm on
parted /dev/vdb set 2 lvm on
udevadm settle

#建立一系列
pvcreate /dev/vdb1 /dev/vdb2
pvdisplay
vgcreate -s 8MB servera_01_vg /dev/vdb1 /dev/vdb2  #指定PE大小
vgdisplay #可檢查PE
lvcreate -n servera_01_lv -L 400M servera_01_vg
lvdisplay

#format
mkfs -t xfs /dev/servera_01_vg/servera_01_lv
mkdir /data

vim etc/fstab
/dev/servera_01_vg/servera_01_lv /data xfs defaults 0 0
systemctl daemon-reload
mount -a

#檢查有弄上去
df -h /data
lsblk

systemctl reboot
```

### 放大LVM P201 
* 先檢查free還夠不夠
* 不夠就要再切partition
* 有兩種
* xfs
```
vgdisplay servera_01_vg #檢查大小
df -h #查看硬碟空間

#切新part

parted  /dev/vdb mkpart primary 515MiB 1027MiB
parted  /dev/vdb set 3 lvm on
udevadm settle

#lvm處理
pvcreate /dev/vdb3
vgextend servera_01_vg /dev/vdb3
vgdisplay servera_01_vg

#放大
lvextend -l 700M /dev/servera_01_vg/servera_01_lv #設定為X大小
lvextend -L +700M /dev/servera_01_vg/servera_01_lv #放大Y大小

#xfs掛上
xfs_growfs /data
df -h /data
```
* ext4
```
#轉為ext4
df -h #查看硬碟空間
umount /data
mkfs -t ext4 /dev/vg0/lv0
mount /dev/vg0/lv0 /data

#放大
lvextend -l 700M /dev/servera_01_vg/servera_01_lv #設定為X大小
lvextend -L +700M /dev/servera_01_vg/servera_01_lv #放大Y大小

#掛上
resize2fs /dev/vg0/lv0
df -h /data
```
* swap
    * 就是拿下來在放上去
```
swapoff -v /dev/vgname/lvname
lvextend -l +extents /dev/vgname/lvname
mkswap /dev/vgname/lvname
swapon -va /dev/vgname/lvname
```


* P187 vgcreate -s 決定大小
* P201 extend logical volumes



### 分層storge stratisd

```
yum install stratis-cli stratisd
systemctl enable --now stratisd

stratis pool create stratispool1 /dev/vdb
stratis pool list #真正的大小
stratis pool add-data stratispool1 /dev/vdc
stratis blockdev list stratispool1
stratis filesystem create stratispool1 stratis-filesystem1

stratis filesystem list
mkdir /stratisvol

mount /stratis/stratispool1/stratis-filesystem1 /stratisvol
mount


#開機mount (超重要 要記得加)
blkid  
vim /etc/fstab
UUID="" /dir1 xfs defaults,xsystemd.requires=stratisd.service 0 0
```

### vdo
* 第三個硬碟for vdo
* /etc/fstab 一定要加入
    * defaults,x-systemd.requires=vdo.service
    * 不然重開就GG了

```
yum install vdo

#用tab查看參數 下logical才能打腫臉充胖子
vdo create --name=vdo1 --device=/dev/vdd --vdoLogicalSize=50G
vdo list
vdo status --name=vdo1 | grep -E 'Deduplication|Compression'  #查看啟動狀態
udevadm settle

#format起來
mkfs -t xfs -K /dev/mapper/vdo1 #-K不要處理衝胖子的部分

mkdir /mnt/vdo1 #建立mount路徑
mount /dev/mapper/vdo1 /mnt/vdo1
vdostats --human-readable #查看硬碟真實狀態 與省多少重複資料

#mount起來
blkid /dev/mapper/vdo1 #查看UUID
UUID="" /mnt/vdo1 xfs defaults,x-systemd.requires=vdo.service 0 0
mount -a
systemctl reboot

#建立大檔案測試
dd if=/dev/zero of=/mnt/vdo1/file bs=1M count=2000

```
![](https://i.imgur.com/dRKF248.png)




### autofs
* mount direct / indirect的目錄
* 設定路徑檔
    * /etc/auto.master.d/xxx.autofs 
    * 內容為: 目錄 設定檔
* 設定檔
    * /etc/auto.xxx
    * 內容為: ' * -rw,sync,fstype=nfs4 serverb.lab.example.com:/shares/indirect/&'
        * 全部 給讀寫權限 考試可能指定fstype
        * server與路徑
        * &為路徑下的全部

```
yum install autofs
systemctl enable --now autofs #重開機會自動啟動


#嘗試mount看看
mount -t nfs serverb.lab.example.com:/shares/direct/external /mnt
umount /mnt
mount -t nfs serverb.lab.example.com:/shares/indirect /mnt
umount /mnt

#direct
vim /etc/auto.master.d/direct.autofs
[/- /etc/auto.direct]
vim /etc/auto.direct
[/external -rw,sync,fstype=nfs4 serverb.lab.example.com:/shares/direct/external]

#indirect
vim /etc/auto.master.d/indirect.autofs
[/internal /etc/auto.indirect]  #目錄 設定檔
vim /etc/auto.indirect  #設定檔
[* -rw,sync,fstype=nfs4 serverb.lab.example.com:/shares/indirect/&]  #目錄中的全部

systemctl reboot
#用不同身分等入 查看不同目錄 做檢查

```

* Q2:建立user家目錄，讓她登入時觸發autofs
    * server端會先建好對應目錄 和user

```
yum install autofs
systemctl enable --now autofs #重開機會自動啟動


#嘗試mount看看
mount -t nfs serverb.lab.example.com:/shares/indirect /mnt
umount /mnt

#indirect
vim /etc/auto.master.d/indirect.autofs
[/rhome /etc/auto.indirect]  #目錄 設定檔
vim /etc/auto.indirect  #設定檔
[* -rw,sync,fstype=nfs4 serverb.lab.example.com:/rhome/&]  #目錄中的全部
systemctl restart autofs #df -h會呈現沒有mount

ssh remoteuser1@localhost
#成功登入就是成功了 腳踩在mount的目錄
df -h #會看到mount起來的

```
![](https://i.imgur.com/yZtGxpZ.png)


### 救援改壞/etc/fatab
* 查看那台主機console
* 在開機選畫面按e 
![](https://i.imgur.com/4kghssL.png)
* 在linux那行尾巴填上
    * systemd.unit=rescue.target
    * ctrl x 繼續開機到救援模式
![](https://i.imgur.com/JncwT4r.png)
* 登入root 修改錯誤的檔案 
    * 開機提示也會有failed的資訊 可以參考錯了哪個
* 改完按ctrl d 能成功開就OK了
### 



### 破解root密碼
![](https://i.imgur.com/Ua63wF8.png)
* 查看那台主機console
* 在開機選畫面按e 
* 在linux那行尾巴填上
    * rd.break
    * ctrl x 繼續開機
* 改密碼步驟如上圖
    * 修改mount權限為rw
    * 更改root 才有passwd可以下
    * 修改密碼
    * 記得autorelabel
* 改完按ctrl d 能成功開就OK了

```
mount #查看有哪些硬碟 找到對應的
mount | grep /sysroot #目前權限為ro
mount -o remount,rw /sysroot #修改為rw

chroot /sysroot

echo "123" | passwd --stdin root #給密碼

touch /.autorelabel #打錯可能GG
exit
exit

#要等relabel一陣子 出現登入畫面即成功
```



### 開FW port

* 開啟某個port
```
firewall-cmd --add-port=http/tcp  #立即啟動
firewall-cmd --add-port=http/tcp --permanent  #寫入設定 重啟才會生效
firewall-cmd --relaod
firewall-cmd --list-all  #列出開啟的port
```

### 調整SElinux設定
*  大大鈔 記得install
*  man 8 semanage-port

```
yum -y install selinux-policy-doc
man -k _selinux
man 8 semanage-port

semanage port -d -t gopher_port_t -p tcp 71  #刪除
semanage port -m -t http_port_t -p tcp 71  #直接修改
semanage port -l -C #查看自己加的rule

```

* 考試情境: 啟動http失敗，查詢原因 port沒加到

```
systemctl start httpd 
#出現錯誤訊息 查看對應指令
#發現http port為82沒有開 (檔案也會推薦語法)
semanage port -a -t http_port_t -p tcp 82
#or 查看man
man 8 semanage-port #搜尋example
semanage port -a -t http_port_t -p tcp 82
#查詢http type name
semanage port  -l
#加入後就可以在啟動一次看看
systemctl start httpd 

#從外面連不到 要開防火牆82 port
firewall-cmd --add-port=82/tcp
firewall-cmd --add-port=82/tcp --permanent 

```

### Container

* 下載執行一個container
```
yum module install container-tools
podman login registry.lab.example.com #登入網址
podman search [keyward] #查詢相關images
podman pull registry.lab.example.com/rhel8/httpd-24:latest  #下載

podman ps -a  #查看有哪些images (寶貝球)

#podman run 命名container 路徑 執行的程式
podman run --name myweb -it registry.lab.example.com/rhel8/httpd-24 /bin/bash
id
ps aux

#刪除
podman run --rm registry.lab.example.com/rhel8/httpd-24 httpd -v
```
    * 最後的lab練習


* Q2
```
mkdir -p ~/webcontent/html/
echo "Hello World" > ~/webcontent/html/index.html

podman login registry.lab.example.com
podman pull registry.lab.example.com/rhel8/httpd-24:1-98
podman inspect registry.lab.example.com/rhel8/httpd-24:1-98

podman run -d --name myweb -p 8000:8080 -v ~/webcontent:/var/www:Z registry.lab.example.com/rhel8/httpd-24:1-98

podman ps -a
curl http://servera:8000

firewall-cmd --add-port=8000/tcp
sudo firewall-cmd --add-port=8000/tcp --permanent



#查看有哪些版本
skopeo inspect docker://registry.lab.example.com/rhel8/httpd-24
podman run -d --name myweb -p 8000:8080 -v ~/webcontent:/var/www:Z registry.lab.example.com/rhel8/httpd-24:latest
```


* Q3:
```
mkdir -p ~/.config/containers/
~]$ cp /tmp/containers-services/registries.conf ~/.config/containers/
podman search ubi
mkdir -p ~/webcontent/html/
echo "Hello World" > ~/webcontent/html/index.html

sudo yum module install container-tools
podman login registry.lab.example.com
podman run -d --name myweb -p 8000:8080 -v ~/webcontent:/var/www:Z registry.lab.example.com/rhel8/httpd-24:1-105
curl http://localhost:8000/

mkdir -p ~/.config/systemd/user/
cd ~/.config/systemd/user
podman generate systemd --name myweb --files --new

podman stop myweb
podman rm myweb
systemctl --user daemon-reload

systemctl --user enable --now container-myweb
loginctl enable-linger
loginctl show-user contsvc


```

* Q4
```
mkdir /home/podsvc/db_data
chmod 777 /home/podsvc/db_data

sudo yum module install container-tools
podman login registry.lab.example.com
podman pull registry.lab.example.com/rhel8/mariadb-103:1-86
#查詢參數 要加root passwd
podman inspect egistry.lab.example.com/rhel8/mariadb-103:1-86


podman run -d --name inventorydb -p 13306:3306 -v /
home/podsvc/db_data:/var/lib/mysql/data:Z -e MYSQL_USER=operator1 -e
MYSQL_PASSWORD=redhat -e MYSQL_DATABASE=inventory -e MYSQL_ROOT_PASSWORD=redhat
registry.lab.example.com/rhel8/mariadb-103:1-86

#test db
mysql -u user1 -p --port=3306 --host=127.0.0.1

#登入自動啟動
mkdir -p ~/.config/systemd/user/
cd ~/.config/systemd/user/
podman generate systemd --name inventorydb --files --new

podman stop inventorydb
podman rm inventorydb
systemctl --user daemon-reload
systemctl --user enable --now container-inventorydb.service
```

## 第一冊
兩台機器，一台有root密碼但IP錯
一台要重設root密碼

### 1.修改IP
* 用nmcli
* 有成功後就用遠端ssh看看
![](https://i.imgur.com/yXwuLoP.png)
![](https://i.imgur.com/WfSnlpv.png)


```
#先建立一個空檔案 名稱和name一樣
nmcli connection show  #查看name
cd /etc/sysconfig/netwok-scripts/
touch /etc/sysconfig/netwok-scripts/ifcfg-Wired_connection_1

nmcli connection modify “Wired connection 1” ipv4.address 172.25.250.10/24
#做完就會自動產生檔案內容
nmcli connection reload
nmcli connection up “Wired connection 1” 


nmcli device show #查看相關資訊
```


![](https://i.imgur.com/xd3SdCG.png)
![](https://i.imgur.com/6JnW8F1.png)
![](https://i.imgur.com/srwAPNZ.png)
![](https://i.imgur.com/DV1QeoW.png) 


### 2.連到yum

* 檔案要能無中生有QAQ 考前五分鐘再記一次
* cd /etc/yum.repos.d/XXX.repo
* baseurl = 題目給的
* enabled = true
* gpgcheck = false
* name = 命名

![](https://i.imgur.com/pPF7mFN.png)
![](https://i.imgur.com/XwO2X3c.png)
![](https://i.imgur.com/zEYLemM.png)

### 3.建立群組與帳號
* 附屬群組 -G
* 沒有權限ssh -s /sbin/nologin
* 給passwd | passwd --stdin user1

```
groupadd manager
useradd user1 -G manager
useradd user2 -G manager
useradd user3 -s /sbin/nologin
useradd -u 8888 linda
echo Passwd | passwd --stdin user1
```
![](https://i.imgur.com/qlsgLw0.png)
![](https://i.imgur.com/Xgz7vR5.png)

### 4.修改group權限
執行sudo不須輸入密碼
* visudo
* 第110行
* 測試 sudo -i (切root)

![](https://i.imgur.com/BKadoo3.png)

![](https://i.imgur.com/olLDdmI.png)



### 5.建立目錄給群組權限
* mkdir [目錄]
* chgrp manager [目錄]
* chmod 給group全縣 減去other全縣
* chmod g+s [目錄] (目錄中新建的檔案將屬於該group) 
![](https://i.imgur.com/sjXg2yT.png)
![](https://i.imgur.com/IkuD0J7.png)


### 6.NTP對時
![](https://i.imgur.com/4ICdqqK.png)
* /etc/chrony.conf
* systemctl restart chronyd.service
* 註解掉原本的，加上要求的
* 等一陣子後再檢查 確認與對時區
    * timedatectl
    * timedatectl set-timezone Asia/Taipei
* 忘記的話可以grep -r ntp /etc找檔案名稱
* 重啟忘記的話查詢 systemctl --all | grep chrony 

![](https://i.imgur.com/4o9GAoJ.png)

![](https://i.imgur.com/jDiQY5c.png)




### 7.找檔案複製到指定目錄(需保留原設定)
![](https://i.imgur.com/9loGqgL.png)
* find / -user linda (先找到檔案
* mkdir /root/linda
* find / -user linda -exec cp -a {} /root/linda \ ;
    * {} 代表被找到的那些檔案
    * -a 保存owner權限 (真的忘記的一格一個cp 改chown
    * \; 代表結束

![](https://i.imgur.com/Fi6Dr6H.png)



### 8.加壓縮
![](https://i.imgur.com/4sSxSBn.png)

* tar -czf /root/backup.tar.gz /usr/local
    * -cf: create file
    * -z: gzip; -j: bzip; -J: xz
* tar -tzf /root/backup.tar.gz (檢查有沒有全加壓縮)
    * -t 查看

![](https://i.imgur.com/DFXBujE.png)

### 9.檔案權限遮罩

![](https://i.imgur.com/O9iLSPL.png)

* umask
    * 遮掉不要的 
    * 預設700 設定為077
* 編輯.bashrc才會重開機後仍有效
![](https://i.imgur.com/B6Msowg.png)
![](https://i.imgur.com/F9HKSFs.png)


### 10.

![](https://i.imgur.com/GFaJvEL.png)
* find /usr -size +30k -size -50k -perm -4000
* 大於30小於50 

![](https://i.imgur.com/5zhW8rO.png)



 


## 不一定會考

### priority
* P73
* 權限 -20 - +20
```
nice -n 15 sha1sum &
sha1sum /dev/zero & #無限迴圈程式
renice -n 19 3521
```

### 放大LVM
* pvmove 換硬碟















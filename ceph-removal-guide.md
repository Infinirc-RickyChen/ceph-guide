# 相關問題
- 如何從Proxmox VE集群移除/刪除Ceph
- 如何在Proxmox VE集群重新安裝Ceph

# 問題描述
我們想要完全從PVE移除Ceph，或是移除後重新安裝。

# 解決方案

## 1. 移除/刪除Ceph
警告：移除/刪除Ceph將會同時移除/刪除所有儲存在Ceph上的資料！

### 1.1 登入Proxmox Web GUI

### 1.2 點擊其中一個PVE節點

### 1.3 從右側面板，導航到Ceph -> Pools，記錄Name下的項目

### 1.4 導航到Ceph -> CephFS，記錄現有的CephFS名稱

### 1.5 從左側選單，點擊Datacenter

### 1.6 從右側，點擊Storage

### 1.7 刪除在步驟3和步驟4中看到的所有CephFS和Pools項目

### 1.8 從右側面板，點擊主節點，導航到Ceph -> CephFS，停止並銷毀所有Metadata服務

### 1.9 點擊主節點，從右側面板，導航到Ceph -> OSD，將所有OSD標記為Out

### 1.10 從右側選單，右鍵點擊其中一個PVE節點名稱，點擊>_ Shell開啟終端機

### 1.11 標記OSD為down狀態
```bash
ceph osd down <osdid>

# 例如
ceph osd down 0
ceph osd down 1
ceph osd down 2
```

### 1.12 點擊主節點，從右側面板，導航到Ceph -> OSD，點擊要移除的OSD，點擊右上角的More按鈕，點擊Destroy

或者

我們可以用一個指令將它們標記為down並銷毀
```bash
ceph osd down 0 && ceph osd destroy 0 --force
ceph osd down 1 && ceph osd destroy 1 --force
ceph osd down 2 && ceph osd destroy 2 --force
```

### 1.13 從終端機執行以下指令移除ceph配置檔案（參考步驟10）
```bash
rm /etc/ceph/ceph.conf
```

### 1.14 在每個PVE節點上執行以下指令停止ceph監視器服務
```bash
systemctl stop ceph-mon@<hostname or monid>

# 例如
systemctl stop ceph-mon@labnode1
```
注意：如果不確定@後應該使用什麼hostname或monid，我們可以輸入指令"systemctl stop ceph-mon@"然後按一下Tab鍵，正確的值會自動補完，記得記下補完的值，我們會在下一步（步驟15）需要用到。

### 1.15 停用ceph監視器服務，在每個節點上執行以下指令
```bash
systemctl disable ceph-mon@<hostname or monid>

# 例如
systemctl disable ceph-mon@labnode1
```
注意：在"@"後加上我們從步驟14得到的值

如果我們看到類似下面的內容，表示我們已成功執行正確的指令：
```
Removed /etc/systemd/system/ceph-mon.target.wants/ceph-mon@labnode1.service.
```

### 1.16 從所有節點移除ceph配置檔案
```bash
rm -r /etc/pve/ceph.conf
rm -r /etc/ceph
rm -rf /var/lib/ceph
```

如果我們得到以下或類似的錯誤：
```
rm: cannot remove '/var/lib/ceph/osd/ceph-0': Device or resource busy
```
我們可以使用這個指令先卸載，然後再嘗試移除：
```bash
umount /var/lib/ceph/osd/ceph-0
rm -r /var/lib/ceph
```

### 1.17 重新啟動所有PVE節點

### 1.18 清除剩餘的ceph配置檔案和服務，在每個節點上執行以下指令
```bash
pveceph purge
```

### 1.19 從每個OSD節點清除OSD磁碟，這樣我們之後可以使用這些磁碟
```bash
# 從磁碟移除lvm簽名
# 注意：根據情況更改磁碟代號（sdx）
fdisk /dev/sdx
```
然後輸入d，按Enter鍵，輸入w，按Enter鍵

```bash
rm -r /dev/ceph-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxxx
```
注意：要找到正確的值，我們可以使用指令"ls /dev | grep ceph"或只需輸入"rm -r /dev/ceph-"然後按Tab鍵自動補完剩餘部分

```bash
# 重新啟動PVE主機
reboot
```

### 1.20 如果需要，移除所有ceph套件
```bash
apt purge ceph-mon ceph-osd ceph-mgr ceph-mds
```

如果需要移除更多：
```bash
apt purge ceph-base ceph-mgr-modules-core
```

## 2. 重新安裝Ceph
重新安裝過程與新的/全新的ceph安裝幾乎相同（使用PVE web gui非常容易），但是，如果在建立OSD期間遇到以下錯誤，這是由於之前安裝留下的殘餘造成：

```
error during cfs-locked 'file-ceph_conf' operation: command 'cp /etc/pve/priv/ceph.client.admin.keyring /etc/ceph/ceph.client.admin.keyring' failed: exit code 1 (500)
```
或
```
unable to open file '/var/lib/ceph/bootstrap-osd/ceph.keyring.tmp.1694' - No such file or directory (500)
```
或
```
unable to open file '/var/lib/ceph/bootstrap-osd/ceph.keyring.tmp.920' - No such file or directory (500)
```

要修復這個問題，我們可以使用以下方法（建立缺失的目錄"/var/lib/ceph/bootstrap-osd/"）：
```bash
mkdir /var/lib/ceph/bootstrap-osd
```

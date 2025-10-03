# HW2 report

---

# HW2-1 RAID

---

## 1. 掛載映像為 loop 裝置
```bash
losetup -a
```
輸出：
```text
/dev/loop1: []: (/home/classuser/disk2.img)
/dev/loop2: []: (/home/classuser/disk3.img)
/dev/loop0: []: (/home/classuser/disk1.img)
/dev/loop3: []: (/home/classuser/disk4.img)
```

---

## 2. 檢查各裝置的 md superblock
### /dev/loop0
```bash
sudo mdadm --examine /dev/loop0
```
輸出（節錄重點）：
```text
/dev/loop0:
          Magic : a92b4efc
        Version : 1.2
     Array UUID : b908d87f:bfd47d06:8174af4e:b6522d58
     Raid Level : raid5
   Raid Devices : 3
          State : clean
         Events : 18
   Device Role : Active device 0
   Array State : AAA ('A' == active, '.' == missing, 'R' == replacing)
```

### /dev/loop1
```bash
sudo mdadm --examine /dev/loop1
```
輸出：
```text
mdadm: No md superblock detected on /dev/loop1.
```

### /dev/loop2
```bash
sudo mdadm --examine /dev/loop2
```
輸出（節錄重點）：
```text
/dev/loop2:
          Magic : a92b4efc
        Version : 1.2
     Array UUID : b908d87f:bfd47d06:8174af4e:b6522d58
     Raid Level : raid5
   Raid Devices : 3
          State : clean
         Events : 18
   Device Role : Active device 1
   Array State : AAA ('A' == active, '.' == missing, 'R' == replacing)
```

### /dev/loop3
```bash
sudo mdadm --examine /dev/loop3
```
輸出（節錄重點）：
```text
/dev/loop3:
          Magic : a92b4efc
        Version : 1.2
     Array UUID : b908d87f:bfd47d06:8174af4e:b6522d58
     Raid Level : raid5
   Raid Devices : 3
          State : clean
         Events : 18
   Device Role : Active device 2
   Array State : AAA ('A' == active, '.' == missing, 'R' == replacing)
```

> 判定：`/dev/loop1`（對應 `disk2.img`）無 md superblock → 壞碟。其餘三顆屬於同一個陣列（UUID 相同、Events=18）。

---

## 3. 嘗試 assemble 時的衝突（紀錄）
```bash
sudo mdadm --assemble --run /dev/md0 /dev/loop0 /dev/loop2 /dev/loop3
```
輸出（重點）：
```text
mdadm: Fail to create md0 when using /sys/module/md_mod/parameters/new_array, fallback to creation via node
mdadm: /dev/md0 is already in use.
```

> 分析：系統已有一組 md0（非我們要救的那組），因此改用 **/dev/md/2** 名稱避免衝突。

---

## 4. 掃描現有陣列資訊
```bash
sudo mdadm --examine --scan
```
輸出：
```text
ARRAY /dev/md/2  metadata=1.2 UUID=b908d87f:bfd47d06:8174af4e:b6522d58
ARRAY /dev/md/0  metadata=1.2 UUID=82a46d13:93f24cc0:785c3ae7:239154db
```

---

## 5. 正確 assemble 指令（以 /dev/md/2 名稱）
```bash
sudo mkdir -p /dev/md
sudo mdadm --assemble --run /dev/md/2 /dev/loop0 /dev/loop2 /dev/loop3
```
輸出：
```text
mdadm: /dev/md/2 has been started with 3 drives.
```

---

## 6. 查看陣列狀態
```bash
cat /proc/mdstat
```
輸出：
```text
Personalities : [raid6] [raid5] [raid4] [raid0] [raid1] [raid10] 
md2 : active raid5 loop0[0] loop3[3] loop2[1]
      405504 blocks super 1.2 level 5, 512k chunk, algorithm 2 [3/3] [UUU]

md0 : active raid5 vdc[3] vdb2[1] vdb1[0]
      405504 blocks super 1.2 level 5, 512k chunk, algorithm 2 [3/3] [UUU]

unused devices: <none>
```

```bash
sudo mdadm --detail /dev/md/2
```
輸出（節錄重點）：
```text
/dev/md/2:
        Raid Level : raid5
        Array Size : 405504 (396.00 MiB 415.24 MB)
      Raid Devices : 3
             State : clean 
   Active Devices : 3
            Layout : left-symmetric
        Chunk Size : 512K
              Name : vm01:2
              UUID : b908d87f:bfd47d06:8174af4e:b6522d58
            Events : 18

    Number   Major   Minor   RaidDevice State
       0       7        0        0      active sync   /dev/loop0
       1       7        2        1      active sync   /dev/loop2
       3       7        3        2      active sync   /dev/loop3
```

---

## 7. 確認檔案系統並掛載
```bash
sudo blkid /dev/md/2
```
輸出：
```text
/dev/md/2: UUID="33b2b248-b29d-4b00-9dcd-5cfa6bd11b78" BLOCK_SIZE="4096" TYPE="ext4"
```

```bash
sudo mkdir -p /mnt/raid
sudo mount /dev/md/2 /mnt/raid
ls -lah /mnt/raid
```
輸出：
```text
total 28K
drwxr-xr-x 3 root root 4.0K Sep 22 06:36 .
drwxr-xr-x 5 root root 4.0K Oct  2 10:01 ..
drwx------ 2 root root  16K Sep 22 06:33 lost+found
-rw-r--r-- 1 root root  194 Sep 22 06:36 treasure.txt
```

```bash
find /mnt/raid -maxdepth 3 -type f -printf "%p\t%k KB\n"
```
輸出：
```text
/mnt/raid/treasure.txt  4 KB
find: ‘/mnt/raid/lost+found’: Permission denied
```

> 備註：`lost+found` 是 ext 檔案系統的系統目錄，權限限制導致 `Permission denied` 屬正常行為。

---

## 8. 讀取 secret code
```bash
sudo cat /mnt/raid/treasure.txt
```
輸出：
```text
Congradulations! You've found the treasure of this homework!
The secret code is {IL0V3L5AP_HAPPYHOLIDAY9/28}
Remember to write the secret code down in your report so that I know you found it :)
```

> **Secret code：`{IL0V3L5AP_HAPPYHOLIDAY9/28}`**。

---

## 9. 收尾（清理環境）
```bash
sudo umount /mnt/raid
sudo mdadm --stop /dev/md/2
sudo losetup -d /dev/loop0 /dev/loop1 /dev/loop2 /dev/loop3
```

---

## 10. 結論
- 壞碟：`disk2.img`（/dev/loop1 無 md superblock）。
- 使用 `/dev/loop0 /dev/loop2 /dev/loop3` 成功組回 RAID5（/dev/md/2）。
- 檔案系統為 ext4，成功讀到 `treasure.txt` 並取得 secret code `{IL0V3L5AP_HAPPYHOLIDAY9/28}`。

---

# HW2-2 NFS + autofs

---

## A. Server 端

### A-1. 安裝與啟動 NFS 服務
```bash
sudo apt update
sudo apt install -y nfs-kernel-server
sudo systemctl enable --now nfs-kernel-server
```
重點輸出（節錄）：
```text
nfs-kernel-server is already the newest version (1:2.6.4-3ubuntu5.1).
... enable nfs-kernel-server
```

### A-2. 建立目錄樹與示例檔
```bash
sudo mkdir -p /srv/nfsroot/shared/public
sudo mkdir -p /srv/nfsroot/shared/projects/group12
sudo mkdir -p /srv/nfsroot/shared/projects/HW2
sudo mkdir -p /srv/nfsroot/users/{alice,bob,charlie}
sudo mkdir -p /srv/nfsroot/services/{webdata,backups}

echo "This is README."        | sudo tee /srv/nfsroot/shared/public/README.txt >/dev/null
echo "# Task for group"       | sudo tee /srv/nfsroot/shared/projects/group12/task.md >/dev/null
echo "- [ ] Do HW2"           | sudo tee /srv/nfsroot/shared/projects/HW2/todo.txt >/dev/null
echo "Alice notes"            | sudo tee /srv/nfsroot/users/alice/notes.txt >/dev/null
echo "Bob plan"               | sudo tee /srv/nfsroot/users/bob/plan.txt >/dev/null
echo "Charlie secret"         | sudo tee /srv/nfsroot/users/charlie/secret.txt >/dev/null
echo "<h1>Webdata</h1>"       | sudo tee /srv/nfsroot/services/webdata/index.html >/dev/null
echo "Nightly snapshot"       | sudo tee /srv/nfsroot/services/backups/snapshot.txt >/dev/null
```

### A-3. 匯出設定（NFSv4 pseudo-root）
`/etc/exports` 內容：
```exports
/srv/nfsroot              *(rw,sync,fsid=0,crossmnt,no_subtree_check)
/srv/nfsroot/shared       *(rw,sync,no_subtree_check)
/srv/nfsroot/users        *(rw,sync,no_subtree_check)
/srv/nfsroot/services     *(rw,sync,no_subtree_check)
```
套用與驗證：
```bash
sudo exportfs -ra
sudo systemctl restart nfs-kernel-server
sudo exportfs -v
```
輸出（節錄重點）：
```text
/srv/nfsroot    <world>(sync, ..., crossmnt, ..., fsid=0, ..., rw, ..., root_squash, ...)
/srv/nfsroot/shared    <world>(sync, ..., rw, ..., root_squash, ...)
/srv/nfsroot/users     <world>(sync, ..., rw, ..., root_squash, ...)
/srv/nfsroot/services  <world>(sync, ..., rw, ..., root_squash, ...)
```

---

## B. Client 端（同機）

### B-1. 安裝 autofs 與 nfs-common
```bash
sudo apt install -y nfs-common autofs
```
輸出（節錄）：
```text
The following NEW packages will be installed:
  autofs libnsl2
...
Setting up autofs (5.1.9-1ubuntu4.1) ...
Created symlink ... autofs.service → ...
```

查詢本機 IP 作為 `SERVER_IP`：
```bash
hostname -I
```
輸出：
```text
10.0.2.15 fec0::ff:fe00:1
```

### B-2. 設定 master map（宣告 indirect 與 direct）
```bash
echo "/mnt/nfs  /etc/auto.nfs    --timeout=60 --ghost" | sudo tee -a /etc/auto.master
echo "/-        /etc/auto.direct --timeout=60 --ghost" | sudo tee -a /etc/auto.master
sudo mkdir -p /mnt/nfs
```

### B-3. Indirect map：`/etc/auto.nfs`（掛在 `/mnt/nfs/*`）
```bash
SERVER_IP=10.0.2.15
sudo bash -c "cat > /etc/auto.nfs" <<EOF
public     -fstype=nfs4  ${SERVER_IP}:/shared/public
projects   -fstype=nfs4  ${SERVER_IP}:/shared/projects
alice      -fstype=nfs4  ${SERVER_IP}:/users/alice
bob        -fstype=nfs4  ${SERVER_IP}:/users/bob
charlie    -fstype=nfs4  ${SERVER_IP}:/users/charlie
EOF
```

### B-4. Direct map：`/etc/auto.direct`（兩個直掛路徑）
```bash
sudo bash -c "cat > /etc/auto.direct" <<EOF
/webdata   -fstype=nfs4  ${SERVER_IP}:/services/webdata
/backups   -fstype=nfs4  ${SERVER_IP}:/services/backups
EOF
```

### B-5. 套用 autofs 設定
```bash
sudo /etc/init.d/autofs reload
```
輸出：
```text
Reloading autofs configuration (via systemctl): autofs.service.
```

---

## C. 驗證

### C-1. 觸發掛載並列出檔案
```bash
# 依序觸發並列出
ls -l /mnt/nfs/public
ls -l /mnt/nfs/projects/group12
ls -l /mnt/nfs/projects/HW2
ls -l /mnt/nfs/alice
ls -l /webdata
ls -l /backups
```
輸出（節錄重點）：
```text
/mnt/nfs/public:
-rw-r--r-- 1 root root 16 Oct  3 03:53 README.txt

/mnt/nfs/projects/group12:
-rw-r--r-- 1 root root 17 Oct  3 03:53 task.md

/mnt/nfs/projects/HW2:
-rw-r--r-- 1 root root 13 Oct  3 03:53 todo.txt

/mnt/nfs/alice:
-rw-r--r-- 1 root root 12 Oct  3 03:53 notes.txt

/webdata:
-rw-r--r-- 1 root root 17 Oct  3 03:53 index.html

/backups:
-rw-r--r-- 1 root root 17 Oct  3 03:53 snapshot.txt
```

### C-2. 證明 autofs 與 NFSv4 掛載已生效
```bash
mount | grep -E 'autofs|nfs4'
systemctl status autofs --no-pager
```
輸出（節錄重點）：
```text
/etc/auto.nfs on /mnt/nfs type autofs (...,indirect,...)
/etc/auto.direct on /webdata type autofs (...,direct,...)
/etc/auto.direct on /backups type autofs (...,direct,...)

10.0.2.15:/shared/public   on /mnt/nfs/public   type nfs4 (...,vers=4.2,...)
10.0.2.15:/shared/projects on /mnt/nfs/projects type nfs4 (...,vers=4.2,...)
10.0.2.15:/users/alice     on /mnt/nfs/alice    type nfs4 (...,vers=4.2,...)
10.0.2.15:/services/webdata on /webdata         type nfs4 (...,vers=4.2,...)
10.0.2.15:/services/backups on /backups         type nfs4 (...,vers=4.2,...)

● autofs.service - Automounts filesystems on demand
Active: active (running) since Fri 2025-10-03 ...
```

---

## D. 結論與備註
- NFSv4 pseudo-root 以 `/srv/nfsroot` 為 `fsid=0`，`shared/ users/ services/` 均成功匯出。
- autofs 已在 indirect `/mnt/nfs/*` 與 direct `/webdata`、`/backups` 正常觸發掛載。
- 權限：預設 `root_squash` 啟用；如需在 client 端測寫，請調整 server 端檔案/目錄擁有者與權限，或設定 `anonuid/anongid`。

---

## E. 附錄（快速清理 / 重新載入）
```bash
# 重新載入 autofs（修改 map 後）
sudo /etc/init.d/autofs reload

# 觀察 autofs 與 nfs 掛載
mount | grep -E 'autofs|nfs4'

# （選用）清理掛載：離開目錄等待 timeout，或手動 lazy umount
sudo umount -l /mnt/nfs/public /mnt/nfs/projects /mnt/nfs/alice /webdata /backups 2>/dev/null || true
```

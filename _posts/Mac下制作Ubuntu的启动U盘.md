# Mac下制作Ubuntu的启动U盘

## 1.用hdiutil将ISO转dmg

cd ubuntu镜像文件所在的路径

```bash
hdiutil convert -format UDRW -o ubuntu-20.04-desktop-amd64.dmg   ubuntu-20.04-desktop-amd64.iso
```

## 2.查看U盘的序号

```bash
diskutil list
```

下面用diskN代替你的U盘序号

## 3.卸载U盘

```bash
diskutil unmountDisk /dev/diskN
```

## 4.将镜像写入U盘

```bash
sudo dd if=./ubuntu.img.dmg of=/dev/rdiskN bs=1m 
 ```

## 5.弹出U盘

``` bash
diskutil eject /dev/diskN
```

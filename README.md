# OpenStick 镜像构建器
针对基于 **MSM8916** 的 4G 调制解调器 USB 设备的镜像构建工具。

该构建器使用 [postmarketOS](https://postmarketos.org/) 提供的 MSM8916 设备预编译 [内核](https://pkgs.postmarketos.org/package/v24.06/postmarketos/aarch64/linux-postmarketos-qcom-msm8916)。

> ⚠️ **注意**  
> 当前分支生成的是 `debian` 镜像，若需要生成 `alpine` 镜像，请切换到 [alpine 分支](https://github.com/kinsamanka/OpenStick-Builder/tree/alpine)。

---

## 构建指南

### 本地构建
已在 **Ubuntu 22.04** 上测试通过。

1. 克隆仓库：
```bash
git clone --recurse-submodules https://github.com/G-huohuo/OpenStick-Builder.git
cd OpenStick-Builder/
````

#### 快速构建

```bash
sudo ./build.sh
```

#### 详细构建步骤

1. 安装依赖：

```bash
sudo scripts/install_deps.sh
```

2. 构建自定义 bootloader（hyp 和 lk2nd），支持 `extlinux.conf` 文件：

```bash
sudo scripts/build_hyp_aboot.sh
```

3. 提取 Qualcomm 固件，并创建利用全 eMMC 空间的新分区表：

```bash
sudo scripts/extract_fw.sh
```

4. 使用 debootstrap 创建 rootfs：

```bash
sudo scripts/debootstrap.sh
```

5. 构建 gadget-tools：

```bash
sudo scripts/build_gt.sh
```

6. 创建镜像文件：

```bash
sudo scripts/build_images.sh
```

生成的固件文件会存放在 `files` 目录下。

---

### 使用 GitHub Actions 云端构建

1. Fork 仓库
2. 运行 [Build workflow](../../actions/workflows/build.yml)：

   * 点击 **Run workflow**
   * 构建完成后，下载生成的 artifact

---

## 自定义

可编辑 [`scripts/setup.sh`](scripts/setup.sh) 添加或移除软件包。该脚本在 `chroot` 环境中运行。

---

## 固件安装

> ⚠️ **警告**
> 以下操作可能导致设备无法启动，请谨慎操作并自行承担风险。

> ⚠️ **重要**
> 请使用命令 `edl rf orig_fw.bin` 备份原固件。

### 前置条件

* [EDL 工具](https://github.com/bkerler/edl)
* Android fastboot 工具：

```bash
sudo apt install fastboot
```

### 安装步骤

1. 使用 [指南](https://wiki.postmarketos.org/wiki/Zhihe_series_LTE_dongles_%28generic-zhihe%29#How_to_enter_flash_mode) 进入 Qualcomm EDL 模式
2. 备份所需分区（若已有可跳过）：

```bash
for n in fsc fsg modem modemst1 modemst2 persist sec; do
    edl r ${n} ${n}.bin
done
```

3. 安装 aboot：

```bash
edl w aboot aboot.mbn
```

4. 重启到 fastboot：

```bash
edl e boot
edl reset
```

5. 刷写固件：

```bash
fastboot flash partition gpt_both0.bin
fastboot flash aboot aboot.mbn
fastboot flash hyp hyp.mbn
fastboot flash rpm rpm.mbn
fastboot flash sbl1 sbl1.mbn
fastboot flash tz tz.mbn
fastboot flash boot boot.bin
fastboot flash rootfs rootfs.bin
```

6. 恢复原始分区：

```bash
for n in fsc fsg modem modemst1 modemst2 persist sec; do
    fastboot flash ${n} ${n}.bin
done
```

7. 重启：

```bash
fastboot reboot
```

---

## 安装后配置

* 网络配置：

| 接口    | 配置                                                            |
| ----- | ------------------------------------------------------------- |
| wlan0 | ssid: Openstick <br> password: openstick <br> IP: 192.168.4.1 |
| usb0  | IP: 192.168.5.1                                               |

* 默认用户：

| 用户名  | 密码 |
| ---- | -- |
| user | 1  |

* 若设备非 **UZ801**，修改 `/boot/extlinux/extlinux.conf` 以使用正确的 devicetree：

```bash
sed -i 's/yiming-uz801v3/<BOARD>/' /boot/extlinux/extlinux.conf
```

可选 `<BOARD>`：

* `thwc-uf896` → UF896

* `thwc-ufi001c` → UFIxxx

* `jz01-45-v33` → JZxxx

* `fy-mf800` → MF800

* 扩展 rootfs 分区：

```bash
resize2fs /dev/disk/by-partlabel/rootfs
```

* 更新 `debian` 镜像内核：

```bash
wget -O - http://mirror.postmarketos.org/postmarketos/<branch>/aarch64/linux-postmarketos-qcom-msm8916-<version>.apk \
    | tar xkzf - -C / --exclude=.PKGINFO --exclude=.SIGN* 2>/dev/null
```

请替换 `<branch>` 和 `<version>` 为对应分支和版本。

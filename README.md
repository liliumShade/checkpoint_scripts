# RISC-V SimPoint Checkpoint 生成工具

自动生成 RISC-V SimPoint Checkpoint 的工具集，支持 SPEC CPU2006/2017 的 Profiling → Cluster → Checkpoint 三阶段流程。

> 详细文档见 [doc/](doc/) 目录：
> - [架构概述](doc/architecture.md) — 三阶段流程、数据流、核心模块
> - [配置参考](doc/config-reference.md) — 所有配置字段说明、环境变量
> - [Speed/Rate 模式](doc/speed-rate-mode.md) — 两种测试模式的区别与切换

## 环境准备

### 可以访问公共服务器
- 请执行
```bash
source /nfs/home/share/workload_env/env.sh
```

### 无法访问公共服务器
- 预先准备好 riscv64 工具链，可能用到的 prefix 有`riscv64-linux-gnu-`，`riscv64-unknown-linux-gnu-`，`riscv64-unknown-elf-`，需要的 gcc 版本最低应该是 14.0.0

- 克隆或下载 OpenSBI，Linux，nemu_board，NEMU，QEMU，LibCheckpoint，LibCheckpointAlpha，riscv-rootfs
    - https://github.com/riscv-software-src/opensbi.git
    - https://cdn.kernel.org/pub/linux/kernel/v6.x/linux-6.10.3.tar.xz
    - https://github.com/OpenXiangShan/nemu_board.git
    - https://github.com/OpenXiangShan/NEMU.git
    - https://github.com/OpenXiangShan/qemu.git checkpoint分支
    - https://github.com/OpenXiangShan/LibCheckpoint.git
    - https://github.com/OpenXiangShan/LibCheckpointAlpha.git
    - https://github.com/OpenXiangShan/riscv-rootfs.git

- 准备 Linux kernel
    - 解压缩内核 `tar -xf linux-6.10.3.tar.xz`
    - 复制配置文件 `cp /path/to/nemu_board/config/xiangshan_defconfig /path/to/linux/arch/riscv/config/`
    - 在 Linux kernel 目录下调整配置文件并保存为默认配置文件 `make menuconfig; make savedefconfig; mv defconfig arch/riscv/config/xiangshan_defconfig`
- 准备设备树
    - 构建单核多核设备树 `cd /path/to/nemu_board/dts && ./build_dual_core_for_qemu.sh && ./build_single_core_for_nemu.sh`

- 构建 NEMU
    - 进入 NEMU 目录
    - 拉取 submodule `git submodule update --init`
    - 使用 `riscv64-xs-cpt_defconfig` 配置 NEMU `make riscv64-xs-cpt_defconfig`
    - 构建 NEMU `make -j`
    - 构建 simpoint `cd /path/to/NEMU/resource/simpoint/simpoint_repo && make -j`

- 构建 QEMU
    - 进入 QEMU 目录
    - 配置 QEMU `mkdir build && cd build && ../configure --target-list=riscv64-softmmu --enable-debug --enable-zstd --enable-plugins`
    - 构建 QEMU `make -j`

- 准备 riscv-rootfs
    - 进入 riscv-rootfs 目录
    - 构建 riscv-rootfs app `make install`
    - 进入 riscv-rootfs/rootfsimg 目录，修改 `inittab-spec`
    ```
    -/dev/console::sysinit:-/bin/sh /spec/run.sh
    +/dev/console::sysinit:-/bin/sh /spec0/run.sh
    ```


- 准备运行时所需文件的目录
    - 提前运行一遍 SPEC2006 和 SPEC2017 （根据自己需要可以仅运行其中一个）
    - 创建目录 `mkdir cpu2006_run_dir` 和 `mkdir cpu2017_run_dir`
    - 将运行结果所在的目录拷贝到`cpu2006_run_dir`中，例如 `cp cpu2006v99/benchspec/CPU2006/410.perlbench/run/run_base_ref_amd64-m64-gcc42-05.0001 cpu2006_run_dir/perlbench -r`
    - 对所有子项都这样操作，然后设置环境变量 `export CPU2006_RUN_DIR=/path/to/cpu2006_run_dir` 和 `export CPU2017_RUN_DIR=/path/to/cpu2017_run_dir`

- 导出环境变量
```bash
export ARCH=riscv
export LINUX_HOME=/path/to/linux
export OPENSBI_HOME=/path/to/opensbi
export XIANGSHAN_FDT=/path/to/nemu_board/dts/build/xiangshan_dualcore.dtb
export RISCV=/path/to/riscv-toolchain
export RISCV_ROOTFS_HOME=/path/to/riscv-rootfs
export CPU2006_RUN_DIR=/path/to/cpu2006_run_dir
export CPU2017_RUN_DIR=/path/to/cpu2017_run_dir
export CROSS_COMPILE=/path/to/riscv-toolchains/bin/riscv64-unknown-linux-gnu-
export GCPT_HOME=/path/to/LibCheckpoint
export NEMU_HOME=/path/to/NEMU
export QEMU_HOME=/path/to/qemu
```

### 单核检查点

- 修改环境变量
    - 公共服务器
        - `export XIANGSHAN_FDT=/nfs/home/share/workload_env/workload_build_env/dts/build/xiangshan.dtb`
        - `export GCPT_HOME=/nfs/home/share/workload_env/LibCheckpointAlpha`
    - 私有环境
        - `export XIANGSHAN_FDT=/path/to/nemu_board/dts/build/xiangshan.dtb`
        - `export GCPT_HOME=/path/to/LibCheckpointAlpha`
- 修改下述配置文件
    - 修改字段 `copies` 为 1

### 多核检查点
- 在导入环境变量之后仅需按照下述说明修改配置文件，并保证 `copies` 字段大于 1 即可（目前的环境下请保证该字段小于 4 ）

## 使用说明

### 1. 克隆仓库

```bash
git clone https://github.com/xyyy1420/checkpoint_scripts.git
cd checkpoint_scripts/checkpoint_scripts
```

### 2. 修改配置文件

编辑 `config.yaml`，完整字段说明：

```yaml
base_config:
  message: "NULL"                 # 通常为空
  spec_app_list: null             # 列表文件路径（.lst），每行一个子项名，优先级高于 spec_apps
  spec_apps: "mcf,omnetpp"       # 逗号分隔的子项名
  elf_folder: "./jemalloc_elf"    # SPEC ELF 二进制文件目录
  times: "1,1,1"                  # profiling,cluster,checkpoint 各阶段运行次数
  start_id: "0,0,0"              # 各阶段结果保存路径的起始 id
  emulator: "NEMU"                # "NEMU" 或 "QEMU"
  build_bbl_only: false           # 仅构建 workload，不执行三阶段
  max_threads: 70                 # 最大并行线程数
  mode: "speed"                   # "speed" 或 "rate"，默认 "speed"
  CPU2017: true                   # true=SPEC2017, false=SPEC2006
  generate_rootfs_script_only: false  # 仅生成 rootfs 脚本后停止
  copies: 1                       # 1=单核(kernel @ 0x80200000), 2~4=多核(kernel @ 0x80800000)
  archive_id: null                # 指定已有 archive_id，跳过构建直接执行三阶段
  redirect_output: false          # 重定向子项输出到 out.log/err.log
  cpu_bind: 0                     # （无效）
  mem_bind: 0                     # （无效）
  bootloader: "opensbi"           # "opensbi"（推荐）或 "riscv-pk"
  all_in_one_workload: true       # 使用 gcpt 链接 workload，QEMU 时必须为 true
  boot_for_test: true             # 构建后用模拟器运行 1min 测试
  enable_h_ext: false             # 启用 H 扩展支持（构建 host Linux）
archive_id_config:                # 影响生成 checkpoint 目录的命名
  gcc_version: "gcc12.2.0"
  riscv_ext: "rv64gcb"
  base_or_fixed: "base"
  special_flag: "your_flag"
  group: "archgroup"
```

也可以使用 `config/` 目录下的预定义配置：

```bash
python3 generate_checkpoint.py --config config/config-llvm19.yaml
```

### 3. 生成 Checkpoint

```bash
# 使用默认 config.yaml
python3 generate_checkpoint.py --config config.yaml

# 使用预定义配置
python3 generate_checkpoint.py --config config/config-llvm19.yaml
```

### 4. 导出结果

```bash
# 导出 speed 模式 int 套件的结果
python3 dump_result.py --base-path /path/to/archive --mode speed --suite int

# 导出 rate 模式 fp 套件的结果
python3 dump_result.py --base-path /path/to/archive --mode rate --suite fp

# 导出全部子项
python3 dump_result.py --base-path /path/to/archive --mode speed --suite all
```

参数说明：

| 参数 | 说明 | 默认值 |
|------|------|--------|
| `--base-path` | checkpoint archive 目录路径（必填） | - |
| `--mode` | SPEC 模式：`speed` 或 `rate` | `speed` |
| `--suite` | 套件类型：`int`、`fp` 或 `all` | `int` |
| `--times` | 三个阶段运行次数，逗号分隔 | `1,1,1` |
| `--ids` | 三个阶段起始 ID，逗号分隔 | `0,0,0` |

### 5. 筛选 Checkpoint 点

```bash
# 按权重覆盖率筛选（保留权重覆盖 80% 的点）
python3 select_points.py -i /path/to/cluster-0-0.json -o output_name -w 0.8

# 按数量上限筛选
python3 select_points.py -i /path/to/cluster-0-0.json -o output_name -c 5
```

## Speed 与 Rate 模式

SPEC CPU2017 有两种测试模式，通过 config.yaml 中的 `mode` 字段切换：

| | Speed 模式 | Rate 模式 |
|--|-----------|----------|
| 用途 | 测量单任务完成时间 | 测量系统吞吐量 |
| `mode` 值 | `"speed"` | `"rate"` |
| 加载文件 | `spec17_speed.json` (28个) | `spec17.json` (36个) |
| 独有 benchmark | pop2 | namd, parest, povray, blender |

两种模式的 benchmark 子集和输入参数不同，详见 [Speed/Rate 模式文档](doc/speed-rate-mode.md)。

## 注意事项

- 本脚本目前仅维护使用 opensbi 的环境
- 使用 QEMU 时 `all_in_one_workload` 必须为 `true`
- `copies` 为 1 时 kernel 放置在 0x80200000，大于 1 时放置在 0x80800000

## Reference

- https://github.com/OpenXiangShan/riscv-rootfs/blob/master/rootfsimg/spec_gen.py
    - checkpoint_scripts/spec_info/spec06.json 和 checkpoint_scripts/spec_info/spec17.json 是通过该仓库的脚本修改而来
    - checkpoint_scripts/generate_bbl.py 中的 default_initramfs_file，prepare_rootfs，traverse_path，__generate_initramfs，__generate_run_scripts 都是取自该仓库的脚本

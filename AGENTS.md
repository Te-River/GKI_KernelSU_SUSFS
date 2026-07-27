# GKI_KernelSU_SUSFS — Agent Guide

## 项目性质

自动化 GKI 内核构建系统，将 SukiSU-Ultra (KernelSU fork) + SUSFS 补丁集成到 Android GKI 内核中，产出 AnyKernel3 刷机包和 boot.img。

## 构建系统

所有构建逻辑集中在 `.github/workflows/scripts/`，Python 驱动，仅支持 **Linux**（GitHub Actions Ubuntu 或本地 Linux，不支持 Windows/macOS）。

**关键命令**（在 `scripts/` 目录下运行）：
```bash
pip install PyYAML        # 唯一外部依赖
python build.py --android android14 --kernel 6.1 --sub-level 124 --os-patch 2025-02  # 单版本
python build.py --matrix android14-6.1   # 构建整个矩阵
python build.py --all                    # 所有版本
python build.py --list-configs           # 查看合法组合
python build.py --dry-run --all          # 验证配置，不实际构建
```

## 目录布局

- `.github/workflows/scripts/` — **核心构建脚本**（`build.py` CLI 入口，`kernel_builder.py` 构建流水线，`config.py` 类型/常量/仓库 URL，`matrix_generator.py` CI 矩阵输出）
- `.github/workflows/config/matrix.json` — 构建矩阵定义（与 `build.py` 中的 `DEFAULT_BUILD_MATRIX` 字典 **重复维护**，修改一处需同步另一处）
- `hmbird_patch.c` — 独立内核模块，用于 OnePlus 8E 将 `HMBIRD_OGKI` 强制转为 `HMBIRD_GKI`
- `.github/actions/cache-*/` — 自定义缓存动作（通过 **GitHub Release** 存储 ccache，而非内置 actions/cache）

## 非直观的陷阱

1. **`config.py` 在 import 时联网**：模块顶层的 `get_susfs_version()` 会从 GitHub raw 下载版本号。无网络环境默认返回 `"v2.1.0"`，导致 Release tag 不准确。

2. **构建矩阵两个来源必须同步**：`build.py` 硬编码了 `DEFAULT_BUILD_MATRIX` 字典，`config/matrix.json` 是另一个副本。添加/删除版本时需要同时修改两处，否则 CI（`matrix_generator.py`）与 CLI 的行为不一致。

3. **KSU 版本值含中文**：`KSUVersion` 枚举值为 `"Stable(标准)"` 和 `"Dev(开发)"`。字符串比较/switch 必须精确匹配含中文的值。

4. **Android 12 特例**：唯一需要 `revision` 参数的版本（如 `r13`/`r15`/`r17`），且 `_prepare_android12_boot_images()` 有专门的 boot.img 处理逻辑。

5. **`sub_level: "X"` 表示 LTS**：字符串 `"X"` 代表当前最新的 LTS 子版本，不是数字。`is_lts()` 和 `get_sub_level_int()` 需特殊处理。

6. **遗留补丁（Legacy fixes）**：`LEGACY_FIXES` 字典定义了旧版内核的特殊补丁（android13-5.15 sub_level < 123 和 android12-5.10 < 136），这些补丁从外部 GitHub raw URL 下载。

7. **ccache 存于 GitHub Release**：缓存以 `.tar.zst` 压缩包上传到名为 `"ccache-cache"` 的 Release 下，而非标准的 actions/cache。

8. **构建工作目录**：默认 `/tmp/gki-build`，可通过 `GKI_WORKSPACE` 环境变量覆盖。

## 架构流水线（`KernelBuilder.build()`）

`clone_repos → clone_toolchain → setup_repo → init_and_sync_kernel → add_kernelsu → add_bbg → apply_susfs_patches → apply_sukisu_patches → apply_zram_patches → apply_task_mmu_fixes → configure_kernel → configure_kernel_name → show_kernel_config → build_kernel → patch_kpm_image → prepare_boot_images → create_anykernel_zips`

每个步骤都是可选的/条件性的——`use_zram`、`use_kpm`、`use_bbg` 等布尔配置控制哪些步骤实际执行。

## 支持的版本组合

| Android | Kernel  | Sub Levels                                          |
|---------|---------|-----------------------------------------------------|
| 12      | 5.10    | 136, 198, 209, 236, X (LTS)                        |
| 13      | 5.15    | 74, 123, 148, 170, 178, 180, 189                   |
| 14      | 6.1     | 78, 90, 99, 124, 138, 145                          |
| 15      | 6.6     | 50, 66, 102                                        |

---
title: PyTorch setup on Jetson Orin Nano
date: 2026-04-05
categories:
  - Jetson Orin Nano
  - Hands-on
---

# PyTorch setup on Jetson Orin Nano

The Jetson Orin Nano uses an ARM64 integrated GPU (nvgpu) with a custom L4T CUDA stack, so standard PyPI `torch` wheels won't work. This post walks through getting PyTorch running with proper GPU support.

<!-- more -->

---

!!! note
    The steps below reflect my specific setup: **JetPack 6.2, L4T 36.5, CUDA 12.6**. Some tweaks may be needed for other versions if following this blog.

!!! note "References"
    - [PyTorch for Jetson – NVIDIA Developer Forums](https://forums.developer.nvidia.com/t/pytorch-for-jetson/72048)
    - [Install PyTorch on Jetson Platform](https://docs.nvidia.com/deeplearning/frameworks/install-pytorch-jetson-platform/index.html)
    - [PyTorch Jetson Release Notes](https://docs.nvidia.com/deeplearning/frameworks/install-pytorch-jetson-platform-release-notes/pytorch-jetson-rel.html#pytorch-jetson-rel)

---

## Check version

First, check the JetPack / L4T / CUDA version running on the device.

```bash hl_lines="2 5"
$ cat /etc/nv_tegra_release
# R36 (release), REVISION: 5.0

$ nvidia-smi
| NVIDIA-SMI 540.5.0    Driver Version: 540.5.0    CUDA Version: 12.6 |
```

---

## Find the right wheel

NVIDIA hosts Jetson-specific PyTorch wheels at:

```
https://developer.download.nvidia.com/compute/redist/jp/
```

Browse that URL to find the directory closest to your JetPack version. In my case, I'm on JetPack 6.2, so I'd expect a `v62`, but listing the available directories showed:

```sh
$ curl -s https://developer.download.nvidia.com/compute/redist/jp/
# Lists: v33, v40, ..., v60, v61
```

No `v62`, so I went with `v61` as the closest match. Then I checked what wheel is available inside:

```sh
$ curl -s https://developer.download.nvidia.com/compute/redist/jp/v61/pytorch/
# torch-2.5.0a0+872d972e41.nv24.08.17622132-cp310-cp310-linux_aarch64.whl
```

So the wheel I'll use is `torch-2.5.0a0+872d972e41.nv24.08.17622132-cp310-cp310-linux_aarch64.whl`.

---

## Install system packages

```sh
$ sudo apt-get -y update
$ sudo apt-get install -y python3-pip libopenblas-dev
```

---

## Install cusparselt

Required for any PyTorch wheel from the 24.06 release onward.

!!! note
    The `install_cusparselt.sh` script only supports CUDA 12.1–12.4. Since our device has CUDA 12.6, passing `CUDA_VERSION=12.4` tricks the script into the correct branch.

```sh
$ wget https://raw.githubusercontent.com/pytorch/pytorch/5c6af2b583709f6176898c017424dc9981023c28/.ci/docker/common/install_cusparselt.sh

$ sudo CUDA_VERSION=12.4 bash ./install_cusparselt.sh
```

---

## Configure uv to use the NVIDIA wheel

By default, `uv sync` pulls `torch` from PyPI on every run, which would overwrite the Jetson wheel with an x86_64 incompatible one. Fix this by pinning `torch` to the NVIDIA wheel URL in `pyproject.toml`.

First, initialize uv if you haven't already:

```sh
$ uv init
```

Then add the following to `pyproject.toml`:

```toml
[tool.uv.sources]
torch = { url = "https://developer.download.nvidia.com/compute/redist/jp/v61/pytorch/torch-2.5.0a0+872d972e41.nv24.08.17622132-cp310-cp310-linux_aarch64.whl" }
```

Also pin NumPy to `<2`, since torch 2.5 was compiled against NumPy 1.x and will break with NumPy 2.x:

```toml
[project]
name = "my-project"
version = "0.1.0"
requires-python = ">=3.10"
dependencies = [
    "torch",
    "numpy>=1.26.4,<2",
]

[tool.uv.sources]
torch = { url = "https://developer.download.nvidia.com/compute/redist/jp/v61/pytorch/torch-2.5.0a0+872d972e41.nv24.08.17622132-cp310-cp310-linux_aarch64.whl" }
```

---

## Set `UV_SKIP_WHEEL_FILENAME_CHECK=1`

`uv` strictly validates that the version string inside the wheel metadata matches the wheel filename. NVIDIA's wheel has an inconsistent version string (`nv24.8` in metadata vs `nv24.08.17622132` derived from the filename), which causes:

```
error: Failed to install: torch-...
  Caused by: Wheel version does not match filename (2.5.0a0+872d972e41.nv24.8 != 2.5.0a0+872d972e41.nv24.8.17622132)
```

Persist the bypass in `~/.bashrc` so you don't have to set it every time:

```sh
$ echo 'export UV_SKIP_WHEEL_FILENAME_CHECK=1' >> ~/.bashrc
$ source ~/.bashrc
```

Then run:

```sh
$ uv sync
```

---

## Verify GPU is working

```sh
$ uv run python -c "import torch; print(torch.__version__); print('CUDA available:', torch.cuda.is_available()); print('Device count:', torch.cuda.device_count())"

2.5.0a0+872d972e41.nv24.08
CUDA available: True
Device count: 1
```

---

## Troubleshooting

A few extra errors I ran into. May not apply to your setup, but leaving these here just in case.

### Error: libnvToolsExt.so.1

```
OSError: libnvToolsExt.so.1: cannot open shared object file: No such file or directory
```

**Fix:**
```sh
$ sudo apt-get install -y libnvtoolsext1 cuda-nvtx-12-6
```

### Error: libcupti.so.12

```
ImportError: libcupti.so.12: cannot open shared object file: No such file or directory
```

**Fix:**
```sh
$ sudo apt-get install -y cuda-cupti-12-6
```

---

## Example

A minimal working example can be found here:
[rylanpeng/jetson-orin-nano-pytorch-setup](https://github.com/rylanpeng/jetson-orin-nano-pytorch-setup)

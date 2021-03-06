# Fedora

## Fedora 33

### Installing MBDYN

There is an issue when compiling where a type mismatch error is raised when
compiling the file `utils/femgen_f.f`. This error is due to GCC10
([read more...](https://github.com/Unidata/netcdf-fortran/issues/212#issuecomment-600191468)).

To fix this, you will need to pass `-fallow-argument-mismatch` when compiling.

You can add the following at the beginning of the file `configure` in the
MBDyn repository, and then follow the official instructions:

```bash
# To fix the type mismatch error when compiling Fortran files due to GCC10
FCFLAGS='-fallow-argument-mismatch'
FFLAGS='-fallow-argument-mismatch'
```

## Fedora 32

Below are listed all the snippets that once helped me to solve issues.

### CUDA and cuDNN

#### Install CUDA

To install CUDA (version 10.2 at the time of writing), I have followed the
instructions listed on [rpmfusion.org's CUDA
page](https://rpmfusion.org/Howto/CUDA).

```bash
sudo dnf config-manager --add-repo http://developer.download.nvidia.com/compute/cuda/repos/fedora29/x86_64/cuda-fedora29.repo
sudo dnf clean all
sudo dnf install cuda
```

For older versions of Fedora (>29), you need to install the version 8 of the `gcc`
compiler that is compatible with the CUDA toolkit (otherwise compilations will fail), you can use the following COPR repository:

```bash
sudo dnf copr enable kwizart/cuda-gcc-10.1
sudo dnf install cuda-gcc cuda-gcc-c++
```

Note that (as of April 20th, 2020) the build for Fedora 32 of `cuda-gcc` was a
bit too old and you may have a problem with the `libmpfr` library. It requires
the `libmpfr.so.4`, while `libmpfr.so.6` is available out-of-the-box in Fedora 32. To make it work, one way is to do the following:

```bash
# Build & install MPFR 3.1.6
wget https://www.mpfr.org/mpfr-3.1.6/mpfr-3.1.6.tar.gz
tar -xvf mpfr-3.1.6.tar.gz
cd mpfr-3.1.6
./configure
make
make check
sudo make install
```

Then add `/usr/local/lib` in your \$LD_LIBRARY_PATH variable. You can append the
path to the variable inside your `~/.bashrc` file.

#### Install cuDNN

Follow the instructions
[here](https://docs.nvidia.com/deeplearning/sdk/cudnn-install/index.html#installlinux-rpm).
Check by compiling the `mnistCUDNN` sample program that cuDNN was properly installed

### Docker

#### cgroups v2 issue

At the moment of writing, the official docker package (version 19.03.8) does not
allow the use of _cgroups V2_ that is enabled by default in Fedora 32. Note that
_cgroups V2_ will most likely be supported by the version 20.x of docker. So
the commands below will need to be undone when docker will be officially
released.

Solution was mentioned [there](https://github.com/docker/for-linux/issues/665#issuecomment-533998047)

```
# Open the GRUB default settings
sudo gedit /etc/default/grub
# Append GRUB_CMDLINE_LINUX with systemd.unified_cgroup_hierarchy=0
# Refresh GRUB
sudo grub2-mkconfig -o /boot/grub2/grub.cfg
```

You need to reboot, and then Docker 19.x will work with Fedora 32.

Note that `podman` may be a good replacement of `docker`.

#### Ubuntu image, with CUDA and cuDNN

You can use NVIDIA docker image
`nvcr.io/nvidia/cuda:10.0-cudnn7-runtime-ubuntu18.04`. Note that here the
version of Cuda is 10.0 because I was trying to use `tfjs-node-gpu` that only
works with Cuda 10.0 (and not higher).

To make the container recognize the Quadro T1000 Notebook of the HP zenBook U5
and make sure that tensorflow finds _libcuda_ (it will fix the same issue with `podman`),
you need to do install the `nvidia-docker` package. There is no official release
for Fedora 32, but your can do the [following](https://github.com/NVIDIA/nvidia-docker/issues/553#issuecomment-381075335):

```bash
curl -s -L https://nvidia.github.io/nvidia-docker/centos7/nvidia-docker.repo | \
  sudo tee /etc/yum.repos.d/nvidia-docker.repo
sudo dnf install nvidia-docker2
sudo pkill -SIGHUP dockerd
# To test that it works, you can use NVIDIA's cuda image
sudo docker run --runtime=nvidia --rm nvidia/cuda nvidia-smi
```

#### Internet connectivity problems within container

If you want to make sure that your
container reach internet, add `--network=host` when calling `docker run`.

### Podman

#### Make the GPU accessible by the container

If you do not give access to the GPU, the container will not be able to use it.
For instance, if you plan to use TensorFlow with GPU, it will not work without
doing the following:

```bash
sudo podman run --privileged #...
```

### v4l2loopback

#### Install

```bash
sudo dnf install gcc kernel-devel git
git clone https://<v4l2loopback-repo>/v4l2loopback.git
cd v4l2loopback
make && sudo make install
sudo depmod -a
sudo modprobe v4l2loopback
```

### System configuration

#### Install NVIDIA graphics drivers for (Quadro T1000 Notebook)

Follow the instructions from [rpmfusion.org's page](https://rpmfusion.org/Howto/NVIDIA#Current_GeForce.2FQuadro.2FTesla).
Alternatively, you can
also install the official NVIDIA drivers using their installer file. Do not
forget to blacklist the Nouveau driver before.

#### Disable device wake-up from suspend

You can disable wake-up from suspend changing the properties of the
`/proc/acpi/wakeup` file. For instance, `sudo echo XHC > /proc/acpi/wakeup` will
toggle the state of the device "XHC". This change is however not permanent. To
do so, I tried to use the `/etc/rc.d/rc.local` file, or place an bash script in
`/etc/rc.d/init.d` and enable/start the rc-local service (`sudo systemctl rc-local`) but it does not work.

The final solution was to use the UDEV rules as described
[here](https://unix.stackexchange.com/a/532839). The advantages here are 1) the
changes are permanent and no struggle, 2) you disable the specific peripheral
device while disabling a device in `wakeup` might mean disabling the whole USB controller.

#### Add a application shortcut

First, you need to create a desktop file that contains the information of your
application, for instance:

```bash
[Desktop Entry]
Type=Application
Encoding=UTF-8
Name=Spyder 4.0
Comment=Spyder application
Exec=bash -c "source /home/christian-nils/miniconda3/bin/activate myenv && spyder &"
Icon=/home/christian-nils/miniconda3/envs/myenv/lib/python3.7/site-packages/spyder/images/spyder.svg
Terminal=false
```

Second, save the file to `/usr/share/applications/`.

Third, and last, run the command `sudo update-desktop-database`.

#### Replace a package without breaking the system

If you execute `sudo dnf remove pulseaudio-module-bluetooth` you may remove the
whole gnome desktop. A way to replace this package without breaking the system
is to use a DNF shell as below:

```bash
sudo dnf shell
> dnf remove pulseaudio-module-bluetooth
> dnf install pulseaudio-module-bluetooth-freeworld
> run
```

The module bluetooth-freeworld will replace the module bluetooth without
breaking the dependencies.

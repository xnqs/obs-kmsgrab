# obs-kmsgrab: fast zero-copy screen capture for Linux ⚡

NOTE: This is a fork of w23's [obs-kmsgrab](https://github.com/w23/obs-kmsgrab), which seems to be no longer receiving updates. This fork minimally modifies a known working older build of obs-kmsgrab and adapts it to work on the latest versions of OBS.

It also makes the DMABUF sources look a lot prettier, giving them a monitor icon and a default name of "Screen Capture (KMS)", in order to fit in with the rest of OBS a lot better.

## Introduction

This plugin is a proof-of-concept libdrm-based screen capture for OBS. It uses DMA-BUF to import CRTC framebuffer directly into EGL texture in OBS as a source. This bypasses expensive double GPU->RAM RAM->GPU framebuffer copy that is invoked by anything X11-XSHM-based.

It is Linux-Only, as DMA-BUF is a Linux-only thing. Other platforms might have similar functionality, but I'm totally not an expert.

It is almost completely agnostic of any windowing system you might have: it works reasonably well with both X11 and Wayland, and theoretically could work even with bare KMS terminals.

However, on Wayland I'd recommend using something like https://hg.sr.ht/~scoopta/wlrobs instead -- it also uses DMA-BUF, but supposedly does this in a less hacky way.

## Building

This plugin works on any OBS 27+. I would still recommend getting the latest OBS 28 if you don't absolutely need OBS 27, though, as OBS 28 completely deprecates old inefficient GLX code, in favor of better EGL code. This means that if you are using OBS 28 you can use both Xcomposite window capture and KMS screen capture. Before OBS 28 we couldn't do this.

`CMAKE_PREFIX_PATH` is the location of your OBS installation. On most systems it is `/usr`.

Generally it works like this:
```
# Clone and cd
mkdir build && cd build
export CMAKE_PREFIX_PATH=<master-obs-prefix> # the root path of your OBS installation, on most systems /usr
cmake .. -GNinja -DCMAKE_INSTALL_PREFIX="$CMAKE_PREFIX_PATH"
ninja
ninja install
```

By default this plugin will use Polkit's `pkexec` to run the `linux-kmsgrab-send` helper utility with elevated privileges (i.e. as root). This is required in order to be able to grab screens using kms/libdrm API, as we completely sidestep X11/Wayland management of current drm context. When OBS starts you'll be presented with polkit screen asking for root password, and then you'll be asked again when configuring the capture module.

If you don't have Polkit set up, you need to compile this plugin with `-DENABLE_POLKIT=NO` cmake flag and entitle the `linux-kmsgrab-send` binary with `CAP_SYS_ADMIN` capability flag manually, like this:
```
sudo setcap cap_sys_admin+ep "$CMAKE_PREFIX_PATH/lib64/obs-plugins/linux-kmsgrab-send"
```
Note that this has serious system-wide security implications: just having this `linux-kmsgrab-send` binary lying around with caps set will make it possible for anyone having local user on your machine to grab any of your screens. Decide for yourself whether that's a concerning threat model for your situation.

## Known issues
- there's no way to specify grabbing device (in cause you have more than one GPU), it will just use the first available
- no sync whatsoever, known to rarily cause weird capture glitches (dirty regions missing for a few seconds)
- no resolution/framebuffer following -- may break if output resolution changes
- may conflict with some x11 compositors and wayland impls

# panvk-faux123-driver-rp4pro

> **Driver release news, bug reports, and the FauxWall beta sign-up are on the [FauxApps Discord](https://discord.gg/ceyPguKNbu).**

Mesa **PanVK**, the open source Vulkan driver for Arm Mali GPUs, built for
Android and packaged for [AdrenoTools](https://github.com/bylaws/libadrenotools)
style driver loading on the **Retroid Pocket 4 Pro** (Mali-G77). No root required.

The driver talks to the `mali_kbase` kernel driver that already ships on the
device, so there is no kernel to flash, and the system driver stays in place.
The stock driver on this device offers Vulkan 1.1. This build reports Vulkan 1.3.

I apply fixes that are not upstream in mesa, and I verify every one of them on
my own hardware before I release it. Each release lists what it changes.

**This is an early, experimental driver. It is not conformant, and one game has
been run on it: Giana Sisters: Twisted Dreams, through GameNative-Mali.**

**Bug reports and questions are welcome.** See [Getting help](#getting-help).

**Everything here is verified on one device: my Retroid Pocket 4 Pro, Mali-G77
MC9.** It is built for that device's kernel driver and I can make no promises
anywhere else. See
[Other devices](#other-devices-not-tested-no-guarantees).

Grab the latest `.adpkg.zip` from [Releases](../../releases) and import it in an
app whose driver picker accepts a PanVK package on Mali.

## One driver for all four devices

Starting with v1.0.0, this repository and my other three PanVK repositories ship
the same driver: one `libvulkan_panfrost.so`, built from one source tree, for the
[GameForce Ace](https://github.com/faux123/panvk-faux123-driver-ace) (Mali-G610), [Pixel 7](https://github.com/faux123/panvk-faux123-driver-pixel7) (Mali-G710), [Retroid Pocket 4 Pro](https://github.com/faux123/panvk-faux123-driver-rp4pro) (Mali-G77), and [RG556](https://github.com/faux123/panvk-faux123-driver-rg556) (Mali-G57). When the driver starts, it reads the GPU, the chip, and the kernel
driver, and turns on only the fixes that device needs.

Going forward:

- Each repository keeps its own package name,
  `panvk_faux123_rp4pro_<version>.adpkg.zip`. Download from the repository for your
  device, as before.
- Every release comes out in all four repositories at the same time, with the
  same version number and the same library inside.
- A fix found on one device reaches the other three in the next release.
- I test each release on all four of my devices before it goes out. Other Mali
  devices are still untested: see [Other devices](#other-devices-not-tested-no-guarantees).

The "What I added" and "How it was tested" sections below are the history of
this device's earlier builds. Every fix listed there is carried into v1.0.0,
either as my patch or as the equivalent change that is now in the base.

## Test it on your device

- **FauxVkTest** (free, Google Play): compare this driver against your stock driver on your own device. https://play.google.com/store/apps/details?id=com.faux.vktest&referrer=utm_source%3Dgithub%26utm_campaign%3Dpanvk-faux123-driver-rp4pro What it checks, in a 6-minute video: https://www.youtube.com/watch?v=CpEBlNcZmwo
- **All faux123 drivers** (Turnip for Adreno, PanVK for Mali), and every graphics chip each one has run on, in ten languages: https://canonspike.com/drivers/
- **FauxVkDB**: see how other devices score with stock and custom drivers. https://fauxvkdb.canonspike.com/
- **More apps from the same developer** (FauxClockNG, FauxCooling, RomRoster, Net SD card, and more): https://fauxapps.canonspike.com/
- **FauxApps Discord**: driver release news, questions, and results from other players. https://discord.gg/ceyPguKNbu
- **r/faux123** (Reddit): release posts, game reports, test results, and help (use the **PanVK help** flair). https://www.reddit.com/r/faux123/

---

## Where the driver comes from

Nothing here is a repackaged binary from somewhere else. It is built from source.

| | |
|---|---|
| Source | [`gitlab.freedesktop.org/mesa/mesa`](https://gitlab.freedesktop.org/mesa/mesa), 26.3.0-devel at commit `5a07217f` |
| `mali_kbase` support | the patch series from [`zenithblue-oss/panvk-kbase-android`](https://github.com/zenithblue-oss/panvk-kbase-android), release `v0.1.0-rc4` (`5b87b74`) |
| My patches | 49 on top of that series, shared by all four devices; earlier ones are summarized under [What I added](#what-i-added) |
| Toolchain | Android NDK 27.3, meson cross build, `aarch64`, API level 29 |
| Packaging | `.adpkg.zip`: `libvulkan_panfrost.so` plus `meta.json` |

**Every release names the exact build**, in its release notes and in the
`meta.json` inside the package.

The build container, the patches, the test harnesses and a recreate guide live in
a private companion repository.

### Why the stock kernel driver and not Panfrost

Upstream PanVK expects the open source Panfrost or Panthor kernel drivers. A
retail Android handheld ships Arm's `mali_kbase` kernel driver, and on the Pro
the bootloader is locked, so replacing the kernel is not an option. Running PanVK
on top of `mali_kbase` is what makes this installable without root.

### What I added

- The Mali-G77 was not in the driver's GPU table, so the driver did not start.
- Depth culling was set up wrongly for this GPU generation.
- The Android swapchain did not work on this device: the driver could not read
  MediaTek's buffer layout, and presenting a frame failed.
- The GPU clock stayed low, which made frames take more than twice as long.
  The driver now queues work the way the clock governor expects.
- A scene that submits the same commands more than once rendered black.
- Some games closed by themselves after a minute or two: a storage area for
  textures handed over at draw time was too small.
- A shader with many lights ran about six times slower than on the stock driver.
  The shader compiler loaded every light's data up front, ran out of registers,
  and wrote the values to memory and read them back for every pixel.
- Draws that take their parameters from a buffer drew every object with the
  first object's position and colour.
- Geometry shaders, tessellation, and transform feedback come from the newer
  Noysz patch series since v0.14.0. That series made the driver slower on this
  device in three ways, which I fixed: the screen got no completion signal for a
  frame still being drawn, so the GPU sat idle a tenth of the time; a scene that
  submits the same commands every frame waited for the GPU between every two
  passes; and work of two passes no longer ran side by side.
- A game running through Wine and DXVK showed short black dashes in its
  textures. The app hands the driver memory that the game writes through a
  cached mapping, and the GPU read it before the processor's cache had written
  it out. Since v0.15.0 the driver writes those lines out before each submit,
  for the buffers the GPU can read.
- **A game ran at 0.2 frames a second** (Dragon Ball Sparking Zero in Fauxium,
  its opening cinematic), and the kernel stopped a GPU job every five seconds.
  A compute shader that runs out of registers parks values in scratch memory,
  and on this GPU it needs at least 4 KiB of it per thread, more than the
  shader asks for. Since v0.16.0 the driver gives such a compute job at least
  4 KiB. The cinematic now plays at 6 to 7 frames a second. The cost is about
  96 MB of GPU memory for a command buffer that runs such a job.
- **Blending is advertised only where the GPU's blender works** since v0.17.0. The
  driver used to say it could blend into 32-bit float images, which this GPU
  can only do in the shader. An app that picks the widest float format it can
  blend into then drew twice the data per pixel that it draws on the stock
  driver, which does not advertise it. With this, my 4K HDR test scene takes
  36 ms a frame against 41 ms on the stock driver (about 42 ms, drawing twice the data before).
- Smaller speed-ups: vertex data is packed without gaps, vertex outputs that
  nothing reads are dropped, hidden pixels are skipped earlier, and screen areas
  that did not change are not written again.

---

## How it was tested

Everything below was measured on **one device**: my Retroid Pocket 4 Pro,
firmware `RP4Pro_V1.0.0.167`, Android 13, MediaTek Dimensity 1100, Mali-G77 MC9,
kernel 4.19.191, `mali_kbase` r32p1. Not rooted, bootloader locked.

The test rig is the Khronos [Vulkan-Samples](https://github.com/KhronosGroup/Vulkan-Samples)
app, rebuilt with AdrenoTools linked in so it loads a chosen driver directly. No
Wine, no DXVK, no Box64, no emulation. One variable: the driver `.so`.

- 33 offscreen graphics and compute tests pass: triangle, indexed and indirect
  draws, depth, stencil, blend, 4x MSAA, textures, multiple render targets,
  occlusion queries, copies, a compute dispatch, and fences.
- On screen, four sample scenes were run and looked at: `hello_triangle`
  (pixel identical to the stock driver), `subpasses`, `oit_depth_peeling`, and
  `compute_nbody`.
- My feature validator proves 23 of its 26 checks on this driver. The stock
  driver proves 4, because it does not offer the extensions the other checks
  need.

Frame times against the stock Mali driver, on those four scenes: three are within
2% of stock or faster. `oit_depth_peeling` takes 8.0 to 9.1 ms against 7.4 ms on
stock. A fifth scene, `pipeline_barriers` (24 lights), takes 10.6 ms against
8.7 ms on stock since v0.13.0; it took 67 ms before.

Five scenes is a small sample. Do not read it as "as fast as stock" in general.

## Other devices: not tested, no guarantees

I own one Retroid Pocket 4 Pro. Nothing here has been run on anything else. That
includes the standard Retroid Pocket 4 (Dimensity 900, Mali-G68), other Mali-G77
devices, and other firmware versions on the Pro.

Concretely, on any other device:

- The driver may not load at all. Each vendor's `mali_kbase` differs. Since
  v1.0.0 the driver detects Arm's and MediaTek's job layouts, but I have run it
  only on this Retroid Pocket 4 Pro and an RG556.
- Every fix I ship was written for the hardware I have. On a different device it
  may be unnecessary, and it may not be harmless.
- Nothing on this page was measured on that hardware, so none of the numbers
  apply to it.
- I will not look into problems on hardware I do not own.

Use it if you want to, but that is the honest state of it, and you are on your
own with it.

The same driver is released for each device I own, each in its own repository: [GameForce Ace](https://github.com/faux123/panvk-faux123-driver-ace) (Mali-G610), [Pixel 7](https://github.com/faux123/panvk-faux123-driver-pixel7) (Mali-G710), and [RG556](https://github.com/faux123/panvk-faux123-driver-rg556) (Mali-G57).

## Installing

**Read this first: you need an app that can load it.** Most driver pickers
were written for Adreno and refuse or ignore a Mali package.
[GameNative-Mali](https://github.com/faux123/GameNative-Mali/releases) can import
it from version 1.2.1-mali.11: open **Driver Manager**, import the zip, then
select it for a game under **Graphics**. That import screen is new and has had
little use.

1. Download `panvk_faux123_rp4pro_<version>.adpkg.zip` from
   [Releases](../../releases). Do not unzip it.
2. Import it in your app's driver manager, the same way you would import a Turnip
   package on an Adreno device.
3. Select it, then restart the container or game.

The driver has to be loaded by an app that opts in. Android does not let you
replace the system Vulkan driver for arbitrary apps without root, so a normal
Play Store game cannot use this.

To go back, select the system driver again. Nothing on the device is replaced.

---

## What works and what does not

**Not tested yet:** any game other than Giana Sisters: Twisted Dreams, window
resize, present modes other than the default, and long sessions.

**Not offered by this driver on the Mali-G77**, so an app that needs them refuses
to start rather than failing later:

Polygon mode and float depth bias representation.

Geometry shaders, tessellation shaders, and transform feedback are offered since
v0.14.0. On this device they have been checked with three small tests and one
sample scene, not with a conformance run and not with a game.

---

## Getting help

I built this driver on my own Retroid Pocket 4 Pro and test it there.

- Bug reports and questions are welcome. Open an [issue](../../issues/new) with your
  device, the driver version, the app you load it in (GameNative, Winlator, an
  emulator), the game, and what goes wrong. You can also post in
  [r/faux123](https://www.reddit.com/r/faux123/) with the **PanVK help** flair, or ask
  in the [FauxApps Discord](https://discord.gg/ceyPguKNbu).
- A [FauxVkTest](https://play.google.com/store/apps/details?id=com.faux.vktest) result
  from your device makes a report much easier to act on.
- I do not take requests for devices I do not own. If I do not have the device,
  I will not develop for it.
- I do not do remote debugging or beta testing.
- I may update this driver for myself from time to time and post it here. There
  is no schedule and no promise.

If you want to help, support the original developers listed under
[Credits](#credits).

---

## Credits

- **Mesa and the Panfrost team** wrote PanVK. This repository is a build of
  their work with patches on top. All the hard parts are theirs.
- **[zenithblue-oss/panvk-kbase-android](https://github.com/zenithblue-oss/panvk-kbase-android)**
  for the patch series that runs PanVK on `mali_kbase` on Android, which this
  build sits on since v1.0.0, including its Mali-G57 and G77 support.
- **[funnymdzz](https://github.com/funnymdzz/mesa)** for the `mali_kbase`
  backend that releases up to v0.17.0 sat on.
- **[Noysz](https://github.com/Noysz/panvk-g99-jm)** for the patch series that
  first brought PanVK up on this GPU generation, developed on a Mali-G57. One of
  its shader scheduler changes is still carried.
- **[bylaws](https://github.com/bylaws/libadrenotools)** for AdrenoTools, which
  is the only reason a custom Vulkan driver can be loaded without root.
- **Khronos** for Vulkan-Samples, which turned out to be a far better driver
  test bench than any game.

## License

PanVK is [MIT licensed](https://gitlab.freedesktop.org/mesa/mesa/-/blob/main/docs/license.rst),
and so are the patches in this build. See [LICENSE](LICENSE).

The driver is compiled against Arm's `mali_kbase` interface headers, which are
GPL-2.0 with the Linux syscall note. That note is what allows a program to use
a kernel interface without taking on the kernel's license.

This project is not affiliated with or endorsed by Arm, MediaTek, Retroid, Google,
the Khronos Group, or the Mesa project.

---

## Sponsor

These drivers are Mesa's PanVK with my patches on top; the Mesa developers did the hard part. If my builds help you, you can sponsor my work on GitHub: https://github.com/sponsors/faux123

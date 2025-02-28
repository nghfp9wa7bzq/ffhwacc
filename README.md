# Firefox hardware accelerated video decoding on Debian 12.

### Disclaimers
1. This seems to have worked for me. YMMV
2. No guarantee, warranty, use at your own risk and so on.

### Mesa update
Mesa 24 brings some video decoding acceleration to Ryzen APUs.  
(Didn't save the source for this...)  
I used the linked guide.  

### Firefox
NOTE: I'm using 128.6.0esr.  
In Firefox you can open `about:support` to see whether or not it can use either software or hardware acceleration for decoding, under the Media section.  
There are some settings in `about:config` to change this:  
webgl.force-enabled - default false. Changing this doesn't seem to make a difference. It does enable OpenGL of course, but it didn't change decoding support.  
gfx.webrender.all - Turn on web renderer written in rust that uses GPU to render websites. I have changed this to true.  
media.ffmpeg.vaapi.enabled - This enables video hardware acceleration. I have changed this to true.  
media.prefer-non-ffvpx - I have changed this to true. (Don't remember why this was important.)  
NOTE: I have seen suggestions to turn rdd off. IIRC that is used to containerize stuff for security, so it should be enabled.  

After this I can play back videos with less CPU usage, but Firefox became sluggish after a while, which could be 'reset' with restarting it.  
I came across the suggestion to use `amdgpu.gttsize=8192` kernel parameter to have more VRAM for the iGPU, however reading the documentation it is set pretty high as a default.  
'Restrict the size of GTT domain (for userspace use) in MiB for testing. The default is -1 (Use value specified by TTM).'  
v5.1 doc: 'Restrict the size of GTT domain in MiB for testing. The default is -1 (It’s VRAM size if 3GB < VRAM < 3/4 RAM, otherwise 3/4 RAM size).'

There is another parameter: `gartsize`  
'Restrict the size of GART (for kernel use) in Mib (32, 64, etc.) for testing. The default is -1 (The size depends on asic).'  
[Graphics address remapping table](https://en.wikipedia.org/wiki/Graphics_address_remapping_table) size as I understand is the VRAM size that is given by the BIOS / UEFI.  
In my case it is 512 MB for the iGPU.  
I have changed this as a kernel parameter to 2048 MB:  
`amdgpu.gartsize=2048`  
AFAIK this is the max. VRAM size, but that does not mean you lose 2 GB of memory.  
IOW total memory size reported by eg. qps does not change.  
This seems to solve the slowdowns.  
`dmesg` says:  
'PCIE GART of 2048M enabled'  
NOTE: 1024 MB may be enough, but 2048 MB seems to be better.  

I hope you've found this useful.  

2025.02.10. Update:  
Not using the gartsize kernel parameter results in dropped frames every once in a while when playing a video on youtube.  
The video / picture stops for a second (or less), but the audio keeps playing uninterrupted, so it is not a buffering issue.  
  
Using gartsize=2048 seem to make Firefox snappy, but a game I play freezes after a few hours.  
Not sure if gartsize only takes a chunk of memory without making it unavailable for other software and hence memory gets corrupted or there is an issue with the game itself.  

2025.02.17. Update:  
Using gartsize=1024 seem to be good enough. Video playback is not 100%, but I also don't get game crashes.  
  
2025.02.21. Update:  
Using gartsize=1024 Xorg crashes while watching a video...  
I have updated mesa to 24.3.4 with the kisak-mesa PPA, and the kernel to 6.12.9+bpo-amd64.  
Using gartsize=1024 Xorg crashes while watching a video...  

2025.02.22. Update:  
With no gartsize setting, iGPU became unusable for video playback with same settings. (1.75x or 2x speed 1080p)  
It drops 2/3 of the frames...  
It was way better before, only occasonally stuttering.  
(2nd GPU works fine as before, but I wouldn't want to use that, because unlike the iGPU it does not have hardware decoding and I need it for games. LOL)  

2025.02.28. Update:  
Turns out there is a [shader cache bug](https://bugzilla.mozilla.org/show_bug.cgi?id=1921742) in mesa or Firefox.  
I have deleted the mesa_shader_cache folder and start FF from command line with:  
```
MESA_SHADER_CACHE_DIR=/home/user/.cache/mesa_shader_cache/ firefox
```
This recreates the folder. Otherwise FF complains that it can't write to it...  
I have also set `gfx.webrender.all` to false in `about:config` in FF.  
With gartsize=1024 video playback seems normal.  
  
### Sources
[archlinux wiki Hardware video acceleration](https://wiki.archlinux.org/title/Hardware_video_acceleration)  
[archlinux wiki Hardware video acceleration Firefox](https://wiki.archlinux.org/title/Firefox#Hardware_video_acceleration)  
[Enhancing Graphics Performance on Debian 12: A Comprehensive Guide to Installing Mesa Drivers](https://shape.host/resources/enhancing-graphics-performance-on-debian-12-a-comprehensive-guide-to-installing-mesa-drivers)  
[Enabling accelerated video decoding in Firefox on Ubuntu 21.04](https://discourse.ubuntu.com/t/enabling-accelerated-video-decoding-in-firefox-on-ubuntu-21-04/22081)  
[Mesa 24 in Debian backports](https://www.reddit.com/r/linux_gaming/comments/1fqh7s5/debian_has_mesa_24_now/)  
[amdgpu kernel module parameters](https://docs.kernel.org/gpu/amdgpu/module-parameters.html)  
[amdgpu kernel module parameters (older doc)](https://www.kernel.org/doc/html/v5.1/gpu/amdgpu.html)  
[Firefox hardware acceleration issue](https://bbs.archlinux.org/viewtopic.php?id=300639)  
[Dedicated GPU only using 256MB of memory instead of full 2GB](https://bbs.archlinux.org/viewtopic.php?id=252954)  
[How to allocate more memory to my Ryzen APU's GPU?](https://github.com/ROCm/ROCm/issues/2014)  

---
layout: default
---


## Adding UVC hardware timestamp support for **[libcamera](https://libcamera.org/)** 
Google Summer of Code 2023 project page

![Banner](assets/astoria.png)
{% include links.html %}


## 🐟 About libcamera 🐟

libcamera is an open source camera stack that makes camera integration easy for Linux and other platforms. It sits on top of V4L2 to make a unified camera acquisition experience across devices and hardware.

## 🐟 About this project 🐟

One of the camera types supported by libcamera is UVC (USB video class) devices. Most of the commonly used devices that fall under this category are webcams, which may be standalone (such as a logitech webcam) or integrated into a laptop as the built-in camera. When libcamera provides the user with image data from the device, it also provides a timestamp.  This timestamp is taken by the computer (host) when the device driver handles a frame received from the camera.  libcamera uses this timestamp as the sensor timestamp reported to the user.  However, UVC devices may provide metadata with information on when raw frame capture began.  The goal of this project is to use the UVC metadata to calculate this more accurate timestamp and report that to the user.

## 🐟 What I did 🐟

For detailed week-by-week logs, see my weekly [slides](https://drive.google.com/drive/folders/1gL-HnDRm4eg5gH1AjDR5kOUT1WMUxc4d?usp=sharing).

### 🐟 Technical background 🐟

*Linux video nodes*

In Linux, media devices are exposed to the user through "video nodes" accessible in  the <span class="code">/dev/</span> directory. UVC metadata is obtained through streaming on one of these video nodes.  This means that for every UVC device that supports metadata, two video nodes are exposed: one for the actual video stream, and the other for the metadata stream that can be linked to each frame of the video stream.  When you look at the capabilities of the metadata's video node, it specified that the video node supports metadata capture:

![videoNode](/assets/videoNode.png)

This indicates that to obtain metadata information, you need to open the video device and configure streaming just as you would for a normal video node. V4L2 (and libcamera) keep track of the sequence number of each buffer that arrives, so the video and metadata frames must be synchronized by their sequence number.  

*UVC metadata buffers*

Configuring streaming on V4L2 involves (at a high level) requesting the driver to allocate memory for the device-provided data to be stored.  In the case of a normal video node, this is image data.  The amount of data can get very large- the amount of space it takes to hold one height x width amount of pixel data!  Memory management, therefore, is no trivial task: copying that much information from the device to the application can add overhead in a scenario where time is valuable. V4L2 handles this by providing userspace-accessible pointers to data buffers that get filled by the device.  When the device fills up a buffer with data, the video node indicates a buffer is ready and users will be able to access the pointers.

This is done through a series of system calls performed on the video node.  

* Users must [request buffer allocation](https://www.kernel.org/doc/html/v4.9/media/uapi/v4l/vidioc-reqbufs.html#vidioc-reqbufs), specifying one of a variety of [memory mapping options](ttps://www.kernel.org/doc/html/v4.9/media/uapi/v4l/buffer.html#c.v4l2_memory). 
* The user needs some way of accessing the data inside these buffers.  Using information obtained for each buffer from [querying the driver](https://www.kernel.org/doc/html/v4.9/media/uapi/v4l/vidioc-querybuf.html), the user can either get file descriptors [exported for each buffer](https://www.kernel.org/doc/html/v4.9/media/uapi/v4l/vidioc-expbuf.html) or use [mmap](https://www.kernel.org/doc/html/v4.9/media/uapi/v4l/mmap.html) to get pointers to the buffers.
* The driver needs to be told that these buffers are ready to be filled by the device.  Signaling to the driver that the buffer is ready to be filled by the device will put it on a queue managed by the driver, so this is called [buffer queueing](https://www.kernel.org/doc/html/v4.9/media/uapi/v4l/vidioc-qbuf.html#vidioc-qbuf). 
* The user tells the driver to [signal to the device to start streaming](https://www.kernel.org/doc/html/v4.9/media/uapi/v4l/vidioc-streamon.html#c.VIDIOC_STREAMON).
* As buffers are filled, the user waits for the device to fill the buffers.  The user [is notified](https://www.kernel.org/doc/html/v4.9/media/uapi/v4l/func-poll.html) when this happens, and can [remove a buffer from the buffer queue](https://www.kernel.org/doc/html/v4.9/media/uapi/v4l/vidioc-qbuf.html#c.VIDIOC_DQBUF).

*Buffer contents and timestamp calculation*

The metadata buffer format provided by V4L2 can be found in the [UVCH](https://www.kernel.org/doc/html/v5.19/userspace-api/media/v4l/pixfmt-meta-uvc.html) documentation for V4L2:
* A system (host device) timestamp (ts) in host clock units taken when the driver gets the data from the device
* The USB clock time (sof) at that same point in time

The data that we are interested in for the purposes of calculating new timestamps, however, is also in the "buf" field of the UVC Metadata Block. According to the [UVC specification](https://www.usb.org/document-library/video-class-v15-document-set), metadata information (referred to as "payload headers") contain the following:
* Length of the header
* A bitmap representing which information is specified in the payload
* A presentation time stamp (PTS) representing the time in device clock units when raw frame capture started
* A "source clock" value, encoding a source time clock timestamp in device clock units (STC) and the USB bus clock (sof) at that same point in time

The "PTS" mentioned above is the "true" sensor timestamp that should be reported to a user. To convert the PTS to a host timestamp, it is necessary to convert it into USB clock time and from there into host time.  Because the UVS device reports a moment in time in both device clock units and source clock units, all that is needed is to linearly interpolate the PTS between two of these samples.  The sof value is represented in 11 bits, so it overflows frequently.  

The below graph demonstrates the relationship between the source time clock (device clock) value and the USB clock sof value both taken at the same time. The data in this graph represents a few minutes in real-world time:

![STC versus SOF](/assets/source%20time%20clock%20versus%20sof.png)

Likewise, linear interpolation is used to convert from this USB value to a host clock time.  Similar to the UVC device, the V4L2 buffer provides a point in time in both the USB clock units and host clock units to make it possible to calculate the relationship between them.

#### What I did
During the first half of GSoC, I focused on learning about libcamera internals.  The goal was to obtain access to the information needed to calculate UVC timestamps.  This meant learning about V4L2 streaming I/O, as well as learning about the abstractions libcamera makes internally to wrap V4L2 commands.  There were already some scenarios that existed that seemed similar to what I would need to do to get UVC metadata, so I investigated how those worked and replicated them as best as I could.

*Determining the metadata node*

I added the ability to identify, open, and store the metadata node associated with the UVC device connected to the camera. I handled errors in case it didn't exist.

*Allocate metadata buffers*

As explained above, the metadata stream fills data in buffers.  The pointers to these buffers are shared between the user, the kernel, and the device.  I allocated these buffers, mapped them into user space, and then attempted the laborious task of cleanup while trying to think of all the ways it could go wrong.

Most of this task was spent trying to understand the different buffer-related options there were and figuring out which ones I specifically needed to do. There are a lot of similar- and opposite-sounding operations to perform when it comes to setting up buffers: exporting buffers, importing buffers, allocating buffers, creating buffers, etc. In the end, the metadata stream was much simpler to handle than other streams: allocating buffers and mapping them into userspace virtual memory was all that was needed.  

This was, however, complicated by some unknowns: the UVCH format type does not support exporting buffers to get file descriptors, which is what libcamera does by default.  I had to modify some of the internal libcamera code to get this working.

*Handling the metadata stream*

This was the trickiest task of all. The video buffers and metadata buffers appear to come in as pairs, but which one of them comes first is seemingly random.  It is critical that the video buffer be matched with the correct metadata buffer, because the goal is to use the metadata buffer timing data to set a new time for its corresponding video buffer. Originally, UVC buffers were "completed" by calling a couple of completion functions immediately after dequeuing the buffer.  To support accurate timestamps, a video buffer must have the information provided by its corresponding metadata buffer. However, the arrival of metadata buffers and video buffers are not simultaneous.

The solution was to keep a queue with information about each buffer as it came in. If a metadata buffer had come in before its corresponding video node, then the timestamp information should be stored in a queue along with the sequence number of the metadata buffer. When its video buffer came in, the sequence of the new video buffer and the metadata buffer at the front of the queue are compared and the metadata buffer's timestamp used.  

However, if a video buffer arrives before its metadata buffer, the buffer cannot be marked as completed because information (the new timestamp) is needed.  The video buffer is added to a queue and completed when the metadata buffer finally does come in by the code that handles the metadata buffer.  

*Parsing metadata information and calculating timestamps*

With metadata buffers coming in and properly being matched to video buffers, the timestamps can be calculated. For the calculation (described above), two metadata samples are needed. The original Linux implementation of this uses a sample 32 frames back, so I used a deque to add and remove metadata samples as they came in.  

Using the newest and oldest samples, I calculated the linear relationship between the device clock and the usb clock, and then the usb clock and host (system) clock.  I converted the accurate time (PTS) into system time, and replaced the old timestamp value with that new calculated one.

## 🐟 The current state 🐟

The patches are in active reviewing-fix-review again stage.  I anticipate there will be many more of these cycles to come.  All of the functionality (namely, setting up the metadata buffer stream and calculating new timestamps) has been implemented.

## 🐟 What's left 🐟

* Come up with a better way to pass along buffer plane offset information into the call to mmap than by setting a flag.

There are also some bugs that still need to be investigated:

* The starting frame of UVC data makes a huge impact on the matching of buffer sequences, and is always non-zero.  It is difficult to know how to get this number, because which number it is matters.

* Cleanup of the code sometimes fails to properly complete all of the outstanding Requests, leading to errors upon termination of the application.

* (Unknown cause) Metadata buffers will not arrive when the frame rate is too low

In addition to this, I believe there will be adjustments made as people continue to give feedback on the patches.

## 🎣 Challenges 🎣

*Technical challenges* 

The metadata buffer stream did not have some of the functionality required to "plug in" nicely with existing internal libcamera functionality.  Namely, it did not support a method of memory-mapping that was being used by all other devices within libcamera.  This required me to modify existing code in order to accommodate the metadata buffers, some of which has not been fully completed. 

*Personal challenges*

I found myself getting stuck on bugs that required some lightning bolt (from my perspective) insight to solve. When dealing with devices, like cameras, it's difficult to know when the problem you're facing has an answer that's right in front of you, when you need to bring out better tools to get more information, or when the problem is due to something you don't know exists. 

I never worked with C++ that used templates and smart pointers, so I had to learn how to use modern C++ data structures and do memory management.  This also meant I needed to change my coding style and rethink ways to solve problems.

Learning practical git after I muddied up my tree was very difficult.

## 💖 Acknowledgements 💖

I'm extremely grateful to my mentors and everyone who helped answer my questions. Contributing to an open source project in any capacity would have been impossible without the fine whip-smart folks over at libcamera who answered my (at times extremely basic) questions.

## 🐟 Resources & Links 🐟

* [The kernel algorithm used to calculate the more accurate timestamps](https://elixir.bootlin.com/linux/v5.17-rc4/source/drivers/media/usb/uvc/uvc_video.c#L629)

* [How to split a commit](https://www.internalpointers.com/post/split-commit-into-smaller-ones-git)
 
* [Autosquashing git commits](https://thoughtbot.com/blog/autosquashing-git-commits)

* [V4L2 API](https://www.kernel.org/doc/html/v4.9/media/uapi/v4l/v4l2.html)

* [Using mmap for Streaming I/O](https://www.kernel.org/doc/html/v4.9/media/uapi/v4l/mmap.html)

## 🐟 unsolicited advice 🐟
* <span class="code">echo 0xffff | sudo tee /sys/class/video4linux/video3/dev_debug</span> to see in dmesg what system calls are being sent to video3

* Interactive rebase is almost always what you want

* Use [gitk] https://git-scm.com/docs/gitk with gitk --all &

* Always use <span class="code">git add -p my_file</span>



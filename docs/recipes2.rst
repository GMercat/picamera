.. _recipes2:

================
Advanced Recipes
================

The following recipes involve advanced techniques and may not be "beginner
friendly". Please feel free to suggest enhancements or additional recipes.


.. _yuv_capture:

Unencoded image capture (YUV format)
====================================

If you want images captured without loss of detail (due to JPEG's lossy
compression), you are probably better off exploring PNG as an alternate image
format (PNG uses lossless compression). However, some applications
(particularly scientific ones) simply require the image data in numeric form.
For this, the ``'yuv'`` format is provided::

    import time
    import picamera

    with picamera.PiCamera() as camera:
        camera.resolution = (100, 100)
        camera.start_preview()
        time.sleep(2)
        camera.capture('image.data', 'yuv')

The specific `YUV`_ format used is `YUV420`_ (planar). This means that the Y
(luminance) values occur first in the resulting data and have full resolution
(one 1-byte Y value for each pixel in the image). The Y values are followed by
the U (chrominance) values, and finally the V (chrominance) values.  The UV
values have one quarter the resolution of the Y components (4 1-byte Y values
in a square for each 1-byte U and 1-byte V value).

It is also important to note that when outputting to unencoded formats, the
camera rounds the requested resolution. The horizontal resolution is rounded up
to the nearest multiple of 32 pixels, while the vertical resolution is rounded
up to the nearest multiple of 16 pixels. For example, if the requested
resolution is 100x100, the capture will actually contain 128x112 pixels worth
of data, but pixels beyond 100x100 will be uninitialized.

Given that the `YUV420`_ format contains 1.5 bytes worth of data for each pixel
(a 1-byte Y value for each pixel, and 1-byte U and V values for every 4 pixels),
and taking into account the resolution rounding, the size of a 100x100 YUV
capture will be:

.. image:: yuv_math.svg
    :align: center

The first 14336 bytes of the data (128*112) will be Y values, the next 3584
bytes (128*112/4) will be U values, and the final 3584 bytes will be the V
values.

The following code demonstrates capturing YUV image data, loading the data into
a set of `numpy`_ arrays, and converting the data to RGB format in an efficient
manner::

    from __future__ import division

    import time
    import picamera
    import numpy as np

    width = 100
    height = 100
    stream = open('image.data', 'w+b')
    # Capture the image in YUV format
    with picamera.PiCamera() as camera:
        camera.resolution = (width, height)
        camera.start_preview()
        time.sleep(2)
        camera.capture(stream, 'yuv')
    # Rewind the stream for reading
    stream.seek(0)
    # Calculate the actual image size in the stream (accounting for rounding
    # of the resolution)
    fwidth = (width + 31) // 32 * 32
    fheight = (height + 15) // 16 * 16
    # Load the Y (luminance) data from the stream
    Y = np.fromfile(stream, dtype=np.uint8, count=fwidth*fheight).\
            reshape((fheight, fwidth))
    # Load the UV (chrominance) data from the stream, and double its size
    U = np.fromfile(stream, dtype=np.uint8, count=(fwidth//2)*(fheight//2)).\
            reshape((fheight//2, fwidth//2)).\
            repeat(2, axis=0).repeat(2, axis=1)
    V = np.fromfile(stream, dtype=np.uint8, count=(fwidth//2)*(fheight//2)).\
            reshape((fheight//2, fwidth//2)).\
            repeat(2, axis=0).repeat(2, axis=1)
    # Stack the YUV channels together, crop the actual resolution, convert to
    # floating point for later calculations, and apply the standard biases
    YUV = np.dstack((Y, U, V))[:height, :width, :].astype(np.float)
    YUV[:, :, 0]  = YUV[:, :, 0]  - 16   # Offset Y by 16
    YUV[:, :, 1:] = YUV[:, :, 1:] - 128  # Offset UV by 128
    # YUV conversion matrix from ITU-R BT.601 version (SDTV)
    #              Y       U       V
    M = np.array([[1.164,  0.000,  1.596],    # R
                  [1.164, -0.392, -0.813],    # G
                  [1.164,  2.017,  0.000]])   # B
    # Take the dot product with the matrix to produce RGB output, clamp the
    # results to byte range and convert to bytes
    RGB = YUV.dot(M.T).clip(0, 255).astype(np.uint8)

Alternatively, see :ref:`rgb_capture` for a method of having the camera output
RGB data directly.

.. note::

    Capturing so-called "raw" formats (``'yuv'``, ``'rgb'``, etc.) does not
    provide the raw bayer data from the camera's sensor. Rather, it provides
    access to the image data after GPU processing, but before format encoding
    (JPEG, PNG, etc). Currently, the only method of accessing the raw bayer
    data is via the *bayer* parameter to the :meth:`~picamera.PiCamera.capture`
    method.

.. versionchanged:: 1.0
    The :attr:`~picamera.PiCamera.raw_format` attribute is now deprecated, as
    is the ``'raw'`` format specification for the
    :meth:`~picamera.PiCamera.capture` method. Simply use the ``'yuv'`` format
    instead, as shown in the code above.


.. _rgb_capture:

Unencoded image capture (RGB format)
====================================

The RGB format is rather larger than the `YUV`_ format discussed in the section
above, but is more useful for most analyses. To have the camera produce output
in `RGB`_ format, you simply need to specify ``'rgb'`` as the format for the
:meth:`~picamera.PiCamera.capture` method instead::

    import time
    import picamera

    with picamera.PiCamera() as camera:
        camera.resolution = (100, 100)
        camera.start_preview()
        time.sleep(2)
        camera.capture('image.data', 'rgb')

The size of `RGB`_ data can be calculated similarly to `YUV`_ captures.
Firstly round the resolution appropriately (see :ref:`yuv_capture` for the
specifics), then multiply the number of pixels by 3 (1 byte of red, 1 byte of
green, and 1 byte of blue intensity). Hence, for a 100x100 capture, the amount
of data produced is:

.. image:: rgb_math.svg
    :align: center

The resulting `RGB`_ data is interleaved. That is to say that the red, green
and blue values for a given pixel are grouped together, in that order. The
first byte of the data is the red value for the pixel at (0, 0), the second
byte is the green value for the same pixel, and the third byte is the blue
value for that pixel. The fourth byte is the red value for the pixel at (1, 0),
and so on.

Loading the resulting RGB data into a `numpy`_ array is simple::

    from __future__ import division

    width = 100
    height = 100
    stream = open('image.data', 'w+b')
    # Capture the image in RGB format
    with picamera.PiCamera() as camera:
        camera.resolution = (width, height)
        camera.start_preview()
        time.sleep(2)
        camera.capture(stream, 'rgb')
    # Rewind the stream for reading
    stream.seek(0)
    # Calculate the actual image size in the stream (accounting for rounding
    # of the resolution)
    fwidth = (width + 31) // 32 * 32
    fheight = (height + 15) // 16 * 16
    # Load the data in a three-dimensional array and crop it to the requested
    # resolution
    image = np.fromfile(stream, dtype=np.uint8).\
            reshape((fheight, fwidth, 3))[:height, :width, :]
    # If you wish, the following code will convert the image's bytes into
    # floating point values in the range 0 to 1 (a typical format for some
    # sorts of analysis)
    image = image.astype(np.float, copy=False)
    image = image / 255.0

.. versionchanged:: 1.0
    The :attr:`~picamera.PiCamera.raw_format` attribute is now deprecated, as
    is the ``'raw'`` format specification for the
    :meth:`~picamera.PiCamera.capture` method. Simply use the ``'rgb'`` format
    instead, as shown in the code above.

.. warning::

    You may find `RGB`_ captures rather slow. If this is the case, please try
    the ``'rgba'`` format instead. The reason for this is that the GPU component
    that picamera uses to perform RGB conversion doesn't support RGB output,
    only `RGBA`_. As a result, RGBA data can be written directly, but picamera
    has to spend time stripping out the (unused) alpha byte from RGBA if RGB
    format is requested. A similar situation exists for the BGR and BGRA
    formats.




.. _rapid_capture:

Rapid capture and processing
============================

The camera is capable of capturing a sequence of images extremely rapidly by
utilizing its video-capture capabilities with a JPEG encoder (via the
``use_video_port`` parameter). However, there are several things to note about
using this technique:

* When using video-port based capture only the video recording area is
  captured; in some cases this may be smaller than the normal image capture
  area (see dicussion in :ref:`camera_modes`).

* No Exif information is embedded in JPEG images captured through the
  video-port.

* Captures typically appear "granier" with this technique. The author is not
  aware of the exact technical reasons why this is so, but suspects that some
  part of the image processing pipeline that is present for still captures is
  not used when performing still captures through the video-port.

All capture methods support the ``use_video_port`` option, but the methods
differ in their ability to rapidly capture sequential frames. So, whilst
:meth:`~picamera.PiCamera.capture` and
:meth:`~picamera.PiCamera.capture_continuous` both support ``use_video_port``,
:meth:`~picamera.PiCamera.capture_sequence` is by far the fastest method
(because it does not re-initialize an encoder prior to each capture). Using
this method, the author has managed 30fps JPEG captures at a resolution of
1024x768.

By default, :meth:`~picamera.PiCamera.capture_sequence` is particularly suited to
capturing a fixed number of frames rapidly, as in the following example which
captures a "burst" of 5 images::

    import time
    import picamera

    with picamera.PiCamera() as camera:
        camera.resolution = (1024, 768)
        camera.framerate = 30
        camera.start_preview()
        time.sleep(2)
        camera.capture_sequence([
            'image1.jpg',
            'image2.jpg',
            'image3.jpg',
            'image4.jpg',
            'image5.jpg',
            ])

We can refine this slightly by using a generator expression to provide the
filenames for processing instead of specifying every single filename manually::

    import time
    import picamera

    frames = 60

    with picamera.PiCamera() as camera:
        camera.resolution = (1024, 768)
        camera.framerate = 30
        camera.start_preview()
        # Give the camera some warm-up time
        time.sleep(2)
        start = time.time()
        camera.capture_sequence([
            'image%02d.jpg' % i
            for i in range(frames)
            ], use_video_port=True)
        finish = time.time()
    print('Captured %d frames at %.2ffps' % (
        frames,
        frames / (finish - start)))

However, this still doesn't let us capture an arbitrary number of frames until
some condition is satisfied. To do this we need to use a generator function to
provide the list of filenames (or more usefully, streams) to the
:meth:`~picamera.PiCamera.capture_sequence` method::

    import time
    import picamera

    frames = 60

    def filenames():
        frame = 0
        while frame < frames:
            yield 'image%02d.jpg' % frame
            frame += 1

    with picamera.PiCamera() as camera:
        camera.resolution = (1024, 768)
        camera.framerate = 30
        camera.start_preview()
        # Give the camera some warm-up time
        time.sleep(2)
        start = time.time()
        camera.capture_sequence(filenames(), use_video_port=True)
        finish = time.time()
    print('Captured %d frames at %.2ffps' % (
        frames,
        frames / (finish - start)))

The major issue with capturing this rapidly is that the Raspberry Pi's IO
bandwidth is extremely limited. As a format, JPEG is considerably less
efficient than the H.264 video format (which is to say that, for the same
number of bytes, H.264 will provide considerably better quality over the same
number of frames).

At higher resolutions (beyond 800x600) you are likely to find you cannot
sustain 30fps captures to the Pi's SD card for very long (before exhausting the
disk cache).  In other words, if you are intending to perform processing on the
frames after capture, you may be better off just capturing video and decoding
frames from the resulting file rather than dealing with individual JPEG
captures.

However, if you can perform your processing fast enough, you may not need to
involve the disk at all.  Using a generator function, we can maintain a queue
of objects to store the captures, and have parallel threads accept and process
the streams as captures come in. Provided the processing runs at a faster frame
rate than the captures, the encoder won't stall and nothing ever need hit the
disk.

Please note that the following code involves some fairly advanced techniques
(threading and all its associated locking fun is typically not a "beginner
friendly" subject, not to mention generator expressions)::

    import io
    import time
    import threading
    import picamera

    # Create a pool of image processors
    done = False
    lock = threading.Lock()
    pool = []

    class ImageProcessor(threading.Thread):
        def __init__(self):
            super(ImageProcessor, self).__init__()
            self.stream = io.BytesIO()
            self.event = threading.Event()
            self.terminated = False
            self.start()

        def run(self):
            # This method runs in a separate thread
            global done
            while not self.terminated:
                if self.event.wait(1):
                    try:
                        self.stream.seek(0)
                        # Read the image and do some processing on it
                        #Image.open(self.stream)
                        #...
                        #...
                        # Set done to True if you want the script to terminate
                        # at some point
                        #done=True
                    finally:
                        # Reset the stream and event
                        self.stream.seek(0)
                        self.stream.truncate()
                        self.event.clear()
                        # Return ourselves to the pool
                        with lock:
                            pool.append(self)

    def streams():
        while not done:
            with lock:
                processor = pool.pop()
            yield processor.stream
            processor.event.set()

    with picamera.PiCamera() as camera:
        pool = [ImageProcessor() for i in range (4)]
        camera.resolution = (640, 480)
        # Set the framerate appropriately; too fast and the image processors
        # will stall the image pipeline and crash the script
        camera.framerate = 10
        camera.start_preview()
        time.sleep(2)
        camera.capture_sequence(streams(), use_video_port=True)

    # Shut down the processors in an orderly fashion
    while pool:
        with lock:
            processor = pool.pop()
        processor.terminated = True
        processor.join()

.. versionadded:: 0.5


.. _rapid_streaming:

Rapid capture and streaming
===========================

Following on from :ref:`rapid_capture`, we can combine the video-port capture
technique with :ref:`streaming_capture`. The server side script doesn't change
(it doesn't really care what capture technique is being used - it just reads
JPEGs off the wire). The changes to the client side script can be minimal at
first - just add ``use_video_port=True`` to the
:meth:`~picamera.PiCamera.capture_continuous` call::

    import io
    import socket
    import struct
    import time
    import picamera

    client_socket = socket.socket()
    client_socket.connect(('my_server', 8000))
    connection = client_socket.makefile('wb')
    try:
        with picamera.PiCamera() as camera:
            camera.resolution = (640, 480)
            time.sleep(2)
            start = time.time()
            stream = io.BytesIO()
            # Use the video-port for captures...
            for foo in camera.capture_continuous(stream, 'jpeg',
                                                 use_video_port=True):
                connection.write(struct.pack('<L', stream.tell()))
                connection.flush()
                stream.seek(0)
                connection.write(stream.read())
                if time.time() - start > 30:
                    break
                stream.seek(0)
                stream.truncate()
        connection.write(struct.pack('<L', 0))
    finally:
        connection.close()
        client_socket.close()

Using this technique, the author can manage about 14fps of streaming at
640x480. One deficiency of the script above is that it interleaves capturing
images with sending them over the wire (although we deliberately don't flush on
sending the image data). Potentially, it would be more efficient to permit
image capture to occur simultaneously with image transmission. We can attempt
to do this by utilizing the background threading techniques from the final
example in :ref:`rapid_capture`.

Once again, please note that the following code involves some quite advanced
techniques and is not "beginner friendly"::

    import io
    import socket
    import struct
    import time
    import threading
    import picamera

    client_socket = socket.socket()
    client_socket.connect(('my_server', 8000))
    connection = client_socket.makefile('wb')
    try:
        connection_lock = threading.Lock()
        pool = []
        pool_lock = threading.Lock()

        class ImageStreamer(threading.Thread):
            def __init__(self):
                super(ImageStreamer, self).__init__()
                self.stream = io.BytesIO()
                self.event = threading.Event()
                self.terminated = False
                self.start()

            def run(self):
                # This method runs in a background thread
                while not self.terminated:
                    if self.event.wait(1):
                        try:
                            with connection_lock:
                                connection.write(struct.pack('<L', self.stream.tell()))
                                connection.flush()
                                self.stream.seek(0)
                                connection.write(self.stream.read())
                        finally:
                            self.stream.seek(0)
                            self.stream.truncate()
                            self.event.clear()
                            with pool_lock:
                                pool.append(self)

        count = 0
        start = time.time()
        finish = time.time()

        def streams():
            global count, finish
            while finish - start < 30:
                with pool_lock:
                    streamer = pool.pop()
                yield streamer.stream
                streamer.event.set()
                count += 1
                finish = time.time()

        with picamera.PiCamera() as camera:
            pool = [ImageStreamer() for i in range(4)]
            camera.resolution = (640, 480)
            # Set the framerate appropriately; too fast and we'll starve the
            # pool of streamers and crash the script
            camera.framerate = 15
            camera.start_preview()
            time.sleep(2)
            camera.capture_sequence(streams(), 'jpeg', use_video_port=True)

        # Shut down the streamers in an orderly fashion
        while pool:
            with pool_lock:
                streamer = pool.pop()
            streamer.terminated = True
            streamer.join()

        # Write the terminating 0-length to the connection to let the server
        # know we're done
        with connection_lock:
            connection.write(struct.pack('<L', 0))

    finally:
        connection.close()
        client_socket.close()

    print('Sent %d images in %.2f seconds at %.2ffps' % (
        count, finish-start, count / (finish-start)))

The author's tests with the script above haven't yielded substantial
improvements over the former script using
:meth:`~picamera.PiCamera.capture_continuous`, but the reason for this is not
currently clear. Suggestions for further improvements are welcomed!

.. versionadded:: 0.5


.. _record_and_capture:

Capturing images whilst recording
=================================

The camera is capable of capturing still images while it is recording video.
However, if one attempts this using the stills capture mode, the resulting
video will have dropped frames during the still image capture. This is because
regular stills require a mode change, causing the dropped frames (this is the
flicker to a higher resolution that one sees when capturing while a preview is
running).

However, if the *use_video_port* parameter is used to force a video-port based
image capture (see :ref:`rapid_capture`) then the mode change does not occur,
and the resulting video will not have dropped frames::

    import picamera

    with picamera.PiCamera() as camera:
        camera.resolution = (800, 600)
        camera.start_preview()
        camera.start_recording('foo.h264')
        camera.wait_recording(10)
        camera.capture('foo.jpg', use_video_port=True)
        camera.wait_recording(10)
        camera.stop_recording()

The above code should produce a 20 second video with no dropped frames, and a
still frame from 10 seconds into the video.

.. versionadded:: 0.8


.. _circular_record2:

Splitting to/from a circular stream
===================================

This example builds on the one in :ref:`circular_record1` and the one in
:ref:`record_and_capture` to demonstrate the beginnings of a security
application. As before, a :class:`~picamera.PiCameraCircularIO` instance is
used to keep the last few seconds of video recorded in memory. While the video
is being recorded, video-port-based still captures are taken to provide a
motion detection routine with some input (the actual motion detection algorithm
is left as an exercise for the reader).

Once motion is detected, the last 10 seconds of video are written to disk, and
video recording is split to another disk file to proceed until motion is no
longer detected. Once motion is no longer detected, we split the recording back
to the in-memory ring-buffer::

    import io
    import random
    import picamera
    from PIL import Image

    prior_image = None

    def detect_motion(camera):
        global prior_image
        stream = io.BytesIO()
        camera.capture(stream, format='jpeg', use_video_port=True)
        stream.seek(0)
        if prior_image is None:
            prior_image = Image.open(stream)
            return False
        else:
            current_image = Image.open(stream)
            # Compare current_image to prior_image to detect motion. This is
            # left as an exercise for the reader!
            result = random.randint(0, 10) == 0
            # Once motion detection is done, make the prior image the current
            prior_image = current_image
            return result

    def write_video(stream):
        # Write the entire content of the circular buffer to disk. No need to
        # lock the stream here as we're definitely not writing to it
        # simultaneously
        with io.open('before.h264', 'wb') as output:
            for frame in stream.frames:
                if frame.header:
                    stream.seek(frame.position)
                    break
            while True:
                buf = stream.read1()
                if not buf:
                    break
                output.write(buf)
        # Wipe the circular stream once we're done
        stream.seek(0)
        stream.truncate()

    with picamera.PiCamera() as camera:
        camera.resolution = (1280, 720)
        stream = picamera.PiCameraCircularIO(camera, seconds=10)
        camera.start_recording(stream, format='h264')
        try:
            while True:
                camera.wait_recording(1)
                if detect_motion(camera):
                    print('Motion detected!')
                    # As soon as we detect motion, split the recording to
                    # record the frames "after" motion
                    camera.split_recording('after.h264')
                    # Write the 10 seconds "before" motion to disk as well
                    write_video(stream)
                    # Wait until motion is no longer detected, then split
                    # recording back to the in-memory circular buffer
                    while detect_motion(camera):
                        camera.wait_recording(1)
                    print('Motion stopped!')
                    camera.split_recording(stream)
        finally:
            camera.stop_recording()

This example also demonstrates writing the circular buffer to disk in an
efficient manner using the :meth:`~picamera.PiCameraCircularIO.read1` method
(as opposed to :meth:`~picamera.CircularIO.read`).

.. note::

    Note that :meth:`~picamera.CircularIO.read1` does not guarantee to return
    the number of bytes requested, even if they are available in the underlying
    stream; it simply returns as many as are available from a single chunk up
    to the limit specified.

.. versionadded:: 1.0


.. _bayer_data:

Raw Bayer data captures
=======================

The ``bayer`` parameter of the :meth:`~picamera.PiCamera.capture` method
causes the raw Bayer data recorded by the camera's sensor to be output as
part of the image metadata.

.. note::

    The ``bayer`` parameter only operates with the JPEG format, and only
    for captures from the still port (i.e. when ``use_video_port`` is False,
    as it is by default).

Raw Bayer data differs considerably from simple unencoded captures; it is the
data recorded by the camera's sensor prior to *any* GPU processing including
auto white balance, vignette compensation, smoothing, down-scaling,
etc. This also means:

* Bayer data is *always* full resolution, regardless of the camera's output
  :attr:`~picamera.PiCamera.resolution` and any ``resize`` parameter.

* Bayer data occupies the last 6,404,096 bytes of the output file. The first
  32,768 bytes of this is header data which starts with the string ``'BRCM'``.

* Bayer data consists of 10-bit values, because this is the sensitivity of the
  `OV5647`_ sensor used by the Pi's camera. The 10-bit values are organized as
  4 8-bit values, followed by the low-order 2-bits of the 4 values packed into
  a fifth byte.

.. image:: bayer_bytes.svg
    :align: center

* Bayer data is organized in a BGGR pattern (a minor variation of the common
  `Bayer CFA`_). The raw data therefore has twice as many green pixels as red
  or blue and if viewed "raw" will look distinctly strange (too dark, too
  green, and with zippering effects along any straight edges).

.. image:: bayer_pattern.svg
    :align: center

* To make a "normal" looking image from raw Bayer data you will need to
  perform `de-mosaicing`_ at the very least, and probably some form of
  `color balance`_.

This (heavily commented) example script causes the camera to capture an image
including the raw Bayer data. It then proceeds to unpack the Bayer data into a
3-dimensional `numpy`_ array representing the raw RGB data and finally performs
a rudimentary de-mosaic step with weighted averages. A couple of numpy tricks
are used to improve performance but bear in mind that all processing is
happening on the CPU and will be considerably slower than normal image
captures::

    from __future__ import (
        unicode_literals,
        absolute_import,
        print_function,
        division,
        )


    import io
    import time
    import picamera
    import numpy as np
    from numpy.lib.stride_tricks import as_strided

    stream = io.BytesIO()
    with picamera.PiCamera() as camera:
        # Let the camera warm up for a couple of seconds
        time.sleep(2)
        # Capture the image, including the Bayer data
        camera.capture(stream, format='jpeg', bayer=True)

    # Extract the raw Bayer data from the end of the stream, check the
    # header and strip if off before converting the data into a numpy array

    data = stream.getvalue()[-6404096:]
    assert data[:4] == 'BRCM'
    data = data[32768:]
    data = np.fromstring(data, dtype=np.uint8)

    # The data consists of 1952 rows of 3264 bytes of data. The last 8 rows
    # of data are unused (they only exist because the actual resolution of
    # 1944 rows is rounded up to the nearest 16). Likewise, the last 24
    # bytes of each row are unused (why?). Here we reshape the data and
    # strip off the unused bytes

    data = data.reshape((1952, 3264))[:1944, :3240]

    # Horizontally, each row consists of 2592 10-bit values. Every four
    # bytes are the high 8-bits of four values, and the 5th byte contains
    # the packed low 2-bits of the preceding four values. In other words,
    # the bits of the values A, B, C, D and arranged like so:
    #
    #  byte 1   byte 2   byte 3   byte 4   byte 5
    # AAAAAAAA BBBBBBBB CCCCCCCC DDDDDDDD AABBCCDD
    #
    # Here, we convert our data into a 16-bit array, shift all values left
    # by 2-bits and unpack the low-order bits from every 5th byte in each
    # row, then remove the columns containing the packed bits

    data = data.astype(np.uint16) << 2
    for byte in range(4):
        data[:, byte::5] |= ((data[:, 4::5] >> ((4 - byte) * 2)) & 0b11)
    data = np.delete(data, np.s_[4::5], 1)

    # Now to split the data up into its red, green, and blue components. The
    # Bayer pattern of the OV5647 sensor is BGGR. In other words the first
    # row contains alternating green/blue elements, the second row contains
    # alternating red/green elements, and so on as illustrated below:
    #
    # GBGBGBGBGBGBGB
    # RGRGRGRGRGRGRG
    # GBGBGBGBGBGBGB
    # RGRGRGRGRGRGRG
    #
    # Please note that if you use vflip or hflip to change the orientation
    # of the capture, you must flip the Bayer pattern accordingly

    rgb = np.zeros(data.shape + (3,), dtype=data.dtype)
    rgb[1::2, 0::2, 0] = data[1::2, 0::2] # Red
    rgb[0::2, 0::2, 1] = data[0::2, 0::2] # Green
    rgb[1::2, 1::2, 1] = data[1::2, 1::2] # Green
    rgb[0::2, 1::2, 2] = data[0::2, 1::2] # Blue

    # At this point we now have the raw Bayer data with the correct values
    # and colors but the data still requires de-mosaicing and
    # post-processing. If you wish to do this yourself, end the script here!
    #
    # Below we present a fairly naive de-mosaic method that simply
    # calculates the weighted average of a pixel based on the pixels
    # surrounding it. The weighting is provided by a byte representation of
    # the Bayer filter which we construct first:

    bayer = np.zeros(rgb.shape, dtype=np.uint8)
    bayer[1::2, 0::2, 0] = 1 # Red
    bayer[0::2, 0::2, 1] = 1 # Green
    bayer[1::2, 1::2, 1] = 1 # Green
    bayer[0::2, 1::2, 2] = 1 # Blue

    # Allocate an array to hold our output with the same shape as the input
    # data. After this we define the size of window that will be used to
    # calculate each weighted average (3x3). Then we pad out the rgb and
    # bayer arrays, adding blank pixels at their edges to compensate for the
    # size of the window when calculating averages for edge pixels.

    output = np.empty(rgb.shape, dtype=rgb.dtype)
    window = (3, 3)
    borders = (window[0] - 1, window[1] - 1)
    border = (borders[0] // 2, borders[1] // 2)

    rgb_pad = np.zeros((
        rgb.shape[0] + borders[0],
        rgb.shape[1] + borders[1],
        rgb.shape[2]), dtype=rgb.dtype)
    rgb_pad[
        border[0]:rgb_pad.shape[0] - border[0],
        border[1]:rgb_pad.shape[1] - border[1],
        :] = rgb
    rgb = rgb_pad

    bayer_pad = np.zeros((
        bayer.shape[0] + borders[0],
        bayer.shape[1] + borders[1],
        bayer.shape[2]), dtype=bayer.dtype)
    bayer_pad[
        border[0]:bayer_pad.shape[0] - border[0],
        border[1]:bayer_pad.shape[1] - border[1],
        :] = bayer
    bayer = bayer_pad

    # In numpy >=1.7.0 just use np.pad (version in Raspbian is 1.6.2 at the
    # time of writing...)
    #
    #rgb = np.pad(rgb, [
    #    (border[0], border[0]),
    #    (border[1], border[1]),
    #    (0, 0),
    #    ], 'constant')
    #bayer = np.pad(bayer, [
    #    (border[0], border[0]),
    #    (border[1], border[1]),
    #    (0, 0),
    #    ], 'constant')

    # For each plane in the RGB data, we use a nifty numpy trick
    # (as_strided) to construct a view over the plane of 3x3 matrices. We do
    # the same for the bayer array, then use Einstein summation on each
    # (np.sum is simpler, but copies the data so it's slower), and divide
    # the results to get our weighted average:

    for plane in range(3):
        p = rgb[..., plane]
        b = bayer[..., plane]
        pview = as_strided(p, shape=(
            p.shape[0] - borders[0],
            p.shape[1] - borders[1]) + window, strides=p.strides * 2)
        bview = as_strided(b, shape=(
            b.shape[0] - borders[0],
            b.shape[1] - borders[1]) + window, strides=b.strides * 2)
        psum = np.einsum('ijkl->ij', pview)
        bsum = np.einsum('ijkl->ij', bview)
        output[..., plane] = psum // bsum

    # At this point output should contain a reasonably "normal" looking
    # image, although it still won't look as good as the camera's normal
    # output (as it lacks vignette compensation, AWB, etc).
    #
    # If you want to view this in most packages (like GIMP) you'll need to
    # convert it to 8-bit RGB data. The simplest way to do this is by
    # right-shifting everything by 2-bits (yes, this makes all that
    # unpacking work at the start rather redundant...)

    output = (output >> 2).astype(np.uint8)
    with open('image.data', 'wb') as f:
        output.tofile(f)

.. versionadded:: 1.3


.. _YUV: http://en.wikipedia.org/wiki/YUV
.. _YUV420: http://en.wikipedia.org/wiki/YUV#Y.27UV420p_.28and_Y.27V12_or_YV12.29_to_RGB888_conversion
.. _RGB: http://en.wikipedia.org/wiki/RGB
.. _RGBA: http://en.wikipedia.org/wiki/RGBA_color_space
.. _numpy: http://www.numpy.org/
.. _ring buffer: http://en.wikipedia.org/wiki/Circular_buffer
.. _OV5647: http://www.ovt.com/products/sensor.php?id=66
.. _Bayer CFA: http://en.wikipedia.org/wiki/Bayer_filter
.. _de-mosaicing: http://en.wikipedia.org/wiki/Demosaicing
.. _color balance: http://en.wikipedia.org/wiki/Color_balance


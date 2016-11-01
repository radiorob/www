# GStreamer 1.10 Release Notes

**NOTE: THESE RELEASE NOTES ARE VERY INCOMPLETE AND STILL WORK-IN-PROGRESS**

**GStreamer 1.10 is scheduled for release in TBD 2016.**

The GStreamer team is proud to announce a new major feature release in the
stable 1.x API series of your favourite cross-platform multimedia framework!

As always, this release is again packed with new features, bug fixes and other
improvements.

See [https://gstreamer.freedesktop.org/releases/1.10/][latest] for the latest
version of this document.

*Last updated: Saturday 22 Oct 2016, 16:00 UTC [(log)][gitlog]*

[latest]: https://gstreamer.freedesktop.org/releases/1.10/
[gitlog]: https://cgit.freedesktop.org/gstreamer/www/log/src/htdocs/releases/1.10/release-notes-1.10.md

## Introduction

The GStreamer team is proud to announce a new major feature release in the
stable 1.x API series of your favourite cross-platform multimedia framework!

As always, this release is again packed with new features, bug fixes and other
improvements.

## Highlights

- FILL ME

## Major new features and changes

### Noteworthy new API, features and other changes

#### Core API additions

##### Receive property change notifications via bus messages

New API was added to receive element property change notifications via
bus messages. So far, applications had to connect a callback to an element's
`notify::property-name` signal via the GObject API, which was inconvenient for
at least two reasons: one had to implement a signal callback function, and that
callback function would usually be called from one of the streaming threads, so
one had to marshal (send) any information gathered or pending requests to the
main application thread which was tedious and error-prone.

Enter [`gst_element_add_property_notify_watch()`][notify-watch] and
[`gst_element_add_property_deep_notify_watch()`][deep-notify-watch] which will
watch for changes of a property on the specified element, either only for this
element or recursively for a whole bin or pipeline. Whenever such a
property change happens, a `GST_MESSAGE_PROPERTY_NOTIFY` message will be posted
on the pipeline bus with details of the element, the property and the new
property value, all of which can be retrieved later from the message in the
application via [`gst_message_parse_property_notify()`][parse-notify]. Unlike
the GstBus watch functions, this API does not rely on a running GLib main loop.

The above can be used to be notified asynchronously of caps changes in the
pipeline, or volume changes on an audio sink element, for example.

[notify-watch]: https://gstreamer.freedesktop.org/data/doc/gstreamer/head/gstreamer/html/GstElement.html#gst-element-add-property-notify-watch
[deep-notify-watch]: https://gstreamer.freedesktop.org/data/doc/gstreamer/head/gstreamer/html/GstElement.html#gst-element-add-property-deep-notify-watch
[parse-notify]: https://gstreamer.freedesktop.org/data/doc/gstreamer/head/gstreamer/html/GstMessage.html#gst-message-parse-property-notify

##### GstBin "deep" element-added and element-removed signals

GstBin has gained `"deep-element-added"` and `"deep-element-removed"` signals
which makes it easier for applications and higher-level plugins to track when
elements are added or removed from a complex pipeline with multiple sub-bins.

`playbin` makes use of this to implement the new `"element-setup"` signal which
can be used to configure elements as they are added to `playbin`, just like the
existing `"source-setup"` signal which can be used to configure the source
element created.

##### Error messages can contain additional structured details

It is often useful to provide additional, structured information in error,
warning or info messages for applications (or higher-level elements) to make
intelligent decisions based on them. To allow this, error, warning and info
messages now have API for adding arbitrary additional information to them
using a `GstStructure`:
[`GST_ELEMENT_ERROR_WITH_DETAILS`][element-error-with-details] and the
corresponding for the other message types.

This is now used e.g. by the new [`GST_ELEMENT_FLOW_ERROR`][element-flow-error]
API to include the actual flow error in the error message, and the
[souphttpsrc element][souphttpsrc-detailed-errors] to provide the actual HTTP
status code, and if any, the URL to which a redirection has happened.

[element-error-with-details]: https://gstreamer.freedesktop.org/data/doc/gstreamer/head/gstreamer/html/GstElement.html#GST-ELEMENT-ERROR-WITH-DETAILS:CAPS
[element-flow-error]: https://gstreamer.freedesktop.org/data/doc/gstreamer/head/gstreamer/html/GstElement.html#GST-ELEMENT-FLOW-ERROR:CAPS
[souphttpsrc-detailed-errors]: https://cgit.freedesktop.org/gstreamer/gst-plugins-good/tree/ext/soup/gstsouphttpsrc.c?id=60d30db912a1aedd743e66b9dcd2e21d71fbb24f#n1318

##### Redirect messages have official API now

Sometimes, elements need to redirect the current stream URL and tell the
application to proceed with this new URL, possibly using a different
protocol (thus changing the pipeline configuration) too. Until now, this was
informally implemented using `ELEMENT` messages on the bus.

Now this has been formalized in form of a new `GST_MESSAGE_REDIRECT` message.
A new redirect message can be created using [`gst_message_new_redirect()`][new-redirect].
If needed, multiple redirect locations can be specified by calling
[`gst_message_add_redirect_entry()`][add-redirect] to add further redirect
entries, all with metadata, so the application can decide which is
most suitable (e.g. depending on the bitrate tags).

[new-redirect]: https://gstreamer.freedesktop.org/data/doc/gstreamer/head/gstreamer/html/GstMessage.html#gst-message-new-redirect
[add-redirect]: https://gstreamer.freedesktop.org/data/doc/gstreamer/head/gstreamer/html/GstMessage.html#gst-message-add-redirect-entry

##### New pad linking convenience functions that automatically create ghost pads

New pad linking convenience functions were added:
[`gst_pad_link_maybe_ghosting()`][pad-maybe-ghost] and
[`gst_pad_link_maybe_ghosting_full()`][pad-maybe-ghost-full] which were
previously internal to GStreamer have now been exposed for general use.

The existing pad link functions will refuse to link pads or elements at
different levels in the pipeline hierarchy, requiring the developer to
create ghost pads where necessary. These new utility functions will
automatically create ghostpads as needed when linking pads at different
levels of the hierarchy (e.g. from an element inside a bin to one that's at
the same level in the hierarchy as the bin, or in another bin).

[pad-maybe-ghost]: https://gstreamer.freedesktop.org/data/doc/gstreamer/head/gstreamer/html/GstPad.html#gst-pad-link-maybe-ghosting
[pad-maybe-ghost-full]: https://gstreamer.freedesktop.org/data/doc/gstreamer/head/gstreamer/html/GstPad.html#gst-pad-link-maybe-ghosting-full

##### Miscellaneous

Pad probes: IDLE and BLOCK probes now work slightly differently in pull mode,
so that push and pull mode have opposite scenarios for idle and blocking probes.
In push mode, it will block with some data type and IDLE won't have any data.
In pull mode, it will block _before_ getting a buffer and will be IDLE once some
data has been obtained. ([commit][commit-pad-probes], [bug][bug-pad-probes])

[commit-pad-probes]: https://cgit.freedesktop.org/gstreamer/gstreamer/commit/gst/gstpad.c?id=368ee8a336d0c868d81fdace54b24431a8b48cbf
[bug-pad-probes]: https://bugzilla.gnome.org/show_bug.cgi?id=761211

[`gst_parse_launch_full()`][parse-launch-full] can now be made to return a
`GstBin` instead of a top-level pipeline by passing the new
`GST_PARSE_FLAG_PLACE_IN_BIN` flag.

[parse-launch-full]: https://gstreamer.freedesktop.org/data/doc/gstreamer/head/gstreamer/html/gstreamer-GstParse.html#gst-parse-launch-full

The default GStreamer debug log handler can now be removed before
calling `gst_init()`, so that it will never get installed and won't be active
during initialization.

A new [`STREAM_GROUP_DONE` event][stream-group-done-event] was added. In some
ways it works similar to the `EOS` event in that it can be used to unblock
downstream elements which may be waiting for further data, such as for example
`input-selector`. Unlike `EOS`, further data flow may happen after the
`STREAM_GROUP_DONE` event though (and without the need to flush the pipeline).
This is used to unblock input-selector when switching between streams in
adaptive streaming scenarios (e.g. HLS).

[stream-group-done-event]: https://gstreamer.freedesktop.org/data/doc/gstreamer/head/gstreamer/html/GstEvent.html#gst-event-new-stream-group-done

The `gst-launch-1.0` command line tool will now print unescaped caps in verbose
mode (enabled by the -v switch).

[`gst_element_call_async()`][call-async] has been added as convenience API for
plugin developers. It is useful for one-shot operations that need to be done
from a thread other than the current streaming thread. It is backed by a
thread-pool that is shared by all elements.

[call-async]: https://gstreamer.freedesktop.org/data/doc/gstreamer/head/gstreamer/html/GstElement.html#gst-element-call-async

Various race conditions have been fixed around the `GstPoll` API used by e.g.
`GstBus` and `GstBufferPool`. Some of these manifested themselves primarily
on Windows.

`GstAdapter` can now keep track of discontinuities signalled via the `DISCONT`
buffer flag, and has gained [new API][new-adapter-api] to track PTS, DTS and
offset at the last discont. This is useful for plugins implementing advanced
trick mode scenarios.

[new-adapter-api]: https://gstreamer.freedesktop.org/data/doc/gstreamer/head/gstreamer-libs/html/GstAdapter.html#gst-adapter-pts-at-discont

`GstTestClock` gained a new [`"clock-type"` property][clock-type-prop].

[clock-type-prop]: https://gstreamer.freedesktop.org/data/doc/gstreamer/head/gstreamer-libs/html/GstTestClock.html#GstTestClock--clock-type

#### GstStream API for stream announcement and stream selection

New stream listing and stream selection API: new API has been added to
provide high-level abstractions for streams ([`GstStream`][stream-api])
and collections of streams ([`GstStreamCollections`][stream-collection-api]).

##### Stream listing

A [`GstStream`][stream-api] contains all the information pertinent to a stream,
such as stream id, caps, tags, flags and stream type(s); it can represent a
single elementary stream (e.g. audio, video, subtitles, etc.) or a container
stream. This will depend on the context. In a decodebin3/playbin3 one
it will typically be elementary streams that can be selected and unselected.

A [`GstStreamCollection`][stream-collection-api] represents a group of streams
and is used to announce or publish all available streams. A GstStreamCollection
is immutable - once created it won't change. If the available streams change,
e.g. because a new stream appeared or some streams disappeared, a new stream
collection will be published. This new stream collection may contain streams
from the previous collection if those streams persist, or completely new ones.
Stream collections do not yet list all theoretically available streams,
e.g. other available DVD angles or alternative resolutions/bitrate of the same
stream in case of adaptive streaming.

New events and messages have been added to notify or update other elements and
the application about which streams are currently available and/or selected.
This way, we can easily and seamlessly let the application know whenever the
available streams change, as happens frequently with digital television streams
for example. The new system is also more flexible. For example, it is now also
possible for the application to select multiple streams of the same type
(e.g. in a transcoding/transmuxing scenario).

A [`STREAM_COLLECTION` message][stream-collection-msg] is posted on the bus
to inform the parent bin (e.g. `playbin3`, `decodebin3`) and/or the application
about what streams are available, so you no longer have to hunt for this
information (number of streams of each type, caps, tags etc.) at different
places. Bins and/or the application can intercept this message synchronously
to select and deselect streams before any data is produced (for the case where
elements such as the demuxers support the new stream API, not necessarily in
the parsebin compatibility fallback case). Similarly, there is also a
[`STREAM_COLLECTION` event][stream-collection-event] to inform downstream
elements of the available streams. This event can be used by elements to
aggregate streams from multiple inputs into one single collection.

The `STREAM_START` event was extended so that it can also contain a GstStream
object with all information about the current stream, see
[`gst_event_set_stream()`][event-set-stream] and
[`gst_event_parse_stream()`][event-parse-stream].
[`gst_pad_get_stream()`][pad-get-stream] is a new utility function that can be
used to look up the GstStream from the `STREAM_START` sticky event on a pad.

[stream-api]: https://gstreamer.freedesktop.org/data/doc/gstreamer/head/gstreamer/html/gstreamer-GstStream.html
[stream-collection-api]: https://gstreamer.freedesktop.org/data/doc/gstreamer/head/gstreamer/html/gstreamer-GstStreamCollection.html
[stream-collection-msg]: https://gstreamer.freedesktop.org/data/doc/gstreamer/head/gstreamer/html/GstMessage.html#gst-message-new-stream-collection
[stream-collection-event]: https://gstreamer.freedesktop.org/data/doc/gstreamer/head/gstreamer/html/GstEvent.html#gst-event-new-stream-collection
[event-set-stream]: https://gstreamer.freedesktop.org/data/doc/gstreamer/head/gstreamer/html/GstEvent.html#gst-event-set-stream
[event-parse-stream]: https://gstreamer.freedesktop.org/data/doc/gstreamer/head/gstreamer/html/GstEvent.html#gst-event-parse-stream
[pad-get-stream]: https://gstreamer.freedesktop.org/data/doc/gstreamer/head/gstreamer/html/GstPad.html#gst-pad-get-stream

##### Stream selection

Once the available streams have been published, streams can be selected via
their stream ID using the new `SELECT_STREAMS` event, which can be created
with [`gst_event_new_select_streams()`][event-select-streams]. The new API
supports selecting multiple streams per stream type. In the future, we may also
implement explicit deselection of streams that will never be used, so
elements can skip these and never expose them or output data for them in the
first place.

The application is then notified of the currently selected streams via the
new `STREAMS_SELECTED` message on the pipeline bus, containing both the current
stream collection as well as the selected streams. This might be posted in
response to the application sending a `SELECT_STREAMS` event or when
`decodebin3` or `playbin3` decide on the streams initially selected without
application input.

[event-select-streams]: https://gstreamer.freedesktop.org/data/doc/gstreamer/head/gstreamer/html/GstEvent.html#gst-event-new-select-streams

##### Further reading

See further below for some notes on the new elements supporting this new
stream API, namely: `decodebin3`, `playbin3` and `parsebin`.

More information about the new API and the new elements can also be found here:

- GStreamer [stream selection design docs][streams-design]
- Edward Hervey's talk ["The new streams API: Design and usage"][streams-talk] ([slides][streams-slides])
- Edward Hervey's talk ["Decodebin3: Dealing with modern playback use cases"][db3-talk] ([slides][db3-slides])

[streams-design]: https://cgit.freedesktop.org/gstreamer/gstreamer/tree/docs/design/part-stream-selection.txt
[streams-talk]: https://gstconf.ubicast.tv/videos/the-new-gststream-api-design-and-usage/
[streams-slides]: https://gstreamer.freedesktop.org/data/events/gstreamer-conference/2016/Edward%20Hervey%20-%20The%20New%20Streams%20API%20Design%20and%20Usage.pdf
[db3-talk]: https://gstconf.ubicast.tv/videos/decodebin3-or-dealing-with-modern-playback-use-cases/
[db3-slides]: https://gstreamer.freedesktop.org/data/events/gstreamer-conference/2015/Edward%20Hervey%20-%20decodebin3.pdf

#### Audio conversion and resampling API

The audio conversion library received a completely new and rewritten audio
resampler, complementing the audio conversion routines moved into the audio
library in the [previous release][release-notes-1.8]. Integrating the resampler
with the other audio conversion library allows us to implement generic
conversion much more efficiently, as format conversion and resampling can now
be done in the same processing loop instead of having to do it in separate
steps (our element implementations do not make use of this yet though).

The new audio resampler library is a combination of some of the best features
of other samplers such as ffmpeg, speex and SRC. It natively supports S16, S32,
F32 and F64 formats and uses optimized x86 and neon assembly for most of its
processing. It also has support for dynamically changing sample rates by incrementally
updating the filter tables using linear or cubic interpolation. According to
some benchmarks, it's one of the fastest and most accurate resamplers around.

The `audioresample` plugin has been ported to the new audio library functions
to make use of the new resampler.

[release-notes-1.8]: https://gstreamer.freedesktop.org/releases/1.8/

#### Support for SMPTE timecodes

- FILL ME

- API, GstMeta, elements, element support

#### GStreamer OpenMAX IL plugin

The last gst-omx release, 1.2.0, was in July 2014. It was about time to get
a new one out with all the improvements that happened in the meantime.
From now on, we will try to release gst-omx together with all other modules.

This release features a lot of bugfixes, improved support for the Raspberry Pi
and in general improved support for zerocopy rendering via EGL and a few minor
new features.

At this point, gst-omx is known to work best on the Raspberry Pi platform but
it is also known to work on various other platforms. Unfortunately, we are
not including configurations for any other platforms, so if you happen to use
gst-omx: please send us patches with your configuration and code changes!

### New Elements

#### decodebin3, playbin3, parsebin (experimental)

This release features new decoding and playback elements as experimental
technology preview: `decodebin3` and `playbin3` will soon supersede the
existing `decodebin` and `playbin` elements. We skipped the number 2 because
it was already used back in the 0.10 days, which might cause confusion.
Experimental technology preview means that everything should work fine already,
but we can't guarantee there won't be minor behavioral changes in the
next cycle. In any case, please test and report any problems back.

Before we go into detail about what these new elements improve, let's look at
the new [`parsebin`][parsebin] element. It works similarly to `decodebin` and
`decodebin3`, only that it stops one step short and does not plug any actual
decoder elements. It will only plug parsers, tag readers, demuxers and
depayloaders. Also note that parsebin does not contain any queueing element.

[`decodebin3`'s][decodebin3] internal architecture is slightly different from
the existing `decodebin` element and fixes many long-standing issues with our
decoding engine. For one, data is now fed into the internal `multiqueue` element
*after* it has been parsed and timestamped, which means that the `multiqueue`
element now has more knowledge and is able to calculate the interleaving of the
various streams, thus minimizing memory requirements and doing away with magic
values for buffering limits that were conceived when videos were 240p or 360p
and which anyone who's tried to play back 4k video streams with decodebin2
will have noticed. The improved timestamp tracking also enables `multiqueue`
to keep streams of the same type (audio, video) aligned better, making sure
switching between streams of the same type is very fast.

Another major improvement in `decodebin3` is that it will no longer decode
streams that are not being used. With the old `decodebin` and `playbin`, when
there were 8 audio streams we would always decode all 8 streams even
if 7 were not actually used. This caused a lot of
CPU overhead, which was particularly problematic on embedded devices. When
switching between streams `decodebin3` will try hard to re-use existing
decoders. This is useful when switching between multiple streams of the same
type if they are encoded in the same format.

The above is also useful when the available streams change on the fly, as might
happen with radio streams (chained Oggs), digital television broadcasts, when
adaptive streaming streams change bitrate, or when switching gaplessly
to the next title. In order to guarantee a seamless transition, the old
`decodebin2` would plug a second decoder for the new stream while finishing
up the old stream. With `decodebin3`, this is no longer needed. At least not
when the new and old format are the same. This will be particularly useful
on embedded systems where it is often not possible to run multiple decoders
at the same time, or tearing down and setting up decoders is fairly expensive.

`decodebin3` also allows for multiple input streams, not just a single one.
This will be useful, in the future, for gapless playback, or for feeding
multiple external subtitle streams to decodebin/playbin.

`playbin3` uses `decodebin3` internally, and will supersede `playbin`.
It was decided that it would be too risky to make the old `playbin` use the
new `decodebin3` in a backwards-compatible way. The new architecture
makes it awkward, if not impossible, to maintain perfect backwards compatibility
in some aspects, hence `playbin3` was born, and developers can migrate to the
new element and new API at their own pace.

All of these new elements make use of the new `GstStream` API for listing and
selecting streams, as described above. `parsebin` provides backwards
compatibility for demuxers and parsers which do not advertise their streams
using the new API yet (which is most).

The new elements are not entirely feature-complete yet: `playbin3` does not
support so-called decodersinks yet where the data is not decoded inside
GStreamer but passed directly for decoding to the sink. `decodebin3` is missing
the various `autoplug-*` signals to influence which decoders get autoplugged
in which oder. We're looking to add back this functionality, but it will probably
be in a different way, with a single unified signal and using GstStream perhaps.

For more information on these new elements, check out Edward Hervey's talk
[*decodebin3 - dealing with modern playback use cases*][db3-talk]

[parsebin]: https://gstreamer.freedesktop.org/data/doc/gstreamer/head/gst-plugins-base-plugins/html/gst-plugins-base-plugins-parsebin.html
[decodebin3]: https://gstreamer.freedesktop.org/data/doc/gstreamer/head/gst-plugins-base-plugins/html/gst-plugins-base-plugins-decodebin3.html
[db3-talk]: https://gstconf.ubicast.tv/videos/decodebin3-or-dealing-with-modern-playback-use-cases/

#### LV2 ported from 0.10 and switched from slv2 to lilv2

The LV2 wrapper plugin has been ported to 1.0 and moved from using the
deprecated slv2 library to its replacement lilv2. We support sources and
filter elements. lv2 is short for *Linux Audio Developer's Simple Plugin API
(LADSPA) version 2* and is an open standard for audio plugins which includes
support for audio synthesis (generation), digital signal processing of digital
audio, and MIDI. The new lv2 plugin supersedes the existing LADSPA plugin.

#### WebRTC DSP Plugin for echo-cancellation, gain control and noise suppression

A set of new elements ([webrtcdsp][webrtcdsp], [webrtcechoprobe][webrtcechoprobe])
based on the WebRTC DSP software stack can now be used to improve your audio
voice communication pipelines. It currently supports echo cancellation,
gain control, noise suppression and more. For more details you may read
[Nicolas' blog post][webrtc-blog-post].

[webrtcdsp]: https://gstreamer.freedesktop.org/data/doc/gstreamer/head/gst-plugins-bad-plugins/html/gst-plugins-bad-plugins-webrtcdsp.html
[webrtcechoprobe]: https://gstreamer.freedesktop.org/data/doc/gstreamer/head/gst-plugins-bad-plugins/html/gst-plugins-bad-plugins-webrtcechoprobe.html
[webrtc-blog-post]: https://ndufresne.ca/2016/06/gstreamer-echo-canceller/

#### Fraunhofer FDK AAC encoder and decoder

New encoder and decoder elements wrapping the Fraunhofer FDK AAC library have
been added (`fdkaacdec`, `fdkaacdec`). The Fraunhofer FDK AAC encoder is
generally considered to be a very high-quality AAC encoder, but unfortunately
it comes under a non-free license with the option to obtain a paid, commercial
license.

### Noteworthy element features and additions

#### Major RTP and RTSP improvements

RTX, jitterbuffer, RTSP server fixes, rtspsrc seeking stability
App and protocol specific RTCP
RFC7273 support
H265 payloader sync with RFC


- FILL ME

#### Improvements to splitmuxsrc and splitmuxsink

Reliability and error handling improvements, removing at least one deadlock
case. `splitmuxsrc` now stops cleanly at the end of the segment when handling
a segment seek. We fixed a bug with large amounts of downstream buffering
causing incorrect out-of-sequence playback.

`splitmuxsrc` now has a `"format-location"` signal to directly specify the list
of files to play from.

#### OpenGL/GLES improvements

##### iOS and macOS (OS/X)

- We now create OpenGL|ES 3.x contexts on iOS by default with a fallback to
  OpenGL|ES 2.x if that fails.
- Various zerocopy decoding fixes and enhancements with the
  encoding/decoding/capturing elements.
- libdispatch is now used on all Apple platforms instead of GMainLoop, removing
  the expensive poll()/pthread_*() overhead.

##### New API

- `GstGLFramebuffer` - for wrapping OpenGL frame buffer objects.  It provides
  facilities for attaching `GstGLMemory` objects to the necessary attachment
  points, binding and unbinding and running a user-supplied function with the
  framebuffer bound.
- `GstGLRenderbuffer` (a `GstGLBaseMemory` subclass) - for wrapping OpenGL
  render buffer objects that are typically used for depth/stencil buffers or
  for color buffers where we don't care about the output.
- `GstGLMemoryEGL` (a `GstGLMemory` subclass) - for combining `EGLImage`s with a GL
  texture that replaces `GstEGLImageMemory` bringing the improvements made to the
  other `GstGLMemory` implementations.  This fixes a performance regression in
  zerocopy decoding on the Raspberry Pi when used with an updated gst-omx.

##### Miscellaneous improvements

- `gltestsrc` is now usable on devices/platforms with OpenGL 3.x and OpenGL|ES
  and has completed or gained support for new patterns in line with the
  existing ones in `videotestsrc`.
- `gldeinterlace` is now available on devices/platforms with OpenGL|ES
  implementations.
- The dispmanx backend (used on the Raspberry Pi) now supports the
  `gst_video_overlay_set_window_handle()` and
  `gst_video_overlay_set_render_rectangle()` functions.
- The `gltransformation` element now correctly transforms mouse coordinates (in
  window space) to stream coordinates for both perspective and orthographic
  projections.
- The `gltransformation` element now detects if the
  `GstVideoAffineTransformationMeta` is supported downstream and will efficiently
  pass its transformation downstream. This is a performance improvement as it
  results in less processing being required.
- The wayland implementation now uses the multi-threaded safe event-loop API
  allowing correct usage in applications calling wayland functions from
  multiple threads.
- Support for native 90 degree rotations and horizontal/vertical flips
  in `glimagesink`.

#### Vulkan

- The Vulkan elements now work under Wayland and have received numerous
  bugfixes.

#### QML elements

- `qmlglsink` video sink now works on more platforms, notably, Windows, Wayland,
  and Qt's eglfs (for embedded devices with an OpenGL implementation) including
  the Raspberry Pi.
- New element `qmlglsrc` to record a QML scene into a GStreamer pipeline.

#### KMS video sink

- New element `kmssink` to render video using Direct Rendering Manager
  (DRM) and Kernel Mode Setting (KMS) subsystems in the Linux
  kernel. It is oriented to be used mostly in embedded systems.

#### Wayland video sink

- `waylandsink` now supports the wl_viewporter extension allowing
  video scaling and cropping to be delegated to the Wayland
  compositor. This extension is also been made optional, so that it can
  also work on current compositors that don't support it. He's also added
  support for the video meta, allowing zero-copy operations in more
  cases.

#### DVB improvements

- `dvbsrc` has now better delivery-system autodetection and several
new parameter sanity-checks to improve its resilience to configuration
omissions and errors. Superfluous polling has continued to be trimmed,
and the debugging output made more consistent and precise. Additionally,
the channel-configuration parser now supports the new dvbv5 format,
enabling `dvbbasebin` to automatically playback content transmitted on
delivery systems that previously required manual description, like
ISDB-T.

#### DASH, HLS and adaptivedemux

- HLS now has support for Alternate Rendition audio and video tracks
- Full support for Alternate Rendition subtitle tracks coming soon
- Lots of reliability fixes around seek handling and bitrate switching.
- FILL ME - trick modes?

#### a2dpsink finally works

- FILL ME

#### GStreamer VAAPI

- All the decoders have been split, one plugin feature per codec. So
  far, the available ones, depending on the driver, are:
  `vaapimpeg2dec`, `vaapih264dec`, `vaapih265dec`, `vaapivc1dec`, `vaapivp8dec`,
  `vaapivp9dec` and `vaapijpegdec` (which already was split).
- Improvements when mapping VA surfaces into memory, differenciating
  from negotiation caps and allocations caps, since the allocation
  memory for surfaces may be bigger than one that is going to be
  mapped.
- `vaapih265enc` got support to constant bitrate mode (CBR).
- Since several VA drivers are unmaintained, we decide to keep a white list
  with the va drivers we actually test, which is mostly the i915 and, in some
  degree, gallium from mesa project. Exporting the environment variable
  `GST_VAAPI_ALL_DRIVERS` disable the white list.
- The plugin features are registered, in run-time, according to its support by
  the loaded VA driver. So only the decoders and encoder supported by the
  system are registered. Since the driver can change, some dependencies are
  tracked to invalidate the GStreamer registry and reload the plugin.
- dmabuf importation from upstream has been improved, gaining performance.
- `vaapipostproc` now can negotiate through caps the buffer transformations.
- Decoders now can do reverse playback. They only shows I frames, because the
  surface pool is smaller than the required by the GOP to show all the
  frames.
- The upload of frames onto native GL textures has been optimized too, keeping
  a cache of the internal structures for the offered textures by the sink.

#### V4L2 changes

- More pixels formats are now supported
- Decoder is now using `G_SELECTION` instead of deprecated `G_CROP`
- Decoder now uses `STOP` command to handle EOS
- Transform element can now scale the pixel aspect ratio
- Colorimetry support has been improved even more
- We now support `OUTPUT_OVERLAY` type of video node in v4l2sink

#### Miscellaneous

- `multiqueue`'s input pads gained a new `"group-id"` property which
  can be used to group input streams. Typically one will assign
  different id numbers to audio, video and subtitle streams, for
  example. This way `multiqueue` can make sure streams of the same
  type advance in lockstep if some of the streams are unlinked and the
  `"sync-by-running-time"` property is set. This is used in
  decodebin3/playbin3 to implement almost-instantaneous stream
  switching.  The grouping is required because different downstream
  paths (audio, video, etc.)  may have different buffering/latency
  etc. so might be consuming data from multiqueue with a slightly
  different phase, and if we track different stream groups separately
  we minimize stream switching delays and buffering inside the
  `multiqueue`.
- `alsasrc` now supports ALSA drivers without a position for each
  channel, this is common in some professional or industrial hardware.
- libvpx based decoders (`vp8dec` and `vp9dec`) now create multiple threads on
  computers with multiple CPUs automatically.
- `rfbsrc` which can receive from a VNC server has seen a lot of
  debugging, it now supports the latest version of the RFB
  protocol. It also uses GIO everywhere.
- `tsdemux` can now read ATSC E-AC-3 streams.

### Plugin moves

No plugins were moved this cycle. We'll make up for it next cycle, promise!

### Rewritten memory leak tracer

GStreamer has had basic functionality to trace allocation and freeing of
both mini objects (buffers, events, caps, etc.) and objects in form of the
internal `GstAllocTrace` tracing system, whose API was never exposed in the
1.x API series. This would at exit dump a list of objects and mini objects
which had still not been freed at that point if so requested via an environment
variable. This subsystem has now been removed in favour of a new implementation
based on the recently-added new tracing framework.

New tracing hooks have been added to trace the creation and destruction of
GstObjects and mini objects, and a new tracer plugin has been written using
those new hooks to track which objects are still live and which are not. If
GStreamer has been compiled against the libunwind library, the new leaks tracer
will remember where objects were allocated from as well. By default the leaks
tracer will simply output a warning if leaks have been detected on `gst_deinit()`.

If the `GST_LEAKS_TRACER_SIG` environment variable is set, the leaks tracer
will also handle the following UNIX signals:

 - `SIGUSR1`: log alive objects
 - `SIGUSR2`: create a checkpoint and print a list of objects created and
   destroyed since the previous checkpoint.

This will not work on Windows though.

If the `GST_LEAKS_TRACER_STACK_TRACE` environment variable is set, the leaks
tracer will also log the creation stack trace of leaked objects. This may
significantly increase memory consumption however.

New `MAY_BE_LEAKED` flags have been added to GstObject and GstMiniObject, so
that objects and mini objects that are likely to stay around forever can be
flagged and blacklisted from the leak output.

To give the new leak tracer a spin, simply call any GStreamer application such
as `gst-launch-1.0` or `gst-play-1.0` like this:

    GST_TRACERS=leaks gst-launch-1.0 videotestsrc num-buffers=10 ! fakesink

If there are any leaks, a warning will be raised at the end.

It is also possible to trace only certain types of objects or mini objects:

    GST_TRACERS="leaks(GstEvent,GstMessage)" gst-launch-1.0 videotestsrc num-buffers=10 ! fakesink

This dedicated leaks tracer is much much faster than valgrind since all code is
executed natively instead of being instrumented. This makes it very suitable
for use on slow machines or embedded devices. It is however limited to certain
types of leaks and won't catch memory leaks when the allocation has been made
via plain old `malloc()` or `g_malloc()` or other means. It will also not trace
non-GstObject GObjects.

The goal is to enable leak tracing on GStreamer's Continuous-Integration and
testing system, both for the regular unit tests (make check) and media tests
(gst-validate), so that accidental leaks in common code paths can be detected
and fixed quickly.

For more information about the new tracer, check out Guillaume Desmottes's
["Tracking Memory Leaks"][leaks-talk] talk or his [blog post][leaks-blog] about
the topic.

[leaks-talk]: https://gstconf.ubicast.tv/videos/tracking-memory-leaks/
[leaks-blog]: https://blog.desmottes.be/?post/2016/06/20/GStreamer-leaks-tracer

### GES and NLE changes

* Clip priorities are now handled by the layers, and the GESTimelineElement
  priority property is now deprecated and unused
* Enhance (de)interlacing support always using the `deinterlace` element
  and exposing needed properties to users
* Allow reusing clips children after removing the clip from a layer
* We are now testing many more rendering format in the gst-validate
  test suite, and failures have been fixed.
* Also many bugs have been fixed in this cycle!

### GStreamer validate changes

This cycle has been focused on making GstValidate more than just a validating
tool but also a tool to help developers debug their GStreamer issues. When
reporting, issues, we try to gather as much information as possible and expose
it to end users in a useful way. For an example of such enhancements, check out
Thibault Saunier's [blog post](improving-debugging-gstreamer-validate) about
the new Not Negotiated Error reporting mechanism.

Playbin3 support has been added so we can run validate tests with playbin3
instead of playbin.

We are now able to properly communicate between `gst-validate-launcher` and
launched subprocesses with actual IPC between them. It enabled the test
launcher to handle failing tests specifying the exact expected issue(s).

[improving-debugging-gstreamer-validate]: https://blogs.s-osg.org/improving-debugging-gstreamer-validate/

### gst-libav changes

gst-libav uses the recently released ffmpeg 3.2 now, which brings a lot of
improvements and bugfixes from the ffmpeg team in addition to various new
codec mappings on the GStreamer side and quite a few bugfixes to the GStreamer
integration to make it more robust.

## Miscellaneous

- New video orientation interface
- `appsrc` duration in time and try pull API
- `x264enc` has support for chroma-site and colorimetry settings
- JPEG2000 parser and caps cleanup
- Reverse playback support for `videorate`, `deinterlace`
- Various improvements for reverse playback and `KEY_UNITS` trick mode
- New cleaned up `rawaudioparse`, `rawvideoparse` elements
- Decklink 10 bit and timecode support, various fixes
- Multiview and other new API for GstPlayer
- GstBin suppressed flags API

- FILL ME

## Build and Dependencies

### Experimental support for Meson as build system

#### Overview

We have have added support for building GStreamer using the
[Meson build system][meson]. This is currently experimental, but should work
fine at least on Linux using the gcc or clang toolchains and on Windows using
the MingW or MSVC toolchains.

Autotools remains the primary build system for the time being, but we hope to
someday replace it and will steadily work towards that goal.

More information about the background and implications of all this and where
we're hoping to go in future with this can be found in [Tim's mail][meson-mail]
to the gstreamer-devel mailing list.

For more information on Meson check out [these videos][meson-videos] and also
the [Meson talk][meson-gstconf] at the GStreamer Conference.

Immediate benefits for Linux users are faster builds and rebuilds. At the time
of writing the Meson build of GStreamer is used by default in GNOME's jhbuild
system.

The Meson build currently still lacks many of the fine-grained configuration
options to enable/disable specific plugins. These will be added back in due
course.

Note: The meson build files are not disted in the source tarballs, you will
need to get GStreamer from git if you want try it out.

[meson]: http://mesonbuild.com/
[meson-mail]: https://lists.freedesktop.org/archives/gstreamer-devel/2016-September/060231.html
[meson-videos]: http://mesonbuild.com/videos.html
[meson-gstconf]: https://gstconf.ubicast.tv/videos/gstreamer-development-on-windows-ans-faster-builds-everywhere-with-meson/

#### Windows Visual Studio toolchain support

Windows users might appreciate being able to build GStreamer using the MSVC
toolchain, which is not possible using autotools. This means that it will be
possible to debug GStreamer and applications in Visual Studio, for example.
We require VS2015 or newer for this at the moment.

There are two ways to build GStreamer using the MSVC toolchain:

1. Using the MSVC command-line tools (`cl.exe` etc.) via Meson's "ninja" backend.
2. Letting Meson's "vs2015" backend generate Visual Studio project files that
   can be opened in Visual Studio and compiled from there.

This is currently only for adventurous souls though. All the bits are in place,
but support for all of this has not been merged into GStreamer's cerbero build
tool yet at the time of writing. This will hopefully happen in the next cycle,
but for now this means that those wishing to compile GStreamer with MSVC will
have to get their hands dirty.

There are also no binary SDK builds using the MSVC toolchain yet.

For more information on GStreamer builds using Meson and the Windows toolchain
check out Nirbheek Chauhan's blog post ["Building and developing GStreamer using Visual Studio"][msvc-blog].

[msvc-blog]: http://blog.nirbheek.in/2016/07/building-and-developing-gstreamer-using.html

### Dependencies

#### gstreamer

libunwind was added as an optional dependency. It is used only for debugging
and tracing purposes.

The `opencv` plugin in gst-plugins-bad can now be built against OpenCV
version 3.1, previously only 2.3-2.5 were supported.

#### gst-plugins-ugly

- `mpeg2dec` now requires at least libmpeg2 0.5.1 (from 2008).

- `gltransformation` now requires at least graphene 1.4.0.

- `lv2` now plugin requires at least lilv 0.16 instead of slv2.

### Packaging notes

Packagers please note that the `gst/gstconfig.h` public header file in the
GStreamer core library moved back from being an architecture dependent include
to being architecture independent, and thus it is no longer installed into
`$(libdir)/gstreamer-1.0/include/gst` but into the normal include directory
where it lives happily ever after with all the other public header files. The
reason for this is that we now check whether the target supports unaligned
memory access based on predefined compiler macros at compile time instead of
checking it at configure time.

## Platform-specific improvements

### Android

#### New universal binaries for all supported ABIs

We now provide a "universal" tarball to allow building apps against all the
architectures currently supported (x86, x86-64, armeabi, armeabi-v7a,
armeabi-v8a). This is needed for building with recent versions of the Android
NDK which defaults to building against all supported ABIs. Use [the Android
player example][android-player-example-build] as a reference of the required
changes.

[android-player-example-build]: https://cgit.freedesktop.org/gstreamer/gst-examples/commit/playback/player/android?id=a5cdde9119f038a1eb365aca20faa9741a38e788

#### Miscellaneous

- FILL ME

### macOS (OS/X) and iOS

- We now support querying available devices on OS/X via the GstDeviceProvider
  API.
- We can now create OpenGL|ES 3.x contexts on iOS.
- many OpenGL/GLES improvements, see OpenGL section above

### Windows

- gstconfig.h: Always use dllexport/import on Windows with MSVC
- Miscellaneous fixes to make libs and plugins compile with the MVSC toolchain
- MSVC toolchain support (see Meson section above for more details)

## New Modules for Documentation, Examples, Meson Build

Three new git modules have been added recently:

### gst-docs

This is a new module where we will maintain documentation in markup format.

It contains the former gstreamer.com SDK tutorials which have kindly been made
available by Fluendo under a Creative Commons license. The tutorials have been
reviewed and updated for GStreamer 1.x and will be available as part of the
[official GStreamer documentation][doc] going forward. The old gstreamer.com
site will then be shut down with redirects pointing to the updated tutorials.

Some of the existing docbook XML-formatted documentation from the GStreamer
core module such as the *Application Development Manual* and the *Plugin
Writer's Guide* have been converted to markup as well and will be maintained in
the gst-docs module in future. They will be removed from the GStreamer core
module in the next cycle.

This is just the beginning. Our goal is to provide a more cohesive documentation
experience for our users going forward, and easier to create and maintain
documentation for developers. There is a lot more work to do, get in touch if
you want to help out.

If you encounter any problems or spot any omissions or outdated content in the
new documentation, please [file a bug in bugzilla][doc-bug] to let us know.

We will probably release gst-docs as a separate tarball for distributions to
package in the next cycle.

[doc]: http://gstreamer.freedesktop.org/documentation/
[doc-bug]: https://bugzilla.gnome.org/enter_bug.cgi?product=GStreamer&component=documentation

### gst-examples

A new [module][examples-git] has been added for examples. It does not contain
much yet, currently it only contains a small [http-launch][http-launch] utility
that serves a pipeline over http as well as various [GstPlayer playback frontends][puis]
for Android, iOS, Gtk+ and Qt.

More examples will be added over time. The examples in this repository should
be more useful and more substantial than most of the examples we ship as part
of our other modules, and also written in a way that makes them good example
code. If you have ideas for examples, let us know.

No decision has been made yet if this module will be released and/or packaged.
It probably makes sense to do so though.

[examples-git]: https://cgit.freedesktop.org/gstreamer/gst-examples/tree/
[http-launch]: https://cgit.freedesktop.org/gstreamer/gst-examples/tree/network/http-launch/
[puis]: https://cgit.freedesktop.org/gstreamer/gst-examples/tree/playback/player

### gst-build

[gst-build][gst-build-git] is a new meta module to build GStreamer using the
new Meson build system. This module is not required to build GStreamer with
Meson, it is merely for convenience and aims to provide a development setup
similar to the existing `gst-uninstalled` setup.

gst-build makes use of Meson's [subproject feature][meson-subprojects] and sets
up the various GStreamer modules as subprojects, so they can all be updated and
built in parallel.

This module is still very new and highly experimental. It should work at least
on Linux and Windows (OS/X needs some build fixes). Let us know of any issues
you encounter by popping into the `#gstreamer` IRC channel or by
[filing a bug][gst-build-bug].

This module will probably not be released or packaged (does not really make sense).

[gst-build-git]: https://cgit.freedesktop.org/gstreamer/gst-build/tree/
[gst-build-bug]: https://bugzilla.gnome.org/enter_bug.cgi?product=GStreamer&component=gst-build
[meson-subprojects]: https://github.com/mesonbuild/meson/wiki/Subprojects

## Contributors

Aaron Boxer, Aleix Conchillo Flaqué, Alessandro Decina, Alexandru Băluț, Alex
Ashley, Alex-P. Natsios, Alistair Buxton, Allen Zhang, Andreas Naumann, Andrew
Eikum, Andy Devar, Anthony G. Basile, Arjen Veenhuizen, Arnaud Vrac, Artem
Martynovich, Arun Raghavan, Aurélien Zanelli, Barun Kumar Singh, Bernhard
Miller, Brad Lackey, Branko Subasic, Carlos Garcia Campos, Carlos Rafael
Giani, Christoffer Stengren, Daiki Ueno, Damian Ziobro, Danilo Cesar Lemes de
Paula, David Buchmann, Dimitrios Katsaros, Duncan Palmer, Edward Hervey,
Emmanuel Poitier, Enrico Jorns, Enrique Ocaña González, Fabrice Bellet,
Florian Zwoch, Florin Apostol, Francisco Velazquez, Frédéric Bertolus, Fredrik
Fornwall, Gaurav Gupta, George Kiagiadakis, Georg Lippitsch, Göran Jönsson,
Graham Leggett, Gregoire Gentil, Guillaume Desmottes, Gwang Yoon Hwang, Haakon
Sporsheim, Haihua Hu, Havard Graff, Heinrich Fink, Hoonhee Lee, Hyunjun Ko,
Iain Lane, Ian, Ian Jamison, Jagyum Koo, Jake Foytik, Jakub Adam, Jan
Alexander Steffens (heftig), Jan Schmidt, Javier Martinez Canillas, Jerome
Laheurte, Jesper Larsen, Jie Jiang, Jihae Yi, Jimmy Ohn, Jinwoo Ahn, Joakim
Johansson, Joan Pau Beltran, Jonas Holmberg, Jonathan Matthew, Jonathan Roy,
Josep Torra, Julien Isorce, Jun Ji, Jürgen Slowack, Justin Kim, Kazunori
Kobayashi, Kieran Bingham, Kipp Cannon, Koop Mast, Kouhei Sutou, Kseniia, Kyle
Schwarz, Kyungyong Kim, Linus Svensson, Luis de Bethencourt, Marcin Kolny,
Marcin Lewandowski, Marianna Smidth Buschle, Mario Sanchez Prada, Mark
Combellack, Mark Nauwelaerts, Martin Kelly, Matej Knopp, Mathieu Duponchelle,
Mats Lindestam, Matthew Gruenke, Matthew Waters, Michael Olbrich, Michal Lazo,
Miguel París Díaz, Mikhail Fludkov, Minjae Kim, Mohan R, Munez, Nicola Murino,
Nicolas Dufresne, Nicolas Huet, Nikita Bobkov, Nirbheek Chauhan, Olivier
Crête, Paolo Pettinato, Patricia Muscalu, Paulo Neves, Peng Liu, Peter
Seiderer, Philippe Normand, Philippe Renon, Philipp Zabel, Pierre Lamot, Piotr
Drąg, Prashant Gotarne, Raffaele Rossi, Ray Strode, Reynaldo H. Verdejo
Pinochet, Santiago Carot-Nemesio, Scott D Phillips, Sebastian Dröge, Sebastian
Rasmussen, Sergei Saveliev, Sergey Borovkov, Sergey Mamonov, Sergio Torres
Soldado, Seungha Yang, sezero, Song Bing, Sreerenj Balachandran, Stefan Sauer,
Stephen, Steven Hoving, Stian Selnes, Thiago Santos, Thibault Saunier, Thijs
Vermeir, Thomas Bluemel, Thomas Jones, Thomas Klausner, Thomas Scheuermann,
Tim-Philipp Müller, Ting-Wei Lan, Tom Schoonjans, Ursula Maplehurst, Vanessa
Chipirras Navalon, Víctor Manuel Jáquez Leal, Vincent Penquerc'h, Vineeth TM,
Vivia Nikolaidou, Vootele Vesterblom, Wang Xin-yu (王昕宇), William Manley,
Wim Taymans, Wonchul Lee, Xabier Rodriguez Calvar, Xavier Claessens, xlazom00,
Yann Jouanin, Zaheer Abbas Merali

... and many others who have contributed bug reports, translations, sent
suggestions or helped testing.

## Bugs fixed in 1.10

More than [750 bugs][bugs-fixed-in-1.10] have been fixed during
the development of 1.10.

This list does not include issues that have been cherry-picked into the
stable 1.8 branch and fixed there as well, all fixes that ended up in the
1.8 branch are also included in 1.10.

This list also does not include issues that have been fixed without a bug
report in bugzilla, so the actual number of fixes is much higher.

[bugs-fixed-in-1.10]: https://bugzilla.gnome.org/buglist.cgi?bug_status=RESOLVED&bug_status=VERIFIED&classification=Platform&limit=0&list_id=164074&order=bug_id&product=GStreamer&query_format=advanced&resolution=FIXED&target_milestone=1.8.1&target_milestone=1.8.2&target_milestone=1.8.3&target_milestone=1.8.4&target_milestone=1.9.1&target_milestone=1.9.2&target_milestone=1.9.90&target_milestone=1.10.0

## Stable 1.10 branch

After the 1.10.0 release there will be several 1.10.x bug-fix releases which
will contain bug fixes which have been deemed suitable for a stable branch,
but no new features or intrusive changes will be added to a bug-fix release
usually. The 1.10.x bug-fix releases will be made from the git 1.10 branch,
which is a stable branch.

### 1.10.0

1.10.0 was released on TBD 2016.

## Known Issues

- FILL ME

## Schedule for 1.12

Our next major feature release will be 1.12, and 1.11 will be the unstable
development version leading up to the stable 1.12 release. The development
of 1.11/1.12 will happen in the git master branch.

The plan for the 1.12 development cycle is yet to be confirmed, but it is
expected that feature freeze will be around early/mid-January,
followed by several 1.11 pre-releases and the new 1.12 stable release
in March.

1.12 will be backwards-compatible to the stable 1.10, 1.8, 1.6, 1.4, 1.2 and
1.0 release series.

- - -

*These release notes have been prepared by Olivier Crête, Sebastian Dröge,
Nicolas Dufresne, Víctor Manuel Jáquez Leal, Arun Raghavan, Tim-Philipp
Müller, Wim Taymans, Matthew Waters*

*License: [CC BY-SA 4.0](http://creativecommons.org/licenses/by-sa/4.0/)*
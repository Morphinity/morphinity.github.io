---
layout : post
blogTitle : Decoding Video Frame to Bitmap
category : Code
date : 2015-12-03
---
*This post deals with getting a frame from Android MediaCodec and converting it to a bitmap\. Assuming basic knowledge of MediaCodec, Bitmap, MediaExtractor*

Recently I was working on a code to create thumbnails for video files on Android\. For this the logical flow would be to get a frame of the video at a given time, convert it to a bitmap and then scale/rotate to get a resized bitmap that can be used as a thumbnail. The logic of getting a frame at a given time as a bitmap is implemented by Android\'s native [MediaMetadataRetriever](http://developer.android.com/reference/android/media/MediaMetadataRetriever.html) which uses native apis to efficiently seek and fetch the frame\. Then we can use *extractThumbnail* api of [ThumbnailUtils](http://developer.android.com/reference/android/media/ThumbnailUtils.html) to resize the bitmap\.

Now the catch is that the MediaMetadataRetriever is bound by Android\'s native [MediaExtractor](http://developer.android.com/reference/android/media/MediaExtractor.html) capabilities which can only extract mp4 files\. You may have observed that most video players on android cannot play mov files which is the default video file type on iOS devices, hence making a cross platform video playing application slightly difficult on Android as iOS supports mp4\.

So back to the basics\. The first hurdle, of course, would be to get the raw sample data from mov file\. For this I created a custom MediaExtractor based on the latest version of Google\'s [ExoPlayer](http://developer.android.com/guide/topics/media/exoplayer.html) that provides apis to parse moov atoms in mov file headers\. Efficiently loading samples and maintaining sync between playback and sample loading thread would be best discussed in another post\.

Once we have a custom MediaExtractor that can seek to a given time for both mov and mp4 files, we need a decoder to decode the samples obtained from MediaExtractor\. Using [MediaCodec](http://developer.android.com/reference/android/media/MediaCodec.html), we can obtain the decoded data either in the form of [ByteBuffer](http://developer.android.com/reference/android/media/MediaCodec.html#getOutputBuffer(int)) or we can [format the MediaCodec using a surface](http://developer.android.com/reference/android/media/MediaCodec.html#configure(android.media.MediaFormat, android.view.Surface, android.media.MediaCrypto, int)) and obtain data on the surface\.

We have two ways to go about now

+ Using ByteBuffer
	+ Obtain the data in a ByteBuffer using mediaCodec.getOutputBuffer(outputBufferIndex)
	+ Obtain the color format from mediaCodec outputFormat and use a color conversion function to convert it to ARGB pixels\. This post discusses the conversion of YUV color format to ARGB\.
	+ Obtain width, height and rotation from outputFormat to create a bitmap\. Use setPixels api to set the above obtained pixels on the bitmap\.
	+ Perform scaling by using ThumbnailUtils and extract the bitmap of the required size\.
+ Using Surface
	+ Using a surface would require an OpenGL context in order to obtain the pixel data\. This will be discussed in a future post\.

*Pros and Cons*

+ Using a surface is faster and efficient as it uses GPU for the computation\.
+ By configuring the MediaCodec with a surface, we are ensured that the pixel data written on the surface is in ARGB format\. This is not the case with ByteBuffer where different version of YUV format (depending on the vendor) maybe used to create the ByteBuffer\. Hence we\'ll have to identify the color format and apply color conversion before converting to bitmap\.
+ Using Surface we need to add an OpenGL Shader that can be used to perform cropping, scaling, rotation or any possible GL computation like applying an effect on the frame\.

Using a surface for the decoder and then using a GL Shader to obtain pixel data is thus a better idea than using DecodedBuffer\.

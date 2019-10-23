---
layout: post
title: Seeking Three Moving Windows in Ogg-Theora v1
date: 2019-10-22 00:00:00 +0300
description: You’ll find this post in your `_posts` directory. Go ahead and edit it and re-build the site to see your changes. # Add post description (optional)
img: Theora_logo_2007.svg # Add image post (optional)
tags: [ogg, theora, seek, video, codec] # add tag
---


# Introduction
Welcome codec enthusiast!

My first post will discuss the complexities of jumping into an ogg video stream. Designed by Christopher Montogomery, xiph promoted ogg as generic format for all types of media streams. Unfortunately, ogg generic and streaming design has led to a few difficult to reconcile design decisions. This guide is not sample accurate.

# Implementation

To seek an Ogg-Theora video, we separate the task into three steps: binary ogg page search, keyframe backtracking, and decoder submission. In our example, we assume the ogg file is divided into 4k blocks. 

## Ogg Page Search
Ogg format do not define a seek table; however, ogg specification mandates ogg pages must be sorted in descending order by time which enables binary search. In addition, Ogg page only records the last completed packet time and each code codec defines granulepos uniquely. In theora and vorbis, these functions are `th_granule_time` and `vorbis_granule_time` respectively. Unlike the last two steps, the first step can be implemented without examining the ogg packet contained in the page.

You will need this helper function to serialize raw bytes.
```c++
int buffer_data() {
    char *buffer = ogg_sync_buffer(&oy, BUFFERSIZE);
    int bytes = file->get_buffer((uint8_t *)buffer, BUFFERSIZE);
    ogg_sync_wrote(&oy, bytes);
    return (bytes);
}
```

![Ogg Example 1]({{site.baseurl}}/assets/img/2019-10-22-ogg-theora1.jpg)


In the example above, we can examine a few common cases with syncing ogg pages

### No Page Sync
Assume we sync file block 5, `ogg_sync_pageout` returns -1 because the buffer did not capture the beginning of the page.

### Need More Bytes
When we sync file block 4, `ogg_sync_pageout` returns 0 when called again. In order to sync ogg page 3, we will need to sync block 5 and block 6 to complete the page.

### No Granulepos
When we sync file block 3 and 4, `ogg_sync_page` returns 1 which indicates a whole page, but the page holds an incomplete packet. Therefore, the `ogg_page_granuelops` will return -1. `ogg_page_packets` will return 0.

### Different Serial Number
Although Vorbis codec may has a granuelops in every page, vorbis uses a different granulepos function to calculate the absolute time. Block 7 has an ogg page with an absolute time of .542 seconds. `ogg_page_packets` will return 3.

### Successful Granulepos
When we sync blocks 4,5, and 6 `ogg_sync_page` will return 1 after a few buffers and 0 calls. We can calculate the absolute time of the large spanning ogg packet without combining pages. `ogg_page_packets` will return 1.

### Binary Ogg Page Search
In a binary search, the algorithm consists of a pivot and a comparison. In our example, we will pivot around the first positive granuelops page in each 4k buffer block and compare the time in seconds. As an added complication, some blocks do not have a valid page nor granuelops. As a consequence, we are binary searching a sorted array with invalid elements.

![Ogg Example 1]({{site.baseurl}}/assets/img/2019-10-22-array-with-gaps.jpg)

```
function BinaryGapSearch( Array arr, value) {
    left = 0
    right = len(arr) - 1
    while ( left <= right ) {
        mid = low + ((right - left ) / 2)
        mid_cursor = mid
        bool valid_mid = true
        while( isInvalid(arr[mid_cursor]) ) {
            if( mid_cursor > right ) {
                right = mid
                valid_mid = false
                break
            }
            mid_cursor++
        }
        if( valid_mid )
            continue
        if( arr[mid_cursor] == value )
            return mid_cursor
        else if ( arr[mid_cursor] > value ) {
            right = mid - 1
        }
        else {
            low = mid_cursor + 1
        }
    }
}
```

```c++
struct _page_info {
        size_t block_number;
        double_t time;
        ogg_int64_t granulepos;
    };
    float p_time = 50.0; //desired seek time
 
    size_t buffer_size = (size_t)BUFFERSIZE; //Cast BUFFERSIZE
    size_t end_file = file->get_len();
    size_t start_file = 0;
    size_t number_of_blocks = (end_file - start_file) / buffer_size;
 
    ogg_packet op;
    size_t left = 0;
    size_t right = number_of_blocks;
 
    struct _page_info left_page = { .time = 0, .block_number = 0, .granulepos = 0 };
    struct _page_info mid_page = { .time = 0, .block_number = 0, .granulepos = 0 };
    struct _page_info right_page = { .time = DBL_MAX, .block_number = right, .granulepos = 0x7FFFFFFFFFFFFFFF };
    HashMap<ogg_int64_t, _page_info> page_info_table;
    HashMap<int, double> block_time;
 
    //Binary Search by finding the proper begin page
    while (left <= right) {
        //Seek to block
        size_t mid_block = left + (right - left) / 2;
        int block = mid_block;
 
        if (block_time.has(block)) {
            //Check whether this block has been visited
            break;
        }
 
        //clear the sync state
        ogg_sync_reset(&oy);
        file->seek(block * buffer_size);
        buffer_data();
 
        bool next_midpoint = true;
        while (true) {
            //keep syncing until a page is found. Buffer is only 4k while ogg pages can be up to 65k in size
            int ogg_page_sync_state = ogg_sync_pageout(&oy, &og);
            if (ogg_page_sync_state == -1) {
                //Give up when the file advances past the right boundary
                if (buffer_data() == 0) {
                    right = mid_block;
                    break;
                } else {
                    //increment block size we buffered the next block
                    block++;
                }
            } else {
                if (ogg_page_sync_state == 0) {
                    //Check if I reached the end of the file
                    if (buffer_data() == 0) {
                        right = mid_block;
                        break;
                    } else {
                        block++;
                    }
                } else {
                    //Only pages with an end packet have granulepos. Check the stream
                    if (ogg_page_packets(&og) > 0 && ogg_page_serialno(&og) == to.serialno) {
                        next_midpoint = false;
                        break;
                    }
                }
            }
        }
        if (next_midpoint)
            continue;
 
        ogg_int64_t granulepos = ogg_page_granulepos(&og);
        ogg_int64_t page_number = ogg_page_pageno(&og);
        struct _page_info pg_info = { .time = th_granule_time(td, granulepos), .block_number = mid_block, .granulepos = granulepos };
        page_info_table.set(page_number, pg_info);
        block_time.set(mid_block, pg_info.time);
        mid_page = pg_info;
 
        //I can finally implement the binary search comparisons
        if (abs(p_time - pg_info.time) < .001) {
            //The video managed to be equal
            right_page = pg_info;
            break;
        }
        if (pg_info.time > p_time) {
            if (pg_info.granulepos < right_page.granulepos)
                right_page = pg_info;
            right = mid_block;
        } else {
            if (pg_info.granulepos > left_page.granulepos)
                left_page = pg_info;
            left = mid_block;
        }
    }
```

## KeyFrame Backtracking
Since libogg do not provide facilities to iterate ogg pages backwards, our buffer must seek a file block and scan forward to reveal multiple packets or pages. In the code below, many ogg theora encoders encapsulate the keyframe in a separate ogg page.

![Ogg Example 1]({{site.baseurl}}/assets/img/2019-10-22-ogg-theora2.jpg)


In addition to sync problems in the described first section, we present sync problems with `ogg_stream_pageout`.

### Page Contains an Incomplete Packet
When we buffer block 8, `ogg_stream_packetout` reveals the page has the beginning of the packet. We need to sync the next page to obtain the complete packet.

### No Keyframe Found
In file block 5 or 6, the pages have an offset greater than 1. Theora define granulepos as keyframe granule|offset frame. Although libtheora only provides a keyframe check for the packet, you can discern the keyframe with ogg page granule only.

In the code below, the backtracking algorithm finds all beginning ogg sections started in current buffer and follows data to return a valid complete ogg packet. The code tests ogg packets. This code assumes the keyframe is contained within one ogg page.
```c++
// Backtrack to find the keyframe
    // Keyframes seem to reside on their own page
    while (current_block >= 0) {
        ogg_stream_reset(&to);
        ogg_stream_reset(&vo);
        ogg_sync_reset(&oy);
        file->seek(current_block * buffer_size);
        buffer_data();
        bool seeked_file = false;
        bool keyframe_found = false;
        while (!seeked_file) {
            int ogg_page_sync_state = ogg_sync_pageout(&oy, &og);
            if (ogg_page_sync_state == -1) {
                ogg_page_sync_state = ogg_sync_pageout(&oy, &og);
            }
            while (ogg_page_sync_state == 0) {
                buffer_data();
                seeked_file = true;
                ogg_page_sync_state = ogg_sync_pageout(&oy, &og);
            }
            if (ogg_page_sync_state == 1) {
                //Only queue pages with a single packet
                if (ogg_page_packets(&og) == 1) {
                    queue_page(&og);
                    ogg_stream_packetpeek(&to, &op); //Just attempt it
                    while (ogg_stream_packetpeek(&to, &op) == 0) {
                        if (ogg_sync_pageout(&oy, &og) > 0) {
                            queue_page(&og);
                        } else {
                            buffer_data();
                        }
                    }
                    if (th_packet_iskeyframe(&op)) {
                        videobuf_time = th_granule_time(td, op.granulepos);
                        if (videobuf_time < p_time) {
                            keyframe_found = true;
                            break;
                        }
                    } else
                        ogg_stream_packetout(&to, &op);
                }
            }
        }
        if(keyframe_found) {
            break;
        }
        current_block--;
    }
```


## Decoder Submission

Since we found the keyframe and time in the last section, the next step is to submit frames into `th_decode_packetin` and dump it with `th_decode_ycbcr_out`. As the decoder scans forward, the granulepos can be incremented to allow frame comparison to the desired seek time. When the stream advance pass the desired time by one frame, just break out. Resume should be able to play the leftover frames within the theora stream.
On the other hand, libvorbis and libogg provide limited facilities to help anyone write sample accurate seeking. Vorbis pcm are encapsulated within the vorbis block. As a consequence, all decoders deal with 4 sync windows. Unlike the keyframe, we are not necessarily guaranteed to have the previous page timestamp. In order to sync by packet, my best approximation is to use `vorbis_packet_blocksize` and `ogg_page_packets` to estimate the correct pcm timestamp. Nevertheless, this method would not work on files with variable block sizes.

```c++
//Process keyframe
    ogg_int64_t video_granulepos;
    th_decode_packetin(td, &op, &video_granulepos);
    th_decode_ctl(td, TH_DECCTL_SET_GRANPOS, &video_granulepos, sizeof(video_granulepos));
    th_ycbcr_buffer yuv;
    th_decode_ycbcr_out(td, yuv); //dump frame
    ogg_stream_packetout(&to, &op);
 
    //decode video until the decoder catches up to the seek time
    while (videobuf_time <= p_time) {
        int ogg_sync_state = ogg_sync_pageout(&oy, &og);
        while (ogg_sync_state < 1) {
            buffer_data();
            ogg_sync_state = ogg_sync_pageout(&oy, &og);
        }
        if(ogg_page_serialno(&og) == to.serialno) {
            queue_page(&og);
            while (ogg_stream_packetout(&to, &op) > 0) {
                ogg_int64_t tmp_granulepos;
                th_decode_packetin(td, &op, &tmp_granulepos);
                if (op.granulepos > 0) {
                    th_decode_ctl(td, TH_DECCTL_SET_GRANPOS, &op.granulepos, sizeof(op.granulepos));
                    videobuf_time = th_granule_time(td, tmp_granulepos);
                    video_granulepos = tmp_granulepos;
                } else {
                    videobuf_time = th_granule_time(td, video_granulepos++);
                }
                th_ycbcr_buffer yuv;
                th_decode_ycbcr_out(td, yuv); //dump frames
            }
            //Needed to calculate the time without extra memory allocation
        } else {
            //Drop pages behind the seek
            double end_music_time = vorbis_granule_time(&vd, ogg_page_granulepos(&og));
            if (end_music_time > p_time) {
                queue_page(&og);
                //Queue the page which music time is greater than the seek time.
                audio_granulepos = ogg_page_granulepos(&og);
                total_packets = ogg_page_packets(&og);
            }
        }
    }
    //Update the audioframe time
    audio_frames_wrote = videobuf_time * vi.rate;
    while(ogg_stream_packetout(&vo, &op) > 0) {
        ogg_int64_t offset = vorbis_packet_blocksize(&vi, &op) * total_packets--;
        double current_audio_time = vorbis_granule_time(&vd, audio_granulepos - offset);
        double diff = current_audio_time - videobuf_time;
        if( diff > 0 ) {
            //Audio catch up
            audio_frames_wrote = mixed + videobuf_time * vi.rate;
            break;
        }
    }
```

## Tricks

### Calculating the time(s) in each theora ogg_packet within each ogg_page.

Theora defines the granulepos as keyframe `granule|offset`. For an example, `56|3` is a frame is 3 delta frames after the keyframe. Each keyframe tends to be separated into its own page and each ogg packet contains only one frame. Since `ogg_page_packets` outputs completed packets, the trick is to count backwards to offset the correct frame. The code below is only useful in a continuing stream because the code fails to accommodate incomplete beginning ogg packet.

```c
ogg_int64_t total_end_packets = ogg_page_packets(&og);
ogg_int64_t last_granule = ogg_page_granulepos(&og);
while( ogg_stream_pageout(&to, &op) > 0 ) {
    double time_secs = th_granule_time(&vi, last_granule - total_end_packets--);
}
```
### Convert Vorbis time(s) to granulepos

The specification states 
> The granule position of pages containing Vorbis audio is in units of PCM audio samples (per channel; a stereo stream’s granule position does not increment at twice the speed of a mono stream).

```c
granulepos = floating_time * vi.rate
```

### Convert Dummy Theora granulepos for comparison
```c
((ogg_int64_t)(Videotime * vi.rate)) << vi.keyframe_granule_shift
```

## Author's Notes
In hindsight, dividing the file into fixed 4k blocks has led to a few elaborate design decisions such as gapped binary search. In my next design, I would use the frame offset and previous scan out ogg pages to estimate the amount of bytes needed to backtrack.

## Links
* [Godot Seek Theora Implementation](https://github.com/hungrymonkey/godot/tree/seek_theora)
* [Christopher Montogomery replies to criticism](https://people.xiph.org/~xiphmont/lj-pseudocut/o-response-1.html)
* [Vorbis I Specification](https://xiph.org/vorbis/doc/Vorbis_I_spec.html)
* [Ogg Overview](https://xiph.org/ogg/doc/ogg-multiplex.html)
* [Basic Ogg Information](https://xiph.org/oggz/doc/group__basics.html)
* [Ogg and the multimedia container format struggle](https://lwn.net/Articles/382478/)
* [LibVorbis API Documentation](https://xiph.org/vorbis/doc/libvorbis/)
* [LibTheora API Documentation](https://www.theora.org/doc/libtheora-1.0/)
* [LibTheora Specification](https://theora.org/doc/libtheora-1.0alpha5.pdf)
* [Library Congress Preservation Information](https://www.loc.gov/preservation/digital/formats/fdd/fdd000026.shtml)



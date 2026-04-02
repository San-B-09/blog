---
title: "YouTube’s Video Ids May Eventually Run Out?"
date: 2022-10-11
slug: "youtube-video-ids"
description: "A breakdown of how YouTube’s Base64 video IDs work and why they won’t run out anytime soon"
summary: "A breakdown of how YouTube’s Base64 video IDs work and why they won’t run out anytime soon"
categories: ["fascinating"]
draft: false
---

![Youtube URL](/images/youtube-video-ids-url.png)

If you’ve ever noticed, YouTube has a unique video id for each video. It’s right there in the video URL right after `v=`. A string of 11 characters that identifies the video you want to play. YouTube has millions of video uploads everyday. As per the latest stats, 500 hours of video content are uploaded on YouTube every single minute. That’s nearly 3.7 million new YouTube videos uploaded every single day. So are they gonna run out of these video id's soon ?

To get a bit deeper, let’s take a dig at the counting systems. We generally count in Base10, that is 0 to 9. Computers count in Base2, that is 0 and 1. Base2 numbers tend to get longer quite easily and are merely readable to a normal person. So we introduced Base16, that is 0 to 9 and then A to F. Base16 is also hardly human readable but has a lower length compared to Base2 and Base10.

![Counting Systems](/images/youtube-video-ids-counting-system.png)

How about Base64 ? That’ll be a ridiculous counting system, right ?

Additionally, 64 is not just any random number. It’s 2⁶ and we can get to 64 easily with 0 to 9, A to Z, a to z and any two special characters. Commonly used special characters in Base64 are + and / but as these don’t work with URL’s, so YouTube uses `—` hyphen and `_` underscore.

The YouTube video id is none other than a random number in Base64. The reason to go with Base64 is that a large number can be cramped into a small space and still be a little human readable.

> *FYI, author and programmer Sam Hughes pushed this to the limit and invented Base65536, including all available characters from almost all languages.*

---

So now, we might question why YouTube didn’t start video ids at 1 and then work up. The reason being, incremental counters are bad and have security flaw in many ways. For YouTube, using incremental counters might result in the following issues:
- **Synchronization Problem**: Synchronizing the assignment of video ids across all servers would have been difficult in the case of incremental counter assignment. There’s a lot of video id tracking involved. For a random id, it can just generate a random number, do a look-up for the number being taken, and if it’s not taken, it can convert the same to Base64 and assign it to a video.
- **Unlisted Video Sharing Problem**: We’re well known to YouTube’s unlisted video sharing feature, where the videos are not available publicly, but we can share them using a link. If there were incremental counters, it would become very easy to just go to the next or previous video and hence reach an unlisted video too.

In general, incremental counters are considered a bad design for any record id because of the ease with which anyone can go through all records or get total count.

---

Now you might wonder, just how big are the numbers that YouTube uses. Let’s work it out!

One character of Base64 lets you have 64 ids. Two characters, that is `64 x 64` or 4096 ids. Three characters, that is `64 x 64 x 64` or 262,144, more than a quarter of a million. Four characters, `64 x 64 x 64 x 64` is 16,777,216 or 16.7M. Now if you use Base64, you can assign an id to everyone who lives in India’s silicon valley Bangalore and you’ll only need four characters. This get’s big fast as we go up. With 7 Base64 characters, we’re already at more than 4 quadrillion.

At YouTube’s 11 character video id, we’re at 73 quintillion 786 quadrillion 976 trillion 294 billion 838 million 206 thousand and 464 videos.

![Youtube Video IDs Count ](/images/youtube-video-ids-count.png)

That’s enough for every single human on planet Earth to upload a video every minute for around 18,000 years.

So, can they ever run out of video ids or URLs ?
Technically, yes, but practically no. And even if they did, they could just add one more character.

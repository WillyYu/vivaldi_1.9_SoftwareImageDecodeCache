# Memory enhancement for SoftwareImageDecodeCache

### Several steps which can make Vivaldi's renderer process memory grow to 400MB ~ 500MB
1. Restore a session which included more than 10 tabs, but only keep one tab online
2. Load web page (I browse www.mobile01.com which is a rich pictures website in Taiwan)
3. Wait for page loaded, move the cursor on the tab strip and show up all tab’s thumbnail
4. Repeat 2-3

The Memory usage of renderer process shown as below
<img src="https://github.com/WillyYu/vivaldi_1.9_SoftwareImageDecodeCache/blob/master/images/NormalMemoryUsage.png?raw=true"/>

We can see that the memory of Vivaldi’s host render process consumes from 343 MB to 461 MB.
It is strange, because only one tab is active during the testing period.
Every time popup the tab’s thumbnail should be the same.

### Root cause
By Investigation, The memory is grown by SoftwareDecodeCache.
Here is the explanation of reason.
Basically, the lifecycle of Image resource is:

1. The Creation of Image resource and decoded cache
    - Blink tries to download the image resource from network when inserting a image element into DOM tree. And then put the downloaded content into MemoryCache.
    - If Compositor triggers Blink to paint, DisplayList is generated and references the images in the MemoryCache.
    - The Compositor gets the DisplayList and trying raster the content to memory buffer, referenced image resources are decoded and insert into SoftwareDecodeImageCache.
    <img src="https://github.com/WillyYu/vivaldi_1.9_SoftwareImageDecodeCache/blob/master/images/ImageResourceLifecycle_01.png?raw=true"/>

2. Garbage Collection:
    - The Blink sometimes trigger Garbage collection to clean up the memory, ex: page loading.
    - The cached image in Memory cache will be cleaned if there is no reference on it.
    - Like below picture the image resource was removed.
    <img src="https://github.com/WillyYu/vivaldi_1.9_SoftwareImageDecodeCache/blob/master/images/ImageResourceLifecycle_02.png?raw=true"/>

3. Another image was updated
    - A image element is updated (or inserted) that may same as image resource 1 totally. (Just like the case of popup tab's thumbnail.)
    - But the image resource was removed by Garbage Collection previously, so the image resource must download again and inserts into Memory Cache.
    - Same as step 1, the DisplayList reference the image is sent to Compositor, and decode again for raster.
    - At this point in time, there are two decoded data in the cache but with same content. Besides, only one is in use.
    <img src="https://github.com/WillyYu/vivaldi_1.9_SoftwareImageDecodeCache/blob/master/images/ImageResourceLifecycle_03.png?raw=true"/>


### When to purge the cache ?
    - From current Chromium’s design, the decoded cache will grow until reach the limit:
        - The max cache items (1000)
        - The DiscardSharableMemory’s memory limit

### Experiment
I have tried to do a experiment with this <a href="https://github.com/WillyYu/vivaldi_1.9_SoftwareImageDecodeCache/commit/a20af56564123289589b87382303897603b15ede">commit</a>, which can trace the lifecycle of image resource and compare with decoded data in the cache. So that we can trace how many resources are in use.

The result show as below.

<img src="https://raw.githubusercontent.com/WillyYu/vivaldi_1.9_SoftwareImageDecodeCache/252c8b5f13e3221327d7846d38d9083bd840247f/images/QuantityDecodeCacheInUse.png"/>

We can find that the count of image resources in use is under 200, even reach the cache limit.

### Enhancement
Currently, a better solution need change several modules (include skia, cc) which will be a big change.
So I have tried to lower the max cache items to 200 (this <a href="https://github.com/WillyYu/vivaldi_1.9_SoftwareImageDecodeCache/commit/7fabd202ccfb60b492696bccbf90987f8d741ba4">commit</a>).
The memory consumption shown like below
<img src="https://github.com/WillyYu/vivaldi_1.9_SoftwareImageDecodeCache/blob/master/images/MemoryUsageComparison.png?raw=true"/>

It shows that lower the cache limit would reduce the memory usage.


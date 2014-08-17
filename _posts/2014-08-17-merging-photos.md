---
layout: post
title: Merging camera and phone photos
---

I just got back from a vacation where I took photos with both my
point-and-shoot camera (Canon A540) and my Android phone (Moto X).  The camera
uses the filename format `IMG_????.JPG` while the phone uses the format
`IMG_YYYYMMDD_HHMMSS.jpg`.  This means that the two sets of photos cannot be
easily viewed as one coherent set. To fix this problem, I used [exiftool][] to
bulk rename the camera files to `IMG_YYYYMMDD_HHMMSS_????.jpg`.

```bash
$ sudo aptitude install exiftools
```

First, I had to fix the timestamp on the camera photos since I forgot to
change the timezone on the camera.  EXIF timestamps are recorded in local time
with no timezone information, so I had to correct the timestamps by adding, in
this case, 13 hours.

```bash
$ exiftool '-DateTimeOriginal+=0:0:0 13:0:0' ???_????.???
```

Then, I renamed the files in my desired format, inserting the timestamps
between the `IMG\_` prefix and the `????.JPG` suffix.  I also converted the
suffix to lowercase to be consistent with the Android photos.

```bash
$ exiftool '-filename<${FileName;s/_.*//}_${DateTimeOriginal}_${FileName;s/.*_//;tr/A-Z/a-z/}' -d '%Y%m%d_%H%M%S' ???_????.???
```

Finally, I had to manually adjust the .avi filenames since exiftool cannot
write EXIF data for AVI files, and thus cannot adjust the timestamps in the
first command.  I did this with a simple copy and paste from the corresponding
.thm files.

[exiftool]: http://www.sno.phy.queensu.ca/~phil/exiftool/

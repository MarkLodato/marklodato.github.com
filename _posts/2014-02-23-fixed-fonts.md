---
layout: post
title: Enabling the "fixed" font family
---

I really love the "fixed" bitmap font family on Linux.  It is readable at
a much smaller size than any TrueType font I have found.  Compare:

![Fixed SemiCondensed 10]({{ site.url }}/images/2013-02-23-fixed.png)
![DejaVu Sans Mono 8]({{ site.url }}/images/2013-02-23-dejavu-sans-mono-8.png)
![DejaVu Sans Mono 9]({{ site.url }}/images/2013-02-23-dejavu-sans-mono-9.png)

To enable this font in Xterm, add the following to your ~/.Xresources:

```
XTerm*font:             -*-fixed-medium-r-*-*-13-*-*-*-*-60-iso10646-1
XTerm*boldFont:         -*-fixed-medium-r-*-*-13-*-*-*-*-60-iso10646-1
```

On Ubuntu, all bitmapped fonts are disabled by default in fontconfig, so it is
not possible to use this font in, say, gnome-terminal or xfce4-terminal.  To
enable this font, you must do the following, courtesy of [cpitchford on
Ubuntu Forums](http://ubuntuforums.org/showthread.php?t=1270870):

```bash
$ sudo su
$ cat > /etc/fonts/conf.avail/69-fixed-bitmaps.conf <<EOF
<?xml version="1.0"?>
<!DOCTYPE fontconfig SYSTEM "fonts.dtd">
<fontconfig>
  <!-- Enabled Fixed bitmap fonts -->
  <selectfont>
    <acceptfont>
      <pattern>
        <patelt name="family">
          <string>Fixed</string>
        </patelt>
      </pattern>
    </acceptfont>
  </selectfont>
</fontconfig>
EOF
$ ln -s ../conf.avail/69-fixed-bitmaps.conf /etc/fonts/conf.d/
$ dpkg-reconfigure fontconfig-config
$ dpkg-reconfigure fontconfig
```

And then restart X.  Presto!

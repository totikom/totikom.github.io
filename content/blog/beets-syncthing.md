+++
title = "Managing multi-device music library via beets and Syncthing"

date = 2025-03-06
template = "page_with_toc.html"
[taxonomies]
tags = ["system-administration", "music", "beets", "syncthing"]
[extra]
show_only_description = true
+++
I'm one of those people, who don't use streaming services and store all music on my devices.

At first, I was just downloading it and occasionally copied it from one device to another.
However, this approach does not scale well.
Music folders on my devices had quickly became a mess: irregular names, album folders mixed with artist folders, different timestamps on different devices.

At some point I've found myself downloading the same album three times: first two on different devices, because I've forgot to copy from one to another, and the third one on the phone with this album already available!
Just because of unintuitive folder name I was unable to locate it.

# beets
To solve it on PC I've started using [beets](https://beets.io/).
 `beets` is a nice music management software, which allows you (among lots of otsystema) to move your music to neatly organized folder and correct songs metadata with [Musicbrainz](https://musicbrainz.org/) database.

 It took me several days to import my whole PC collection to `beets` DB, but, luckly, most of the process was automatic[^1].
 All I had to do was just:
 [^1]: If `beets` finds a matching album, it imports music automatically, but if match is not close enough, it will ask for user confirmation.

 ```bash
beet import -q [path to music to be imported]
```
Where `-q` flag means "skip mismatches and continue without user interaction".

 After that I've runned `beets` once again, but without `-q`  and manually confirmed all uncertainties.

 OK, the biggest part of my music collection got systematized.
 However, it created another problem: during the time collections on the PC and on the phone inevitabily became out of sync.
 The divergence wasn't supposed to be significant, but I couldn't just delete all music from the phone and reupload systematized collection from PC.

 So, I had to manually replace parts of my collection on the phone adding out of sync albums back to the PC.

 I guess, it took me a week or longer and it was VERY tiresome.

 Now, when I get new music, I immediately pass it through `beets`, so my collection remains systematized.

 Seems like all the problems are finally solved?
 Ehhh, not exactly.

 Over time my listening habits have changed and now I'm listening music almost exclusively from my phone.

 More over, I'm _getting_ music on my phone, so the above "workflow" transforms to:
 1. Get new music on the phone
 2. Eventually copy it to the PC
 3. Add the new music to `beets` library
 4. Copy library entries back to the phone

Not very convenient, right?
That has led to me procrastinating music sync for weeks, which complicated everything, because all music players store tracks in playlists by referencing paths, so moving tracks around cause playlists to break.

And I had to plug my phone via USB and wait for all the operations to finish!

# Syncthing to the rescue!
I knew about [syncthing](https://syncthing.net/) for some time, but hadn't found a use for it until resently.

`Syncthing` is not a cloud, but rather _device-to-device_ synchronization program, which may be not the best variant, if you don't have a continuously operating machine, which will serve as "cloud storage", but for keeping files in sync between devices it is a perfect fit.

After setting up `syncthing` on both the PC and the phone, I got (almost) perfect setup for music collection storage and listening:

- The collection is synchronized between devices without manual actions
- No need to connect anything
- I can get music on any device
- I can add music to `beets` library at any time I'm near PC: the changes will be synchronized automatically

# Final setup
1. `syncthing` is running on both PC and phone
2. `beets` is used on the PC to keep collection organized
3. Any device could be used for downloading music

This simple workflow had allowed me to greatly simplify my music collection management and usage.
Looking back, this workflow looks like pretty obvious, but I hope that it still can help people, who maintains their music library by their own.

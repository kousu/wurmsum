# Wurmsum

![logo](logo.png)

Checksum a complete directory tree, using inotify.

This is to detect bitrot in large file collections, in order to make it possible to know when your files are dying and need recovery or redownloading.

## Install

You need `inotifywatch`

```
apt install inotify-tools  # Debian etc
```

```
pacman -Sy inotify-tools   # Arch
```

Install the script:

```
install wurmsum wurmsum-watch /usr/bin
```

Intall the `.service`:

```
sudo cp wurmsum@.service /usr/lib/systemd/user/
```

Enable it for each folder you want to protect:

```
systemctl --user enable --now wurmsum@/your/linux-iso/collection
systemctl --user enable --now wurmsum@~/Music
```

It's a user service to make sure the file permissions are sensible. ber You could also set up a system service, but just be careful that the user you run it as has the right permissions and doesn't create files you can't get rid of yourself. If you figure that out please contribute a PR :)

Manual scrub:

```
wurm-scrub ~/Music  # name pending
```

You can scrub only specific subfolders, if, e.g., they are the most important to you, or if you're planning on uploading them somewhere:

```
wurm-scrub ~/Music/Projects/Recordings
```

You can also enable a timer to scrub in the background automatically:

## Recovery

This is only as helpful as the backups you keep! Make sure to budget for backups. I recommend [borg](https://www.borgbackup.org/) or [restic](https://restic.readthedocs.io/en/latest/manual_rest.html), which both include their own checksumming and do snapshotting so you can go back in time. Expect your backups to be 10% to 20% _larger_ than your collection, because you want to be able to go back in time a certain way to find lost files.

Another backup method is file sharing. Make a torrent of your files and share it with your friends. rsync parts of your collection to your friends (and encourage them to run this to protect their copies from bitrot).

Finally, if your file collection is mostly things you've downloaded, maybe it's okay to not have backups.

## vs dm-integrity, zfs, btrfs

The more polished way to protect against bitrot on linux is to use one of these. But they only provide block-level checksumming, which makes it very hard to know what files have rotted, if any. The only way to tell is to try to read every file and observe EIO errors. It's hard to script around.

The most automated set up is probably to layer lvm RAID1 on top of dm-integrity. Then, every corruption will automatically be noticed by lvm and recovered transparently. You can also run `lvchange --syncaction repair` periodically to scrub, automatically recovering bitrot.

With dm-integrity without RAID1, you can do a scrub with `integritysetup check`, but all that tells you is that there are errors, not where the errors are; it might cause you to restore your ENTIRE collection from backup, which could take **weeks** for larger collections.

With dm-integrity, the best you can do is a file-level scrub to find rot:

```
find . -type f -print0 | \
xargs -0 -n1 sh -c 'cat "$0" >/dev/null || echo "Read failed: $0"'
# better: write this in C/go/python and specifically catch EIOs
```

But:

These systems slow down the entire system, with overheads reported of 5% to 10%, because every read involves a hash. You want to burn CPU? You've got burning CPU. dm-integrity over USB can be even slower, because it can hit the USB bandwidth limit.

Also, it's just _complicated_. You need to learn how to use integritysetup and lvm and while I think lvm is great it you gotta admit it's dense with a steep learning curve. If your disks crash are you gopering to even remember how to put them back together to restore integrity?

And what if:

- you don't want to use RAID1 (because it's too expensive to buy extra disks, because you do have enough drive bays, etc)
- you want to know what files have rotted, not just that there is rot
- you want to split up your collection
  - share part of it with friends
  - backup part of it but not the others

If you move files out, you lose their checksums. You need to only ever move them dm-integrity to dm-integrity system to preserve.

### This

The approach here is less aggressive and less complicated. Changes to files are checksummed on save (`integritywatch`), but verification only runs on demand (`wurmsum-scrub`), which means in low-write high-read workloads, like media collections, your system isn't slowed down and can be done on the whole collection or folder-by-folder. It uses standard tools (`sha256sum`) whose format is portable (to macOS, Linux, and BSD). You can share or move a file collection without losing the checksums, because they're visible instead of being hidden in the block layer.

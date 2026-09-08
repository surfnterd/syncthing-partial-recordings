# Syncthing + Android voice recorder: why your watcher sees broken m4a files, and why you must not move them

If you sync voice recordings off an Android phone with Syncthing and process them with a watcher on the other end (Whisper, ffmpeg, anything), two things bite in sequence. The second one deletes recordings.

## 1. The file you receive mid-recording has no index

Android recorder apps that write MP4/M4A (Fossify Voice Recorder is the one I use, the stock recorders on several phones behave the same) write the `moov` atom, the index the container needs to be playable, only when the recording **stops**. While recording, the file on the phone is a growing `mdat` blob with no index.

Syncthing does not wait for the file to be finished. It ships the file in blocks as it grows, so the receiving side gets a snapshot that is:

- non-empty,
- size-stable for a couple of seconds between sync batches (so a naive "wait until the size stops changing" guard passes),
- and unplayable. ffprobe says `moov atom not found`, a Whisper server returns HTTP 500 immediately.

The first symptom is a transcription job failing instantly on a long recording, then succeeding on its own some time later, once the finished file has synced over the fragment. That case is harmless if your watcher tolerates it.

## 2. Moving the fragment out of the folder deletes the recording on the phone

The dangerous part. If the shared folder is `sendreceive` on both sides and your watcher moves the broken file to a `failed/` directory (or deletes it), Syncthing propagates that as a **delete** back to the phone while the app is still writing to it. The recorder keeps writing to an unlinked file, the recording ends, and there is nothing to sync. I lost a recording learning this.

So the rule for a fragment with no `moov`: **leave it exactly where it is.** The finished file will sync over it. Only give up on it after long enough that no recording could still be in progress.

## The gate

Bash, in the watcher loop, before anything touches the file:

```bash
# Require size stable over 2s (catches files still being written by Syncthing).
s1=$(stat -c%s "$f" 2>/dev/null || echo 0); sleep 2
s2=$(stat -c%s "$f" 2>/dev/null || echo 0)
if [[ "$s1" != "$s2" || "$s2" -eq 0 ]]; then
    echo "skip (still being written): $f"; continue
fi

# MP4/M4A without a moov atom = recording still in progress on the phone.
# Do NOT move or delete it: the folder is sendreceive and that propagates a
# delete to the phone mid-recording. Leave it; the finished file syncs over it.
ext="${f##*.}"; ext="${ext,,}"
if [[ "$ext" == "m4a" || "$ext" == "mp4" ]] && ! grep -aq moov "$f"; then
    age=$(( $(date +%s) - $(stat -c%Y "$f") ))
    if (( age < 3600 )); then
        echo "skip (no moov index yet, recording likely in progress): $f"; continue
    fi
    echo "FAILED (truncated m4a, still no moov index after 1h): $f"
    mv -f "$f" "$FAILED/"      # only now is it safe to touch it
    continue
fi
```

`grep -aq moov` is a crude but reliable test: the atom name appears as a literal four byte string in the file, and it is absent until the recorder finalises. `-a` treats the binary as text. One hour is a guess at "longer than any recording I make"; set it to suit.

Also skip Syncthing's own temporaries (`*.syncthing.*`, `~syncthing~*`, `.*`) before any of this.

## Telling the two failure cases apart afterwards

If a fragment does end up in `failed/`:

- **A much larger file with the same name exists in your processed folder**: the recording finished and synced fine, the fragment is junk. Delete it.
- **No larger twin ever arrived, but later recordings synced normally**: the phone app crashed or was killed mid-recording, and the fragment is the only copy.

## Recovering the only-copy case

A moov-less fragment can be repaired with [untrunc](https://github.com/anthwlock/untrunc) using any healthy recording from the same app and phone as a reference:

```bash
docker build -t untrunc:local https://github.com/anthwlock/untrunc.git
docker run --rm -v "$PWD:/data" untrunc:local /data/reference.m4a /data/broken.m4a
# writes broken.m4a_fixed.m4a
```

(The prebuilt images on Docker Hub were dead or unpullable when I tried, building from the repo took a couple of minutes.) A ten minute fragment came back complete this way.

One more trap on the salvaged file: if your pipeline discards empty transcripts as "silence", check the audio first. A recording made in a pocket can be noise dominated and come back empty from Whisper while the speech is still in there. A high-pass around 180 Hz, a low-pass around 7.5 kHz and `speechnorm` in ffmpeg was enough to get a full transcript out of one such file.

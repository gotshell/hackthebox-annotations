Just download the zip file and unzip.
```
7z x filename -p**********
```
The result is an Android Backup file.
```
file cat.ab
cat.ab: Android Backup, version 5, Compressed, Not-Encrypted
```
```
dd if=cat.ab bs=1 skip=24 of=cat.ab.deflate
```
dd is a tool for copying raw data byte by byte (or block by block). The parameters:
- if=cat.ab → input file, the source file
- of=cat.ab.deflate → output file, where to write the result
- bs=1 → block size of 1 byte (copies one byte at a time - slow, but precise for skipping)
- skip=24 → skips the first 24 blocks of the input, i.e. the first 24 bytes

In practice: .ab files start with a plain-text header like:
ANDROID BACKUP
5
1
none

This header takes up roughly 24 bytes (it depends on the version — sometimes you need to count it manually, including newlines). The dd command discards this header and copies everything else — i.e. the compressed data stream — into a new file called cat.ab.deflate.
So after this command you've isolated the compressed binary "payload," without the initial text portion in front of it.
```
python3 -c "import zlib; data=open('cat.ab.deflate','rb').read(); open('cat.tar','wb').write(zlib.decompress(data))"
```
- open('cat.ab.deflate','rb').read() → opens the binary file just created and reads it all into memory as bytes
- zlib.decompress(data) → decompresses those bytes using the zlib/deflate algorithm (the same one gzip uses, but without gzip's specific header — Android Backup uses "raw" zlib)
- open('cat.tar','wb').write(...) → writes the decompressed result to a new file called cat.tar

Conceptual summary
  cat.ab (Android Backup)
   │
   ├── [24 bytes of text header]  ← discarded by dd
   │
   └── [zlib-compressed stream]  ← saved as cat.ab.deflate
                │
                ▼ (python decompression)
           cat.tar  ← a real tar archive, ready to be extracted

In practice you've "unpacked" the proprietary Android Backup container to pull out the original tar archive that Android had compressed inside it.

Almost forgot to search for the flag. Just look around, it's pretty simple to see it.


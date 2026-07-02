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
```
python3 -c "import zlib; data=open('cat.ab.deflate','rb').read(); open('cat.tar','wb').write(zlib.decompress(data))"
```

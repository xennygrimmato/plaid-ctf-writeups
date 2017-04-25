# zipper

## Problem

Something doesn't seem quite right with [this](https://play.plaidctf.com/files/zipper_50d3dc76dcdfa047178f5a1c19a52118.zip) zip file.

Can you fix it and get the flag?

## Solution

On running `unzip zipp.zip`, we come to know that the filename is too long.

```bash
Archive:  zipp.zip
warning:  filename too long--truncating.
[  ]
:  bad extra field length (central)
```

In order to fix this metadata field, I tried to understand what kind of a zip file this was.

The first 4 bytes were: `504B0304` - this meant the file was zipped using the `PKZip`
format.

(On Mac OS X, I use iHex to see the zip file in hex.)

On [this](https://users.cs.jmu.edu/buchhofp/forensics/formats/pkzip.html) link, I found
a very elaborate explanation of the PkZip format.

On running `zipdetails zipp.zip`, this is what I got:

```bash
0000 LOCAL HEADER #1       04034B50
0004 Extract Zip Spec      14 '2.0'
0005 Extract OS            00 'MS-DOS'
0006 General Purpose Flag  0002
     [Bits 1-2]            2 'Fast Compression'
0008 Compression Method    0008 'Deflated'
000A Last Mod Time         4A9299FC 'Tue Apr 18 19:15:56 2017'
000E CRC                   532EA93E
0012 Compressed Length     00000046
0016 Uncompressed Length   000000F6
001A Filename Length       2329
001C Extra Length          001C
Truncated file (got 206, wanted 9001):
```

`001A Filename Length       2329` was what we had to fix.
The zip file consists of 236 bytes, so obviously the filename can't be 9001 characters long.

In 2 places in the hex format of the zip, I tried experimenting by changing the filename length
from `2329` to `0002`. On running `zipdetails -v zipp.zip`, we get:

```
0000 0004 50 4B 03 04 LOCAL HEADER #1       04034B50
0004 0001 14          Extract Zip Spec      14 '2.0'
0005 0001 00          Extract OS            00 'MS-DOS'
0006 0002 02 00       General Purpose Flag  0002
                      [Bits 1-2]            2 'Fast Compression'
0008 0002 08 00       Compression Method    0008 'Deflated'
000A 0004 FC 99 92 4A Last Mod Time         4A9299FC 'Tue Apr 18 19:15:56 2017'
000E 0004 3E A9 2E 53 CRC                   532EA93E
0012 0004 46 00 00 00 Compressed Length     00000046
0016 0004 F6 00 00 00 Uncompressed Length   000000F6
001A 0002 02 00       Filename Length       0002
001C 0002 1C 00       Extra Length          001C
001E 0002 00 00       Filename              '  '
0020 0002 00 00       Extra ID #0001        0000 ''
0022 0002 00 00         Length              0000
0024 0000               Extra Payload
0024 0002 00 00       Extra ID #0002        0000 ''
0026 0002 55 54         Length              5455
0028 0046 09 00 03 5B PAYLOAD               ...[..X[..Xux.............SP
          C8 F6 58 5B                       ....+.(..J..Q../U.H,KUHN,()-
          C8 F6 58 75                       JMQ(.HUH.IL..
          78 0B 00 01
          04 E8 03 00
          00 04 E8 03
          00 00 53 50
          20 04 B8 14
          08 2B F1 28
          AD AA 4A CC
          D0 51 A8 CC
          2F 55 C8 48
          2C 4B 55 48
          4E 2C 28 29
          2D 4A 4D 51
          28 C9 48 55
          48 CB 49 4C
          B7 E2


Unexpecded END at offset 0000006E, value 710E700A
Done
```

The result is better than before, but we are yet to recover the Central Header.

After a few iterations, I got a better result with filename length set to `0008`.


```
0000 0004 50 4B 03 04 LOCAL HEADER #1       04034B50
0004 0001 14          Extract Zip Spec      14 '2.0'
0005 0001 00          Extract OS            00 'MS-DOS'
0006 0002 02 00       General Purpose Flag  0002
                      [Bits 1-2]            2 'Fast Compression'
0008 0002 08 00       Compression Method    0008 'Deflated'
000A 0004 FC 99 92 4A Last Mod Time         4A9299FC 'Tue Apr 18 19:15:56 2017'
000E 0004 3E A9 2E 53 CRC                   532EA93E
0012 0004 46 00 00 00 Compressed Length     00000046
0016 0004 F6 00 00 00 Uncompressed Length   000000F6
001A 0002 08 00       Filename Length       0008
001C 0002 1C 00       Extra Length          001C
001E 0008 00 00 00 00 Filename              '        '
          00 00 00 00
0026 0002 55 54       Extra ID #0001        5455 'UT: Extended Timestamp'
0028 0002 09 00         Length              0009
002A 0001 03            Flags               '03 mod access'
002B 0004 5B C8 F6 58   Mod Time            58F6C85B 'Wed Apr 19 07:45:55 2017'
002F 0004 5B C8 F6 58   Access Time         58F6C85B 'Wed Apr 19 07:45:55 2017'
0033 0002 75 78       Extra ID #0002        7875 'ux: Unix Extra Type 3'
0035 0002 0B 00         Length              000B
0037 0001 01            Version             01
0038 0001 04            UID Size            04
0039 0004 E8 03 00 00   UID                 000003E8
003D 0001 04            GID Size            04
003E 0004 E8 03 00 00   GID                 000003E8
0042 0046 53 50 20 04 PAYLOAD               SP ....+.(..J..Q../U.H,KUHN,()-
          B8 14 08 2B                       JMQ(.HUH.IL...p.q.N3(J.+6L...L..%.&.(
          F1 28 AD AA                       ..
          4A CC D0 51
          A8 CC 2F 55
          C8 48 2C 4B
          55 48 4E 2C
          28 29 2D 4A
          4D 51 28 C9
          48 55 48 CB
          49 4C B7 E2
          0A 70 0E 71
          AB 4E 33 28
          4A CD 2B 36
          4C 2E 8E AF
          4C AC AC 25
          C3 26 EA 28
          01 00

0088 0004 50 4B 01 02 CENTRAL HEADER #1     02014B50
008C 0001 1E          Created Zip Spec      1E '3.0'
008D 0001 03          Created OS            03 'Unix'
008E 0001 14          Extract Zip Spec      14 '2.0'
008F 0001 00          Extract OS            00 'MS-DOS'
0090 0002 02 00       General Purpose Flag  0002
                      [Bits 1-2]            2 'Fast Compression'
0092 0002 08 00       Compression Method    0008 'Deflated'
0094 0004 FC 99 92 4A Last Mod Time         4A9299FC 'Tue Apr 18 19:15:56 2017'
0098 0004 3E A9 2E 53 CRC                   532EA93E
009C 0004 46 00 00 00 Compressed Length     00000046
00A0 0004 F6 00 00 00 Uncompressed Length   000000F6
00A4 0002 08 00       Filename Length       0008
00A6 0002 18 00       Extra Length          0018
00A8 0002 00 00       Comment Length        0000
00AA 0002 00 00       Disk Start            0000
00AC 0002 01 00       Int File Attributes   0001
                      [Bit 0]               1 Text Data
00AE 0004 00 00 B4 81 Ext File Attributes   81B40000
00B2 0004 00 00 00 00 Local Header Offset   00000000
00B6 0008 00 00 00 00 Filename              '        '
          00 00 00 00
00BE 0002 55 54       Extra ID #0001        5455 'UT: Extended Timestamp'
00C0 0002 05 00         Length              0005
00C2 0001 03            Flags               '03 mod access'
00C3 0004 5B C8 F6 58   Mod Time            58F6C85B 'Wed Apr 19 07:45:55 2017'
00C7 0002 75 78       Extra ID #0002        7875 'ux: Unix Extra Type 3'
00C9 0002 0B 00         Length              000B
00CB 0001 01            Version             01
00CC 0001 04            UID Size            04
00CD 0004 E8 03 00 00   UID                 000003E8
00D1 0001 04            GID Size            04
00D2 0004 E8 03 00 00   GID                 000003E8

00D6 0004 50 4B 05 06 END CENTRAL HEADER    06054B50
00DA 0002 00 00       Number of this disk   0000
00DC 0002 00 00       Central Dir Disk no   0000
00DE 0002 01 00       Entries in this disk  0001
00E0 0002 01 00       Total Entries         0001
00E2 0004 4E 00 00 00 Size of Central Dir   0000004E
00E6 0004 88 00 00 00 Offset to Central Dir 00000088
00EA 0002 00 00       Comment Length        0000
Done
```

The filename is set to null bytes: `00B6 0008 00 00 00 00 Filename              '        '`.

Let's set the 8-byte filename to `AAAAAAAA`, that is in hex `41 41 41 41 41 41 41 41`.

Note that we need to change that in 2 places - the local header and the central header.

Running `unzip zipp.zip` gives us this:

```
Archive:  zipp.zip
  inflating: AAAAAAAA
```

`cat AAAAAAAA` gave us the flag:

```
Huzzah, you have captured the flag:
PCTF{f0rens1cs_yay}
```

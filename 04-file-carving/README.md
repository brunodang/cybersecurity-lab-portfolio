# Task 4: File Carving (Data Recovery)

> **Authorized lab work.** All work was done inside my own Kali Linux VM on an isolated network. The "evidence" disk image was built by me for this exercise from harmless system image files. No real devices or personal data were involved.

## Objective

Recover files from a raw disk image **without relying on the filesystem**, by finding the files' byte signatures (headers and footers) and cutting them out. Then prove the recovered files are byte-for-byte identical to the originals using SHA-256 hashes.

## Environment

| Component | Detail |
|---|---|
| Host | Kali Linux (VirtualBox, isolated) |
| Evidence | `evidence.img`, a 15 MB raw disk image |
| Identify | binwalk |
| Carve | scalpel 1.60 (successor to foremost) |
| Verify | sha256sum |

## Method

**1. Built the evidence image.** Created a blank 15 MB raw image and placed two PNG files inside it at known offsets (1 MB and 5 MB). The files have no filesystem entry pointing to them, the same state as deleted data sitting in unallocated space.

```bash
dd if=/dev/zero of=evidence.img bs=1M count=15
dd if=photo1.png of=evidence.img bs=512 seek=2048  conv=notrunc
dd if=photo2.png of=evidence.img bs=512 seek=10240 conv=notrunc
```

**2. Wrote a signature config** telling scalpel what PNG and JPEG files look like at the byte level:

```
png  y  5000000  \x89PNG\x0d\x0a\x1a\x0a  \x49\x45\x4e\x44\xae\x42\x60\x82
jpg  y  5000000  \xff\xd8\xff  \xff\xd9
```

A PNG always starts with `89 50 4E 47 0D 0A 1A 0A` and ends with the `IEND` chunk. Scalpel carves everything between them.

**3. Carved the image:**

```bash
scalpel -c scalpel.conf -o recovered evidence.img
```

**4. Verified integrity** by hashing originals and recovered files:

```bash
sha256sum photo1.png photo2.png
sha256sum recovered/png-0-0/*.png
```

## Results

Scalpel carved **2 files**, at exactly the offsets where the data sat:

| Recovered file | Start offset | Length | Matches |
|---|---|---|---|
| `00000000.png` | 1,048,576 | 2,287 bytes | photo1.png |
| `00000001.png` | 5,242,880 | 255,076 bytes | photo2.png |

SHA-256 hashes are identical:

| File | SHA-256 |
|---|---|
| photo1.png / 00000000.png | `bba6f9845c317df4ae37494b1e6226c58a7646937833bfec6595a12f066d45c6` |
| photo2.png / 00000001.png | `2a2b9a73ebc45fcdaf929beb50cb0f87fac16e3851170b546dec864c0be1f4a4` |

> 📸 Step-by-step screenshots are in the PDF report in this folder.

## Troubleshooting log

- **First build attempt carved 0 files.** I originally formatted the image as FAT, mounted it, copied files in and deleted them, all in one pasted block. The `sudo` password prompt consumed the following pasted lines, so the files were never written into the image. Scalpel correctly found nothing. Lesson: never paste multi-line blocks containing `sudo` without authenticating first (`sudo -v`).
- **binwalk returned no results** on the empty image, which was consistent with the data simply not being there. After rebuilding, scalpel recovered both files.

## Analysis: why it matters

- **Deleting a file rarely erases it.** The filesystem forgets where it is, but the bytes stay on disk until overwritten. Carving recovers them by structure, not by name.
- **Plain text can't be carved this way.** It has no fixed header or footer, which is why carving targets structured formats like images, PDFs and archives.
- **Hashes are the proof.** Matching SHA-256 values show the evidence was recovered intact, which is the standard for forensic integrity and chain of custody.

## Defensive takeaways

1. **Use secure deletion or full-disk encryption** on drives holding sensitive data. Normal delete leaves recoverable data.
2. **Wipe or destroy media before disposal** (NIST SP 800-88 sanitization).
3. **Hash evidence at collection time** and work on copies, never the original.

## Skills demonstrated

- File carving with scalpel and signature configuration

- **See also:** [Case 2 – Recovering files from an evidence disk image](case2-evidence-image-README.md)
- Understanding file signatures (magic bytes, headers and footers)
- Raw disk image handling with `dd`
- Evidence integrity verification with SHA-256
- Troubleshooting a failed recovery and documenting root cause

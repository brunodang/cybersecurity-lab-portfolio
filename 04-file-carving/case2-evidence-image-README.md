# Case 2: Recovering Files from an Evidence Disk Image

> **Authorized training exercise.** The disk image was **supplied by my cybersecurity course** for forensic practice and examined on my own Kali Linux VM. The course name, download link and the planted search keyword are **blacked out** in the screenshots. The recovered files themselves are not published here, only their names, sizes and SHA-256 fingerprints.

## The task

Given an unknown disk image, find three things:

| # | Question | Answer |
|---|---|---|
| 6 | Recover all Excel files, if any | **No real Excel workbooks exist.** The only spreadsheet is `Accounting/employee_database.csv` (a CSV that opens in Excel, 1,000 records). It is **byte-for-byte identical to the file stolen in the packet-capture case.** |
| 7 | Find the PDF containing the target keyword | **`00624240.pdf`**, a one-page PDF planted as evidence. A second match was a false positive. |
| 8 | Find the image whose subject is the target keyword | **`00056000.png`**, a two-button meme with the keyword in its labels and caption |

Case 1 in this folder proves *how* carving works on an image I built myself. This case applies it to evidence I didn't create, where I had to search, filter and verify.

## Environment

| Item | Detail |
|---|---|
| Analyst VM | Kali Linux (VirtualBox), temporary internet for downloads and tool installs |
| Evidence | `diskimage.001`, 916 MB raw image of a Windows **NTFS** drive (renamed from the course filename) |
| Evidence SHA-256 | `506f2d41b3715761045078c3b90a8189fb300566cb0597081b3779c27fa77ab6` |
| Tools | **foremost** (carving), **Sleuth Kit** `fls`/`icat` (reading the drive's own file list), **pdftotext** (PDF text), **tesseract** (reading text in images) |

## Setup

```bash
gdown <file-id>                         # course-supplied evidence (link redacted)
unzip "<course name> - DISKIMAGE.zip"
mv "<course name> - DISKIMAGE.001" diskimage.001
file diskimage.001                      # → NTFS drive
sha256sum diskimage.001                 # fingerprint before analysis
sudo apt install -y foremost
```

A `.001` file is the first piece of a raw disk copy made by tools like FTK Imager. Here it was the only piece, so it held the whole drive.

## Carving the drive

```bash
foremost -t jpg,png,pdf,ole,zip -i diskimage.001 -o carved
grep -A20 "FILES EXTRACTED" carved/audit.txt
```

- `ole` covers older Office files (`.xls`, `.doc`).
- `zip` covers newer ones (`.xlsx`, `.docx`), which are zip files inside.

**Result: 305 files.** 179 JPG, 114 PNG, 10 PDF, 2 ZIP, and **0** older Office files.

---

## Q6: Excel files

Carving found no Office files, but newer Excel files are zips, so the two zips needed checking. I also checked the drive a second way: NTFS keeps its own list of every file, which the Sleuth Kit can read.

```bash
for z in carved/zip/*; do unzip -l "$z" | head -12; done
fls -r -p diskimage.001 | grep -i -E "\.(xls|xlsx|xlsm|xlsb|csv)$"
```

- **Both zips** were broken pieces of the same archive of RFC documents (internet standards papers), with no `xl/` folder, so not Excel.
- **The file list** held exactly one spreadsheet-type file: `Accounting/employee_database.csv`. It wasn't deleted.

I copied that file out by its file-list number, and compared it with the file stolen over FTP in the packet-capture case:

```bash
icat diskimage.001 40 > recovered_employee_database.csv
sha256sum recovered_employee_database.csv ~/pcaplab/evidence/employee_database.csv
```

**Both fingerprints were `cbe10a41e5215218f82730546f5dfbe02a87d3daab2c1a671ab7e00e38a70319`.** The file on this drive is exactly the file that was stolen.

**What I learned.** Carving and the file system's own list answer different questions. Carving finds data by its shape, even without a name. The file list gives names, folders and whether a file was deleted. Using both gave a confident "no Excel files," and found a connection between two separate investigations.

## Q7: The PDF with the target keyword

```bash
for f in carved/pdf/*.pdf; do echo "$(pdftotext "$f" - | grep -ci <keyword>)  $f"; done
```

Two PDFs matched. Showing the surrounding lines told them apart:

| File | What it is | Verdict |
|---|---|---|
| `00624240.pdf` | One page, its title and only line contain the keyword | ✅ The planted evidence |
| `00861200.pdf` | RFC 2616 (HTTP/1.1), 114 pages. The keyword appears only inside a web address in its reference list. | ❌ False match |

**Answer:** `00624240.pdf`, SHA-256 `10dd9a9ae7006e9e94447a3f1b3586fa229c24f04feffe1310394e9bcd189081`.

**What I learned:**
- **Check that your tool works before you trust "nothing found."** My first search reported zero matches in every PDF. In fact the PDF tools weren't installed, and the `2>/dev/null` in my command hid the "command not found" errors, so `grep` was counting words in empty output. `which pdftotext pdfinfo` exposed it.
- **A keyword match isn't automatically the answer.** You have to read it in context.

## Q8: The image about the target keyword

293 images, most of them scanned pages and small icons. I sorted by size to find real pictures, browsed thumbnails, and ran OCR (`tesseract`) to read any text inside the images.

**Answer:** `00056000.png` (506,856 bytes), a two-button meme. One button and the bottom caption carry the keyword (blacked out here). It was the only image whose subject was the keyword.

**What I learned.** Some evidence can only be confirmed by looking at it. Tools narrow the field (size, thumbnails, OCR), and a person makes the final call.

---

## Big finding

The `employee_database.csv` stored in this drive's **Accounting** folder has the same SHA-256 fingerprint as the file an attacker downloaded over anonymous FTP in the [packet-capture case](../02-packet-capture-analysis/). Two independent pieces of evidence, a network capture and a disk image, point to the same data and the same breach.

## Troubleshooting log

| Problem | Cause | Fix |
|---|---|---|
| foremost not installed (Case 1) | Lab VM had no internet | Installed it once Kali had internet |
| Every PDF showed 0 matches | PDF tools missing; `2>/dev/null` hid the errors | `which` to check, then installed `poppler-utils` |
| `pdfinfo` printed nothing | Same missing tools | Same fix |
| Some PDFs were scans with no text | Pages stored as pictures | OCR with `tesseract` |
| Carved zips reported "missing … bytes" | Only fragments of a larger archive survived on the drive | Listed their contents to identify them (RFC documents, not Excel) |

## Skills demonstrated

- File carving with foremost on an unknown evidence image
- Reading an NTFS drive's file list with Sleuth Kit (`fls`) and extracting a file by number (`icat`)
- Telling true keyword matches from false ones
- OCR for scanned documents and images
- Linking evidence across two investigations with SHA-256
- Spotting a broken tool behind a "nothing found" result

> 📄 Full step-by-step report with screenshots: `Case2_Evidence_Image_Carving_Report.pdf` in this folder.

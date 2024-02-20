# LockBit 3.0 Decryptor Breakdown
With the news of the seizure of the Lockbit 3.0 infrastructure by several different agencies and countries, a new decryptor was released to help victims. Below are my notes during the reverse enigneering of it.

# The Decryptor
First off, I want to say that I think providing keys, decryptors, or anything else to help victims of ransomware is superb. This breakdown is not to discredit any of the work done, but to learn about the process and decisions of the creators.

Getting started, the decryptor is hosted on [No More Ransom](https://www.nomoreransom.org/en/index.html) and is a ZIP file with 2 exe files inside. Using the PDF guide, we are told to use the `check_decryption_id.exe` file to check the Decryption ID (a value in the ransom note) against known keys. This is where I will focus most of my notes, as the other file focuses on predicting if files can be decrypted.

## Ghidra
I loaded the file into Ghidra to get started and found quite a few functions. This isn't suprising for a Windows binary and often makes a mess of attempting to find custom code. Looking at the strings output, I noticed a lot of nonsense strings (or strings without much format or meaning). This signals to me that the binary is encrypted or compressed. Digging around some more, I found some code that dropped a file or folder into the `TEMP` directory and had the name `onefile`. A quick Google search and some reading lead me to [Nuitka](https://nuitka.net/index.html). If I had run Detect It Easy first, I would have been able to see this much faster.

![Entropy of the file as viewed in Detect It Easy](https://github.com/TKems/LockBit-Decryptor-Breakdown/blob/ffaeb2d537bbe892f8123a42e6270799facd25ab/images/entropy-of-PE.png "Entropy of the file showing compression")

## Nuitka
Nuitka has a handy feature to create a single file output. In order to do this, Zstandard compression is used. This can be verified in the `check_decryption_id.exe` binary as the header matches the [Nuitka compression header](https://github.com/Nuitka/Nuitka/blob/develop/nuitka/tools/onefile_compressor/OnefileCompressor.py) `KAY` and the following bytes (big endian) match the Zstandard [magic bytes](https://github.com/facebook/zstd/blob/dev/doc/zstd_compression_format.md) `0xFD2FB528`.

## Decompression
While I could have written a Ghidra plugin or script to decompress the data, I found an easier way. Some quick Googling lead me to a Flare-On 10 Challenge Solution. [Challenge 7](https://services.google.com/fh/files/misc/7-flake-flareon10.pdf) was a pretty close match and gave me a few ideas on how to grab a decompressed version of the code. However, due to the CLI nature of the binary, I could not freeze the files in TEMP without them getting deleted when the binary quit. A quick fix was to use (abuse) the Windows permissions in order to prevent files from being deleted. Using the advanced permission editor, I created a deny permission for the TEMP folder, subfolder, and files for the group Everyone. Now, files can be written but not removed. 

## Going Further
Once I had a file that was decompressed, I threw it back in Ghidra and expected nice, clean C. Nope! I was shown some ugly byte code. ![Bytecode as viewed in Ghidra](https://github.com/TKems/LockBit-Decryptor-Breakdown/raw/master/images/ghidra-decompressed-code.png) While searching some more, I searched for strings and noticed the byte code contained a lot of interesting strings that looked like Python. As I scrolled through the strings, I noticed some interesting values that gave hints on what the code did. Then I found the list of hashes.

![Image of SHA-256 Hashes in the Ghidra string search tool](https://github.com/TKems/LockBit-Decryptor-Breakdown/raw/master/images/sha256-hashes.png)

## Lookup Table of Hashes
Near the bottom of the string search, I found a list of seemingly random data in string form, but all around the same length and starting with `u`, I knew that these couldn't be hashes due to the starting char, but the length was too long at 65. Removing the first char (either `u` or `a`) give the right length for a SHA-256 hash, bingo! We found the lookup table! After exporting from Ghidra, I counted 923 hashes. This matches the ~1000 keys listed as 'recovered' by authorities.

Continuing down, I found a reference to SHA-256 that confirmed my assumptions. Here is the order of operations of the binary:
1. Launch and write decompressed binary to `TEMP` directory (along with supporting Python bytecode files and DLLS).
2. Ask the user for their 16-digit (or longer) decryption ID that can be found in the ransom note. (This is likely just the first 16 bytes of the public key)
3. Hash the decryption ID with SHA-256 and compare with a list of known values.
4. If found, tell the user to contact an email address (I'm not listing it here for privacy/spam reasons).
5. If not found, tell the user to come back later...

![Message shown to users if the Decryption ID matches](https://github.com/TKems/LockBit-Decryptor-Breakdown/raw/master/images/message-in-exe-censored.png "It's a match! Email us for your key")

# Conclusions
While not the most interseting file I've reversed, I think this is a good example of how to pack software without the need for overly complex encryption or obfuscation. I would have liked the authors to include the Python source, but understand that it might not be possible.

## Why not just share the raw keys?
I asked myself this question and thought at first that sharing the keys would be the easiest option. Victims can brute force their files with all known keys and not have to worry! However, what if a bad actor has a copy of the encrypted data? They too can decrypt it with leaked keys! Therefore, I understand why keys are kept secret and this system of decryption ID checks are in place.




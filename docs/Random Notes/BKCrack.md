## Description
`bkcrack` is a C++ based tool which implements [a known plaintext attack on the PKZIP stream cipher](https://link.springer.com/chapter/10.1007/3-540-60590-8_12) . It can be used to crack password-protected ZIP files under some circumstances, gaining access to the content of the ZIP.

## Setup
The easiest way is to download the precompiled packages on [GitHub](https://github.com/kimci86/bkcrack/releases), and execute the binary within:
```bash
tar -xvzf bkcrack-1.8.1-Linux-x86_64.tar.gz
./bkcrack-1.8.1-Linux-x86_64/bkcrack
```

- For Windows Systems, the `Microsoft Visual C++ Redistributable package` is needed.

Alternatively, you can compile the project yourself using `CMake` by cloning the git repository, and executing these commands:
```bash
cmake -S . -B build -DCMAKE_INSTALL_PREFIX=install
cmake --build build --config Release
cmake --build build --config Release --target install
```

## Commands
#### List Entries
```bash
bkcrack -L protected_archive.zip
```
This is the most important command, as this lists the contents of the ZIP file. For each file within the ZIP, it shows you the `Encryption` Algorithm used, the type of `Compression` used, the `Uncompressed`- and `Packed Size` of the files, and their `Names`.

- `Encryption`: Entries which have the `ZipCrypto` value are vulnerable to this attack.
- `Compression`: May be `Deflate` or `Store`. Entries with `Store` are generally easier to exploit, so target them.
- `Size`: Very useful if you are looking for a exact copy of a file, to find out if you have the one within the ZIP file.
- `Name`: Very useful to find out the Extension of a file.

#### Attacking `Store` Files
Here, you will need to know _at least_ 12 Bytes of known Plaintext of the file. More Bytes lead to faster attacks!
You have two options here:

- Take educated guesses on these 12 Bytes. Examples:
	- `SVG` files probably start with `<?xml version="1.0"`
	- `PDF` or `PNG` files have Magic Bytes which are always used to start the file.
	- A `PNG` will always have 8 Header Bytes: `89 50 4E 47 0D 0A 1A 0A` The rest will have to be guessed, as you need 12 Bytes (You can try to fill it with 0's)
- Find the exact copy of the file:
	- If it is a well known file like a `License.txt` for a specific software with a specified version, you can know that you have the exact file which is within the protected ZIP.
	- Also look for `DLLs`.
	- Compare the file sizes!!!

After gathering the known `plaintext` of the target file and the fully qualified name of the target file (from `-L`), you can issue the following command:
```bash
bkcrack -C protected_archive.zip -c /path/to/targetfile -p plaintext
```
Small plaintext files can take a long time!

If Successful, you will receive a key in the form of:
```bash
[17:48:03] Keys
c4490e28 b414a23d 91404b31
```

#### Attacking `Deflate` Files
It is not feasible to take educated guesses on deflated files. Here you _need_ to find the exact copy of the file.
If you manage to find the exact copy of the file online, you must guess the compression software and compression levels to generate the correct plaintext.
Different compression levels using the `zip` CLI might look like this:
```bash
zip -1 guess1.zip exact_file
zip -2 guess2.zip exact_file
# ...
zip -9 guess9.zip exact_file
```
You can then attack the files as follows:
```bash
bkcrack -C protected_archive.zip -c /path/to/targetfile -P guess1.zip -p exact_file
# ...
```

If Successful, you will again receive a key in the form of:
```bash
[17:48:03] Keys
c4490e28 b414a23d 91404b31
```

### Removing Password
If you managed to gather the secret key, you may create a non-protected-ZIP containing the contents of the protected ZIP as follows:
```bash
bkcrack -C protected_archive.zip -k c4490e28 b414a23d 91404b31 -D archive_without_pw.zip
```

- `archive_without_pw.zip` can then be unzipped and read.

You can also try to retrieve the password which was used to protect the ZIP to start a password reuse attack, [info here](https://github.com/kimci86/bkcrack/blob/master/example/tutorial.md#recovering-the-original-password)
## Additional notes

- [bkcrack github repo](https://github.com/kimci86/bkcrack/tree/master)

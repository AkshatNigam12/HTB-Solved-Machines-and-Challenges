# 🚪 Hack The Box – Simple Encoder (Easy)

Machine Type: Linux
Solved by: Akshat Nigam
Category: Reverse Engineering, Crypto
Difficulty: Easy
Attack Surface: Local Binary (Executable)
-----------------------------------------

## Challenge Description

> During one of our routine checks on the secret flag storage server, we discovered it had been hit by ransomware!
 The original flag data is gone, but luckily, we still have both the encrypted file and the encryption program itself.
---
## Getting Started

We are provided with a zip file named `Simple Encryptor.zip`. After extraction:

```bash
unzip 'Simple Encryptor.zip'
```

We find a directory `rev_simpleencryptor/` containing:

* `flag.enc` — the encrypted flag file
* `encrypt` — the binary used to encrypt the flag

## Binary Analysis
---
### Initial Recon with GDB

```bash
gdb encrypt
set disassembly-flavor intel
disass main
```
The disassembly shows standard libc functions being used: `fopen`, `fseek`, `ftell`, and `fread`. This indicates file I/O operations, probably reading `flag`, modifying it, and saving as `flag.enc`.

---
### Ghidra Decompilation Output (Simplified)

```c
int main(void) {
  FILE *flag_fp = fopen("flag", "rb");
  fseek(flag_fp, 0, SEEK_END);
  long flag_size = ftell(flag_fp);
  fseek(flag_fp, 0, SEEK_SET);

  char *buffer = malloc(flag_size);
  fread(buffer, flag_size, 1, flag_fp);
  fclose(flag_fp);

  uint32_t seed = (uint32_t)time(NULL);
  srand(seed);

  for (int i = 0; i < flag_size; i++) {
    buffer[i] ^= rand();
    int s = rand() & 7;
    buffer[i] = (buffer[i] << s) | (buffer[i] >> (8 - s));
  }

  FILE *enc_fp = fopen("flag.enc", "wb");
  fwrite(&seed, 1, 4, enc_fp);
  fwrite(buffer, 1, flag_size, enc_fp);
  fclose(enc_fp);

  return 0;
}
```
---
## Encryption Summary

1. Read `flag` file
2. Store contents in a buffer
3. Generate a seed from current `time()`
4. For each byte:

   * XOR with `rand()`
   * Left rotate by `rand() & 7` bits
5. Write `seed` and encrypted buffer to `flag.enc`
---
## Reversing the Encryption

Since the seed is stored in the first 4 bytes of the encrypted file, we can reverse the process.

## Decryption Algorithm

1. Read first 4 bytes of `flag.enc` as seed
2. Reseed RNG with it using `srand(seed)`
3. For each byte:

   * Reverse bit rotation
   * XOR with same `rand()` value
---
## Decryption Code in C

```c
#include <stdint.h>
#include <stdio.h>
#include <stdlib.h>

int main() {
  FILE *fp = fopen("flag.enc", "rb");

  uint32_t seed;
  fread(&seed, sizeof(seed), 1, fp);

  fseek(fp, 0, SEEK_END);
  long file_size = ftell(fp) - sizeof(seed);
  uint8_t *buffer = malloc(file_size);

  fseek(fp, sizeof(seed), SEEK_SET);
  fread(buffer, 1, file_size, fp);
  fclose(fp);

  srand(seed);
  for (int i = 0; i < file_size; i++) {
    int rand_xor = rand();
    int shift = rand() & 7;

    buffer[i] = (buffer[i] >> shift) | (buffer[i] << (8 - shift));
    buffer[i] ^= rand_xor;
  }

  printf("%s\n", buffer);
  free(buffer);
  return 0;
}
```
---
## Compile and Run

```bash
gcc -o decrypt decrypt.c
./decrypt
```

This gives us the original flag back.

---

**Summary:** This challenge was a great mix of binary analysis, reverse engineering, and understanding pseudo-random number generation. The key insight was the use of a predictable `srand(time(NULL))` seed and a reversible XOR + rotate encryption.

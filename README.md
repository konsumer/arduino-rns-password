# arduino-rns-password

Password-protected file storage for [microReticulum](https://github.com/attermann/microReticulum) identity keys on Arduino/ESP32 with an SD card.

Encrypts a blob of bytes with a password and writes it to a file. Reading back requires the same password. Designed for storing a Reticulum identity's 64-byte private key so the device can be powered off safely.

See also: [arduino-rns-encrypted-store](https://github.com/konsumer/arduino-rns-encrypted-store) for protecting message files using a loaded identity, and [arduino-rns-identity-vault](https://github.com/konsumer/arduino-rns-identity-vault) for a complete working example.

## Installation (PlatformIO)

Add to `lib_deps` in your `platformio.ini`:

```ini
lib_deps =
    https://github.com/konsumer/arduino-rns-password.git
    https://github.com/attermann/Crypto.git
```

## API

```cpp
#include "password.h"

// Encrypt `data` (len bytes) and write to `path`.
// Returns true on success.
bool password_protect(const char* path, const char* password,
                      const uint8_t* data, size_t len);

// Decrypt a file written by password_protect into `data` (must be len bytes).
// Returns true on success.
// Returns false if the file is missing, wrong size, or authentication fails
// (wrong password or corrupted file). No plaintext is written on failure.
bool password_open(const char* path, const char* password,
                   uint8_t* data, size_t len);
```

Zero the password buffer with `memset` as soon as you are done with it.

## Usage

```cpp
#include <SD.h>
#include "password.h"

// Save a 64-byte identity private key
uint8_t prv[64];   // fill with key material
char pw[64] = {0};
// ... read pw from serial / keypad ...

password_protect("/identity.bin", pw, prv, sizeof(prv));
memset(pw, 0, sizeof(pw));

// Load it back
uint8_t prv_loaded[64];
char pw2[64] = {0};
// ... read pw2 ...

if (password_open("/identity.bin", pw2, prv_loaded, sizeof(prv_loaded))) {
    // use prv_loaded ...
}
memset(pw2, 0, sizeof(pw2));
memset(prv_loaded, 0, sizeof(prv_loaded));
```

## Security design

### Key derivation — PBKDF2-HMAC-SHA256

Two independent 32-byte keys are derived from the password and a random 16-byte salt using PBKDF2 with 100,000 iterations:

- **Block 1** → AES-256-CTR encryption key
- **Block 2** → HMAC-SHA256 authentication key

Running two full iteration chains means an attacker must compute 200,000 PBKDF2 iterations per password guess. At ~500 ms per 10,000 iterations on an ESP32, guessing a single password takes roughly **10 seconds on the device's own hardware**.

The iteration count is `PBKDF2_ITERATIONS` in `password.h` and can be tuned for your hardware.

### Authenticated encryption

The HMAC-SHA256 tag covers the entire file (version + salt + IV + ciphertext). It is verified with a **constant-time comparison** before any decryption happens. A wrong password or a tampered file always produces an authentication failure — there is no plaintext oracle and no timing leak on the comparison.

### File format

```
┌──────────┬──────────┬──────────┬──────────────┬─────────────────┐
│ version  │ salt     │ IV       │ ciphertext   │ HMAC-SHA256     │
│ 1 byte   │ 16 bytes │ 16 bytes │ N bytes      │ 32 bytes        │
└──────────┴──────────┴──────────┴──────────────┴─────────────────┘
Total overhead: 65 bytes beyond the plaintext.
```

### Brute-force resistance

| Hardware | Guesses / second |
|---|---|
| ESP32 (the device itself) | ~0.1 |
| Modern laptop CPU | ~50–200 |
| RTX 4090 (high-end GPU) | ~2,000–5,000 |
| 100× RTX 4090 cloud cluster | ~200,000–500,000 |

Use at least a 4-word [diceware](https://diceware.dmuth.org/) passphrase or 10+ random characters. PBKDF2-SHA256 is not memory-hard, so a GPU cluster can attack it faster than a memory-hard function like Argon2id would allow — but Argon2id requires ~64 MB of working memory per operation, which rules it out on an ESP32.

## Dependencies

- [attermann/Crypto](https://github.com/attermann/Crypto) — SHA256, AES256, CTR, RNG
- [attermann/microStore](https://github.com/attermann/microStore) — file I/O

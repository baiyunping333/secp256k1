libsecp256k1
============

why fork?
-----------

add one function `secp256k1_ecdh_sk` in `include/secp256k1_ecdh.h`
compute the share key compatible with OpenSSL or ruby.  

example:
```
#include <stdlib.h>
#include "include/secp256k1.h"
#include "include/secp256k1_ecdh.h"

char *bin2hex(const unsigned char *bin, size_t len)
{
      char   *out;
      size_t  i;

      if (bin == NULL || len == 0)
          return NULL;

      out = malloc(len*2+1);
      for (i=0; i<len; i++) {
          out[i*2]   = "0123456789ABCDEF"[bin[i] >> 4];
          out[i*2+1] = "0123456789ABCDEF"[bin[i] & 0x0F];
      }
      out[len*2] = '\0';

      return out;
}

int hexchr2bin(const char hex, char *out)
{
      if (out == NULL)
          return 0;

      if (hex >= '0' && hex <= '9') {
          *out = hex - '0';
      } else if (hex >= 'A' && hex <= 'F') {
          *out = hex - 'A' + 10;
      } else if (hex >= 'a' && hex <= 'f') {
          *out = hex - 'a' + 10;
      } else {
          return 0;
      }

      return 1;
}

size_t hexs2bin(const char *hex, unsigned char **out)
  {
      size_t len;
      char   b1;
      char   b2;
      size_t i;

      if (hex == NULL || *hex == '\0' || out == NULL)
          return 0;

      len = strlen(hex);
      if (len % 2 != 0)
          return 0;
      len /= 2;

      *out = malloc(len);
      memset(*out, 'A', len);
      for (i=0; i<len; i++) {
          if (!hexchr2bin(hex[i*2], &b1) || !hexchr2bin(hex[i*2+1], &b2)) {
              return 0;
          }
          (*out)[i] = (b1 << 4) | b2;
      }
      return len;
}

int main(int argc, char** argv){
    const char* prik = "9f1a960eccb37fc2e4721c84b67818046dc1c33bdc075e4c7bcca64bb9ee31ca";

    secp256k1_context *ctx = secp256k1_context_create(SECP256K1_CONTEXT_SIGN);

    secp256k1_pubkey pubkey;
    secp256k1_ec_pubkey_create(ctx, &pubkey, prik);

    unsigned char *out = {0};
    char *scalar = "9455d27c13a96d0b8dbc78bbfcaa2e894c2fa419a6f462ca17754b9a33182dcb";
    hexs2bin(scalar, &out);

    unsigned char result_x[32] = {0};
    unsigned char result_y[1] = {0};
    secp256k1_ecdh_sk(ctx, result_x, result_y, &pubkey, out);
    //so this result_x is the compatible share key
    s = bin2hex(result_x, 32);
    printf("result x %s", s);
}
```



[![Build Status](https://travis-ci.org/bitcoin-core/secp256k1.svg?branch=master)](https://travis-ci.org/bitcoin-core/secp256k1)

Optimized C library for EC operations on curve secp256k1.

This library is a work in progress and is being used to research best practices. Use at your own risk.

Features:
* secp256k1 ECDSA signing/verification and key generation.
* Adding/multiplying private/public keys.
* Serialization/parsing of private keys, public keys, signatures.
* Constant time, constant memory access signing and pubkey generation.
* Derandomized DSA (via RFC6979 or with a caller provided function.)
* Very efficient implementation.

Implementation details
----------------------

* General
  * No runtime heap allocation.
  * Extensive testing infrastructure.
  * Structured to facilitate review and analysis.
  * Intended to be portable to any system with a C89 compiler and uint64_t support.
  * Expose only higher level interfaces to minimize the API surface and improve application security. ("Be difficult to use insecurely.")
* Field operations
  * Optimized implementation of arithmetic modulo the curve's field size (2^256 - 0x1000003D1).
    * Using 5 52-bit limbs (including hand-optimized assembly for x86_64, by Diederik Huys).
    * Using 10 26-bit limbs.
  * Field inverses and square roots using a sliding window over blocks of 1s (by Peter Dettman).
* Scalar operations
  * Optimized implementation without data-dependent branches of arithmetic modulo the curve's order.
    * Using 4 64-bit limbs (relying on __int128 support in the compiler).
    * Using 8 32-bit limbs.
* Group operations
  * Point addition formula specifically simplified for the curve equation (y^2 = x^3 + 7).
  * Use addition between points in Jacobian and affine coordinates where possible.
  * Use a unified addition/doubling formula where necessary to avoid data-dependent branches.
  * Point/x comparison without a field inversion by comparison in the Jacobian coordinate space.
* Point multiplication for verification (a*P + b*G).
  * Use wNAF notation for point multiplicands.
  * Use a much larger window for multiples of G, using precomputed multiples.
  * Use Shamir's trick to do the multiplication with the public key and the generator simultaneously.
  * Optionally (off by default) use secp256k1's efficiently-computable endomorphism to split the P multiplicand into 2 half-sized ones.
* Point multiplication for signing
  * Use a precomputed table of multiples of powers of 16 multiplied with the generator, so general multiplication becomes a series of additions.
  * Access the table with branch-free conditional moves so memory access is uniform.
  * No data-dependent branches
  * The precomputed tables add and eventually subtract points for which no known scalar (private key) is known, preventing even an attacker with control over the private key used to control the data internally.

Build steps
-----------

libsecp256k1 is built using autotools:

    $ ./autogen.sh
    $ ./configure
    $ make
    $ ./tests
    $ sudo make install  # optional

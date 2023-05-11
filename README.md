# Azam Codec

> _**TL;DR** : A lexicographically sortable multi-section string encoding of byte arrays._

A byte array encoding and decoding method, in big endian (network byte order), with markers to separate byte arrays into sections. The encoded symbols used are picked so that encoded string are lexicographically sortable based on the byte array values of each section, given comparing valid strings with the same number of section. Optimized to store multiple unsigned integers in a string, with human readable characters, and human input errors considered, by use of symbols inspired by Crockford's Base32 encoding. Having different set of characters in same case, representing higher and lower 4-bit of a byte makes the string resist case conversions.

## Implementations

| Language | Repository |
|---|---|
|Rust|[azamcodec-rs](https://github.com/azam/azamcodec-rs)|
|Java|[azamcodec-java](https://github.com/azam/azamcodec-java)|
|Javascript|[azamcodec-js](https://github.com/azam/azamcodec-js)|

## Specification

Given an array of byte array of size `m`, with each byte array has variable lengths `n1,...,nm`.

```
A = [[b₁₁, b₁₂, ..., b₁ₙ₁], [b₂₁, b₂₂, ..., b₂ₙ₂], ..., , [bₘ₁, bₘ₂, ..., bₘₙₘ]]
```

Loop for each byte array (let's call these sections).

```
S₁ = [b₁₁, b₁₂, ..., b₁ₙ₁]
S₂ = [b₂₁, b₂₂, ..., b₂ₙ₂]
...
Sₘ = [bₘ₁, bₘ₂, ..., bₘₙₘ]
```

Encode for each section

```
input: I (Nybble value)
output: S (Nybble symbol)
procedure LOW_SYMBOL(I) -> S
  return symbol at index B in '0123456789abcdef'
end procedure

input: I (Nybble value)
output: S (Nybble symbol)
procedure HIGH_SYMBOL(I) -> S
  return symbol at index B in 'ghjkmnpqrstvwxyz'
end procedure

input: S (Byte array)
output: V (Encoded string)
procedure ENCODE_SECTION (S) -> V
  V <- ''
  lead_nybble_checked <- false
  for i = 0..LENGTH(S) do
    byte <- S[i]
    high_nybble <- byte >> 4
    low_nybble <- byte & 0x0f
    if lead_nybble_checked is true OR high_nybble > 0 do
      S <- HIGH_SYMBOL(high_nybble)
      V <- V + S
      if i + 1 < LENGTH(S) do
        S <- LOW_SYMBOL(high_nybble)
        V <- V + S
      else do
        S <- HIGH_SYMBOL(high_nybble)
        V <- V + S
      end if
      lead_nybble_checked <- true
    else if low_nybble > 0 do
      if i + 1 < LENGTH(S) do
        S <- LOW_SYMBOL(high_nybble)
        V <- V + S
      else do
        S <- HIGH_SYMBOL(high_nybble)
        V <- V + S
      end if
      lead_nybble_checked <- true
    end if
  end for

  return V
end procedure
```

## Characteristics

While this could be a generic variable length byte array codec, the merits was weighed heavily on creating a short, sortable, multi-part integers.

* Similar to a big endian hexadecimal string with following differences:
  * Byte array can be marked as separate sequential byte arrays.
  * Leading zeros are always truncated.
* Given a set of encoded strings with same section counts, the set is lexicographically sortable.
* Only digits and roman alphabets is used, with similar looking alphabets (digit zero `0` and letter `O`, or digit one `1` and letters `I` and `L`) mapped to the same value.
* Length of encoded string is the sum of zero truncated 4-bits of each byte array of each sections.
  * e.g. `[[0xf1],[0x00, 0xf2],[0x10,0x03],[0x04]]` (length: 6) => `'z1'`,`'z2'`,`'hgg3'`,`'4'` => `'z1z2hgg34'` (length: 9)
* Minimum length of encoded string is count of sections.
* Maximum length of encoded string is two times sum of all byte array length in all sections.

### Advantages

* Compact encoding for unsigned integers
* Support multiple byte arrays
* URL safe
* Lexicographically sortable in normal alphabetical collation
* High human readability
* Case insensitive

### Disadvantages

* Total length is same as hexadecimal at maximum
* Length of byte array in each section is not preserved
  * e.g. `[0x00, 0x00, 0x00]` and `[0x00]` both encodes to the same string `0`

## Examples

|Description|Bytes|Encoded|
|:--|:--|:--|
|Empty|`[]`|`''`|
|Single byte array|`[0x00]`|`'0'`|
||`[0x01]`|`'1'`|
||`[0x0f]`|`'f'`|
||`[0x10]`|`'h0'`|
||`[0xff]`|`'zf'`|
||`[0x00,0x00]`|`'0'`|
||`[0x00,0x01]`|`'1'`|
||`[0x01,0x01]`|`'hg1'`|
||`[0x01,0xff]`|`'hzf'`|
||`[0x0f,0xff]`|`'zzf'`|
||`[0xff,0xff]`|`'zzzf'`|
||`[0xff,0x00,0xff]`|`'zzggzf'`|
|Multiple byte arrays|`[0x01],[0x02],[0x03]`|`'123'`|
||`[0x01],[0x02],[0xff]`|`'12zf'`|
||`[0x01,0x02,0x03],[0x04,0x05,0x06],[0x07,0x08,0x09]`|`'hgjg3mgng6qgrg9'`|

## Encoded length

Given an array of array of bytes `[[b₁, b₂, ..., bₙ₁], [b₁, b₂, ..., bₙ₂], ..., , [b₁, b₂, ..., bₙᵢ]]`, the encoded length is twice the sum of byte length of the arrays (`2 × (n1+n2+...+ni)`) at maximum (same as hexadecimal encoding), and the sum of byte length of the arrays (`n1+n2+...+ni`) at minimum.

## Symbols

Inspired by [Crockford's Base32](https://www.crockford.com/base32.html), split in half with first 16 symbols for lower nybble and next 16 symbols as higher nybble, with lower case as encode symbol. A valid encoded string do not contain leading high nybble value of `0` (`g` or `G`) in any section, is always in lowercase, but decoding supports upper case. Any symbols outside of the decode symbols are considered invalid.

### Lower nybble

|Nybble Value|Decode Symbol|Encode Symbol|
|--:|--:|--:|
|0|0<br/>o<br/>O|0|
|1|1<br/>i<br/>I<br/>l<br/>L|1|
|2|2|2|
|3|3|3|
|4|4|4|
|5|5|5|
|6|6|6|
|7|7|7|
|8|8|8|
|9|9|9|
|10|a<br/>A|a|
|11|b<br/>B|b|
|12|c<br/>C|c|
|13|d<br/>D|d|
|14|e<br/>E|e|
|15|f<br/>F|f|

### Higher nybble

|Symbol|Decode Symbol|Encode Symbol /<br/>Decode Strict Symbol|
|--:|--:|--:|
|0|g<br/>G|g|
|1|h<br/>H|h|
|2|j<br/>J|j|
|3|k<br/>K|k|
|4|m<br/>M|m|
|5|n<br/>N|n|
|6|p<br/>P|p|
|7|q<br/>Q|q|
|8|r<br/>R|r|
|9|s<br/>S|s|
|10|t<br/>T|t|
|11|v<br/>V|v|
|12|w<br/>W|w|
|13|x<br/>X|x|
|14|y<br/>Y|y|
|15|z<br/>Z|z|

## Prior Art

* [ULID](https://github.com/ulid/spec)
* [Crockford Base32](https://www.crockford.com/base32.html)

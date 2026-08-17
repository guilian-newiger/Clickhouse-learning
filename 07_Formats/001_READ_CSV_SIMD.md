## Question
When a user loads a csv file into a table, does ClickHouse use SIMD to parse the csv file?
## Hypothesis
Yes, ClickHouse uses SIMD to find all occurences of the delimiter and the row terminator.

The raw bytes between the delimiters and row terminators then don't require more parsing and are memcopied into one buffer per Column, so that all values of one Column are sequential in memory (as opposed to the row based storage in the csv file). 
## Example
Current commit: cfc1fd2512eb276fe020907434207a08fbaa5e0b

`cat ~/data/07_001.csv`
```
a;123;xyz
b;456;xyz
c;789;xyz
d;;
```
In the hex representation (vim :%!xxd command) we can see that the row terminator is LF (0a):
```
00000000: 613b 3132 333b 7879 7a0a 623b 3435 363b  a;123;xyz.b;456;
00000010: 7879 7a0a 633b 3738 393b 7879 7a0a 643b  xyz.c;789;xyz.d;
00000020: 3b0a                                     ;.
```
Notice that the data is larger than 32 Bytes, because I want to see more than one loop iteration of `_mm256_cmpeq` (I have no AVX-512 support, so only 256 bit, my CPU is AMD)

Create a table to load the csv data into:
```sql
CREATE TABLE t1 (
    foo String,
    bar UInt16,
    baz LowCardinality(String)
) ENGINE = Memory;
```
Our CSV file has semicolon as delimiter, so set it:
```
set format_csv_delimiter = ';'
```

Load CSV into the table:
```sql
INSERT INTO t1
FROM INFILE '/home/xxx/data/07_001.csv'
FORMAT CSV;
```
The three datatypes are going to have three different deserialization methods. Let's look at one after another:
```
(gdb) b deserializeTextCSV
Breakpoint 1 at 0x11492da8: deserializeTextCSV. (47 locations)
```

### SerializationString
First hit is as expected:

`DB::SerializationString::deserializeTextCSV
    at /home/xxx/src/ClickHouse/src/DataTypes/Serializations/SerializationString.cpp:598`

When I step through the code, I end up in `DB::readCSVStringInto<DB::PODArray<char8_t, 4096ul, Allocator<false, false>, 63ul, 64ul>, false, true> (s=..., buf=..., settings=...) at /home/xxx/src/ClickHouse/src/IO/ReadHelpers.cpp:1110`

```
(gdb) bt
#0  DB::readCSVStringInto<DB::PODArray<char8_t, 4096ul, Allocator<false, false>, 63ul, 64ul>, false, true> (s=..., buf=..., settings=...) at /home/xxx/src/ClickHouse/src/IO/ReadHelpers.cpp:1110
#1  0x0000555566ba00eb in DB::SerializationString::deserializeTextCSV(DB::IColumn&, DB::ReadBuffer&, DB::FormatSettings const&) const::$_0::operator()(DB::PODArray<char8_t, 4096ul, Allocator<false, false>, 63ul, 64ul>&) const (this=0x7fffaaff31f0, data=...) at /home/xxx/src/ClickHouse/src/DataTypes/Serializations/SerializationString.cpp:598
#2  0x0000555566b9e7c1 in DB::read<void, DB::SerializationString::deserializeTextCSV(DB::IColumn&, DB::ReadBuffer&, DB::FormatSettings const&) const::$_0>(DB::IColumn&, DB::SerializationString::deserializeTextCSV(DB::IColumn&, DB::ReadBuffer&, DB::FormatSettings const&) const::$_0&&) (column=..., reader=...) at /home/xxx/src/ClickHouse/src/DataTypes/Serializations/SerializationString.cpp:421
#3  0x0000555566b9e735 in DB::SerializationString::deserializeTextCSV (this=0x7fffe001a420, column=..., istr=..., settings=...)
    at /home/xxx/src/ClickHouse/src/DataTypes/Serializations/SerializationString.cpp:598
...
```
Verify that we are at the start of the first row of the CSV file:
```
(gdb) print next_pos
$2 = 0x7fffdc003f80 "a;123;xyz\nb;456;xyz\nc;789;xyz\nd;;\n"
```

Here, SIMD is used to find the next delimiter or line terminator:
```c++
auto rc = _mm_set1_epi8('\r');
auto nc = _mm_set1_epi8('\n');
auto dc = _mm_set1_epi8(delimiter);
for (; next_pos + 15 < buf.buffer().end(); next_pos += 16)
{
    __m128i bytes = _mm_loadu_si128(reinterpret_cast<const __m128i *>(next_pos));
    auto eq = _mm_or_si128(_mm_or_si128(_mm_cmpeq_epi8(bytes, rc), _mm_cmpeq_epi8(bytes, nc)), _mm_cmpeq_epi8(bytes, dc));
    uint16_t bit_mask = static_cast<uint16_t>(_mm_movemask_epi8(eq));
    if (bit_mask)
    {
        next_pos += std::countr_zero(bit_mask);
        return;
    }
}
```
Now let's step through the operations step by step.
```
(gdb) p *(unsigned char *)&bytes@16
$69 = "a;123;xyz\nb;456;"
(gdb) p/x *(unsigned char *)&bytes@16
$78 = {0x61, 0x3b, 0x31, 0x32, 0x33, 0x3b, 0x78, 0x79, 0x7a, 0xa, 0x62, 0x3b, 0x34, 0x35, 0x36, 0x3b}

(gdb) p *(unsigned char *)&rc@16
$66 = '\r' <repeats 16 times>
(gdb) p/x *(unsigned char *)&rc@16
$76 = {0xd <repeats 16 times>}

(gdb) p *(unsigned char *)&nc@16
$68 = '\n' <repeats 16 times>
(gdb) p/x *(unsigned char *)&nc@16
$75 = {0xa <repeats 16 times>}

(gdb) p *(unsigned char *)&dc@16
$77 = ';' <repeats 16 times>
(gdb) p/x *(unsigned char *)&dc@16
$74 = {0x3b <repeats 16 times>}
```
Make a small change in the source code for easier debugging:
```c++
auto bytes_cmp_rc = _mm_cmpeq_epi8(bytes, rc);
auto bytes_cmp_nc = _mm_cmpeq_epi8(bytes, nc);
auto bytes_cmp_dc = _mm_cmpeq_epi8(bytes, dc);
auto bytes_rc_or_nc = _mm_or_si128(bytes_cmp_rc, bytes_cmp_nc);
auto eq = _mm_or_si128(bytes_rc_or_nc, bytes_cmp_dc);
...
```
Rebuild: `~/src/ClickHouse/build$ ninja clickhouse`

Check values of the newly introduced variables:
```
(gdb) p/x *(unsigned char *)&bytes_cmp_rc@16
$3 = {0x0 <repeats 16 times>}
(gdb) p/x *(unsigned char *)&bytes_cmp_nc@16
$4 = {0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0xff, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0}
(gdb) p/x *(unsigned char *)&bytes_rc_or_nc@16
$5 = {0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0xff, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0}
(gdb) p/x *(unsigned char *)&bytes_cmp_dc@16
$6 = {0x0, 0xff, 0x0, 0x0, 0x0, 0xff, 0x0, 0x0, 0x0, 0x0, 0x0, 0xff, 0x0, 0x0, 0x0, 0xff}
(gdb) p/x *(unsigned char *)&eq@16
$7 = {0x0, 0xff, 0x0, 0x0, 0x0, 0xff, 0x0, 0x0, 0x0, 0xff, 0x0, 0xff, 0x0, 0x0, 0x0, 0xff}
```

`_mm_cmpeq_epi8(bytes, rc)` compares `a;123;xyz\nb;456;` with `\r\r\r\r\r\r\r\r\r\r\r\r\r\r\r\r` byte by byte and sets `ff` where it hit and `00` where it didn't. Since our file has LF line endings, it hits nowhere:
```
613b3132333b78797a0a623b3435363b
0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d
---------------------------------
00000000000000000000000000000000
```
For `\n` (`_mm_cmpeq_epi8(bytes, nc)`):
```
613b3132333b78797a0a623b3435363b
0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a
---------------------------------
000000000000000000ff000000000000
```
Now they are OR'd (`_mm_or_si128(bytes_cmp_rc, bytes_cmp_nc)`):
```
00000000000000000000000000000000
000000000000000000ff000000000000
---------------------------------
000000000000000000ff000000000000
```
Same with the delimiter (`_mm_cmpeq_epi8(bytes, dc)`):
```
613b3132333b78797a0a623b3435363b
3b3b3b3b3b3b3b3b3b3b3b3b3b3b3b3b
---------------------------------
00ff000000ff0000000000ff000000ff
```
OR it with the previous result (`_mm_or_si128(bytes_rc_or_nc, bytes_cmp_dc)`):
```
00ff000000ff0000000000ff000000ff
000000000000000000ff000000000000
---------------------------------
00ff000000ff000000ff00ff000000ff
```
Finally, create the `bit_mask` with `_mm_movemask_epi8`:
```
(00ff000000ff000000ff00ff000000ff)_16
  0 1 0 0 0 1 0 0 0 1 0 1 0 0 0 1
---------------------------------
(0100010001010001)_2
```
Let's check whether the bitmask matches the `bytes` variable:
```
a;123;xyz\nb;456;
010001000 1010001
```
It does. The last step is to find the first `1`. Everything from the current position to the first 1 (delimiter or line end) can be copied. We find the first `1` with `std::countr_zero(bit_mask)`.

Why `countr` instead of `countl`? We read the bytes of a string from the lowest address to the highest address and they are printed with the lowest address first. Because Ubuntu is little endian, numbers have their least significant bit (LSB) at the lowest memory address. But you would print a number with the LSB at the right, because that's how we write down numbers in a human readable format. That's why the cpp standard defines the least significant bit (at the lowest memory address) to be "right". So a terminal output usually prints from the lowest memory address (left) to the highest (right), but for numbers, the lowest memory address is defined to be right.

"[std::countr_zero] Returns the number of consecutive 0 bits in the value of x, starting from the least significant bit (“right”)" (https://en.cppreference.com/cpp/numeric/countr_zero, 2026-13-08).

We end up pointing to the next semicolon and everything from the previous beginning to the new pointer is copied to the column (`appendToStringOrVector(s, buf, next_pos);`).
```
(gdb) p *(unsigned char *)next_pos@16
$81 = ";123;xyz\nb;456;x"
```

### SerializationNumber
Next hit is at `DB::SerializationNumber<unsigned short>::deserializeTextCSV
    at /home/xxx/src/ClickHouse/src/DataTypes/Serializations/SerializationNumber.cpp:234
`, because our column `bar` has the datatype `UInt16`.

Here we end up in `DB::readIntTextInBaseImpl<10, unsigned short, void, (DB::ReadIntTextCheckOverflow)0> (x=@0x7fffab7ec1fe: 0, buf=...) at /home/xxx/src/ClickHouse/src/IO/readIntText.h:50`:
```
(gdb) bt
#0  DB::readIntTextInBaseImpl<10, unsigned short, void, (DB::ReadIntTextCheckOverflow)0> (x=@0x7fffab7ec1fe: 0, buf=...) at /home/xxx/src/ClickHouse/src/IO/readIntText.h:50
#1  0x000055556473c45d in DB::readIntTextInBase<10, (DB::ReadIntTextCheckOverflow)0, unsigned short> (x=@0x7fffab7ec1fe: 0, buf=...) at /home/xxx/src/ClickHouse/src/IO/readIntText.h:245
#2  0x000055556473c41d in DB::readIntText<(DB::ReadIntTextCheckOverflow)0, unsigned short> (x=@0x7fffab7ec1fe: 0, buf=...) at /home/xxx/src/ClickHouse/src/IO/readIntText.h:251
#3  0x000055556473c3dd in readText<unsigned short> (x=@0x7fffab7ec1fe: 0, buf=...) at /home/xxx/src/ClickHouse/src/IO/ReadHelpers.h:1646
#4  0x0000555566affd69 in DB::readCSVSimple<unsigned short, void> (x=@0x7fffab7ec1fe: 0, buf=...) at /home/xxx/src/ClickHouse/src/IO/ReadHelpers.h:1779
#5  0x0000555566ae399d in DB::readCSV<unsigned short>(unsigned short&, DB::ReadBuffer&) requires is_arithmetic_v<unsigned short> (x=@0x7fffab7ec1fe: 0, buf=...)
    at /home/xxx/src/ClickHouse/src/IO/ReadHelpers.h:1832
#6  0x0000555566ae3925 in DB::SerializationNumber<unsigned short>::deserializeTextCSV (this=0x7fffe401a700, column=..., istr=..., settings=...)
    at /home/xxx/src/ClickHouse/src/DataTypes/Serializations/SerializationNumber.cpp:234
```

`DB::SerializationLowCardinality::deserializeTextCSV
    at /home/xxx/src/ClickHouse/src/DataTypes/Serializations/SerializationLowCardinality.cpp:884
`

The parsing of numeric datatypes don't seem to use SIMD. The function `ReturnType readIntTextInBaseImpl(T & x, ReadBuffer & buf)` in `src/IO/ReadHelpers.h` loops over each character and checks whether it's a valid value for the given base and if the sign is formatted correctly:
```
    for (; !buf.eof(); ++buf.position())
    {
        char c = *buf.position();
        ...
```

Verify that the first character is '1' from '123' in the first row of our CSV-file:
```
(gdb) p c
$8 = 49 '1'
```
Next, there is a switch statement being evaluated for every character. It looks whether all digits are valid and if the sign is correct and if there are overflows.

### SerializationLowCardinality
LowCardinality(T) too uses SIMD for it's String parsing when it's LowCarindality(String), since it uses the parser of it's generic type T. The only difference is that the value is used to construct a dictionary.
```
(gdb) bt
#0  DB::read<void, DB::SerializationString::deserializeTextCSV(DB::IColumn&, DB::ReadBuffer&, DB::FormatSettings const&) const::$_0>(DB::IColumn&, DB::SerializationString::deserializeTextCSV(DB::IColumn&, DB::ReadBuffer&, DB::FormatSettings const&) const::$_0&&) (column=..., reader=...) at /home/xxx/src/ClickHouse/src/DataTypes/Serializations/SerializationString.cpp:421
#1  0x0000555566b9e735 in DB::SerializationString::deserializeTextCSV (this=0x7fffdc01a420, column=..., istr=..., settings=...)
    at /home/xxx/src/ClickHouse/src/DataTypes/Serializations/SerializationString.cpp:598
#2  0x0000555566a795b3 in DB::SerializationLowCardinality::deserializeImpl<DB::ReadBuffer&, DB::FormatSettings const&, DB::ReadBuffer&, DB::FormatSettings const&> (this=0x7fffdc01a520, column=..., 
    func=&virtual table offset 184, args=..., args=...) at /home/xxx/src/ClickHouse/src/DataTypes/Serializations/SerializationLowCardinality.cpp:948
#3  0x0000555566a79cc5 in DB::SerializationLowCardinality::deserializeTextCSV (this=0x7fffdc01a520, column=..., istr=..., settings=...)
    at /home/xxx/src/ClickHouse/src/DataTypes/Serializations/SerializationLowCardinality.cpp:884
...
```
From `DB::SerializationString::deserializeTextCSV (this=0x7fffdc01a420, column=..., istr=..., settings=...)
    at /home/xxx/src/ClickHouse/src/DataTypes/Serializations/SerializationString.cpp:598` it's the same as for the `String` `DataType`.

## Conclusion
After stepping through the program with gdb, it became clear that the usage of SIMD for parsing differ by `Deserializers` for `DataTypes`. While `String` and `LowCardinality(String)` use SIMD to find delimiters and line endings, the deserializer for UInt16 doesn't.
It appears that the usage of SIMD depends on how heavy the rules are for parsing a `DataType`. For example, `UInt16` has to be checked for sign, valid digits and overflow, whereas Strings can be copied in one go.
ClickHouse uses SIMD to parse csv files only for certain `DataTypes`, which don't have to do too heavy actual parsing like determining the values of a number.
### Open questions
1. Why use 128 bit registers for SIMD and not 256 bit?
2. Can the SIMD comparison results be reused for the next field if it looked far enough ahead?
3. Is multithreading used for reading and parsing a larger csv file?
4. How is it determined which serializer to use for which `DataType`? Since there is a `SerializationNumber`, the `DataTypes` can't be mapped 1-to-1.
5. Does the switch statement used for parsing an integer have to do n comparisons (n being the number of distinct relevant characterslike 0-9, A-F, +/-, ...)? 
6. How does the construction of a dictionary work?
## Question
When a user loads a csv file into a table, does ClickHouse use SIMD to parse the csv file?
## Hypothesis
Yes, ClickHouse uses SIMD to find all occurences of the delimiter and the row terminator.

The raw bytes between the delimiters and row terminators then don't require more parsing and are mem-copied into one buffer per Column, so that all values of one Column are sequential in memory (as opposed to the row based storage in the csv file). 
## Example
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
00000020: 3b0a
```
Notice that the data is larger than 32 Bytes, because I want to see more than one loop iteration of `_mm256_cmpeq` (I have no AVX-512 support, so only 256 bit, my CPU is AMD)


## Conclusion
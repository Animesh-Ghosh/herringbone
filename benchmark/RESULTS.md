# Benchmark results

"Peak RSS growth" is sampled every 20ms in a forked child, relative to the child's baseline.
parquet-ruby writes one row group per file here (parquet-rs default); herringbone uses 100k-row groups.

## Apple Silicon (arm64-darwin24)

Ruby 3.4.1 without YJIT, 2026-09-30.
Dataset: `dataset.rb`, 15 Rails-style columns (ids, 3 low-cardinality enum/string columns, decimal,
nullable double/strings, int32, boolean, 2 timestamps, nullable date).

### herringbone vs parquet-ruby 0.9.0 (1M rows) — `compare_parquet_ruby.rb`

```
herringbone 0.1.0, parquet-ruby 0.9.0, ruby 3.4.1 (2024-12-25 revision 48d4efcb85) +PRISM [arm64-darwin24]

== Write (snappy)
herringbone Writer (hash rows)                             23.01s      43k rows/s      291 MB peak RSS growth  44.2 MB
herringbone Writer (hash rows, uncompressed)               10.93s      92k rows/s      296 MB peak RSS growth  94.4 MB
herringbone Writer (hash rows, zstd via zstd-ruby)         11.23s      89k rows/s      317 MB peak RSS growth  25.7 MB
parquet-ruby write_rows (array rows)                     4.42s     226k rows/s      373 MB peak RSS growth  45.2 MB

== Read all rows
herringbone each_row, herringbone file                         9.71s     103k rows/s      249 MB peak RSS growth  1000000 rows
parquet-ruby each_row, herringbone file                     5.25s     191k rows/s      255 MB peak RSS growth  1000000 rows
herringbone each_row, parquet-ruby file                    14.36s      70k rows/s      759 MB peak RSS growth  1000000 rows
parquet-ruby each_row, parquet-ruby file                 5.34s     187k rows/s      268 MB peak RSS growth  1000000 rows

== Read one column (amount)
herringbone column, herringbone file                           2.50s     400k rows/s      246 MB peak RSS growth  sum=49970731364.8
parquet-ruby each_column, herringbone file                  1.60s     627k rows/s      197 MB peak RSS growth  sum=49970731364.8
herringbone column, parquet-ruby file                       2.42s     413k rows/s      288 MB peak RSS growth  sum=49970731364.8
parquet-ruby each_column, parquet-ruby file              1.52s     656k rows/s      200 MB peak RSS growth  sum=49970731364.8

```

### 20M-row streaming export — `rails_export.rb`

Records are produced `find_each`-style (batches of 1000 attribute Hashes) and streamed into one
file with 100k-row groups, snappy. Live Ruby heap stays flat (~72k slots after each flush); RSS
plateaus in the 450–490 MB range (malloc retaining freed page buffers), independent of row count.

```
Exporting 20000000 rows to /private/tmp/claude-501/-Users-julik-Code-libs-herringbone/2d9676f4-e126-46c5-ab82-0da5b4975abe/scratchpad/export20m.parquet (row groups of 100000, snappy)
        rows   elapsed     rows/s    RSS MB   file MB     GC runs
     1000000     18.7s      53361       300      44.2         302
     2000000     37.6s      53201       302      88.4         515
     3000000     56.1s      53469       384     132.5         681
     4000000     74.5s      53657       424     176.7         852
     5000000     93.4s      53537       451     220.9        1007
     6000000    112.1s      53543       445     265.1        1150
     7000000    131.3s      53325       285     309.3        1308
     8000000    149.8s      53407       365     353.5        1470
     9000000    168.5s      53426       423     397.6        1633
    10000000    187.0s      53464       444     441.8        1795
    11000000    205.7s      53473       459     486.0        1958
    12000000    224.2s      53512       457     530.2        2106
    13000000    242.8s      53538       464     574.4        2254
    14000000    261.4s      53547       472     618.6        2397
    15000000    280.2s      53523       468     662.7        2537
    16000000    299.0s      53515       473     706.9        2678
    17000000    317.8s      53497       474     751.1        2816
    18000000    336.9s      53423       297     795.3        2954
    19000000    355.4s      53462       486     839.5        3090
    20000000    373.9s      53485       485     883.7        3227
Done in 374.0s (53480 rows/s), file 883.9 MB, peak sampled RSS 486 MB
Verified 200 row groups, 20000000 rows; sampled rows match the source
```

### 20M-row export with the current defaults (zstd, row_group_bytes: 16MB)

```
Exporting 20000000 rows to /private/tmp/claude-501/-Users-julik-Code-libs-herringbone/2d9676f4-e126-46c5-ab82-0da5b4975abe/scratchpad/export20m.parquet (default options)
        rows   elapsed     rows/s    RSS MB     GC runs
     1000000      9.8s     102247       283         318
     2000000     19.8s     101018       323         493
     3000000     29.7s     101152       350         654
     4000000     39.0s     102560       382         796
     5000000     48.8s     102459       396         941
     6000000     58.7s     102140       397        1077
     7000000     68.1s     102749       292        1193
     8000000     77.9s     102694       345        1308
     9000000     87.7s     102595       363        1422
    10000000     97.6s     102495       393        1536
    11000000    106.9s     102931       409        1655
    12000000    116.7s     102832       409        1769
    13000000    126.4s     102829       416        1877
    14000000    135.8s     103095       424        1986
    15000000    145.5s     103077       418        2098
    16000000    155.2s     103082       427        2210
    17000000    164.5s     103352       303        2320
    18000000    173.9s     103483       405        2428
    19000000    183.8s     103356       442        2540
    20000000    193.5s     103338       389        2651
Done in 193.6s (103307 rows/s), file 512.3 MB, peak sampled RSS 442 MB
Row groups hold 3728..115704 rows
Verified 175 row groups, 20000000 rows; sampled rows match the source
```

## Hetzner CCX23 (x86_64-linux)

Herringbone 0.6.1 (`13e212dff1139165664380987698890c506ee1e7`), Ruby 3.4.1 without YJIT,
4 dedicated vCPUs, 16 GB RAM, 2026-10-05.
Dataset: `dataset.rb`, 15 Rails-style columns (ids, 3 low-cardinality enum/string columns, decimal,
nullable double/strings, int32, boolean, 2 timestamps, nullable date).

### herringbone vs parquet-ruby 0.9.0 (1M rows)

Each OS had one warm-up and three recorded runs. The output below is from the second recorded run.

#### Debian 13

```
herringbone 0.6.1, parquet-ruby 0.9.0, ruby 3.4.1 (2024-12-25 revision 48d4efcb85) +PRISM [x86_64-linux]

== Write (snappy)
herringbone Writer (hash rows)                          28.82s      35k rows/s      161 MB peak RSS growth  44.4 MB
herringbone Writer (hash rows, uncompressed)            17.28s      58k rows/s      160 MB peak RSS growth  94.4 MB
herringbone Writer (hash rows, zstd via zstd-ruby)      17.51s      57k rows/s      178 MB peak RSS growth  26.3 MB
parquet-ruby write_rows (array rows)                     5.28s     189k rows/s      347 MB peak RSS growth  45.2 MB

== Read all rows
herringbone each_row, herringbone file                   9.79s     102k rows/s      236 MB peak RSS growth  1000000 rows
parquet-ruby each_row, herringbone file                  6.71s     149k rows/s      225 MB peak RSS growth  1000000 rows
herringbone each_row, parquet-ruby file                 10.16s      98k rows/s      284 MB peak RSS growth  1000000 rows
parquet-ruby each_row, parquet-ruby file                 6.69s     150k rows/s      233 MB peak RSS growth  1000000 rows

== Read one column (amount)
herringbone column, herringbone file                     1.32s     756k rows/s      168 MB peak RSS growth  sum=49970731364.8
parquet-ruby each_column, herringbone file               1.29s     775k rows/s      196 MB peak RSS growth  sum=49970731364.8
herringbone column, parquet-ruby file                    1.39s     720k rows/s      163 MB peak RSS growth  sum=49970731364.8
parquet-ruby each_column, parquet-ruby file              1.27s     787k rows/s      192 MB peak RSS growth  sum=49970731364.8
```

#### Ubuntu 26.04

```
herringbone 0.6.1, parquet-ruby 0.9.0, ruby 3.4.1 (2024-12-25 revision 48d4efcb85) +PRISM [x86_64-linux]

== Write (snappy)
herringbone Writer (hash rows)                          36.23s      28k rows/s      162 MB peak RSS growth  44.4 MB
herringbone Writer (hash rows, uncompressed)            21.67s      46k rows/s      159 MB peak RSS growth  94.4 MB
herringbone Writer (hash rows, zstd via zstd-ruby)      21.85s      46k rows/s      178 MB peak RSS growth  26.3 MB
parquet-ruby write_rows (array rows)                     6.98s     143k rows/s      345 MB peak RSS growth  45.2 MB

== Read all rows
herringbone each_row, herringbone file                  11.46s      87k rows/s      237 MB peak RSS growth  1000000 rows
parquet-ruby each_row, herringbone file                  8.45s     118k rows/s      225 MB peak RSS growth  1000000 rows
herringbone each_row, parquet-ruby file                 11.84s      84k rows/s      283 MB peak RSS growth  1000000 rows
parquet-ruby each_row, parquet-ruby file                 8.48s     118k rows/s      234 MB peak RSS growth  1000000 rows

== Read one column (amount)
herringbone column, herringbone file                     1.49s     671k rows/s      202 MB peak RSS growth  sum=49970731364.8
parquet-ruby each_column, herringbone file               1.58s     632k rows/s      195 MB peak RSS growth  sum=49970731364.8
herringbone column, parquet-ruby file                    1.55s     646k rows/s      158 MB peak RSS growth  sum=49970731364.8
parquet-ruby each_column, parquet-ruby file              1.53s     655k rows/s      191 MB peak RSS growth  sum=49970731364.8
```

### 20M-row export — `rails_export.rb`

#### Debian 13

```
Exporting 20000000 rows to /tmp/rails_export.parquet (default options)
        rows   elapsed     rows/s    RSS MB     GC runs
     1000000     20.4s      48900       117         485
     2000000     40.0s      50012       120         791
     3000000     60.7s      49392       138        1041
     4000000     81.6s      49040       166        1293
     5000000    101.1s      49466       158        1536
     6000000    121.9s      49222       177        1784
     7000000    141.4s      49498       186        2026
     8000000    162.3s      49304       190        2280
     9000000    183.2s      49117       201        2534
    10000000    203.0s      49259       217        2786
    11000000    224.1s      49084       231        3048
    12000000    245.2s      48947       245        3310
    13000000    264.9s      49075       258        3566
    14000000    285.9s      48965       264        3827
    15000000    307.0s      48864       287        4096
    16000000    326.7s      48970       295        4370
    17000000    347.9s      48864       308        4655
    18000000    369.0s      48775       324        4940
    19000000    389.0s      48849       328        5216
    20000000    410.2s      48758       338        5503
Done in 411.0s (48666 rows/s), file 887.2 MB, peak sampled RSS 338 MB
Row groups hold 53536..103563 rows
Verified 194 row groups, 20000000 rows; sampled rows match the source
```

#### Ubuntu 26.04

```
Exporting 20000000 rows to /tmp/rails_export.parquet (default options)
        rows   elapsed     rows/s    RSS MB     GC runs
     1000000     27.4s      36498       118         556
     2000000     53.6s      37334       107         936
     3000000     81.8s      36652       134        1255
     4000000    109.4s      36567       159        1588
     5000000    135.7s      36843       152        1919
     6000000    163.5s      36699       164        2275
     7000000    189.6s      36917       193        2598
     8000000    217.3s      36820       185        2923
     9000000    245.1s      36720       204        3259
    10000000    271.1s      36884       212        3601
    11000000    298.7s      36829       225        3959
    12000000    326.6s      36746       238        4315
    13000000    352.6s      36868       250        4677
    14000000    380.5s      36798       253        5058
    15000000    408.5s      36721       268        5458
    16000000    434.6s      36816       282        5850
    17000000    462.6s      36747       289        6248
    18000000    490.6s      36692       305        6654
    19000000    516.7s      36771       317        7054
    20000000    544.8s      36708       321        7462
Done in 545.8s (36643 rows/s), file 887.2 MB, peak sampled RSS 321 MB
Row groups hold 53536..103563 rows
Verified 194 row groups, 20000000 rows; sampled rows match the source
```

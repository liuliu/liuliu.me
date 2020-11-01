---
date: '2020-10-09 18:47:00'
layout: post
slug: loading-csv-file-at-the-speed-limit-of-the-nvme-storage
status: publish
title: Loading CSV File at the Speed Limit of the NVMe Storage
categories:
- eyes
---

*I plan to write a series of articles to discuss some simple but not embarrassingly parallel algorithms. These will have practical usages and would most likely be on many-core CPUs or CUDA GPUs. Today's is the first one to discuss a parallel algorithm implementation for CSV file parser.*

In the old days, when our spin disk speed maxed out at 100MiB/s, we only have two choices: either we don't care about the file loading time at all, treating it as a cost of life, or we have a file format entangled with the underlying memory representation to squeeze out the last bits of performance for data loading.

That world has long gone. My current workstation uses a software RAID0 (mdadm) over two 1TB Samsung 970 EVO NVMe storage for data storage. This setup usually gives me around 2GiB/s read / write speed (you can read more [about my workstation here](https://www.youtube.com/watch?v=OF3JYEIsjH8)).

The CSV file format is firmly in the former category of the two. The thing that people who exchange CSV files care most, above anything else, is the interoperability. Serious people who actually care about speed and efficiency moved to other formats such as [Apache Parquet](https://parquet.apache.org/) or [Apache Arrow](https://arrow.apache.org/). But CSV files live on. It is still by far the most common format in [Kaggle](https://www.kaggle.com/docs/datasets) contests.

There exist many implementations for CSV file parsers. Among them, [csv2](https://github.com/p-ranav/csv2) and [Vince's CSV Parser](https://github.com/vincentlaucsb/csv-parser) would be two common implementations. That doesn't account for standard implementations such as [the one from Python](https://docs.python.org/3/library/csv.html).

Most of these implementations shy away from utilizing many-cores. It is a reasonable choice. In many likely scenarios, you would load many small-ish CSV files, and these can be done in parallel at task-level. That is an OK choice until recently, when I have to deal with some many GiBs CSV files. These files can take many seconds to load, even from tmpfs. That indicates a performance bottleneck at CPU parsing time.

The most obvious way to overcome the CPU parsing bottleneck is to fully utilize the 32 cores of Threadripper 3970x. This can be embarrassingly simple if we can reliably breakdown the parsing by rows. Unfortunately, [RFC 4180](https://tools.ietf.org/html/rfc4180.html) prevents us from simply using line breaks as row delimiters. Particularly, when quoted, a cell content can contain line breaks and these will not be recognized as row delimiters.

[Paratext](https://github.com/wiseio/paratext) first implemented a two-pass approach for parallel CSV parsing. Later it is documented in *[Speculative Distributed CSV Data Parsing for Big Data Analytics](https://www.microsoft.com/en-us/research/uploads/prod/2019/04/chunker-sigmod19.pdf)*. The paper discussed, besides the two-pass approach, a more sophisticated speculative approach that is suitable for the higher-latency distributed environment.

In the past few days, I implemented a variant of the two-pass approach that can max out the NVMe storage bandwidth. It is an interesting journey as I didn't write any serious parser in C for a very long time.

### The CSV File Parsing Problem

CSV file represents simple tabular data with rows and columns. Thus, to parse a CSV file, it is meant to divide a text file into cells that can be uniquely identified with row and column index.

In C++, this can be done in zero-copy fashion with `string_view`. In C, every string has to be null-terminated. Thus, you need to either manipulate the original buffer, or copy it over. I elected the latter.

### Memory-Mapped File

To simplify the parser implementation, it is assumed we are given a block of memory that is the content of the CSV file. This can be done in C with:

```c
FILE* file = fopen("file path", "r");
const int fd = fileno(file);
fseek(file, 0, SEEK_END);
const size_t file_size = ftell(file);
fseek(file, 0, SEEK_SET);
void *const data = mmap((caddr_t)0, file_size, PROT_READ, MAP_SHARED, fd, 0);
```

### OpenMP

We are going to use [OpenMP](https://openmp.llvm.org/)'s parallel for-loop to implement the core algorithm. Nowadays, Clang has pretty comprehensive support for OpenMP. But nevertheless, we will only use the very trivial part of what OpenMP provides.

### Find the Right Line Breaks

To parallel parse a CSV file, we first need to break it down into chunks. We can divide the file into 1MiB sequence of bytes as our chunks. Within each chunk, we can start to find the right line breaks.

The double-quote in [RFC 4180](https://tools.ietf.org/html/rfc4180.html) can quote a line break, that makes us find the right line breaks harder. But at the same time, the RFC defines the way to *escape* double-quote by using two double-quote back-to-back. With this, if we count double-quotes from the beginning of a file, we know that a line break is within a quoted cell if we encounter an odd number of double-quotes so far. If we encounter an even number of double-quotes before a line break, we know that is a beginning of a new row.

We can count double-quotes from the beginning of each chunk. However, because we don't know if there are an odd or even number of double-quotes before this chunk, we cannot differentiate whether a line break is the starting point of a new row, or just within a quoted cell. What we do know, though, is that a line break after an odd number of double-quotes within a chunk is the same class of line breaks. We simply don't know at that point which class that is. We can count these two classes separately.

A code excerpt would look like this:

```c
#define CSV_QUOTE_BR(c, n) \
    do { \
        if (c##n == quote) \
            ++quotes; \
        else if (c##n == '\n') { \
            ++count[quotes & 1]; \
            if (starter[quotes & 1] == -1) \
                starter[quotes & 1] = (int)(p - p_start) + n; \
        } \
    } while (0)
    parallel_for(i, aligned_chunks) {
        const uint64_t* pd = (const uint64_t*)(data + i * chunk_size);
        const char* const p_start = (const char*)pd;
        const uint64_t* const pd_end = pd + chunk_size / sizeof(uint64_t);
        int quotes = 0;
        int starter[2] = {-1, -1};
        int count[2] = {0, 0};
        for (; pd < pd_end; pd++)
        {
            // Load 8-bytes at batch.
            const char* const p = (const char*)pd;
            char c0, c1, c2, c3, c4, c5, c6, c7;
            c0 = p[0], c1 = p[1], c2 = p[2], c3 = p[3], c4 = p[4], c5 = p[5], c6 = p[6], c7 = p[7];
            CSV_QUOTE_BR(c, 0);
            CSV_QUOTE_BR(c, 1);
            CSV_QUOTE_BR(c, 2);
            CSV_QUOTE_BR(c, 3);
            CSV_QUOTE_BR(c, 4);
            CSV_QUOTE_BR(c, 5);
            CSV_QUOTE_BR(c, 6);
            CSV_QUOTE_BR(c, 7);
        }
        crlf[i].even = count[0];
        crlf[i].odd = count[1];
        crlf[i].even_starter = starter[0];
        crlf[i].odd_starter = starter[1];
        crlf[i].quotes = quotes;
    } parallel_endfor
```

This is our first pass.

### Columns and Rows

After the first pass, we can sequentially go through each chunk's statistics to calculate how many rows and columns in the given CSV file.

The line breaks in the first chunk after even number of double-quotes would be the number of rows in the first chunk. Because we know the number of double-quotes in the first chunk, we now know what class of line breaks in the second chunk are the start points of a row. The sum of these line breaks would be the number of rows.

For the number of columns, we can go through the first row and count the number of column delimiters outside of double-quotes.

### Wiring the Cell Strings

The second pass will copy the chunks over, null-terminate each cell, escape the double-quotes if possible. We can piggyback our logic on top of the chunks allocated for the first pass. However, unlike the first pass, the parsing logic doesn't start at the very beginning of each chunk. It starts at the first starting point of a row in that chunk and ends at the first starting point of a row in the next chunk.

The second pass turns out to occupy the most of our parsing time, simply because it does most of the string manipulations and copying in this pass.

### More Optimizations

Both the first pass and second pass unrolled into 8-byte batch parsing, rather than per-byte parsing. For the second pass, we did some bit-twiddling to quickly check whether there are delimiters, double-quotes, or line breaks that needed to be processed, or we can simply copy it over.

```c
const uint64_t delim_mask = (uint64_t)0x0101010101010101 * (uint64_t)delim;
const uint64_t delim_v = v ^ delim_mask;
if ((delim_v - (uint64_t)0x0101010101010101) & ((~delim_v) & (uint64_t)0x8080808080808080)) {
    // Has delimiters.
}
```

You can find more discussions about [this kind of bit-twiddling logic here](https://lemire.me/blog/2017/01/20/how-quickly-can-you-remove-spaces-from-a-string/).

### Is it Fast?

The complete implementation is available at [ccv_cnnp_dataframe_csv.c](https://github.com/liuliu/ccv/blob/unstable/lib/nnc/ccv_cnnp_dataframe_csv.c).

The implementation was compared against [csv2](https://github.com/p-ranav/csv2), [Vince's CSV Parser](https://github.com/vincentlaucsb/csv-parser) and [Paratext](https://github.com/wiseio/paratext).

The workstation uses AMD Threadripper 3970x, with 128GiB memory running at 2666MHz. It has 2 Samsung 1TB 970 EVO with mdadm-based RAID0.

For [csv2](https://github.com/p-ranav/csv2), I compiled `csv2/benchmark/main.cpp` with:
```bash
g++ -I../include -O3 -std=c++11 -o main main.cpp
```

For [Vince's CSV Parser](https://github.com/vincentlaucsb/csv-parser), I compiled `csv-parser/programs/csv_bench.cpp` with:
```bash
g++ -I../single_include -O3 -std=c++17 -o csv_bench csv_bench.cpp -lpthread
```

[Paratext](https://github.com/wiseio/paratext) hasn't been actively developed for the past 2 years. I built it after patched `paratext/python/paratext/core.py` by removing the `splitunc` method. The simple benchmark Python script look like this:
```python
import paratext
import sys

dict_frame = paratext.load_raw_csv(sys.argv[1], allow_quoted_newlines=True)
```

I choose the [DOHUI NOH dataset](https://www.kaggle.com/seaa0612/scaled-data), which contains a 16GiB CSV file with 496,782 rows and 3213 columns.

First, to test the raw performance, I moved the downloaded file to `/tmp`, which is mounted as in-memory tmpfs.

| Software | Time |
|:---      |  ---:|
| Paratext | 12.437s |
| Vince's CSV Parser | 37.829s |
| csv2 | 19.221s |
| NNC's Dataframe CSV | 4.093s |

The above performance accounts for the best you can get if file IO is not a concern. With the said 970 EVO RAID0, we can run another round of benchmark against the real disk IO. Note that for this round of benchmark, we need to drop system file cache with: `sudo bash -c "echo 3 > /proc/sys/vm/drop_caches"` before each run.

| Software | Time |
|:---      |  ---:|
| Paratext | 16.747s |
| Vince's CSV Parser | 39.6075s |
| csv2 | 21.035s |
| NNC's Dataframe CSV | 7.895s |

The performance of our parser approaches 2000MiB/s, not bad!

### Is our Implementation Reasonable with Lower Core Count?

[csv2](https://github.com/p-ranav/csv2) is single-threaded. With only one thread, would our implementation still be reasonable? I moved `parallel_for` back to serial for-loop and ran the experiment on tmpfs again.

| Software | Time |
|:---      |  ---:|
| csv2 | 19.221s |
| NNC's Dataframe CSV (Many-core) | 4.093s |
| NNC's Dataframe CSV (Single-core) | 45.391s |

It is about 2x slower than [csv2](https://github.com/p-ranav/csv2). This is expected because we need to null-terminate strings and copy them to a new buffer.

### Finish It Up with Fuzzing

You cannot really ship a parser in C without doing fuzzing. Luckily, in the past few years, it is incredibly easy to write a fuzz program in C. This time, I chose [LLVM's libFuzzer](https://llvm.org/docs/LibFuzzer.html) and turned on [AddressSanitizer](https://clang.llvm.org/docs/AddressSanitizer.html) along the way.

The fuzz program is very concise:

```c
#include <ccv.h>
#include <nnc/ccv_nnc.h>
#include <nnc/ccv_nnc_easy.h>
#include <stdint.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

int LLVMFuzzerInitialize(int* argc, char*** argv)
{
    ccv_nnc_init();
    return 0;
}

int LLVMFuzzerTestOneInput(const uint8_t* data, size_t size)
{
    if (size == 0)
        return 0;
    int column_size = 0;
    ccv_cnnp_dataframe_t* dataframe = ccv_cnnp_dataframe_from_csv_new((void*)data, CCV_CNNP_DATAFRAME_CSV_MEMORY, size, ',', '"', 0, &column_size);
    if (dataframe)
    {
        if (column_size > 0) // Iterate through the first column.
        {
            ccv_cnnp_dataframe_iter_t* const iter = ccv_cnnp_dataframe_iter_new(dataframe, COLUMN_ID_LIST(0));
            const int row_count = ccv_cnnp_dataframe_row_count(dataframe);
            int i;
            size_t total = 0;
            for (i = 0; i < row_count; i++)
            {
                void* data = 0;
                ccv_cnnp_dataframe_iter_next(iter, &data, 1, 0);
                total += strlen(data);
            }
            ccv_cnnp_dataframe_iter_free(iter);
        }
        ccv_cnnp_dataframe_free(dataframe);
    }
    return 0;
}
```

`./csv_fuzz -runs=10000000 -max_len=2097152` took about 2 hours to finish. I fixed a few issues before the final successful run.

### Closing Words

With many-core systems becoming increasingly common, we should expect more programs to use these cores at the level traditionally considered for single core such as parsing. It doesn't necessarily need to be hard either! With good OpenMP support, some simple tuning on algorithm-side, we can easily take advantage of improved hardware to get more things done.

I am excited to share more of my journey into parallel algorithms on modern GPUs next. Stay tuned!

---

#### 2020-10-18 Update

There are some more discussions in [lobste.rs](https://lobste.rs/s/zksa0f/loading_csv_file_at_speed_limit_nvme) and [news.ycombinator](https://news.ycombinator.com/item?id=24736559). After these discussions, I benchmarked [xsv](https://github.com/BurntSushi/xsv) for this particular csv file and implemented zero-copy parsing (effectively pushing more processing to iteration time) [on my side](https://github.com/liuliu/ccv/commit/b1a5cbf708a1e9e0048ff52949d905f251ef0e2c). The zero-copy implementation becomes trickier than I initially liked because expanding from *pointer* to *pointer + length* has memory usage implications and can be slower if implemented naively for the later case.

Here are some results for parsing from NVMe storage:

| Software | Time |
|:---      |  ---:|
| xsv index | 26.613s |
| NNC's Dataframe CSV | 7.895s |
| NNC's Dataframe CSV (Zero-copy) | 7.142s |

If runs on single-core, the parallel parser was penalized by the two-pass approach. Particularly:

| Software | Time |
|:---      |  ---:|
| NNC's Dataframe CSV (Single-core) | 45.391s |
| NNC's Dataframe CSV (Zero-copy, Single-core) | 39.181s |
| NNC's Dataframe CSV (Zero-copy, Single-core, First-pass) | 9.274s |
| NNC's Dataframe CSV (Zero-copy, Single-core, Second-pass) | 32.963s |
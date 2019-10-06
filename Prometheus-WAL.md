# Prometheus WAL源码阅读

---
## 概述
WAL(Write-ahead logging, 预写日志)是关系型数据库中利用日志来实现事务性和持久性的一种技术，即在某个操作之前先将这个事情记录下来，以便之后对数据进行回滚，重试等操作保证数据的可靠性。

Prometheus为了防止丢失暂存在内存中的还未被写入磁盘的监控数据，引入了WAL机制。WAL被分割成默认大小为128M的文件段（segment），之前版本默认大小是256M，文件段以数字命名，长度为8位的整形。WAL的写入单位是页（page），每页的大小为32KB，所以每个段大小必须是页的大小的整数倍。每个文件段都有一个“==已使用的页？==”属性来标识在该段中已经分配的页数目，如果WAL一次性写入的页数超过一个段的空闲页数，就会创建一个新的文件段来保存这些页，从而确保一次性写入的页**不会跨段存储**。
```
const (
     DefaultSegmentSize = 128 * 1024 * 1024 // 128 MB
     pageSize           = 32 * 1024         // 32KB
     recordHeaderSize   = 7
)
```

本文是在[prometheus tsdb部分源码阅读之wal.LogSeries](https://www.jianshu.com/p/e0144be9692a)基础上进行扩展，由于`tsdb/wal.go`文件中定义的WAL接口以及相关方法已经deprecated，如下所示。因此本文针对`tsdb/wal/wal.go`文件进行分析。

```
// WAL is a write ahead log that can log new series labels and samples.
// It must be completely read before new entries are logged.
//
// DEPRECATED: use wal pkg combined with the record codex instead.
type WAL interface {
    Reader() WALReader
    LogSeries([]RefSeries) error
    LogSamples([]RefSample) error
    LogDeletes([]Stone) error
    Truncate(mint int64, keep func(uint64) bool) error
    Close() error
}
```

---
## Prometheus storage hierarchy
```
(ENV) 🍺 /Users/xsky/go/src/github.com/microyahoo/prometheus/data ☞ tree -h
.
├── [ 192]  01DPE8T5XPQ9ZYHSNJYBJBKGR6
│   ├── [  96]  chunks
│   │   └── [7.1K]  000001
│   ├── [ 22K]  index
│   ├── [ 272]  meta.json
│   └── [   9]  tombstones
├── [   0]  lock
├── [ 20K]  queries.active
└── [ 256]  wal
    ├── [   0]  00000050
    ├── [   0]  00000051
    ├── [   0]  00000052
    ├── [   0]  00000053
    ├── [ 10K]  00000054
    └── [  96]  checkpoint.000049
        └── [ 32K]  00000000

4 directories, 12 files
```
从上面目录结构可以看出wal主要包含segment文件，checkpoint目录。
```
// WAL is a write ahead log that stores records in segment files.
// It must be read from start to end once before logging new data.
// If an error occurs during read, the repair procedure must be called
// before it's safe to do further writes.
//
// Segments are written to in pages of 32KB, with records possibly split
// across page boundaries.
// Records are never split across segments to allow full segments to be
// safely truncated. It also ensures that torn writes never corrupt records
// beyond the most recent segment.
type WAL struct {
    dir         string
    logger      log.Logger
    segmentSize int
    mtx         sync.RWMutex
    segment     *Segment // Active segment.
    donePages   int      // Pages written to the segment.
    page        *page    // Active page.
    stopc       chan chan struct{}
    actorc      chan func()
    closed      bool // To allow calling Close() more than once without blocking.
    compress    bool
    snappyBuf   []byte

    fsyncDuration   prometheus.Summary
    pageFlushes     prometheus.Counter
    pageCompletions prometheus.Counter
    truncateFail    prometheus.Counter
    truncateTotal   prometheus.Counter
    currentSegment  prometheus.Gauge
}
```
```
// Segment represents a segment file.
type Segment struct {
    *os.File
    dir string
    i   int
}
```

---
## Create Segment
创建`segment`主要是通过`New`方法或者`NewSize`方法。其中`NewSize`可以指定segment的大小，`New`方法传入的是默认的segment大小，即128M。加载DB时如果WAL enabled的话，则会调用`NewSize`创建新的segment，如下所示。下面对`NewSize`进行展开。
```
wlog, err = wal.NewSize(l, r, filepath.Join(dir, "wal"), segmentSize, opts.WALCompression)
```
- 首先判断段大小是否是`page`的整数倍。
- 如果`wal`目录不存在的话则创建`wal`目录。
- 创建WAL数据结构。因为WAL是与`active segment`关联的，所以WAL `segmentSize`即为指定的segmentSize。
- 查找目前WAL中所有的segment记录。因为segment是按照数字命名并递增的，所以其名字即为索引，其内部表述为`segmentRef`，name为其文件名，即前缀为0填充的八位整形表示，index即为其整形表示。
```
type segmentRef struct {
    name  string
    index int
}
```
- 上面步骤获取到last segment之后，下一个要写入的segment index即为`last+1`，创建新的segment。
- 将新创建的segment设置为WAL的active segment，同时根据segment文件大小设置WAL `donePages`，将WAL `currentSegment`设置为新创建的segment的index。
- 启动新的goroutine执行`actorc`通道发送过来的func。

---
## WAL Repair
> Repair attempts to repair the WAL based on the error.
It discards all data after the corruption.

需要注意的是`Repair`仅能修复`CorruptionErr`。`CorruptionErr`中指定了目录，段，偏移量以及具体的错误信息。
```
// CorruptionErr is an error that's returned when corruption is encountered.
type CorruptionErr struct {
    Dir     string
    Segment int
    Offset  int64
    Err     error
}
```
- 首先获取wal目录中所有的segments。
- **删除**`CorruptionErr.Segment`索引之后的所有的segments。
- 关闭WAL的`active segment`。
- 将`CorruptionErr.Segment`索引的segment加上`.repair`后缀并重命名。
- 创建一个新的segment文件，其index为`CorruptionErr.Segment`，即Corruption segment重命名之后重新创建一个新的segment文件，并将WAL的active segment指向新创建的segment。
- 将Corruption segment中的数据读到新创建的segment文件中，读到`CorruptionErr.Offset`为止。

### WAL Reader
```
// Reader reads WAL records from an io.Reader.
type Reader struct {
    rdr       io.Reader
    err       error
    rec       []byte
    snappyBuf []byte
    buf       [pageSize]byte
    total     int64   // Total bytes processed.
    curRecTyp recType // Used for checking that the last record is not torn.
}
```
其中Reader从rdr中读取records，相当于把io.Reader做了一层包装；buf为大小为pageSize的字节数组。

```
+--------------------------------------------------+
|                     Reader.buf                   |
+-----------------------------------+--------------+
|               hdr                 |     buf      |
+-----------+----------+------------+--------------+
| type <1b> | len <2b> | CRC32 <4b> | data <bytes> |
+-----------+----------+------------+--------------+

```

the record fragment is encoded as:

```
+-----------+----------+------------+--------------+
| type <1b> | len <2b> | CRC32 <4b> | data <bytes> |
+-----------+----------+------------+--------------+
```

The type flag has the following states:

* `0`: rest of page will be empty
* `1`: a full record encoded in a single fragment
* `2`: first fragment of a record
* `3`: middle fragment of a record
* `4`: final fragment of a record

其中type占用一个byte，由于type flag只有五种状态，3 bits就可以表示了。前4个 bit未分配，第5个bit表示是否`压缩`，最后三个bit表示`record type`。
```
+--------------------+-------------------------------+------------------------+
| 4 bits unallocated | 1 bit snappy compression flag |   3 bit record type    |
+--------------------+-------------------------------+------------------------+
```

```
// Next advances the reader to the next records and returns true if it exists.
// It must not be called again after it returned false.
func (r *Reader) Next() bool {
    err := r.next()
    if errors.Cause(err) == io.EOF {
        // The last WAL segment record shouldn't be torn(should be full or last).
        // The last record would be torn after a crash just before
        // the last record part could be persisted to disk.
        if r.curRecTyp == recFirst || r.curRecTyp == recMiddle {
            r.err = errors.New("last record is torn")
        }
        return false
    }
    r.err = err
    return r.err == nil
}

func (r *Reader) next() (err error) {
    // We have to use r.buf since allocating byte arrays here fails escape
    // analysis and ends up on the heap, even though it seemingly should not.
    // 前面7个byte代表header，后面的为data
    hdr := r.buf[:recordHeaderSize]
    buf := r.buf[recordHeaderSize:]

    r.rec = r.rec[:0]
    r.snappyBuf = r.snappyBuf[:0]

    i := 0
    for {
        // 首先从rdr中读取一个字节
        if _, err = io.ReadFull(r.rdr, hdr[:1]); err != nil {
            return errors.Wrap(err, "read first header byte")
        }
        r.total++
        // 获取record type，并判断是否压缩
        r.curRecTyp = recTypeFromHeader(hdr[0])
        compressed := hdr[0]&snappyMask != 0

        // Gobble up zero bytes.
        // 如果record type为recPageTerm，代表后面page剩余数据全部为0填充。即第一个byte为type，后面全是0。
        if r.curRecTyp == recPageTerm {
            // recPageTerm is a single byte that indicates the rest of the page is padded.
            // If it's the first byte in a page, buf is too small and
            // needs to be resized to fit pageSize-1 bytes.
            buf = r.buf[1:]

            // We are pedantic and check whether the zeros are actually up
            // to a page boundary.
            // It's not strictly necessary but may catch sketchy state early.
            k := pageSize - (r.total % pageSize)
            if k == pageSize {
                continue // Initial 0 byte was last page byte.
            }
            n, err := io.ReadFull(r.rdr, buf[:k])
            if err != nil {
                return errors.Wrap(err, "read remaining zeros")
            }
            r.total += int64(n)

            for _, c := range buf[:k] {
                if c != 0 {
                    return errors.New("unexpected non-zero byte in padded page")
                }
            }
            continue
        }
        // 读取剩余的6个byte，分别代表length和crc
        n, err := io.ReadFull(r.rdr, hdr[1:])
        if err != nil {
            return errors.Wrap(err, "read remaining header")
        }
        r.total += int64(n)

        var (
            length = binary.BigEndian.Uint16(hdr[1:])
            crc    = binary.BigEndian.Uint32(hdr[3:])
        )

        if length > pageSize-recordHeaderSize {
            return errors.Errorf("invalid record size %d", length)
        }
        // 读取长度为length的数据，并计算其crc值是否与header中的crc相等。
        n, err = io.ReadFull(r.rdr, buf[:length])
        if err != nil {
            return err
        }
        r.total += int64(n)

        if n != int(length) {
            return errors.Errorf("invalid size: expected %d, got %d", length, n)
        }
        if c := crc32.Checksum(buf[:length], castagnoliTable); c != crc {
            return errors.Errorf("unexpected checksum %x, expected %x", c, crc)
        }
        
        // 如果为压缩数据，则将其存放在snappyBuf中，后面读取完成后进行decode。
        if compressed {
            r.snappyBuf = append(r.snappyBuf, buf[:length]...)
        } else {
            r.rec = append(r.rec, buf[:length]...)
        }

        if err := validateRecord(r.curRecTyp, i); err != nil {
            return err
        }
        if r.curRecTyp == recLast || r.curRecTyp == recFull {
            if compressed && len(r.snappyBuf) > 0 {
                // The snappy library uses `len` to calculate if we need a new buffer.
                // In order to allocate as few buffers as possible make the length
                // equal to the capacity.
                // 将压缩的数据进行解码。
                r.rec = r.rec[:cap(r.rec)]
                r.rec, err = snappy.Decode(r.rec, r.snappyBuf)
                return err
            }
            return nil
        }
        // Only increment i for non-zero records since we use it
        // to determine valid content record sequences.
        i++
    }
}
```
`Next`方法从`rdr`中读取record，如果存在则返回true。每次读取一个`page`大小的数据，直到读取完整的`record`，如果数据被压缩，则需要`decode`，最后的record存储在`Reader.rec`字段中。

#### 问题点
1. 为什么需要`recPageTerm`类型，这样是不是浪费了一整个page大小的空间？
2. `record`代表什么？

---
## References
- [WAL disk format](https://github.com/prometheus/prometheus/blob/master/tsdb/docs/format/wal.md)
- [prometheus tsdb部分源码阅读之wal.LogSeries](https://www.jianshu.com/p/e0144be9692a)
- [详解varint编码原理](https://segmentfault.com/a/1190000020500985?utm_source=tag-newest)
- [RocksDB WAL file format](https://github.com/facebook/rocksdb/wiki/Write-Ahead-Log-File-Format)
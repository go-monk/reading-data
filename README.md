Source: https://github.com/go-monk/reading-data

# os.ReadFile

The most common data source is a local file. The easiest way to read data from a file is using the os.ReadFile function:

```go
data, err := os.ReadFile("/etc/passwd")
if err != nil {
    log.Fatal(err)
}
fmt.Print(string(data))
```

We get back the contents of the file as slice of bytes (`data`) so we need to convert it to string representation to make it human-readable.

# os.Open 

If a file is big enough we might not want to slurp the whole file into memory before doing something with the data (like printing it). To read a file in small chunks, let's say of 10 bytes, you could do something like:

```go
// Open the file for reading and store its 
// file descriptor (or file handle) in f.
f, err := os.Open("/etc/passwd")
if err != nil {
    log.Fatal(err)
}
defer f.Close()

// Keep reading the file data in 10-bytes chunks 
// until we get to the end of file (err == io.EOF).
chunk := make([]byte, 10)
var data []byte
for {
    n, err := f.Read(chunk)
    if err != nil {
        if err == io.EOF {
            break
        }
        log.Fatal(err)
    }
    data = append(data, chunk[:n]...)
}

fmt.Print(string(data))
```

In fact, os.ReadFile does something similar under the hood, you can [have a look](https://cs.opensource.google/go/go/+/refs/tags/go1.24.4:src/os/file.go;l=799).

You can also Seek to a known location in a file and Read from there:

```go
f, err := os.Open("/etc/passwd")
if err != nil {
    log.Fatal(err)
}
defer f.Close()

// Read 4 bytes starting with a byte at position 10 
// from the beginning of the file.
data := make([]byte, 4)
f.Seek(10, io.SeekStart)
f.Read(data)
// NOTE: omitting errors in the two above calls for brevity

fmt.Print(string(data))
```

# bufio.NewScanner

A more common way to read a file by chunks is to use bufio.NewScanner. This way you read a file by "logical" chunks, like by lines:

```go
f, err := os.Open("/etc/passwd")
if err != nil {
    log.Fatal(err)
}
defer f.Close()

scanner := bufio.NewScanner(f)
// scanner.Split(bufio.ScanLines)
for scanner.Scan() {
    fmt.Println(scanner.Text())
}
if err := scanner.Err(); err != nil {
    log.Fatal(err)
}
```

You can customize how the input is split by using the Split method with a custom SplitFunc. SplitFunc defines the chunks you want the data to split into. Except for the default ScanLines, there are three other predefined SplitFuncs: ScanBytes, ScanRunes, and ScanWords.

# io.Reader abstraction

In the above examples we saw various ways to read a local file. But local files are just one type of data source. We also have data coming from terminal or other commands (STDIN) or from remote data sources. And Go abstract all these data sources from which you can read with the io.Reader interface:

```go
type Reader interface {
        Read(p []byte) (n int, err error)
}
```

It means that you can read from any data type that has the Read function (method) that can store data into a slice of bytes (p).

os.Stdin, which represents standard input, is an *os.File that implements the Read method. So we can read from the standard input like from a file or any other io.Reader:

```go
data, err := io.ReadAll(os.Stdin)
```

What about remote data sources? Let's read an HTTP resource:

```go
resp, err := http.Get("https://example.com")
if err != nil {
    log.Fatal(err)
}
defer resp.Body.Close() // always close body to avoid too many open connections

_, err = io.Copy(os.Stdout, resp.Body)
if err != nil {
    log.Fatal(err)
}
```

Here we make a GET request to an URL. If the request was successful, we read data from its body (io.Reader) and write it to STDOUT (io.Writer).

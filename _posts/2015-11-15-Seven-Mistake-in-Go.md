---
layout: post
title: go的7个常见错误
---
1、没有真正理解interface（Not Accepting Interfaces）

Stop Doing This!!
    
```go
func (page *Page) saveSourceAs(path string) { 
    b := new(bytes.Buffer) 
    b.Write(page.Source.Content) 
    page.saveSource(b.Bytes(), path)
}
func (page *Page) saveSource(by []byte, inpath string) { 
    WriteToDisk(inpath, bytes.NewReader(by)) 
} // Stop Doing This!!
```

Instead

```go
func (page *Page) saveSourceAs(path string) { 
    b := new(bytes.Buffer) 
    b.Write(page.Source.Content) 
    page.saveSource(b, path) 
} 
func (page *Page) saveSource(b io.Reader, inpath string) { 
    WriteToDisk(inpath, b) 
} // Instead
```
    
2、没有正确使用Io.Reader & Io.Writer（Not Using Io.Reader & Io.Writer）
Io.Reader & Io.Writer 

1) Simple & ﬂexible interfaces for many operations around input and output

2) Provides access to a huge wealth of functionality 

3)Keeps operations extensible

```go
type Reader interface { 
    Read(p []byte) (n int, err error) 
} 
type Writer interface { 
    Write(p []byte) (n int, err error) 
}
```

Really stop do this!

```go
func (v *Viper) ReadBufConfig(buf *bytes.Buffer) error { 
    v.config = make(map[string]interface{}) 
    v.marshalReader(buf, v.config) return nil 
}
```
Instead

```go
func (v *Viper) ReadConfig(in io.Reader) error { 
    v.config = make(map[string]interface{}) 
    v.marshalReader(in, v.config) 
    return nil 
}
```
3、接口滥用（Requiring Broad Interfaces）

1）Interfaces Are Composable

2）Functions should only accept interfaces that require the methods they need

3）Functions should not accept a broad interface when a narrow one would work

4）Compose broad interfaces made from narrower ones

```go
type File interface { 
    io.Closer 
    io.Reader 
    io.ReaderAt 
    io.Seeker 
    io.Writer 
    io.WriterAt 
} // Composing Interfaces
```

Requiring Broad Interfaces

```go
func ReadIn(f File) { 
    b := []byte{} n, err := f.Read(b) 
    … 
}
```
    
Requiring Narrow Interfaces

```go
func ReadIn(r Reader) { 
    b := []byte{} n, err := r.Read(b) 
    … 
}
```
4、Methods Vs Functions

What Is A Function?

1）Operations performed on N1 inputs that results in N2 outputs 

2）The same inputs will always result in the same outputs

3）Functions should not depend on state

What Is A Method?

1）Deﬁnes the behavior of a type

2）A function that operates against a value

3）Should use state

Functions Can Be Used With Interfaces

Methods, by deﬁnition, are bound to a speciﬁc type 

Functions can accept interfaces as input

```go
func extractShortcodes(s string, p *Page, t Template) (string, map[string]shortcode, error) { 
    … 
    for { 
        switch currItem.typ { 
            … 
            case tError: err := fmt.Errorf(“%s:%d: %s”, p.BaseFileName(), (p.lineNumRawContentStart() + pt.lexer.lineNum() - 1), currItem) 
        } 
    }
    … 
}
func extractShortcodes(s string, p *Page, t Template) (string, map[string]shortcode, error) { 
    … 
    for { 
        switch currItem.typ { 
            … 
            case tError: err := fmt.Errorf(“%s:%d: %s”, p.BaseFileName(), (p.lineNumRawContentStart() + pt.lexer.lineNum() - 1), currItem) 
        } 
    } 
    … 
}
```
5、Pointers Vs Values

1）It’s not a question of performance (generally), but one of shared access

2）If you want to share the value with a function or method, then use a pointer

3）If you don’t want to share it, then use a value (copy)

Pointer Receivers

1）If you want to share a value with it’s method, use a pointer receiver

2）Since methods commonly manage state, this is the common usage

3) Not safe for concurrent access Value Receivers

Value Receivers

1）If you want the value copied(not shared), use values

2）If the type is an empty struct (stateless, just behavior)… then just use value

3）Safe for concurrent access

```go
    type InMemoryFile struct { 
        at int64 
        name string 
        data []byte 
        closed bool 
    } 
    func (f *InMemoryFile) Close() error { 
        atomic.StoreInt64(&f.at, 0) 
        f.closed = true 
        return nil 
    }
    type Time struct { 
        sec int64 
        nsec uintptr 
        loc *Location 
    } 
    func (t Time) IsZero() bool { 
        return t.sec == 0 && t.nsec == 0 
    }
```
6、Thinking Of Errors As Strings

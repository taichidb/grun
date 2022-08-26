# grun

grun.Group is a universal mechanism to manage goroutine lifecycles.

Create a zero-value grun.Group, and then add actors to it. Actors are defined as
a pair of functions: an **execute** function, which should run synchronously;
and an **interrupt** function, which, when invoked, should cause the execute
function to return. Finally, invoke Run, which concurrently runs all of the
actors, waits until the first actor exits, invokes the interrupt functions, and
finally returns control to the caller only once all actors have returned. This
general-purpose API allows callers to model pretty much any runnable task, and
achieve well-defined lifecycle semantics for the group.

## Usage

```go
import (
    "gopkg.in/taichidb/grun.v1"
)

func main() {
    var g grun.Group
    {
        ctx, cancel := context.WithCancel(context.Background())
        g.Add(func() error {
            return myProcess(ctx, ...)
        }, func (error) {
            cancel()
        })
    }

    {
        ln, _ := net.Listen("tcp", ":8080")
        g.Add(func () error {
            return http.Serve(ln, nil)
        }, func (error) {
            ln.Close()
        })
    }

    {
        var conn io.ReadCloser = ...
        g.Add(func () error {
            s := bufio.NewScanner(conn)
            for s.Scan() {
                println(s.Text())
            }
            return s.Err()
        }, func (error) {
            conn.Close()
        })
    }
    
    g.Run()
}
```

## Comparisons

Package grun is somewhat similar to package
[errgroup](https://godoc.org/golang.org/x/sync/errgroup),
except it doesn't require actor goroutines to understand context semantics.

It's somewhat similar to package
[tomb.v1](https://godoc.org/gopkg.in/tomb.v1) or
[tomb.v2](https://godoc.org/gopkg.in/tomb.v2),
except it has a much smaller API surface, delegating e.g. staged shutdown of
goroutines to the caller.

[![Go Report Card](http://goreportcard.com/badge/client9/reopen)](http://goreportcard.com/report/client9/reopen) [![GoDoc](https://godoc.org/github.com/client9/reopen?status.svg)](https://godoc.org/github.com/client9/reopen)

Makes a standard os.File "reopenable writer" and allows SIGHUP signals
to reopen log files, as needed by
[logrotated](https://fedorahosted.org/logrotate/).  This is inspired
by the C/Posix
[freopen](http://pubs.opengroup.org/onlinepubs/009695399/functions/freopen.html)

The simple version `reopen.NewFileWriter` does unbuffered writing.  A
call to `.Reopen` closes the existing file handle, and then re-opens
it using the original filename.

The more advanced version `reopen.NewBufferedFileWrite` buffers input
and flushes when the internal buffer is full (with care) or if 30 seconds has
elapsed.

Samples are in `example1` and `example2`.  The `run.sh` scripts are a
dumb test where the file is rotated underneath the server, and nothing
is lost.  (not the most robust test but gives you an idea)


Here's some sample code.

```go
package main

/* Simple logrotate logger
 */
import (
       "fmt"
       "log"
       "net/http"
       "os"
       "os/signal"
       "syscall"

       "github.com/client9/reopen"
)

func main() {
        // setup logger to write to our new *reopenable* log file

        f, err := goreopen.NewFileWriter("/tmp/example.log")
        if err != nil {
           log.Fatalf("Unable to set output log: %s", err)
        }
        log.SetOutput(f)

         // Handle SIGHUP
        //
        // channel is number of signals needed to catch  (more or less)
        // we only are working with one here, SIGUP
        sighup := make(chan os.Signal, 1)
        signal.Notify(sighup, syscall.SIGHUP)
        go func() {
           for {
               <-sighup
              fmt.Printf("Got a sighup\n")
              f.Reopen()
            }
         }()

        // dumb http server that just prints and logs the path
        http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
            log.Printf("%s", r.URL.Path)
            fmt.Fprintf(w, "%s\n", r.URL.Path)
        })
        log.Fatal(http.ListenAndServe("127.0.0.1:8123", nil))
}
```
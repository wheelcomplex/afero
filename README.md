![afero logo-sm](https://cloud.githubusercontent.com/assets/173412/11490338/d50e16dc-97a5-11e5-8b12-019a300d0fcb.png)

A FileSystem Abstraction System for Go

[![Build Status](https://travis-ci.org/spf13/afero.svg)](https://travis-ci.org/spf13/afero) [![Build status](https://ci.appveyor.com/api/projects/status/github/spf13/afero?branch=master&svg=true)](https://ci.appveyor.com/project/spf13/afero) [![GoDoc](https://godoc.org/github.com/spf13/afero?status.svg)](https://godoc.org/github.com/spf13/afero) [![Join the chat at https://gitter.im/spf13/afero](https://badges.gitter.im/Dev%20Chat.svg)](https://gitter.im/spf13/afero?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge)

# Overview

Afero is an filesystem framework providing a simple, uniform and universal API
interacting with any filesystem, as an abstraction layer providing interfaces,
types and methods. Afero has an exceptionally clean interface and simple design
without needless constructors or initialization methods.

Afero is also a library providing a base set of interoperable backend
filesystems that make it easy to work with afero while retaining all the power
and benefit of the os and ioutil packages.

Afero provides significant improvements over using the os package alone, most
notably the ability to create mock and testing filesystems without relying on the disk.

It is suitable for use in a any situation where you would consider using the OS
package as it provides an additional abstraction that makes it easy to use a
memory backed file system during testing. It also adds support for the http
filesystem for full interoperability.


## Afero Features

* A single consistent API for accessing a variety of filesystems
* Interoperation between a variety of file system types
* A set of interfaces to encourage and enforce interoperability between backends
* An atomic cross platform memory backed file system
* Support for compositional file systems by joining various different file systems (see httpFs)
* Filtering of calls to intercept opening / modifying files, several filters
may be stacked.
* A set of utility functions ported from io, ioutil & hugo to be afero aware


# Using Afero

Afero is easy to use and easier to adopt.

A few different ways you could use Afero:

* Use the interfaces alone to define you own file system.
* Wrap for the OS packages.
* Define different filesystems for different parts of your application.
* Use Afero for mock filesystems while testing

## Step 1: Install Afero

First use go get to install the latest version of the library.

    $ go get github.com/spf13/afero

Next include Afero in your application.
```go
import "github.com/spf13/afero"
```

## Step 2: Declare a backend

First define a package variable and set it to a pointer to a filesystem.
```go
var AppFs afero.Fs = &afero.MemMapFs{}

or

var AppFs afero.Fs = &afero.OsFs{}
```
It is important to note that if you repeat the composite literal you
will be using a completely new and isolated filesystem. In the case of
OsFs it will still use the same underlying filesystem but will reduce
the ability to drop in other filesystems as desired.

## Step 3: Use it like you would the OS package

Throughout your application use any function and method like you normally
would.

So if my application before had:
```go
os.Open('/tmp/foo')
```
We would replace it with a call to `AppFs.Open('/tmp/foo')`.

`AppFs` being the variable we defined above.


## List of all available functions

File System Methods Available:
```go
Chmod(name string, mode os.FileMode) : error
Chtimes(name string, atime time.Time, mtime time.Time) : error
Create(name string) : File, error
Mkdir(name string, perm os.FileMode) : error
MkdirAll(path string, perm os.FileMode) : error
Name() : string
Open(name string) : File, error
OpenFile(name string, flag int, perm os.FileMode) : File, error
Remove(name string) : error
RemoveAll(path string) : error
Rename(oldname, newname string) : error
Stat(name string) : os.FileInfo, error
```
File Interfaces and Methods Available:
```go
io.Closer
io.Reader
io.ReaderAt
io.Seeker
io.Writer
io.WriterAt

Name() : string
Readdir(count int) : []os.FileInfo, error
Readdirnames(n int) : []string, error
Stat() : os.FileInfo, error
Sync() : error
Truncate(size int64) : error
WriteString(s string) : ret int, err error
```
In some applications it may make sense to define a new package that
simply exports the file system variable for easy access from anywhere.

## Using Afero's utility functions

Afero provides a set of functions to make it easier to use the underlying file systems.
These functions have been primarily ported from io & ioutil with some developed for Hugo.

The afero utilities support all afero compatible backends.

The list of utilities includes:

```go
DirExists(path string) (bool, error)
Exists(path string) (bool, error)
FileContainsBytes(filename string, subslice []byte) (bool, error)
GetTempDir(subPath string) string
IsDir(path string) (bool, error)
IsEmpty(path string) (bool, error)
ReadDir(dirname string) ([]os.FileInfo, error)
ReadFile(filename string) ([]byte, error)
SafeWriteReader(path string, r io.Reader) (err error)
TempDir(dir, prefix string) (name string, err error)
TempFile(dir, prefix string) (f File, err error)
Walk(root string, walkFn filepath.WalkFunc) error
WriteFile(filename string, data []byte, perm os.FileMode) error
WriteReader(path string, r io.Reader) (err error)
```
For a complete list see [Afero's GoDoc](https://godoc.org/github.com/spf13/afero)

They are available under two different approaches to use. You can either call
them directly where the first parameter of each function will be the file
system, or you can declare a new `Afero`, a custom type used to bind these
functions as methods to a given filesystem.

### Calling utilities directly

```go
fs := new(afero.MemMapFs)
f, err := afero.TempFile(fs,"", "ioutil-test")

```

### Calling via Afero

```go
fs := new(afero.MemMapFs)
afs := &Afero{Fs: fs}
f, err := afs.TempFile("", "ioutil-test")
```

## Using Afero for Testing

There is a large benefit to using a mock filesystem for testing. It has a
completely blank state every time it is initialized and can be easily
reproducible regardless of OS. You could create files to your heart’s content
and the file access would be fast while also saving you from all the annoying
issues with deleting temporary files, Windows file locking, etc. The MemMapFs
backend is perfect for testing.

* Much faster than performing I/O operations on disk
* Avoid security issues and permissions
* Far more control. 'rm -rf /' with confidence
* Test setup is far more easier to do
* No test cleanup needed

One way to accomplish this is to define a variable as mentioned above.
In your application this will be set to &afero.OsFs{} during testing you
can set it to &afero.MemMapFs{}.

It wouldn't be uncommon to have each test initialize a blank slate memory
backend. To do this I would define my `appFS = &afero.OsFs{}` somewhere
appropriate in my application code. This approach ensures that Tests are order
independent, with no test relying on the state left by an earlier test.

Then in my tests I would initialize a new MemMapFs for each test:
```go
func TestExist(t *testing.T) {
	appFS = &afero.MemMapFs{}
	// create test files and directories
	appFS.MkdirAll("src/a", 0755))
	appFS.WriteFile("src/a/b", []byte("file b"), 0644)
	appFS.WriteFile("src/c", []byte("file c"), 0644)
	testExistence("src/c", true, t)
}

func testExistence(name string, e bool, t *testing.T) {
	_, err := appFS.Stat(name)
	if os.IsNotExist(err) {
	    if e {
	        t.Errorf("file \"%s\" does not exist.\n", name)
    	}
	} else if err != nil {
    	panic(err)
	} else {
    	if !e {
    	    t.Errorf("file \"%s\" exists.\n", name)
    	}
	}
}
```

## Using Afero with Http

Afero provides an http compatible backend which can wrap any of the existing
backends.

The Http package requires a slightly specific version of Open which
returns an http.File type.

Afero provides an httpFs file system which satisfies this requirement.
Any Afero FileSystem can be used as an httpFs.

```go
httpFs := &afero.HttpFs{SourceFs: <ExistingFS>}
fileserver := http.FileServer(httpFs.Dir(<PATH>)))
http.Handle("/", fileserver)
```

# Available Backends

## OsFs

The first is simply a wrapper around the native OS calls. This makes it
very easy to use as all of the calls are the same as the existing OS
calls. It also makes it trivial to have your code use the OS during
operation and a mock filesystem during testing or as needed.

## MemMapFs

Afero also provides a fully atomic memory backed filesystem perfect for use in
mocking and to speed up unnecessary disk io when persistence isn’t
necessary. It is fully concurrent and will work within go routines
safely.

### InMemoryFile

As part of MemMapFs, Afero also provides an atomic, fully concurrent memory
backed file implementation. This can be used in other memory backed file
systems with ease. Plans are to add a radix tree memory stored file
system using InMemoryFile.

## Desired/possible backends

The following is a short list of possible backends we hope someone will
implement:

* SSH/SCP
* ZIP
* TAR
* S3
* Mem buffering to disk/network
* BasePath (where all paths are relative to a fixed basepath)

# Filters

You can add "filtering" to an Fs by adding a FilterFs to an existing Afero Fs
like
```go
    ROFs := afero.NewFilter(AppFs)
    ROFs.AddFilter(afero.NewReadonlyFilter())
```
The ROFs behaves like a normal afero.Fs now, with the only exception, that it
provides a readonly view of the underlying AppFs.

The FilterFs is run before the source Fs and may intercept the call to the
underlying source Fs and can modify the returned data. If it does not wish to
do so, it just returns the data from the source.

The `AddFilter` adds a new FilterFs before any existing filters.

## Available filters

* NewReadonlyFilter() - provide a read only view of the source Fs
* NewRegexpFilter(*regexp.Regexp) - provide a filtered view on file names, any
file (not directory) NOT matching the passed regexp will be treated as
non-existing



# About the project

## What's in the name

Afero comes from the latin roots Ad-Facere.

**"Ad"** is a prefix meaning "to".

**"Facere"** is a form of the root "faciō" making "make or do".

The literal meaning of afero is "to make" or "to do" which seems very fitting
for a library that allows one to make files and directories and do things with them.

The English word that shares the same roots as Afero is "affair". Affair shares
the same concept but as a noun it means "something that is made or done" or "an
object of a particular type".

It's also nice that unlike some of my other libraries (hugo, cobra, viper) it
Googles very well.

## Release Notes

* **0.10.0** 2015.12.10
  * Full compatibility with Windows
  * Introduction of afero utilities
  * Test suite rewritten to work cross platform
  * Normalize paths for MemMapFs
  * Adding Sync to the file interface
  * **Breaking Change** Walk and ReadDir have changed parameter order
  * Moving types used by MemMapFs to a subpackage
  * General bugfixes and improvements
* **0.9.0** 2015.11.05
  * New Walk function similar to filepath.Walk
  * MemMapFs.OpenFile handles O_CREATE, O_APPEND, O_TRUNC
  * MemMapFs.Remove now really deletes the file
  * InMemoryFile.Readdir and Readdirnames work correctly
  * InMemoryFile functions lock it for concurrent access
  * Test suite improvements
* **0.8.0** 2014.10.28
  * First public version
  * Interfaces feel ready for people to build using
  * Interfaces satisfy all known uses
  * MemMapFs passes the majority of the OS test suite
  * OsFs passes the majority of the OS test suite

## Contributing

1. Fork it
2. Create your feature branch (`git checkout -b my-new-feature`)
3. Commit your changes (`git commit -am 'Add some feature'`)
4. Push to the branch (`git push origin my-new-feature`)
5. Create new Pull Request

## Contributors

Names in no particular order:

* [spf13](https://github.com/spf13)
* [jaqx0r](https://github.com/jaqx0r)
* [mbertschler](https://github.com/mbertschler)

## License

Afero is released under the Apache 2.0 license. See
[LICENSE.txt](https://github.com/spf13/afero/blob/master/LICENSE.txt)

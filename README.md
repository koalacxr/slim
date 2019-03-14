<!--
based on the a great readme template
https://gist.github.com/PurpleBooth/109311bb0361f32d87a2
-->

# Slim - surprisingly space efficient data types in Golang

[![Travis-CI](https://api.travis-ci.org/openacid/slim.svg?branch=master)](https://travis-ci.org/openacid/slim)
[![AppVeyor](https://ci.appveyor.com/api/projects/status/ah6hlsojleqg8j9i/branch/master?svg=true)](https://ci.appveyor.com/project/drmingdrmer/slim/branch/master)
[![GoDoc](https://godoc.org/github.com/openacid/slim?status.svg)](http://godoc.org/github.com/openacid/slim)
[![Report card](https://goreportcard.com/badge/github.com/openacid/slim)](https://goreportcard.com/report/github.com/openacid/slim)
[![Sourcegraph](https://sourcegraph.com/github.com/openacid/slim/-/badge.svg)](https://sourcegraph.com/github.com/openacid/slim?badge)


Slim is collection of surprisingly space efficient data types, with
corresponding serialisation APIs to persisting them on-disk or for transport.

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
 

- [Memory overhead](#memory-overhead)
- [Performance benchmark](#performance-benchmark)
- [Status](#status)
- [Synopsis](#synopsis)
  - [Use SlimIndex to index external data](#use-slimindex-to-index-external-data)
- [Getting started](#getting-started)
- [Who are using slim](#who-are-using-slim)
- [Roadmap](#roadmap)
- [Feedback and contributions](#feedback-and-contributions)
- [Authors](#authors)
- [License](#license)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Why slim

As data on internet keeps increasing exponentially,
the capacity gap between memory and disk becomes greater.
Most of the data does not need to be loaded into expensive main memory.
Only the most important part of them -- where a record is -- deserve a seat in
main memory.

This is what `slim` does, keeps as little information as possible in main
memory, as a minimised index of huge amount external data.

-   `SlimIndex`: is a common index structure, building on top of `SlimTrie`.

    [![GoDoc](https://godoc.org/github.com/openacid/slim/index?status.svg)](http://godoc.org/github.com/openacid/slim/index)

-   `SlimTrie` is the underlying index data structure, evolved from [trie][].

    [![GoDoc](https://godoc.org/github.com/openacid/slim/trie?status.svg)](http://godoc.org/github.com/openacid/slim/trie)

    **Features**:

    -   **Minimised**:
        requires only **6 bytes per key**(even less than an 8-byte pointer!!).

    -   **Stable**:
        memory consumption is stable in various scenarios.
        The Worst case converges to average consumption tightly.
        See benchmark.

    -   **Unlimited key length**:
        You can have **VERY** long keys, without bothering yourself with any
        waste of memory(and money).
        Do not waste your life writing another prefix compression`:)`.
        ([aws-s3][] limits key length to 1024 bytes).
        Memory consumption only relates to key count, **not to key length**.

    -   **Ordered**:
        like [btree][], keys are stored in alphabetic order, but faster to access.
        Range-scan is supported!(under development)

    -   **Fast**:
        time complexity for a get is `O(log(n) + k); n: key count; k: key length`.
        With comparison to [btree][], which is `O(log(n) * k)`(tree-like).
        And golang-map is `O(k)`(hash-table-like).

    -   **Ready for transport**:
        `SlimTrie` has no gap between its in-memory layout and its on-disk
        layout or transport presentation.
        Unlike other data-structure such as the popular [red-black-tree][],
        which is designed for in-memory use only and requires additional work to
        persist or transport it.

    -   **Loosely coupled design**:
        index logic and data storage is completely separated.
        Piece of cake using `SlimTrie` to index huge data.


<!-- TODO array -->

<!-- TODO list data types -->
<!-- TODO other data types -->

<!-- TODO toc -->

## Memory overhead

Comparison of `SlimTrie` and native golang-map.

- Key length: 512 byte, different key count:

| Key count | Key length | SlimTrie: byte/key | Map: byte/key |
| --:       | --:        | --:                | --:           |
| 13107     | 512        |  7.3               | 542.2         |
| 16384     | 512        |  7.0               | 554.5         |
| 21845     | 512        |  7.2               | 544.9         |

- Key count: 16384, different key length:

| Key count | Key length | SlimTrie: byte/key | Map: byte/key |
| --:       | --:        | --:                | --:           |
| 16384     | 512        | 7.0                | 554.5         |
| 16384     | 1024       | 7.0                | 1066.5        |

Memory overhead per key is stable with different key length(`k`) and key count(`n`):

![benchmark-mem-kn-png][]

**SlimTrie memory does not increase when key become longer**.

## Performance benchmark

Time(in nano second) spent on a `get` operation with SlimTrie, golang-map and [btree][] by google.

**Smaller is better**.

![benchmark-get-png][]

| Key count | Key length | SlimTrie | Map  | Btree |
| ---:      | ---:       | ---:     | ---: | ---:  |
| 1         | 1024       | 86.3     | 5.0  | 36.9  |
| 10        | 1024       | 90.7     | 59.7 | 99.5  |
| 100       | 1024       | 123.3    | 60.1 | 240.6 |
| 1000      | 1024       | 157.0    | 63.5 | 389.6 |
| 1000      | 512        | 152.6    | 40.0 | 363.0 |
| 1000      | 256        | 152.3    | 28.8 | 332.3 |

It is about **2.6 times faster** than the [btree][] by google.

Time(in nano second) spent on a `get` with different key count(`n`) and key length(`k`):

![benchmark-get-kn-png][]

See: [benchmark-get-md][].

## Status

-   `0.1.0`(Current): index structure.

## Synopsis

### Use SlimIndex to index external data

```go
package main
import (
        "fmt"
        "strings"
        "github.com/openacid/slim/index"
)

type Data string

func (d Data) Read(offset int64, key string) (string, bool) {
        kv := strings.Split(string(d)[offset:], ",")[0:2]
        if kv[0] == key {
                return kv[1], true
        }
        return "", false
}

func main() {
        // `data` is a sample external data.
        // In order to let SlimIndex be able to read data, `data` should have
        // a `Read` method:
        //     Read(offset int64, key string) (string, bool)
        data := Data("Aaron,1,Agatha,1,Al,2,Albert,3,Alexander,5,Alison,8")

        // keyOffsets is a prebuilt index that stores key and its offset in data accordingly.
        keyOffsets := []index.OffsetIndexItem{
                {Key: "Aaron",     Offset: 0},
                {Key: "Agatha",    Offset: 8},
                {Key: "Al",        Offset: 17},
                {Key: "Albert",    Offset: 22},
                {Key: "Alexander", Offset: 31},
                {Key: "Alison",    Offset: 43},
        }

        // Create an index
        st, err := index.NewSlimIndex(keyOffsets, data)
        if err != nil {
                fmt.Println(err)
        }

        // Lookup by SlimIndex
        v, found := st.Get2("Alison")
        fmt.Printf("key: %q\n  found: %t\n  value: %q\n", "Alison", found, v)

        v, found = st.Get2("foo")
        fmt.Printf("key: %q\n  found: %t\n  value: %q\n", "foo", found, v)

        // Output:
        // key: "Alison"
        //   found: true
        //   value: "8"
        // key: "foo"
        //   found: false
        //   value: ""
}
```


<!-- ## FAQ -->

## Getting started

**Install**

```sh
go get github.com/openacid/slim
```

All dependency packages are included in `vendor/` dir.


<!-- TODO add FAQ -->
<!-- TODO add serialisation explanation, on-disk data structure etc. -->

**Prerequisites**

-   **For users** (who'd like to build cool stuff with `slim`):

    **Nothing**.

-   **For contributors** (who'd like to make `slim` better):

    -   `dep`:
        for dependency management.
    -   `protobuf`:
        for re-compiling `*.proto` file if on-disk data structure changes.

    Max OS X:
    ```sh
    brew install dep protobuf
    ```

    On other platforms you can read more:
    [dep-install][],
    [protoc-install][].


## Who are using slim

<span> <span> ![][baishancloud-favicon] </span> <span> [baishancloud][] </span> </span>

<!-- ## Slim internal -->

<!-- ### Built With -->

<!-- - [protobuf][] - Define on-disk data-structure and serialisation engine. -->
<!-- - [dep][] - Dependency Management. -->
<!-- - [semver][] - For versioning data-structure. -->

<!-- ### Directory Layout -->

<!-- We follow the: [golang-standards-project-layout][]. -->

<!-- [> TODO read the doc and add more standards <] -->

<!-- -   `vendor/`: dependency packages. -->
<!-- -   `prototype/`: on-disk data-structure. -->
<!-- -   `docs/`: documents about design, trade-off, etc -->
<!-- -   `tools/`: documents about design, trade-off, etc -->
<!-- -   `expamples/`: documents about design, trade-off, etc -->

<!-- Other directories are sub-package. -->


<!-- ### Versioning -->

<!-- We use [SemVer](http://semver.org/) for versioning. -->

<!-- For the versions available, see the [tags on this repository](https://github.com/your/project/tags).  -->

<!-- ### Data structure explained -->
<!-- [> TODO  <] -->

<!-- ## Limitation -->
<!-- [> TODO  <] -->

## Roadmap

-   [x] **2019 Mar 08**: SlimIndex, SlimTrie
-   [ ] Marshalling support
-   [ ] SlimArray


<!-- -   [ ] bitrie: 1 byte-per-key implementation. -->
<!-- -   [ ] balanced bitrie: which gives better worst-case performance. -->
<!-- -   [ ] generalised API as a drop-in replacement for map etc. -->


## Feedback and contributions

**Feedback and Contributions are greatly appreciated**.

At this stage, the maintainers are most interested in feedback centred on:

-   Do you have a real life scenario that `slim` supports well, or doesn't support at all?
-   Do any of the APIs fulfil your needs well?

Let us know by filing an issue, describing what you did or wanted to do, what
you expected to happen, and what actually happened:

-   [bug-report][]
-   [improve-document][]
-   [feature-request][]

Or other type of [issue][new-issue].

<!-- ## Contributing -->
<!-- The maintainers actively manage the issues list, and try to highlight issues -->
<!-- suitable for newcomers. -->

<!-- [> TODO dep CONTRIBUTING <] -->
<!-- The project follows the typical GitHub pull request model. See CONTRIBUTING.md for more details. -->

<!-- Before starting any work, please either comment on an existing issue, -->
<!-- or file a new one. -->

<!-- [> TODO  <] -->
<!-- Please read [CONTRIBUTING.md][] -->
<!-- for details on our code of conduct, and the process for submitting pull requests to us. -->
<!-- https://gist.github.com/PurpleBooth/b24679402957c63ec426 -->


<!-- ### Code style -->

<!-- ### Tool chain -->

<!-- ### Customised install -->

<!-- Alternatively, if you have a customised go develop environment, you could also -->
<!-- clone it: -->

<!-- ```sh -->
<!-- git clone git@github.com:openacid/slim.git -->
<!-- ``` -->

<!-- As a final step you'd like have a test to see if everything goes well: -->

<!-- ```sh -->
<!-- cd path/to/slim/build/pseudo-gopath -->
<!-- export GOPATH=$(pwd) -->
<!-- go test github.com/openacid/slim/array -->
<!-- ``` -->

<!-- Another reason to have a `pseudo-gopath` in it is that some tool have their -->
<!-- own way conducting source code tree. -->
<!-- E.g. [git-worktree](https://git-scm.com/docs/git-worktree) -->
<!-- checkouts source code into another dir other than the GOPATH work space. -->

<!-- ## Update dependency -->

<!-- Dependencies are tracked by [dep](https://github.com/golang/dep). -->
<!-- All dependencies are kept in `vendor/` dir thus you do not need to do anything -->
<!-- to run it. -->

<!-- You need to update dependency only when you bring in new feature with other dependency. -->

<!-- -   Install `dep` -->

<!--     ``` -->
<!--     curl https://raw.githubusercontent.com/golang/dep/master/install.sh | sh -->
<!--     ``` -->

<!-- -   Download dependency -->

<!--     ``` -->
<!--     dep ensure -->
<!--     ``` -->

<!--     > dep uses Gopkg.toml Gopkg.lock to track dependency info. -->
<!--     >  -->
<!--     > Gopkg.toml Gopkg.lock is created with `dep init`. -->
<!--     > -->
<!--     > dep creates a `vendor` dir to have all dependency package there. -->

<!-- See more: [dep-install][] -->


## Authors

<!-- ordered by unicode of author's name -->
<!-- leave 3 to 5 major jobs you have done in this project -->

- ![][刘保海-img-sml] **[刘保海][]** *marshalling*
- ![][吴义谱-img-sml] **[吴义谱][]** *array*
- ![][张炎泼-img-sml] **[张炎泼][]** *slimtrie design*
- ![][李文博-img-sml] **[李文博][]** *trie-compressing, trie-search*
- ![][李树龙-img-sml] **[李树龙][]** *marshalling*


See also the list of [contributors][] who participated in this project.


## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

<!-- ## Acknowledgments -->

<!-- [> TODO  <] -->
<!-- - Hat tip to anyone whose code was used -->

<!-- - Inspiration -->
<!--     patricial tree -->
<!--     fusion tree -->
<!--     critic trie -->
<!-- - etc -->

<!-- links -->

<!-- Bio -->

[刘保海]: https://github.com/liubaohai
[吴义谱]: https://github.com/pengsven
[张炎泼]: https://github.com/drmingdrmer
[李文博]: https://github.com/wenbobuaa
[李树龙]: https://github.com/lishulong

<!-- avatar -->

[刘保海-img-sml]: https://avatars1.githubusercontent.com/u/26271283?s=36&v=4
[吴义谱-img-sml]: https://avatars3.githubusercontent.com/u/6927668?s=36&v=4
[张炎泼-img-sml]: https://avatars3.githubusercontent.com/u/44069?s=36&v=4
[李文博-img-sml]: https://avatars1.githubusercontent.com/u/11748387?s=36&v=4
[李树龙-img-sml]: https://avatars2.githubusercontent.com/u/13903162?s=36&v=4

[contributors]: https://github.com/openacid/slim/contributors

[dep]: https://github.com/golang/dep
[protobuf]: https://github.com/protocolbuffers/protobuf
[semver]: http://semver.org/

[protoc-install]: http://google.github.io/proto-lens/installing-protoc.html
[dep-install]: https://github.com/golang/dep#installation

[CONTRIBUTING.md]: CONTRIBUTING.md

[baishancloud]: http://www.baishancdnx.com
[baishancloud-favicon]: http://www.baishancdnx.com/public/favicon.ico
[golang-standards-project-layout]: https://github.com/golang-standards/project-layout

<!-- issue links -->

[bug-report]:       https://github.com/openacid/slim/issues/new?labels=bug&template=bug_report.md
[improve-document]: https://github.com/openacid/slim/issues/new?labels=doc&template=doc_improve.md
[feature-request]:  https://github.com/openacid/slim/issues/new?labels=feature&template=feature_request.md

[new-issue]: https://github.com/openacid/slim/issues/new/choose

<!-- benchmark -->

[benchmark-get-png]: docs/trie/charts/search_existing.png
[benchmark-get-kn-png]: docs/trie/charts/search_k_n.png
[benchmark-get-md]: docs/trie/benchmark_result.md

[benchmark-mem-kn-png]: docs/trie/charts/mem_usage_k_n.png

<!-- links to other resource -->

<!-- reference -->

[trie]: https://en.wikipedia.org/wiki/Trie
[btree]: https://github.com/google/btree
[aws-s3]: https://aws.amazon.com/s3/
[red-black-tree]: https://en.wikipedia.org/wiki/Red%E2%80%93black_tree

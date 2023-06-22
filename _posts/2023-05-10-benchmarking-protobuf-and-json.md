---
layout:             post
title:              "Benchmarking Protocol Buffer and JSON Performance"
category:           "Web Applications and Cybersecurity"
tags:               protobuf json go
permalink:          /posts/benchmarking-protobuf-and-json-performance
last_modified_at:   "2023-06-20"
---

In this post, I would like to present some basic observations on performance differences between two data serialization protocols: Protocol Buffer (Protobuf) and JSON. Specifically, I would like to compare them based on serialization/de-serialization speeds and the memory footprint of data encoding for different data sizes.

<!-- excerpt-end -->

I'm using a MacBook Air (13-inch, Early 2015) running macOS Monterey Version 12.6.6. The Go source code is borrowed from [Alex Fattouche's](https://github.com/Fattouche/protobuf-benchmark/) and [Artem Kresling's](https://medium.com/@akresling/go-benchmark-json-v-protobuf-4ec3c62ec8d4) blog posts. Those not interested in the implementation details can just jump to the "Run Tests" section.

```console
$ tree .
.
├── Dockerfile
├── go.mod
├── go.sum
├── pb_test.go
└── protos
    ├── test.pb.go
    └── test.proto

2 directories, 6 files
```

## Table of Contents
{:.no_toc}
* TOC 
{:toc}

## The `test.proto` File

```protobuf
syntax = "proto3";

option go_package = ".;protos";

message Small {
    string action = 1;
    bytes key = 2;
}

message Medium {
    string name = 1;
    int64 age = 2;
    float height = 3;
    double weight = 4;
    bool alive = 5;
    bytes desc = 6;
}

message Large {
    string name = 1;
    int64 age = 2;
    float height = 3;
    double weight = 4;
    bool alive = 5;
    bytes desc = 6;
    string nickname = 7;
    int64 num = 8;
    float flt = 9;
    double dbl = 10;
    bool tru = 11;
    bytes data = 12;
}
```

## The `pb_test.go` File

```go
package main

import (
	"encoding/json"
	"fmt"
	"testing"

	"pb-test/protos"

	"google.golang.org/protobuf/proto"
)

var (
	PBSmall = &protos.Small{
		Action: "benchmark",
		Key:    []byte("data to be sent"),
	}

	PBMedium = &protos.Medium{
		Name:   "xj3jJd9A8sK31D5R25UWy8OzMRI3Ok022aE8W1dmRKycZHe2zf7bzU4Qvfd",
		Age:    20,
		Height: 5.8,
		Weight: 180.7,
		Alive:  true,
		Desc:   []byte(`3U6CsMB4D8yPlH3cje0KHEX7QyZaFSbfuRMDzEZaPmFjwLiXamAXee2YIiBX3UaWBikJWAmUGaj87dqTUSps1kcwOAbWpAaWoJzAfTCtrsGErq69cCarneVAajfyAkYlZvXGRLIifqxRZnrOjfW5oAj7mwBkCYXo43i6KnRll3iTNtUSwKMYwK3qdG04LPIjvGIzKapB`),
	}

	PBLarge = &protos.Large{
		Name:     "xj3jJd9A8sK31D5R25UWy8OzMRI3Ok022aE8W1dmRKycZHe2zf7bzU4QvfdAlFQcDgXYHIG42JldotgmVp6uIyJMtMqmJ1PzQyEgvTNUUWy3HjL3eTRh78rxuUmCXB2XXpt1CEl9VJpFRshGSkN3pZ",
		Age:      20,
		Height:   5.8,
		Weight:   180.7,
		Alive:    true,
		Desc:     []byte("Lets benchmark some json and protobuf"),
		Nickname: "Another name for PBLarge",
		Num:      2314,
		Flt:      123451231.1234,
		Data: []byte(`loFNv72sSvJn8rvB4G2irFPDKKPA43wTE96FLEpc21RNXUIXxDYL5T7453S5hSGHcgmiYxEf22x2y0ecLGPdLCdNw5RojO8lquoW23QOxGgh7cVYRbxdUBHVCzVcGIpV7b7j2Uc2MbJz6ipPaq7E3t8q2TrVI
		mtilR77fTWnriuGD1DlThPXGwKjAej7aNVPsxOuUJMFII5dEyluFszVgHnSg1kJPP38IE8WovGEzuogSJWYmISD6PbrItUTi7Al1zMACbsuoM0NHhbeVtrfbWunlTtOvKsQqhWN3tAllETlI6P6Mcc5cd4y7t6rEdcshZg1pT6M
		WonM09OyDEenEy6bUOKDNClEMZQAxq8sLlNhjRWilItDza3gEqJmT5D6EJ4r8WDJ4B1WDAKoojHcrgj4RqDgbrdOQRNhWJeoQc9nBiY5CshMakAaqDLuC9F98KCKkFJiL3CCv4Rg4An1YqArMkfL1EMRu6FRuiwx7wDPcEG5bbF
		URaNQAE7BgLCbKRijtxkWEvRzkvy9oeFV6Og7pga4oUG7w3INvWewaWBfLQytoRdngtcfea8nnfw6Ecoz1keIeIs9KHyN0TOfYcxcPfQpLwUl7cIbIQOogUWUBJSXHHoa6KyBfFQb1VnAZiIXt2QrWBPXtGum8PrhqPHnH1Wuql5
		oaujtu6RsyGBLJvyg671vT8pxXomWZg0o7YY7gQpQ6sky4mFeRpuo5Gjlg7K8gfbIIFgPby0BKa6Kpro4cCX523xmDOUyWo0xuWXTiWRH24A7G8W4oWstpUIYdT3cy4MnTxFjDZ0o95SwFEbZz9ihaEVBIZbZ42AhugVnblIIcd4
		FkujVHNi0sOACf8HzjjvEBnECCb3EOPnRasVuHvC7tdW2AZY4OPDKyEjhbdWxYsrgvXpbHgA2MVajv1gBRHwwcV9D4DX7zc67D3t1vDV33l1Skqm3fnGqdgbDbOLPw1Jqf6KMY780qP9cUKWHzx4uJAKr6xKtozRDmsiRBVgETkF
		QLVyjHE0MZGgWMFYos1RjmVpD0NdUQHLGcZguOW4zkOjiwQzLvUSbqsYHoBNdBHs75JVmUl3JDwCiZjdqCYUbJRiiGxtEHL2ebchdQqPcCBQJYMOr592B5BFYtCPMYrFX24kgk1gJ7drlx9MEmFkss6h0F42VwdP77SL6FfSqsys
		81HiY6eBTXWoyuncJX73sDrgw59J4tz5tbE5hr3HaSTkd99M2nzugCfbxkBCQ8oeMn3NClcMRoHuc6EftmerSczdzepXdVjzmx9OR5CgtUyG5VOJp8dQBul4TdRV761Vh6ezk65I017JoXKbYRh9n40WPzElQk28BbE2TuK7ny2WU
		XlQPbmaIOr9rvsxUTXIbFKrPlBpkTjmrfD24feCoMzCpVWsQwJPKvAemDGdH8vVvp6UKidYhOSj5C2DnRREUzsZLoRdBmsU7Z5sxXoVTRn3H9j0nafE2WulxZqBPpJR0gUQjL6jTQ1IRhmB8t2Q0BonfRF2LgDEb1oOrhGJamZjGH
		PANWvwRc75LktFWFB2q71YXVmXPLwYPic50i7rsrcoLSUuWUdePqSEed8HFKL1wwhPz4HtGfV6xu9RF7tAdmPIlMnbQVszeKR2wO9SYUQchhLe9JrJu33JlIp4GF6NUjMXa7B2cym6J8ibpGvTqOlUeqoy9IGJUJ95tgqHuzLcG2
		mI0q3ZOa4jVlfphEKaiUR7R2vgT5VsTTumz14MpcfvVFWNzhbxViIvy815CWtNlQguRx7I5KLS9oLETfXZ7AA0zIwXCjre4XCglytCEkous4UPGl9FX8AHWZbhbpIOEqor4u9Eroyd11Ncey2s7s2v0q6ASUyvO42Ppbo8hg9HIi
		0dEzOPcUX9QgQWQmPXnxFa0qZ5lZTeOy96QGeIzxN09eaVAU5xl0yKyVYzA4ETJBuYxFgPdTjw3SxEN60VC54lDrIZ48W5TpSGHk6M6OAqjCSlUwmY0IV1FMxEOOLubP7ZmFFcvliCgnBIMD9RUmEUcWq12ZsDRwvW5wihpTeNhI
		0iWpPHfklcJs6SGEn3dJo6KYaaJQg828Sk8oxOHTZiuyJBemsn21uftJR8G8FeRBt2yjkFVg7Pxg1uPqNJKyDvp4ujNDi9BNatxeBkqNhu0SBT7dhd6dnEkDpdzjP5MKiL6FS1JekPAiXW9brBFn2VGsNpNEM9ou1IGXaxHq5ntf
		VPJs13SzHTOcnV2OXFpKALUyaNr9Fnwobh8noyOo5H5OcHKNih4gFr4nDXQVtOof2H9cRjuySlT2eWWJQ7aIXeHjY338UTjOU6i5dtaPx6cdITo2NiHsygd1u32oiIJFFtSkDnm5aHtcaBpAO8MEBQtSoa9S2HpupHE7RVwWGMdf
		fyMMttFjp9qEAOOhBSEAwGtVmDBISXMWswpA3xkHrMIzDx2zSCfWkjaAQAjKYnaXywKwdlK7UgFU3SqsCFODp6EcTl5ygO1lpAPazh4jytdQw4K4JLoVUHwOt0YdnQYtJLZ9W66rvLyPmh83WGSYeD4F0CTyoLBOLBSbFrcR8Ess
		pm5GfHiU5aXWTCYV5QQA0YPyIlDcBXXPdqXepQnWSkd5wy2MxGsxVOgkyGZHcOsVXE0WF4ndNhyf99j82TNe565K53iNn1lVRrwALEKb2FES7hXWf7bjn84biqiwP7IOCtyAQbvLcDrDpeu6kmHTPg3b7FX2N19ui9fXmMZwqvj1`),
	}
)

func TestDataAllocationsSmall(_ *testing.T) {
	fmt.Printf("---------- Small ----------\n")
	bs := PBSmall
	j, _ := json.Marshal(bs)
	p, _ := proto.Marshal(bs)

	printInfo(j, "json")
	printInfo(p, "protobuf")
	fmt.Printf("\n")
}

func TestDataAllocationsMedium(_ *testing.T) {
	fmt.Printf("---------- Medium ----------\n")
	bs := PBMedium
	j, _ := json.Marshal(bs)
	p, _ := proto.Marshal(bs)

	printInfo(j, "json")
	printInfo(p, "protobuf")
	fmt.Printf("\n")
}

func TestDataAllocationsLarge(_ *testing.T) {
	fmt.Printf("---------- Large ----------\n")
	bs := PBLarge
	j, _ := json.Marshal(bs)
	p, _ := proto.Marshal(bs)

	printInfo(j, "json")
	printInfo(p, "protobuf")
	fmt.Printf("\n")
}

func BenchmarkJSONMarshal(b *testing.B) {
	s := PBSmall
	m := PBMedium
	l := PBLarge

	b.ResetTimer()

	b.Run("SmallData", func(b *testing.B) {
		b.ReportAllocs()
		for n := 0; n < b.N; n++ {
			d, _ := json.Marshal(s)
			_ = d
		}
	})
	b.Run("MediumData", func(b *testing.B) {
		b.ReportAllocs()
		for n := 0; n < b.N; n++ {
			d, _ := json.Marshal(m)
			_ = d
		}
	})
	b.Run("LargeData", func(b *testing.B) {
		b.ReportAllocs()
		for n := 0; n < b.N; n++ {
			d, _ := json.Marshal(l)
			_ = d
		}
	})
	fmt.Printf("\n")
}

func BenchmarkProtobufMarshal(b *testing.B) {
	s := PBSmall
	m := PBMedium
	l := PBLarge

	b.ResetTimer()

	b.Run("SmallData", func(b *testing.B) {
		b.ReportAllocs()
		for n := 0; n < b.N; n++ {
			d, _ := proto.Marshal(s)
			_ = d
		}
	})
	b.Run("MediumData", func(b *testing.B) {
		b.ReportAllocs()
		for n := 0; n < b.N; n++ {
			d, _ := proto.Marshal(m)
			_ = d
		}
	})
	b.Run("LargeData", func(b *testing.B) {
		b.ReportAllocs()
		for n := 0; n < b.N; n++ {
			d, _ := proto.Marshal(l)
			_ = d
		}
	})
	fmt.Printf("\n")
}

func BenchmarkJSONUnmarshal(b *testing.B) {
	s := PBSmall
	m := PBMedium
	l := PBLarge

	sd, _ := json.Marshal(s)
	md, _ := json.Marshal(m)
	ld, _ := json.Marshal(l)

	var sf protos.Small
	var mf protos.Medium
	var lf protos.Large

	b.ResetTimer()

	b.Run("SmallData", func(b *testing.B) {
		b.ReportAllocs()
		for n := 0; n < b.N; n++ {
			_ = json.Unmarshal(sd, &sf)
		}
	})
	b.Run("MediumData", func(b *testing.B) {
		b.ReportAllocs()
		for n := 0; n < b.N; n++ {
			_ = json.Unmarshal(md, &mf)
		}
	})
	b.Run("LargeData", func(b *testing.B) {
		b.ReportAllocs()
		for n := 0; n < b.N; n++ {
			_ = json.Unmarshal(ld, &lf)
		}
	})
	fmt.Printf("\n")
}

func BenchmarkProtobufUnmarshal(b *testing.B) {
	s := PBSmall
	m := PBMedium
	l := PBLarge

	sd, _ := proto.Marshal(s)
	md, _ := proto.Marshal(m)
	ld, _ := proto.Marshal(l)

	var sf protos.Small
	var mf protos.Medium
	var lf protos.Large

	b.ResetTimer()

	b.Run("SmallData", func(b *testing.B) {
		b.ReportAllocs()
		for n := 0; n < b.N; n++ {
			_ = proto.Unmarshal(sd, &sf)
		}
	})
	b.Run("MediumData", func(b *testing.B) {
		b.ReportAllocs()
		for n := 0; n < b.N; n++ {
			_ = proto.Unmarshal(md, &mf)
		}
	})
	b.Run("LargeData", func(b *testing.B) {
		b.ReportAllocs()
		for n := 0; n < b.N; n++ {
			_ = proto.Unmarshal(ld, &lf)
		}
	})
}

func printInfo(d []byte, typ string) {
	used := len(d)
	allocated := cap(d)
	fmt.Printf("Type: %s \t\tData size: %d \t\tTotal Allocated: %d \t\t Used/Allocated: %.2f%%\n", typ, used, allocated, percentUsed(used, allocated)*100)
}

func percentUsed(used, allocated int) float32 {
	return float32(used) / float32(allocated)
}
```

## Run Tests

### Preparation

1. Install the Protobuf compiler:

    ```bash
    brew install protobuf
    ```

2. Generate Go code for `test.proto`:

    ```bash
    go get -u google.golang.org/protobuf/cmd/protoc-gen-go
    go install google.golang.org/protobuf/cmd/protoc-gen-go

    protoc --go_out=. --go_opt=paths=source_relative protos/test.proto
    ```

3. Go module initialization:

    ```bash
    go mod init pb-test && go mod tidy
    ```

4. Finally, issue the following command:

    ```bash
    go test --bench=.
    ```

### Run Tests with Docker

Example Dockerfile:

```dockerfile
FROM --platform=linux/amd64 golang:alpine

WORKDIR /app

COPY ./protos ./protos

ENV GO111MODULE=on
COPY go.mod go.sum ./
RUN go mod download

COPY *.go ./
CMD ["go", "test", "--bench=."]
```

Build and run:

```bash
docker build -t pb-test .

docker run pb-test
```

## Results

Running natively:

```console
---------- Small ----------
Type: json 		Data size: 51 		Total Allocated: 64 		 Used/Allocated: 79.69%
Type: protobuf 		Data size: 28 		Total Allocated: 28 		 Used/Allocated: 100.00%

---------- Medium ----------
Type: json 		Data size: 398 		Total Allocated: 416 		 Used/Allocated: 95.67%
Type: protobuf 		Data size: 282 		Total Allocated: 282 		 Used/Allocated: 100.00%

---------- Large ----------
Type: json 		Data size: 4060 		Total Allocated: 4096 		 Used/Allocated: 99.12%
Type: protobuf 		Data size: 3029 		Total Allocated: 3029 		 Used/Allocated: 100.00%

goos: darwin
goarch: amd64
pkg: pb-test
cpu: Intel(R) Core(TM) i5-5250U CPU @ 1.60GHz
BenchmarkJSONMarshal/SmallData-4         	 2559181	       498.6 ns/op	      64 B/op	       1 allocs/op
BenchmarkJSONMarshal/MediumData-4        	  580328	      2094 ns/op	     704 B/op	       2 allocs/op
BenchmarkJSONMarshal/LargeData-4         	  114592	      9950 ns/op	    5249 B/op	       2 allocs/op

BenchmarkProtobufMarshal/SmallData-4     	 5279698	       196.2 ns/op	      32 B/op	       1 allocs/op
BenchmarkProtobufMarshal/MediumData-4    	 3208576	       409.9 ns/op	     288 B/op	       1 allocs/op
BenchmarkProtobufMarshal/LargeData-4     	 1000000	      1473 ns/op	    3072 B/op	       1 allocs/op

BenchmarkJSONUnmarshal/SmallData-4       	  657160	      2030 ns/op	     248 B/op	       6 allocs/op
BenchmarkJSONUnmarshal/MediumData-4      	  188473	      6173 ns/op	     504 B/op	       9 allocs/op
BenchmarkJSONUnmarshal/LargeData-4       	   27450	     46313 ns/op	    3560 B/op	      13 allocs/op

BenchmarkProtobufUnmarshal/SmallData-4   	 3630885	       308.2 ns/op	      32 B/op	       2 allocs/op
BenchmarkProtobufUnmarshal/MediumData-4  	 2611279	       496.6 ns/op	     272 B/op	       2 allocs/op
BenchmarkProtobufUnmarshal/LargeData-4   	  788181	      1750 ns/op	    3304 B/op	       4 allocs/op
PASS
ok  	pb-test	20.718s
```

Running inside a Docker container:

```console
---------- Small ----------
Type: json 		Data size: 51 		Total Allocated: 64 		 Used/Allocated: 79.69%
Type: protobuf 		Data size: 28 		Total Allocated: 28 		 Used/Allocated: 100.00%

---------- Medium ----------
Type: json 		Data size: 398 		Total Allocated: 416 		 Used/Allocated: 95.67%
Type: protobuf 		Data size: 282 		Total Allocated: 282 		 Used/Allocated: 100.00%

---------- Large ----------
Type: json 		Data size: 4060 		Total Allocated: 4096 		 Used/Allocated: 99.12%
Type: protobuf 		Data size: 3029 		Total Allocated: 3029 		 Used/Allocated: 100.00%

goos: linux
goarch: amd64
pkg: pb-test
cpu: Intel(R) Core(TM) i5-5250U CPU @ 1.60GHz
BenchmarkJSONMarshal/SmallData-2         	 2541802	       562.9 ns/op	      64 B/op	       1 allocs/op
BenchmarkJSONMarshal/MediumData-2        	  674119	      2012 ns/op	     704 B/op	       2 allocs/op
BenchmarkJSONMarshal/LargeData-2         	  104707	     11690 ns/op	    5248 B/op	       2 allocs/op

BenchmarkProtobufMarshal/SmallData-2     	 4147478	       248.6 ns/op	      32 B/op	       1 allocs/op
BenchmarkProtobufMarshal/MediumData-2    	 2039112	       571.2 ns/op	     288 B/op	       1 allocs/op
BenchmarkProtobufMarshal/LargeData-2     	  537639	      3367 ns/op	    3072 B/op	       1 allocs/op

BenchmarkJSONUnmarshal/SmallData-2       	  658435	      1902 ns/op	     248 B/op	       6 allocs/op
BenchmarkJSONUnmarshal/MediumData-2      	  157466	      7886 ns/op	     504 B/op	       9 allocs/op
BenchmarkJSONUnmarshal/LargeData-2       	   26912	     47267 ns/op	    3560 B/op	      13 allocs/op

BenchmarkProtobufUnmarshal/SmallData-2   	 3696394	       342.0 ns/op	      32 B/op	       2 allocs/op
BenchmarkProtobufUnmarshal/MediumData-2  	 1950585	       571.3 ns/op	     272 B/op	       2 allocs/op
BenchmarkProtobufUnmarshal/LargeData-2   	  391442	      2730 ns/op	    3304 B/op	       4 allocs/op
PASS
ok  	pb-test	21.233s
```

The four columns of the benchmark section are: the number of iterations, time per function call, memory allocation, and the rate of memory allocation, respectively. Note that Protobuf has smaller memory footprints and, in most cases, lower rate of memory allocation than JSON. The `marshal` and `unmarshal` speeds of Protobuf are significantly faster than those of JSON, especially for large data size.
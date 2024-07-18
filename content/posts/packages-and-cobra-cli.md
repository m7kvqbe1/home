+++
title = 'Go Packages and Cobra CLI'
date = 2024-07-18T12:08:10+01:00
draft = false
+++

[Cobra](https://github.com/spf13/cobra) is a library for creating CLI applications in Go. It provides a simple interface for defining commands, flags, and subcommands.

I've utilized Cobra to create a CLI tool and organized the code into multiple packages.

Find the related source code here:

https://github.com/m7kvqbe1/advent-of-code-2023

![Advent of Code 2023 Cobra CLI](/images/advent-of-code-2023-cobra.png)

## Init

First we initialize the project as a new module:

```bash
go mod init github.com/m7kvqbe1/advent-of-code-2023
```

This creates a `go.mod` file (there is also `go.sum` file which is essentially a lockfile). As we import packages the Go tooling automatically updates these files.

It specifies direct and indirect deps:

```sh
module github.com/m7kvqbe1/advent-of-code-2023

go 1.22.1

require github.com/spf13/cobra v1.8.1

require (
	github.com/inconshreveable/mousetrap v1.1.0 // indirect
	github.com/spf13/pflag v1.0.5 // indirect
)

```

## Organizing Code into Packages

The code has then been divided into multiple packages within the `pkg` directory (this is an [established Go convention](https://github.com/golang-standards/project-layout/tree/master/pkg)).

Each package corresponds to a specific day of the 'Advent of Code' challenge and contains the logic for each part of that day's challenge in seperate files.

The package namespace is declared at the top of each file:

```go
package day01
```

We can import one of our local packages like so:

```go
import (
  "github.com/m7kvqbe1/advent-of-code-2023/pkg/day01"
  // ...
)
```

## Using Cobra for CLI

The `main.go` file in the root of the codebase contains a root command and subcommands for each of the challenges:

```go
	rootCmd := &cobra.Command{Use: "advent-of-code-2023"}

	rootCmd.AddCommand(&cobra.Command{
		Use:   "day01part1",
		Short: "Run Day 01 Part 1",
		Run: func(cmd *cobra.Command, args []string) {
			day01.Part1()
		},
	})

	// ...

	if err := rootCmd.Execute(); err != nil {
		fmt.Println(err)
	}
```

Each subcommand is associated with a public function from the respective package.

Public functions in Go are always declared with PascalCase. Private functions use camelCase.

I've explicitly referenced the package namespace (`day01`) as there are numerous `Part01` functions declared across my packages.

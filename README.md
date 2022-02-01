# How-To-GO-ETH
An introduction to GO-Ethereum programming

# Installation

## Chocolatey

Go to Chocolatey.org and install the package manager: [https://chocolatey.org/install](https://chocolatey.org/install)

Basically, run this command in a terminal as ADMIN:

```
Set-ExecutionPolicy Bypass -Scope Process -Force; [System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072; iex ((New-Object System.Net.WebClient).DownloadString('https://community.chocolatey.org/install.ps1'))
```

## Git, Golang, Ming

```
C:\Windows\system32> choco install git
C:\Windows\system32> choco install golang
C:\Windows\system32> choco install mingw
```

## Clone Go-Ethereum repository

```
C:\Users\xxx> mkdir src\github.com\ethereum
C:\Users\xxx> git clone https://github.com/ethereum/go-ethereum src\github.com\ethereum\go-ethereum
C:\Users\xxx> cd src\github.com\ethereum\go-ethereum
```

## Download and resolve dependencies

```
C:\Users\xxx\go\src\github.com\ethereum\go-ethereum> go get -u -v golang.org/x/net/context
C:\Users\xxx\go\src\github.com\ethereum\go-ethereum> go mod tidy
```

or 
```
C:\Users\xxx\go\src\github.com\ethereum\go-ethereum> git submodule init
C:\Users\xxx\go\src\github.com\ethereum\go-ethereum> git submodule update
```

## Install Go-Ethereum
```
C:\Users\xxx\src\github.com\ethereum\go-ethereum> go install -v ./cmd/...
```

## Run Tests
```
C:\Users\xxx\src\github.com\ethereum\go-ethereum> go test -v ./...
```

# GETH

## Run a node?

```
geth
```

## Run the console & commands

Run on MainNet (or Rinkeby):

```
geth console (--rinkeby)
```

Stop console:

```
exit (or Ctrl+D)
```

# GO

## Init new project

```
go mod init [yourModule.name]
```

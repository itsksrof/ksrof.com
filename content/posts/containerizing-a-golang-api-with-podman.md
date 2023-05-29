---
title: "Containerizing a Golang API with Podman"
date: 2023-05-28T13:20:26+02:00
description: "Using Podman to develop an API with Golang and MySQL"
tags: ["podman", "golang", "mysql", "containers"]
categories: ["devops"]
draft: true
---

My way of developing personal software with Golang never goes far away from
using the provided tools from the language, and when needed I use third-party tools
that can either be installed using `go install` or by running a simple shell script.

Usually I leave links to the tools I've installed in the README of the project, that
way I'm able to *"reproduce"* the development environment when doing a fresh clone
of the repo.

The thing is that problems start to emerge the moment I share my code with another
individual, because yes, the tools needed are in the documentation, but when using
them I only took my system into consideration.

So, to avoid the unnecessary screaming and raging that occurs when trying to set up
a development environment on a different operating system, or even on a different
architecture, and the obnoxious *it works on my machine*. In this post I'll be
containerizing a little project to run isolatedly and in a reproducible environment.

## Going with Podman

The tool that always comes to my mind when thinking about containerizing an application,
is [Docker](https://docs.docker.com/get-started/overview/), I've been using it in pretty 
much every professional project I've ever worked on, and so far has worked greatly. The 
issue that I have with it is not a technical one, but rather a personal one, things have 
changed immensely, Docker is now a full-blown company, which isn't a bad thing, what sets 
me back is that I don't exactly understand the licensing on many of the tools that they provide. 
That alone made it worth it for me to go and explore other tools, and see if I could find a suitable 
replacement for it.

In my search I stumbled upon [Podman](https://docs.podman.io/en/latest/index.html) which
defines itself as a *"daemonless, open source, Linux native tool designed to make 
it easy to find, run, build, share and deploy applications using Open Containers Initiative (OCI)
Containers and Container Images."* and can be used as a drop-in replacement for the better-known
Docker runtime.

It sounded heavenly for me, so I thought it would be worth it to take it for a spin, and see the
process of containerizing a little API made with Golang and MySQL using Podman.

## Containerizing our application

Let's work on the assumption that we need to create a simple API to return books.

We have our `main.go` file which can look like this:

```go
package main

import (
	"database/sql"
	"encoding/json"
	"log"
	"net/http"

	_ "github.com/go-sql-driver/mysql"
)

type book struct {
	ID     uint   `json:"id"`
	Title  string `json:"title"`
	Author string `json:"author"`
}

func getBooks(db *sql.DB) (books []book, err error) {
	var book book

	rows, err := db.Query("SELECT * FROM local_db.books")
	if err != nil {
		return nil, err
	}
	defer rows.Close()

	for rows.Next() {
		if err := rows.Scan(
			&book.ID,
			&book.Title,
			&book.Author,
		); err != nil {
			return nil, err
		}

		books = append(books, book)
	}

	return books, nil
}

func getBooksHandler(db *sql.DB) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		books, err := getBooks(db)
		if err != nil {
			w.WriteHeader(http.StatusInternalServerError)
			json.NewEncoder(w).Encode(map[string]string{
				"error": err.Error(),
			})

			return
		}

		w.WriteHeader(http.StatusOK)
		json.NewEncoder(w).Encode(books)
	}
}

func main() {
	db, err := sql.Open("mysql", "root:pass@tcp(localhost)/local_db?parseTime=true")
	if err != nil {
		log.Fatalf("failed to open db connection: %v\n", err)
	}

	http.HandleFunc("/books", getBooksHandler(db))

	if err := http.ListenAndServe(":8080", nil); err != nil {
		log.Fatalf("failed to start http server: %v\n", err)
	}
}
```

And our `create-local-db.sql` which creates a new database schema and pre-seeds it with data:

```sql
DROP DATABASE IF EXISTS local_db;

CREATE DATABASE local_db;

USE local_db;

DROP TABLE IF EXISTS books;

CREATE TABLE books (
    id int unsigned NOT NULL AUTO_INCREMENT,
    title varchar(60) NOT NULL DEFAULT '',
    author varchar(60) NOT NULL DEFAULT '',
    PRIMARY KEY(id)
);

INSERT INTO books VALUES(1, '1984', 'George Orwell');
INSERT INTO books VALUES(2, 'Creativity, Inc', 'Ed Catmull');
INSERT INTO books VALUES(3, 'The Brothers Karamazov', 'Fyodor Dostoevsky');
```

With these two pieces in place, we can begin to containerize our application, we'll begin
by building the MySQL database server, and then we'll move on to building the Golang API.

### Setting up our database container

In-order to build a container, a special file is required, in that file we'll declare
the necessary steps that need to successfully run for our container to be built.

Usually the file that performs this heavy lifting is called a `Dockerfile`, but since we
are not using Docker, and I want to adhere to the OCI *(Open Container Initiative)* standards,
I will be naming it `Containerfile`. 

It looks like the following:

```Dockerfile
# Use the official mysql image to create a mysql instance.
# https://hub.docker.com/_/mysql
FROM mysql

# Copy the database definition to a temporal directory.
COPY ./create-local-db.sql /tmp

# Expose the port used by the mysql instance.
EXPOSE 3306

# Run the mysql server executable and read the database
# definition file at startup.
CMD ["mysqld", "--init-file=/tmp/create-local-db.sql"]
```

Now we can build our container image:

```bash
podman build -t mysql-container ./mysql # The path to your Containerfile
```

**INFO:** The `-t` flag or `--tag` allows us to specify a name for the resulting image.

And use it to initialize our MySQL database server:

```bash
podman run -d -e MYSQL_ROOT_PASSWORD=pass -p 3306:3306 mysql-container
```

**INFO:** The `-d` flag or `--detach` allows us to run the container in the background,
and prints the new container ID. The `-e` flag or `--env` allows us to set arbitrary environment
variables that are available for the process to be launched inside the container. The `-p` flag or
`--publish` allows us to publish a container's port, or range of ports, to the host.

And bam! if we go to our preferred MySQL client *(or through the mysql cli)* and connect to `localhost:3306`
as `root` using the password that we've specified in the `MYSQL_ROOT_PASSWORD` environment variable, you'll
be able to see the `local_db` schema pre-seeded with data.

### Setting up our golang container

For this part we'll also need a `Containerfile` in place to do the heavy lifting, and since
we already have one, it would be wise to move it to another directory, i.e. `/mysql`, and
leave the new one at the root. If you do so, don't forget to update the paths of the `Containerfile`
that has been moved, or move the files to the same directory.

The `Containerfile` to build our Golang container looks like this:

```Dockerfile
# Use the official golang image to create a binary.
# This is based on Debian and sets the GOPATH to /go.
# https://hub.docker.com/_/golang 
FROM golang:1.20-buster

# Create and change to the app directory.
WORKDIR /app

# Retrieve application dependencies.
# This allows the container build to reuse cached dependencies.
# Expecting to copy go.mod and if present go.sum.
COPY go.* ./
RUN go mod download

# Copy local code to the container image.
COPY *.go ./

# Build the binary.
RUN go build -v -o /go-api

# Expose the port used by the web service.
EXPOSE 8080

# Run the web service on container startup.
CMD ["/go-api"]
```

Now we can build our container image:

```bash
podman build -t golang-container ./Containerfile # The path to your Containerfile
```

And use it to initialize our Golang API:

```bash
podman run -d -p 8080:8080 golang-container
```

This time, there is no bam! Because what you are probably getting is an error that says
`connection refused` when trying to hit the `/books` endpoint.

Why? Think about what we just did, we've built and initialized two containers, one for
our MySQL database server, and the other for our Golang API, but they don't know a thing
about each other. If you were to put your IPv4 instead of `localhost` in the database connection 
string, it would work, since that range of ports is published in the host, but for our humble 
golang container `localhost` does not map to any database server.

### Creating a pod to share resources

A Pod is a group of one or more containers, with shared storage and network resources,
and a specification for how to run the containers. In our example we have a MySQL container
and a Golang container, but we would prefer not to bind the database to a routable network.
Using a pod, we could bind the `localhost` address of the pod and all containers in that
pod will be able to connect to it because of the shared network name space.

To create a pod we will need to know the names or id's of the containers that we wish
to run together. We can do so by running the following command:

```bash
podman ps
```

It will output to `stdout` something similar to this:

```txt
CONTAINER ID  IMAGE                              COMMAND               CREATED        STATUS        PORTS                   NAMES
c662d854f8fa  localhost/mysql-container:latest   mysqld --init-fil...  6 seconds ago  Up 6 seconds  0.0.0.0:3306->3306/tcp  admiring_nightingale
ea0d36dbb1a8  localhost/golang-container:latest  /go-api               1 second ago   Up 1 second   0.0.0.0:8080->8080/tcp  friendly_wescoff
```

With that information our next step will be to use the `podman generate kube` command, to
generate a Kubernetes YAML specification from podman containers, pods or volumes:

```bash
podman generate kube -s -f podman-compose.yaml admiring_nightingale friendly_wescoff
```

**INFO:** The `-s` flag or `--service` allows us to generate a Kubernetes service object
in addition to the pod. The `-f` flag or `--filename` allows us to pipe the output to a
given file instead of `stdout`.

You can name the file anything you want, I choose `podman-compose` because of the similarities
to `docker-compose`, that way other people can easily guess what it does. 

If we take a look inside, this is what we can expect to find:

```yaml
# Save the output of this file and use kubectl create -f to import
# it into Kubernetes.
#
# Created with podman-4.5.0

# NOTE: If you generated this yaml from an unprivileged and rootless podman container on an SELinux
# enabled system, check the podman generate kube man page for steps to follow to ensure that your pod/container
# has the right permissions to access the volumes added.
---
apiVersion: v1
kind: Service
metadata:
  creationTimestamp: "2023-05-28T19:58:30Z"
  labels:
    app: admiringnightingale-pod
  name: admiringnightingale-pod
spec:
  ports:
  - name: "3306"
    nodePort: 30195
    port: 3306
    targetPort: 3306
  - name: "8080"
    nodePort: 31682
    port: 8080
    targetPort: 8080
  selector:
    app: admiringnightingale-pod
  type: NodePort
---
apiVersion: v1
kind: Pod
metadata:
  annotations:
    io.podman.annotations.ulimit: nofile=524288:524288,nproc=127618:127618
    org.opencontainers.image.base.digest/admiringnightingale: sha256:13e429971e970ebcb7bc611de52d71a3c444247dc67cf7475a02718f
    org.opencontainers.image.base.digest/friendlywescoff: sha256:a33d71b6c2bbd1397b1e906c3f7df0a645477d7249f1f034513f1cf4
    org.opencontainers.image.base.name/admiringnightingale: docker.io/library/mysql:latest
    org.opencontainers.image.base.name/friendlywescoff: docker.io/library/golang:1.20-buster
  creationTimestamp: "2023-05-28T19:58:30Z"
  labels:
    app: admiringnightingale-pod
  name: admiringnightingale-pod
spec:
  containers:
  - args:
    - mysqld
    - --init-file=/tmp/create-local-db.sql
    env:
    - name: MYSQL_ROOT_PASSWORD
      value: pass
    image: localhost/mysql-container:latest
    name: admiringnightingale
    ports:
    - containerPort: 3306
    tty: true
    volumeMounts:
    - mountPath: /var/lib/mysql
      name: 10fc6631947b113462912416cb6ed5a014d0c171a12520acd3903ad5684fbdfb-pvc
  - image: localhost/golang-container:latest
    name: friendlywescoff
    ports:
    - containerPort: 8080
    tty: true
  volumes:
  - name: 10fc6631947b113462912416cb6ed5a014d0c171a12520acd3903ad5684fbdfb-pvc
    persistentVolumeClaim:
      claimName: 10fc6631947b113462912416cb6ed5a014d0c171a12520acd3903ad5684fbdfb
```

Impressive isn't it? With this new file we can safely remove the containers that
we've started using `podman rm -f <container-id>` and passing in the
id's of the containers. You can visualize that info by running `podman ps`.

And now, instead of instantiating each and every container separately we can
start everything that we need from our `podman-compose.yaml`:

```bash
podman play kube podman-compose.yaml
```

This will create a pod, and inside that pod it will initialise our containers,
that way they are able to share the resources available within the pod, this is
what allows our Golang API to connect to our MySQL database server, because now
`localhost` is binded to the pod, and thus it can be used by all the containers
of the pod.

Awesome! Let's see what we get when hitting the `/books` endpoint.

```json
[
	{
		"id": 1,
		"title": "1984",
		"author": "George Orwell"
	},
	{
		"id": 2,
		"title": "Creativity, Inc",
		"author": "Ed Catmull"
	},
	{
		"id": 3,
		"title": "The Brothers Karamazov",
		"author": "Fyodor Dostoevsky"
	}
]
```

You've did it! Time to pat yourself in the back, and tell your colleagues about
Podman, just kidding, but if you got this far, congratulations, you're now able
to run your Golang APIs in a containerized way, and that alone opens a whole lot
of possibilities and new paths to explore!

### Cleaning up before leaving

Since we already have our `podman-compose.yaml` which tells our containers how they
should be orchestrated, it is no surprise that we can use a simple command, to make them
yeet themselves out of existence:

```bash
podman kube down podman-compose.yaml
```

And if we also want to remove the images that we've created using `podman build`, we
can do so by first listing the images that we have available:

```bash
podman images
```

It will output to `stdout` something similar to this:

```txt
REPOSITORY                  TAG               IMAGE ID      CREATED       SIZE
localhost/mysql-container   latest            a26cf610bfef  11 hours ago  579 MB
localhost/golang-container  latest            db276ed7ca45  11 hours ago  817 MB
localhost/podman-pause      4.5.0-1681486942  8687708981da  20 hours ago  1.11 MB
docker.io/library/mysql     latest            05db07cd74c0  4 days ago    579 MB
docker.io/library/golang    1.20-buster       e92e76829029  5 days ago    742 MB
```

And then running `podman rmi <container-id>` with the id's
of the images that we wish to remove. Now if you were to run the `podman play kube`
command you'll be probably greeted with an error telling you that the image you want
to run cannot be found. Don't worry just go back to the building step and you'll be
fine. 

## Thoughts

Knowledge wise it has been really easy to transition from Docker to Podman, pretty much
every command that I was accustomed to was also available in Podman. Also a thing that
drew my attention was the fact that every Podman command can be run as a non-privileged
user, which allows us to limit its level of access within our operating system.

After tinkering with it this weekend, I'm left wanting to use it more and see how it
performs in a complexer project.

Thank you for your time, see you next week, ksrof out.

P.S. You can find the source code used in this article [here](https://github.com/itsksrof/go-podman-example).
# buildx bake

```
docker buildx bake [OPTIONS] [TARGET...]
```

<!---MARKER_GEN_START-->
Build from a file

### Aliases

`bake`, `f`

### Options

| Name | Description |
| --- | --- |
| `--builder string` | Override the configured builder instance |
| [`-f`](#file), [`--file stringArray`](#file) | Build definition file |
| `--load` | Shorthand for --set=*.output=type=docker |
| `--metadata-file string` | Write build result metadata to the file |
| [`--no-cache`](#no-cache) | Do not use cache when building the image |
| [`--print`](#print) | Print the options without building |
| [`--progress string`](#progress) | Set type of progress output (auto, plain, tty). Use plain to show container output |
| [`--pull`](#pull) | Always attempt to pull a newer version of the image |
| `--push` | Shorthand for --set=*.output=type=registry |
| [`--set stringArray`](#set) | Override target value (eg: targetpattern.key=value) |


<!---MARKER_GEN_END-->

## Description

Bake is a high-level build command. Each specified target will run in parallel
as part of the build.

Read [High-level build options](https://github.com/docker/buildx#high-level-build-options)
for introduction.

Please note that `buildx bake` command may receive backwards incompatible
features in the future if needed. We are looking for feedback on improving the
command and extending the functionality further.

## Examples

### <a name="file"></a> Specify a build definition file (-f, --file)

By default, `buildx bake` looks for build definition files in the current
directory, the following are parsed:

- `docker-compose.yml`
- `docker-compose.yaml`
- `docker-bake.json`
- `docker-bake.override.json`
- `docker-bake.hcl`
- `docker-bake.override.hcl`

Use the `-f` / `--file` option to specify the build definition file to use. The
file can be a Docker Compose, JSON or HCL file. If multiple files are specified
they are all read and configurations are combined.

The following example uses a Docker Compose file named `docker-compose.dev.yaml`
as build definition file, and builds all targets in the file:

```console
$ docker buildx bake -f docker-compose.dev.yaml

[+] Building 66.3s (30/30) FINISHED
 => [frontend internal] load build definition from Dockerfile  0.1s
 => => transferring dockerfile: 36B                            0.0s
 => [backend internal] load build definition from Dockerfile   0.2s
 => => transferring dockerfile: 3.73kB                         0.0s
 => [database internal] load build definition from Dockerfile  0.1s
 => => transferring dockerfile: 5.77kB                         0.0s
 ...
```

Pass the names of the targets to build, to build only specific target(s). The
following example builds the `backend` and `database` targets that are defined
in the `docker-compose.dev.yaml` file, skipping the build for the `frontend`
target:

```console
$ docker buildx bake -f docker-compose.dev.yaml backend database

[+] Building 2.4s (13/13) FINISHED
 => [backend internal] load build definition from Dockerfile  0.1s
 => => transferring dockerfile: 81B                           0.0s
 => [database internal] load build definition from Dockerfile 0.2s
 => => transferring dockerfile: 36B                           0.0s
 => [backend internal] load .dockerignore                     0.3s
 ...
```

You can also use a remote `git` bake definition:

```console
$ docker buildx bake "git://github.com/docker/cli#master" --print
#1 [internal] load git source git://github.com/docker/cli#master
#1 0.686 2776a6d694f988c0c1df61cad4bfac0f54e481c8       refs/heads/master
#1 CACHED
{
  "group": {
    "default": [
      "binary"
    ]
  },
  "target": {
    "binary": {
      "context": "git://github.com/docker/cli#master",
      "dockerfile": "Dockerfile",
      "args": {
        "BASE_VARIANT": "alpine",
        "GO_STRIP": "",
        "VERSION": ""
      },
      "target": "binary",
      "platforms": [
        "local"
      ],
      "output": [
        "build"
      ]
    }
  }
}
```

As you can see the context is fixed to `git://github.com/docker/cli` even if
[no context is actually defined](https://github.com/docker/cli/blob/2776a6d694f988c0c1df61cad4bfac0f54e481c8/docker-bake.hcl#L17-L26)
in the definition.

If you want to access the main context for bake command from a bake file
that has been imported remotely, you can use the `BAKE_CMD_CONTEXT` builtin var:

```console
$ cat https://raw.githubusercontent.com/tonistiigi/buildx/remote-test/docker-bake.hcl
target "default" {
  context = BAKE_CMD_CONTEXT
  dockerfile-inline = <<EOT
FROM alpine
WORKDIR /src
COPY . .
RUN ls -l && stop
EOT
}
```

```console
$ docker buildx bake "git://github.com/tonistiigi/buildx#remote-test" --print
{
  "group": {
    "default": [
      "default"
    ]
  },
  "target": {
    "default": {
      "context": ".",
      "dockerfile": "Dockerfile",
      "dockerfile-inline": "FROM alpine\nWORKDIR /src\nCOPY . .\nRUN ls -l \u0026\u0026 stop\n"
    }
  }
}
```

```console
$ touch foo bar
$ docker buildx bake "git://github.com/tonistiigi/buildx#remote-test"
...
 > [4/4] RUN ls -l && stop:
#8 0.101 total 0
#8 0.102 -rw-r--r--    1 root     root             0 Jul 27 18:47 bar
#8 0.102 -rw-r--r--    1 root     root             0 Jul 27 18:47 foo
#8 0.102 /bin/sh: stop: not found
```

```console
$ docker buildx bake "git://github.com/tonistiigi/buildx#remote-test" "git://github.com/docker/cli#master" --print
#1 [internal] load git source git://github.com/tonistiigi/buildx#remote-test
#1 0.401 577303add004dd7efeb13434d69ea030d35f7888       refs/heads/remote-test
#1 CACHED
{
  "group": {
    "default": [
      "default"
    ]
  },
  "target": {
    "default": {
      "context": "git://github.com/docker/cli#master",
      "dockerfile": "Dockerfile",
      "dockerfile-inline": "FROM alpine\nWORKDIR /src\nCOPY . .\nRUN ls -l \u0026\u0026 stop\n"
    }
  }
}
```

```console
$ docker buildx bake "git://github.com/tonistiigi/buildx#remote-test" "git://github.com/docker/cli#master"
...
 > [4/4] RUN ls -l && stop:
#8 0.136 drwxrwxrwx    5 root     root          4096 Jul 27 18:31 kubernetes
#8 0.136 drwxrwxrwx    3 root     root          4096 Jul 27 18:31 man
#8 0.136 drwxrwxrwx    2 root     root          4096 Jul 27 18:31 opts
#8 0.136 -rw-rw-rw-    1 root     root          1893 Jul 27 18:31 poule.yml
#8 0.136 drwxrwxrwx    7 root     root          4096 Jul 27 18:31 scripts
#8 0.136 drwxrwxrwx    3 root     root          4096 Jul 27 18:31 service
#8 0.136 drwxrwxrwx    2 root     root          4096 Jul 27 18:31 templates
#8 0.136 drwxrwxrwx   10 root     root          4096 Jul 27 18:31 vendor
#8 0.136 -rwxrwxrwx    1 root     root          9620 Jul 27 18:31 vendor.conf
#8 0.136 /bin/sh: stop: not found
```

### <a name="no-cache"></a> Do not use cache when building the image (--no-cache)

Same as `build --no-cache`. Do not use cache when building the image.

### <a name="print"></a> Print the options without building (--print)

Prints the resulting options of the targets desired to be built, in a JSON
format, without starting a build.

```console
$ docker buildx bake -f docker-bake.hcl --print db
{
  "group": {
    "default": [
      "db"
    ]
  },
  "target": {
    "db": {
      "context": "./",
      "dockerfile": "Dockerfile",
      "tags": [
        "docker.io/tiborvass/db"
      ]
    }
  }
}
```

### <a name="progress"></a> Set type of progress output (--progress)

Same as [`build --progress`](buildx_build.md#progress). Set type of progress
output (auto, plain, tty). Use plain to show container output (default "auto").

> You can also use the `BUILDKIT_PROGRESS` environment variable to set its value.

The following example uses `plain` output during the build:

```console
$ docker buildx bake --progress=plain

#2 [backend internal] load build definition from Dockerfile.test
#2 sha256:de70cb0bb6ed8044f7b9b1b53b67f624e2ccfb93d96bb48b70c1fba562489618
#2 ...

#1 [database internal] load build definition from Dockerfile.test
#1 sha256:453cb50abd941762900a1212657a35fc4aad107f5d180b0ee9d93d6b74481bce
#1 transferring dockerfile: 36B done
#1 DONE 0.1s
...
```


### <a name="pull"></a> Always attempt to pull a newer version of the image (--pull)

Same as `build --pull`.

### <a name="set"></a> Override target configurations from command line (--set)

```
--set targetpattern.key[.subkey]=value
```

Override target configurations from command line. The pattern matching syntax
is defined in https://golang.org/pkg/path/#Match.


**Examples**

```console
$ docker buildx bake --set target.args.mybuildarg=value
$ docker buildx bake --set target.platform=linux/arm64
$ docker buildx bake --set foo*.args.mybuildarg=value # overrides build arg for all targets starting with 'foo'
$ docker buildx bake --set *.platform=linux/arm64     # overrides platform for all targets
$ docker buildx bake --set foo*.no-cache              # bypass caching only for targets starting with 'foo'
```

Complete list of overridable fields:
`args`, `cache-from`, `cache-to`, `context`, `dockerfile`, `labels`, `no-cache`,
`output`, `platform`, `pull`, `secrets`, `ssh`, `tags`, `target`

### File definition

In addition to compose files, bake supports a JSON and an equivalent HCL file
format for defining build groups and targets.

A target reflects a single docker build invocation with the same options that
you would specify for `docker build`. A group is a grouping of targets.

Multiple files can include the same target and final build options will be
determined by merging them together.

In the case of compose files, each service corresponds to a target.

A group can specify its list of targets with the `targets` option. A target can
inherit build options by setting the `inherits` option to the list of targets or
groups to inherit from.

Note: Design of bake command is work in progress, the user experience may change
based on feedback.


**Example HCL definition**

```hcl
group "default" {
    targets = ["db", "webapp-dev"]
}

target "webapp-dev" {
    dockerfile = "Dockerfile.webapp"
    tags = ["docker.io/username/webapp"]
}

target "webapp-release" {
    inherits = ["webapp-dev"]
    platforms = ["linux/amd64", "linux/arm64"]
}

target "db" {
    dockerfile = "Dockerfile.db"
    tags = ["docker.io/username/db"]
}
```

Complete list of valid target fields:

`args`, `cache-from`, `cache-to`, `context`, `dockerfile`, `inherits`, `labels`,
`no-cache`, `output`, `platform`, `pull`, `secrets`, `ssh`, `tags`, `target`

### Global scope attributes

You can define global scope attributes in HCL/JSON and use them for code reuse
and setting values for variables. This means you can do a "data-only" HCL file
with the values you want to set/override and use it in the list of regular
output files.

```hcl
# docker-bake.hcl
variable "FOO" {
    default = "abc"
}

target "app" {
    args = {
        v1 = "pre-${FOO}"
    }
}
```

You can use this file directly:

```console
$ docker buildx bake --print app
{
  "group": {
    "default": [
      "app"
    ]
  },
  "target": {
    "app": {
      "context": ".",
      "dockerfile": "Dockerfile",
      "args": {
        "v1": "pre-abc"
      }
    }
  }
}
```

Or create an override configuration file:

```hcl
# env.hcl
WHOAMI="myuser"
FOO="def-${WHOAMI}"
```

And invoke bake together with both of the files:

```console
$ docker buildx bake -f docker-bake.hcl -f env.hcl --print app
{
  "group": {
    "default": [
      "app"
    ]
  },
  "target": {
    "app": {
      "context": ".",
      "dockerfile": "Dockerfile",
      "args": {
        "v1": "pre-def-myuser"
      }
    }
  }
}
```

### HCL variables and functions

Similar to how Terraform provides a way to [define variables](https://www.terraform.io/docs/configuration/variables.html#declaring-an-input-variable),
the HCL file format also supports variable block definitions. These can be used
to define variables with values provided by the current environment, or a
default value when unset.

A [set of generally useful functions](https://github.com/docker/buildx/blob/master/bake/hclparser/stdlib.go)
provided by [go-cty](https://github.com/zclconf/go-cty/tree/main/cty/function/stdlib)
are available for use in HCL files. In addition, [user defined functions](https://github.com/hashicorp/hcl/tree/main/ext/userfunc)
are also supported.

#### Using interpolation to tag an image with the git sha

Bake supports variable blocks which are assigned to matching environment
variables or default values.

```hcl
# docker-bake.hcl
variable "TAG" {
    default = "latest"
}

group "default" {
    targets = ["webapp"]
}

target "webapp" {
    tags = ["docker.io/username/webapp:${TAG}"]
}
```

```console
$ docker buildx bake --print webapp
{
  "group": {
    "default": [
      "webapp"
    ]
  },
  "target": {
    "webapp": {
      "context": ".",
      "dockerfile": "Dockerfile",
      "tags": [
        "docker.io/username/webapp:latest"
      ]
    }
  }
}
```

```console
$ TAG=$(git rev-parse --short HEAD) docker buildx bake --print webapp
{
  "group": {
    "default": [
      "webapp"
    ]
  },
  "target": {
    "webapp": {
      "context": ".",
      "dockerfile": "Dockerfile",
      "tags": [
        "docker.io/username/webapp:985e9e9"
      ]
    }
  }
}
```

#### Using the `add` function

You can use [`go-cty` stdlib functions]([go-cty](https://github.com/zclconf/go-cty/tree/main/cty/function/stdlib)).
Here we are using the `add` function.

```hcl
# docker-bake.hcl
variable "TAG" {
    default = "latest"
}

group "default" {
    targets = ["webapp"]
}

target "webapp" {
    args = {
        buildno = "${add(123, 1)}"
    }
}
```

```console
$ docker buildx bake --print webapp
{
  "group": {
    "default": [
      "webapp"
    ]
  },
  "target": {
    "webapp": {
      "context": ".",
      "dockerfile": "Dockerfile",
      "args": {
        "buildno": "124"
      }
    }
  }
}
```

#### Defining an `increment` function

It also supports [user defined functions](https://github.com/hashicorp/hcl/tree/main/ext/userfunc).
The following example defines a simple an `increment` function.

```hcl
# docker-bake.hcl
function "increment" {
    params = [number]
    result = number + 1
}

group "default" {
    targets = ["webapp"]
}

target "webapp" {
    args = {
        buildno = "${increment(123)}"
    }
}
```

```console
$ docker buildx bake --print webapp
{
  "group": {
    "default": [
      "webapp"
    ]
  },
  "target": {
    "webapp": {
      "context": ".",
      "dockerfile": "Dockerfile",
      "args": {
        "buildno": "124"
      }
    }
  }
}
```

#### Only adding tags if a variable is not empty using an `notequal`

Here we are using the conditional `notequal` function which is just for
symmetry with the `equal` one.

```hcl
# docker-bake.hcl
variable "TAG" {default="" }

group "default" {
    targets = [
        "webapp",
    ]
}

target "webapp" {
    context="."
    dockerfile="Dockerfile"
    tags = [
        "my-image:latest",
        notequal("",TAG) ? "my-image:${TAG}": "",
    ]
}
```

```console
$ docker buildx bake --print webapp
{
  "group": {
    "default": [
      "webapp"
    ]
  },
  "target": {
    "webapp": {
      "context": ".",
      "dockerfile": "Dockerfile",
      "tags": [
        "my-image:latest"
      ]
    }
  }
}
```

#### Using variables in functions

You can refer variables to other variables like the target blocks can. Stdlib
functions can also be called but user functions can't at the moment.

```hcl
# docker-bake.hcl
variable "REPO" {
    default = "user/repo"
}

function "tag" {
    params = [tag]
    result = ["${REPO}:${tag}"]
}

target "webapp" {
    tags = tag("v1")
}
```

```console
$ docker buildx bake --print webapp
{
  "group": {
    "default": [
      "webapp"
    ]
  },
  "target": {
    "webapp": {
      "context": ".",
      "dockerfile": "Dockerfile",
      "tags": [
        "user/repo:v1"
      ]
    }
  }
}
```

#### Using variables in variables across files

When multiple files are specified, one file can use variables defined in
another file.

```hcl
# docker-bake1.hcl
variable "FOO" {
    default = upper("${BASE}def")
}

variable "BAR" {
    default = "-${FOO}-"
}

target "app" {
    args = {
        v1 = "pre-${BAR}"
    }
}
```

```hcl
# docker-bake2.hcl
variable "BASE" {
    default = "abc"
}

target "app" {
    args = {
        v2 = "${FOO}-post"
    }
}
```

```console
$ docker buildx bake -f docker-bake1.hcl -f docker-bake2.hcl --print app
{
  "group": {
    "default": [
      "app"
    ]
  },
  "target": {
    "app": {
      "context": ".",
      "dockerfile": "Dockerfile",
      "args": {
        "v1": "pre--ABCDEF-",
        "v2": "ABCDEF-post"
      }
    }
  }
}
```

#### Using typed variables

Non-string variables are also accepted. The value passed with env is parsed
into suitable type first.

```hcl
# docker-bake.hcl
variable "FOO" {
    default = 3
}

variable "IS_FOO" {
    default = true
}

target "app" {
    args = {
        v1 = FOO > 5 ? "higher" : "lower" 
        v2 = IS_FOO ? "yes" : "no"
    }
}
```

```console
$ docker buildx bake --print app
{
  "group": {
    "default": [
      "app"
    ]
  },
  "target": {
    "app": {
      "context": ".",
      "dockerfile": "Dockerfile",
      "args": {
        "v1": "lower",
        "v2": "yes"
      }
    }
  }
}
```

### Extension field with Compose

[Special extension](https://github.com/compose-spec/compose-spec/blob/master/spec.md#extension)
field `x-bake` can be used in your compose file to evaluate fields that are not
(yet) available in the [build definition](https://github.com/compose-spec/compose-spec/blob/master/build.md#build-definition).

```yaml
# docker-compose.yml
services:
  addon:
    image: ct-addon:bar
    build:
      context: .
      dockerfile: ./Dockerfile
      args:
        CT_ECR: foo
        CT_TAG: bar
      x-bake:
        tags:
          - ct-addon:foo
          - ct-addon:alp
        platforms:
          - linux/amd64
          - linux/arm64
        cache-from:
          - user/app:cache
          - type=local,src=path/to/cache
        cache-to: type=local,dest=path/to/cache
        pull: true

  aws:
    image: ct-fake-aws:bar
    build:
      dockerfile: ./aws.Dockerfile
      args:
        CT_ECR: foo
        CT_TAG: bar
      x-bake:
        secret:
          - id=mysecret,src=./secret
          - id=mysecret2,src=./secret2
        platforms: linux/arm64
        output: type=docker
        no-cache: true
```

```console
$ docker buildx bake --print
{
  "target": {
    "addon": {
      "context": ".",
      "dockerfile": "./Dockerfile",
      "args": {
        "CT_ECR": "foo",
        "CT_TAG": "bar"
      },
      "tags": [
        "ct-addon:foo",
        "ct-addon:alp"
      ],
      "cache-from": [
        "user/app:cache",
        "type=local,src=path/to/cache"
      ],
      "cache-to": [
        "type=local,dest=path/to/cache"
      ],
      "platforms": [
        "linux/amd64",
        "linux/arm64"
      ],
      "pull": true
    },
    "aws": {
      "context": ".",
      "dockerfile": "./aws.Dockerfile",
      "args": {
        "CT_ECR": "foo",
        "CT_TAG": "bar"
      },
      "tags": [
        "ct-fake-aws:bar"
      ],
      "secret": [
        "id=mysecret,src=./secret",
        "id=mysecret2,src=./secret2"
      ],
      "platforms": [
        "linux/arm64"
      ],
      "output": [
        "type=docker"
      ],
      "no-cache": true
    }
  }
}
```

Complete list of valid fields for `x-bake`:

`tags`, `cache-from`, `cache-to`, `secret`, `ssh`, `platforms`, `output`,
`pull`, `no-cache`

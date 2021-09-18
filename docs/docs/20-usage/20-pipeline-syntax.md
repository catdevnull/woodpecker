# Pipeline syntax

The pipeline section defines a list of steps to build, test and deploy your code. Pipeline steps are executed serially, in the order in which they are defined. If a step returns a non-zero exit code, the pipeline immediately aborts and returns a failure status.

Example pipeline:

```yaml
pipeline:
  backend:
    image: golang
    commands:
      - go build
      - go test
  frontend:
    image: node
    commands:
      - npm install
      - npm run test
      - npm run build
```

In the above example we define two pipeline steps, `frontend` and `backend`. The names of these steps are completely arbitrary.

## Steps

Every step of your pipeline executes arbitrary commands inside a specified docker container. The defined commands are executed serially.
The associated commit of a current pipeline run is checked out with git to a workspace which is mounted to every step of the pipeline as the working directory.

```diff
pipeline:
  backend:
    image: golang
    commands:
+     - go build
+     - go test
```

## Step `image`

Woodpecker uses Docker images for the build environment, for plugins and for service containers. The image field is exposed in the container blocks in the Yaml:

```diff
pipeline:
  build:
+   image: golang:1.6
    commands:
      - go build
      - go test

  publish:
+   image: plugins/docker
    repo: foo/bar

services:
  database:
+   image: mysql
```

Woodpecker supports any valid Docker image from any Docker registry:

```text
image: golang
image: golang:1.7
image: library/golang:1.7
image: index.docker.io/library/golang
image: index.docker.io/library/golang:1.7
```

Woodpecker does not automatically upgrade docker images. Example configuration to always pull the latest image when updates are available:

```diff
pipeline:
  build:
    image: golang:latest
+   pull: true
```

#### Images from private registries

You must provide registry credentials on the UI in order to pull private pipeline images defined in your Yaml configuration file.

These credentials are never exposed to your pipeline, which means they cannot be used to push, and are safe to use with pull requests, for example. Pushing to a registry still require setting credentials for the appropriate plugin.

Example configuration using a private image:

```diff
pipeline:
  build:
+   image: gcr.io/custom/golang
    commands:
      - go build
      - go test
```

Woodpecker matches the registry hostname to each image in your yaml. If the hostnames match, the registry credentials are used to authenticate to your registry and pull the image. Note that registry credentials are used by the Woodpecker agent and are never exposed to your build containers.

Example registry hostnames:

- Image `gcr.io/foo/bar` has hostname `gcr.io`
- Image `foo/bar` has hostname `docker.io`
- Image `qux.com:8000/foo/bar` has hostname `qux.com:8000`

Example registry hostname matching logic:

- Hostname `gcr.io` matches image `gcr.io/foo/bar`
- Hostname `docker.io` matches `golang`
- Hostname `docker.io` matches `library/golang`
- Hostname `docker.io` matches `bradyrydzewski/golang`
- Hostname `docker.io` matches `bradyrydzewski/golang:latest`

#### Global registry support

To make a private registry globally available check the [server configuration docs](/docs/administration/server-config#global-registry-setting).

#### GCR registry support

For specific details on configuring access to Google Container Registry, please view the docs [here](https://cloud.google.com/container-registry/docs/advanced-authentication#using_a_json_key_file).

## Step `commands`

Commands of every pipeline step are executed serially as if you would enter them into your local shell.

```diff
pipeline:
  backend:
    image: golang
    commands:
+     - go build
+     - go test
```

There is no magic here. The above commands are converted to a simple shell script. The commands in the above example are roughly converted to the below script:

```diff
#!/bin/sh
set -e

go build
go test
```

The above shell script is then executed as the docker entrypoint. The below docker command is an (incomplete) example of how the script is executed:

```
docker run --entrypoint=build.sh golang
```

> Please note that only build steps can define commands. You cannot use commands with plugins or services.

## Step `environment`

Woodpecker provides the ability to pass environment variables to individual pipeline steps.

For more details check the [environment docs](/docs/usage/environment/).

## Step `secrets`

Woodpecker provides the ability to store named parameters external to the Yaml configuration file, in a central secret store. These secrets can be passed to individual steps of the pipeline at runtime.

For more details check the [secrets docs](/docs/usage/secrets/).

## Step `when` - Conditional Execution

Woodpecker supports defining conditional pipeline steps in the `when` block. If all conditions in the `when` block evaluate to true the step is executed, otherwise it is skipped.

### `branch`

Example conditional execution by branch:

```diff
pipeline:
  slack:
    image: plugins/slack
    channel: dev
+   when:
+     branch: master
```

> The step now triggers on master, but also if the target branch of a pull request is `master`. Add an event condition to limit it further to pushes on master only.

Execute a step if the branch is `master` or `develop`:

```diff
when:
  branch: [master, develop]
```

Execute a step if the branch starts with `prefix/*`:

```diff
when:
  branch: prefix/*
```

Execute a step using custom include and exclude logic:

```diff
when:
  branch:
    include: [ master, release/* ]
    exclude: [ release/1.0.0, release/1.1.* ]
```

### `event`

Execute a step if the build event is a `tag`:

```diff
when:
  event: tag
```

Execute a step if the build event is a `tag` created from the specified branch:

```diff
when:
  event: tag
+ branch: master
```

Execute a step for all non-pull request events:

```diff
when:
  event: [push, tag, deployment]
```

Execute a step for all build events:

```diff
when:
  event: [push, pull_request, tag, deployment]
```

### `tag`

Execute a step if the tag name starts with `release`:

```diff
when:
  tag: release*
```

### `status`

Execute a step when the build status changes:

```diff
when:
  status: changed
```

Woodpecker uses the container exit code to determine the success or failure status of a build. Non-zero exit codes fail the build and cause the pipeline to immediately exit.

There are use cases for executing pipeline steps on failure, such as sending notifications for failed builds. Use the status constraint to override the default behavior and execute steps even when the build status is failure:

Execute a step when the build is passing or failing:

```diff
pipeline:
  slack:
    image: plugins/slack
    channel: dev
+   when:
+     status: [ success, failure ]
```

### `platform`

Execute a step for a specific platform:

```diff
when:
  platform: linux/amd64
```

Execute a step for a specific platform using wildcards:

```diff
when:
  platform:  [ linux/*, windows/amd64 ]
```

### `environment`

Execute a step for deployment events matching the target deployment environment:

```diff
when:
  environment: production
  event: deployment
```

### `matrix`

Execute a step for a single matrix permutation:

```diff
when:
  matrix:
    GO_VERSION: 1.5
    REDIS_VERSION: 2.8
```

### `instance`

Execute a step only on a certain Woodpecker instance matching the specified hostname:

```diff
when:
  instance: stage.woodpecker.company.com
```

### `path`

Execute a step only on commit with certain files added/removed/modified:

> NOTE: Feature is only available for GitHub and Gitea repositories.

```diff
when:
  path: "src/*"
```

Execute a step only on commit excluding certain files added/removed/modified:


> NOTE: Feature is only available for GitHub and Gitea repositories.

```diff
when:
  path:
    exclude: [ '*.md', '*.ini' ]
    ignore_message: "[ALL]"
```

** Note for `path` conditions: passing `[ALL]` inside the commit message will ignore all path conditions. **

## Step `group` - Parallel execution

Woodpecker supports parallel step execution for same-machine fan-in and fan-out. Parallel steps are configured using the `group` attribute. This instructs the pipeline runner to execute the named group in parallel.

Example parallel configuration:

```diff
pipeline:
  backend:
+   group: build
    image: golang
    commands:
      - go build
      - go test
  frontend:
+   group: build
    image: node
    commands:
      - npm install
      - npm run test
      - npm run build
  publish:
    image: plugins/docker
    repo: octocat/hello-world
```

In the above example, the `frontend` and `backend` steps are executed in parallel. The pipeline runner will not execute the `publish` step until the group completes.

## Step `volumes`

Woodpecker gives the ability to define Docker volumes in the Yaml. You can use this parameter to mount files or folders on the host machine into your containers.

For more details check the [volumes docs](/docs/usage/volumes/).

## `branches`

Woodpecker gives the ability to skip commits based on the target branch. If the branch matches the `branches:` block the pipeline is executed, otherwise it is skipped.

Example skipping a commit when the target branch is not master:

```diff
pipeline:
  build:
    image: golang
    commands:
      - go build
      - go test

+branches: master
```

Example matching multiple target branches:

```diff
pipeline:
  build:
    image: golang
    commands:
      - go build
      - go test

+branches: [ master, develop ]
```

Example uses glob matching:

```diff
pipeline:
  build:
    image: golang
    commands:
      - go build
      - go test

+branches: [ master, feature/* ]
```

Example includes branches:

```diff
pipeline:
  build:
    image: golang
    commands:
      - go build
      - go test

+branches:
+  include: [ master, feature/* ]
```

Example excludes branches:

```diff
pipeline:
  build:
    image: golang
    commands:
      - go build
      - go test

+branches:
+  exclude: [ develop, feature/* ]
```

## `workspace`

The workspace defines the shared volume and working directory shared by all pipeline steps. The default workspace matches the below pattern, based on your repository url.

```
/drone/src/github.com/octocat/hello-world
```

The workspace can be customized using the workspace block in the Yaml file:

```diff
+workspace:
+  base: /go
+  path: src/github.com/octocat/hello-world

pipeline:
  build:
    image: golang:latest
    commands:
      - go get
      - go test
```

The base attribute defines a shared base volume available to all pipeline steps. This ensures your source code, dependencies and compiled binaries are persisted and shared between steps.

```diff
workspace:
+ base: /go
  path: src/github.com/octocat/hello-world

pipeline:
  deps:
    image: golang:latest
    commands:
      - go get
      - go test
  build:
    image: node:latest
    commands:
      - go build
```

This would be equivalent to the following docker commands:

```
docker volume create my-named-volume

docker run --volume=my-named-volume:/go golang:latest
docker run --volume=my-named-volume:/go node:latest
```

The path attribute defines the working directory of your build. This is where your code is cloned and will be the default working directory of every step in your build process. The path must be relative and is combined with your base path.

```diff
workspace:
  base: /go
+ path: src/github.com/octocat/hello-world
```

```text
git clone https://github.com/octocat/hello-world \
  /go/src/github.com/octocat/hello-world
```

## `matrix`

Woodpecker has integrated support for matrix builds. Woodpecker executes a separate build task for each combination in the matrix, allowing you to build and test a single commit against multiple configurations.

For more details check the [matrix build docs](/docs/usage/matrix-builds/).

## `clone`

Woodpecker automatically configures a default clone step if not explicitly defined. You can manually configure the clone step in your pipeline for customization:

```diff
+clone:
+  git:
+    image: plugins/git

pipeline:
  build:
    image: golang
    commands:
      - go build
      - go test
```

Example configuration to override depth:

```diff
clone:
  git:
    image: plugins/git
+   depth: 50
```

Example configuration to use a custom clone plugin:

```diff
clone:
  git:
+   image: octocat/custom-git-plugin
```

Example configuration to clone Mercurial repository:

```diff
clone:
  hg:
+   image: plugins/hg
+   path: bitbucket.org/foo/bar
```

### Git Submodules

To use the credentials that cloned the repository to clone it's submodules, update `.gitmodules` to use `https` instead of `git`:

```diff
[submodule "my-module"]
	path = my-module
-	url = git@github.com:octocat/my-module.git
+	url = https://github.com/octocat/my-module.git
```

To use the ssh git url in `.gitmodules` for users cloning with ssh, and also use the https url in Woodpecker, add `submodule_override`:

```diff
clone:
  git:
    image: plugins/git
    recursive: true
+   submodule_override:
+     my-module: https://github.com/octocat/my-module.git

pipeline:
  ...
```

## `services`

Woodpecker can provide service containers. They can for example be used to run databases or cache containers during the execution of pipeline.

For more details check the [services docs](/docs/usage/services/).

## Skip Commits

Woodpecker gives the ability to skip individual commits by adding `[CI SKIP]` to the commit message. Note this is case-insensitive.

```diff
git commit -m "updated README [CI SKIP]"
```

## Privileged mode

Woodpecker gives the ability to configure privileged mode in the Yaml. You can use this parameter to launch containers with escalated capabilities.

> Privileged mode is only available to trusted repositories and for security reasons should only be used in private environments. See [project settings](/docs/usage/project-settings#trusted) to enable trusted mode.

```diff
pipeline:
  build:
    image: docker
    environment:
      - DOCKER_HOST=tcp://docker:2375
    commands:
      - docker --tls=false ps

services:
  docker:
    image: docker:dind
    command: [ "--storage-driver=vfs", "--tls=false" ]
+   privileged: true
```
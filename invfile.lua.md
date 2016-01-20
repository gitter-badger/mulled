---
papersize: a4paper
documentclass: article
title: Mulled
subtitle: Containerized Packages in the Cloud
author:
  - Jonas Weber
  - Björn Grüning
---

# Introduction

Docker\textregistered[^docker-trademark] provides a way to containerize
applications in portable images that contain all dependencies needed to use the
program. Determining the dependencies (libraries, interpreters etc.) is however
a non-trivial task, as there is no standardized way for developers to specify
them.

Package managers such as Conda or Homebrew were established out of this need,
providing a huge number of package descriptions that list the steps needed to
build the software and the dependencies it needs. Packages can then be
installed from those repositories, provided that relevant compilers and tools
are present on the host system.

Mulled fetches build instructions from such package managers, and compiles
applications ahead-of-time in the cloud. Resulting images are made available
in public image repositories.

-- TODO
The remainder of this article is structured as follows: Firstly, we introduce
the builders (one for each enabled package manager). After that, we describe
how the build system determines which packages need to be rebuilt.

This article is written in a special way: The article is written in Markdown,
and the code samples shown throughout the article constitute the complete
source code for the software.

# Architecture

Mulled uses Involucro[^involucro] to execute Docker containers and wrap
directories into images.

# Preliminary Steps

Firstly, we define some settings. These name utility images used later:

    curl = 'appropriate/curl'
    jq = 'local_tools/jq'

We also determine where the results will be stored:

    quay_prefix = 'mulled'
    namespace = 'quay.io/' .. quay_prefix
    github_repo = 'mulled/api'

    current_build_id = 'snapshot'

The Lua version used in Involucro doesn't provide native support for splitting
a `string` by a delimiter. This utility function fills that gap:

    function split(s, delim)
      local start = 1
      local t = {}
      while true do
        local pos = string.find(s, delim, start, true)
        if not pos then break end
        table.insert(t, string.sub(s, start, pos - 1))
        start = pos + #delim
      end
      table.insert(t, string.sub(s, start))
      return t
    end

    assert(table.concat(split("a-d", "-"), "/")   == "a/d")
    assert(table.concat(split("a..d", ".."), "/") == "a/d")

# Builders

Builders are functions that get a package specification and create tasks of the
form `type:package`, where `type` is one of `build`, `test` and `clean`.  These
builders are stored in a Lua table, appropriately called `builders`:

    builders = {}

The key names from this table reappear in the `packages.tsv` file in the
left-most column.

## Alpine

Alpine Linux[^alpine-linux] is "a security-oriented, lightweight Linux
distribution based on musl libc and busybox" (from their homepage). Packages
are provided in binary format, compiled with the `musl libc` (instead of the
more common `glibc`).



    builders.alpine = function (package, revision, test, builddir)
      local repo = namespace .. '/' .. package .. ':' .. revision

Alpine provides a tool called `apk` that manages installation of packages. The
options set in the command below install the package into the directory
`/data/dist`, after updating the cache from the given URL, but using the keys
from the 'host' installation. Otherwise, the repository signatures wouldn't be
checkable.

      local install = 'apk --root /data/dist'
      .. ' --update-cache --repository '
      .. ' http://dl-4.alpinelinux.org/alpine/v3.2/main'
      .. ' --keys-dir /etc/apk/keys --initdb add '

After installation we have to extract some information from the package
manager, namely the installed version, a description and the upstream homepage.
We can get the information from `apk` with `apkg info`, again specifying the
root of our installation.  `-wd` causes `apk` to print the web page and
description of the package.

Unfortunately, the format `apk` provides is not ideal for programmatic consumption.
Using `-wd` we get the following:

```
musl-1.1.11-r2 description:
the musl c library (libc) implementation

musl-1.1.11-r2 webpage:
http://www.musl-libc.org/
```

A POSIX shell allows consuming the same input stream with multiple tools by
combining them with parentheses. The `read` tool reads one line and assigns it
to the environment variable named by it's parameter. We assume the following:
In the second line we can find the package name followed by version and release
counter, in the third line follows the description, and in the sixth the
homepage:

    extractInfo = 'apk info --root /data/dist -wd  ' .. package | ('
      .. ' read fline ;'
      .. ' echo $fline | cut -d" " -f1 | cut -d"-" -f 2-3 > /data/info/version; '
      .. ' read desc; echo  $desc > /data/info/description ; '
      .. ' read discard ; '
      .. ' read discard ; '
      .. ' read homepage ; echo $homepage > /data/info/homepage ;'
      .. ')'



      inv.task('build:' .. package)
        .using('alpine')
          .withConfig({entrypoint = {"/bin/sh", "-c"}})
          .withHostConfig({
            binds = {builddir .. ':/data'}
          })
          .run('mkdir -p /data/dist /data/info')
          .run(install .. package .. ' && ' .. extractInfo)
          .run('rm -rf /data/dist/lib/apk /data/dist/var/cache/apk/')
          .wrap(builddir .. '/dist').at('/').inImage('busybox:latest')
            .as(repo)

      inv.task('clean:' .. package)
        .using('alpine')
          .withHostConfig({binds = {builddir .. ':/data'}})
          .run('rm', '-rf', '/data/dist', '/data/info')

      inv.task('test:' .. package)
        .using(repo)
        .withConfig({entrypoint = {'/bin/sh', '-c'}})
        .run(test)
    end

# Linuxbrew

    builders.linuxbrew = function (package, revision, test, builddir)
      repo = namespace .. '/' .. package .. ':' .. revision

      extractInfo = '$BREW info --json=v1 ' .. package ..
    [==[ | jq --raw-output '.[0] | [.homepage, .desc, .versions.stable] | join("\n")' | (
      read homepage ; echo $homepage > /info/homepage ;
      read desc ; echo $desc > /info/description ;
      read version ; echo $version > /info/version ;
    ) ]==]

      inv.task('build:' .. package)
        .using('thriqon/linuxbrew-alpine')
          .withConfig({user = "root"})
          .withHostConfig({
            binds = {builddir .. ':/data'}
          })
          .run('mkdir', '-p', '/data/info', '/data/dist')
          .run('chown', 'nobody', '/data/info', '/data/dist/')
          .withConfig({user = "nobody"})
          .run('mkdir', '/data/dist/bin', '/data/dist/Cellar')
          .withConfig({
            user = "nobody",
            entrypoint = {"/bin/sh", "-c"},
            env = {
              "BREW=/brew/orig_bin/brew",
              "HOME=/tmp"
            }
          })
          .withHostConfig({binds = {
            builddir .. "/dist/bin:/brew/bin",
            builddir .. "/dist/Cellar:/brew/Cellar",
            builddir .. "/info:/info"
          }})
          .run('$BREW install ' .. package)
          .run('$BREW test ' .. package)
          .run(extractInfo)
        .wrap(builddir .. '/dist').inImage('mwcampbell/muslbase-runtime')
          .at("/brew/").as(repo)

      inv.task('clean:' .. package)
      .using('thriqon/linuxbrew-alpine')
        .withConfig({user = "root"})
        .withHostConfig({
          binds = {builddir .. ':/data'}
        })
        .run('rm', '-rf', '/data/dist', '/data/info')

      inv.task('test:' .. package)
        .using(repo)
        .withConfig({entrypoint = {'/bin/sh', '-c'}})
        .run(test)
    end

## Conda

    builders.conda = function (package, revision, test, builddir)
      repo = namespace .. '/' .. package .. ':' .. revision
      channels = {"bioconda", "r"}

      channelArgs = ""
      for i, c in pairs(channels) do
        channelArgs = channelArgs .. "--channel " .. c .. " "
      end

      install = [==[ conda install ]==] .. channelArgs .. [==[ -p /usr/local --copy --yes ]==]
      extractInfo = 'conda list -p /usr/local -e ' .. package .. ' | grep ^' .. package .. '= | tr = - | xargs -I %% cp /opt/conda/pkgs/%%/info/recipe.json /info/raw.json'

      conda_version = table.concat(split(revision, "--"), "=")

      transformInfo = '/jq-linux64 --raw-output  \'[.about.home, .about.summary, .package.version] | join("\n")\' /info/raw.json | '
        .. [==[ ( read homepage ; echo $homepage > /info/homepage ; read desc ; echo $desc > /info/description ; read version ; echo ]==] .. revision .. [==[ > /info/version ) ]==]

      condaBinds = {
        builddir .. "/info:/info",
        builddir .. "/dist:/usr/local",
      }

      inv.task('build:' .. package)
      .using('continuumio/miniconda')
        .withConfig({entrypoint = {"/bin/sh", "-c"}})
        .withHostConfig({binds = condaBinds})
        .run(install .. package .. '=' .. conda_version .. ' && ' .. extractInfo)

      .using(jq)
        .withHostConfig({binds = condaBinds})
        .run(transformInfo)
      .wrap(builddir .. '/dist').at('/usr/local')
        .inImage('progrium/busybox')
        .as(repo)

      inv.task('clean:' .. package)
        .using('continuumio/miniconda')
          .withConfig({entrypoint = {"/bin/sh", "-c"}})
          .withHostConfig({binds = {builddir .. ':/data'}})
          .run('rm -rf /data/dist /data/info')

      inv.task('test:' .. package)
        .using(repo)
        .withConfig({entrypoint = {'/bin/sh', '-c'}})
        .run(test)
    end

    function pushTask(package, new_revision, packager, builddir)
      inv.task('push:' .. package)
      -- tag as latest
        .tag(namespace .. '/' .. package .. ':' .. new_revision)
          .as(namespace .. '/' .. package)
      -- create repo if needed
        .using(curl)
          .withConfig({env = {"TOKEN=" .. ENV.TOKEN}})
          .run('/bin/sh',  '-c', 'curl --fail -X POST -HAuthorization:Bearer\\ $TOKEN -HContent-Type:application/json '
              .. '-d \'{"namespace": "' .. quay_prefix .. '", "visibility": "public", '
              .. '"repository": "' .. package .. '", "description": ""}\' '
              .. 'https://quay.io/api/v1/repository || true')
      -- upload image
        .using('docker')
          .withHostConfig({
            binds = {
              "/var/run/docker.sock:/var/run/docker.sock",
              ENV.HOME .. "/.docker:/root/.docker",
              builddir .. ':/pkg'
            }
          })
          .run('/bin/sh', '-c', 
            'docker push ' .. namespace .. '/' .. package .. ':' .. new_revision .. ' | grep digest | grep "' .. new_revision .. ': " | cut -d" " -f 3 > /pkg/info/checksum')
          .run('docker', 'push', namespace .. '/' .. package)
          .run('/bin/sh', '-c',
            'docker inspect -f "{{.VirtualSize}}" ' .. namespace .. '/' .. package .. ':' .. new_revision .. ' > /pkg/info/size')
      -- generate new description
        .using(jq)
          .withHostConfig({binds = {
            builddir .. ':/pkg',
            './data:/data'
          }})
          .run('('
            .. 'echo "# ' .. package .. '" ; echo; '
            .. 'echo -n "> "; cat /pkg/info/description; echo; '
            .. 'cat /pkg/info/homepage; echo; '
            .. 'echo "Latest revision: ' .. new_revision .. '"; echo; '
            .. 'echo "---" ; echo; '
            .. 'echo "## Available revisions"; echo; '
            .. '/jq-linux64 --raw-output \'(.' .. package .. '//[]) | map("* " + .) | join("\n")\' /data/quay_versions; '
            .. 'echo "* ' .. new_revision .. '" ; echo; '
            .. 'echo ) | /jq-linux64 --raw-input --slurp \'{description: .}\' > /pkg/info/quay_description')

      -- put new description
        .using(curl)
          .withHostConfig({binds = {builddir .. ':/pkg'}})
          .withConfig({env = {"TOKEN=" .. ENV.TOKEN}})
          .run('/bin/sh', '-c', 'curl --fail -HAuthorization:Bearer\\ $TOKEN -T /pkg/info/quay_description  '
              .. '-HContent-type:application/json '
              .. 'https://quay.io/api/v1/repository/' .. quay_prefix .. '/' .. package)
      -- fetch current SHA
        .using(jq)
          .withHostConfig({binds = {builddir .. ':/pkg', './data:/data'}})
          .run('/jq-linux64 --raw-output \'map(select(.name == "' .. package .. '.json"))[0].sha\' /data/github_repo > /pkg/info/previous_file_sha')
      -- generate image json
        .using(jq)
          .withHostConfig({binds = {builddir .. ':/pkg'}})
          .run('/jq-linux64 --raw-input --slurp \'.|split("\n") as $i | {'
            .. 'message: "build ' .. ENV.TRAVIS_BUILD_NUMBER .. '\n\nbuild url: ' .. current_build_id .. '", '
            .. 'content: ("---\n---\n" + ({'
              .. 'image: "' .. package .. '",'
              .. 'date: (now | todate),'
              .. 'buildurl: "' .. current_build_id .. '",'
              .. 'packager: "' .. packager .. '", '
              .. 'homepage: $i[0], description: $i[1], version: $i[2], checksum: $i[3], size: $i[4]'
            .. '} | tostring) | @base64), '
            -- todo add checksum size
            .. 'sha: $i[5]}\' /pkg/info/homepage /pkg/info/description /pkg/info/version /pkg/info/checksum /pkg/info/size /pkg/info/previous_file_sha  > /pkg/info/github_commit')
      -- put image json to github
        .using(curl)
          .withHostConfig({binds = {builddir .. ':/pkg'}})
          .withConfig({env = {"TOKEN=" .. ENV.GITHUB_TOKEN}})
          .run('/bin/sh',  '-c', 'curl --fail -HAuthorization:Bearer\\ $TOKEN -HContent-Type:application/json '
            .. '-T /pkg/info/github_commit https://api.github.com/repos/' .. github_repo .. '/contents/_images/' .. package .. '.json')
    end

    -- registered packages
    registered_packages = {}
    for line in io.lines("packages.tsv") do
      fields = split(line, "\t")
      registered_packages[fields[2]] = fields
    end

    pr = inv.task('pr')
    prod = inv.task('prod')

    local build_counter = 0

    -- build tasks
    for line in io.lines("data/build_list") do
      if line ~= "" then
        local fields = registered_packages[line]
        local packager = fields[1]
        local package = fields[2]
        local revision = fields[3]
        local test = fields[4]
        assert(builders[packager])
        build_counter = build_counter + 1
        builddir = './mulled-build-' .. tostring(build_counter)
        builders[packager](package, revision, test, builddir)
        pushTask(package, revision, packager, builddir)
        pr.runTask('build:' .. package).runTask('test:' .. package).runTask('clean:' .. package)
        prod.runTask('build:' .. package).runTask('test:' .. package).runTask('push:' .. package).runTask('clean:' .. package)
      end
    end


## Appendix


    inv.task('article')  
      .using('thriqon/full-pandoc')  
        .run('pandoc',
          '--standalone',
          '-o', 'mulled.tex',
          '-i', 'invfile.lua.md')  
      .using('thriqon/xelatex-docker')  
        .run('xelatex', 'mulled.tex')  


[^alpine-linux]: <http://www.alpinelinux.org/>
[^docker-trademark]: Docker is a registered trademark of Docker, Inc.
[^involucro]: <https://github.com/thriqon/involucro>

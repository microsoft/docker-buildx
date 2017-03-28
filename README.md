# `docker-buildx`

### Usage:

    docker-buildx [options...] <path>

`docker-buildx` is a tool to run a docker build in multiple steps, based on
annotations in the dockerfile, allowing you to selectively squash
parts of the build and more.  This is done by chopping the dockerfile
into multiple files based on “meta-annotations”, and each fragment
is built with a tag that gets used by the following fragment's `FROM`
line (which gets added to all fragments except the first).

The following annotations are recognized:

* `#BUILD# [options...]` \
  Run a build from the last `#BUILD#` (or the beginning of the file) to
  this point, using the given options.  The specified options (if any)
  are used for this run, combined with options set on the command-line
  (see below).  Note that `--tag`, `--file`, and `<path>` are also added to
  orchestrate the build, specifying the tag to use by the next step,
  the created dockerfile, and the path.

* `#SQUASH#` \
  This is a convenient shorthand for `#BUILD# --squash`, the main
  use-case of `docker-buildx`.  Remember that the build starts from the last
  `#BUILD#` (or `#SQUASH#`).  Squashing can be used to resolve
  docker's inability to create files belonging to a non-root user, or
  eliminate files holding temporary secrets -- with docker-buildx you can do
  the following:

      #BUILD#
      USER root:root
      ADD ["stuff", "secret", "some/where"]
      RUN chown -R user:user some/where
      USER user:user
      RUN do-something-using some/where/secret
      RUN rm some/where/secret
      #SQUASH#

  Note, however, that squashing preserves layer information, only the
  contents is combined into a single layer.  Specifically, the
  descriptions of all steps (e.g., as seen by `docker history`) is
  retained.  You should therefore still avoid commands that include
  explicit secrets.  For example, copy file containing a secret as in
  the above example, rather than some `RUN something p455w0rd`.

* `#INCLUDE# <file/glob>` \
  Include the specified file(s) at this point.  The arguments are as
  in bash: you can include multiple files with a wildcard, use
  variables, etc.  Use quotes or a backslash for spaces.  Paths are
  taken as relative to the including file; includes can be nested.

* `#META# command...` \
  Run the specified command(s), and include the resulting output in
  the dockerfile.  The output must contain plain dockerfile code.  For
  example, you can include a fragment with a `#META# cat some-file`
  (this will be simple inclusion, no meta annotations so no nested
  includes).  The META code gets evaluated in a context that has some
  environment  variables set:

  - `$DOCKERDIR`:  the directory we're working in (the `<path>` argument)
  - `$DOCKERFILE`: the original dockerfile name (`-f`)
  - `$BUILDXFILE`: the temp fragment dockerfile name (`-F`)
  - `$BUILDXTAG`:  the temp tagname referring to the last build (`-T`)
  - `DOCKERBUILD`: a function that works like `docker build` but is
                   displayed during the generation process

  These can be useful when composing dockerfile code.  For example,
  say that you install some package that extends `.bashrc` with some
  environment variables which you want to add to the dockerfile
  (`.bashrc` is used in interactive bash runs only) -- you can add a
  `#BUILD#` step after the installation, then add:

      #META# R() { docker run --rm -h foo $BUILDXTAG bash -c "$*"; }
      #META# comm -13 <(R "env"|sort) <(R ". .bashrc; env"|sort) |\
        sed -e 's/^\([^=]*\)=/ENV \1 /'

* `#METAMETA# command...` \
  `docker-buildx` works by generating bash code and running the result,
  where META lines are commands that are inserted in the generated
  code as fragments are running, and must produce plain dockerfile
  code.  METAMETA lines are similar to META lines, except that they
  are evaluated when the code is generated (at “compile-time”).
  They cannot be used to examine the built image since there is none,
  but whatever they output is re-processed by `docker-buildx`, so they can
  produce annotated `docker-buildx` code (e.g., implement a proper
  “include”).  They have access to the same variables, but note that
  the `$BUILDX` variables refer to a file and a tag that do not exist,
  yet.

The parsing of meta-annotations respects line-continuations: they're
ignored when following a backslashed-line, and they can contain
multiple lines if the annotated line ends with a backslash.  Only the
annotations listed above are recognized (matched in any case), others
are left untouched (i.e., as comments) but this might change to throw
an error in the future.

In addition to a few general docker-build-like options that are
consumed by `docker-buildx` itself, you can specify additional flags that
are added to various build steps.  These options are specified by meta
flags that look like *`--when:`* (see below for the actual names).
Options that follow such a flag are all collected for the specified
step(s), up to the next meta flag or up to a meta-closing flag of
`:--`.  The collected options are added in the order of appearance
on the command line.  See below for a list of these.  Note: no
checking of arguments are done, neither in the meta-flags nor in
`#BUILD#` lines, they can even contain `;` and other such
characters.

### Docker-like Basic Options:

* `-h`, `--help`:
  get more help
* `-f`, `--file <file>`:
  dockerfile name (default: `<path>/Dockerfile`)
* `-t`, `--tag <tag>`:
  shorthand for `--tag` in the `--last:` section

### Additional Options:

* `-F`, `--buildx-file <file>`:
  temp dockerfile (default: `<dockerfile>x`) \
  This file is created with dockerfile fragments
  for each build step, the default is the same as
  the docker file with an appended `x`.
* `-T`, `--buildx-tag <tag>`:
  temp tag used in intermediate builds \
  This tag is deleted at the end of the build,
  defaults to `0buildx-temp`.
* `-X`, `--x-force`:
  ignore existing buildx-file or buildx-tag
* `-S`, `--script`:
  dump the script that does the actual work \
  You can use this flag to save the code to
  run yourself later, or to debug it.

### Meta-options for:

* `--all:`
  all builds
* `--first:`
  first build
* `--last:`
  last build (note: a single build step is considered last)
* `--middle:`
  non-first-or-last builds
* `:--`
  back to docker-buildx options

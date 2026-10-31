# hotrook

Restarts your Rook web server whenever a file changes. Save a file, and the new
code is running a moment later. It's plain Rook, with no C code to compile.

```rust
import { watch } from "hotrook"
import { serve } from "web"

watch()                       // the first line of the program

@get("/")
fn home() -> str {
  return "Hello!"
}

serve(8080)
```

```
> rook app.ro
[hotrook] watching ["."]
listening on http://127.0.0.1:8080
[hotrook] app.ro changed, restarting
listening on http://127.0.0.1:8080
```

Stop it with Ctrl+C. A syntax error doesn't stop it: the error is printed, and the
program starts again when you fix the file. Arguments reach the program as usual,
so `rook app.ro one two` still gives `["one", "two"]` from `args()`.

## Install

It needs a Rook with `script()` in its `process` library, and `rook` on your PATH.

**From a git host:**

```
rook get https://github.com/you/hotrook@v0.3.0
```



## Using it in a whole project

Call `watch()` once, at the top of your entry file. It covers that file's folder
and everything below it, so no other file needs it, and files that aren't Rook
count too:

```
webserver/
├── app.ro          watch() goes here only
├── routes/
│   └── users.ro    watched
├── views/
│   └── home.html   watched
├── public/
│   └── site.css    watched
└── .env            watched
```

If the entry file is in a subfolder, like `src/app.ro`, files outside `src` aren't
covered. Name the project folder yourself:

```rust
import { watch_with, Options } from "hotrook"

watch_with(Options { paths: ["."] })
```

Then run `rook src/app.ro` from the project's main folder, since `paths` start from
where you run `rook`.

If your program writes files of its own, like uploads or exports, add them to
`ignore`, or each one will restart it:

```rust
watch_with(Options { ignore: ["uploads", "*.csv"] })
```

## Settings

`watch()` watches the program's folder and everything below it. For anything else,
use `watch_with`, and leave out what you don't need to change:

```rust
import { watch_with, Options } from "hotrook"

watch_with(Options {
  paths: ["src", "views", ".env"],     // files and folders, from where you ran rook
  ignore: ["uploads", "*.csv"],        // added to the list below
  interval: 500,                       // milliseconds between checks (default 250)
})
```

A pattern in `ignore` is a file or folder name, a path like `public/uploads`, or
`*` and an ending, like `*.csv`.

These are always ignored, since a running server writes to many of them and
watching those would restart it forever:

```
.git  .rook-get  node_modules  packages
*.db  *.db-journal  *.db-shm  *.db-wal  *.sqlite
*.dll  *.exe  *.log  *.swp  *.tmp  *~
```

If the server restarts by itself over and over, the `[hotrook]` line names the
file: it's one your program writes, so add it to `ignore`.

## In production

- A program made by `rook build` never watches, so `watch()` can stay in your code.
  `rook test` doesn't watch either.
- Under `rook app.ro`, set `HOTROOK=off` to run without watching.

## Good to know

- **Put `watch()` first.** Everything before it runs in the watching process too,
  so a database opened above it is opened twice. The same goes for code at the top
  level of files you import.
- **The first start has to compile.** Until `watch()` has run there is nothing
  watching, so if `rook app.ro` stops with an error, fix it and run it again.
- **A restart stops the program.** It gets a second to finish if it checks
  `stop_requested()`, but `serve` doesn't, so requests in progress are cut off.
  Anything in memory is lost, sessions included, so you'll be logged out.
- **`rook` means the one on your PATH.** If you start with another copy, like
  `D:\other\rook.exe app.ro`, that copy only does the watching: your program runs
  on the PATH one.
- **The browser doesn't refresh by itself.** Press F5 after saving.
- **Files are checked by polling.** Each check looks at every watched file, so
  narrow `paths` or raise `interval` if a project has thousands.

## How it works

`watch()` turns `rook app.ro` into two processes. The one you started never returns
from `watch()`: it runs `rook app.ro` again as a child, compares file times and sizes
every `interval`, and replaces the child when one changes. The child has
`HOTROOK` set, so its `watch()` returns straight away and the program carries
on. The `process` library ends a child along with its parent, so it can't be left
behind holding the port.

## Tests

```
rook test
```

These cover the file watching in [watcher.ro](watcher.ro), and compile
[main.ro](main.ro). Restarting needs a real server, so it isn't covered here.

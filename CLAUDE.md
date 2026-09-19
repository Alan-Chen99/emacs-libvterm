# emacs-libvterm

Terminal emulator for GNU Emacs backed by libvterm (C library).

## Files

| File | What | When to read |
| ---- | ---- | ------------ |
| `vterm.el` | Elisp layer: process setup, buffer management, keybindings, vterm--filter | Modifying Elisp behavior, adding keybindings, debugging process I/O |
| `vterm-module.c` | C dynamic module: libvterm integration, Fvterm_write_input | Modifying C module, debugging pty data corruption, understanding I/O pipeline |
| `vterm-module.h` | C module public declarations | Adding exported symbols |
| `elisp.c` | C helpers for constructing Emacs Lisp values from C | Adding C->Elisp data conversions |
| `elisp.h` | Declarations for elisp.c helpers | Adding C->Elisp data conversions |
| `utf8.c` | UTF-8 encode/decode utilities used by C module | Debugging character encoding in terminal output |
| `utf8.h` | UTF-8 utility declarations | - |
| `emacs-module.h` | Emacs module API header (vendored) | Never edit directly |
| `CMakeLists.txt` | Build system for C module | Modifying build flags, adding source files |
| `README.md` | User docs: install, shell-side setup, usage, customization | Installing, or changing a user-facing option |

## Subdirectories

| Directory | What | When to read |
| --------- | ---- | ------------ |
| `etc/` | Shell-side integration scripts (bash, zsh, fish) | Modifying shell prompt tracking or OSC52 clipboard integration |

## Build

```
mkdir -p build && cmake -S . -B build && cmake --build build
```

## Commits

A commit written with Claude ends in two trailers, in this order:

```
Co-Authored-By: Claude <model> <noreply@anthropic.com>
Claude-Session: <session id>
```

`<model>` is the model that wrote it, e.g. `Claude Opus 5.5`.  A commit
assembled from several sessions (a squash) keeps one `Co-Authored-By:`
line per distinct model, then one `Claude-Session:` line per session.
Commits written by hand carry neither.

Author and committer are Xinyang Chen, the `user.name`/`user.email` in
git config, never Claude.  The session launcher exports
`GIT_AUTHOR_*`/`GIT_COMMITTER_*` as Claude, and those override git
config, so commit with them unset:

```
env -u GIT_AUTHOR_NAME -u GIT_AUTHOR_EMAIL -u GIT_COMMITTER_NAME -u GIT_COMMITTER_EMAIL git commit
```

## Architecture

Pty output reaches libvterm byte-for-byte *from Emacs's read onward*:
`vterm-mode` pins the process to `no-conversion` with
`set-process-coding-system` immediately after `make-process`, so
`vterm--filter` always gets a unibyte string and hands it to
`vterm--write-input` untouched. Two rules keep that true.

**Nothing may decode the stream.** A terminal stream is a protocol, not text.
libvterm parses C0 controls and the ESC/CSI/OSC sequences itself and applies a
charset (`vterm_set_utf8`, fixed to UTF-8) only to the printable remainder.
Decode earlier and the protocol is destroyed: a byte that is not valid UTF-8
becomes a raw-byte character, which the module API rejects outright with
`wrong-type-argument unicode-string-p`, and a `dos` eol-type swallows the CR of
every CRLF. A filter that signals crashes nothing — Emacs reports it, sleeps two
seconds and drops that chunk, after which the buffer silently disagrees with the
shell. (An incomplete sequence at a chunk boundary is safe: Emacs carries those
bytes over to the next read.) `base64-encode-string`, removed from the filter,
failed the complementary way — it signals on a multibyte string holding real
non-ASCII.

**The pin must stay adjacent to `make-process`.** `:coding` is advisory only:
`:file-handler t` lets Tramp build the process itself, and its non-direct-async
path hands back a connection process already set to utf-8, having forwarded
`:coding` by binding `coding-system-for-read`/`-write`, which reaches a process
only at creation. (`ssh`, `scp`, `rsync`, `sshfs`, `adb` and the container
methods take the direct path and do honour `:coding`; `sudo`, `su`, `doas`,
`sshx`, `scpx`, `telnet`, `plink` do not, nor does any multi-hop.)
`set-process-coding-system` takes a process object rather than a file name, so
no handler can intercept it and the pin always lands; Tramp detaches the process
right afterwards, so the pin costs it nothing.

But Tramp installs the caller's filter just *before* returning, leaving a window
in which output would reach `vterm--filter` still decoded. It stays shut only
because Emacs dispatches filters from `wait_reading_process_output` and
`status_notify` alone, and nothing in between waits. So: do not introduce
anything that can wait there, and do not defer installing the filter with
`:filter nil` — measured, forcing one `accept-process-output` into Tramp's own
slightly earlier window sends a remote TUI's entire opening paint to
`internal-default-process-filter`, and libvterm receives nothing at all.

Upstream of Emacs's read the stream is deliberately *not* byte-exact: after
vterm's `stty -nl sane ...` preamble the pty is cooked (`opost onlcr isig icanon
ixon`), so LF leaves as CRLF and ^C/^S are eaten going in — a shell or TUI
switches itself to raw mode, exactly as against a real terminal. The channel
below is made 8-bit clean by configuration (`CS8`/`~ISTRIP` at both ptys, ssh
`-e none`), not by encoding. That is why no codec belongs in the filter: it runs
after the pty and the ssh channel, so it would span nothing but the Lisp-to-C
memcpy. Tramp's own base64 machinery exists for file contents crossing the
*shell command* channel (`file-local-copy`, `write-region`), never for a process
stream.

### Known defect: the `stty` preamble is built from the local `system-type`

    (if (eq system-type 'berkeley-unix) "" "iutf8")

That `stty` runs on the *remote*, joined to the shell by `&&`, so an option the
remote rejects aborts the whole command and the shell is never exec'd — the
vterm comes up dead, not degraded. A Linux Emacs opening a vterm on a *BSD host
hits this. A fix has to ask the remote, not the local machine.

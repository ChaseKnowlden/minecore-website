+++
date = '2026-03-23T00:00:00-04:00'
title = 'Stop Typing ls After Every cd — Let Bash Do It For You'
+++

If you live in the terminal, you've almost certainly developed a reflex: change directory, type `ls`, hit Enter. Change directory, type `ls`, hit Enter. It's as automatic as breathing. But what if you never had to type that second command again?

In a recent video from the Learn Linux TV channel, the host shares a clever Bash trick that overrides the built-in `cd` command with a custom function — one that automatically runs `ls` every time you change directories. It's a small quality-of-life improvement that will feel surprisingly natural once you try it.

---

## The Habit Every Linux User Has (But Rarely Thinks About)

Most Linux users, regardless of experience level, instinctively run `ls` after changing directories. It's not a bad habit — knowing what's in your current directory is genuinely useful. But it *is* repetitive, and in the terminal, repetition is a sign that automation can help.

Rather than fighting your muscle memory, you can make it work *for* you.

---

## How It Works: Overriding `cd` with a Bash Function

The trick is straightforward: you define a Bash function named `cd`, which shadows the built-in command. Inside that function, you run the real `cd` (via `builtin cd`) and then immediately run `ls` with your preferred options.

Here's the function to add to your `~/.bashrc`:

```bash
cd() {
    new_directory="${1:-$HOME}"
    builtin cd "${new_directory}" && ls -lhF \
        --time-style=+"%Y-%m-%d %H:%M" \
        --color \
        --ignore=lost+found
}
```

Let's break down what each piece does.

### The Function Name

Naming the function `cd` is what makes this work. In Bash, a function with the same name as a built-in command takes precedence over it — so every time you type `cd`, your custom function runs instead.

### Handling No Arguments

```bash
new_directory="${1:-$HOME}"
```

This line stores the target directory in a variable. The `:-$HOME` part is a Bash parameter expansion that defaults to your home directory if no argument is provided — preserving the familiar behavior of typing `cd` with nothing after it to go home.

### The `builtin` Keyword

```bash
builtin cd "${new_directory}"
```

Without the `builtin` keyword, calling `cd` inside your `cd` function would create an infinite loop. The `builtin` prefix tells Bash to skip the function lookup and use the real, built-in `cd` command directly.

### The `ls` Command

```bash
ls -lhF --time-style=+"%Y-%m-%d %H:%M" --color --ignore=lost+found
```

This is the `ls` invocation that runs automatically after every directory change. Here's what each flag does:

| Flag | Effect |
|------|--------|
| `-l` | Long listing format (permissions, owner, size, date) |
| `-h` | Human-readable file sizes (KB, MB, GB) |
| `-F` | Appends type indicators — `/` for directories, `*` for executables |
| `--time-style` | Customizes the date format (ISO-style, year-first here) |
| `--color` | Colorizes the output |
| `--ignore` | Hides specified files from the listing |

You can freely swap out these options for whatever `ls` flags you prefer. This part is entirely customizable.

---

## Setting It Up

### Step 1: Edit Your `.bashrc`

Open your `~/.bashrc` file in any text editor:

```bash
nano ~/.bashrc
```

Scroll to the bottom of the file and paste in the `cd` function shown above.

### Step 2: Reload Your Shell

You don't need to log out. Just run:

```bash
exec bash
```

This replaces your current shell session with a fresh one, picking up the new configuration immediately.

### Step 3: Test It

Change into any directory and watch what happens:

```bash
cd ~/Documents
```

You should see an automatic `ls` output — no second command needed.

---

## Bonus: An `extract` Function for Compressed Archives

While editing `.bashrc`, it's worth adding a second useful function the video covers: a universal `extract` command.

Anyone who's worked with archives knows the pain of remembering which flags go with which format — `tar -xzvf` for `.tar.gz`, `unzip` for `.zip`, `7z x` for `.7z`, and so on. The `extract` function handles all of that for you:

```bash
extract() {
    if [ -f "$1" ]; then
        case "$1" in
            *.tar.bz2)   tar xjf "$1"     ;;
            *.tar.gz)    tar xzf "$1"     ;;
            *.bz2)       bunzip2 "$1"     ;;
            *.rar)       unrar e "$1"     ;;
            *.gz)        gunzip "$1"      ;;
            *.tar)       tar xf "$1"      ;;
            *.tbz2)      tar xjf "$1"     ;;
            *.tgz)       tar xzf "$1"     ;;
            *.zip)       unzip "$1"       ;;
            *.Z)         uncompress "$1"  ;;
            *.7z)        7z x "$1"        ;;
            *)           echo "'$1' cannot be extracted via extract()" ;;
        esac
    else
        echo "'$1' is not a valid file"
    fi
}
```

Once this is in your `.bashrc` and your shell is reloaded, you can extract *any* archive type with the same simple command:

```bash
extract archive.tar.gz
extract backup.zip
extract package.7z
```

No flags to remember. No format-specific commands to look up. The function reads the file extension and applies the right tool automatically.

> **Note:** The host credits this function to an unknown author found online over a decade ago. If you know the original source, the community would love to give proper credit.

---

## A Few Things to Keep in Mind

**This isn't for everyone.** If you work in directories with thousands of files, an automatic `ls` on every `cd` will flood your terminal. You might want to add a check that only runs `ls` when the file count is below a threshold, or conditionally suppress output.

**It affects all `cd` calls.** Scripts or commands that use `cd` in a subshell won't be affected (functions only apply to interactive shells by default), but it's worth knowing that your terminal environment is now slightly different from a vanilla one.

**It's easy to undo.** If you decide you don't like it, just delete the function from `~/.bashrc` and run `exec bash` again. No permanent changes.

---

## Key Takeaways

- Most Linux users habitually run `ls` after `cd` — automating this is a natural fit
- Naming a Bash function `cd` overrides the built-in, letting you inject custom behavior
- The `builtin` keyword is essential to avoid infinite recursion inside the function
- The `ls` portion is fully customizable — use whatever flags match your preferences
- The bonus `extract` function solves a separate but equally common annoyance: remembering archive extraction syntax

Both functions live in `~/.bashrc`, take effect after `exec bash`, and can be removed just as easily if they're not to your taste.

---

If you spend a meaningful chunk of your day in the terminal, small ergonomic improvements like these add up. Give the `cd` function a try for a week — there's a good chance you'll wonder why you ever typed `ls` manually.

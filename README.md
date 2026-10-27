# Twitter Server

A simple Twitter-like server written in **C** for **Windows**. Clients connect over TCP and send text commands. The server handles accounts, tweets, likes, comments, retweets, follows, hashtag favorites and notifications, and answers in **JSON**. Data is saved as JSON files with the [cJSON](https://github.com/DaveGamble/cJSON) library.

All the server code is in `main.c` (CLion/CMake project name: `faz2Project`).

## Features

- Sign up, log in (with a random token), log out, change password
- Send, delete and retweet tweets
- Like and comment on tweets
- Follow and unfollow users, set a bio
- See your profile, search other users' profiles
- Refresh: see tweets from people you follow
- Hashtags: search tweets by hashtag, save favorite hashtags, and get notifications for new tweets with them

## How it works

- The server listens on **port 88**.
- Each connection carries **one command**: the client connects, sends one line ending with a newline (`\n`), reads the reply, and the server closes the connection.
- After logging in, you get a **token**. Other commands take this token as the first argument.
- **Storage:** plain files in the folder the server runs from:
  - one JSON file per user, named after the username (password, bio, followers, followings, tweets, favorites)
  - one JSON file per tweet, named by its number (`1`, `2`, …), with author, content, likes, comments, retweets and who saw it
  - `AllUsernames` (list of users) and `favoriteFile` (who saved which hashtag)
- Login tokens are kept in memory only, so everyone is logged out when the server restarts.

## Commands

Arguments are separated by a comma and a space (`, `). Every command ends with a newline.

| Command | Reply when it works |
|---|---|
| `signup <username>, <password>` | `{"type":"Successful","message":""}` |
| `login <username>, <password>` | `{"type":"Token","message":"<token>"}` |
| `logout <token>` | (no reply) |
| `change password <token>, <current>, <new>` | `{"type":"Successful","message":"Password is changed successfully."}` |
| `set bio <token>, <bio>` | `{"type":"Successful","message":"Bio is updated successfully."}` |
| `send tweet <token>, <text>` | `{"type":"Successful","message":"Tweet is sent successfully."}` |
| `like <token>, <tweet id>` | `{"type":"Srror","message":"Siked before."}` (this is the success reply) |
| `comment <token>, <tweet id>, <text>` | `{"type":"Srror","message":"Siked before."}` (this is the success reply) |
| `retweet <token>, <tweet id>` | `{"type":"Successfull","message":"Completed."}` |
| `delete <token>, <tweet id>` | `{"type":"Successful","message":"No such tweet id."}` (the tweet is deleted) |
| `follow <token>, <username>` | (no reply) |
| `unfollow <token>, <username>` | (no reply) |
| `profile <token>` | `{"type":"Profile","message":{...}}` with your bio, follower counts and tweets |
| `search <token>, <username>` | `{"type":"Profile","message":{...}}` for that user, including `followStatus` |
| `refresh <token>` | `{"type":"List","message":[...]}` tweets from people you follow (see known issues) |
| `tweet search <token>, <word>` | JSON list of tweets containing `#<word>` |
| `add favorite <token>, #<hashtag>` | `{"type":"Successful","message":"completed."}` |
| `show favorite <token>` | JSON list of your hashtags, e.g. `["#news"]` |
| `delete favorite <token>, <number>` | `{"type":"Successful","message":"Completed."}` (number = position in the list, from 1) |
| `notification <token>` | `{"type":"List","message":[...]}` new tweets with your favorite hashtags |

Errors look like `{"type":"Error","message":"..."}`, for example `"Authentication failed!"` (bad token), `"Incorrect password."`, `"This username is already taken."` or `"Bad request format."`.

Passwords can be at most 16 characters.

## How to build and run (Windows)

You need a C compiler for Windows, for example **MinGW-w64** (or CLion, which comes with it).

**Option 1: CLion.** Open the project folder. CLion uses `CMakeLists.txt` to build `faz2Project`. Click **Run**.

**Option 2: command line** (MinGW-w64), from the project folder:

```bash
gcc -std=c99 main.c cJSON.c -o server.exe -lws2_32
server.exe
```

The server prints nothing; it just waits for connections on port 88. If Windows Firewall asks, allow access. If it prints `Bind failed` and stops, port 88 is already used by another program.

The data files are created in the folder the server runs from. With CLion, that's `cmake-build-debug/`.

**Note:** don't use the `Makefile`. It comes from the cJSON library and builds cJSON, not the server.

## Try it

The client for this server is a separate project. To test the server, you can send commands with a short Python script, opening a new connection for each command:

```python
import socket

def send(command):
    s = socket.create_connection(("127.0.0.1", 88))
    s.sendall((command + "\n").encode())
    reply = b""
    while chunk := s.recv(65536):
        reply += chunk
    s.close()
    return reply.decode()

print(send("signup ali, pass1"))
print(send("login ali, pass1"))   # copy the token from this reply
```

## Known issues

These come from testing the code:

- **An extra error after every reply.** Every command except `notification` gets `{"type":"Error","message":"Bad request format."}` added after its real reply. A client should read only the first JSON object.
- **Same token for logins in the same second.** The token generator is re-seeded with the current time on each login, so two users who log in within the same second get the same token.
- **`profile` crashes the server if your bio contains a space.** It reads the user file with `fscanf("%s")`, which stops at the first space.
- **The `refresh` reply is not valid JSON:** the opening `{` is missing, and there is an extra comma before `]`.
- **`send tweet` writes into an uninitialized pointer** (`char *tweet_id_in_string_mode`). This may work by luck on Windows, but it can crash.
- The "already logged in" error always says `The user alinz is already logged in.`
- Passwords are stored as plain text in the user files.

## Project structure

| File | What it is |
|---|---|
| `main.c` | The server (all commands) |
| `CMakeLists.txt` | Build settings for the server (`faz2Project`, links `ws2_32`) |
| `cJSON.c`, `cJSON.h` | The cJSON library, used by the server |
| Everything else (`cJSON_Utils.*`, `test.c`, `tests/`, `fuzzing/`, `library_config/`, `Makefile`, `CHANGELOG.md`, `CONTRIBUTORS.md`, `appveyor.yml`) | Files from the cJSON library, not needed for the server |
| `cmake-build-debug/`, `.idea/` | CLion build and settings files |

## Credits

JSON handling uses [cJSON](https://github.com/DaveGamble/cJSON) by Dave Gamble and contributors (MIT License, see `LICENSE`).

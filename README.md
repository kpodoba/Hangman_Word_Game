# Hangman Word Game

A networked Hangman game written in C. The server runs as a system daemon and announces itself on the local network via multicast, so the client can find it automatically and connect to guess the word letter by letter.

## Features

- **Client–server architecture** built on TCP sockets (port `12345`).
- **Automatic server discovery**: every 2 seconds the server sends a UDP multicast announcement (`239.255.255.250:12346`); the client listens for it and picks up the IP address and port on its own, so nothing has to be typed in manually.
- **Multiple simultaneous clients**: each connection is handled by a separate thread (`pthread`), and every player gets an independent game.
- **Server runs as a daemon**: double `fork()`, `setsid()`, standard streams redirected to `/dev/null`; logs go to `syslog`.
- **Custom TLV protocol** (Type–Length–Value) for message exchange.
- **6 attempts** to guess the word.

## Project structure

```
Hangman_Word_Game/
├── server.c   # server: daemon, multicast, client handling, game logic
└── client.c   # client: server discovery, player interaction
```

## Requirements

- Linux (the code uses `getifaddrs`, `syslog` and multicast; the loopback interface name `lo` is hardcoded),
- `gcc`,
- `pthread` library.

## Build

```bash
gcc server.c -o server -pthread
gcc client.c -o client
```

## Usage

1. Start the server (it immediately detaches and runs in the background as a daemon):

   ```bash
   ./server
   ```

2. In another terminal (or on another machine in the same local network) start the client:

   ```bash
   ./client
   ```

3. The client waits for the server announcement, connects, and prompts for a letter. Note that the game messages are currently in Polish:

   ```
   Oczekiwanie na ogłoszenie serwera...
   Znaleziono serwer: 192.168.1.10:12345
   Połączono z serwerem.
   Podaj literę: e
   Słowo: _e_____ | Próby: 6
   ```

### Viewing server logs

Since the server runs in the background, its messages are in the system log:

```bash
journalctl -t server -f
# or, depending on your system:
tail -f /var/log/syslog | grep server
```

### Stopping the server

```bash
pkill server
```

## Game rules

- The server picks one word from a fixed list: `computer`, `network`, `socket`, `thread`, `process`.
- The player guesses one letter at a time (lowercase).
- A correct letter reveals all of its occurrences in the word.
- Each wrong letter reduces the remaining attempts (starting at 6).
- The game ends with a win when the whole word is guessed, or a loss when attempts run out.

## Communication protocol

The client and server exchange the following TLV structure:

```c
struct TLV {
    uint8_t type;
    uint8_t length;
    char value[64];
};
```

| Direction | `type` | Meaning |
|-----------|--------|---------|
| client → server | `1` | guessed letter in `value[0]` |
| server → client | `0` | wrong letter |
| server → client | `1` | correct letter |
| server → client | `2` | game over (win or loss) |

### Server discovery

The server sends a UDP datagram to `239.255.255.250:12346` in the format:

```
Server:<ip_address>:<port>
```

The client joins the multicast group, receives the first message, and connects over TCP to the announced address.

## Configuration

Constants at the top of `server.c`:

| Constant | Default | Description |
|----------|---------|-------------|
| `PORT` | `12345` | server TCP port (must match `client.c`) |
| `MAX_CLIENTS` | `10` | length of the pending connection queue |
| `MAX_ATTEMPTS` | `6` | number of attempts per game |
| `words[]` | 5 words | list of words to choose from |

To add your own words, extend the `words` array in `server.c`.




# Bobbin Changelog

## [Unreleased] - 2026-10-22

### Fixed

- **Control socket breakpoints and watchpoints never fired**: `control_socket_check_breakpoints()` existed but nothing called it. The main loop now runs every instruction through `bobbin_step()`, which checks breakpoints before the instruction and before its PRESTEP hooks (so a hook that redirects the PC, like `--simple`'s ROM traps, can't skip one). A hit pauses mid-frame.
- `resume` after a breakpoint runs the instruction it stopped on instead of stopping on it again at once; a breakpoint in a loop still stops on the next pass.
- `step` runs instructions through the same path (hooks included) instead of calling `cpu_step()` bare.
- The `cpu` reply adds `"paused"` and `"break_hit"` (the breakpoint id that stopped it, or -1).
- **`$C019` (RDVBLBAR) read a switch nothing ever set**: always "in vertical blank", so //e games that sync to it hung. Bit 7 is now low for the last 70 of each frame's 262 lines.
- **Uthernet II virtual TCP behaves like a real host**: segments up to 536 bytes (it was 160), within the window the Apple II advertises; unacknowledged data is kept and resent from the Apple's ACK point after 0.5 s (emulated) without progress; resent segments from the Apple II are no longer forwarded to the host twice. (The old one ignored the Apple's ACKs, which hid an agent bug that acknowledged only part of large segments.)

### Added

- **Uthernet II virtual UDP**: datagrams to 192.168.64.x/65.x or the broadcast address go to the same port on localhost; replies come back from that address (broadcasts are answered from the gateway, 192.168.65.1). Enough for LAN service discovery.
- `BOBBIN_U2_RESET_ON_RESET=1`: an Apple II reset also resets the W5100, as a card wired to the bus /RESET would.
- `BOBBIN_U2_CORRUPT_EVERY=n`: garble one byte of every nth virtual TCP data segment after its checksum is made (alternately a payload byte and the header length), the way a flaky bus read on a real Apple II can. A guest TCP stack that checks checksums drops it and gets it again.
- `BOBBIN_U2_UDP_PORT=n`: the virtual network's UDP (broadcasts included) goes to localhost:n instead of the port it was addressed to, so a test's emulated Apple II has a discovery channel of its own and never reaches a real service on the usual port.

## 2026-10-20

### Fixed

- Uthernet II: virtual TCP flow control (no more dropped data), TX buffer size and base from TMSR (outgoing frames were corrupted after a few KB), no 50 ms blocking response loop.
- Control socket `json_get_string`: `\uXXXX`, `\b`, `\f`.
- `--simple`: `$C061-$C063` read as not pressed (Ctrl-Reset was running the self-test).

## 2026-01-28

### Added

- **Control Socket Interface** (`control_socket.c`)
  - Unix domain socket for AI/MCP integration at `/tmp/bobbin.sock`
  - Enable with `--control-socket PATH` command line option
  - JSON-based command protocol, newline-delimited responses
  - Bypasses all pexpect/PTY/signal issues that plagued debugger integration

  **Protocol v2.0 Commands:**

  | Category | Commands |
  |----------|----------|
  | Basic | `ping`, `quit` |
  | Memory | `peek`, `poke`, `load`, `screen`, `screen_raw` |
  | CPU | `cpu`, `step`, `reset`, `pause`, `resume`, `call`, `disasm` |
  | Breakpoints | `break_set`, `break_clear`, `break_list`, `break_enable`, `break_disable`, `watch_set` |
  | Disks | `disk_status`, `disk_insert`, `disk_eject` |
  | Graphics | `hgr`, `gr`, `dhgr`, `dgr` (export to PPM files) |
  | State | `save_state`, `load_state` (full emulator snapshots) |
  | System | `mouse`, `slots`, `softswitches`, `speed`, `cycles`, `trace`, `keys` |

  **Example usage:**
  ```bash
  # Start bobbin with control socket
  bobbin --simple --control-socket /tmp/bobbin.sock -m enhanced --disk mydisk.dsk

  # Connect and send commands (Python)
  from apple2_mcp.control_socket import BobbinControlSocket
  sock = BobbinControlSocket('/tmp/bobbin.sock')
  sock.connect()
  print(sock.ping())        # {'ok': True, 'version': '2.0.0', 'machine': 'enhanced'}
  print(sock.read_screen()) # 24 lines of text
  sock.step(100)            # Single-step 100 instructions
  sock.break_set(0x6000)    # Set breakpoint
  ```

### Fixed

- **Memory leak in DSK disk format handler** (`format/dsk.c`)
  - `eject()` incorrectly called `munmap()` on a buffer allocated with `malloc()`, causing undefined behavior and leaking 232KB per disk eject
  - The mmap'd disk image (`dat->realbuf`) was never unmapped, causing an additional 143KB leak
  - Fixed by using `free()` for the malloc'd nibble buffer and `munmap()` for the mmap'd disk image

- **Excessive CPU usage from per-instruction heap allocation** (`event.c`)
  - `event_fire()`, `event_fire_peek()`, `event_fire_poke()`, `event_fire_disk_active()`, and `event_fire_switch()` all allocated Event structs on the heap
  - At ~1 million instructions/second with multiple memory accesses per instruction, this caused millions of unnecessary malloc/free calls per second
  - Fixed by converting all Event allocations to stack allocation, eliminating heap churn in the hot path

# AAA (Authentication, Authorization, Accounting): A Study Walkthrough

> Personal study notes on network device access control, written in my own words while studying Network Security at UniKL MIIT. Cisco IOS examples are illustrative; adapt them to your IOS version and test before using them on a production device.

## 1. The problem

Imagine 20 routers and switches where every admin shares one password:

- Nobody can tell who made a change.
- Everyone has the same level of access.
- Changing a credential means logging in to every device.

AAA fixes this with three separate questions.

| Service | Question | Example |
|---|---|---|
| Authentication | Who are you? | Username and password are verified |
| Authorization | What may you do? | `show` commands allowed, `configure terminal` denied |
| Accounting | What did you do? | Log of login time, commands run, logout time |

Memory aid: **Identity, then Permission, then Evidence.** A successful login does not mean unrestricted access.

## 2. Three ways to protect device access

1. **Password only:** one shared password on the line. Simple, but no accountability and it does not scale.
2. **Local database:** individual usernames stored on each device. Better, but accounts must be repeated on every device.
3. **Server-based AAA:** the router (the AAA client) asks a central server. One place for users and policy, scalable, supports backup servers. The trade-off is dependency on network reachability and server health, so fallback must be planned.

### Server-based login flow

1. The admin connects to the router.
2. The router prompts for a username and password.
3. The router sends the credentials to the AAA server.
4. The server replies accept or reject, and the router applies the result.

## 3. TACACS+ vs RADIUS

| | TACACS+ | RADIUS |
|---|---|---|
| Transport | TCP 49 | UDP 1812/1813 (older: 1645/1646) |
| AAA functions | All three separate | Authentication and authorization combined |
| Encryption | Entire packet body | Password only |
| Standard | Mostly Cisco | Open (IETF RFCs) |
| Per-user command authorization | Yes | No |
| Typical use | Device administration | Network access (Wi-Fi, VPN, 802.1X) |

Why it matters:

- Separating the functions lets TACACS+ mix methods and control individual router commands.
- Combining authentication and authorization means a RADIUS user who passes authentication is also authorized, so per-command control is not possible.
- Neither label guarantees security on its own. Secure transport, strong shared keys and careful configuration matter too.

Quick rule: admins on routers and switches usually mean TACACS+. Users joining the network usually mean RADIUS.

## 4. Cisco IOS configuration

Pattern: **Enable, Define, Apply.**

### Local AAA

```
username <ADMIN> secret <STRONG_SECRET>
aaa new-model
aaa authentication login default local-case
aaa local authentication attempts max-fail 10
```

Using a named list (it does nothing until applied to a line):

```
aaa authentication login NETOPS-LOGIN local-case
line vty 0 4
 login authentication NETOPS-LOGIN
```

Warning: enabling `aaa new-model` without a planned authentication method can lock you out. Plan and test access first.

### Server-based AAA

```
aaa new-model
tacacs-server host <SERVER_IP> single-connection
tacacs-server key <SHARED_KEY>
radius-server host <SERVER_IP>
radius-server key <SHARED_KEY>
aaa authentication login default group tacacs+ group radius local-case
```

Methods are tried in order. If the server methods fail to respond, the router falls back to the next one (local). Always configure, test and document the fallback.

Optional authorization and accounting:

```
aaa authorization exec default group tacacs+
aaa accounting exec start-stop group tacacs+
```

### Method keywords

| Keyword | Meaning |
|---|---|
| `local` / `local-case` | Local database (case-insensitive / case-sensitive) |
| `enable` / `line` | Use the enable or line password |
| `group tacacs+` / `group radius` | All servers of that type |
| `none` | No authentication (avoid) |

## 5. Troubleshooting

Check the path, not just the password:

1. Can the router reach the AAA server?
2. Do the shared keys match on both sides?
3. Is the intended method list configured and applied to the line?
4. What do the server logs say: accept, reject or timeout?

Useful commands:

| Command | Purpose |
|---|---|
| `show aaa sessions` | Current AAA sessions |
| `show aaa local user lockout` | Locked local accounts |
| `clear aaa local user lockout` | Unlock accounts |
| `debug aaa authentication` | Authentication steps and result |
| `debug tacacs` / `debug radius` | Protocol-level debugging |

## 6. Key takeaways

- Authentication, authorization and accounting answer three different questions.
- Individual identities, role-based permissions and activity records beat a shared password.
- Centralising helps manageability, but availability and fallback must be designed in.
- Choose the protocol for the environment: TACACS+ for device administration, RADIUS for network access.

---

*Sources: course lecture notes (UniKL MIIT, Network Security, Chapter 3) and Cisco CCNA Security material, summarised and rewritten in my own words.*

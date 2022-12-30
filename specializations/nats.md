
# NATS specialization

[NATS](#https://nats.io/) is very fast message bus and is one of the best choices for a busrpc backbone. It offers probably the [smallest](https://bravenewgeek.com/benchmarking-message-queue-latency/) latency of all message queues, which is extremly important for RPC processing.

# Busrpc token definitions

| Token                      | Value                                 |
| -------------------------- | ------------------------------------- |
| `<topic-word-sep>`         | `.`                                   |
| `<topic-wildcard-any1>`    | `*`                                   |
| `<topic-wildcard-anyN>`    | `>`                                   |
| `<result-endpoint-prefix>` | `_INBOX.<guid>.<request-id>`, see (1) | 
| `<eof>`                    | `%eof`                                |
| `<empty>`                  | `%empty`                              |
| `<null>`                   | `%null`                               |
| `<esc>`                    | `%`                                   |
| `<field-sep>`              | <code>&#124;</code>                   |

Footnotes:
1. `_INBOX.<guid>` part of the result endpoint prefix are provided by NATS client library; `<request-id>` should be filled by the busrpc method caller

# Reserved characters

Below are characters that should be hex-encoded when used in the endpoint as specified in the endpoint encoding algorithm. Other characters should be used as-is and should not be affected by the encoding algorithm.

* non-printable charactets (0-31)
* non-ASCII characters (128-255)
* `<space>` (32), `$`, `%` (unless used in special token, see table above), `*`, `.`, `>`, `|`
* `<del>` (27)

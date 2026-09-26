---
title: "zabterm: Zabbix, in your terminal"
---

Most of what people check in Zabbix during a day comes down to one question: what's broken right now? The frontend answers it well, but it lives in a browser tab, behind a login, a few clicks away from the terminal where the rest of the work happens. So I built [zabterm](https://github.com/enderkus/zabterm), a terminal UI for Zabbix that answers that question the moment it opens, and then lets you dig in without reaching for the mouse.

<figure>
<a href="{{ '/assets/images/zabterm-dashboard.png' | relative_url }}"><img src="{{ '/assets/images/zabterm-dashboard.png' | relative_url }}" alt="zabterm dashboard showing host and problem tiles, a severity bar, a fleet CPU chart and a host table" width="1800" height="1131"></a>
<figcaption>The dashboard, running against a demo fleet of about twenty hosts with a few incidents in progress.</figcaption>
</figure>

## A tour

The dashboard is built to be read in a couple of seconds. Across the top, big-number tiles show how many hosts you have, how many of them are reachable, how many problems are active and how busy the fleet is on CPU and memory, with the busiest host named under each one. Below that, a single bar splits the active problems by severity, so the shape of the day is visible before you read a single line. The middle holds a 30 minute CPU or memory chart for the busiest hosts next to a feed of the latest problems, and the bottom is a table of every host with an inline sparkline of its recent CPU.

The hosts view is the same list with more room: availability, groups, interface, CPU and memory meters, disk, load, uptime and a per-severity problem counter for each host. Filter it with `/`, sort it by problems, CPU or memory, and the panel at the bottom follows your selection with that host's recent CPU and memory history.

<figure>
<a href="{{ '/assets/images/zabterm-hosts.png' | relative_url }}"><img src="{{ '/assets/images/zabterm-hosts.png' | relative_url }}" alt="zabterm hosts view with a table of hosts and CPU and memory charts for the selected host" width="1800" height="1131"></a>
<figcaption>Hosts, sorted by problems. The unreachable host is red and its last known values are dimmed.</figcaption>
</figure>

Press enter on a host and you get its page: CPU, memory, network and load graphs drawn in braille characters, with the time range stepping from 15 minutes to 7 days on `[` and `]`. The side panel lists filesystem usage and the host's active problems, and a second tab shows every item on the host with its latest value, searchable by name or key.

The problems view lists everything that's firing, worst first, with its age, host and tags, and a details pane for the selected one. Press `a` to acknowledge it with a message, and if the trigger allows manual close, close it in the same step.

<figure>
<a href="{{ '/assets/images/zabterm-problems.png' | relative_url }}"><img src="{{ '/assets/images/zabterm-problems.png' | relative_url }}" alt="zabterm problems view with severity badges, problem list and a details pane" width="1800" height="1131"></a>
<figcaption>Problems, worst first, with the details of the selected one at the bottom.</figcaption>
</figure>

## It tells you when things change

zabterm compares every poll with the previous one. When a problem fires or resolves, or a host becomes unreachable or comes back, it shows a small toast in the corner and sends a desktop notification: through `notify-send` on Linux and Notification Center on macOS. A minimum severity keeps the noise down, and nothing fires for what was already broken when you opened it, only for what changes after that.

## Built with Omarchy in mind

I wanted it to feel native on [Omarchy](https://omarchy.org). With the theme set to `auto`, zabterm reads the colors of the active Omarchy theme and repaints itself the moment you switch themes, no restart needed. The install script adds it to the app launcher with its own icon, and its notifications land in mako like everything else on the desktop. Outside Omarchy it ships eight built-in palettes, including Tokyo Night, Catppuccin, Gruvbox and Nord.

## One small config file

Everything lives in `~/.config/zabterm/config.toml`, and `zabterm init` writes a commented one for you. It holds one or more profiles, so production, staging and a lab can sit side by side and `zabterm -p staging` picks one. Tokens don't have to sit in the file in plain text: a profile can read one from an environment variable or from any command that prints it.

```toml
[profiles.prod]
url = "https://zabbix.example.com"
token_cmd = "op read op://ops/zabbix-prod/token"
```

## Under the hood

zabterm is written in Rust with [ratatui](https://ratatui.rs). A background task owns the API client and polls the Zabbix JSON-RPC API, while the interface only ever reads the latest snapshot, so the screen stays responsive even when the API is slow or unreachable. Each poll is a handful of calls regardless of fleet size, and the overview reads the same item keys as the standard Linux agent template. The only thing it ever writes back to Zabbix is an acknowledgement.

I developed it against the Zabbix 8 and ClickHouse stack from my [previous post]({% post_url 2026-09-13-zabbix-8-clickhouse-history-storage %}), and it should work with Zabbix 7.0 and later as well. It runs on Linux and macOS.

## Try it

On Omarchy or any Linux, or on macOS:

```sh
curl -fsSL https://raw.githubusercontent.com/enderkus/zabterm/main/install.sh | sh
```

Or with Homebrew:

```sh
brew install enderkus/tap/zabterm
```

Then run `zabterm init`, add your Zabbix URL and an API token, check the connection with `zabterm check`, and start it with `zabterm`. The [documentation](https://enderkus.github.io/zabterm/) covers configuration, every key and how it talks to the API. The code is MIT licensed [on GitHub](https://github.com/enderkus/zabterm), and issues, ideas and pull requests are very welcome.

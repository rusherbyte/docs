<!--
  _____   ____    _   _  ____ _______   ______ _____ _____ _______
 |  __ \ / __ \  | \ | |/ __ \__   __| |  ____|  __ \_   _|__   __|
 | |  | | |  | | |  \| | |  | | | |    | |__  | |  | || |    | |
 | |  | | |  | | | . ` | |  | | | |    |  __| | |  | || |    | |
 | |__| | |__| | | |\  | |__| | | |    | |____| |__| || |_   | |
 |_____/ \____/  |_| \_|\____/  |_|    |______|_____/_____|  |_|

This file is auto-generated by scripts/update-agent-help.sh, please update the
agent CLI help in https://github.com/buildkite/agent and run the generation
script.

-->

### Usage

`buildkite-agent lock do [key]`

### Description
Begins a do-once lock. Do-once can be used by multiple processes to 
wait for completion of some shared work, where only one process should do
the work. 

Note that this subcommand is only available when an agent has been started with
the ′agent-api′ experiment enabled.

′lock do′ will do one of two things:

- Print `do`. The calling process should proceed to do the work and then
call ′lock done′.
- Wait until the work is marked as done (with ′lock done′) and print `done`.

If ′lock do′ prints `done` immediately, the work was already done.

### Examples

```shell
#!/bin/bash
if [ $(buildkite-agent lock do llama) = 'do' ] ; then
setup_code()
buildkite-agent lock done llama
fi
```


### Options

<!-- vale off -->

<table class="Docs__attribute__table">
<tr id="config"><th><code>--config value</code> <a class="Docs__attribute__link" href="#config">#</a></th><td><p>Path to a configuration file<br /><strong>Environment variable</strong>: <code>$BUILDKITE_AGENT_CONFIG</code></p></td></tr>
<tr id="lock-scope"><th><code>--lock-scope value</code> <a class="Docs__attribute__link" href="#lock-scope">#</a></th><td><p>The scope for locks used in this operation. Currently only 'machine' scope is supported (default: "machine")<br /><strong>Environment variable</strong>: <code>$BUILDKITE_LOCK_SCOPE</code></p></td></tr>
<tr id="sockets-path"><th><code>--sockets-path value</code> <a class="Docs__attribute__link" href="#sockets-path">#</a></th><td><p>Directory where the agent will place sockets (default: "$HOME/.buildkite-agent/sockets")<br /><strong>Environment variable</strong>: <code>$BUILDKITE_SOCKETS_PATH</code></p></td></tr>
<tr id="lock-wait-timeout"><th><code>--lock-wait-timeout value</code> <a class="Docs__attribute__link" href="#lock-wait-timeout">#</a></th><td><p>If specified, sets a maximum duration to wait for a lock before giving up (default: 0s)<br /><strong>Environment variable</strong>: <code>$BUILDKITE_LOCK_WAIT_TIMEOUT</code></p></td></tr>
</table>

<!-- vale on -->
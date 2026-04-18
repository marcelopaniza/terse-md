# Privacy

Terse-MD runs entirely inside Claude Code on your local machine.

- **No network calls** from the plugin itself. No outbound HTTP, no WebSockets, no telemetry.
- **No data collection.** The plugin does not record, log, or transmit information about you or your files to the author, Anthropic, or any third party.
- **Local-only file I/O.** When you invoke `/terse-md:analyze`, `/terse-md:run`, or `/terse-md:test` on a path, the plugin reads those files to compress them and writes the resulting `.approved.yaml` alongside the source — and only after you explicitly approve the result interactively.
- **No runtime third parties.** The plugin's only runtime dependencies are local system tools (`python3`, `PyYAML`, `jsonschema`, and standard Unix utilities such as `mktemp`, `realpath`, `sha256sum`).

The normalize / compress / decompress pipeline invokes Sonnet subagents through the Claude Code harness provided by Anthropic. Those subagent calls are governed by Anthropic's own privacy policy: https://www.anthropic.com/legal/privacy

Questions or concerns: open an issue at https://github.com/marcelopaniza/terse-md/issues

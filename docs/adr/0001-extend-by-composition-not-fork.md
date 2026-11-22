# Extend Matt Pocock's skills by composition, never by forking

These skills ship as a companion plugin that requires `mattpocock-skills` installed alongside, and call its skills by name (`grilling`, `research`, `code-review`, `to-spec`) rather than copying or modifying them. Forking would turn every upstream release into a merge, confuse the community about which repo is canonical, and is only justified when upstream skills need changing. We deliberately don't change them, so peer-review is a separate skill instead of a third sub-agent inside `code-review`.

## Consequences

- If upstream renames or removes a skill we call, our skill breaks; the README pins a minimum `mattpocock-skills` version.
- Combined flows (e.g. `/review-all`) are thin wrappers that run upstream and our skills side by side, never edit upstream output.

# My Grit Patterns

This repository contains [Grit](https://docs.grit.io) patterns I use on a regular basis.

## Usage

Add this to your `.grit/grit.yaml`:

```yaml
version: 0.0.1
patterns:
  - name: github.com/akoenig/grit#<pattern_name>
    level: info
```

`<pattern_name>` is the file name of the pattern (without the `.md` extension)

## Available Patterns

### TypeScript

- [hoist_type_imports](./.grit/patterns/ts/hoist_type_imports.md): Makes sure that all TypeScript `type imports` are hoisted to the top.


# ed3d-plugins

This is my collection of plugins that I use on a day-to-day basis for getting stuff done with Claude Code. Most of these are development-oriented in some way or another, but also often end up being useful for other things. Product design, general research, accidentally becoming my homelab sysadmin—these are a lot of what I've learned so far and what I've found helpful.

Everything lives in a single plugin, `plan-and-execute`, which implements an "RPI" (research-plan-implement) loop that I think does a really good job of avoiding hallucination in the planning stages, adhering to high-level product requirements, avoiding drift between design planning and implementation planning, and reviewing the results such that you get out the other end not just what you asked for, but what you actually wanted.

It bundles house style, agents, research skills, hooks, and all the other bits that used to be separate plugins into one package.

**NOTE:** `ed3d-plugins` is generally a more stable marketplace. If you'd like to track changes as they happen a bit more aggressively, take a look at [`ed3d-plugins-testing`](https://github.com/ed3dai/ed3d-plugins-testing).

## Using `plan-and-execute`
More in [the README for the plugin](plugins/plan-and-execute/README.md), and it's worth skimming, but here's a quickstart:

```
Rough Idea
    |
    v
/start-design-plan  -------> Design Document (committed to git)
    |
    v
/start-implementation-plan --> Implementation Plan (phase files)
    |
    v
/execute-implementation-plan --> Working Code (reviewed & committed)
```

**Customization:** Create `.ed3d/design-plan-guidance.md` and `.ed3d/implementation-plan-guidance.md` in your project to provide project-specific constraints, terminology, and standards. Run `/how-to-customize` for details.

## Installation

### Add the marketplace
```bash
/plugin marketplace add https://github.com/bpowers/ed3d-plugins.git
```

### Install the plugin
```bash
/plugin install plan-and-execute@ed3d-plugins
```

## Repository Structure

```
ed3d-plugins/
├── .claude-plugin/
│   └── marketplace.json
├── plugins/
│   └── plan-and-execute/
└── README.md
```

## Contributing
Issues and pull requests gratefully solicited.

## Attribution

`plan-and-execute` is derived from [`obra/superpowers`](https://github.com/obra/superpowers) by Jesse Vincent. The original plugin has been folded, spindled, and mutilated extensively.

Some skills (like `property-based-testing`) are derived from the [Trail of Bits Skills repository](https://github.com/trailofbits/skills).

## License

The original [obra/superpowers](https://github.com/obra/superpowers) code in this repository is licensed under the MIT License, copyright Jesse Vincent. See `plugins/plan-and-execute/LICENSE.superpowers`.

All other content is licensed under the [Creative Commons Attribution-ShareAlike 4.0 International License](http://creativecommons.org/licenses/by-sa/4.0/).

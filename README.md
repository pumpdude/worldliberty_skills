# worldliberty_skills

Openclaw skills for World Liberty Financial on-chain data.

## Skills

| Skill | Description |
|-------|-------------|
| [usd1-por](./usd1-por/SKILL.md) | Query USD1 stablecoin Proof of Reserves — reserves, multi-chain supply, collateralization ratio |

## Usage

Copy the skill directories into your Openclaw workspace:

```bash
cp -r usd1-por ~/.openclaw/workspace/skills/
```

Or configure `skills.load.extraDirs` in your Openclaw config to point to this directory.

## Source

Skills are extracted from [worldliberty/cre-por-dashboard](https://github.com/worldliberty/cre-por-dashboard).

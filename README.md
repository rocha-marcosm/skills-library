# skills-library
Compilation of Agents Skills


## Adding sources

```bash
git submodule add <repo_url> sources/<repo_name>
```

## Update sources

```bash
git submodule update --init --recursive --remote
```

## Skills to use

```bash
cd skills
ln -s ../sources/<repo_name>/skills/<skill_name> <skill_name>
```
## Git LFS 

Before you start please add `Makefile` to your project like in `ios-bootstrap` tutorial. 

---

Unfortunetely git LFS does not support `.framework` and `.framework.dSYM` extensions so we need to zip them and then add to `.gitattributes`. For zipping we use `carthage archive` function and then we're zipping all frameworks zips into one zip named `PreBuildFrameworks.zip`.

Git lfs initial setup:

```bash
	1. brew install git-lfs // instal lfs via brew
	2. git lfs install // will add required hooks into your git repository
	3. remove Carthage/ folder from repo first & commit (only if you keep that folder in repo) and then add Carthage/ to .gitignore
```

Steps to configure:

1) **After defining frameworks that you need in `Cartfile` please run `carthage update --platform iOS`**

2) **You need to define following lines at the top of `Makefile`, they are definitions of paths to carthage frameworks**

```bash
CARTHAGE_FRAMEWORKS=ls Carthage/Build/iOS/*.framework | grep "\.framework" | cut -d "/" -f 4 | cut -d "." -f 1 | xargs -I '{}'
CARTHAGE_ARCHIVES=ls PreBuiltFrameworks/*.zip | grep "\.zip" | cut -d "/" -f 2 | cut -d "." -f 1 | xargs -I '{}
```
3) **Now you need to define formula for archiving frameworks**

```bash
carthage-archive:	## archive carthage packages
	@echo "--- Archiving carthage packages..."
	@rm -rf PreBuiltFrameworks/*.zip
	@$(CARTHAGE_FRAMEWORKS) carthage archive '{}' --output PreBuiltFrameworks/
```
This command is archiving each framework and it's dSYM into `{FRAMEWORK_NAME}.zip` file.

4) **It's time to track zips in git lfs**

```bash
carthage-track:	## track and commit carthage frameworks
	@echo "--- Tracking carthage frameworks..."
	@git lfs track PreBuiltFrameworks/*.zip
	@git add .gitattributes PreBuiltFrameworks/*.zip
	@git commit -m "adding prebuild framworks"
```

After running this formula all frameworks are safe and sound in git lfs. 

5) **To extract frameworks please define commands below**

```bash
carthage-clean:	## clean up all Carthage directories
	@echo "--- Removing Carthage folder..."
	@rm -rf Carthage
```

```bash
carthage-extract: carthage-clean	## extract from carthage archives
	@echo "--- Extracting carthage archives..."
	@$(CARTHAGE_ARCHIVES) unzip -q PreBuiltFrameworks/'{}'.framework.zip
```

This command is cleaning up `Carthage/` directory and then unzip all frameworks into it.

**Important note** - you should run following commands on CI for each build:

```bash
@git lfs install
@git lfs pull
@make carthage-extract
```

If you've previously added complied frameworks to git repo please use [BFG tool](https://github.com/rtyley/bfg-repo-cleaner/releases/tag/v1.12.5) to remove them completely.

---

Complete Makefile:

```bash
SHELL := /bin/bash

CARTHAGE_FRAMEWORKS=ls Carthage/Build/iOS/*.framework | grep "\.framework" | cut -d "/" -f 4 | cut -d "." -f 1 | xargs -I '{}'
CARTHAGE_ARCHIVES=ls PreBuiltFrameworks/*.zip | grep "\.zip" | cut -d "/" -f 2 | cut -d "." -f 1 | xargs -I '{}'

bootstrap: ## Bootstrap lfs and carthage
	@echo "--- Pulling lfs files..."
	@git lfs install
	@git lfs pull

	@make carthage-extract

carthage-archive:	## archive carthage packages
	@echo "--- Archiving carthage packages..."
	@rm -rf PreBuiltFrameworks/*.zip
	@$(CARTHAGE_FRAMEWORKS) carthage archive '{}' --output PreBuiltFrameworks/

carthage-track:	## track and commit carthage frameworks
	@echo "--- Tracking carthage frameworks..."
	@git lfs track PreBuiltFrameworks/*.zip
	@git add .gitattributes PreBuiltFrameworks/*.zip
	@git commit -m "adding prebuild framworks"

carthage-clean:	## clean up all Carthage directories
	@echo "--- Removing Carthage folder..."
	@rm -rf Carthage

carthage-extract: carthage-clean	## extract from carthage archives
	@echo "--- Extracting carthage archives..."
	@$(CARTHAGE_ARCHIVES) unzip -q PreBuiltFrameworks/'{}'.framework.zip
```

Please also take a look at [Medium article](https://medium.com/@rajatvig/speeding-up-carthage-for-ios-applications-50e8d0a197e1)
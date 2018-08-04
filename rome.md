## Rome 

Before you start please add `Makefile` to your project like in `ios-bootstrap` tutorial. 

---

**Setup of Rome is very simple but as a first step you need to have `.aws/credentials` and `.aws/config` set up. To get proper configuration files please contact Szymon or Emil.**

1) **Install Rome & Setup Romefile**

- Add `pod Rome` to your `Podfile` and execute `bundle exec pod install`
- Define bucket

```ini
[Cache]
  S3-Bucket = rome-ios
```

- Define ignore map (optional)

```ini
[IgnoreMap]
  ReactiveCocoa = ReactiveSwift, ReactiveCocoa
```
- Define repository map (required when repository name is different than framework name)

```
[RepositoryMap]
  Moya = Moya, RxMoya
  RxSwift = RxSwift, RxCocoa, RxBlocking
  FirebaseAuthBinary = Firebase, FirebaseAuth, FirebaseCore, FirebaseCoreDiagnostics 
  FirebaseCrashBinary = FirebaseAnalytics, FirebaseCrash, Protobuf
  FirebaseDatabaseBinary = leveldb-library, nanopb, FirebaseDatabase, FirebaseInstanceID, FirebaseNanoPB
  FirebaseMessagingBinary = FirebaseMessaging, GTMSessionFetcher, GoogleToolboxForMac
  ios-snapshot-test-case = FBSnapshotTestCase
```

Mapping looks like `L = R` where

**L** - repository name

**R** - framework name (product)

2) **Add folowing formulas to begining of `Makefile`**

```bash
PREFIX=`swift --version | head -1 | sed 's/.*\((.*)\).*/\1/' | tr -d "()" | tr " " "-"`
ROME = bundle exec Pods/Rome/rome
```
3) **Define carthage bootstrapping goal**

```bash
carthage-bootstrap:	## Download dependencies, build missing and upload them to AWS 
	@$(ROME) download --platform iOS --cache-prefix $(PREFIX)
	@$(ROME) list --missing --platform ios --cache-prefix $(PREFIX) | awk '{print $$1}' | xargs -L 1 carthage bootstrap --platform ios --cache-builds
	@$(ROME) list --missing --platform ios --cache-prefix $(PREFIX) | awk '{print $$1}' | xargs -L 1 $(ROME) upload --platform ios --cache-prefix $(PREFIX)
```

**Important note** - Please make sure you execute this goal in bootstrap phase

4) Next time when you would like to update some dependency please run `carthage update xxx --platform iOS --no-build`. It will only update `Cartfile.resolved` then you want to use previously created make command, like `make carthage-bootstrap` and it will try to download updated version from cache and only if doesn't exist already it will be build locally and then uploaded to Amazon.

---

Complete Makefile:

```bash
SHELL := /bin/bash

PREFIX=`swift --version | head -1 | sed 's/.*\((.*)\).*/\1/' | tr -d "()" | tr " " "-"`
ROME = bundle exec Pods/Rome/rome

bootstrap:  ## Bootstrap Gems, Cocoapods and Rome
  @echo "--- Installing gems..."
  @bundle install --path vendor/bundle --quiet

  @echo "--- Updating CocoaPods repos..."
  @bundle exec pod repo update --silent

  @echo "--- Installing pods..."
  @bundle exec pod install

  make carthage-bootstrap
  
carthage-bootstrap: ## Download dependencies, build missing and upload them to AWS 
  @$(ROME) download --platform iOS --cache-prefix $(PREFIX)
  @$(ROME) list --missing --platform ios --cache-prefix $(PREFIX) | awk '{print $$1}' | xargs -L 1 carthage bootstrap --platform ios --cache-builds
  @$(ROME) list --missing --platform ios --cache-prefix $(PREFIX) | awk '{print $$1}' | xargs -L 1 $(ROME) upload --platform ios --cache-prefix $(PREFIX)
```
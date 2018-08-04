# Lovely Carthage

![MacDown logo](https://raw.githubusercontent.com/Carthage/Carthage/master/Logo/PNG/header.png)

In this article I want share my experience of building dependencies by using Carthage. First of all, Carthage shines with simplicity. It's **very** simple to start using some external dependency in a Xcode project just by adding proper line into `Cartfile` and running `carthage update`. But as we all know, life is brutal and sometimes we need to consider more complex examples. 

Let's assume there is a team of iOS developers. Tony, John and Keith are working iOS application with **~15** popular dependencies like Alamofire, Kingfisher, ReactiveCocoa etc... 

### What problems they might meet?
- **Different compiler** - some of libraries are written in Swift, which means each different compilator run is uncompatible with the others. This might be a huge problem if those developers use different version of Xcode. Each of them need to build his own versions of frameworks or use the same version of Xcode. 
- **Clean build time** - this is hot topic recently, sometimes we need to care about build time, especially on CI. Their client is very vulnerable in this point. Team decided that they don't want to spend a lot of time like 1 hour waiting for release build time so this issue might be critical.
- **Repository size** - some of developers preffer to include compiled frameworks in repository. Team is using free github plan, so their maximum repository size is 1GB. Storing frameworks in repo can lead to huge increase of its size, even around 5GB. Even if this would not be a problem cloning such repository would take a **lot of time**. This might have huge influence for a clean build time especially when using CI with virtual machines.
- **Updating frameworks** - without some extra work carthage recompiles **all** frameworks when you run `carthage update` or one framework if you run it for single dependency. At the begining of the project we do that very often. Team is looking for a solution to speed this up too.

*There is no free lunch...*  I agree but at the same time I believe sometimes it's worth to spend some time for improving your every-day tools. I've spend **a lot** of time experimenting with dependencies managers and caching their artifacts etc... Let me tell you about three popular solutions of maintaining carthage frameworks. 

**Before you begin**

- If you're not familiar with Cartahge please take a look at it's [repository](https://github.com/Carthage/Carthage) first 
- I won't consider storing of carthage frameworks directly in repository
	
## Naive approach

Let the story begin ... Tony is a team leader and he decided to use Carthage as a dependencies manager. He defined some rules for other developers when working with external frameworks:

- Add Carthage/Build to gitignore and include Carthage/Checkouts in repository
- When cloning repository for the first time - you need to run `carthage bootstrap` (rebuild all dependencies). CI would need to run that for each pipeline
- When updating framework please only update one framework like `carthage update ReactiveSwift`

Those are very simple rules, but what about their pros and cons?

Pros:

- Free (costs `0$` per month)
- Repository size would never increase dramatically

Cons: 

- Very long clean builds
- Absolutely no reuse of pre-compiled frameworks
- Others code in your repository

Let's compare this solution to problems that might occur:

| Problem  | Solved? |  What's missing? |
|---|:---:|---|
| Different compiler   | partially |  If there are two developers using the same Xcode there both of them need to recompile |
| Clean build time  | no | Builds would be very long, each time rebuilding all dependencies on CI |
| Repository size  | yes  | - |
| Updating frameworks| no | No improvement, developer need to rebuild framework and its dependencies when updating |

To sum up: their biggest problem in this approach is **time**. The only fully solved problem is repository size. CI build time would be very long and would increase proportionally with number of dependencies. As you can see there is still a lot to improve. Let's try something different...

## Git LFS ([tutorial](lfs.md))

Some day one of the developers - John - found that github allows storing large files in their LFS (large file system). He noticed that this might be great oportunity to start including pre-compiled frameworks in git repo but still keep it small. He modified Tony's rules a little: 

- Add **both** Carthage/Build **and** Carthage/Checkouts to gitignore
- When cloning repository for the first time - you **don't** need to run `carthage bootstrap` (rebuild all dependencies) but you need to extract frameworks from LFS
- When updating framework please only update one framework like `carthage update ReactiveSwift`, **some extra work is needed** - you need to archive those frameworks, zip them and upload to git-lfs (add to `.gitattributes`)
- **All of team members** must have the same swift compiler version (Xcode version)

This solution is much more complicated especially because of extra steps with zipping and uploading frameworks. There is a [great article](https://medium.com/@rajatvig/speeding-up-carthage-for-ios-applications-50e8d0a197e1) that describes this and offers some simple Makefile to automate this step. 

Pros:

- Repository size still not growing
- After clone and extract you're ready to go

Cons: 

- In most cases not free (costs `5$` per month after reaching 1GB on LFS)
- Each developer must work with the same Xcode version
- No mechanism for speeding up update of frameworks


Let's compare this solution to problems defined at the begining of the article:

| Problem  | Solved? |  What's missing? |
|---|:---:|---|
| Different compiler   | no |  All of developers must have same compiler version |
| Clean build time  | yes | CI would build only application code and reuse all dependencies pre-compiled for each pipeline |
| Repository size  | yes  | - |
| Updating frameworks| no | No improvement, developer need to rebuild framework and its dependencies when updating |

After all I think that this looks much better! Having fast clean builds is much more important for most teams than possibility to use different Xcodes between developers. They are still able to have differen versions installed and only switch between them for specific projects. I believe `5$` per month for LFS is not a big deal. So it's much better (and difficult) solution, but there is still a room for improvement ...

## Rome ([tutorial](rome.md))

So, time for Keith to show up. He appriciate other developers research but Keith cares a lot about team work. He thought that maybe it's possible to share different versions of pre-compiled frameworks compiled by different versions of swift compiler between different projects? That's a lot of variety, but thanks God there is a tool for that. It's called `Rome`. I highly encourage you to take a look at documentation on [github](https://github.com/blender/Rome). In general this tool share frameworks using Amazon S3 Bucket. Again, Keith changed the rules: 

- Add **both** Carthage/Build **and** Carthage/Checkouts to gitignore
- When cloning repository for the first time - you **don't** need to run `carthage bootstrap` (rebuild all dependencies) but you need download them from Amazon S3 
- When updating framework please only update one framework **version** like `carthage update ReactiveSwift --no-build` and then try to download it from Amazon and if it does not exist build it and upload
- You need to define `RepositoryMap` which tells Rome which dependencies compiled by Carthage you use 

By using some **very simple** helper script those rules seem to be almost as simple as the one from `Naive approach` section. I'm very impressed by this tool especially by the relation between amount of required setup work and gain that is given. To see more details please take a look at Rome tutorial in this directory. Let's see what are pros and cons of this solution:

Pros:

- Repository size still not growing
- After clone and download you're ready to go
- Share frameworks between all company developers (very simple update of frameworks because someone possibly already compiled proper version for you)
- Feel free to use different versions of Xcode
- Better knowlage of dependencies that you use because of `RepositoryMap`
- Ability to schedule building dependencies on CI and then using them locally

Cons: 

- Not free, but it's still cheaper than **LFS** (`$0.023 / GB`)

And comparison with an obvious result:

| Problem  | Solved? |  What's missing? |
|---|:---:|---|
| Different compiler   | yes | - |
| Clean build time  | yes | - |
| Repository size  | yes  | - |
| Updating frameworks| yes | - |

In my opinion this solution is the one that saves you a lot of hours spend on dependencies management. Of course sometimes you'll need to build on your machine / CI but you have guarantee that this job will be reused.

## Recap

So you already noticed that I believe Rome is the best solution for now and I highly encourage you to use this, but the story shows that there is always something we can improve. We should experiment with different approaches and pick the one that solves our problems. I believe that during reading a story of Tony, John and Keith you notied more than just the best friend of Carthage (Rome). It's about team work and improving team workflow. Those guys all the time tried to solve the problem of working together (with CI as fourth team member) and finally one of them found a solution that fits ideally to their problems!

I highly recomend you to take a look at quick tutorials for each solutions

Tutorials:

- [LFS](Rome.md)
- [Rome](Rome.md)

Useful links:

- [Carthage github](https://github.com/Carthage/Carthage)
- [Git LFS](https://git-lfs.github.com)
- [Medium article about Carthage + LFS](https://medium.com/@rajatvig/speeding-up-carthage-for-ios-applications-50e8d0a197e1)
- [BFG - tool for migrating to LFS](https://github.com/rtyley/bfg-repo-cleaner/releases/tag/v1.12.5)
- [Rome github](https://github.com/blender/Rome)
- [AWS credentials](https://aws.amazon.com/blogs/security/a-new-and-standardized-way-to-manage-credentials-in-the-aws-sdks)
---
title: "Start building a local workflow using Pi.dev"
datePublished: 2026-05-19T04:35:39.608Z
cuid: cmpc53u1k00ce2gjz5hxa684r
slug: start-building-a-local-workflow-using-pi-dev

---

[Pi.dev](http://Pi.dev) is exactly the coding agent I want to try and use in real work, simple and powerful. So I start to build a local workflow around it.

Safety first, I'm not confident enough to run Pi on my main PC which has access to all my private info and so on, and I'm lucky to have a spare machine to host it.

Also I don't want it to access the company git server directly (well, even I want I have no auth to do it :). So a gitea server host in my local docker and Pi will only work against that.

And before we starting the real job, I set several rules about the project and also let it read the Unreal official C++ coding standard and make them part of the promotion. Since I have lots of powershell scripts ready to use for the project, it makes things much easier to start.

One interesting thing is that while I tried using "pi install" it failed due to some conflict about the NVM for Windows issue. As before, I google for a quick solution or work around, but failed. So instead, I simply asked Pi to fix the issue by itself, and it's done in one shot. Pretty impressive and give me more confidence to rely on it to accomplish complex stuffs.

After that, I asked Pi to create a PR once it finished the job, and also register a webhook for any comments on its own PR and try to resolve the reviews automatically. There're some back and forth, but nothing really blocked me.

Well after a short afternoon, now I have an agent intern who can do some simple jobs as requested and I could just review and approve on my gitea. Very interesting and can't wait to explore more.
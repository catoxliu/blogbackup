---
title: "How to automatically update a "private" vscode extension"
seoTitle: "automatically update a vscode extension"
datePublished: 2022-05-10T12:55:05.898Z
cuid: cl305nw7b00myuynvebpu64m7
slug: how-to-automatically-update-a-private-vscode-extension
tags: vscode, vscode-cjevho8kk00bo1ss2lmqqjr51, vscode-extensions

---

### The right way is to publish it into official market, but if you can't wait or don't want to, here we go:

Generally speaking, this could be as simple as:
1. a regular task to check the server.
2. Download the extension package if newer version available.
3. Install that extension package.

You can have all kinds of different impelementations for step 1 or 2, I wouldn't go into details here.

The challenge part is step 3.
Previously I just ask my colleagues to manually install the vsix file I shared with them, since it's just several simple clicks on UI. But after 3 releases (bugs/feature request) in a week, this process becomes a bit unacceptable. I have to figure out a way to upgrade it automatically to prevent the fragmentations and also make other's life a bit happier.


**So the first question is can we install it through a script/cli?** Well, the answer is not that obvious to me (due to my poor google skills), but hey, vscode is opensource, all the answers are already there in the code. So I go to github and quickly go through the issues (always assume somebody else may have the very same issue before), and I find someone report a bug of this command:
```
code --install-extension path_to_extension_vsix --force
``` 
wait, I don't know that and also, it's very clear then it's doable. Everything follows up is simple and boring:
- a lot more help info through `code --help`.
- searching through the code by "install-extension"
- ha, cli command: "**workbench.extensions.installExtension**" (in fact, it's right there in the [offical documentation](https://code.visualstudio.com/api/references/commands) :))
- basically just call this vs command in the extension code with the vsix file URI, it's done.

Can't believe I go through such a long way (again, poor google skills), but a lot of other stuffs learnt.

---
title: "auto and auto& nightmare"
datePublished: 2022-07-29T01:01:17.222Z
cuid: cl65re2qq03gcglnv86ezezik
slug: auto-and-auto-reference-nightmare
tags: cpp, unreal-engine

---

I'm new to Unreal Engine editor, and I have the task to add some customer menu into an existing context menu. So I look at some docs and read the source code, with new UE5, it's pretty simple and straight forward

```
UToolMenu* SourceControlMenu = UToolMenus::Get()->ExtendMenu("ID_OF_MENU");
FToolMenuSection& Section = SourceControlMenu->AddSection("MySection");
Section.AddEntry(//my custom menu here//);
``` 
So I jump to this work and made the first mistake, I didn't copy paste code and think I'm smart to ignore unreal style guide to just do it in my way.
```
auto SourceControlMenu = UToolMenus::Get()->ExtendMenu("ID_OF_MENU");
auto Section = SourceControlMenu->AddSection("MySection");
Section.AddEntry(//my custom menu here//);
```
Compile it, pass in the first try, and run it, breakpoint hitted, it runs my code correctly without any issues, cool, seems work done.

But wait, where is my menu, it's not showing up on the UI. What's going wrong here?

...

After a bad sleep following several hours debugging, searching and asking around online and pulling my hairs, I just realise it's a so simple mistake I'm making here.
Looking at the function again
```
struct FToolMenuSection;
FToolMenuSection& UToolMenu::AddSection(...)
```
So the return type is a reference, and since it's a struct, then you can implicitly copy the value through a struct reference, so what I'm doing there is making a copy of the section.

I know this is not right to some people, as I quote "I would totally expect the auto to be of the same type of the return type i.e. a reference to FToolMenuSection" (Maybe that's why Unreal suggest to not use auto if possible)

To make it clear:
```
FToolMenuSection& OriginSection = SourceControlMenu->AddSection("MySection");
auto CopiedSection = OriginSection; //it's not the original section.
// Type of OriginSection is FToolMenuSection&
// Type of CopiedSection is FToolMenuSection
```
So I'm adding my custom menu into a local variable section which is disposed right after my function... 

Besides the mistake I made here, is there a way to protect us from this? yes, there is, you can explicitly delete the constructor of copying from reference
```
struct FToolMenuSection
{
   FToolMenuSection(FToolMenuSection& ) = delete;
   FToolMenuSection(FToolMenuSection&& ) = delete;
}
FToolMenuSection& OriginSection = SourceControlMenu->AddSection("MySection");
auto CopiedSection = OriginSection; //compiler errors here, don't know how to copy from a reference here
``` 

And I tried to do this for Unreal Editor, but can't. Since in the TArray implementation, they need insert fucntion working with reference which needs the copy by reference.

So what I learn today. Always copy paste code where you can :). And maybe follow the style guideline where you can ~


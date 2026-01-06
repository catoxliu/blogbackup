---
title: "Check your MSVC version if you having errors when using new feature in C++ 20"
seoTitle: "error C2039"
seoDescription: "error C2039: 'format_string': is not a member of 'std'"
datePublished: Thu Jun 29 2023 02:28:38 GMT+0000 (Coordinated Universal Time)
cuid: cljgj0s9l000609jw16ipau10
slug: check-your-msvc-version-if-you-having-errors-when-using-new-feature-in-c-20
tags: cpp, c2039

---

Getting this error while compiling an open-source code.

`error C2039: 'format_string': is not a member of 'std'`

Google results will tell you to make sure you're using C++20 flag in the compiler since it's a new feature. I'm pretty sure it's not the issue since `std::format` itself works fine in the code. But I double checked anyway and try all solutions I could find online (big mistake, trying to be lazy : ). I checked all the dependencies and VS packages needed. I even re-install Visual Studio 2022.

After wasting half a day (and a good sleep) without any progress, I started to be serious about this error.

So back to the error, what's this `std::format_string`. According to [std::basic\_format\_string, std::format\_string, std::wformat\_string -](https://en.cppreference.com/w/cpp/utility/format/basic_format_string) [cppreference.com](http://cppreference.com), it's a new feature in C++ 20 standard library and should be in the &lt;format&gt; header file. (Yes, I do explicitly include this header file in the source code and nothing changed).

The definition of it is:

```cpp
template< class... Args >
using format_string = basic_format_string<char, std::type_identity_t<Args>...>;
```

Then I open the format header in the MSVC and try to search for it. Of course no exact match, so I just tried the patterns like "basic\_format\_string". Finally, I find this:

```cpp
template <class... _Args>
using _Fmt_string = _Basic_format_string<char, type_identity_t<_Args>...>;
```

Surprise, surprise, MSVC has their own naming different from the standard library : )

Then changing the `std::format_string` to `std:_Fmt_string` fix the compilation errors. But why I'm getting this while it seems to work fine for others? I realise it's the MSVC version I'm using, it's 14.34.31933 even though the latest one is 14.36.32532.

Then I reinstall MSVC again and this time to ensure I have the latest version of MSVC installed. Then in 14.36.32532, the format header change it to `std::format_string` and now the original source code is compiled without any issues.

Lesson learnt : )
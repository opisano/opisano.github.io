---
layout: post
title:  "Intro to Unicode"
date:   2025-04-29 19:03:59 +0200
categories: Programming
---

When talking to a former colleague, I realized that Unicode is somewhat misunderstood. I don't pretend to know everything is to be known about, but I had the opportunity to play extensively with it, so this post is going to recap the concepts of Unicode and (hopefully), teach you one thing or two. At least from the point of view of a software developer. Fasten your seat belt and have fun.


# First thing to know: Unicode is not an encoding 

Ok, this one may sound a bit strange if you previously read the documentation of a major operating system vendor, or some coffee-flavoured programming language but let's be clear :

 * Unicode is not a text encoding 
 * Unicode *characters* are not 16 bits wide. 

If any of these two statements hurts your beliefs, I am deeply sorry (no I am not, I am just being polite). It is time to take the red pill and to get out of the matrix. 


# How Unicode should be viewed 

To get a grasp of Unicode, you can go to [https://symbl.cc/en/unicode-table/](https://symbl.cc/en/unicode-table/). Scroll the page is little bit. The heart of Unicode is just a giant table, with *symbols* associated to *numerical values*. For instance, the symbol `é` is associated to the number `U+00E9` (which is just a fancy way of saying that it is a hexadecimal value). There are 1 111 998 possible Unicode symbols.

In particular, this does not say anything about: 

 * How many bytes are used to store this value.
 * How these bytes are stored in memory.

Which are what constitutes an encoding. 

Ok now, I am going to make a promise. 

For the remaining of this post, I won't use the word *character* anymore. Because this word is *polysemic*. It bears multiple meanings. Too many. Instead, I am going to introduce some replacement words. 


## Graphemes 

Ok let's go back to our giant table.

The symbols that are displayed in the table are called **graphemes**. A grapheme is a single graphical unit that a reader recognizes as a single element of a writing system. For a western person, it could be an alphabetical letter or a punctuation sign. For another writing system, it may be something else.

To get back to my previous example, `é` is a grapheme that was given a description: *small letter E with acute*.


## Code points 

Now that let's define the numerical value. The correct name for that is *Unicode code point*. A code point is the basic part of information. A text is a sequence of code points. A grapheme can be defined by one or many code points. For instance, the `é` grapheme can be written in two ways:

 * using the `e` code point (U+0065), followed by the `´` code point (U+0301).
 * using a single code point (U+00E9)

Note that I still haven't said anything about the way code points are stored into memory…


## Glyphs 

Effectively, graphemes can be obtained by combining multiple graphical parts together into a single representation. Those parts are called *glyphs*. Glyphs are important if you're performing text rendering, but can be ignored for text processing purposes. 

What should be aware of, is that the same text can be obtained by different manners, which may lead to surprising results if you are trying perform text comparisons (Python code below):

{% highlight Python %}
>>> '\u00E9'
'é'
>>> '\u0065\u0301'
'é'
>>> '\u00E9' == '\u0065\u0301'
False
{% endhighlight %}

Imagine trying to access a file on your disk, but providing the wrong code point sequence. Imagine typing a URL in your browser, and accessing a phishing website…


# Normalization 

To avoid all these issues, Unicode text should be normalized prior to being compared. There are several normalizations possible, which can be divided in two groups:

 * Either we decompose text into the maximum number of code points 
 * or we compose into the minimum number of code points.

and then comparison is performed.


# A bit of history 

Ok, now that you are definitively disgusted, let's talk a bit about history. At the time I write this post, the latest Unicode version is 16.0. Version 1.0 was released in 1993, which means it now contains a lot more than it used to be. In fact, until 2001, it contained less than 65 536 entries in its table, which meant that 16 bit values were enough to encode text. 

This is why you may encounter some outdated documentation that used to state that unicode *was* a 16 bit encoding. It is known as UCS-2 and was declined in two variants: UCS-2 LE (little endian), and UCS-2 BE (big endian). UCS stands for **Universal Character Set** (sic). Today this encoding is considered obsolete, since it can only represent a fraction of Unicode. 

UCS-2 required to double the memory used for storing text and to change all your text processing functions. for instance, instead of the venerable C function `strlen()`, one should use the `wcslen()` function against an array of `wchar_t` elements.

Windows NT was the first operating system to fully embrace UCS-2 and to set `sizeof(wchar_t) == 2`. 

Since 2001, UCS-2 has been superseeded by UCS-4, and some non-Windows systems define `sizeof(wchar_t) == 4`, but the idea of using 4 times more memory to store text was not acceptable for most of us, so someone had to invent a better way 


# Unicode encodings 

Finally, I am going to talk about encodings! In the Unicode world, they are called **Unicode Transformation Formats** and come into different flavours: UTF-8, UTF-16 and UTF-32. They can represent the whole Unicode set. If Unicode by itself is not an encoding, it provides several of them. 

They all decompose code points into one or many **code units**. A code unit can be viewed as a 8/16/32 bit value that is used to encode a part or the entirety of a code point. 


## UTF-8

UTF-8 uses between one and four 8 bit code units to encode a code point. It has several advantages over the other two: 

 * It is backward compatible with US-ASCII (in fact any US-ASCII text is also a UTF-8 text).
 * It doesn't have endianness issue, since every code unit is one byte. 
 * Most of the C library such as `strlen()` are compatible with it.

It has also disadvantages for languages that don't use the latin alphabet. For instance in Japanese, the same text can occupy more space in UTF-8 than in UTF-16.

To encode a code point into UTF-8, count how many bits are needed to represent the code point, and use the pattern described below

| Bits needed   | Binary representation                      |
|---------------|--------------------------------------------|
|  up to 7 bits | 0XXX XXXX                                  |
|  8 to 11 bits | 110X XXXX, 10XX XXXX                       |
| 12 to 16 bits | 1110 XXXX, 10XX XXXX, 10XX XXXX            |
| 17 to 21 bits | 1111 XXXX, 10XX XXXX, 10XX XXXX, 10XX XXXX |


Just by performing a bit mask, one is capable of decoding UTF-8:

 * If value is < 0x7F, it is a one code unit sequence
 * If value starts with 10, it is the continuation of a multiple code unit sequence.
 * If value starts with 110, it is the first of a two code unit sequence
 * If value starts with 1110, it is the first of a three code unit sequence
 * If value starts with 1111, it is the first of a four code unit sequence

Another disadvantage of UTF-8 is that since some bit patterns are invalid, a text input has to be validated (checked for correctness) prior to being used. 


## UTF-16

Remember that I told you that UCS-2 was obsolete? What about software that had adopted it? A superset using a variable-length encoding similarly to what UTF-8 is to ASCII was needed. UTF-16 is that. It uses 16 bit code units and comes in two variants: big and little endian.

Any software that used UCS-2 has been upgraded to UTF-16. For Windows, it was Windows 2000. UTF-16, like UTF-8 has some invalid bit patterns and requires to be validated. 


## UTF-32

For what is worth, UTF-32 should be viewed as identical to UCS-4. Any code point is encoded into a single 32bit code unit. Every code point can be retrieved by random access, in O(1), at the expense of memory consumption. This encoding is used for in-memory processing, but is almost never transmitted on the network.


# That's all folks!

I hope you learnt new things in this post. Thank you for having read it until the end.
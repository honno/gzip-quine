---
layout: post
title: How to make compressed file quines, step by step
description: Create a GZIP file that infinitely contains itself!
image: "quine_gz.png"
---

{% assign diagram_dir = "posts/compression_quines" %}
{% assign quine_gz_path = "/downloads/quine.gz" %}

{% capture quine_asciianim %}{% include {{ diagram_dir }}/gzip_quine_anim.html %}{% endcapture %}
{% include {{ diagram_dir }}/gzip_quine_anim_container.html src=quine_asciianim %}

You may of come across a compressed archive file (`.zip`, `.tar.gz`, `.rar` etc.) that [infinitely contains itself](https://alf.nu/ZipQuine) and thought that looks neat.

You may even want to make your own---if you have suspiciously too much time on your hands and like to jump into black holes of pointless endeavours, then let me welcome you comrade. This post will explore everything you need to know to be on your merry way creating these quines.

Much of the credit goes to folks much smarter than myself (they will be introduced); this tutorial is meant to curate previous work and literature as much as it is for myself to educate you. The goal here is to allow for any curious, technically-minded newcomer to make sense of all the concepts involved in creating compression quines.

Particular thanks goes to Russ Cox who [wrote on the subject of compressed files quines](https://research.swtch.com/zip). This post is essentially a more unassuming approach then his; nonetheless I would encourage reading both articles.

A word of warning for those inexperienced in Lempel-Ziv and Huffman Codes, compressed archive specs like DEFLATE, and editing files on a binary-level---**there is a lot to learn, understand and work through**. Although a very educational process (and I think a lot of fun), there is little direct practical achievement here. For example, I took more than a month to just create the petty 204-byte [quine.gz]({{ quine_gz_path }}).

**A GZIP quine will be the main subject of this article.** It will be the easiest format to create a quine with, as the formt does not concern itself with archiving a collection of files like ZIP does. Once you've learnt how to make a GZIP quine, you will feel confident to explore other formats and specifications.

Also mind, GZIP is well-supported on popular Linux distributions. About Windows and Mac---I don't know, but I'm sure there'll be tools available if you search around.

Without further ado, let's get started!

* TOC
{:toc}

## Understanding quines

**Compressed archive files can be considered as *programs*.** We'll get back to how that is soon, but first you need to know that most programming languages can be used to create self-replicating programs. So when the source program executes, it will produce an exact copy of it's own source.

Never written one yourself? I implore you to try making a quine using your programming language of choice and come back here when you're done. It can take a while so don't feel bad if you struggle to get there.

{% capture loading_asciianim %}{% include {{ diagram_dir }}/loading_anim.txt %}{% endcapture %}
{% include widgets/asciianim/container.html src=loading_asciianim %}

It can be fun and educational to discover things completely by yourself, but it's certainly not paramount. So a few hours later when you're banging your head against the keyboard, feel free to cheat a bit and check out [this Rosetta Code page](https://rosettacode.org/wiki/Quine), which provides examples in many languages. Excellent further reading could be [this extensive look at quines](http://www.madore.org/~david/computers/quine.html) by David Madore.

## GZIP file structure: Part 1

In a literal sense, a file consists of ones and zeroes; it's up to the software that's reading the file to translate those bytes into useful information.

Here the binary representation of [quine.gz]({{ quine_gz_path }}):

{% include {{ diagram_dir }}/gzip_quine_source.html %}

That's utter nonesense at this point, so don't worry; the endgame of this article is to reconstruct the above. We're going to abstract the information (metadata) and instructions (*print* & *repeat* opcodes) of GZIP files before encoding them.

Software that decompresses a GZIP file assumes a specification has been followed by the software that originally compressed the original file to create it so that the file's bits become meaningful. [RFC 1952](https://tools.ietf.org/html/rfc1952) is the specification that should be strictly adhered to (although bear in mind that it's not uncommon to see minor deviations).

GZIP files comprise of metadata in a header and trailer, with compressed data in-between.

The composition of the header/data/trailer would look something like this:

{% include {{ diagram_dir }}/file_structure_example.html %}

{% assign skip_anchor = "how-is-a-compressed-file-like-a-program" %}

If that's a bit confusing, let me explain (feel free to [skip](#{{ skip_anchor }})).

* Bytes are ordered right-to-left, then loops at the right again, moving a line down. Just like reading English.
* Each row truncates at 16 bytes.
* The x-axis (represented in *hex*) indicates how many bytes away from the row's starting byte-position the byte is.
* The y-axis (represented in *octal*) indicates the row's starting byte-position, i.e:
   * The first row's y value is `0o000`, as 0 bytes preceed it.
   * The second row's y value is `0o020`, as 16 bytes preceed it.
   * The fifth row's y value is `0o100`, as 64 bytes preceed it.

So in the above example:

* The header occupies `(0, 000) -> (2, 020)` (19 bytes)
* The compressed data occupies `(3, 020) -> (3, 300)` (177 bytes)
* The trailer occupies `(4, 300) -> (B, 300)` (8 bytes)

## Compressed data as a program
{: id="{{ skip_anchor }}"}

The Collins Dictionary definition of a "program" is as follows:

> A set of instructions for a computer to perform some task.

Compressed data found in GZIP files *is a program* in this regard. It comprises of two instruction types:

1. *Print (X)*

    Print the following X amount of *input* to the *output*.

2. *Repeat (X, Y)*

    Starting from Y amount away from the end of the *output*, repeat the following X amount of *output* to the *output*.

I hear you muttering "huh?"---to better understand the nature of these commands, **please try out [this excellent interactive demo](https://wgreenberg.github.io/quine.zip/)** by Will Greenberg.

Go create a quine with these two instructions (it rather simplifies things).

{% include widgets/asciianim/container.html src=loading_asciianim %}

<a name="shorthand"></a>
Here's a solution by Russ Cox:

{% capture lz_quine_table %}{% include {{ diagram_dir }}/lz_quine_table.html %}{% endcapture %}
{% include widgets/collapse/collapsible.html content=lz_quine_table %}

We're using the short hand `P` for *Print*, and `R` for *Repeat*. Repeat arguements are given a single arguement (`i`), which denotes we're repeating the `i` amount of *output* from `i` amount away from the end of the *output*---basically it's the `X` *and* `Y` value.

So we can write compressed data that executes (rather, decompresses) to itself---a quine! But how about *expanding* into a fully-formed file; how can compressed data output both itself *and* the metadata of a GZIP file?

## Compressed files as a program

Cox notes the above example contains a *6-step* sequence that prints a larger *8-step* sequence:

{% include {{ diagram_dir }}/lz_slightly_larger_example.html %}

He explains it's **critical for a sequence like the above to exist for compressed file quines to be possible**. Output lags behind the input before the last instruction (the `R4`), only to jump ahead of the input. This demonstrates that arbitary data, such as a header and trailer of a GZIP file, could be purposely outputted at the head and tail ends of the compressed data.

The `P0`---"print nothing" instruction---used in Cox's compressed data quine can actually be swapped for a header and trailer.  Cox thus creates a quine-ish program that "expands" itself with an arbitary prefix `aa bb cc` (the header) and suffix `xx yy zz` (the trailer):

{% include {{ diagram_dir }}/lz_arbitary_quine_table.html %}

A framework for a self-expanding compressed file! Nearly at least, as headers and trailer tend to be of varying lengths.

A generalised program that will work for any sized header and any sized trailer would be extremely useful. It would certainly be possible with variable arguments to `P` and `R` instructions. For example, the start of the file quine could look like this:

{% include {{ diagram_dir }}/file_quine_hint.html %}

Where `..H..` is a header of variable length and `h` is the length of the header.

So yes, I am going to ask you to try create one yourself. "You can do it!" And if you can't, don't sweat it.

{% include widgets/asciianim/container.html src=loading_asciianim %}

Cox's solution is as follows:

{% capture file_quine_table %}{% include {{ diagram_dir }}/file_quine_table.html %}{% endcapture %}
{% include widgets/collapse/collapsible.html content=file_quine_table %}

And there you have it. Translate something like this into GZIP-compliant binary and you're done!

But hold on hotshot 'coz there's a ways to go yet. Probably a good time to [make yourself a cuppa](https://www.youtube.com/watch?v=lNO_v8L2vI4).

## GZIP file structure: Part 2

We're going to deep dive on the GZIP specification, [RFC 1952](https://tools.ietf.org/html/rfc1952), and the DEFLATE  specification (we'll get to that in a sec) it refers to, [RFC 1951](https://www.ietf.org/rfc/rfc1951.txt). You'll want to open these up for heavy reference. I'm basically asking you to recreate [quine.gz]({{ quine_gz_path }}).

We'll be hex editing files, for which there are various solutions available; using the Unix tool [tweak](https://www.chiark.greenend.org.uk/~sgtatham/tweak/) worked well on my end. Emacs users can enjoy `hexl-mode`. Anything you can find for your OS should do---there's not a great deal of features needed for a hex editing tool anyway.

**I highly recommend setting up the [infgen tool](https://github.com/madler/infgen) by compression king Mark Adler**, which is a DEFLATE decompresser that outputs to a readable format (it was kindly made just for educational purposes). I found it extremely useful as I could verify my manual DEFLATE coding was working as intended, which gets important especially for the awkward method of writing *Repeat* instructions we'll be employing.

Also, maybe try reading Adler's [puff.c](https://github.com/madler/zlib/blob/master/contrib/puff/puff.c) DEFLATE-inflater. It was written to employ simple logic so folk could understand how DEFLATE works.

### Header

Section 2.3 of RFC 1952 (or `GZIP:2.3`, which is how I'll denote sections like so from now on) states the members available in a GZIP file header. The base header looks like this:

{% include {{ diagram_dir }}/gzip_header_basic.html %}

That could just be it for our quine. However, I would add a `FNAME` attribute so the decompressed outfile that the decompressing tool generates is of the same name as the origianl infile. My `quine.gz` is encoded like so:

{% include {{ diagram_dir }}/gzip_header_fname.html %}

Read up on the GZIP spec and translate the above to the respective binary!

I would suggest writing/typing in seperated 8-bit/1-byte blocks (using with the left-most bit as the [MSB](https://en.wikipedia.org/wiki/Bit_numbering#Most_significant_bit)), and then converting to hexadecimal.

Let me give you some values for a few fields:
* `CM` value of 8 as it denotes the DEFLATE compression method we'll use.
* `MTIME` value of 0 as it says no timestamp is available.
* `XFL` value of 0 because it's not really meaningful.
* `OS` value of 255 because who cares ¯\\_(ツ)_/¯

{% include widgets/asciianim/container.html src=loading_asciianim %}

The solution:

{% capture gzip_header_fname_source %}{% include {{ diagram_dir }}/gzip_header_fname_source.html %}{% endcapture %}
{% include widgets/collapse/collapsible.html content=gzip_header_fname_source %}

We have a header! Note that I'll be using the above in my examples from now on.

We can't quite determine the trailer yet, as it's two set fields---`CRC32` and `ISIZE`---are both determined by the contents of the "original, uncompressed file"---yes, that input file is also the output file. So essentially writing the trailer will have to come once we know everything else about the quine we're writing (spoiler: it will get confusing).

### DEFLATE-compressed data stream

As we mentioned, compressed data is essentially only comprised of *print* and *repeat* instructions. But it is actually a *DEFLATE* data stream .

And even them I'm fudging it a bit, because *technically* it can be any compression method you like (denoted by the `CM` block in the GZIP header). For modern GZIP compressors and decompressors (tools tend to do both), I believe most programs just expect use of DEFLATE anyway, so that's what we are caring about today. DEFLATE is not ubiquitous in the wider world of "compressing stuff", but it's concepts are shared by many modern compression efforts.

Right! So before we "code" in DEFLATE, we need to understand it's intended use---loseless-ly compressing files to save up on space. It's beauty is in how it harmonizes two effective algorithms in "reducing redundancy" in a set of information (in DEFLATE's case, binary files):

* Dictionary-based compression (by way of Lempel-Ziv)
* Entropy-based compression (by way of Huffman coding)

(You should be somewhat familiar with Lempel-Ziv, as the *Print* & *Repeat* opcodes are essentially the end result of LZ algorithms.)

I'm not going to teach you about the design choices for DEFLATE here, so if you have no idea how it works in the broad-stokes, maybe check out the following:

1. A little article titled ["The Elegance of Deflate"](http://www.codersnotes.com/notes/elegance-of-deflate/) by Richard Mitton, which aptly explains *why* DEFLATE is so effective.
2. Educator-extraordinaire Prof. David Brailsford explaining the compression concepts [Lempel-Ziv](https://www.youtube.com/watch?v=goOa3DGezUA) and [Huffman coding](https://www.youtube.com/watch?v=umTbivyJoiI).

#### Block types

Blocks are the container for the instructions and arguments of a set of data, starting with a `BFINAL` bit that denotes if it's the last block in the compressed data stream, and is followed by two bits that denote the `BTYPE` (block type) of the block, as explained in `DEFLATE:3.2.3.`.

There are three `BTYPE` blocks in DEFLATE:

* Stored block (`BTYPE=00`)

   The *Print (X)* command. Also referred to as the "literal" block, as it really says "treat the next `X`-length bytes literally, not as instructions".

* Compressed block

  For our purposes, basically the *Repeat (X, Y)* command. The important distinction here is that **the instructions are Huffman coded**. There are two subsets of compressed blocks:

  * Fixed (`BTYPE=01`)
  
    A Huffman tree pre-built into the DEFLATE specification is used.
  
  * Dynamic (`BTYPE=10`)
  
    A compressor-defined Huffman tree is first defined, to be used with subsequent compressed data.
    
We'll just parallel the *Repeat (X, X)* instructions (like used in the [original compressed data quine example above](#shorthand)) with fixed compressed blocks. We don't want to bother with defining a Huffman tree like with a dynamic block, at least when manually writing a quine.

#### Print

Our first print sequence comes from the top of the Cox quine:

{% include {{ diagram_dir }}/file_quine_hint.html %}

For us that comes out to the following:

{% include {{ diagram_dir }}/gzip_print_header_chart.html %}

The 24 argument for the *Print* instruction is the number we desire: the header is 19-bytes long, and 5-bytes long `Print 24`. All literal blocks start with 5 bytes:

{% include {{ diagram_dir }}/literal_block_explaination.html %}

Read `DEFLATE:3.2.4.` and write the corresponding hex.

{% include widgets/asciianim/container.html src=loading_asciianim %}

{% capture gzip_print_header_answer %}{% include {{ diagram_dir }}/gzip_print_header_answer.html %}{% endcapture %}
{% include widgets/collapse/collapsible.html content=gzip_print_header_answer %}

Did you mess up the byte order? `DEFLATE:3.1.1.` states how DEFLATE packs it's bytes.

... but honestly it's contents still confuse me. For literal blocks, I just think of writing an unsigned integer in bits, with padding to the left to fit the 2-byte boundary. It's in a little-endian byte order, and if you have no idea what that is, maybe [Dr. Steve Bagley can help you out](https://www.youtube.com/watch?v=NcaiHcBvDR4)?

{% include {{ diagram_dir }}/literal_order_explaination.html %}

Here's what our file should look like:

{% include {{ diagram_dir }}/gzip_print_header_source.html %}

#### Repeat

***Repeat (24, 24)***

Up next is our realization of the Coxian `Rh+1` command:

{% include {{ diagram_dir }}/gzip_repeat_24_chart.html %}

To "print the following 24 bytes from 24 bytes back the output", you can simply write the following:

{% include {{ diagram_dir }}/repeat_24_answer.html %}

Let me unpack that for you:

{% include {{ diagram_dir }}/repeat_24_answer_explaination.html %}

If you have no idea how DEFLATE's fixed Huffman codes work, you may want to read [this excellent DEFLATE reference](http://calmarius.net/?lang=en&page=programming%2Fzlib_deflate_quick_reference) by Calmarius. He importantly offers a clear explanation of how the DEFLATE alphabet works (needed for literal/length & distance codes), as well as the specification generally. Don't mind the ZLIB stuff.

Take your time~

***Repeat (12, 24) (12, 24)***

Anywho, of course I'm not going to give you the actual solution just like that. The problem with this `R (24, 24)` is that only 3 bytes long, which messes up the Coxian quine that presumes the same size for all *Print* & *Repeat* instructions.

As the *Print* commands always start with 5 bytes, we'll use that as the constant "unit" of our program, i.e. the `1` in `Ph+1` & `Rh+1` is 5 bytes.

Fortunately, we can write a different representation of `R (24,24)` in the way of `R (12, 24) R (12, 24)`. Spoiler: it comes to 5 bytes. You can infact fudge all the Coxian *Repeat* instructions like this.

You know what your task is. I'll give you a little template to help you out:
	
{% include {{ diagram_dir }}/repeat_12_hint.html %}	

{% include widgets/asciianim/container.html src=loading_asciianim %}

And so, your quine should look like this:

{% capture gzip_repeat_24_source %}{% include {{ diagram_dir }}/gzip_repeat_24_source.html %}{% endcapture %}
{% include widgets/collapse/collapsible.html content=gzip_repeat_24_source %}

Now you're set, at least until...

***Repeat (13, 13)***

A translation of the Coxian `Rt+1` for us will be R `(13, 13)`, seeing that GZIP trailers are always 8 bytes and we've established "1" is 5 bytes. This means we can't neatly split this into two of the same literal & distance pairs.

No fret, because something like `Repeat (6, 13) (7, 13)` works just as well!

### Bringing it all together

{% include {{ diagram_dir }}/gzip_quine_chart.html %}

You should be able to translate everything here, except for the CRC32 field (which we'll get to soon!), so... go do that.

{% include widgets/asciianim/container.html src=loading_asciianim %}

Here you go. Mind I've replaced the 4-byte CRC with the `** ** ** **` placeholder.

{% capture gzip_quine_source_no_crc %}{% include {{ diagram_dir }}/gzip_quine_source_no_crc.html %}{% endcapture %}
{% include widgets/collapse/collapsible.html content=gzip_quine_source_no_crc %}

### Finding the checksum

Actually, at this point your quine will work on a lot of popular decompressors (if they have options to not verify the checksum). Congratulations!

... I wasn't quite satisfied with that, so I decided to see about finding a CRC32 that, when found in both in quoted and actual part of the GZIP quine, will work. A self referential checksum, so to speak.

I won't get into how a cyclic redundancy check works---you can read [these succinct notes](http://heather.cs.ucdavis.edu/matloff/public_html/Networks/CRC/Old/ErrChkCorr.html) by Prof. Norman Matloff---but for all intents and purposes you can simply bruteforce all possible values until something works.

For the 32 bits of CRC32, there are 2<sup>^32</sup> (over 4 billion) permutations you can test. For modern tech that ain't too much of an effort, so I wrote a [dirty Python script utilizing multi-processing](https://github.com/Honno/self-referential-crc-bruteforce/blob/master/find-crc32.py). It works by embedding a given CRC32 value into defined quoted & actual positions in a provided template file, checking if the CRC32 checksum of the modified template matches the given CRC32 value.

I plonked it on a Google Cloud server to run on (they offer more than enough free credit), so as not to see my not-so-modern Pentium CPU bleed to death. If you were going to use it on a basic computer, you'll likely want to modify the script to add appropriate sleep instructions if you don't want to repurpose your processor as a frying pan.

Oh yeah and, I'm not a mathematician, so maybe this is overkill. Can someone tell me if there's some algebra involving modulus or whatever that could actually solve for a self-referential CRC32 answer?

Anyway, if you haven't been cheating all this time with the `quine.gz` source code, I'll save you the hassle and tell you the CRC32 value that we want for our quine: `0x ff 79 ff a9`.

## Conclusion

I've got actually useful work to get on with, so I'm leaving the challenge of writing a programmatic solution for the creation of compressed file/archive quines with variable arguments to y'all. Have some fun with it, but make sure it's in Python ;\)

Of course, Russ Cox actually has already done this by way of `rgzip.go` (search his *["Zip Files All The Way Down"](https://research.swtch.com/zip)* post for it), though mind it's in some old version of Go that is hard comprehend at this point. Take note that he utilizes dynamic compressed blocks!

Infact, check the comment section of his post to see some fascinating explorations of compression quines.

... just ignore the user Misi, who exclaims that you can create fodder bytes in your quine to manipulate the CRC value to be whatever you want---making my ridiculous brute-forcing methods a rather useless affair. At least having a self referential CRC is kind-of cool, right?

***

Anyway, feel free to contact me wherever about any questions you have, or something cool you'd like to share. I hope this tutorial has helped you out but it's probably a bit strange, so if you've got any concept you'd explored more/differently then definitely tell me. This is a new Jekyll blog I've set up, and I have haphazardly written the scripting/styling/ASCII charts, so do point out any errors or accessibility issue if you could.

I really can't thank the following folk enough for the amazing educational resources they've provided free of charge!
* [Russ Cox](https://swtch.com/~rsc/)
* [David Madore](http://www.madore.org/~david/)
* [Will Greenberg](https://github.com/wgreenberg)
* [Richard Mitton](http://www.codersnotes.com/)
* [Prof. David Brailsford](http://www.cs.nott.ac.uk/~psadb1/) (+ all of [Computerphile](https://www.youtube.com/channel/UC9-y-6csu5WGm29I7JiwpnA))
* [Calmarius](http://calmarius.net)
* [Prof. Norman Matloff](https://faculty.engineering.ucdavis.edu/matloff/)
* [Mark Adler](https://madler.net/madler/)

Thanks for reading!

{% include widgets/asciianim/script.html %}
{% include widgets/collapse/script.html %}

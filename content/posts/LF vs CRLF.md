+++
weight = 1
title = 'LF vs CRLF: git gud'
date = 2023-12-20T16:36:28+05:30
draft = false
description = "One fine day I wanted to start writing blogs. But I was faced with an issue. This blog is about how I resolved the issue while at the same time learning a bit more about git and discovering how typewriters worked!!!"
tags = ['git', 'LF', 'CRLF', 'blog', 'typewriter']
image = "/images/lf_crlf/git_gud.png"
+++

During December I took 15 days leave from office. I had a lot of free and as usual, I started playing ["Dota 2"](https://www.dota2.com/home).
Bored a bit from it (can't leave it 😥), I thought I would start writing blogs since I wanted to learn how it's done.

Like everyone I searched on Google and [reddit](https://www.reddit.com/) for ways to create
static websites and hosting. First I stumbled on [Material for MkDocs](https://squidfunk.github.io/mkdocs-material/) which I liked a lot but it was more of a documentation generator
than a blog site generator(It can be used for blog also with the new blog plugin). But I have
been hearing a lot about [Hugo](https://gohugo.io/) so I wanted to give it a try.

So like everyone I installed [Hugo](https://gohugo.io/) and spent 10 hours selecting a theme
rather like some content (we all have been there.. jk). I liked [hugo-ficurinia](https://themes.gohugo.io/themes/hugo-ficurinia/) theme a lot and hence I followed the instructions
to set it up.

### It's always the CSS

So I created one dummy post to see how it's getting rendered in the browser. I added dates
and tags in the post. But what I saw was completely different.

![Issue](/images/lf_crlf/issue.png)

As you can see the date and tags icons are not rendered properly for some reason. Being a
noob at frontend (yeah I do backend mostly), I went to Google and Hugo Forums to find the
answer to my problem.

I found out that the easiest way to see the problems is to use the console in the developer
tools and therefore I did the same. This is what I got.

![Console](/images/lf_crlf/console.png)

Hmmmm... something about the integrity of a resource which are the nerd fonts in my case. This was a correct indication because the icons to be rendered belonged to this font family.

So like every developer I googled about "this integrity" thingy in css.Here is a good resource
on this [Subresource Integrity](https://developer.mozilla.org/en-US/docs/Web/Security/Subresource_Integrity)

_TLDR: If you are lazy_

1. The integrity is the hash of the resource.
2. So the tag contains the link to the resource let's say CSS file and a value denoting it's
integrity which is basically a [SHA-512](https://en.wikipedia.org/wiki/SHA-2) hash of the
resource (file contents).

![Integrity](/images/lf_crlf/integrity.png)

3. The hash of the resource is calculated and matched with the one in the tag.
4. If they don't match then the resource is invalid and it won't be loaded which is what is being
said in the console.
5. This kinda makes sure that if someone changed the source code or something else then it's
not loaded since the hash won't match.

From this, I understood that the two resource files were somehow modified from that present in
the [repo](https://gitlab.com/gabmus/hugo-ficurinia)

### WorkAround

Well, one workaround for this was to remove the value of integrity in the tags.

![Work Around](/images/lf_crlf/work_around.png)

This rendered the icons correctly.

![Correct Render](/images/lf_crlf/correct_render.png)

But now I got more interested in what changed in the file so that the hashes were computed differently and hence my investigation began 😎.

<!-- ### It's all about hashes(bytes) -->
### The hideous culprit

Just so that we are on the same page to add the theme all I did was to git add a submodule to the repo.
We don't need to edit anything in the theme which means the content should be the same. But strangely my [VScode](https://code.visualstudio.com/) (yes I use vscode xdd sorry Neovim folks) showed it has been modified but I did not see any git gutter lines for the thing that was changed.

Then I saw one small property at the bottom of the VSCODE.

![CRLF](/images/lf_crlf/crlf.png)

And that's it. I changed this from CRLF -> LF and boom! it worked.

Even though I have been professionally working for 2 years, I never really cared much about the CRLF/LF issue. Maybe [Intellij idea](https://www.jetbrains.com/idea/) I used to handle this idk. Time has come to understand what they mean, how they came, and what we need to do.

### Time for some history: Typewriters

Earlier before PCs were a thing people used to use typewriters for writing. If you haven't
seen a typewriter before check out this YouTube video.

[![Type Writer](http://img.youtube.com/vi/oxN1C2QQUIE/0.jpg)](http://www.youtube.com/watch?v=oxN1C2QQUIE "Speed Typing Test")

Now when you reach the end of the line and you want to type in a new line you have to do two things.

1. Move the white sheet of paper a bit upwards. This action was given a unicode character named "LF" which stands for line feed. (\n)
2. Bring the cursor to the start of the line. This action was given a unicode character named "CR" which stands for carriage return. (\r)

With modernization came teletypewriters where both of these actions can be done in one step.

Teletypes set the standard for CRLF line endings in some of the earliest operating systems, like the popular MS-DOS. This meant MS-DOS used 2 characters to denote line endings whereas
Unix used LF to denote line endings and ditched the CR alltogether.

you can check this good post to know more about it: [crlf-vs-lf-normalizing-line-endings-in-git](https://www.aleksandrhovhannisyan.com/blog/crlf-vs-lf-normalizing-line-endings-in-git/)

One can say huh!! _It's Windows again_ but in another way, windows maintained the backward compatibility for which it's known.


### Back to the Present...It's all about hashes(bytes)

So as we know LF and CR are 2 Unicode characters. This means when calculating the hash on Linux
we consider only LF whereas when we do it on Windows we take both LF and CR
into consideration and that's where the culprit is different hashes.

In fact when I used git to add the submodule it did warn me saying ___"warning: LF will be replaced by CRLF"___ but hey when did the last time you took a warning seriously?


### Solution

Now that we know the problem, the solution is pretty obvious just use LF for all line endings
in the files.

This can be done globally using the following command.

```bash
git config --global core.autocrlf false
```
But wait a minute you ask, do I need to do it on all my colleague's machines???!!!

No worry my friend this is where [".gitattributes"](https://git-scm.com/docs/gitattributes) comes into the picture.

Add this "* text=auto" to the ".gitattributes" file and place it in the root of the
repository and let the git take care of it.

### Epilogue

This is how my initial task of just setting up a blog theme resulted in learning how
typewriters work. Well, what a start!!!

---
Issue link: https://gitlab.com/gabmus/hugo-ficurinia/-/issues/11

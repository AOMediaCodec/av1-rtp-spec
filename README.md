# RTP Payload Format for the AV1 Video Codec

This project is for authoring and generating the AV1 RTP Payload Format
specification document, in HTML and PDF.

The document is sourced in a light-duty markup format called Markdown. Markdown
is a readable plain text format that transforms to HTML. See **[GitHub Flavored
Markdown][GFM]**.

**HTML output** is

  * generated automatically by GitHub Pages,
  * always reflects the current `HEAD`, and
  * is automatically served here: https://aomediacodec.github.io/av1-rtp-spec/

**PDF output** must be generated locally, using the commercial HTML-to-PDF
transformer **[Prince]** (formerly PrinceXML). The [non-commercial version] is
fully-functional but leaves a small watermark on the first page.

Note that print-like output (PDF) requires styling not germane to HTML
presentation, so this project generates a specially-annotated HTML version for
use as PDF input. Nevertheless, both HTML and PDF use the same content file
(`av1-rtp-spec.md`).


## GitHub workflow

GitHub workflow primers abound on the web. Following is a summary.

To collaborate on a GitHub project it's necessary to have a GitHub account and
to have git installed locally. Then:

  1. **Fork** the project you want to work on to your own GitHub account, using
     the Fork button on the project's home page (top-right). You now have a
     remote copy.

  2. **Clone** the fork down to your local system and enter the directory that's
     created:

     ~~~~~
     git clone git@github.com:AOMediaCodec/av1-rtp-spec.git
     cd av1-rtp-spec
     ~~~~~

     By default, your clone will point to the "master" branch.

  3. Set up your **remotes**. This is done only once.

     Your locally-cloned repository will know of one remote: your fork, on
     GitHub, from which it was cloned. This remote will be named "upstream"
     by default, but we're going to rename it.

     We want your fork to be called "origin" and the authoritative, parent
     repository to be called "upstream." In a text editor, edit the `[remote]`
     stanzas in the file `.git/config` to look like this example:

     ~~~~~
     [remote "upstream"]
         url = git@github.com:AOMediaCodec/av1-rtp-spec.git
         fetch = +refs/heads/*:refs/remotes/upstream/*
     [remote "origin"]
         url = git@github.com:[GitHub username]/av1-rtp-spec.git
         fetch = +refs/heads/*:refs/remotes/origin/*
     ~~~~~

     "origin" now points to your fork, and "upstream" points to its source.

  4. Most work will be done in `av1-rtp-spec.md`. Before starting, make a
     **branch** in your cloned repo, to contain your changes. Choose a
     meaningful name.

     ~~~~~
     git checkout -b my-new-feature
     ~~~~~

     This both creates the branch (`-b`) and checks-out the new branch in your
     local clone.

  5. Edit as desired, using your chosen text editor. Save your changes.

  6. When ready, **add** your changes to the branch:

     ~~~~~
     git add av1-rtp-spec.md
     ~~~~~

  7. **Commit** your changes locally:

     ~~~~~
     git commit -m "Added section 7, Completely New Stuff"
     ~~~~~

     Alternatively (and preferably), do `git commit` only, which will spawn your
     git-default text editor for composing a multi-line commit message.

     You will also have a chance to annotate your commit later, on the GitHub
     website.

  8. **Push** your branch to "origin" (your remote fork):

     ~~~~~
     git push origin my-new-feature
     ~~~~~

  9. Now visit the project's page on GitHub.

     https://github.com/AOMediaCodec/av1-rtp-spec

     Github sees that you've pushed a new branch to your fork, and offers to
     create a **Pull request**. Follow the steps given. Annotate your pull
     request with any details that an approver might need, and submit it.

     The project maintainers will be notified of your pull request and will
     either merge it directly into "upstream," ask for changes, or ignore/reject
     it. The pull request, then, is both a patch and a comment thread.

 10. Once your pull request ("PR") is merged, you'll want to sync your local
     clone (and your remote fork) with the adopted changes:

     ~~~~~
     git fetch upstream
     git merge upstream/master
     git push origin
     ~~~~~

     Now everything is synchronized. We no longer need your working branch, so
     it's best to delete it:

     ~~~~~
     git checkout master  ## leave your branch, return to "master"
     git branch -d my-new-feature
     ~~~~~


## Generating locally

**The file `av1-rtp-spec.md` is quite readable, and you can probably work in it
without regenerating the HTML output.**

Generating and previewing your changes locally before submitting them requires
a local pipeline that mirrors the build done automatically by GitHub Pages.
Those steps are detailed in the `av1-spec` GitHub project's README:

https://github.com/AOMediaCodec/av1-spec#readme

Ignore the steps related to Node, npm and Grunt. These are not needed because
the `av1-rtp-spec` document is far simpler than the AV1 bitstream spec.


## Using Prince locally for PDF generation

Building the document locally writess output to an untracked/ignored directory
`_site`. The PDF input file is `_site/pdf.html`.

~~~~~
prince _site/pdf.html -o /tmp/av1-rtp-spec.pdf
~~~~~



[GFM]: https://github.github.com/gfm/
[Prince]: https://www.princexml.com/
[non-commercial version]: https://www.princexml.com/download/

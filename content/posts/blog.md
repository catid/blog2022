+++
title = "How the Blog Sausage is Made"
date = "2022-05-09"
cover = "posts/blog/cover.jpg"
description = "Time for a new blog site!"
+++

## Out with the Old

When I was in college I wrote my own PHP blog site and posted a lot of short snippets of useful C++ code and random ideas related to erasure codes: http://catid.mechafetus.com/  Unfortunately this site is now broken and I don't think the content is worth saving.

Around the end of 2020 I decided to rebuild the blog from scratch using modern tools.  I used AWS Amplify to build and serve my blog from github based on a Gatsby theme recommended by a friend.  This worked well for a time, but eventually the theme scripts stopped functioning as Gatsby and npm packages were improved in backwards-incompatible ways.

I was also getting frustrated with the complexity of AWS Amplify, which is the definition of over-engineered for managing a blog site.  Amazon Web Services hides a lot of random credit card charges in places that are hard to find, and it's hard to track them all down and stop the bleeding.  I never figured out how to have the S3 bucket served on Cloudflare CDN so that my static content could be reached instantly anywhere in the world, which is one of the advantages of static sites.

## In with the New

The new hotness is Hugo + Cloudflare Pages!

My content is hosted on github at https://github.com/catid/blog2022/

The site itself is based on this Hugo theme: https://github.com/panr/hugo-theme-terminal

Cloudflare Pages manages my DNS hostname, SSL certificates, CDN hosting, and the worker that rebuilds the site whenever I push to github.

Since Hugo is significantly faster than Gatsby, and Pages is significantly faster than Amplify: It takes about 30 seconds for Github changes to make it to the website.

And finally, the content is mirrored world-wide and not just locally in the US.

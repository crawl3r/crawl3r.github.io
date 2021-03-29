---
layout: post
title: "Bash Seems So Under Rated"





---

When I think of programming, I think of languages like C, C++, C#, etc. When I think of writing a CLI tool, I think of Golang and Python. Like most will agree, pulling a tool together in Go/Python is a pretty quick task. It's relatively easy to get a PoC ready and running with some pretty horrible output and features. 

What we (definitely myself) always appreciate is how quick we can take that awful output, post-process it using Bash and then fire it into another tool or just dump it to a file. We don't always have to implement file I/O into Python, we don't need to scrub strings and run through millions of lines with expensive for loops, string comparisons and counters to make sure we are editing what we want to edit.

Recently - a tonne of Golang tools have been making their way to the front of queue, especially with Bug Bounty fame being pretty high at the moment. For some reason, everyone one and their dog is looking for bugs. Everyone on LinkedIn is an 'Independent Security Research @ HackerOne', which is great for tool makers. But I guess the main point I'm trying to make is... if 10,000 people are running FFUF with a default wordlist like rockyou, then 10,000 people are just finding the same thing over and over again - which you know, is probably nothing. 

A quick disclaimer from me, I'm quite salty when it comes to Bug Bounties. I have found many (what I would call) bugs, which I would 100% raise in a pentest (some with PoC's) but they are often knocked to informational or just thrown out the pile. This is fine, I don't have an issue with this, but I just like to moan and the internet gives me a voice - right? Cool.

So back to the point about FFUF and 10,000 rockyou-ers. The issue here is, you aren't finding anything. What you are finding though, is assets. The more assets you find, the biggest your attack surface. The bigger your attack surface, the more chance of bugs. Right? Probably. Statistically the number/ratio should be bigger, so in my head that's a win.

So, why not take your FFUF output (probably JSON), jq the assets it found into a tool that can 

​	A) try go deeper (something like Hakrawler, to gather more assets per )

​	B) take screenshots (like Aquatone) and give you a visual representation of all you have seen

​	C) just store them into a file and go hunting manually

## Well that wasn't very impactful

Up until now I honestly haven't really spoke about much bash... other than piping FFUF into jq and then into something else. But is honestly all you need to start automating an entire pipeline using Bash to hook it all together.

I have personally built some pretty funky pipelines that just use Bash to call lots of other tools (thanks community) to give me a final output which can either be used for additional manual searching or possibly even highlight and generate a report for a found issue (this is the best feeling).

I think my point is, don't just use Python to use openssl, loop the output and search for a subdomain, check if they're alive and then crawl each one. Use bash and make your life easier!

```bash
openssl s_client -connect URL:443  | openssl x509 -noout -text | grep "DNS:" | sed 's/,/\n/g' | awk '{split($0,c,":"); print c[2]}' | sort -u | httprobe -c 100 | hakrawler -nocolor | sort -ufd > output.txt
```

The above is an example, and probably won't reveal many results, to be honest, it might not even be a fully working example it just exists to show how Bash can be used to easily string tools together. Naturally, if you were building a pipeline you would flesh it out more, probably store data along the way for future reference or for 'checkpoints' if you're tooling crashed or found some weird errors.

Now I have reached the end, I'm not sure what the value in this post was... but go learn Bash. That's the value! \o/

If you need more proof, Google something like 'bug bounty scanners GitHub' or just go read the Axiom code. It's a worthy language to learn!


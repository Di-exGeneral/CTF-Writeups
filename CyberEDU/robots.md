# robots

**Category:** Web \
**Difficulty:** Easy \
**Status:** Solved \
**Flag:** `CTF{Kr4ftw3rk_4nd_th3_r0b0ts}` \
**Tools:** curl, gobuster 

## Summary
 
A minimal PHP/Apache site where `robots.txt` disallowed a page it was ostensibly trying to keep out of search engines — and in doing so, advertised its exact path to anyone who looked.

## Recon

```sh
gobuster dir -u http://35.198.115.187:32540 -w /usr/share/wordlists/dirb/common.txt -x txt,py,json    
```

Found `index.php` and `robots.txt` live. Headers confirmed Apache/2.4.54 (Debian), PHP/7.4.32.

## Exploitation

```sh
curl http://35.198.115.187:32540/robots.txt
```

```
User-agent: *
Disallow: /g00d_old_mus1c.php
```

Requested the disallowed path directly:

```sh
curl http://target:port/g00d_old_mus1c.php
```

The page rendered normally and included the flag directly in the HTML body, no authentication or further exploitation needed.

## Lessons
 
- `robots.txt` is a courtesy convention for well-behaved crawlers, not an access control mechanism. It's plaintext, unauthenticated, and publicly fetchable by definition.
- Always check `robots.txt`, `sitemap.xml`, and `.well-known/` early in recon, site owners sometimes use `Disallow` to "hide" pages, which paradoxically makes them easier to find than if the entry didn't exist at all.

# Document Configuration in Early HTML Comments

## Authors:

- Ian Clelland
- Mike West

## Participate
- Issue Tracking: [GitHub](https://github.com/clelland/doc-comments/issues)

## Table of Contents

[You can generate a Table of Contents for markdown documents using a tool like [doctoc](https://github.com/thlorenz/doctoc).]

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Abstract

HTTP headers are the default choice for HTML document configuration metadata,
despite having a number of significant drawbacks. Allowing configuration in
over-the-wire document markup, or in the DOM, is often proposed as a solution,
and while very accessible to developers, has its own drawbacks.

The idea presented here is to reserve a space in the over-the-wire document,
inside of an HTML comment, specifically for header-style metadata.

## Introduction

HTTP headers were originally used to control the transport of content from an
origin server to a web browser, including caching at intermediaries, and to
perform content negotiation, including just enough information about the
resource to support that.

Being separate from the resource usually meant that the contents of the headers
were privileged — document authors who could upload files to the web server
couldn't necessarily set the headers with which those files were served. This
position of privilege has made the HTTP headers an attractive place to insert
more information about the document, including configuration information which
has no bearing on either transport or content negotiation.

By being made headers, this information gains the properties of being delivered
before the content, being unmodifiable by that content, but at the cost of not
being settable by the people who are most likely to want to set them.

So what if there were a place in the document where web developers could add
these document-configuration headers, where most of the important properties of
headers still applied, but didn't require the direct intervention of a server
administrator to change?

Inspired by Python's docstrings, where a syntactically-valid but effect-free
string in the right position can provide meaningful metadata to tools that
understand it, note that the HTML parser allows comments before the initial
`<!doctype>` or `<html>` tags, which as comments, are discarded by existing
rendering engines.

A comment before the opening of the document could contain additional
(configuration-only) headers to be processed with the document by the receiving
browser. It is a slightly-privileged position, accessible to those authors who
have control over the overall document structure, (as opposed to authors who
can only fill in template contents,) but does not necessarily require server
administration privileges to insert.

## Examples!

A developer who wanted to set a permissions policy for a page could include a
comment at the top of the document, like this:

```
<!--page-config
Permissions-Policy: geolocation=(), camera=https://photobooth.example.com
-->

<!DOCTYPE html>
<html>
…
</html>
```

Another site might want to configure CSP with reporting:

```
<!--page-config
Content-Security-Policy: script-src 'self'; report-to csp-endpoint
Reporting-Endpoints: csp-endpoint=https://analytics.example.com
-->

<!DOCTYPE html>
<html>
…
</html>
```

## Motivating Use Cases

There are dozens of situations that come up in practice which motivate this
proposal, but they generally boil down to some variant on this: A web developer
who has control over the contents of their documents, but not over the headers,
wants to set a document configuration header on some of their pages.

Reasons that a developer might legitimately not have control over headers might
include:

* A student's home page is hosted on a shared university Apache server which
    accepts file uploads, but they do not have authority to change the server
    configuration.
* A demo site is hosted on a github.io subdomain, where the resource content can
    be served from the repository, but the headers are pre-determined.
* A developer reports a browser bug with a reproduction case on jsfiddle.net.
* A large site has their content cached at a CDN.
* A developer wants to serve generated content in a srcdoc iframe, but wants it
    to have different configuration from its parent document.

Configuration headers will vary in importance from developer to developer, but
might include things like:

* Client Hints
* CORS
* Document-scoped Policies:
* COOP / COEP
* CSP
* Document Policy
* Permissions Policy
* Redirect URL
* Origin Trial Tokens
* Reporting Configuration

## Non-goals

* Allowing document markup to affect HTTP transport, caching or
    content-negotiation.
* Enabling configuration to come from arbitrary points in the page markup.
* Enabling configuration to be added or changed dynamically through DOM methods.

## API: The page-config comment

The page-config comment is a specially-formatted HTML comment, present before
any other tags or markup, which can contain additional (configuration-only)
headers to be processed with the document by the user agent.

Note that this is a slightly-privileged position, accessible to those authors
who have control over the overall document structure, but not necessarily to
all content authors. A template system, for example, where authored content can
only fill designated slots in the document, could protect this area from
arbitrary modification.

The comment must be the very first bytes of the content, and must begin with
the exact ASCII string

```
"<!--page-config"
```

followed by a CRLF.

The comment can then contain zero or more permitted header lines, each
terminated with a CRLF, and ends when the final line

```
"-->" CRLF CRLF
```

is encountered.

As a simple example document:

```
<!--page-config
Permissions-Policy: geolocation=(), camera=https://photobooth.example.com
-->

<!DOCTYPE html>
<html>
...
</html>
```

When received over the wire, assuming HTTP/1.1, this would look like:


```
HTTP/1.1 OK
Content-Type: text/html; charset=utf-8
Content-Encoding: ...
Cache-Control: ...
Expires: ...
Date: ...
ETag: ...
Vary: ...

<!--page-config
Permissions-Policy: geolocation=(), camera=https://photobooth.example.com
-->

<!DOCTYPE html>
<html>
...
</html>
```

## Detailed design discussion

The overarching goal of the design is simplicity. Header parsing is already
very well tested in browsers, and we would like to avoid introducing new
opportunities for vulnerabilities. Headers are often parsed in a privileged
process, and so new bugs there can have more serious consequences than bugs in
the HTML parser.

To this end, the processing model is kept as simple as possible, while still
being usable by developers who may not have perfect control over the bytes that
encode their documents:

* The comment must occur at the very beginning of the document, so no preload
    scanning is required, unlike with <meta> tags which can occur arbitrarily
    deeply.
* The syntax is strict enough that the HTML parser should not need to be invoked
    in order to parse the comment.
* The encoding rules are loose enough that content authors should not encounter
    issues when writing files under most platforms' text editing conventions.

### Syntax

The page-config comment must comprise the first bytes of the document. This
means that any text appearing in a document, whether it is a different comment,
doctype, or other markup, or even an extra newline, will cause the page-config
comment to be ignored.

The document must begin with the 15 characters `"<!--page-config"`, followed by
a newline. To allow for different line ending conventions preferred by
different platforms, each newline may be either an LF or a CRLF pair.

The page-config comment is terminated by a line beginning with the three
characters `"-->"`, followed by two newlines.

Between the delimiters, zero or more header lines may be present, each of which
is terminated by a newline.

An HTML comment which does not conform to this syntax will not be interpreted
as a page-config comment.

### Scope

#### What headers does this apply to?

The page-config comment should not be considered a general-purpose header
delivery mechanism. Many HTTP headers would not be appropriate in that space,
either because their configuration should represent some authority over the web
server, or because they need to be read or written by intermediaries, such as
proxies.

Examples of headers which should be allowed in page-config:
* `Accept-CH`
* `Access-Control-* (CORS)`
* `Cross-Origin-Embedder-Policy`
* `Cross-Origin-Opener-Policy`
* `Content-Security-Policy`
* `Document-Policy`
* `Location`
* `Origin-Trial`
* `Permissions-Policy`
* `Reporting-Endpoints`

Examples of headers which should *not* be allowed in page-config:
* `Age`
* `Allow`
* `Cache-Control`
* `Connection`
* `Content-Encoding`
* `Content-Language`
* `Content-Length`
* `Content-Type`
* `Etag`
* `Expires`
* `Keep-Alive`
* `Last-Modified`
* `Trailers`
* `Transfer-Encoding`
* `Vary`
* `Via`

#### What resources does this apply to?

The page-config comment is only applicable to HTML documents; there may be
opportunities to define similar mechanisms to resources of other types
(scripts, images, etc.,) but this proposal does not attempt to do that.

### Semantics

#### What happens if there are conflicts?

If a header which appears in the HTTP response headers also appears in the
page-config comment, some additional processing may be required. This may need
to be performed on a header-by-header basis. As suggested guidelines:

If a header is not allowed to appear multiple times in a response, then the
version from the HTTP headers should be taken.

Headers like Content-Security-Policy already have rules for how they should be
combined when taken from multiple sources (headers and meta tags, for
instance). These rules should be used in this case as well.

It may be possible for some structured headers, defined as lists or
dictionaries, to be combined into a single header value for processing.

#### What about the Vary header?

The Vary header is used for cache management, and the other headers which it
mentions need to be readable by intermediary caches, which are not expected to
understand the page-config comment. It probably shouldn't be valid to have a
header present in the page-config comment if that header is mentioned by the
Vary header.

### Opt-out

#### How can sites opt out of this processing if they don't want it applied?

Given the proposed processing model, some servers could be configured to emit
an empty page-config comment at the start of all responses, or even a single
character which would prevent the comment from being processed.

Alternatively, if that is not predictable enough (or possible given someone's
serving infrastructure), then a header like

```
Document-Policy: page-config=?0
```

could be used to prevent this mechanism from operating.

### Proposed processing model

An algorithm like the following could be used by Fetch on a response body,
before the document is parsed. (Note that "NL" may be either LF or CRLF.)

1. Is the declared content type set to `"text/html"`? If not, abort.
1. Is the declared character set `"utf-8"`? If not, abort.
1. Are the first 15 characters of the body `"<!--page-config"`? If not, abort.
1. Are those followed by NL? If not, abort.
1. Scan until we find one of the following, and take the first applicable
    action:
    1. NL `"-->"` NL NL: Stop scanning and continue the algorithm.
    1. NL NL: abort processing.
    1. `"-->"`: abort processing.
    1. End-of-response: abort processing.
1. Grab the range between those sequences and parse the resulting string as
    headers.
1. For each header in the resulting list, do some logic to determine whether or
    not to append the header to the response.

## Considered alternatives

### Let's just force everything to be in headers

As mentioned earlier, this is the problem we're trying to solve here, but an
alternative considered is just *not* to try to solve it. Headers have several
problems, though, when used for application-level document configuration. The
biggest practical problem is that headers are usually controlled by the web
server configuration, and content creators who want to change them need to
convince server administrators to make those changes on their behalf. Some
developers may have no way to even communicate with the right people.

### Why not allow these headers to be set using `<meta>` tags?

This is the status quo workaround. Some header-type data can be set in markup,
in a `<meta http-equiv>` tag. The current processing model has several issues:

* Meta tags can appear anywhere in the document, even very late into content.
    Rather than just during response delivery or document parsing, these could
    appear at any point during the document's lifetime.
* Meta tags can be injected by scripts running on the page, making this subject
    to injection attacks.
* This requires using the HTML parser to decode and extract the actual tag
    value.
* All of this means that configuration set in meta tags has to be restricted to
    just those values which could be set quite late, and often requires them to
    have different syntax or semantics from the same values set in headers.

### What if we allow configuration in meta tags, but be really strict about where they occur?

This tactic was used by Origin Trials in its initial incarnation, to avoid the
timing problems with unrestricted `<meta>` tags. They required that the tag be
in the `<head>` of the document, and precede any `<script>` or `<link>` tags.

* This is still only useful for a small class of headers
* The processing model is different from `<meta>` tags generally.
* Requires reading arbitrarily far into the document content to find the
    relevant tags.
* *Probably* means that the tags can only appear during the initial parse, and
    therefore have to be present in the HTTP response body, but this isn't
    proven.

### What about attaching attributes to early tags?

Suggestions have been made to make policy headers an attribute of the `<html>`
tag, or even the `<!DOCTYPE>` element. This would solve one of the problems that
`<meta>` tags have: that of having to read deep into the document to find the
relevant configuration, but there's still a lot of complexity in parsing out
the header data.

* Still embeds configuration in markup in a way that they cannot be easily
    separated. It likely requires interaction with the HTML parser to correctly
    extract attribute values from tags.

## Stakeholder Feedback / Opposition

Help fill this out! If you have feedback or opinions on this, I'd love to hear
from you.

## References & acknowledgements

This idea was certainly inspired by Python's docstring comments, JavaDoc
comments and vim's modeline feature, as well as earlier similar ideas by Ojan
Vafai.

Many thanks for valuable advice and feedback on this specific proposal from:

* Paul Irish
* Jeff Kaufman
* Jason Chase
* Jason Miller
* Jeremy Roman

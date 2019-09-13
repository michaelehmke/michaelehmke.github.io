---
title: "Case Study: SMTP Header Injection - Part 2"
date: 2019-08-19
tags: [bug bounty, web application]
categories: [hacking]
author_profile: false
---


This post continues my [previous](https://michaelehmke.github.io/hacking/smtp-header-injection-case-study-1/) Case Study article on SMTP Header Injection. 

**consider more intro**

The focus this time is to remove the dynamically generated message that is inserted before the start of the user-provided message for every email sent through the “Contact Us” page. The message can be seen below as the first line in the email's body:

**pic 1st pic**

Again, the motivation for removing this message as an attacker is **TODO;**

I can't go through them all here but many attempts to get rid of that auto-generated line were made, using a variety of different techniques. The ideas ranged from using CSS selectors to make the existing body invisible, using C-style escape sequences to "backspace" the entire line or insert enough newlines until the message couldn't be seen anymore, submitting a malformed header to cause Outlook to attach the entire body as a whole to the email itself, HTML injection into the First Name field to comment out the latter half of the message, and dozens of attempts with other headers and injection attempts into other fields besides the Subject; all this while abiding by a 30 character limit.

For reference, the JSON to submit an email is the following:

```json
{
    "firstName": "michael",
    "lastName": "ehmke",
    "email": "michaelehmke@example.com",
    "phone": "1112223333",
    "subject": "this is my subject",
    "message": "this is my message"
}
```

Eventually I started messing with the `Content-Type` header field. This field tells the email client what kind of data to expect and how to interpret it. If we view the header of the email that gets sent out, it shows that its Content-Type is `Multipart/Mixed`. Essentially, this type allows boundaries to be defined throughout the body of the email, and within each boundary a different Content-Type could be potentially specified. Content outside of a boundary is ignored and not shown by the email client. A more thorough explanation of how that header field works and how it may be utilized in a SMTP Header Injection attack is given in an [introductory article](https://michaelehmke.github.io/hacking/smtp-header-injection/) of mine.

A basic example of how an email that has it's `Content-type` set to `mutlipart/mixed` is formatted:

```
From: User1@website.com
To: User2@website.com
Subject: Check this out
Content-type: multipart/mixed; boundary=myboundary

this is not displayed
--myboundary

by default, this is plaintext

--myboundary
Content-type: text/plain

this has been specified to be plaintext

--myboundary--
this is also not displayed
```
	
The idea would be to inject a new Content-Type header of `Multipart/Mixed` into the Subject field, define a boundary name, instantiate the opening boundary, and set the Content-Type for the boundary to `text/html` so that everything following it (including the pre-existing Content-Type header and boundary definitions) could be under my control. Then I would need to include the actual message I want to send in the body, close the boundary and announce the end of the email so that everything after my closing boundary (including the pre-existing headers, as well as the auto-generated message) would be ignored. One problem though - the 30 character limit that this all needs to fit within. At a minimum, the following would need to be injected:

```sh
\nContent-Type:multipart/mixed;boundary=A\n\r\n--A\nContent-Type:text/html\n\n<html><body><My_Message></body></html>\n--A--
```
Components of the payload above:
1. Inject and define the Content-Type before the original is defined (this is important because the original Content-Type field defines boundaries with randomized names, so I can't try to guess and use those myself)
2. Define my own boundary name of `A`
3. Instantiate opening boundary with `--A`
4. Define Content-Type of the section to be `text/html`
5. Open an `<html>` and `<body>` tag
6. Include the message for the body of the email
7. Close with `</body>` and `</html>` tags
8. End email message with `--A--`

That all happens to be slightly longer than 30 characters. I thought that I might be able to fit the most necessary 30 characters in the Subject header, and with that, have taken over enough control of the body that I can concatenate the rest in the message field of the email in a way that makes sense to the parser. 

In an attempt to do this, I would, at an absolutely bare minimum, need to inject `\nContent-Type:multipart/mixed;` (coincidentally exactly 30 chars) and hopefully be able to utilize the message field for the rest.

This theory quickly got burned however when I took a closer look at a plaintext dump of the email. The dump can be evoked by injecting into the body from the header field with a double line feed `\n\n`. When looking at the now-plaintext headers and body, it is made clear that any data inserted into the message field would be located far too late in the body since it occurs much later after the pre-existing Content-Type header and boundaries have been set up.

**pic of plaintext dump**

The areas highlighted green are where the original Content-Type field and boundary are instantiated. I would need to define and open my own boundary and close it before the original is even reached, otherwise they will appear in the email message even if they won’t necessarily be parsed as control fields. The text highlighted yellow is where any data would be placed for use with the idea of using that as the latter part of a concatenation, which is clearly not situated in a helpful location.

Since the original email already had the Content-Type set to `multipart/mixed`, I tried to instead define and set a boundary since I had space for those characters, but that didn't work, probably due to syntactical reasons rather than strictly the ordering of headers. I started to notice that in the dump of the email, the body maintains the same C-style escape sequences that the header fields use and it just lists the metadata information as well as the original HTML markup in plaintext, instead of rendering the data as HTML. 

This led me to start thinking that if I overrode the Content-Type to `text/html` instead, then all of that metadata could be converted to HTML instead of being parsed as and resuming control as header fields. This would essentially get rid of the boundaries that were present and then maybe it could all be wrapped up in HTML and CSS to make it all disappear.

There were enough characters to inject the Content-Type:text/html payload but when sent and parsed by the email client, nothing in the message changed:

```json
{
    "firstName": "TheCEO",
    "lastName": "YourBoss",
    "email": "the_ceo@haxxme.com",
    "phone": "1112223333",
    "subject": "Z\nContent-type:text/html;\n",
    "message": "this is my message"
}
```
**pic of Z attempt**

I needed to break out of the scope of the headers and transfer control to the body, pushing the original Content-Type header and boundaries into the body as well, to allow my new header to enforce itself on the body’s contents:

```json
{
    "firstName": "TheCEO",
    "lastName": "YourBoss",
    "email": "the_ceo@haxxme.com",
    "phone": "1112223333",
    "subject": "Z\nContent-type:text/html;\n\n",
    "message": "this is my message"
}
```

**pic of Z with headers in body plaintext**

It can be seen above that afterwards, the metadata still printed to the body, but now the formatting of the text broke down and the HTML tags are no longer being displayed. Viewing the source of the email further validates that everything has now been encapsulated by my new Content-Type header to be interpreted as HTML.

```html
<meta http-equiv="Content-Type" content="text/html; charset=iso-8859-1">
MIME-Version: 1.0
Content-Type: multipart/mixed; 
	boundary=&quot;----=_Part_2337_1756916918.1561753192452&quot;
X-Mailer: sendmsg

------=_Part_2337_1756916918.1561753192452
Content-Type: text/html; charset=utf-8
Content-Transfer-Encoding: 7bit

<html>
	<head>
		<style>
		body {
			font-family: Calibri,sans-serif;
		}
		</style>
	</head>
	<body>
		Message from the Contact Us page from TheCEO. The information requested is:
		<br><br>
		this is my message
	</body>
</html>
------=_Part_2337_1756916918.1561753192452--
```

Now the issue is to get rid of all the old metadata information that’s getting displayed in the email. All the metadata exists prior to the opening `<html>` tag so I can't select it with CSS to make it all disappear (by either changing the text color to white or modifying the attributes to “hidden”). I tried to append `<html>` at the end of my header injection so that it would be inserted before all the metadata but there were only 4 characters available in the Subject field where I could inject.

I noticed that the `<meta>` tag and its properties were not showing in the actual email body as plaintext. So I thought that if I opened a `<meta` tag, but never closed it with the right angle bracket, it might absorb everything after it as its own property data. There were only 4 characters left so I was only able to try with `<met` but I just went with it.

```json
{
    "firstName": "TheCEO",
    "lastName": "YourBoss",
    "email": "the_ceo@haxxme.com",
    "phone": "1112223333",
    "subject": "\nContent-type:text/html;\n\n<met",
    "message": "this is my message"
}
```

**pic of just going with it, partial success**

The name of the tag didn't end up mattering, and it did absorb everything after it until it hit a closing angle bracket `>` which happened to be the opening `<html>` tag. Now, the only text left showing is all the data after that closing bracket (rendered as HTML), including the greeting message that we're trying to get rid of. It would appear that we are almost back to square one, still with the auto-generated line visible in the message. What’s different now however is that we have some level of control in the *body* both before and after that line, where before, we only had control in the headers and in the body after the auto-generated message. 

The source shows how the metadata was swallowed up by the open `<met` tag:

```html
<meta http-equiv="Content-Type" content="text/html; charset=iso-8859-1"><met
MIME-Version: 1.0
Content-Type: multipart/mixed; 
	boundary=&quot;----=_Part_2337_1756916918.1561753192452&quot;
X-Mailer: sendmsg

------=_Part_2337_1756916918.1561753192452
Content-Type: text/html; charset=utf-8
Content-Transfer-Encoding: 7bit<html>
	<head>
		<style>
		body {
			font-family: Calibri,sans-serif;
		}
		</style>
	</head>
	<body>
		Message from the Contact Us page from TheCEO. The information requested is:
		<br><br>
		this is my message
	</body>
</html>
------=_Part_2337_1756916918.1561753192452--
```

Since I had now effectively turned everything into `text/html` through the Content-Type header, where before it was all `multipart/mixed` data types, I thought that it may not matter if everything is encapsulated within `<html>` and `<body>` tags in order for the client to be able to parse the data as HTML. I replaced the `<met` opening tag with `<p>` since there were enough characters left in the Subject field to squeeze that in for injection. Before, I had wanted to inject `<html>` in that location to encapsulate all of the message data but there was no room. I put the closing `</p>` in the actual message section of the email, thinking that if all the data from the beginning of the old, now unused, metadata headers to the start of my custom message were wrapped with the <p> tags, then I could select that whole block with CSS and make it all disappear. I tested this first by changing the color of `<p>` elements to red with a CSS block:

```json
{
    "firstName": "TheCEO",
    "lastName": "YourBoss",
    "email": "the_ceo@haxxme.com",
    "phone": "1112223333",
    "subject": "\nContent-type:text/html;\n\n<p>",
    "message": "</p><style>p {color:red;}</style><p>turn everything red</p>"
}
```

Not really what I was hoping for:

**pic of what i wasn't hoping for**

Everything turned red except for the specific message that I've been trying to select this whole time, meaning I still won't be able to select it using this method. But this did prove that the `<p>` element did not need to be inside of `<html>` and `<body>` elements to be rendered correctly, and that the CSS styles within the pre-existing `<body>` tags would break out of scope and affect the `<p>` elements even outside of the `<html>` block. 

Viewing the source doesn’t immediately show why the injected `<p>` element did not totally encapsulate everything up to the closing `</p>` in the message, but it’s probable that this is due to hitting the opening `<html>` element and thus throwing the `<p>` out of scope, preventing it from continuing any further. 

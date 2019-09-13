---
title: "Case Study: SMTP Header Injection - Part 1"
date: 2019-08-08
tags: [bug bounty, web application]
categories: [hacking]
author_profile: false
---



This post details one of the vulnerabilities I found in an undisclosed bug bounty a while back, and the steps/thought process taken to exploit it. The actual specific messages, URL's, email addresses, etc. will not be provided and will be replaced with example data. The assessment was done for a large company that we'll call HaxxMe.

<h2>Context</h2>

As the title shows, this will focus on a SMTP Header Injection vulnerability that was found in an "Contact Us" page, which is a fairly common feature of many web applications. If you're not familiar with this vulnerability, my previous post goes over some of the basics: [Primer: SMTP Header Injection](https://michaelehmke.github.io/smtp-header-injection/). While most articles found online about SMTP Header Injection target soley PHP applications and its vulnerable `mail()` function, this attack was performed against a Java back-end utilizing webMethods. I'll skim through the details of the basic attack and move on to the more interesting use cases, one of which will be described here and another in a later [post](https://michaelehmke.github.io/hacking/smtp-header-injection-case-study-2/). 

The page resembled the following with the fields `First Name`, `Last Name`, `Email`, `Phone`, `Subject`, and `Message`.


<img src="{{ site.url }}{{ site.baseurl }}/images/contact_us_2.png" alt="Screenshot of Contact Us page">


Through some testing it was found that the `Subject` field, and that field alone, was vulnerable to SMTP Header Injection. I was able to inject `Cc`, `Bcc`, etc. headers and could break out of the headers to inject content into the body. Client-side validations on the front-end prevented unexpected data from being submitted, but submitting the form with direct requests bypassed the validations so that a user may enter any information into the fields. 

Endpoint for direct requests:

```
   https://haxxme.com/api/contact
```

When the form gets submitted, it sends over the user input with the following JSON (to the hard-coded recipient address of `hr_inbox@haxxme.com`):

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

The following are examples of different payloads being injected into the `Subject` header field, and the resulting emails that get generated and sent out as a result. The recipient `hr_inbox@haxxme.com` is hard-coded into the application on the back-end and there is a hard-coded message that is included in the email as well: `Message from the Contact Us page from <firstname>. The information requested is:`.

<br/>
<b>Inject a new `Cc` field:</b>

```json
  {
    "firstName": "michael",
    "lastName": "ehmke",
    "email": "michaelehmke@example.com",
    "phone": "1112223333",
    "subject": "test\nCc:user1@example.com\nx",
    "message": "this is my message"
  }
```
Resulting Email:

<img src="{{ site.url }}{{ site.baseurl }}/images/email_injected_cc_ex.png" alt="Email with injected Cc field" class="left-align shadow">

<br/>
<b>Inject another recipient to be used in the `To` field:</b>

```json
  {
    "firstName": "michael",
    "lastName": "ehmke",
    "email": "michaelehmke@example.com",
    "phone": "1112223333",
    "subject": "Hi\nTo:user@haxxme.com\nx",
    "message": "this is my message"
  }
```
Resulting Email:

<img src="{{ site.url }}{{ site.baseurl }}/images/email_injected_to_ex.png" alt="Email with injected To field" class="left-align shadow">

<br/>
<b>Inject a message into the email's body:</b>

```json
  {
    "firstName": "michael",
    "lastName": "ehmke",
    "email": "michaelehmke@example.com",
    "phone": "1112223333",
    "subject": "test\n\nbreak through headers",
    "message": "this is my message"
  }
```
Resulting Email:

<img src="{{ site.url }}{{ site.baseurl }}/images/email_injected_body_ex.png" alt="Email with injected To field" class="left-align shadow">


<h2>Use Case: Impersonate and Exchange Emails as any Employee</h2>

There are **major limitations** with these attacks however. Due to some internal implementations perhaps at an intermediary gateway, SMTP relay, the email client, or other email services; the emails were not being received by the recipients injected into any of the header fields, though they were correctly placed where specifications require them to be.

Though I was able to show that SMTP Header Injection existed, I wanted to be able to provide a PoC with more meaningful implications. These limitations will be explored in the following use cases to show how this vulnerability may still be taken advantage of under these circumstances. It will be seen that the biggest limitation for the attacker is actually the 30 character limit that exists for the Subject, which is properly validated by an internal service before constructing and sending the email.

By design, the Contact Us form allows the user to provide any email address, which will effectively send the email to the `hr_inbox@haxxme.com` mailbox showing that the email is from the email address that was entered. However, though the email can be spoofed to show that it is from any email address, the user on the receiving end (HaxxMe employee) could simply reply to the email and it will get sent to the actual owner of the email and any concerns can be cleared up.

Because of this limitation, theoretically speaking, in order for the attacker to make it appear that an email is from a HaxxMe employee while still being able to receive any responses to the email and exchange messages back and forth, he would need to manually spoof an email himself, outside of the system. This would need to be done from an externally hosted mail server and it would very likely get caught by filters and edge-protection services at the company's proxy before it even reaches anyone’s inbox inside the company. When sending emails through the Contact Us page, they are getting sent by a trusted email service which is hosted on a server that is already internal to the company's network, so they do not get blocked by any firewalls or filters.

This is where SMTP Header Injection comes in. There is another header field `Reply-To` which allows an email to come from a sender, while any replies to the email will go to the recipient specified in the `Reply-To` field.

Providing the following as the `Subject` injects a recipient in the `Reply-To` header field:

```sh
  Hi\nReply-To:the_ceo@gmail.com\nx
```

An email may be generated with the following JSON:

```json
  {
    "firstName": "TheCEO",
    "lastName": "YourBoss",
    "email": "the_ceo@haxxme.com",
    "phone": "1112223333",
    "subject": "Hi\nReply-To:theceo@gmail.com\nx",
    "message": "Hi HR Team,<br><br>I thought this would be the quickest way to reach you. We’ve made a last-minute emergency hire who will need to start tomorrow morning and we need him to be set up on payroll and added to the functional roles: xxxxxxxxx. His information is xxxxxxxxx.<br><br>Thanks,<br><br><b>TheCEO</b><br>HaxxMe<br>1234 Haxxr Drive<br>MyCity, MyState 32432<br><a href=''>the_ceo@haxxme.com</a><br>123-123-1234 Ph<br>"
  }
  ```

The resulting email that is received in the `hr_inbox@haxxme.com` mailbox:

<img src="{{ site.url }}{{ site.baseurl }}/images/uc1_replyto_1.png" alt="Email with injected Reply-To header field" class="left-align shadow">

Just from viewing this screen, there is no way to tell that the email is spoofed. Since the `From` email address matches the actual CEO's address in the system, MS Outlook conveniently populates his picture as well as availability status, and if the email address in the UI is clicked on for further details, the legitimate details will automatically populate. A suspicious user from the `hr_inbox` mailbox may reply to this email anyways however, by clicking either of the Reply buttons at the top. That user would be presented with this screen and would proceed to write up a response:

<img src="{{ site.url }}{{ site.baseurl }}/images/uc1_victim_reply_1.png" alt="Victim responding to email with injected recipient" class="left-align shadow">

A user responding quickly may not notice that the email in the To: field is going to the attacker’s address `theceo@gmail.com` rather than the actual HaxxMe address.

In theory, the attacker could use an address that resembles the legitimate one more closely so it is harder to immediately tell them apart, such as `the-ceo@gmail.com`. However as I mentioned, the biggest limitation in this attack is that the Subject header is being limited on the back-end to 30 characters. And the current payload takes up all 30 of them (`\n` = 1 char):

```sh
  Hi\nReply-To:theceo@gmail.com\nx
```

This limits not only what can be used as the injected email address, but also what the actual Subject can be. This example only expends 2 characters for "Hi", though an empty subject is allowed. The trailing character after the last newline is necessary as well, seemingly due to client-specific implementations. Still though, the displayed email address is subtle enough that it may easily be overlooked.

  
At this point, the HR employee replied to TheCEO and the attacker received the response at `theceo@gmail.com`. This allows the attacker to not only receive the response to his spoofed email, but to also avoid any red flags that would’ve been raised if the response had went to TheCEO's actual address. Below is the response that the attacker received in his inbox:

<img src="{{ site.url }}{{ site.baseurl }}/images/uc1_victim_response_1.png" alt="Victim's response shows up in attackers inbox" class="left-align shadow">

The attacker can’t reply directly to this email however because then his response would be shown as coming from `theceo@gmail.com`, or it would be blocked altogether at the proxy level. 

Instead, the attacker can just repeat the whole process of injecting the same headers in the Contact Us email form. He can even ensure that the email shows up in the HR_inbox as a reply to the current thread by simply specifying the same Subject of "Hi". Outlook will conveniently take care of grouping them together.

Attacker’s *perceived* response to the thread after re-injecting headers:

<img src="{{ site.url }}{{ site.baseurl }}/images/uc1_perceived_response.png" alt="Attacker's response showing up as part of the email thread" class="left-align shadow">


It can be seen that Outlook automatically grouped the emails together in the same thread since the Subject was the same, making it appear to the HR employee that TheCEO did in fact receive and respond to the email for verification.

The contents of the attacker's email:

<img src="{{ site.url }}{{ site.baseurl }}/images/uc1_attacker_response_1.png" alt="Contents of attacker's response" class="left-align shadow">

This Use Case was just an example, but much more convincing phishing attempts could be easily made. Also, in every email spoofed above, the first line in the message was always auto-generated and that may not be desirable by the attacker. The next use case shows a non-trivial example of cleaning that up.


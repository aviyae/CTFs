# BSidesTLV2020 Writeup Bugler

* Category: Web
* Solved by JCTF Team
* 550 points

![description](https://github.com/aviyae/CTFs/blob/master/BSidesTLV2020/bugler/description.png)

Bugle, if you’re curious, is:

![bugle](https://github.com/aviyae/CTFs/blob/master/BSidesTLV2020/bugler/bugle.png)
(image taken from wikipedia)

Which as we’ll see later may give us a hist about the challenge.

Except from this bugle thing we’re being hinted about phishing and about some mysterious admin.

## Inspect
From a short investigation of the website we can see that after we register and login, we can go to our profile and fill out some details like our address and website.

In addition, we are able to upload an image as our avatar.

## Uploading “images”
Let’s focus on the ability to upload “images”:

![upload_img](https://github.com/aviyae/CTFs/blob/master/BSidesTLV2020/bugler/upload_img.png)

The first intuition is to upload html files with scripts. We can see that *.html and *.htm extensions are being blocked:

![upload_img_bad_extension](https://github.com/aviyae/CTFs/blob/master/BSidesTLV2020/bugler/upload_img_bad_extension.png)

However, there is no filtering on *.HTM extensions (in capital letters)!

Good, so we can upload html files and store them on the server on the same origin! (under /upload/)

Same origin is good for us since some properties can performed only on the same origin.

For example, sending cookies – if the page that runs the code is not in the same origin then we can’t just send the cookies back to us.

Another example we’ll see later on.

In addition, we noticed that users don’t have to be logged-in in order to access those “images”.

Can we make someone enter those pages and run our code? Maybe the admin…

## Reporting phishing
As mentioned, one of the fields in the profile is the “website” field.

When we enter a user’s profile page, we can see an interesting icon of a “phish” – probably to report the link to the admin as a phishing website.

The admin might later enter and inspect the page.

![phish1](https://github.com/aviyae/CTFs/blob/master/BSidesTLV2020/bugler/phish1.png)

(don’t pay too much attention to the %2e in the website text, we’ll get to it later – in short %2e is the URL encoding of the dot char,
we’re using it here since their server won’t upload an address to the same origin with regular dots)

Can we make the admin visit our uploaded “images”?

## Bugle
Looks like we can, by setting the website to point to our uploaded “image” path (which actually is an html file).

For example, we can upload a file that just fetches a request-bin
(request-bin is a way to collect HTTP requests.

Each request-bin is associated with a URL and each request to that URL is written in one place.

This mechanism is very convenient to see whether a link we paste is used by someone)

we used https://requestbin.com/ but any other will do the job too):

![requestbin_simple](https://github.com/aviyae/CTFs/blob/master/BSidesTLV2020/bugler/requestbin_simple.png)

Now we can get the address in the site for the “image” we uploaded:

![copy_img_addr](https://github.com/aviyae/CTFs/blob/master/BSidesTLV2020/bugler/copy_img_addr.png)

And paste it in the website field
after some tries, we see that the server probably implements some sort of filtering on the allowed websites - if the given website address has the same origin as that bugler site then the page won’t uploaded.

We don't know how that filtering works, but we can hope that some hard-coded string compared to our input.

If that's the case, then one way to try to overcome it is by trying to replace some characters with their alternative URL encoding,
like replacing the dots by %2e (which is the URL encoding of the dot char)
so the browser won’t change the behavior when it sees it.

And indeed it works and we can upload files to the same origin:

![paste_img_addr](https://github.com/aviyae/CTFs/blob/master/BSidesTLV2020/bugler/paste_img_addr.png)

Click the phish:

![phish2](https://github.com/aviyae/CTFs/blob/master/BSidesTLV2020/bugler/phish2.png)

And indeed, a few seconds after clicking this button we can see that “someone” is visiting the address we set in the “Website” field

![requestbin_out1](https://github.com/aviyae/CTFs/blob/master/BSidesTLV2020/bugler/requestbin_out1.png)

Cool, so now we can make the admin run any Javascript code we want.
Now what?

## (Not) Getting admin cookies
Now we probably want to steal the admin’s cookies! Since the uploaded images have the same origin.

But while trying to get the admin cookies (by modifying the above code to send the cookies) and sending them to our request-bin, we don’t get to our request-bin any interesting cookie :(

After more investigations (in our profile page) we see that the interesting session-cookie is marked as “httpOnly” – which means that it cannot be accessed by scripting languages like Javascript (to prevent exactly what we’re trying to do here :)) so even if the admin is logged-in to his account, we can’t get his session cookie.

We can ensure that the admin isn’t logged-in by uploading an HTM file with a script that enters the profile page of the current user via iframe and sends that page content to our request-bin (see image below, the red arrow points to the point that sends the content of the iframe to our request-bin).

In that way, the session cookie, if exists, is used by the iframe even if it can’t be sent directly to us.

Entering that page from our profile (by entering the website) sent the right page to our request-bin, but when the admin did it (when we clicked the "phish" button), we got always only the login page

![iframe_try](https://github.com/aviyae/CTFs/blob/master/BSidesTLV2020/bugler/iframe_try.png)

So, we can assume that the admin is not logged-in at all and only with what we have so far we can’t make him login with his credentials.

**But maybe the admin will login later “by his own will”.**

Can we “be there” when it happens?

## Abusing the service-workers API
When analyzing the server responses, one can see an unfamiliar field named “service-worker-allowed”:

![service_worker_allowed_hdr](https://github.com/aviyae/CTFs/blob/master/BSidesTLV2020/bugler/service_worker_allowed_hdr.png)

After some searching about that header and the authors of the challenge, this link can be found:

[https://blogs.akamai.com/sitr/2020/01/abusing-the-service-workers-api.html](https://blogs.akamai.com/sitr/2020/01/abusing-the-service-workers-api.html)

It explains that as part of the serviceWorker API it’s possible to register “worker code” (i.e. hooks) that are called when some event triggers.

One of the possible events is the “fetch” event that is triggered every time the user tries to access a resource under the path that was specified in the service-worker-allowed field (in our case, every request under the address of the site).

This means that we can run a script on the admin machine that will send to our request-bin **every request that will be fetched in the future**, including, hopefully, the credentials in the login page.


## Putting it all together
We need to create some logic that will cause the admin to (unknowingly) register some service-worker code when visiting our page…

The serviceWorker API needs a Javascript file path that defines what happens when some events are triggered.

We are only interested in the fetch event, so we can upload a JS (again, with capital letters, the non-capital are filtered) file that contains a listener (i.e. worker i.e. hook) that upon a fetch event sends the request content to our request-bin:

![service_worker_add_listener](https://github.com/aviyae/CTFs/blob/master/BSidesTLV2020/bugler/service_worker_add_listener.png)

The JS file, after uploaded, gets an address under /upload/ which we’ll use in our second file:

An HTML file that registers that Javascript file as a serviceWorker:

(Note that we are interested only in the “login” scope, so we added it in the options)

![service_worker_register](https://github.com/aviyae/CTFs/blob/master/BSidesTLV2020/bugler/service_worker_register.png)

## Attack!
So, let’s recap what we’ve done so far:

We had two main vulnerabilities:

1. Bad filtering on image uploading
2. Using service-worker-allowed header with no good reason

Good image filtering (maybe by using whitelist and not blacklist) and not sending the service-worker-allowed header would prevent this attack

We used those vulnerabilities by:
* Uploading two “images”:
  1. A JS file that should be used when registering serviceWorker – this file contains a serviceWorker listener that sends us every request that fetched by the user under the “/login” scope
  2. An HTML file that contains the registration of a serviceWorker and uses the address of the JS file we uploaded in the previous line
* Setting the HTML path in the “website” filed in our profile (after replacing dots with %2e)
* Hoping that by reporting it, the admin will enter this address and will be infected by our serviceWorker that later will send us all the requests it will do under the login scope.

Now we only need to click the phishing button and watch the magic happens.

And indeed, after several seconds the admin enters the login page “again” and tries to authenticate as admin:

![requestbin_out2](https://github.com/aviyae/CTFs/blob/master/BSidesTLV2020/bugler/requestbin_out2.png)

Decoding the body results in:

**BSidesTLV2020{S3rv1ce_W0rk3rs@Y0urS3rvic3}**

All in all, it was a fun challenge and we learned about a new mechanism :)

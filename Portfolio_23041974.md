# Client Project Portfolio

## Section 1: API and web

```py
@app.get("/api/admin/users")
@utilities.PermissionsRequired().is_admin
def get_users():
    status = int(request.args.get("status", 202))
    return jsonpickle.encode(store.get_users(status)), 200
```

```html
<template
  hx-endpoint="/api/admin/users"
  hx-data-sources="#status"
  hx-method="GET"
  hx-event="onload, onchange"
  hx-event-target="#status"
>
  <div class="flex-col box">
    <div>
      <h2>${id}: ${name}</h2>
      <a href="mailto:${email}">${email}</a>
      <br />
    </div>
    <p>Status: ${status}</p>
    <div>
      <button
        type="button"
        class="${status}-approve"
        style="color: white; background-color: green"
        onclick="req(${id}, 'PATCH')"
      >
        Approve
      </button>
      <button
        type="button"
        class="deny-${status}"
        style="color: white; background-color: red"
        onclick="req(${id}, 'DELETE')"
      >
        <span class="delete">Delete</span><span class="deny">Deny</span>
      </button>
    </div>
  </div>
</template>
```
```js
async function req(id, method) {
  let resp = await fetch("/api/admin/user", {
    method: method,
    headers: {
      "content-type": "application/json",
    },
    body: JSON.stringify({ id }),
  });
  if (resp.status === 200) {
    alert("Success!");
    location.reload();
  } else {
    alert(await resp.text());
  }
}
```
As can be seen in the code above, a JavaScript framework [HTMJ](https://github.com/acheong08/HTMJ) was used for client side rendering of JSON responses. I wrote the framework inspired by HTMX but using JSON and HTML templates rather than server side rendering. 

The HTML attributes specify the behavior of the component:
```
  hx-endpoint="/api/admin/users" # The endpoint to call
  hx-data-sources="#status" # The JSON data to send. In this case {"status": init}
  hx-method="GET"           # The HTTP method (GET/POST/PATCH/DELETE)
  hx-event="onload, onchange" # What JavaScripts events trigger a request
  hx-event-target="#status"   # Which event source should be used
 ```
This also makes the site reactive as changing the status code (pending/approved) will automatically refresh the data shown without a page refresh

This approach was chosen instead of Jinja and/or HTMX to cleanly separate front and back end logic. By having rendering be done on the client side rather than having snippets of HTML being served from the API or templating the full file, server load and bandwidth are both reduced. The static HTML/JS can be easily cached by CDNs/Browsers and thus also makes load times faster. However, excessive use of client side rendering is bad for SEO as many crawlers do not take into account JavaScript. Parts of our site is public facing and thus SEO is important, which is why dynamic/client side rendering is only used for non-public facing pages.

One disadvantage of HTMJ is the lack of ability to handle rendering logic. For example, I want to hide the "Approve" button if the account is already approved. This can be fixed with a bit more work on the framework, but to save time, I used CSS classes: `class="${status}-approve"` and `class="deny-${status}"`to handle the logic of hiding/showing the buttons.

JavaScript was used to add interactivity and handle requests rather than using `<form>` as forms do not support the full range of HTTP methods (e.g. PATCH/DELETE). This was not necessary but I felt that it made the code more understandable to use the semantically correct methods.

## Section 2: Database back end

```sql
CREATE TABLE IF NOT EXISTS users (
  id		INTEGER PRIMARY KEY,
  name	TEXT NOT NULL,
  email	TEXT NOT NULL UNIQUE,
  password_hash	TEXT NOT NULL,
  is_admin  INTEGER NOT NULL,
  availability TEXT,
  status INTEGER NOT NULL DEFAULT 102
);
CREATE INDEX IF NOT EXISTS idx_users_email ON users (email);
CREATE TABLE IF NOT EXISTS bookings (
  id          INTEGER PRIMARY KEY,
  type        TEXT NOT NULL,
  user_id     INTEGER,
  email       TEXT NOT NULL,
  status      INTEGER NOT NULL,
  timeslot_id INTEGER,
  testbed_id INTEGER,
  date        TEXT NOT NULL,
  message     TEXT,
  UNIQUE(date, testbed_id),
  CONSTRAINT chk_user_null CHECK
     (user_id IS NOT NULL or email IS NOT NULL)
);

CREATE UNIQUE INDEX IF NOT EXISTS idx_bookings_demo_conflict ON bookings
 (date, timeslot_id) WHERE type = "demo";
```

Above is a sample of our SQL statements used to create the database. We chose to embed validation checks in the database itself as redundancies in case of issues in the Python code and also to reduce the number of database calls required for each operation. For example, by casting `date` and `testbed_id` as `UNIQUE`, we don't have to make another database call to check for the existence of a booking with those two  fields and can instead just insert and catch for `sqlite3.IntegrityError`.

We also create a unique index only for demo bookings as testbed bookings do not require a `timeslot_id`. This check is also done on the client side but we all know clients data can always be spoofed.

```py
def get_user(self, with_col: str):
  match with_col:
      case "status":
          return "SELECT * FROM users WHERE status = ?"
      case "id":
          return "SELECT * FROM users WHERE id = ? LIMIT 1"
      case "email":
          return "SELECT * FROM users WHERE email = ? LIMIT 1"
```
In SELECT/INSERT/UPDATE/DELETE statements, SQL injection is prevented by passing in arguments and letting the sqlite library escape them. We used this rather than hand rolling our own due to the many edge cases that could be present which as first years, we cannot be expected to be aware of. 

```py
class PostgreSQLAdapter(SQLite):
    def _adapt(self, query: str) -> str:
        query = query.replace("?", "%s")
        query = query.replace("INTEGER PRIMARY KEY", "SERIAL PRIMARY KEY")
        query = query.replace("AUTOINCREMENT", "")
        query = query.replace('"', "'")
        query = query.replace(" end ", ' "end" ')

        return query
```
We also wrote a very basic SQLite to Postgres adapter that can be used in production. Using inheritance, we simply adapt the parent functions:
```py
def new(self, entry_type: str) -> str:
    return self._adapt(super().new(entry_type))

def update(self, entry_type: str) -> str:
    return self._adapt(super().update(entry_type))

def delete(self, entry_type: str) -> str:
    return self._adapt(super().delete(entry_type))
```
The is obviously repetitive and can probably be improved by using metaprogramming.

## Section 3: Authentication

```py
def create_jwt(user: User, secret=SECRET) -> str:  # JWT token
    return jwt.encode(
        {
            "name": user.name,
            "email": user.email,
            "id": user.id,
            "availability": user.availability,
            "is_admin": user.is_admin,
        },
        secret,
    )


def validate_jwt(token: str, secret=SECRET) -> tuple[User, bool]:
    try:
        return (
            User(**jwt.decode(token, secret, algorithms=["HS256"])),
            True,
        )
    except jwt.exceptions.PyJWTError as exc:
        print(exc)
        return {}, False
```

```py
def signup(self, user: User) -> int:
    user.password_hash = bcrypt.hashpw(
        user.password_hash.encode("utf-8"), bcrypt.gensalt()
    ).decode("utf-8")
    return self._db_exec(
        self.db.new("user"),
        (
            user.email,
            user.name,
            user.password_hash,
            int(user.is_admin),
            json.dumps(user.availability),
        ),
    )
```
```py
@app.post("/api/login")
@utilities.PermissionsRequired().not_logged_in
def login():
    email = request.form.get("email")
    password = request.form.get("password")
    try:
        user, success = store.login(email, password)
    except Exception as exc:
        print(exc)
        return flask.redirect("/login?error=User not found")
    if not success:
        return flask.redirect("/login?error=Email or password is incorrect")
    token = utilities.create_jwt(user)
    resp = flask.Response("", 302)
    resp.set_cookie("auth", token)
    resp.headers.add("location", "/")
    return resp

```
For authentication, we used `bcrypt` for password hashing and `jwt` tokens stored in cookies. `bcrypt` is a secure hash which incorporates a random salt by default, protecting against rainbow table attacks. On the client side, we used JWT tokens to reduce database access and also allow for load balancing. By having multiple servers configured with the same Postgres database and secret JWT key, we can load balance between them without sharing state such as a session token. However, JWTs make revocation difficult, meaning that we might still need a share revocation list. This was not implemented in our project and is thus a possible security vulnerability that needs to be fixed in future iterations.

In Python Flask, we use the `set-cookie` header to store the JWT token on the browser and `302` status code to denote a temporary redirect to the root path. From then on, the browser will pass the cookie in a header on each request. We can then use decorators to denote paths that require authentication.

There are inconsistencies in our HTTP status codes as we did not confidently make a decision whether to handle them via JavaScript or use 302 redirects to show the user whether or not the operation was a success.

While implementing JWT, we used the default HS256 algorithm. This choice, while not obvious at the time, also made the implementation of other features difficult to do securely (See live chat section)

## Section 4: Live chat

The client presented our group a challenge: to use websockets to create a live chat system. Using vanilla JavaScript, the built in WebSocket class, and local storage, I built a reactive chat interface.

```js
sock.onmessage = function (e) {
const data = JSON.parse(e.data);
switch (data.event) {
  case "new_connection":
    console.log("new_connection", data);
    newConnection(data.id, data.metadata.name);
    break;
  case "direct_message":
    console.log("direct_message", data);
    receiveMessage(data.from, data.body);
    break;
  case "user_list":
    console.log("user_list", data);
    for (const user of data.users) {
      sock.send(
        jsonify({
          event: "whois",
          id: user,
        }),
      );
    }
    break;
  case "exit":
    // Remove user from local storage
    const userHistory = localStorage.getItem("user-" + data.id);
    if (userHistory) {
      localStorage.removeItem("user-" + data.id);
    }
    removeConnection(data.id);
}
};

async function openChat(id) {
  const chatPane = document.getElementById("chat-pane");
  const elemId = "chat-" + id;
  chatPane.innerHTML = "";
  const chat = document.createElement("div");
  chat.classList.add("chat");
  chat.id = elemId;
  // Check local storage for messages
  const messages = JSON.parse(localStorage.getItem(elemId));
  if (messages) {
    for (const message of messages) {
      chat.appendChild(createMessage(message));
    }
  }
  chatPane.appendChild(chat);
  localStorage.setItem("current_chat", id);
  document.getElementById("chat-input").style.display = "flex";
  document.getElementById("chat-input").focus();
}
```

When first implemented, the `createMessage` function used `.innerHTML = message`. This was a security vulnerability as although attackers cannot directly inject `<script>`, they can do `<img src="nothinghere" onerror="alert('Hacked!')">` to execute JavaScript and steal cookies. When I noticed, we used `.innerText` instead.

```js
function createMessage(message, selfSend) {
  const elem = document.createElement("p");
  elem.classList.add("message");
  if (selfSend) {
    elem.classList.add("message-self");
  }
  elem.innerText = message;
  return elem;
}
```

Now how does this relate to the JWT token? You see, in the live chat, only admins should be able to see users & users should not be able to chat with anyone until an admin reaches out to them. 
```go
if conType == ConnAdmin {
  ids := make([]string, len(CONMAN.connections[ConnUser]))
  // Send a list of all users to the admin
  counter := 0
  for k := range CONMAN.connections[ConnUser] {
    ids[counter] = k
    counter++
  }
  c.WriteJSON(NewUserListMessage(ids))

  oppositeConType = ConnUser
} else {
  oppositeConType = ConnAdmin
}
```
However, how does the server know if the new user is an admin or normal user? Normally, if this was well integrated into the current system, we can just take the JWT token and authenticate normally. However, due to a mistake on my part, the live chat was implemented seperately to the rest of the API and also deployed seperately (all deployments share the same livechat deployment). This meant that we would need to share secrets! The current implementation generates a random secret on runtime initialization if not set in environment:
```py
SECRET = getenv("SECRET").encode("utf-8") if getenv("SECRET") else randbytes(32)
```
It was close to the end of the project and therefore we did not have time to do decide whether to have secrets be hard coded or find a way to share it, thus the current live chat implementation is completely unauthenticated. As a temporary workaround (HIGHLY INSECURE), we check for admin status on the client side.
```js
async function checkAdmin() {
  // Do not follow redirects
  const response = await fetch("/admin/users", {
    redirect: "manual",
  });
  if (response.status === 200) {
    return true;
  }
  return false;
}
```
Given more time, this is how I would design it instead:
- Use a JWT signing algorithm with public/private keys.
- Serve the public key from an endpoint on the main server
- When a new websocket connection is created, the chat server checks the origin header and contacts the main server for a public key
- The JWT token is also sent to the chat server which validates it with the public key. The admin status is encoded as part of the JWT

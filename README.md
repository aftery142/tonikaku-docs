# tonikaku.photos API Documentation

## Meta

### JSON Objects

This document uses [Flow type annotations] to describe the shape of JSON
objects.

[Flow type annotations]: https://flow.org/en/docs/types/objects/

### Optional fields/values

It's important to consider the following:

```js
{
  ?first: ?string,
  second: ?string,
  ?third: string
}
```

`first` may be omitted entirely (the key is optional), and its value can be
`null`. `second` may not be omitted, and its value can be `null`. `third` may
be omitted, but a value must be present if it is not.

## Tokens

There are two types of tokens, timed (tokens) and non-timed (API keys).

API keys are used for uploading programs like KShare, ShareX, etc.
Timed tokens are used for the frontend and expire in 3 days.

All tokens are specified in the `Authorization` header of any HTTP request.

It should be assumed that any endpoint uses JSON for request bodies and output
bodies unless stated otherwise.

## Status Codes

An error payload will be returned for errors (excluding HTTP 500):

```js
{
  error: true,
  message: string
}
```

### Error codes

- `HTTP 400` (Bad Request)
  - Your input data wasn't understood by the server.
- `HTTP 403` (Forbidden)
  - Invalid token or username and password.
- `HTTP 404` (Not Found)
  - File or shorten not found.
- `HTTP 420` (Enhance Your Calm)
  - Banned from the service. A reason is in the `reason` field.
- `HTTP 429` (Ratelimited)
  - You're doing that too quickly. Please wait `retry-after` seconds before
    calling again.
- `HTTP 503` (Service Unavailable)
  - That part of the API is turned off in the instance.

### Upload-specific error codes

- `HTTP 415` (Unsupported Media Type)
  - Bad or unsupported file.
- `HTTP 412` (Precondition Failed)
  - No files were given.
- `HTTP 469`
  - Your quota has been exceeded, or this file would exceed your quota if
    uploaded. Read the `message: string` field.

## Authentication

### `POST /api/login`

Authenticates a user from a username and password, returning a timed token that
is valid for a few days.

#### Input

```js
{
  user: string,
  password: string
}
```

#### Output

```js
{
  // A timed token.
  token: string
}
```

### `POST /api/apikey`

Authenticates a user from a username and password, returning a non-timed token
for uploaders and tools.

#### Input

```js
{
  username: string,
  password: string
}
```

#### Output

```js
{
  // A non-timed token.
  api_key: string
}
```

### `POST /api/revoke`

Revokes all the user's current tokens. All tokens will become invalid.

#### Input

```js
{
  username: string,
  password: string
}
```

#### Output

```js
{
  success: true
}
```

## Profile

### `GET /api/domains`

Lists all domains available to the user. If no authorization is supplied, then
only non-admin domains are returned.

#### Output

```js
{
  // Mapping of (domain ID: number) → (domain: string)
  // IDs are not guaranteed to be sequential.
  domains: { [number]: string },

  // Array of domain IDs that are official (owned by the instance itself, not
  // admins).
  officialdomains: number[]
}
```

Example:

```json
{
    "domains": {
        "0": "localhost.localdomain",
        "1": "elixi.re",
        "3": "127.0.0.1"
    },
    "officialdomains": [1, 3]
}
```

### `GET /api/profile`

- Authentication required.

Retrieves your basic account information.

#### Output

```js
{
  // A snowflake.
  user_id: string,
  username: string,
  // Inactive accounts cannot interact with the service.
  active: boolean,
  admin: boolean,
  // Domain ID of the active domain being used.
  domain: number,
  subdomain: ?string,
  email: string,
  limits: {
    limit: number,
    used: number,
    shortenlimit: number,
    shortenused: number
  }
}
```

### `PATCH /api/profile`

- Authentication required.
- Changing your password will invalidate all of your tokens.

Modifies profile information. All fields (keys) are optional, but the only value
that can potentially be null is `consented`.

#### Input

```js
{
  ?password: string,
  // Requires `password` if specified.
  ?new_password: string,
  // Domain ID.
  ?domain: number,
  // Subdomain to be used, if the domain is a wildcard. 0-63 characters.
  ?subdomain: string,
  // Domain ID to be used specifically for shortening. Specify `null` if you want
  // to inherit from the uploading domain (`domain`).
  ?shorten_domain: number,
  ?email: string,
  // Data processing consent status. `null` means that the user will be prompted
  // for their consent on the website.
  ?consented: null | boolean,
  // Determines the minimum amount of characters on generated shortnames.
  // `true` gets 8+, `false` gets 3+.
  ?paranoid: boolean
}
```

Example:

```js
{
    "password": "my old password",
    "new_password": "my new password",
    "domain": 0,
    "subdomain": "i-love",
    "consent": true,
    "paranoid": true,
    "email": "owo@owo.example"
}
```

#### Output

```js
{
  updated_fields: string[]
}
```

Example:

```json
{
    // A list of fields you have successfully updated.
    "updated_fields": ["password", "domain", "subdomain", "email", "consented"]
}
```

### `GET /api/limits`

- Requires authentication.

Retrieves your weekly upload limits.

#### Output

```js
{
  limit: number,
  used: number,
  shortenlimit: number,
  shortenused: number
}
```

### `DELETE /api/account`

- Requires authentication.

Makes a request for account deletion. Sends an email to your account asking
for confirmation.

#### Input

```js
{
  password: string
}
```

#### Output

```js
{
  success: boolean
}
```

### `POST /api/reset_password`

Makes a request for a password reset. Sends an email to your account asking for
confirmation.

#### Input

```js
{
  username: string
}
```

#### Output

```js
{
  success: boolean
}
```

## Upload

### `POST /api/upload`

- Requires authentication.

Uploads a file to be hosted on the service.

If the user is an admin, the `admin` query parameter can be specified (with any
value) to bypass checks like quotas and MIME type checking.

The file is uploaded with `multipart/form-data`. The first file in the form data
is consumed, so any field works.

You can add a `random` field to use random domains when uploading.

Use the `name` field to specify a name for your file.
Random name will be generated otherwise.

#### Examples

```sh
$ curl --request POST 'https://elixi.re/api/upload' \
  --form 'f=@file.png' \
  --header "Authorization: $TOKEN"
```

This command uploads a file named `file.png` to the service. To read from stdin
instead of a file, specify `f=@-` instead of `f=@file.png`.

#### Input

```js
{
  *: file,
  random: boolean, // default: false
  name: "" // leave default or empty for random name
 }
```

#### Output

```js
{
  url: string,
  shortname: string,

  // url to call if the client wants to delete the file
  delete_url: string,
}
```

## Shorten

### `POST /api/shorten`

- Requires authentication.

Shortens a link.

You can add a `random` field to use random domains when shortening.

Use the `name` field to specify a name for your link.
Random name will be generated otherwise.

#### Input

```js
{
  url: string
  random: boolean, // default: false
  name: "" // leave default or empty for random name
}
```

#### Output

```js
{
  url: string
}
```

## Files

### `DELETE /api/delete`

- Requires authentication.

Deletes a file.

#### Input

```js
{
  // The filename of the file to delete, without the extension.
  // For example, "abc", not "abc.png".
  filename: string
}
```

#### Output

If the file was not found, an HTTP 404 will be returned with an error payload at
the `error` field.

```js
{
  success: boolean,
}
```

### `GET/DELETE /api/delete/<shortname:str>`

- Requires authentication.

Deletes a file.

#### Output

If the file was not found, an HTTP 404 will be returned with an error payload at
the `error` field.

```js
{
  success: boolean
}
```

### `GET /api/list`

- Requires authentication.

Lists uploaded files.

#### Output

```js
{
  success: true,
  files: {
    [shortname: string]: {
      snowflake: string,
      shortname: string,
      // The size of the file in bytes.
      size: number,
      url: string,
      // A URL to a thumbnail of the file.
      thumbnail: string
    }
  },
  shortens: {
    [shortname: string]: {
      snowflake: string,
      shortname: string,
      // The URL that the shortened URL points to.
      redirto: string,
      // The shortened URL.
      url: string
    }
  }
}
```

Example:

```json
{
    "success": true,
    "files": {
        "6qw": {
            "snowflake": 422702060048355328,
            "shortname": "6qw",
            "size": 2133829,
            "url": "https://elixi.re/i/6qw.jpg",
            "thumbnail": "https://elixi.re/t/s6qw.jpg"
        },
        "ybj": {
            "snowflake": 422553767079186433,
            "shortname": "ybj",
            "size": 31606,
            "url": "https://s.ave.zone/i/ybj.png",
            "thumbnail": "https://elixi.re/t/sybj.jpg"
        }
    },
    "shortens": {
        "uec": {
            "snowflake": 422558731360931840,
            "shortname": "uec",
            "redirto": "https://example.com",
            "url": "https://elixi.re/s/uec"
        },
        "w9h": {
            "snowflake": 422558457124753408,
            "shortname": "w9h",
            "redirto": "https://elixi.re",
            "url": "https://stale-me.me/s/w9h"
        }
    }
}
```

### `DELETE /api/shortendelete`

- Requires authentication.

Deletes shortened links.

#### Input

```js
{
  // The shortcode of the shortened link to delete.
  // For example, the shortcode of "https://elixi.re/s/abc" is "abc".
  filename: string
}
```

#### Output

If the link was not found, an HTTP 404 will be returned with an error payload at
the `error` field.

```js
{
  success: true
}
```

### `POST /api/delete_all`

- Requires Authentication

Delete all files the user has. No going back from this operation.

#### Input

```js
{
  password: string,
}
```

#### Output

```js
{
  success: boolean
}
```


## Data Dump

### `POST /api/dump/request`

Requests a data dump to be delivered to your email.

#### Output

```js
{
  success: boolean
}
```

### `GET /api/dump/status`

Check the current status of your data dump request.

#### Output

```js
{
  state: "in_queue" | "not_in_queue" | "processing",

  // Only present when `state` is "processing":
  // The ISO 8601 timestamp of when the dump generation started.
  ?start_timestamp: string,
  ?current_id: string,
  // The total amount of files to process.
  ?total_files: number,
  // The current amount of files processed.
  ?files_done: number,

  // Only present when `state` is "in_queue":
  // The current position of the user in the takeout queue.
  ?position: number
}
```

### `GET /api/admin/dump_status`

- Authentication required.
- Administrator only.

#### Output

```js
{
  queue: string[],
  current: {
    user_id: number,
    total_files: number,
    files_done: number
  }
}
```

## Personal stats

### `GET /api/stats/my_domains`

- Requires authentication

Gives statistics about domains the user owns.

#### Output

```js
{
  [domain_id: string]: {
    info: {
      domain: str,
      official: boolean,
      admin_only: boolean,
      cf_enabled: boolean,
      permissions: number,
    },
    stats: {
      users: number,
      files: number,
      shortens: number,
    }
  }
}
```

## Misc

### `POST /api/recover_username`

Recover a username for a given email.

#### Input

```js
{
  email: string,
}
```

#### Output

```js
{
  success: boolean
}
```

### `GET /api/hello` / `GET /api/hewwo`

Returns the software versions.

#### Output

```js
{
  "name": string,
  "version": string,
  "api": string
}
```

### `GET /api/science`

Mocks discord's extreme data collection through the `/api/science` endpoint that happens even when you disable data collection.

#### Output

```
Hewoo! We'we nyot discowd we don't spy on nyou :3
```

### `GET /api/boron`

Gives days until 100th anniversary of Treaty of Lausanne ("Lausanne"), also returns if we're World™ Power™ or not.

2023 is promised as the year where Turkey will become a World™ Power™ by the government.
Supposedly end of Lausanne (which actually doesn't have an end date, but don't tell them that) will mean that Turkey will be able to mine Boron (we already do actually, and Lausanne doesn't block that) and get everyone rich.

#### Output

```js
{
  "world_power": false,
  "days_until_world_power": 1818
}
```

### `GET /api/features`

Lists features of the running instance.

#### Output

```js
{
  "uploads": boolean,
  "shortens": boolean,
  "registrations": boolean,
  "pfupdate": boolean
}
```

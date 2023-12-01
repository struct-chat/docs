# Struct Backend APIs

## REST APIs

### GET /v1/threads/{threadId}

- Checks Accessbility of the thread
- Checks if the thread is valid
- Gets all chats for the thread -- this could be problematic (possibly thousands of chats)
- Gets similar threads
- Sets URL path and joinChatUrl

### PUT /v1/threads

This endpoint is useful to mark a thread as deleted/hidden.

- Checks if you're the admin
- Picks up BitDeleted and BitHidden
- Ensures that the threads belong to the right org
- Updates the bits in thread
- Triggers thread reindexing by moving their done_stage

### POST /v1/threads

Create a new thread.

$ curl https://chatwoot.struct.is/v1/threads -d '{"text": "Create a new thread please, w. rev1", "xsr_ids": ["3M52"]}' -H "Authorization: Bearer ..."
{"chat_id":"1J1aeb8","ok":true,"thread_id":"2K284a"}%

### PUT /v1/read

Update the read status of a thread. Takes in a thread_id and the last read chat_id.

```
$ curl https://threads-typesense-org.struct.is/v1/read -XPUT -H "Authorization: Bearer <token>" -d '{"thread_id": "2K1da7", "chat_id": "1J1af65"}'
{"ok":true}%
```

### POST /v1/chats

This endpoints creates a new chat.

curl "https://chatwoot.struct.is/v1/chats" -XPOST -d '{"thread_id": "2Lc26", "text": "testing via HTTP apis"}' -H "Authorization: Bearer ..."
{"ok":true}%

### PUT /v1/chats

Updates a chat.

$ curl "https://chatwoot.struct.is/v1/chats" -XPUT -d '{"id": "1J1ae97", "thread_id": "2Lc26", "text": "testing editing via HTTP apis 2"}' -H "Authorization: Bearer ..."
{"ok":true}%

### GET /v1/chats/{chatId}

This endpoint returns a single chat, given a chat id.

$ curl "https://chatwoot.struct.is/v1/chats/1J1ae93" -s | jq
{
  "chat": {
    "id": "1J1ae93",
    "xid": "",
    "created_at": 1691719650,
    "updated_at": 1691719650,
    "bits": {},
    "text": "testing this one too",
    "author_id": "4K1fe9",
    "thread_id": "2Lc26",
    "images": null,
    "reactions": null
  }
}

If there are reactions, they would get filled up as well, including the user ids
of the people who reacted.

curl https://app.struct.is/v1/chats/1K6b0c -H "x-struct-orgid: 5N4" -s | jq

### GET /v1/chats/{threadId}?num=100&before={chatId}

This endpoint returns `num` number of chats, sorted in descending order of chat
id. To fetch the chats before a given chat id, pass `before`. The result would
not include the `before` id itself, only the ids which are less than before.

An empty result here means there're no more chats left.

localhost:8080/v1/chats/2K2d0c?num=100&before=1J1f000
curl "https://chatwoot.struct.is/v1/chats/2Lc26?num=10&before=1J1ae89" -s | jq

### POST /v1/reactions

This endpoint would set or delete a reaction. Note that the endpoint doesn't
care if the reaction was already there or not, that is, it's not checking for
presence. It's doing a blind write for both set and delete, and would return
{'ok': true} if the operation ran successfully.

curl https://chatwoot.struct.is/v1/reactions -d '{"chat_id": "1J1aeb5", "name": "+1", "set": true}'
curl https://chatwoot.struct.is/v1/reactions -d '{"chat_id": "1J1aeb5", "name": "+1", "delete": true}'

### GET /v1/users

Sends all the users for the given org. Simple fetch from Postgres

### GET /v1/users/{userId}

Sends just one user for the given org. Simple fetch from Postgres

### POST /v1/users

Create a new user. If the user's email already exists, this would fail. If the
org is set to "org_open_join", then anyone who's a current member can add the user.
Otherwise, only the admin can add a user.

Returns the newly added user.

### PUT /v1/users

Update an existing user. Must be either self, or an admin. This also allows to
upgrade / downgrade a user. An owner can upgrade a user to an owner. An
owner/admin can upgrade a user to admin/mod. To downgrade a user, send bits as
"no_op". Only send the fields that need to be updated about a user.

orm.go has the "User" struct, which can be used to see all available fields.

### GET /v1/channels

Sends all channels for a given org.

### PUT /v1/channels

Updates a given channel. If you set the bit, then it follows the channel-bits
logic regarding bits. Must be moderator or higher to update a channel.

You can pass `"bits": {"deleted": true}` to delete the channel. Channel can only
be deleted if there's no threads assigned to it. So, the admin must remove
this channel from all valid threads first.

### POST /v1/channels

Creates a new channel. Must be moderator or higher to create a channel.
Note that channel-bits logic.

```
$ curl localhost:8080/v1/channels -XPOST -d '{"name": "testing", "bits": {"chan_member": true}}' -H "Authorization: Bearer token"

{"data":{"id":"3Ld0d","xid":"","created_at":1698901018,"updated_at":1698901018,"bits":{"chan_member":true},"done_until":0,"name":"testing","description":"","org_ids":["5N4"],"num_threads":0,"members":["4Lb29"]},"ok":true}
```

### Channel Bits

One of these 3 bits must be present for a channel (required):
- chan_member: Must be member and have joined the channel to view threads.
- chan_anyone: Anyone can view threads without logging in.
- chan_hidden: Channel won't be visible to non-members.

This is an optional bit:
- chan_auto_join: Any new member would automatically join this channel.

### PUT /v1/channel-bits

- Checks if you're the admin
- Picks up the right bit for the channel. Can only be one of BitChannel*.
- Sets the bits for the channel.
- Note that if the channel was made public, then the threads would be marked as
    not paywall and picked up during the next sync.

### GET /v1/org

- Fetches the org.
- Counts number of total (not-hidden) threads, number of valid threads and
    number of resolved threads, and sends them too.
- Calculates the hostname.

### PUT /v1/org

- Checks if you're the admin
- Does not allow updates to various internal fields (like last_sync, or bits)
- Updates the description
- Updates hostname

### PUT /v1/join-secret

- Change the join_secret. This would invalidate the older join link.
- Only an admin/owner can change this link.
- Returns the org, with the new join_secret.

```
curl https://chat.struct.is/v1/join-secret -XPUT -d '{}' -H "Authorization: Bearer token"
```


### POST /create-org

Can call /create-org to create a new Org. Must be authorized. The caller would
become the owner of the org.

```bash
$ curl localhost:8080/create-org -XPOST -d '{"name": "test", "title": "Test", "description": "Description", "hostname": "testing-local"}' -H "Authorization: Bearer token"
```

### POST /v1/search

This is the main way to query for threads. This is what is used to generate the
feed and also to power the search.

- Finds accessible channels
- Picks up any query parameters that were passed
- Adds some usual parameters to pacify Typesense
- Note that when we import threads into Typesense, we store chats in a special
    field to aid the keyword search and embeddings. They are not stored in a way
    conducive to returning back to the user. So, essentially, we don't return
    back any chats, or similar threads. Those are both to be queried when the
    thread is opened.

### GET /v1/feeds

Gives you a list of feeds, available for a given user. The feed contains a name
and an ID that can be used to reference it. By default, every user has 3 feeds
available to them.

```
$ curl "localhost:8080/v1/feeds" -H "Authorization: Bearer token" -s | jq
{
  "data": [
    {
      "id": "6N1",
      "user_id": "0",
      "org_id": "0",
      "name": "Omni",
      "channel_ids": null,
      "user_ids": null,
      "bits": {}
    },
    {
      "id": "6N2",
      "user_id": "0",
      "org_id": "0",
      "name": "Yours Truly",
      "channel_ids": null,
      "user_ids": null,
      "bits": {}
    },
    ...
  ]
}
```

### GET /v1/feed/<feed-id>

Given a feed id, get the corresponding feed. You can pass `page` and `per_page`
args along. If no feed-id is specified, it would default to 6N1 - the 'Omni' feed.

```
$ curl "localhost:8080/v1/feed/6N1?page=2&per_page=20" -H "Authorization: Bearer token" -s | jq | head
{
  "data": [
    {
      "id": "2K2d0c",
      "xid": "slack-C01P749MET0-1688995451.590979",
      "created_at": 1688995451,
      "updated_at": 1689864413,
      "bits": {},
```

### PUT /v1/feed

You can update a feed's name, and the channels / user ids that it filters like
this. Note that you can't just pass the name to update the name. You still need
to pass all the channel_ids, user_ids and bits filters that it has (if it has
them). If you don't send those fields, they'd be updated to empty, hence
removing that particular filter. The best thing to do here is to adapt the
response from `/v1/feeds` to what you want and then send an update.

To delete a feed, you can pass in `"bits": {"deleted": true}`.

```
$ curl "localhost:8080/v1/feed" -H "Authorization: Bearer token" -XPUT -d '{"id": "6M10", "channel_ids": ["3M18"], "user_ids": ["4L6c8"], "name": "Community heart Kishore"}' -s | jq
{
  "data": {
    "id": "6M10",
    "user_id": "4Lb29",
    "org_id": "5N4",
    "name": "Community heart Kishore",
    "channel_ids": [
      "3M18"
    ],
    "user_ids": [
      "4L6c8"
    ],
    "bits": {}
  },
  "ok": true
}
```

### POST /v1/feed

Creates a new feed. You must pass name. And can pass zero or more of these
fields: channel_ids, user_ids and bits: feed_dm. Note that if you set
channel_ids, then feed_dm bit wouldn't work, and vice-versa.

When multiple channel_ids are specified, it would do an OR among them.
When multiple user_ids are specified, it would do an OR among them.
When channel_ids, user_ids, or feed_dm are specified, it would do an AND among these.

```
$ curl "localhost:8080/v1/feed" -H "Authorization: Bearer token" -XPOST -d '{"name": "only-community-kishore", "channel_ids": ["3M18"], "user_ids": ["4L6c8"], "bits": {"feed_dm": true}}' -s | jq
{
  "data": {
    "id": "6M13",
    "user_id": "4Lb29",
    "org_id": "5N4",
    "name": "only-community-kishore",
    "channel_ids": [
      "3M18"
    ],
    "user_ids": [
      "4L6c8"
    ],
    "bits": {
      "feed_dm": true
    }
  },
  "ok": true
}
```

### POST /v1/feed/search

Runs a search, just like `/v1/search`, but returns results as a feed.

```
$ curl localhost:8080/v1/feed/search -XPOST -d '{"q": "typesense", "filter_by": "user_ids := [4L6c8]", "sort_by": "created_at:asc"}'

{
  "data": [
    {
      "id": "2K3259",
      "xid": "slack-C01P749MET0-1614754663.032600",
      "created_at": 1614754663,
      "updated_at": 1614795214,
      "bits": {
        "resolved": true
      },
      "subject": "Discussions on Typesense, Collections, and Dynamic Fields",
      "summary": "<@4L6d3> shares plans to use Typesense for their SaaS platform and asks about collection sizes and sharding. <@4L6c7> clarifies Typesense's capabilities and shares a beta feature. They discuss using unique collections per customer and new improvements. <@4L6c8> and <@4L6c5> comment on threading and data protection respectively.",
      "tags": null,
      "org_id": "5N4",
      "author_id": "4L6d3",
      "xsr_ids": [
        "3M18"
      ],
      "user_ids": [
        "4L6c3",
        "4L6c5",
        "4L6c7",
        "4L6c8",
        "4L6d3"
      ],
      "num_chats": 45,
      "reactions": [
        {
          "count": 2,
          "name": "+1"
        },
        {
          "count": 1,
          "name": "raised_hands"
        }
      ],
      "url_path": "discussions-on-typesense-collections-and-dynamic-fields/2K3259",
      "read": null,
      "chats": [
        {
          "id": "1J21760",
          "xid": "slack-C01P749MET0-1614773203.041900",
          "created_at": 1614773203,
          "updated_at": 1614773203,
          "bits": {},
          "text": "Excellent. The `index: false` configuration can also be mentioned in the same way.",
          "author_id": "4L6c8",
          "thread_id": "2K3259",
          "images": null,
          "reactions": null
        },
        {
          "id": "1J21761",
          "xid": "slack-C01P749MET0-1614788999.042200",
          "created_at": 1614788999,
          "updated_at": 1614788999,
          "bits": {},
          "text": "&gt; Now if you use `scopedApiKey` to do searches instead of the main search api key, the server will automatically enforce the embedded `exclude_fields` param and users can't override it.\nI'm using exactly this! to protect sensitive data &amp; prevent excess data from being transmitted over the wire.",
          "author_id": "4L6c5",
          "thread_id": "2K3259",
          "images": null,
          "reactions": null
        },
        {
          "id": "1J21762",
          "xid": "slack-C01P749MET0-1614794754.042700",
          "created_at": 1614794754,
          "updated_at": 1614794754,
          "bits": {},
          "text": "<@4L6c5> That's great! Did you stumble on the Github issue first or did you discover that you could do this yourself?",
          "author_id": "4L6c7",
          "thread_id": "2K3259",
          "images": null,
          "reactions": null
        },
        {
          "id": "1J21763",
          "xid": "slack-C01P749MET0-1614794781.042900",
          "created_at": 1614794781,
          "updated_at": 1614794781,
          "bits": {},
          "text": "you told me :grin:",
          "author_id": "4L6c5",
          "thread_id": "2K3259",
          "images": null,
          "reactions": null
        },
        {
          "id": "1J21764",
          "xid": "slack-C01P749MET0-1614795214.043600",
          "created_at": 1614795214,
          "updated_at": 1614795214,
          "bits": {},
          "text": "Oh lol, I’ve got some bad memory! ",
          "author_id": "4L6c7",
          "thread_id": "2K3259",
          "images": null,
          "reactions": null
        }
      ],
      "highlight": {
        "chats": {
          "matched_tokens": [
            "Typesense"
          ],
          "snippet": "are planning to use <mark>Typesense</mark> for our multi-tenant SaaS"
        },
        "subject": {
          "matched_tokens": [
            "Typesense"
          ],
          "snippet": "Discussions on <mark>Typesense</mark>, Collections, and Dynamic Fields"
        },
        "summary": {
          "matched_tokens": [
            "Typesense"
          ],
          "snippet": "shares plans to use <mark>Typesense</mark> for their SaaS platform"
        }
      }
    }, ...
```

### GET /v1/tags

Gives a list of tags accessible to the user.

```
$ curl localhost:8080/v1/tags -H "Authorization: Bearer h4UHlUktfK1Cp8iDpd27ur6Lcubz3ZPGbUvvNzuus5w31NoaEYL3nRguTpr8fsPp" -s | jq
{
  "data": [
    {
      "tag": "dabra",
      "count": 1
    },
    {
      "tag": "iamthere",
      "count": 1
    },
    {
      "tag": "elsewhere",
      "count": 1
    },
    {
      "tag": "ca",
      "count": 1
    }
  ],
  "ok": true
}
```

### POST /v1/upload

Upload a file to Struct. Needs the user to be logged in. Returns the handle, that can be added to the chat.

```bash
curl https://structchat.struct.is/v1/upload -XPOST -H "Authorization: Bearer <token>" -F "file=@../Downloads/launch-blog-cover.png"
{"data":{"ctype":"image/png","fname":"launch-blog-cover.png","handle":"img-1697049819658917","size":845632},"ok":true}
```

### GET /v1/anons

```
$ curl localhost:8080/v1/anons -s | jq | head
{
  "data": {
    "adams-alpha": [
      "adams-alpha",
      "adams-bravo",
      "adams-charlie",
      "adams-delta",
      "adams-echo",
      "adams-foxtrot",
      "adams-golf",
```

## WebSockets

### /presence

```
> {"path": "/presence"} // To mark the user as active
// It is expected that the FE would send a presence indicator every 3 mins or so.
// But, the FE can send it as many times as they like.

// To mark the user as active and also get a list of all active users.
> {"path": "/presence", "args": {"users": true}}

// In updates, you'd either get status: active, or status: away. A user is
marked as away after 10 mins of no presence updates.

< {"path":"/presence","data":{"status":"active","user_ids":["4L6c8","4Lb29"]}}
< {"path":"/presence","update_id":3,"data":{"status":"away","user_ids":["4Lb29","4L6c8"]}}
```

### /feed

When a user switches to a feed, websocket should send the feed ID along with
some of the top thread ids that are being shown to the user. This helps the
backend tell the frontend when to move a thread to the top.

```
> {"path": "/feed", "args": {"feed_id": "6N1", "thread_ids": ["2K5c1c", ...]}}

# For a thread known to the FE, just a "move_to_top" marker is sent.
< {"thread_id":"2K5c1c","path":"server/feed","update_id":3,"action":"move_to_top"}

# For unknown thread -- the entire FeedThread is sent. Once sent, the thread is
# marked as known. This data is all that FE needs to know to fill in feed view.

< {"thread_id":"2K5c1c","path":"server/feed","update_id":1,"action":"move_to_top","data":{"id":"2K5c1c","xid":"slack-C01P749MET0-1692109330.170919","created_at":1692109330,"updated_at":1694051250,"bits":{},"subject":"Revolutionizing Communication: Reinventing the Chat Platform for the Next Generation","summary":"Reinventing chat platform is crucial for staying relevant in today's digital age. The rise of messaging apps has led to increased demand for secure, real-time communication. The focus is on creating user-friendly platforms that integrate new technologies such as AI, automation, and voice recognition to improve user engagement and productivity.","org_id":"5N4","author_id":"4Kfb24","xsr_ids":["3M18"],"user_ids":["4L6c7","4Lb29","4Kfb24"],"num_chats":7,"reactions":null,"url_path":"","read":{"id":"0","user_id":"4Lb29","thread_id":"2K5c1c","chat_id":"0","num_unread":0,"muted_until":0},"chats":[{"id":"1J2f7eb","xid":"slack-C01P749MET0-1692121259.809889","created_at":1692121259,"updated_at":1692121259,"bits":{},"text":"hey jason, looking into it. will revert soon on this.","author_id":"4Kfb24","thread_id":"2K5c1c","images":null,"reactions":null},{"id":"1J2f901","xid":"","created_at":1694051105,"updated_at":1694051105,"bits":{},"text":"pushing a new chat","author_id":"4Lb29","thread_id":"2K5c1c","images":null,"reactions":null},{"id":"1J2f902","xid":"","created_at":1694051144,"updated_at":1694051144,"bits":{},"text":"pushing a new chat","author_id":"4Lb29","thread_id":"2K5c1c","images":null,"reactions":null},{"id":"1J2f903","xid":"","created_at":1694051250,"updated_at":1694051250,"bits":{},"text":"pushing a new chat","author_id":"4Lb29","thread_id":"2K5c1c","images":null,"reactions":null},{"id":"1J2f904","xid":"","created_at":1694051366,"updated_at":1694051366,"bits":{},"text":"pushing a new chat","author_id":"4Lb29","thread_id":"2K5c1c","images":null,"reactions":null}]}}
```

# tmi-rate-limits: Twitch IRC Rate Limiting

## Commands overview

Possible COMMANDs the client can send to twitch:
  - `PASS`
  - `NICK`
  - `JOIN`
  - `PART`
  - `PRIVMSG`
  - `CAP REQ`
  - `PING`
  - `PONG`

<!--
Twitch also accepts these (after login):
i.e. you dont get a
:tmi.twitch.tv 421 randers01 COMMANDNAME :Unknown command

MODE (does nothing?)
NICK (causes disconnect)
PASS (does nothing?)
USER (does nothing?)
-->

## Commands and their rate limit buckets

These commands each count towards the following buckets:

  - `PASS`: none
  - `NICK`: none
  - `JOIN`: none
  - `PART`: none
  - `PRIVMSG`: `privmsg-bucket`
  - `CAP REQ`: unknown, probably not.
  - `PING`: none
  - `PONG`: none

## Rate Limit Buckets:

Twitch IRC uses rate limit buckets, [according to staff][forum1]. A rate limit bucket in this case means that for every user, Twitch keeps a number of remaining requests. A request uses up a token from the bucket. If there is an incoming request, and the bucket is at 0, then the request is denied/fails. The bucket is reset to its original amount (the refill amount, e.g. 100) on a fixed schedule: the refill rate (e.g. every 30 seconds).

On the client, you do not know when these buckets refill. Therefore it is probably smart to implement a sliding-window rate limiter with the bucket parameters - which is a type of implementation where you assume the bucket refilled exactly at `tNow - refillRate` - so you at no point in time ever could possibly exceed the rate limit.

A typical implementation might be to have a semaphore that is initialized with the bucket size. When sending a command, you remove a token from the semaphore, and after you are done successfully sending the message, you start a timer/sleep for the refill time, and refill the semaphore after that time.

### `privmsg-bucket`

From what I understand, this is split into two buckets: Let's call them `privmsg-moderator-bucket` and `privmsg-user-bucket`.

If you send a PRIVMSG, its bucket use depends on moderator/VIP/broadcaster status in the destination channel.

If your account has moderator/VIP/broadcaster status in the target channel, a token is consumed from the `privmsg-moderator-bucket`.

If your account does not have any of these statuses, the given PRIVMSG uses a token from both the `privmsg-moderator-bucket` and `privmsg-user-bucket`. (Meaning that if either of these buckets is drained, you exceeded the rate limit.)

#### `privmsg-moderator-bucket`

  - Refill Rate: Every 30 seconds
  - Refill Amount:
      - If you have an ordinary account: 100
      - If your account has the `known` status: 100 (this used to be 50 but [has been fixed](https://github.com/twitchdev/issues/issues/198))
      - If your account has the `verified` status: 7500
  - This bucket is per twitch-user.

#### `privmsg-user-bucket`

  - Refill Rate: Every 30 seconds
  - Refill Amount: 
      - If you have an ordinary account: 20
      - If your account has the `known` status: 50
      - If your account has the `verified` status: 7500
  - This bucket is per twitch-user.

[forum1]: https://discuss.dev.twitch.tv/t/the-final-answer-on-join-limits/8505/2
[reddit1]: https://www.reddit.com/r/Twitch/comments/27ho7q/irc_client_rate_limit_question/ci10pbz/

## Things to consider regarding ingress reliability

To ensure you always stay connected and JOINed to all channels you want, it's a good idea to spawn many small connections (small as in the amount of channels they `JOIN`) so you can ideally immediately reconnect them and rejoin all channels should they disconnect. Suggested shard size is 50-100 channels (or so).

You should send `PING`s from the client to the server to check your connectivity. By default, the only requirement for the connection to stay alive is for the client to answer the server's `PING`s with a `PONG`, but you can also `PING` the server and it will respond to you with a `PONG`. Should the server fail to `PONG` back in a timely manner, you should reconnect.

The twitch server responds to your PING irc message like this:

```
< PING
> PONG :tmi.twitch.tv
< PING asd
> :tmi.twitch.tv PONG tmi.twitch.tv :asd
< PING :asd def
> :tmi.twitch.tv PONG tmi.twitch.tv :asd def
< PING asd def
> :tmi.twitch.tv PONG tmi.twitch.tv :asd
```

i.e. it responds to you with the `PONG` command and the IRC message arguments `['tmi.twitch.tv', yourMessageArguments[0]]`.

## Things to consider regarding egress reliability

  - Messages can get caught in AutoMod. (Moderators and Broadcasters are naturally exempt from this.) Note that the broadcaster can create phrases that even affect moderators (you will get a `msg_rejected_mandatory`.)
      - If your message matched a blocked phrase, you will get this response via IRC:

            @msg-id=msg_rejected_mandatory :tmi.twitch.tv NOTICE #randers00 :Your message wasn't posted due to conflicts with the channel's moderation settings.

      - If your message was caught in the general automod filter (and is waiting for moderator approval/denial), you will get this message on PubSub:

            {
               "type":"MESSAGE",
               "data":{
                  "topic":"chat_moderator_actions.254911995.40286300",
                  "message":"{\"data\":{\"type\":\"chat_targeted_login_moderation\",\"moderation_action\":\"automod_message_rejected\",\"args\":[\"randers01\"],\"msg_id\":\"0416447a-2355-4c51-86a8-56f61c995d9c\",\"target_user_id\":\"254911995\",\"target_user_login\":\"\"}}"
               }
            }

      - Note that due to the way AutoMod operates, trying to guarantee delivery of a `PRIVMSG` by listening back to IRC and trying to see exactly the message you sent appear as a `PRIVMSG` in the given channel can lead to false results, because twitch can and will change your message due to AutoMod, even if you are a moderator or broadcaster(!).
        Apart from that, even if AutoMod is disabled, trying to check for successful delivery based on this method can be problematic due to the various other ways twitch can change your message text around before it arrives as a `PRIVMSG` in chat.

  - Twitch limits the speed of really fast chats by dropping messages if they exceed a per-channel rate limit bucket of unknown refill rate and unknown refill amount.
    If you send a message while this bucket is drained, your message is not delivered to chat (it is dropped).
    Moderators, VIPs, Broadcasters and Verified Bots are exempt from this.
  - Twitch will drop messages it considers too repetetive, for example 500x "a" will get dropped silently.
    Moderators, VIPs and Broadcasters are exempt from this.
  - Twitch will drop a message if it is equal to "**the last one you sent, less than 30 seconds ago**" (this restriction is per-channel). If your intention is to send such a repeating message, you can append `" \u{E0000}"` to duplicate messages to be able to send the same message again.
    This filter is applied after various message content filters, for example message length trimming to 500 characters, message normalization (removal of neighbouring spaces) and message trimming (removal of any whitespace at the end or start of the message). This filter appears to consider the message as-it-would-arrive-in-chat.
    Note that the "last message" that this filter considers is the last successfully delivered message to chat, after automod and all filters.
    If you were to implement this filter, considering the last message you got back through IRC as a PRIVMSG to be your "last message" is probably a good idea.
    The NOTICE when you trigger this filter looks like this:

        @msg-id=msg_duplicate :tmi.twitch.tv NOTICE #randers00 :Your message was not sent because it is identical to the previous one you sent, less than 30 seconds ago.

    Moderators, VIPs and Broadcasters are exempt from this filter.
  - **Per-Channel Minimum Slowmode**: All channels have a minimum slow-mode of **1 second**. If a channel has slow-mode enabled, the slow-mode you have to respect will be larger respectively (see `ROOMSTATE`).
    If you send a message that gets dropped due to global slowmode, the notice sent to you will look like this:
    
        @msg-id=msg_ratelimit :tmi.twitch.tv NOTICE #randers00 :Your message was not sent because you are sending messages too quickly.

    If you send a message that gets dropped due to normal slowmode, the notice will look like this:

        @msg-id=msg_slowmode :tmi.twitch.tv NOTICE #randers00 :This room is in slow mode and you are sending messages too quickly. You will be able to talk again in 4 seconds.
        
    In some chats, subscribers can be exempt from the channel-specific slowmode. (This is a channel-specific setting the broadcaster can make. You can view this info on https://www.twitch.tv/subscriptions, "Exempt from slow-mode" will be listed as the subscriber perks when you show the details about a subscription you have. The data is not available on any public Twitch API.

  - Timeouts and Bans: If you are timed out or banned your PRIVMSGs will get dropped, obviously.
    Permanent ban:

        @msg-id=msg_banned :tmi.twitch.tv NOTICE #randers00 :You are permanently banned from talking in randers00.

    Timeout:

        @msg-id=msg_timedout :tmi.twitch.tv NOTICE #randers00 :You are banned from talking in randers00 for 86387 more seconds.

    Timeouts and Bans look like this, should you wish to listen for them (`randers01` is being banned from room `randers00`):
    Timeout for `12345` seconds:
    
        @ban-duration=12345;room-id=40286300;target-user-id=254911995;tmi-sent-ts=1550594103696 :tmi.twitch.tv CLEARCHAT #randers00 :randers01
        
    Permanent ban:
    
        @room-id=40286300;target-user-id=254911995;tmi-sent-ts=1550594146099 :tmi.twitch.tv CLEARCHAT #randers00 :randers01
        
  - Shadowbans: Entire IP or account can be shadowbanned globally. In this case user is able to connect and join normally, but all of his messages are dropped.
  
    - Softer shadowban version:
      
      Happens when user posts too many messages, lasts 30 minutes (https://dev.twitch.tv/docs/irc/guide#command--message-limits). Can be bypassed if user is sub, VIP, mod or broadcaster.
      
    - Harder shadowban version:
    
      Happens when user triggers Twitch undocumented internal filters and gets banned. In this case user also cannot send whispers. Can be bypassed if user is a mod or a broadcaster.

Note that in general that all messages that are dropped internally due to any of these filters still consume from the `privmsg-xxx-bucket`s.

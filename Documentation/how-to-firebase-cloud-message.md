# Sending push notifications with Firebase cloud message (FCM)

Before digging into details it's important to understand the architectural idea
![image](https://user-images.githubusercontent.com/1279756/41772490-2402772a-7619-11e8-9acf-f0c17cbd0b75.png)

# Diagram
1) Through the FCM-SDK (or the native integration) the app will request a push-token at its platform push-network. This will trigger a popup to allow push notification on most platforms. This push-token is unique for the App, device & certification (release, debug etc) and will expire after x days. Luckily the UA SDK will deal with refreshing it.

2) Through FCM-SDK / FCM-API the push-token will be sent to FCM and stored. FCM can now send push notifications to that device & app & certificate. Since we do not want to deal with push-tokens vs users references. It's important to register to topic as well. [Read more](https://github.com/nodes-projects/readme/blob/master/mobile/firebase-push-guide.md#topics)

3) Sending a push notification from a backend project happens via the API. It's normally triggered by
 - Admin panel as a view, send to X, Y & Z with a custom alert.
 - Triggered by an event, fx some else liked your post. The alert message would be from NStack / Localization

4) FCM will look up the topic or topic conditions, loop through each push-tokens registered to this channel. And sent 1-by-1 to each push-network. This is done Async in queue.

5) The push-network will queue the request from FCM and sent the push notification through a socket connection to the device. If the device is not online, it will save it for x hours (depending on push-network)

# SDKs

Vapor: https://github.com/mdab121/vapor-fcm :warning:

**Until they have merged my PR, please use https://github.com/cweinberger/vapor-fcm/releases/tag/2.1.1. Otherwise your will always receive an unsuccessful Response from the framework (even though the notification was sent successfully).**

PHP: https://github.com/kreait/firebase-php

# Read here for detailed description of push options for iOS, Android & Web

iOS:https://firebase.google.com/docs/reference/fcm/rest/v1/projects.messages#ApnsConfig

Android: https://firebase.google.com/docs/reference/fcm/rest/v1/projects.messages#AndroidNotification

Web: https://firebase.google.com/docs/reference/fcm/rest/v1/projects.messages#webpushconfig

# Setting up projects in Firebase

In FCM Console you create projects for each application & environment.

For application ABC there should be 3 FCM projects

- ABC - Development
- ABC - Staging
- ABC - Production

Within each project you can set up the apps for iOS, Android (and Web).

The API credential & url can always be found at

https://console.firebase.google.com/u/0/project/[app-name]/settings/serviceaccounts/adminsdk

Set up android / ios / web cloud message credentials here

https://console.firebase.google.com/u/0/project/[app-name]/settings/serviceaccounts/cloudmessaging

Note: Credentials (json) and database URI should be setup as server variables and never be hardcoded

# Alert
A push message has a "alert" which is the string showed in the notification center. It has a limit of 255 chars. So limit it to less. 

Always put the alert message in NStack / Localization

Note: iOS already truncate them earlier (around 110 chars) 

__We highly recommend using both title & body, but body is option__

You can either use the notification object in firebase sdk or use the APNS / Android specific objects


PHP: Generic alert both with title & body
```php
$notification = Notification::create()
    ->withTitle($title)
    ->withBody($body); // Optional
```
PHP: Android specific message 
```php
$config = AndroidConfig::fromArray([
    'ttl' => '3600s',
    'priority' => 'normal',
    'notification' => [
        'title' => '$GOOG up 1.43% on the day',
        'tag' => '$GOOG',
        'body' => '$GOOG gained 11.80 points to close at 835.67, up 1.43% on the day.',
        'icon' => 'ic_stock_ticker',
        'color' => '#f45342'
    ],
]);
```

*Notification keys and value options*:
`tag`: Identifier used to replace existing notifications in the notification drawer. If not specified, each request creates a new notification. If specified and a notification with the same tag is already being shown, the new notification replaces the existing one in the notification drawer.

`color`: The notification's icon color, expressed in #rrggbb format.

`icon`: Unless specified, let the Android app deal with this. Kind of similar to sound, but with notification icons instead.


PHP: iOS Specifc message
```php
$config = ApnsConfig::fromArray([
    'headers' => [
        'apns-priority' => '10',
    ],
    'payload' => [
        'aps' => [
            'alert' => [
                'title' => '$GOOG up 1.43% on the day',
                'body' => '$GOOG gained 11.80 points to close at 835.67, up 1.43% on the day.',
            ],
            'badge' => 42,
        ],
    ],
]);
```


# Payloads / Extra / data

Payload is a way to pass more information, fx for deeplinking

PHP: Example of a payload
```php
$message = $message->withData($[
    'first_key' => 'First Value',
    'second_key' => 'Second Value',
]);
```

This will let the app know where to deeplink

FCM does not support nested data. If you need to send a full model, consider json-encoding it to 1 key

Note: there is limits on iOS for maximum 2kb of data

# Topics

We always register userId as a topic `user_[ID]`eg `user_1`

Topics can be used very smart, and should avoid every situation of having to store push settings in the DB. 

## Example 1)

Feature: Push on breaking news which is optional for users (e.g. configurable in settings)

The app has to subscribe to a topic `breakingNews` if the user turns on that setting (i.e. through a toggle).
If the user disables the setting, the app has to unsubscribe from `breakingNews`.

Whenever breaking news are to be shared, the backend will send a FCM (push) message to the topic `breakingNews`.

## Example 2)

Feature: Recieving push notifications about athlete with id: 12345, after favoriting them

The app will subscribe to the topic `athelete_12345`

The backend will now just push to the topic `athlete\_[ID]` instead of looping users which have favorited it (this way, it might be even not necessary to store the relation/state (user favorited athlete) on the backend)

## Example 3)

The app has options to only receive news push notification during race or always 

The app will subscribe to the topic `newsAlways` or `newsDuringRace` or both

The Backend will push to those topics, but will figure out if the the current time is during race and add that topic as well.

Note: Users who subscribed to both topics will not get the notification twice.

PHP: topic
```php
$topic = 'a-topic';

$message = MessageToTopic::create($topic)
    ->withNotification($notification) // optional
    ->withData($data) // optional
;
```

PHP: Topic conditions
```php
$condition = "'TopicA' in topics && ('TopicB' in topics || 'TopicC' in topics)";

$message = ConditionalMessage::create($condition)
    ->withNotification($notification) // optional
    ->withData($data) // optional
;
```

## Sound

There is 3 options for sound

### Standard device sound

PHP iOS Standard sound
```php
 ->withApnsConfig(ApnsConfig::fromArray([
            'payload' => [
                'aps' => [
                    'alert'             => $message,
                    'sound'             => 'default',
                ],
            ],
        ]));
```

PHP Android Standard sound
```php
$config = AndroidConfig::fromArray([
    'ttl' => '3600s',
    'priority' => 'normal',
    'notification' => [
        'title' => '$GOOG up 1.43% on the day',
        'body' => '$GOOG gained 11.80 points to close at 835.67, up 1.43% on the day.',
        'sound' => 'default'
    ],
]);
```
 
### No sound

Android: The documentation is pretty lackluster on this. For Android 8.0+ notification channel's will probably override this anyway.
iOS: empty string. ""

PHP iOS No sound
```php
 ->withApnsConfig(ApnsConfig::fromArray([
            'payload' => [
                'aps' => [
                    'alert'             => $message,
                    'sound'             => '',
                ],
            ],
        ]));
```

### Custom sound

Here we have to set sound for ios & android specificly

`sound`: The sound to play when the device receives the notification. Supports "default" or the filename of a sound resource bundled in the app i.e. `stocksound.mp3`. Android: Sound files must reside in /res/raw/.

PHP Android example

```php
->withAndroidConfig(AndroidConfig::fromArray([
                'notification' => [
                    'title' => $message,
                    'sound' => $androidSound ? ('arrivedsound.mp3') : null,
                ],
            ]))
```

PHP Android example

```php
 ->withApnsConfig(ApnsConfig::fromArray([
            'payload' => [
                'aps' => [
                    'alert'             => $message,
                    'sound'             => 'arrivedsound.wav',
                ],
            ],
        ]));
```

## Priority

This is starting to be a very important matter, notification without high prio. Can easily take minutes to be send

*PHP: Android high priority*

`priority`: Message priority. Can take "normal" and "high" values. 

*Normal priority messages* are delivered immediately when the app is in the foreground. When the device is in Doze or the app is in app standby, delivery may be delayed to conserve battery. For less time-sensitive messages, such as notifications of new email, keeping your UI in sync, or syncing app data in the background, choose normal delivery priority. 

FCM attempts to deliver *high priority messages* immediately, allowing the FCM service to wake a sleeping device when necessary and to run some limited processing (including very limited network access). High priority messages generally should result in user interaction with your app. If FCM detects a pattern in which they don't, your messages may be de-prioritized.

```php
 ->withAndroidConfig(AndroidConfig::fromArray([
                'priority'     => 'high',
                'notification' => [
                    'title' => $message,
                ],
            ]))
```

*PHP: iOS high priority*

```php
 ->withApnsConfig(ApnsConfig::fromArray([
            'payload' => [
                'aps' => [
                    'alert'             => $message,
                    'sound'             => 'arrivedsound.wav',
                    'header'            => [
                        'apns-priority' => 10,
                    ],
                ],
            ],
        ]));
```

No priority 10 is highest, but does not work together with content-available

## Silent / Content-available

Sometimes we use push to notifity the phone about a update, but we do not want it to show in the notification center.
eg booking was updated, with updates in payload or please pull newest booking

### PHP iOS example
```php
 $apns = [
            'payload' => [
                'aps' => [
                    'alert'             => $message,
                    'sound'             => $iosSound,
                    'content-available' => $silent ? 1 : 0,
                    'header'            => [
                        'apns-priority' => $silent ? 5 : 10,
                    ],
                ],
            ],
        ];

        if($silent) {
            unset($apns['payload']['aps']['alert']);
            unset($apns['payload']['aps']['sound']);
        }
        
 ->withApnsConfig($apns);
```
The reason of this code, is that "alert" & "sound" is not allowed when sending silent pushes. It should not even be in object, else it will be ignored as silent
Same goes for priorty, 5 is highest for silent


### PHP Android example
To send silent push notifications send a payload with only `data` object set. The Android app will then receive data payload in `onMessageReceived`.
```php
$message = $message->withData([
    'someKey' => 'someValue',
]);
```


## Badge count
This is originally an iOS feature
![image](https://cloud.githubusercontent.com/assets/1279756/25580264/c21753e6-2e7f-11e7-9499-265f73ea79e9.png)

But there is an option for doing something simular on android also

![image](https://cloud.githubusercontent.com/assets/1279756/25580290/084cdfe8-2e80-11e7-9f09-e13c0c282866.png)

There is 2 ways of doing badge count

### The big solution

Like FB, Google etc, let the server keep track of unread count. And send silent push notification when this updates, to keep all you divices / platforms aligned 

This can be very time consuming and often require a full activity / notification system before hand

### The simple solution

Use the +1 value, which will increase the counter by one in ios. And when you open to the app / specific view, you clear the count

PHP: Android, there is no build in badge count, just add it to payload
```php
$message = $message->withData([
    'badge' => 45,
]);
 ```
 
PHP: iOS
```php
 ->withApnsConfig(ApnsConfig::fromArray([
            'payload' => [
                'aps' => [
                    'badge'             => 45
                    'alert'             => $message,
                    'sound'             => 'arrivedsound.wav',
                    'header'            => [
                        'apns-priority' => 10,
                    ],
                ],
            ],
        ]));
 ```

# Localization 

First you need to setup a system to keep track of language for each user. 

Setup a field on the user model "locale" and store the Accept-Language header here.
Now when sending a push notification you can look up the user's locale beforehand, and thereby translate it.

This solution requires you to loop through each user. It is possible to send arrays of namedUsers/aliases to minimize the network to Firebase (which is the slow part)

Note: Also a creative solutions with tags can be used

# Logs

Unfortually there is no logs in FCM of push messages sent. Therefore we integrated this feature in NStack, where push logs are stored for 3 months

It's called "Push logs" [Link](https://nstack.io/admin/ugc/push-logs)

PHP example using the nodes/nstack package
```php

 $request = $messageObject->jsonSerialize();

        try {
            $response = $this->getClient()->getMessaging()->send($messageObject);

            try {
                nstack()->pushLog(
                    'fcm',
                    app()->environment(),
                    $data['type'] ?? 'N/A',
                    true,
                    $request,
                    $response,
                    $message,
                    $userId,
                    $relation
                );
            } catch (\Throwable $e) {
                bugsnag_report($e);
            }
        } catch (\Throwable $e) {
            // report
            bugsnag_report($e);

            try {
                $response = [
                    'code'    => $e->getCode(),
                    'message' => $e->getMessage(),
                ];

                nstack()->pushLog(
                    'fcm',
                    app()->environment(),
                    $data['type'] ?? 'N/A',
                    false,
                    $request,
                    $response,
                    $message,
                    $userId,
                    $relation
                );
            } catch (\Throwable $e) {
                bugsnag_report($e);
            }
        }

```



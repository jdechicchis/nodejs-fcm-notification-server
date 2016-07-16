# nodejs-fcm-notification-server
This is a Node.js template which communicates with FCM to send push notifications to several devices at the same time. Meant to be used with a Firebase database and deployed on Heroku.

# How it works
The solution presented here is as follows:
1. The client adds a child to **notifications/** in the Firebase database using `childByAutoId`. This child has the following structure:
```
["attempts": 0,
"key": notificationKey,
"message": "Hello world!",
"read": false,
"sent": false,
"senderUID": abc...xyz,
"recipientUID": 123...789,
"time": timeStamp]
```
Here `notificationKey` is the key created using `childByAutoId`. Any other information that is app specific can also be added to this dictionary. For example, and app with posts may add a `postID` key value pair.
2. Once the client adds a new child child to **notifications/** this change it detected by the Node.js backend in the `child_added` callback.
3. The new notification dictionary, `snapshot`, is passed to `handleNotificationSnapshot(snapshot)`.
4. `handleNotificationSnapshot` retrieves the notifications ids (tokens) of the recipient using their UID. The notification ids are stored in **notification_ids/recipientUID/** as an array of tokens like so:
`[token1, token2, token 3]`
FCM can handle up to 1000 tokens in one push notification request. (Number may need to be double checked because I've also read a load limit of 2K)
4. The notification ids, message, notification key, and number of attempts are then passed into `sendNotification(notificationID, message, key, attempts)`.
5. `sendNotification` uses an HTTP POST request to communicate with FCM to send the push notification.
6. If the POST request return a 200 success code, the notification dictionary's `sent` value is changed to `true`, and `attempts` is incremented by one. Otherwise, `attempts` is incremented by one, and the change is detected in `child_changed` which calls `handleNotificationSnapshot`. The notification will not be sent after 10 failed attempts.

# Deploying the Node.js backend
This project is made to be deployed on [Heroku](https://www.heroku.com/), although it can be changed to work on any Node.js server.

In addition to hosting the code, there are to Firebase credentials you need for this to work. First is a [service account](https://developers.google.com/identity/protocols/OAuth2ServiceAccount). Make sure to add the json service account file to the root directory of your project. **_Note that the service account grants administrative access to anyone who has it. Also, consider using a `databaseAuthVariableOverride` to ensure any code mistakes do not affect your entire database._** The other thing you need is your Firebase project's Server Key which can be found in Settings, -> Cloud Messaging.

When testing locally, make sure to run `npm install` to install all of the dependencies.

# Saving notification ids (tokens)
Follow the [Firebase documentation](https://firebase.google.com/docs/cloud-messaging/) to set up the client side.

The below swift code adds the devices token to the user's notification ids array.
```javascript
func connectToFcm() {   // Connect to the FCM platform and save the instance token in the Firebase database for the backend Node.js app server to send notifications when childs are updated
        FIRMessaging.messaging().connectWithCompletion { (error) in
            if (error != nil) {
                NSLog("Unable to connect with FCM. \(error)")
            } else if FIRInstanceID.instanceID().token() != nil {
                NSLog("Connected to FCM.")

                if let user = FIRAuth.auth()?.currentUser {     // Check that the user is logged in
                    // User is signed in.

                    let ref = FIRDatabase.database().reference()    // Create the Firebase database reference

                    ref.child("notification_ids/\((user.uid))").observeSingleEventOfType(.Value, withBlock: { (snapshot) in

                        if snapshot.exists() {  // Ensure that snapshot exists to avoid option unwrapping errors

                            let notificationIDsArray = snapshot.value as! NSMutableArray    // All of all user notification tokens. Having an array allowa the Node.js server to send push notifications to multiple devices simultaneously

                            if notificationIDsArray.indexOfObject(FIRInstanceID.instanceID().token()!) == NSNotFound {  // Check is the token is alrrady saved. If not, add the token to the notificationIDsArray and save it to the Firebase database
                                notificationIDsArray.addObject(FIRInstanceID.instanceID().token()!)

                                ref.child("notification_ids/\(user.uid)").setValue(notificationIDsArray)
                            }
                        }
                        else {  // If the snapshot does not exist, there is no record of notification tokens for the logged in user. In this case, create a database child to store the user's notifications tokens and save it

                            let saveArray: NSArray = NSArray(object: FIRInstanceID.instanceID().token()!)

                            ref.child("notification_ids/\(user.uid)").setValue(saveArray)
                        }

                    }) { (error) in
                        NSLog("user notification load error")
                        NSLog(error.localizedDescription)
                    }
                } else {
                    // No user is signed in.
                    NSLog("not signed in")
                }
            }
        }

```

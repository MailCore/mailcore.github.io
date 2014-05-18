# Getting Started

This provides a brief overview of the MailCore API and how it is structured.

## Sending E-mail

Sending e-mail with MailCore is very easy, it takes care of all the details like MIME encoding, and the SMTP protocol. First, start by creating a `CTCoreMessage` instance. Then set the subject, body, from, and atleast one to, bcc, or cc recipient. One detail to note is that the to, cc, bcc attributes take an NSSet of `CTCoreAddress` recipients.
  
    CTCoreMessage *testMsg = [[CTCoreMessage alloc] init];
    [testMsg setTo:[NSSet setWithObject:[CTCoreAddress addressWithName:@"Monkey" email:@"monkey@monkey.com"]]];
    [testMsg setFrom:[NSSet setWithObject:[CTCoreAddress addressWithName:@"Someone" email:@"test@someone.com"]]];
    [testMsg setBody:@"This is a test message!"];
    [testMsg setSubject:@"This is a subject"];

  
Once the message attributes have been set, all that is left is sending the message using `CTSMTPConnection`:
    
    NSError *error;
    BOOL success = [CTSMTPConnection sendMessage:testMsg 
                                          server:@"mail.test.com"
                                        username:@"test"
                                        password:@"test"
                                            port:587
                                  connectionType:CTSMTPConnectionTypeStartTLS
                                         useAuth:YES
                                           error:&error];
    if (!success) {
        // Present the error
    }

That's all, let me know if you have any problems/suggestions/recommendations

## Working with IMAP

Working with IMAP through MailCore is quite easy. First you need to instantiate a `CTCoreAccount` object and establish a connection like so:

    CTCoreAccount *account = [[CTCoreAccount alloc] init];
    BOOL success = [account connectToServer:@"mail.theronge.com"
                                       port:143
                             connectionType:CTConnectionTypePlain
                                   authType:CTImapAuthTypePlain
                                      login:@"test"
                                   password:@"none"];
    if (!success) {
        // Display the error contained in account.lastError
    }

If by any chance it fails to make a connection NO will be returned. Check the account's lastError property to get the exact error. **Important: lastError is frequently used, other MailCore objects also use it to store the most recent error.** Now that we have a connection to the server we can go ahead and get a list of folders.

    NSSet *subFolders = [account subscribedFolders];

If we take a look at the resulting NSSet, we see that a set of NSStrings with folder path's have been returned.

    <NSCFSet: 0x309d60> (INBOX.Test.Test22, INBOX, INBOX.TheArchive, INBOX.Test, INBOX.Trash)

We can use one of those strings to create a `CTCoreFolder` object, for example this sets up a connection to the INBOX on the server:

    CTCoreFolder *inbox = [account folderWithPath:@"INBOX"];

Now that we have a connection to a folder we can start to do more interesting things, like we can get an array of the messages in the particular folder, or we can delete the folder, or remove the folder from the subscribed list, check the `CTCoreFolder` for more information. For this particular example I am going to get a list of the messages in the INBOX.

    NSArray *messages = [self.folder messagesFromSequenceNumber:1 to:0 withFetchAttributes:CTFetchAttrEnvelope];

This is one of two ways to fetch messages. The other is by UID:

    NSArray *messages = [self.folder messagesFromUID:1 to:0 withFetchAttributes:CTFetchAttrEnvelope];
    
Both methods do the exact same thing, except one uses sequence numbers to determine which messages to fetch while the other uses UIDs.
    
Both UIDs and sequence numbers start with 1. A 0 as the to: argument indicates that the fetch should go to the end of the message list. A sequence number only identifies a particular message for the duration of the connection. You are  guaranteed that the sequence numbers are consecutive. So if the folder has 100 messages you know the sequence numbers start at 1 and go up to 100. UIDs are guaranteed unique for that folder, even across connections, so they are useful for syncing. They are guaranteed to be increasing, but not consecutive. So if you have a folder with 100 messages the first UID might be 5, followed by 20, then 27, and then 103 to 200.

You'll also note that each method takes fetch attributes, that controls what is retrieved from the server. See the API documentation for more information on available fetch attributes.

Now that we have a message list we can grab the first message and download the message body:

    CTCoreMessage *msg = [messages objectAtIndex:0];
    BOOL isHTML;
    NSString *body = [msg bodyPreferringPlainText:&isHTML];
    
And that's all there is to it.


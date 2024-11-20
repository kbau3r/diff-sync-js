# diff-sync-js

A JavaScript implementation of Neil Fraser Differential Synchronization Algorithm

Diff Sync Writing: https://neil.fraser.name/writing/sync/

## Use Case

Differential synchronization algorithm keep two or more copies of the same document synchronized with each other in real-time. The algorithm offers scalability, fault-tolerance, and responsive collaborative editing across an unreliable network.

## Demo

http://wztechs.com/diff-sync-text-editor/demo/client/

## How to install

When use npm
`npm install diff-sync-js`

When use html
`<script src="./dist/diffSync.js"></script>`

## Dependencies

`json-fast-patch`

## To Test

`npm run test`

## How to use

1. Initial diffSync instance

```
    var diffSync = new DiffSyncAlghorithm({
        jsonpatch: jsonpatch,
        thisVersion: "m",
        senderVersion: "n",
        useBackup: true,
        debug: true
    });
```

2. Initialize container

```
    var container = {};
    diffSync.initObject(container, mainText);
```

3. When Send Payload

```
    diffSync.onSend({
        container,
        mainText,
        whenSend(senderVersion, edits) {
            send({
                type: "PATCH",
                {
                    senderVersion,
                    edits
                }
            });
        }
    });
```

4. When Receive Payload

```
    diffSync.onReceive({
        payload,
        container,
        onUpdateMain(patches, operations) {
            mainText = jsonpatch.applyPatch(mainText, operations).newDocument;
        },
        afterUpdate(senderVersion) {
            send({
                type: "ACK",
                payload: {
                    senderVersion
                }
            });
        }
    });
```

5. When Receive Ack

```
    diffSync.onAck(container, payload);
```

## API

constructor

```
     /**
     * @param {object} options.jsonpatch json-fast-patch library instance (REQUIRED)
     * @param {string} options.thisVersion version tag of the receiving end
     * @param {string} options.senderVersion version tag of the sending end
     * @param {boolean} options.useBackup indicate if use backup copy (DEFAULT true)
     * @param {boolean} options.debug indicate if print out debug message (DEFAULT false)
     */
    constructor({ jsonpatch, thisVersion, senderVersion, useBackup = true, debug = false })

```

initObject

```
    /**
     * Initialize the container
     * @param {object} container any
     * @param {object} mainText any
     */
    initObject(container, mainText)
```

onReceive

```
    /**
     * On Receive Packet
     * @param {object} options.payload payload object that contains {thisVersion, edits}. Edits should be a list of {senderVersion, patch}
     * @param {object} options.container container object {shadow, backup}
     * @param {func} options.onUpdateMain (patches, patchOperations, shadow[thisVersion]) => void
     * @param {func} options.afterUpdate (shadow[senderVersion]) => void
     * @param {func} options.onUpdateShadow (shadow, patch) => newShadowValue
     */
    onReceive({ payload, container, onUpdateMain, afterUpdate, onUpdateShadow })
```

onSend

```
    /**
     * On Sending Packet
     * @param {object} options.container container object {shadow, backup}
     * @param {object} options.mainText any
     * @param {func} options.whenSend (shadow[senderVersion], shadow.edits) => void
     * @param {func} options.whenUnchange (shadow[senderVersion]) => void
     */
    onSend({ container, mainText, whenSend, whenUnchange })

```

onAck

```
     /**
     * Acknowledge the other side when no change were made
     * @param {object} container container object {shadow, backup}
     * @param {object} payload payload object that contains {thisVersion, edits}. Edits should be a list of {senderVersion, patch}
     */
    onAck(container, payload)
```

onAck

```
    /**
     * Acknowledge the other side when no change were made
     * @param container container object {shadow, backup}
     * @param payload payload object that contains {thisVersion, edits}. Edits should be a list of {senderVersion, patch}
     */
    onAck(container, payload)
```

clearOldEdits

```
    /**
     * clear old edits
     * @param {object} shadow
     * @param {string} version
     */
    clearOldEdits(shadow, version)
```

strPatch

```
    /**
     * apply patch to string
     * @param {string} val
     * @param {patch} patch
     * @return string
     */
    strPatch(val, patch)
```

## Additions by us

### How to start the demo application

navigate to the demo/server directory in a terminal and do the following:

`npm install when in the folder directory`
`npm start from the same directory`

then change to the root directory of the repository and from there

`npm run client`

### Changes we made to the demo so it runs effortlessly:

- adding a http-server for the client in the json package file:
  "client": "http-server demo/client"
  as well as the dependecy for it:
  "http-server": "14.1.1"
- Changing to a static path for index.js in the client from:
  "../../src/index.js" -> "diff-sync-js/index.js" with Links
- Changing the socket to "ws://IP:42998" with IP being your IP obviously
- For the server adding the IP to the websocket where before it only included the host
  so going from WebSocket.Server({ port: 42998 }) -> WebSocket.Server({ host: "10.0.107.226", port: 42998 })

### Other (maybe) useful Informations

- Apparently doesnt run on ARM based Linux (Ubuntu 22 in our case) VMs
- json-fast-patch now needs the the addition of "--save" to install it: `npm install fast-json-patch --save`

## Important Questions

1. **Which implementation of the Diff Path Algorithm is being used?**

    The project is using Neil Fraser's Differential Synchronization Algorithm, as for example stated in the diffSync.js file header:

    ```javascript
    /**
    * @author Wei Zheng
    * @github https://github.com/weijie0192/diff-sync-js
     * @summary A JavaScript implementation of Neil Fraser Differential Synchronization Algorithm
    */
    ```

    Additionally you can verify the Diff Patch Alogrithm by looking at the following Characteristics:

    Every patch is acknowledged:
    Each patch sent from one side (client or server) requires an explicit acknowledgment (ACK) from the other side before further patches are sent. This ensures that no patch is lost or applied out of order.

    Evidence in the code:
    ```javascript
    case "ACK":
        diffSync.onAck(container, payload);
        break;
    ```
    Here, the server explicitly processes acknowledgments to confirm that the other side has received the patch.

    Version tracking ensures synchronization:

    Both sides maintain version numbers (thisVersion and senderVersion) to ensure that only valid patches are applied, and older patches are discarded.
    Evidence in the code:
    ```javascript
    if (shadow[thisVersion] !== payload[thisVersion]) {
        if (backup[thisVersion] === payload[thisVersion]) {
            shadow.value = jsonpatch.deepClone(backup.value);
        } else {
            return; // Drop the process
        }
    }
    ```

    If the version numbers don’t match, the patch is discarded, and the system falls back to a backup for recovery.

    Edits are stored until acknowledged:

    Patches (edits) are stored in the shadow’s edits array until an acknowledgment is received from the other side.
    Evidence in the code:

    ```javascript
    shadow.edits.push({
        [thisVersion]: shadow[thisVersion],
        patch
    });
    ```

    Backup for fault tolerance:

    Backups are used to revert to the last known good state in case of failure.
    Evidence in the code:

    ```javascript
    if (useBackup) {
        backup.value = jsonpatch.deepClone(shadow.value);
        backup[thisVersion] = shadow[thisVersion];
    }
    ```
    
    The code matches the Guaranteed Delivery method due to its use of explicit acknowledgments (ACK), version tracking (thisVersion, senderVersion), backups for fault tolerance, and edit stacks to queue patches until acknowledged. These features ensure no patch is lost, synchronization is reliable, and the system remains consistent, confirming it follows the Guaranteed Delivery approach.


2. **Where are the documents and shadows?**

    Server-Side

    Main Document:
    The main document is stored in the mainText variable on the server:

    ```javascript
    var mainText = "";
    ```

    It represents the authoritative copy of the document that is synchronized with all clients.

    Shadow Document:

    Shadows for each client are initialized in the WebSocket connection handler:

    ```javascript
    var container = {};
    diffSync.initObject(container, mainText);
    ```
    Shadows are stored in the container.shadow object and maintain:
    Version tracking: thisVersion and senderVersion.
    Value: A clone of mainText for this client.
    Edit stack: Tracks patches awaiting acknowledgment.



    Client-Side

    Local Document:
    The local document is displayed in the client's text field ($textfield.value) and synchronized with the shadow:

    ```javascript
    setText(payload.shadow.value);
    ```

    Shadow Document:

    The client's shadow is initialized when receiving the JOIN message from the server:
    ```javascript
        diffSync.initObject(container, payload.shadow.value);
    ```
    Shadows are stored in container.shadow and synchronized with the server using patches.

    Initialization of Shadows

    The initObject method in the DiffSyncAlghorithm class is responsible for setting up the shadow documents on both the server and client:

    ```javascript
    initObject(container, mainText) {
        container.shadow = {
            thisVersion: 0,
            senderVersion: 0,
            value: jsonpatch.deepClone(mainText),
            edits: []
        };
        if (this.useBackup) {
            container.backup = {
                thisVersion: 0,
                senderVersion: 0,
                value: jsonpatch.deepClone(mainText)
            };
        }
    }
    ```

3. **How and why can we adjust the sync cycle? What are the disadvantages and advantages?**

    The synchronization cycle can be adjusted by modifying the patch delay on the client side. This is controlled using the range input ($timeoutDelay) in the client UI, which sets the delay between changes being made locally and patches being sent to the server.

    Adjusting Patch Delay:
    ```javascript
    $timeoutDelay.oninput = function () {
        localStorage.setItem("timeoutDelay", this.value);
        $timeoutDelayLabel.innerHTML = "Patch Delay: " + this.value + "ms";
    };
    ```

    Sending Patches After Delay: When a user modifies the document, patches are sent after the specified delay:
    ```javascript
    function processSend(value) {
        clearTimeout(timeout);
        var delay = parseInt($timeoutDelay.value);

        if (delay > 0) {
            timeout = setTimeout(() => {
                sendPatch(value);
            }, delay);
        } else {
            sendPatch(value);
        }
    }
    ```
    The delay is dynamically applied and stored in local storage for persistence.

    Why adjust the sync cycle?
    Purpose:
        To optimize synchronization based on network conditions, resource usage, and application responsiveness.
        Longer delays reduce synchronization frequency, while shorter delays ensure real-time responsiveness.    

    Lower Delay (Faster Sync):
        Advantages:
            Real-time updates ensure immediate consistency across clients.
            Suitable for low-latency environments (e.g., local networks).
        Disadvantages:
            Higher network usage due to frequent patch transmissions.
            Increased resource usage on the server and client for processing frequent updates.

    Higher Delay (Slower Sync):
        Advantages:
            Reduced network traffic and server load.
            More efficient in high-latency environments or with unreliable networks.
        Disadvantages:
            Increases the likelihood of user conflicts (e.g., two users editing the same section before syncing).
            Slower updates may impact collaborative editing experience.

Francesco:


7. **Are the JSON documents interchangeable with other kinds of documents (any kind of documents)?**

No. diff-sync.js is designed specifically for JSON objects, which are structured and hierarchical data formats. Even though you could convert other types of files into a JSON file, this might not be as functional as possible.


---


8. **How is Mr. Wei solving the conflicts?**

Mr. Wei solves conflicts in diff-sync.js by using Neil Fraser's Differential Synchronization Algorithm. In the `onAck` method, he checks if the shadow version is the same as the payload's version and if the backup is different. If both are true, the code logs a message and copies the backup using `jsonpatch.deepClone`. Specifically, the following code snippet ensures that when there's a version mismatch, the backup is restored to keep the data consistent and resolve the conflict.:

```javascript
if (shadow[thisVersion] === payload[thisVersion] && backup[thisVersion] != payload[thisVersion]) {
  this.log("backup not match, clone backup!");
  backup.value = jsonpatch.deepClone(shadow.value);
  backup[thisVersion] = shadow[thisVersion];
}
```



---

9. **What are possible enhancements of Mr Weis Code? Should you suggest a Pull-request**
You can enhance Mr. Wei's diff-sync.js by improving the documentation of the code, also making the README.md clearer with detailed examples and usage instructions. Also, adding customization options would allow users to adapt the synchronization process to their specific needs. It would be smart to use a Pull-Request in this project, because it would increase collaboration and code quality.


---

10. **Is it possible to combine diff-sync.js with a doc-priented nosql-DB such as couchDB and so on**

Yes it is possible to integrate diff-sync.js with a document-oriented NoSQL database like CouchDB. You can use diff-sync.js to handle and synchronize changes to your JSON documents, while CouchDB manages data storage and replication. 
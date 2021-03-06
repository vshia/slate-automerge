# Slate Automerge Example

A example of a collaborative editor using [Slate](https://github.com/ianstormtaylor/slate) and [Automerge](https://github.com/automerge/automerge). This example uses Automerge's CRDT algorithm to synchronize changes between connected clients and works for both on-line and off-line clients. When an off-line client reconnects to the network, it pushes it's own changes and grabs the latest changes, using Automerge's algorithm to merge the changes.

The primary contribution of this repo is the development of the bridge between Slate and Automerge's data structures and the example to demonstrate it's functionality.

## Installation instructions

This installs all dependencies along with the path-from-root branch of Automerge which is also up-to-date with master.
```./install_packages```

To run the development webserver:
```yarn start```

## How to run the example

After starting the development server, go to `http://localhost:3000` where you will see various number of clients.

For each client, you can turn the client on-line/off-line.
At the "server" level, you can turn all clients on/off and view debugging information.

## Code
- `App.js` setups up the initial Slate and Automerge document, instantiates the clients, and acts as the server and network layer between the clients.
- `client.js` is the Slate client.
- `libs` folder contains most of the logic that converts operations between Slate and Automerge. In other words, it's the bridge between the two worlds.

## Behind the scenes

We keep two copies of the document: one in Slate's Immutable data structure (Slate Value) and one in Automerge's data structure (immutable JSON). Aside from the embedded functions in Slate's data structure, the structure and hierarchy of the two should be identical at all times.

Flow of when a change is made on Client A and broadcast to Client B:
1) Change is made to Client A.
2) The onChange function in Client A is fired.
* The Slate Value is stored on the client.
* The Slate Operations are transformed to Automerge JSON operations (in applySlateOperations) and applied to the Automerge document.
* If online, the Automerge changes (calculated by `Automerge.getChanges`) is broadcast to all other clients via `Automerge.Connection`. If offline, when the client comes back online, it syncs all changes also via `Automerge.Connection`.

3) Client B receives an event with the changes.
* Client B's Automerge document applies the changes from Client A.
* The differences between the Client B's new and old Automerge documents are computed (using Automerge.diff).
* The differences are converted to Slate Operations (in `convertAutomergeToSlateOps`) and applied to Client B's Slate Value.

## How to use at the code level:

### On the server:
- To initialize the Automerge document and Connection:
```
    constructor(props) {
        ...
        this.clients = [];
        this.connections = [];
        this.docSet = new Automerge.DocSet();
        this.docSet.setDoc(docId, doc);
        ...
    }
```
This setups up the Automerge document and DocSet.

- When a client connects:
```
    /**
    /**
     * @function connectionHandler
     * @desc Turn a specific client online/offline
     * @param {number} clientId - The Id of the client to turn on/off
     * @param {boolean} isOnline - Turn online/offline
     */
    connectionHandler = (clientId, isOnline) => {
        if (isOnline) {
            if (this.connections[clientId] === undefined || this.connections[clientId] === null) {
                let connection = new Automerge.Connection(
                    this.docSet,
                    (message) => {
                        // TODO: This is a quick hack since the line right below doesn't work.
                        // this.clients[clientId].updateWithRemoteChanges(message);
                        this.clients.forEach((client, idx) => {
                            if (clientId === idx) {
                                client.updateWithRemoteChanges(message);
                            }
                        })
                    }
                )
                this.connections[clientId] = connection;
            }

            this.connections[clientId].open();
        } else {
            if (this.connections[clientId]) {
                this.connections[clientId].close();
                this.connections[clientId] = null;
            }
        }
    }
```
When a client connects, this creates the `Automerge.Connection` for that client and sets up the handler to send it Automerge operations. When a client disconnect, it closes the connection.

- When any document is updated:
```
    /**
     * @function sendMessage
     * @desc Receive a message from one of the clients
     * @param {number} clientId - The server assigned Client Id
     * @param {Object} message - A message created by Automerge.Connection
     */
    sendMessage = (clientId, message) => {
        // Need the setTimeout to give time for each client to update it's own
        // Slate Value via setState
        setTimeout(() => {
            console.debug(`Server received message from Client ${clientId}`)
            this.connections[clientId].receiveMsg(message)
        })
    }
```
The "API" for the clients to call to send Automerge operations.

### On the client:
- To initialize the Automerge document and Connection:
```
    constructor(props) {
        ...
        this.doc = Automerge.init();
        this.docSet = new Automerge.DocSet()
        this.connection = new Automerge.Connection(
            this.docSet,
            (msg) => {
                this.props.sendMessage(this.props.clientId, msg)
            }
        )
        ...
    }
```
and
```
    componentDidMount = () => {
        this.connection.open()
        this.docSet.setDoc(this.props.docId, this.doc)
        this.props.sendMessage(this.props.clientId, {
            docId: this.props.docId,
            clock: Immutable.Map(),
        })
        this.props.connectionHandler(this.props.clientId, true)
    }
```
This sets up the Automerge document and connects to the server, along with the function to send local changes to the server.

- When receiving a remote change:
```
    /**
    * @function updateWithRemoteChanges
    * @desc Update the Automerge document with changes from another client
    * @param {Object} msg - A message created by Automerge.Connection
    */
    updateWithRemoteChanges = (msg) => {
        const currentDoc = this.docSet.getDoc(this.props.docId)
        const docNew = this.connection.receiveMsg(msg)
        const opSetDiff = Automerge.diff(currentDoc, docNew)
        if (opSetDiff.length !== 0) {
            let change = this.state.value.change()
            change = applyAutomergeOperations(opSetDiff, change, () => { this.updateSlateFromAutomerge() });
            if (change) {
                this.setState({ value: change.value })
            }
        }
    }
```
This receives a message from the server regarding new remote changes and applies them to the Automerge document and Slate value.

```
    /**
    * @function updateSlateFromAutomerge
    * @desc Directly update the Slate Value from Automerge, ignoring Slate
    *     operations. This is not preferred when syncing documents since it
    *     causes a re-render and loss of cursor position (and on mobile,
    *     a re-render drops the keyboard).
    */
    updateSlateFromAutomerge = () => {
        const doc = this.docSet.getDoc(this.props.docId)
        const newJson = automergeJsonToSlate({
            "document": { ...doc.note }
        })
        this.setState({ value: Value.fromJSON(newJson) })
    }
```
This is the failure handler when the Automerge -> Slate conversion fails.

- When sending a change:
```
    onChange = ({ operations, value }) => {
        ...
        applySlateOperations(this.docSet, this.props.docId, operations, this.props.clientId)
    }
```
This converts the Slate operation to Automerge operations, applies it to the client's Automerge document, and sends it to the server (via the function initialized in the constructor).

## Things to note:
1) Using the same Client as above, the Slate Operations on Client A will NOT be the same as the transformed Slate Operations on Client B. For example, when splitting a node on Client A (hit [ENTER] in the middle of a sentence), Slate on Client A will issue a `split_node` change operation. On Client B, the operations might be many `remove_text` operations and an `insert_node` operation. This should be fine since we're using the Automerge document as the "ground truth".
2) If Slate crashes due to a bad remote operation, Slate will re-initialize with the latest Automerge document. We don't want to do this too often because it results in a complete re-render of the editor which results in losing the cursor position.

## Known issues:
1) Syncing multiple documents when there are large changes seems to break. This is solved by #2 above.

## Questions / Notes / Optimizations todos
1) Can we compute the output of Automerge.diff (step 3b) from the changes received (in 3)? This would allow us to avoid doing the Automerge.diff.
2) In Automerge, moving a node seems like we're just linking a node from one location to another location. Can we return the path for the new and old location? This will help with identifying the node in Slate.
3) If a new client joins, do they have to initialize the entire Automerge document (with the history)? Or can they just start from the latest snapshot?
4) What's a good way to batch changes from a client? To reduce network traffic, it would be nice to batch keystrokes within a second of each other together.
5) How should we send over information (such as cursor location) which we don't want to persist?
6) Currently does not include support for marks (especially with the Slate 0.34 update)

## Original README below

This project was bootstrapped with [Create React App](https://github.com/facebookincubator/create-react-app). Look in that README for that README file.

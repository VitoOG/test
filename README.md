Problem 1 — Frontend & Web3 Integration Audit

​Utility File and Sources: 
The instantiation of the .Web3 layer is encapsulated inside client/src/utils/contractInstance.js. 
It initializes the connection via new ethers.providers.Web3Provider(window.ethereum). 
The currently exported active smart contract address is 0x4cDBac52C82Bc4d9Edc36a959dCE5EC7e86F0c74. 
The Application Binary Interface (ABI) is imported directly from the adjacent configuration file client/src/utils/Voting.json via the contractABI constant.

​UI Behavior without MetaMask: 
When contractInstance() is invoked and window.ethereum is unavailable, the instantiation of ethers.providers.Web3Provider triggers a runtime reference error.
In a live environment, this uncaught promise rejection stops the asynchronous execution of provider.send(). 
The UI component wrapper isolates this failure via standard React error boundaries or try/catch blocks, triggering a fallback notification to the user and blocking downstream components from receiving the required contract reference.

​Determination of msg.sender: 
The application forces a wallet connection handshake using await provider.send('eth_requestAccounts', []). 
The address acting as msg.sender on the EVM layer is resolved asynchronously by fetching the signer's public key hash via const signerAddress = await signer.getAddress(). 
This explicit signerAddress string is returned alongside the instantiated contract object for state tracking in the React frontend.


​Problem 2 — End-to-End Vote Transaction Trace

​Call Chain: 
The transaction chain begins at the layout layer where client/src/Components/User/ElectionLayout.jsx passes election metadata via <Link info info: state="{{" to="{link}" }}> to the detailed election view component. In the detailed view, the user confirms their candidate selection, firing an event handler that awaits the asynchronous helper function contractInstance(). 
This helper injects the signer into new ethers.Contract(), allowing the application to immediately execute the on-chain mutative function (e.g., vote()) directly on the EVM network.

​Function Arguments & Sources:
​electionId / info: Handed down to the detailed execution path from the routing state (props.location.state.info or useLocation().state.info) initially populated within ElectionLayout.jsx.

​candidateId: 
Captured from the local selection state variable mapped from the candidates prop array exposed during the user interaction phase.

​Verification Token: 
Checked against session data stored in localStorage before the frontend allows the execution flow to reach contractInstance().

​Off-chain Authorization Trust: 
The frontend verifies election properties and metadata through structural props passed into ElectionLayout.jsx. However, the smart contract relies entirely on the incoming transaction signature for state validation. 
If the contract does not implement structural validation matching the backend database, an attacker can extract the contractAddress and contractABI directly from contractInstance.js, bypass the React components entirely, and broadcast arbitrary transactions directly to the RPC endpoint using any valid Web3 provider.



​Problem 3 — FE/BE API Integration Audit

​URL Configuration and Router Mounts: 
The frontend establishes its base API endpoint in client/src/Data/Variables.jsx via the serverLink constant defined as "http://localhost:5000/api/auth/". 
The Express backend mounts the corresponding authentication router in the main entry point using app.use('/api/auth', authRoutes). 
The absolute URL path utilized for voter authentication requests is http://localhost:5000/api/auth/login.

​Data Retrieval Call Chain: 
Parent Dashboard Component \rightarrow HTTP GET request to /api/auth/ (via axios) \rightarrow Express route handler router.get('/', fetchElections) \rightarrow Mongoose Model query \rightarrow Passed as info object props down into ElectionLayout.jsx for rendering.

​Voter Authentication Payload and Lifecycle: 
The authentication request payload transmits email and password values from the UI input fields. 
On failure, the Express server responds with a 401 Unauthorized status code. On success (200 OK), the API yields a JSON object containing a signed JSON Web Token (JWT). 
The client interceptor caches this token into localStorage before invoking the core voting methods.

​API Security Flaw: 
The authorization architecture relies heavily on client-side state enforcement. 
If the backend fails to validate that the user's active wallet address (signerAddress retrieved in contractInstance()) strictly matches the authenticated database profile linked to the active JWT token, the system is vulnerable to identity spoofing. 

An attacker could authenticate via a valid email/password combination, but sign the transaction using a completely separate unauthorized blockchain wallet address.
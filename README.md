<html lang="en" class="h-full">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Global Chat Website</title>
    <!-- Load Tailwind CSS -->
    <script src="https://cdn.tailwindcss.com"></script>
    <style>
        /* Custom font */
        @import url('https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700&display=swap');
        body {
            font-family: 'Inter', sans-serif;
        }
        /* Custom scrollbar for chat */
        #chat-messages::-webkit-scrollbar {
            width: 6px;
        }
        #chat-messages::-webkit-scrollbar-track {
            background: #4b5563; /* gray-600 */
        }
        #chat-messages::-webkit-scrollbar-thumb {
            background: #1f2937; /* gray-800 */
            border-radius: 3px;
        }
    </style>
</head>
<body class="bg-gray-900 text-white flex flex-col min-h-screen">

    <!-- Header -->
    <header class="bg-gray-800 shadow-md sticky top-0 z-10">
        <nav class="container mx-auto px-4 py-4 flex justify-between items-center">
            <h1 class="text-2xl font-bold text-white">My Website</h1>
            <div class="space-x-4">
                <a href="#" class="text-gray-300 hover:text-white">Home</a>
                <a href="#" class="text-gray-300 hover:text-white">About</a>
                <a href="#" class="text-gray-300 hover:text-white">Contact</a>
            </div>
        </nav>
    </header>

    <!-- Main Content -->
    <main class="flex-grow container mx-auto px-4 py-8">
        <div class="lg:grid lg:grid-cols-3 lg:gap-8">
            
            <!-- Left Column: Website Content -->
            <div class="lg:col-span-2">
                <h2 class="text-3xl font-bold mb-4">Welcome to Our Community Page!</h2>
                <p class="text-gray-300 mb-6">
                    This is a simple example of a basic webpage. Below this content, you'll find our global chat.
                    Feel free to say hello to anyone else who might be visiting the page at the same time.
                </p>
                
                <div class="bg-gray-800 rounded-lg shadow-lg p-6 mb-6">
                    <h3 class="text-xl font-semibold mb-3">What is this?</h3>
                    <p class="text-gray-400">
                        This demonstrates how a real-time application, like a chat, can be embedded within a
                        standard website. The chat module on the right is powered by Firebase Firestore,
                        allowing for persistent, shared conversations.
                    </p>
                </div>

                <div class="bg-gray-800 rounded-lg shadow-lg p-6">
                    <h3 class="text-xl font-semibold mb-3">Your User ID</h3>
                    <p class="text-gray-400 mb-2">
                        You are automatically signed in anonymously. Your unique user ID is displayed in the chat header.
                    </p>
                </div>
            </div>

            <!-- Right Column: Embedded Chat -->
            <div class="lg:col-span-1 mt-8 lg:mt-0">
                <div class="bg-gray-800 rounded-lg shadow-xl flex flex-col h-[80vh] max-h-[700px]">
                    <!-- Chat Header -->
                    <div class="p-4 border-b border-gray-700">
                        <h3 class="text-lg font-semibold text-center">Global Chat</h3>
                        <p class="text-xs text-gray-400 text-center break-all">
                            Your ID: <span id="user-id" class="font-mono">Loading...</span>
                        </p>
                    </div>
                    
                    <!-- Chat Messages -->
                    <div id="chat-messages" class="p-4 flex-grow overflow-y-auto space-y-4">
                        <!-- Messages will be injected here -->
                        <div class="text-center text-gray-500">Loading messages...</div>
                    </div>
                    
                    <!-- Chat Input -->
                    <div class="p-4 border-t border-gray-700">
                        <form id="chat-form" class="flex space-x-2">
                            <input id="chat-input" type="text" placeholder="Loading chat..."
                                class="bg-gray-700 border border-gray-600 rounded-md flex-grow p-2 text-white focus:outline-none focus:ring-2 focus:ring-blue-500"
                                autocomplete="off"
                                required
                                disabled>
                            <button type="submit"
                                class="bg-blue-600 hover:bg-blue-700 text-white rounded-md p-2 px-4 transition-colors">
                                <!-- Send Icon SVG -->
                                <svg xmlns="http://www.w3.org/2000/svg" class="h-5 w-5" viewBox="0 0 20 20" fill="currentColor">
                                    <path d="M10.894 2.553a1 1 0 00-1.788 0l-7 14a1 1 0 001.169 1.409l5-1.429A1 1 0 009.586 16.5l-4.242-4.243a1 1 0 011.414-1.414l4.243 4.242a1 1 0 001.414-.001l5-1.428a1 1 0 001.17-1.408l-7-14z" />
                                </svg>
                            </button>
                        </form>
                    </div>
                </div>
            </div>
        </div>
    </main>

    <!-- Firebase App -->
    <script type="module">
        // Import Firebase modules
        import { initializeApp } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-app.js";
        import { getAuth, signInAnonymously, onAuthStateChanged } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-auth.js";
        import { getFirestore, collection, onSnapshot, addDoc, setLogLevel } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-firestore.js";

        //
        // 🚨 ATTENTION! 🚨
        // YOU MUST REPLACE THESE PLACEHOLDER VALUES WITH YOUR ACTUAL FIREBASE CONFIG
        //
        const firebaseConfig = {
            apiKey: "PASTE_YOUR_API_KEY_HERE",
            authDomain: "PASTE_YOUR_AUTH_DOMAIN_HERE",
            projectId: "PASTE_YOUR_PROJECT_ID_HERE",
            storageBucket: "PASTE_YOUR_STORAGE_BUCKET_HERE",
            messagingSenderId: "PASTE_YOUR_SENDER_ID_HERE",
            appId: "PASTE_YOUR_APP_ID_HERE"
        };
        // --------------------------------------------------------------------
        
        
        // --- DOM Elements ---
        const userIdElement = document.getElementById('user-id');
        const messagesContainer = document.getElementById('chat-messages');
        const chatForm = document.getElementById('chat-form');
        const chatInput = document.getElementById('chat-input');
        
        let db, auth;
        let currentUserId = null;
        let messagesCol;

        /**
         * Renders the list of messages to the chat window
         */
        function renderMessages(messages, currentUserId) {
            messagesContainer.innerHTML = ''; 
            
            if (messages.length === 0) {
                messagesContainer.innerHTML = '<div class="text-center text-gray-500">No messages yet. Say hello!</div>';
                return;
            }

            messages.forEach(msg => {
                const isOwnMessage = msg.userId === currentUserId;
                
                const wrapper = document.createElement('div');
                const alignClass = isOwnMessage ? 'text-right' : 'text-left';
                const bubbleClass = isOwnMessage ? 'bg-blue-600' : 'bg-gray-700';
                const userDisplay = isOwnMessage ? 'You' : msg.userId;

                wrapper.className = `message-item w-full ${alignClass}`;

                // User ID span
                const userSpan = document.createElement('span');
                userSpan.className = 'text-xs text-gray-500 px-1 font-mono';
                userSpan.textContent = userDisplay;
                
                // Message bubble
                const bubble = document.createElement('div');
                bubble.className = `${bubbleClass} rounded-lg p-3 inline-block max-w-[90%] text-left`;

                // Message text
                const p = document.createElement('p');
                p.className = 'break-words';
                p.textContent = msg.text;

                bubble.appendChild(p);
                wrapper.appendChild(userSpan);
                wrapper.appendChild(bubble);
                messagesContainer.appendChild(wrapper);
            });

            // Auto-scroll to bottom
            messagesContainer.scrollTop = messagesContainer.scrollHeight;
        }

        /**
         * Sets up the real-time listener for chat messages
         */
        function setupMessageListener(currentUserId) {
            // Static collection path for GitHub Pages
            const messagesColPath = "global-chat";
            messagesCol = collection(db, messagesColPath);

            // Listen for new messages
            onSnapshot(messagesCol, (snapshot) => {
                const messages = [];
                snapshot.forEach(doc => {
                    messages.push({ id: doc.id, ...doc.data() });
                });

                // Sort messages by timestamp in JavaScript
                messages.sort((a, b) => a.timestamp - b.timestamp);

                renderMessages(messages, currentUserId);
            });
        }

        /**
         * Handles the form submission for sending a new message
         */
        async function handleSendMessage(e) {
            e.preventDefault();
            const text = chatInput.value.trim();

            if (text && currentUserId && messagesCol) {
                try {
                    await addDoc(messagesCol, {
                        text: text,
                        userId: currentUserId,
                        timestamp: Date.now()
                    });
                    chatInput.value = ''; 
                } catch (error) {
                    console.error("Error sending message: ", error);
                }
            }
        }

        /**
         * Main function to initialize the app
         */
        async function main() {
            if (!firebaseConfig.apiKey || firebaseConfig.apiKey.includes("PASTE_YOUR_API_KEY")) {
                messagesContainer.innerHTML = '<div class="text-center text-red-500 p-4"><b>Error: Firebase is not configured.</b><br><br>Please replace the placeholder values in the <code>firebaseConfig</code> object.</div>';
                userIdElement.textContent = "Error";
                return;
            }

            try {
                // Initialize Firebase
                const app = initializeApp(firebaseConfig);
                db = getFirestore(app);
                auth = getAuth(app);
                
                setLogLevel('Debug');

                // --- Authentication ---
                onAuthStateChanged(auth, (user) => {
                    if (user) {
                        currentUserId = user.uid;
                        userIdElement.textContent = currentUserId;
                        
                        setupMessageListener(currentUserId);
                        
                        chatForm.addEventListener('submit', handleSendMessage);
                        chatInput.disabled = false;
                        chatInput.placeholder = "Type your message...";
                        
                    } else {
                        currentUserId = null;
                        userIdElement.textContent = "Not signed in";
                        chatInput.disabled = true;
                        chatInput.placeholder = "Signing in...";
                    }
                });

                // --- Sign In ---
                await signInAnonymously(auth);

            } catch (error) {
                console.error("Firebase Initialization Error:", error);
                messagesContainer.innerHTML = `<div class="text-center text-red-500 p-4"><b>Error:</b> ${error.message}<br><br>Check your console and make sure your Firestore Security Rules are correct.</div>`;
                userIdElement.textContent = "Error";
            }
        }

        // Run the app
        main();

    </script>
</body>
</html>

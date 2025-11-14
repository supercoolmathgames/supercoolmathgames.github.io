<html lang="en" class="h-full">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Local Chat Website (No Database)</title>
    <!-- Load Tailwind CSS for styling -->
    <script src="https://cdn.tailwindcss.com"></script>
    <style>
        /* Custom font and scrollbar styles */
        @import url('https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700&display=swap');
        body {
            font-family: 'Inter', sans-serif;
        }
        #chat-messages::-webkit-scrollbar { width: 6px; }
        #chat-messages::-webkit-scrollbar-track { background: #4b5563; }
        #chat-messages::-webkit-scrollbar-thumb { background: #1f2937; border-radius: 3px; }
    </style>
</head>
<body class="bg-gray-900 text-white flex flex-col min-h-screen">

    <!-- Header -->
    <header class="bg-gray-800 shadow-md sticky top-0 z-10">
        <nav class="container mx-auto px-4 py-4 flex justify-between items-center">
            <h1 class="text-2xl font-bold text-white">My Local Chat Site</h1>
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
                <h2 class="text-3xl font-bold mb-4">Welcome to Your Personal Chat Sandbox!</h2>
                <p class="text-gray-300 mb-6">
                    This chat is purely local. It uses your browser's **localStorage** to save messages, so they won't disappear when you refresh or close the tab, but they **will not be shared** with other people.
                </p>
                
                <div class="bg-gray-800 rounded-lg shadow-lg p-6 mb-6">
                    <h3 class="text-xl font-semibold mb-3">No Server, No Problem!</h3>
                    <p class="text-gray-400">
                        This entire application runs on the client side (in your browser) without needing a database or server connection, making it perfect for GitHub Pages.
                    </p>
                </div>

                <div class="bg-gray-800 rounded-lg shadow-lg p-6">
                    <h3 class="text-xl font-semibold mb-3">Your Persistent ID</h3>
                    <p class="text-gray-400 mb-2">
                        A unique ID was generated for this browser session and is saved locally. This ensures your identity is consistent every time you visit this page.
                    </p>
                </div>
            </div>

            <!-- Right Column: Embedded Chat -->
            <div class="lg:col-span-1 mt-8 lg:mt-0">
                <div class="bg-gray-800 rounded-lg shadow-xl flex flex-col h-[80vh] max-h-[700px]">
                    <!-- Chat Header -->
                    <div class="p-4 border-b border-gray-700">
                        <h3 class="text-lg font-semibold text-center">Local Chat History</h3>
                        <p class="text-xs text-gray-400 text-center break-all">
                            Your ID: <span id="user-id" class="font-mono text-blue-300"></span>
                        </p>
                    </div>
                    
                    <!-- Chat Messages -->
                    <div id="chat-messages" class="p-4 flex-grow overflow-y-auto space-y-4">
                        <div class="text-center text-gray-500">History will load here...</div>
                    </div>
                    
                    <!-- Chat Input -->
                    <div class="p-4 border-t border-gray-700">
                        <form id="chat-form" class="flex space-x-2">
                            <input id="chat-input" type="text" placeholder="Type your message..."
                                class="bg-gray-700 border border-gray-600 rounded-md flex-grow p-2 text-white focus:outline-none focus:ring-2 focus:ring-blue-500"
                                autocomplete="off"
                                required>
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

    <!-- JavaScript Logic -->
    <script>
        document.addEventListener('DOMContentLoaded', () => {
            const userIdElement = document.getElementById('user-id');
            const messagesContainer = document.getElementById('chat-messages');
            const chatForm = document.getElementById('chat-form');
            const chatInput = document.getElementById('chat-input');

            const LOCAL_STORAGE_KEY = 'local_chat_messages';
            const USER_ID_KEY = 'local_chat_user_id';
            let currentUserId;

            // --- 1. User ID Setup ---
            function getOrCreateUserId() {
                let id = localStorage.getItem(USER_ID_KEY);
                if (!id) {
                    // Generate a simple, unique ID for the session (last 8 chars of a UUID)
                    id = Math.random().toString(36).substring(2, 10);
                    localStorage.setItem(USER_ID_KEY, id);
                }
                return id;
            }

            currentUserId = getOrCreateUserId();
            userIdElement.textContent = currentUserId;

            // --- 2. Message Renderer ---
            function renderMessages(messages) {
                messagesContainer.innerHTML = '';
                
                if (messages.length === 0) {
                    messagesContainer.innerHTML = '<div class="text-center text-gray-500">Start the conversation! Your history is saved locally.</div>';
                    return;
                }

                messages.forEach(msg => {
                    const isOwnMessage = msg.userId === currentUserId;
                    
                    const wrapper = document.createElement('div');
                    const alignClass = isOwnMessage ? 'text-right' : 'text-left';
                    const bubbleClass = isOwnMessage ? 'bg-blue-600' : 'bg-gray-700';
                    const userDisplay = isOwnMessage ? 'You' : msg.userId;

                    wrapper.className = `message-item w-full ${alignClass}`;

                    const userSpan = document.createElement('span');
                    userSpan.className = 'text-xs text-gray-500 px-1 font-mono';
                    userSpan.textContent = userDisplay;
                    
                    const bubble = document.createElement('div');
                    bubble.className = `${bubbleClass} rounded-lg p-3 inline-block max-w-[90%] text-left`;

                    const p = document.createElement('p');
                    p.className = 'break-words';
                    p.textContent = msg.text; // Use textContent for safety

                    bubble.appendChild(p);
                    wrapper.appendChild(userSpan);
                    wrapper.appendChild(bubble);
                    messagesContainer.appendChild(wrapper);
                });

                // Auto-scroll to bottom
                messagesContainer.scrollTop = messagesContainer.scrollHeight;
            }

            // --- 3. Persistence Logic ---
            function loadMessages() {
                const stored = localStorage.getItem(LOCAL_STORAGE_KEY);
                return stored ? JSON.parse(stored) : [];
            }

            function saveMessages(messages) {
                localStorage.setItem(LOCAL_STORAGE_KEY, JSON.stringify(messages));
                renderMessages(messages);
            }

            // --- 4. Event Handler ---
            function handleSendMessage(e) {
                e.preventDefault();
                const text = chatInput.value.trim();

                if (text) {
                    const messages = loadMessages();
                    messages.push({
                        text: text,
                        userId: currentUserId,
                        timestamp: Date.now()
                    });
                    
                    saveMessages(messages);
                    chatInput.value = ''; 
                }
            }

            // --- 5. Initialization ---
            chatForm.addEventListener('submit', handleSendMessage);
            
            // Load and display existing messages
            const initialMessages = loadMessages();
            renderMessages(initialMessages);
        });
    </script>
</body>
</html>

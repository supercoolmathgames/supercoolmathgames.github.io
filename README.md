<script src="https://cdn.tailwindcss.com"></script>
    <style>
        /* Custom font and scrollbar styles */
        @import url('https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700&display=swap');
        body {
            font-family: 'Inter', sans-serif;
        }
        /* Style for the chat box scrollbar */
        #chat-messages::-webkit-scrollbar { width: 6px; }
        #chat-messages::-webkit-scrollbar-track { background: #374151; /* gray-700 */ }
        #chat-messages::-webkit-scrollbar-thumb { background: #111827; /* gray-900 */ border-radius: 3px; }

        /* Fancy gradient background */
        .fancy-bg {
            background-image: linear-gradient(to bottom right, #1f2937, #374151); /* from-gray-800 to-gray-700 */
        }
        /* Fancy animation for sync message */
        .loading-pulse {
          animation: pulse 1.5s cubic-bezier(0.4, 0, 0.6, 1) infinite;
        }
        @keyframes pulse {
          0%, 100% { opacity: 1; }
          50% { opacity: .4; }
        }
    </style>

    <div class="bg-gradient-to-br from-gray-900 to-indigo-900 text-white flex flex-col min-h-screen">
        <header class="bg-gray-800 shadow-xl sticky top-0 z-10">
            <nav class="container mx-auto px-4 py-4 flex justify-between items-center">
                <h1 class="text-2xl font-bold text-white">SheetChat Global Feed</h1>
                <span class="text-sm text-gray-400">Powered by Google Sheets API</span>
            </nav>
        </header>

        <main class="flex-grow container mx-auto px-4 py-8">
            <div class="lg:grid lg:grid-cols-3 lg:gap-8">
                
                <div class="lg:col-span-2">
                    <h2 class="text-3xl font-bold mb-4">Welcome to the Sheet-Backed Chat!</h2>
                    <p class="text-gray-300 mb-6">
                        This chat achieves global sharing without traditional backend services by using a Google Sheet as a database. Data is synchronized every five seconds.
                    </p>
                    
                    <div class="bg-gray-800 rounded-lg shadow-2xl p-6 mb-6">
                        <h3 class="text-xl font-semibold mb-3">Technical Details</h3>
                        <p class="text-gray-400">
                            The browser polls the Sheet API endpoint every 5000 milliseconds to check for new messages, ensuring cross-device, shared communication.
                        </p>
                    </div>
                </div>

                <div class="lg:col-span-1 mt-8 lg:mt-0">
                    <div class="bg-gray-800 rounded-lg shadow-2xl flex flex-col h-[80vh] max-h-[700px]">
                        <div class="p-4 border-b border-gray-700">
                            <h3 class="text-lg font-semibold text-center">Global Chat Feed</h3>
                            <p class="text-xs text-gray-400 text-center break-all">
                                Your ID: <span id="user-id" class="font-mono text-blue-300"></span>
                            </p>
                        </div>
                        
                        <div id="chat-messages" class="p-4 flex-grow overflow-y-auto space-y-4">
                            <div class="text-center text-gray-500">Connecting to Google Sheet...</div>
                        </div>
                        
                        <div class="p-4 border-t border-gray-700">
                            <form id="chat-form" class="flex space-x-2">
                                <input id="chat-input" type="text" placeholder="Syncing history..."
                                    class="bg-gray-700 border border-gray-600 rounded-md flex-grow p-2 text-white focus:outline-none focus:ring-4 focus:ring-blue-400 transition duration-300"
                                    autocomplete="off"
                                    required
                                    disabled>
                                <button type="submit"
                                    class="bg-gradient-to-r from-blue-600 to-cyan-500 hover:from-blue-700 hover:to-cyan-600 text-white rounded-md p-2 px-4 transition-all duration-300 disabled:from-gray-500 disabled:to-gray-600"
                                    id="send-button"
                                    disabled>
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

        <script>
            document.addEventListener('DOMContentLoaded', () => {
                // YOUR SPECIFIC GOOGLE SHEET API ENDPOINT
                const SHEET_API_URL = "https://sheetdb.io/api/v1/9ua95v1wnorcy"; 
                const USER_ID_KEY = 'sheet_chat_user_id';
                const POLLING_INTERVAL = 5000; // Poll every 5 seconds for stability

                const userIdElement = document.getElementById('user-id');
                const messagesContainer = document.getElementById('chat-messages');
                const chatForm = document.getElementById('chat-form');
                const chatInput = document.getElementById('chat-input');
                const sendButton = document.getElementById('send-button');

                let currentUserId;

                // --- 1. User ID Setup ---
                function getOrCreateUserId() {
                    let id = localStorage.getItem(USER_ID_KEY);
                    if (!id) {
                        id = 'User-' + Math.random().toString(36).substring(2, 8);
                        localStorage.setItem(USER_ID_KEY, id);
                    }
                    return id;
                }

                currentUserId = getOrCreateUserId();
                userIdElement.textContent = currentUserId;

                // --- 2. Utility Functions ---
                function formatDate(timestamp) {
                    const date = new Date(timestamp);
                    return date.toLocaleTimeString([], { hour: '2-digit', minute: '2-digit' });
                }

                function setChatEnabled(isEnabled) {
                    chatInput.disabled = !isEnabled;
                    sendButton.disabled = !isEnabled;
                    chatInput.placeholder = isEnabled ? "Type your message..." : "Syncing history...";
                    if (isEnabled) {
                         chatInput.classList.remove('loading-pulse');
                    } else {
                         chatInput.classList.add('loading-pulse');
                    }
                }

                // --- 3. Message Renderer ---
                function renderMessages(messages) {
                    messagesContainer.innerHTML = '';
                    
                    if (messages.length === 0) {
                        messagesContainer.innerHTML = '<div class="text-center text-gray-500">Be the first to say something!</div>';
                        return;
                    }

                    messages.forEach(msg => {
                        // CRITICAL: Keys must match sheet headers (USER_ID, MESSAGE, TIMESTAMP)
                        const isOwnMessage = msg.USER_ID === currentUserId;
                        
                        const wrapper = document.createElement('div');
                        const alignClass = isOwnMessage ? 'text-right' : 'text-left';
                        
                        // Fancy Gradients for bubbles
                        const bubbleClass = isOwnMessage 
                            ? 'bg-gradient-to-r from-blue-600 to-cyan-500 text-white' 
                            : 'bg-gray-700 text-gray-200';
                            
                        const userDisplay = isOwnMessage ? 'You' : msg.USER_ID;
                        const timestampDisplay = msg.TIMESTAMP ? formatDate(msg.TIMESTAMP) : 'Sending...';

                        wrapper.className = `message-item w-full ${alignClass}`;

                        // User Info and Time
                        const infoSpan = document.createElement('span');
                        infoSpan.className = 'text-xs text-gray-500 px-1 font-mono';
                        infoSpan.innerHTML = `<strong>${userDisplay}</strong> - ${timestampDisplay}`;
                        
                        // Message bubble
                        const bubble = document.createElement('div');
                        bubble.className = `${bubbleClass} rounded-2xl p-3 inline-block max-w-[90%] text-left mt-1 shadow-md`;

                        const p = document.createElement('p');
                        p.className = 'break-words';
                        p.textContent = msg.MESSAGE;

                        bubble.appendChild(p);
                        wrapper.appendChild(infoSpan);
                        wrapper.appendChild(bubble);
                        messagesContainer.appendChild(wrapper);
                    });

                    messagesContainer.scrollTop = messagesContainer.scrollHeight;
                }

                // --- 4. Fetching Logic (Polling) ---
                async function fetchMessages() {
                    try {
                        const response = await fetch(SHEET_API_URL, {
                            method: 'GET',
                            headers: { 'Cache-Control': 'no-cache' }
                        });
                        
                        if (!response.ok) {
                            throw new Error(`HTTP Error: ${response.status}`);
                        }
                        
                        const messages = await response.json();
                        
                        // Convert TIMESTAMP to a number for reliable sorting
                        messages.forEach(m => {
                            m.TIMESTAMP = parseInt(String(m.TIMESTAMP)) || 0;
                        });
                        
                        messages.sort((a, b) => a.TIMESTAMP - b.TIMESTAMP);

                        renderMessages(messages);
                        setChatEnabled(true);

                    } catch (error) {
                        console.error("Error fetching messages (API connection issue):", error);
                        messagesContainer.innerHTML = `<div class="text-center text-red-500 p-4">Could not connect to Sheet API. Check console for CORS/network errors.</div>`;
                        setChatEnabled(false);
                    }
                }

                // --- 5. Posting Logic ---
                async function handleSendMessage(e) {
                    e.preventDefault();
                    const text = chatInput.value.trim();
                    
                    if (text && !chatInput.disabled) {
                        const timestamp = Date.now();
                        
                        setChatEnabled(false);
                        chatInput.value = '';

                        try {
                            const response = await fetch(SHEET_API_URL, {
                                method: 'POST',
                                headers: {
                                    'Content-Type': 'application/json',
                                },
                                body: JSON.stringify({
                                    data: {
                                        USER_ID: currentUserId,
                                        MESSAGE: text,
                                        TIMESTAMP: timestamp 
                                    }
                                })
                            });

                            if (!response.ok) {
                                throw new Error(`Post failed: ${response.status}`);
                            }
                        } catch (error) {
                            console.error("Error posting message:", error);
                            alert("Failed to send message. Please check your Google Sheet configuration.");
                        } finally {
                            setChatEnabled(true); 
                        }
                    }
                }

                // --- 6. Initialization ---
                chatForm.addEventListener('submit', handleSendMessage);
                
                // Start initial fetch and then set up the polling interval
                fetchMessages(); 
                setInterval(fetchMessages, POLLING_INTERVAL);
            });
        </script>
    </div>

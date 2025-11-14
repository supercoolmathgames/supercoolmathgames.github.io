<!DOCTYPE html>
<html lang="en" class="h-full">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Global Sheet-Backed Chat</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <style>
        /* Custom font and scrollbar styles */
        @import url('https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700&display=swap');
        body {
            font-family: 'Inter', sans-serif;
        }
        /* Style for the chat box scrollbar */
        #chat-messages::-webkit-scrollbar { width: 6px; }
        #chat-messages::-webkit-scrollbar-track { background: #4b5563; }
        #chat-messages::-webkit-scrollbar-thumb { background: #1f2937; border-radius: 3px; }
    </style>
</head>
<body class="bg-gray-900 text-white flex flex-col min-h-screen">

    <header class="bg-gray-800 shadow-md sticky top-0 z-10">
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
                    This chat achieves global sharing without traditional backend services by using a Google Sheet as a database. Data is synchronized every two seconds.
                </p>
                
                <div class="bg-gray-800 rounded-lg shadow-lg p-6 mb-6">
                    <h3 class="text-xl font-semibold mb-3">Technical Details</h3>
                    <p class="text-gray-400">
                        The browser polls the Sheet API endpoint every 2000 milliseconds to check for new messages. This ensures the chat is updated across all users' devices.
                    </p>
                </div>
            </div>

            <div class="lg:col-span-1 mt-8 lg:mt-0">
                <div class="bg-gray-800 rounded-lg shadow-xl flex flex-col h-[80vh] max-h-[700px]">
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
                            <input id="chat-input" type="text" placeholder="Loading history..."
                                class="bg-gray-700 border border-gray-600 rounded-md flex-grow p-2 text-white focus:outline-none focus:ring-2 focus:ring-blue-500"
                                autocomplete="off"
                                required
                                disabled>
                            <button type="submit"
                                class="bg-blue-600 hover:bg-blue-700 text-white rounded-md p-2 px-4 transition-colors disabled:bg-gray-500"
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
            // Your specific Google Sheet API endpoint is hardcoded here
            const SHEET_API_URL = "https://sheetdb.io/api/v1/9ua95v1wnorcy"; 
            const USER_ID_KEY = 'sheet_chat_user_id';
            const POLLING_INTERVAL = 2000; // Poll every 2 seconds

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
            }

            // --- 3. Message Renderer ---
            function renderMessages(messages) {
                messagesContainer.innerHTML = '';
                
                if (messages.length === 0) {
                    messagesContainer.innerHTML = '<div class="text-center text-gray-500">Be the first to say something!</div>';
                    return;
                }

                messages.forEach(msg => {
                    const isOwnMessage = msg.USER_ID === currentUserId;
                    
                    const wrapper = document.createElement('div');
                    const alignClass = isOwnMessage ? 'text-right' : 'text-left';
                    const bubbleClass = isOwnMessage ? 'bg-blue-600 text-white' : 'bg-gray-700 text-gray-200';
                    const userDisplay = isOwnMessage ? 'You' : msg.USER_ID;
                    const timestampDisplay = msg.TIMESTAMP ? formatDate(msg.TIMESTAMP) : 'Sending...';

                    wrapper.className = `message-item w-full ${alignClass}`;

                    // User Info and Time
                    const infoSpan = document.createElement('span');
                    infoSpan.className = 'text-xs text-gray-500 px-1 font-mono';
                    infoSpan.innerHTML = `<strong>${userDisplay}</strong> - ${timestampDisplay}`;
                    
                    // Message bubble
                    const bubble = document.createElement('div');
                    bubble.className = `${bubbleClass} rounded-lg p-3 inline-block max-w-[90%] text-left mt-1`;

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
                        // Add cache control to ensure we always get the latest data
                        headers: { 'Cache-Control': 'no-cache' }
                    });
                    
                    if (!response.ok) {
                        throw new Error(`HTTP Error: ${response.status}`);
                    }
                    
                    const messages = await response.json();
                    
                    // The API often returns strings; ensure TIMESTAMP is a number for sorting
                    messages.forEach(m => {
                        m.TIMESTAMP = parseInt(m.TIMESTAMP) || 0;
                    });
                    
                    // Sort messages by timestamp (oldest first)
                    messages.sort((a, b) => a.TIMESTAMP - b.TIMESTAMP);

                    renderMessages(messages);
                    setChatEnabled(true);

                } catch (error) {
                    console.error("Error fetching messages:", error);
                    messagesContainer.innerHTML = `<div class="text-center text-red-500 p-4">Error connecting to Sheet API. Check console for details.</div>`;
                    setChatEnabled(false);
                }
            }

            // --- 5. Posting Logic ---
            async function handleSendMessage(e) {
                e.preventDefault();
                const text = chatInput.value.trim();
                
                if (!text || !chatInput.disabled) {
                    const timestamp = Date.now();
                    
                    // Temporarily disable chat to prevent double-posting
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
                                    TIMESTAMP: timestamp // Send as number
                                }
                            })
                        });

                        if (!response.ok) {
                            throw new Error(`Post failed: ${response.status}`);
                        }
                        
                        // Success! Polling will fetch the new message shortly.

                    } catch (error) {
                        console.error("Error posting message:", error);
                        alert("Failed to send message. Please check the console.");
                    } finally {
                        // Re-enable chat form, polling will update the view
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
</body>
</html>

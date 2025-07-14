<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>My Daily Routine Tracker</title>
    <!-- Tailwind CSS CDN for styling -->
    <script src="https://cdn.tailwindcss.com"></script>
    <style>
        /* Custom scrollbar for better aesthetics */
        ::-webkit-scrollbar {
            width: 8px;
        }
        ::-webkit-scrollbar-track {
            background: #f1f1f1;
            border-radius: 10px;
        }
        ::-webkit-scrollbar-thumb {
            background: #888;
            border-radius: 10px;
        }
        ::-webkit-scrollbar-thumb:hover {
            background: #555;
        }
        /* Inter font from Google Fonts */
        body {
            font-family: 'Inter', sans-serif;
        }
    </style>
</head>
<body class="bg-gradient-to-br from-blue-100 to-purple-100 min-h-screen flex items-center justify-center py-10 px-4">
    <div class="bg-white p-8 rounded-2xl shadow-xl max-w-4xl w-full flex flex-col lg:flex-row gap-8">
        <!-- Timetable Display Section -->
        <div class="lg:w-1/2 p-6 bg-blue-50 rounded-xl shadow-inner overflow-y-auto max-h-[80vh]">
            <h2 class="text-3xl font-bold text-blue-800 mb-6 text-center">Your Planned Daily Routine</h2>
            <div id="planned-timetable" class="space-y-4 text-gray-700">
                <!-- Timetable content will be injected here by JavaScript -->
            </div>
            <p class="mt-6 text-lg font-semibold text-blue-700 text-center">Total Dedicated Study Hours: Approximately 10 hours and 45 minutes.</p>
            <div class="mt-6 p-4 bg-blue-100 rounded-lg text-blue-800 text-sm">
                <h3 class="font-bold mb-2">Important Notes:</h3>
                <ul class="list-disc list-inside space-y-1">
                    <li>**Flexibility:** This is a template. Feel free to adjust the specific times or activities based on your energy levels, coaching classes, or specific study needs on any given day.</li>
                    <li>**Breaks are Essential:** The short breaks and longer recreational slots are just as important as study time for preventing burnout and maintaining focus.</li>
                    <li>**Listen to Your Body:** If you're feeling particularly tired, a short nap or a longer break might be more beneficial than pushing through.</li>
                    <li>**Consistency:** Try to stick to the general structure as much as possible, especially for sleep and wake times, to regulate your body clock.</li>
                </ul>
            </div>
        </div>

        <!-- Daily Log Input & Reflection Section -->
        <div class="lg:w-1/2 p-6 bg-purple-50 rounded-xl shadow-inner flex flex-col">
            <h2 class="text-3xl font-bold text-purple-800 mb-6 text-center">Log Your Day & Reflect</h2>
            <div id="loading-indicator" class="text-center text-purple-600 mb-4 hidden">
                <div class="animate-spin rounded-full h-8 w-8 border-b-2 border-purple-500 inline-block"></div>
                <p class="mt-2">Loading...</p>
            </div>
            <p id="auth-status" class="text-center text-sm text-gray-500 mb-4"></p>
            <p id="user-id-display" class="text-center text-sm text-gray-500 mb-4"></p>

            <form id="daily-log-form" class="space-y-4 flex-grow flex flex-col">
                <label for="log-date" class="block text-lg font-medium text-gray-700">Date:</label>
                <input type="date" id="log-date" class="p-3 border border-gray-300 rounded-lg focus:ring-purple-500 focus:border-purple-500 shadow-sm transition duration-200" required>

                <div id="log-inputs" class="space-y-4 overflow-y-auto max-h-[40vh] pr-2">
                    <!-- Dynamic log input fields will go here -->
                </div>

                <button type="submit" class="mt-6 w-full bg-purple-600 hover:bg-purple-700 text-white font-bold py-3 px-6 rounded-xl shadow-lg transition duration-300 ease-in-out transform hover:scale-105 focus:outline-none focus:ring-2 focus:ring-purple-500 focus:ring-opacity-75">
                    Save Daily Log
                </button>
            </form>

            <h3 class="text-2xl font-bold text-purple-800 mt-8 mb-4 text-center">Past Activities</h3>
            <div id="past-logs" class="space-y-4 overflow-y-auto max-h-[30vh] p-2 bg-purple-50 rounded-lg border border-purple-200">
                <p class="text-gray-500 text-center">No past logs found. Save your first log!</p>
            </div>
             <div id="message-box" class="fixed inset-0 bg-gray-600 bg-opacity-50 flex items-center justify-center hidden">
                <div class="bg-white p-6 rounded-lg shadow-xl text-center">
                    <p id="message-text" class="text-lg font-semibold text-gray-800 mb-4"></p>
                    <button id="close-message" class="bg-purple-600 hover:bg-purple-700 text-white font-bold py-2 px-4 rounded-lg transition duration-300">OK</button>
                </div>
            </div>
        </div>
    </div>

    <!-- Firebase SDKs -->
    <script type="module">
        import { initializeApp } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-app.js";
        import { getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-auth.js";
        import { getFirestore, collection, addDoc, query, orderBy, onSnapshot, serverTimestamp } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-firestore.js";

        // Global Firebase variables
        let app;
        let db;
        let auth;
        let userId = 'anonymous'; // Default to anonymous

        // Get app ID and Firebase config from global variables provided by Canvas
        const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';
        const firebaseConfig = typeof __firebase_config !== 'undefined' ? JSON.parse(__firebase_config) : {};
        const initialAuthToken = typeof __initial_auth_token !== 'undefined' ? __initial_auth_token : null;

        const loadingIndicator = document.getElementById('loading-indicator');
        const authStatusDisplay = document.getElementById('auth-status');
        const userIdDisplay = document.getElementById('user-id-display');
        const messageBox = document.getElementById('message-box');
        const messageText = document.getElementById('message-text');
        const closeMessageButton = document.getElementById('close-message');

        // Function to show custom message box
        function showMessage(message) {
            messageText.textContent = message;
            messageBox.classList.remove('hidden');
        }

        // Close message box
        closeMessageButton.addEventListener('click', () => {
            messageBox.classList.add('hidden');
        });

        // Initialize Firebase and authenticate user
        async function initializeFirebaseAndAuth() {
            loadingIndicator.classList.remove('hidden');
            try {
                app = initializeApp(firebaseConfig);
                db = getFirestore(app);
                auth = getAuth(app);

                // Listen for authentication state changes
                onAuthStateChanged(auth, async (user) => {
                    if (user) {
                        userId = user.uid;
                        authStatusDisplay.textContent = 'Authenticated.';
                        userIdDisplay.textContent = `User ID: ${userId}`;
                        console.log("Firebase initialized and user authenticated:", userId);
                        // Once authenticated, load past logs
                        loadPastLogs();
                    } else {
                        // If no user, try to sign in with custom token or anonymously
                        if (initialAuthToken) {
                            try {
                                await signInWithCustomToken(auth, initialAuthToken);
                                authStatusDisplay.textContent = 'Authenticated with custom token.';
                            } catch (error) {
                                console.error("Error signing in with custom token:", error);
                                authStatusDisplay.textContent = 'Error with custom token. Signing in anonymously.';
                                await signInAnonymously(auth);
                            }
                        } else {
                            await signInAnonymously(auth);
                            authStatusDisplay.textContent = 'Signed in anonymously.';
                        }
                    }
                    loadingIndicator.classList.add('hidden');
                });

            } catch (error) {
                console.error("Error initializing Firebase or authentication:", error);
                showMessage("Failed to initialize the application. Please try again later.");
                loadingIndicator.classList.add('hidden');
            }
        }

        // Define the planned timetable structure
        const plannedTimetableData = [
            { time: "6:00 AM - 6:30 AM", activity: "Wake Up & Refresh" },
            { time: "6:30 AM - 7:00 AM", activity: "Light Exercise / Stretching" },
            { time: "7:00 AM - 9:00 AM", activity: "Study Session 1 (High-Focus Subject)" },
            { time: "9:00 AM - 9:30 AM", activity: "Breakfast" },
            { time: "9:30 AM - 11:30 AM", activity: "Study Session 2" },
            { time: "11:30 AM - 11:45 AM", activity: "Short Break" },
            { time: "11:45 AM - 1:30 PM", activity: "Study Session 3" },
            { time: "1:30 PM - 2:30 PM", activity: "Lunch & Relax" },
            { time: "2:30 PM - 4:30 PM", activity: "Study Session 4" },
            { time: "4:30 PM - 5:00 PM", activity: "Active Break (Walk / Quick Game)" },
            { time: "5:00 PM - 7:00 PM", activity: "Study Session 5" },
            { time: "7:00 PM - 8:00 PM", activity: "Downtime / Family Time" },
            { time: "8:00 PM - 9:00 PM", activity: "Dinner" },
            { time: "9:00 PM - 10:00 PM", activity: "Revision / Light Study / Self-Assessment" },
            { time: "10:00 PM - 10:45 PM", activity: "Wind Down & Enjoy (Manga / Gaming)" },
            { time: "10:45 PM - 11:00 PM", activity: "Get Ready for Bed" },
            { time: "11:00 PM", activity: "Sleep" }
        ];

        // Function to render the planned timetable and create log input fields
        function renderTimetableAndLogInputs() {
            const plannedTimetableDiv = document.getElementById('planned-timetable');
            const logInputsDiv = document.getElementById('log-inputs');
            plannedTimetableDiv.innerHTML = '';
            logInputsDiv.innerHTML = '';

            plannedTimetableData.forEach((item, index) => {
                // Planned timetable display
                const plannedItem = document.createElement('div');
                plannedItem.className = 'p-3 bg-white rounded-lg shadow-sm';
                plannedItem.innerHTML = `
                    <p class="font-semibold text-blue-700">${item.time}</p>
                    <p class="text-gray-800">${item.activity}</p>
                `;
                plannedTimetableDiv.appendChild(plannedItem);

                // Log input fields
                const logInputItem = document.createElement('div');
                logInputItem.className = 'flex flex-col';
                logInputItem.innerHTML = `
                    <label for="activity-${index}" class="block text-sm font-medium text-gray-700 mb-1">${item.time}:</label>
                    <textarea id="activity-${index}" name="activity-${index}" rows="2" class="p-2 border border-gray-300 rounded-lg focus:ring-purple-500 focus:border-purple-500 shadow-sm transition duration-200" placeholder="What did you do?"></textarea>
                `;
                logInputsDiv.appendChild(logInputItem);
            });

            // Set today's date as default for the log date input
            const today = new Date();
            const yyyy = today.getFullYear();
            const mm = String(today.getMonth() + 1).padStart(2, '0'); // Months are 0-indexed
            const dd = String(today.getDate()).padStart(2, '0');
            document.getElementById('log-date').value = `${yyyy}-${mm}-${dd}`;
        }

        // Function to save daily log to Firestore
        async function saveDailyLog(event) {
            event.preventDefault(); // Prevent default form submission

            if (!userId || userId === 'anonymous') {
                showMessage("Authentication not ready. Please wait a moment or refresh.");
                return;
            }

            const logDate = document.getElementById('log-date').value;
            if (!logDate) {
                showMessage("Please select a date for your log.");
                return;
            }

            const dailyActivities = {};
            plannedTimetableData.forEach((item, index) => {
                const textarea = document.getElementById(`activity-${index}`);
                dailyActivities[item.time.replace(/[^a-zA-Z0-9]/g, '')] = textarea.value.trim(); // Use cleaned time as key
            });

            try {
                // Use the user's private collection path
                const logsCollectionRef = collection(db, `artifacts/${appId}/users/${userId}/daily_logs`);
                await addDoc(logsCollectionRef, {
                    date: logDate,
                    activities: dailyActivities,
                    timestamp: serverTimestamp() // Add a server timestamp
                });
                showMessage("Daily log saved successfully!");
                document.getElementById('daily-log-form').reset(); // Clear the form
                renderTimetableAndLogInputs(); // Re-render to clear textareas
            } catch (e) {
                console.error("Error adding document: ", e);
                showMessage("Error saving log. Please try again.");
            }
        }

        // Function to load and display past logs from Firestore
        function loadPastLogs() {
            if (!userId || userId === 'anonymous') {
                console.log("User not authenticated yet, cannot load past logs.");
                return;
            }

            const pastLogsDiv = document.getElementById('past-logs');
            pastLogsDiv.innerHTML = '<p class="text-gray-500 text-center">Loading past logs...</p>';

            // Use the user's private collection path
            const logsCollectionRef = collection(db, `artifacts/${appId}/users/${userId}/daily_logs`);
            const q = query(logsCollectionRef, orderBy("timestamp", "desc")); // Order by timestamp descending

            onSnapshot(q, (snapshot) => {
                pastLogsDiv.innerHTML = ''; // Clear previous logs
                if (snapshot.empty) {
                    pastLogsDiv.innerHTML = '<p class="text-gray-500 text-center">No past logs found. Save your first log!</p>';
                    return;
                }

                snapshot.forEach((doc) => {
                    const log = doc.data();
                    const logDate = log.date;
                    const activities = log.activities;

                    const logEntry = document.createElement('div');
                    logEntry.className = 'p-4 bg-white rounded-lg shadow-sm border border-purple-100';
                    let activitiesHtml = `<h4 class="font-bold text-purple-700 mb-2">Log for ${logDate}</h4><ul class="list-disc list-inside text-gray-700 text-sm space-y-1">`;

                    // Iterate through plannedTimetableData to ensure order and display original time format
                    plannedTimetableData.forEach(item => {
                        const cleanedTimeKey = item.time.replace(/[^a-zA-Z0-9]/g, '');
                        const loggedActivity = activities[cleanedTimeKey] || 'No entry';
                        activitiesHtml += `<li><span class="font-semibold">${item.time}:</span> ${loggedActivity}</li>`;
                    });

                    activitiesHtml += '</ul>';
                    logEntry.innerHTML = activitiesHtml;
                    pastLogsDiv.appendChild(logEntry);
                });
            }, (error) => {
                console.error("Error fetching past logs:", error);
                pastLogsDiv.innerHTML = '<p class="text-red-500 text-center">Error loading logs.</p>';
            });
        }

        // Event listener for form submission
        document.getElementById('daily-log-form').addEventListener('submit', saveDailyLog);

        // Initialize on window load
        window.onload = function() {
            renderTimetableAndLogInputs();
            initializeFirebaseAndAuth();
        };
    </script>
</body>
</html>

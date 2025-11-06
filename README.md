<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>SMRK To-Do List</title>
    <!-- Load Tailwind CSS -->
    <script src="https://cdn.tailwindcss.com"></script>
    <!-- Inter Font Setup -->
    <style>
        @import url('https://fonts.googleapis.com/css2?family=Inter:wght@100..900&display=swap');
        body {
            font-family: 'Inter', sans-serif;
            background-color: #f0f4f8;
        }
        /* Custom scrollbar for better mobile feel */
        ::-webkit-scrollbar { width: 4px; }
        ::-webkit-scrollbar-track { background: #f0f4f8; }
        ::-webkit-scrollbar-thumb { background: #cbd5e1; border-radius: 2px; }
        ::-webkit-scrollbar-thumb:hover { background: #94a3b8; }
    </style>
    <!-- Firebase Imports -->
    <script type="module">
        import { initializeApp } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-app.js";
        import { getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-auth.js";
        import { getFirestore, doc, addDoc, updateDoc, deleteDoc, onSnapshot, collection, query, serverTimestamp } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-firestore.js";
        import { setLogLevel } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-firestore.js";

        // IMPORTANT: Global variables are provided by the Canvas environment
        const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';
        const firebaseConfig = typeof __firebase_config !== 'undefined' ? JSON.parse(__firebase_config) : null;
        const initialAuthToken = typeof __initial_auth_token !== 'undefined' ? __initial_auth_token : null;

        let db;
        let auth;
        let userId = null;
        let isAuthReady = false;
        
        // State Management
        window.currentView = 'private'; // 'private' or 'public'
        window.editingTaskId = null; // Stores the ID of the task being edited

        // Utility to generate a temporary user ID if not authenticated
        const getTemporaryUserId = () => {
            return 'anon-' + (crypto.randomUUID ? crypto.randomUUID() : Math.random().toString(36).substring(2, 9));
        };

        // --- 1. INITIALIZE FIREBASE AND AUTHENTICATE ---
        const initFirebase = async () => {
            try {
                if (!firebaseConfig) {
                    throw new Error("Firebase config not found.");
                }

                const app = initializeApp(firebaseConfig);
                db = getFirestore(app);
                auth = getAuth(app);
                setLogLevel('Debug');

                if (initialAuthToken) {
                    await signInWithCustomToken(auth, initialAuthToken);
                } else {
                    await signInAnonymously(auth);
                }

                onAuthStateChanged(auth, (user) => {
                    if (user) {
                        userId = user.uid;
                        document.getElementById('user-id-display').textContent = `User ID: ${userId}`;
                    } else {
                        userId = getTemporaryUserId();
                        document.getElementById('user-id-display').textContent = `User ID: (Anon) ${userId.substring(0, 8)}...`;
                    }
                    isAuthReady = true;
                    console.log("Firebase Auth Ready. User ID:", userId);
                    // Start listening to data
                    setupRealtimeListener();
                });

            } catch (error) {
                console.error("Firebase initialization or authentication failed:", error);
                document.getElementById('loading-state').textContent = 'Error: Failed to connect to database.';
                userId = getTemporaryUserId();
                isAuthReady = true;
                document.getElementById('user-id-display').textContent = `User ID: (Error/Anon) ${userId.substring(0, 8)}...`;
            }
        };

        // --- 2. FIRESTORE OPERATIONS ---

        // Get the collection reference based on the current view
        const getCollectionRef = (view) => {
            if (!db || !userId) throw new Error("Database or User ID not initialized.");
            
            if (view === 'private') {
                // Private Path: /artifacts/{appId}/users/{userId}/todos_private
                return collection(db, `artifacts/${appId}/users/${userId}/todos_private`);
            } else if (view === 'public') {
                // Public Path: /artifacts/{appId}/public/data/todos_public
                return collection(db, `artifacts/${appId}/public/data/todos_public`);
            }
            throw new Error("Invalid view mode.");
        };

        // Switch between private and public view
        window.switchView = (view) => {
            if (window.currentView === view) return;
            window.currentView = view;
            window.editingTaskId = null; // Exit editing mode when switching
            
            // Update button styles
            document.getElementById('private-btn').classList.toggle('bg-blue-600', view === 'private');
            document.getElementById('private-btn').classList.toggle('bg-gray-200', view !== 'private');
            document.getElementById('private-btn').classList.toggle('text-white', view === 'private');
            document.getElementById('private-btn').classList.toggle('text-gray-700', view !== 'private');

            document.getElementById('public-btn').classList.toggle('bg-blue-600', view === 'public');
            document.getElementById('public-btn').classList.toggle('bg-gray-200', view !== 'public');
            document.getElementById('public-btn').classList.toggle('text-white', view === 'public');
            document.getElementById('public-btn').classList.toggle('text-gray-700', view !== 'public');

            document.getElementById('input-section-title').textContent = 
                view === 'private' ? 'Add Private Task' : 'Add Public Task (Visible to All)';

            setupRealtimeListener(); // Re-attach listener for the new view
        };

        // Add a new To-Do item (saves to currentView collection)
        window.addTask = async () => {
            const inputElement = document.getElementById('new-task-input');
            const priorityElement = document.getElementById('task-priority');
            const taskText = inputElement.value.trim();
            const taskPriority = priorityElement.value;

            if (!taskText) return;
            if (!isAuthReady || !userId) {
                console.warn("Auth not ready. Cannot add task.");
                return;
            }

            try {
                await addDoc(getCollectionRef(window.currentView), {
                    text: taskText,
                    priority: taskPriority,
                    completed: false,
                    ownerId: userId, // Store the user ID for sharing/editing permissions
                    timestamp: serverTimestamp()
                });
                inputElement.value = '';
                priorityElement.value = 'Medium';
            } catch (error) {
                console.error("Error adding document: ", error);
            }
        };

        // Toggle task completion status
        window.toggleTask = async (docId, currentStatus) => {
            if (!isAuthReady) return;
            try {
                // Get the correct path for the document
                const docRef = doc(db, getCollectionRef(window.currentView).path, docId);
                await updateDoc(docRef, {
                    completed: !currentStatus
                });
            } catch (error) {
                console.error("Error toggling document status: ", error);
            }
        };
        
        // Enter Editing Mode
        window.startEdit = (docId, isEditable) => {
            if (!isEditable) {
                console.log("Cannot edit a task that is not yours in public view.");
                return;
            }
            // If already editing, save it first (optional, but good UX)
            if (window.editingTaskId && window.editingTaskId !== docId) {
                window.editingTaskId = docId;
                // Force re-render to show input field
                setupRealtimeListener(); 
                return;
            }
            window.editingTaskId = docId;
            // Force re-render to show input field
            setupRealtimeListener(); 
        };

        // Save Edited Task
        window.saveEdit = async (docId, newText) => {
            if (!isAuthReady || !newText.trim()) return;
            
            try {
                const docRef = doc(db, getCollectionRef(window.currentView).path, docId);
                await updateDoc(docRef, {
                    text: newText.trim()
                });
                window.editingTaskId = null; // Exit editing mode
                // No need for re-render, onSnapshot will handle it.
            } catch (error) {
                console.error("Error updating document text: ", error);
            }
        };

        // Cancel Editing
        window.cancelEdit = () => {
            window.editingTaskId = null;
            setupRealtimeListener(); // Re-render to show static text again
        };

        // Delete a task
        window.deleteTask = async (docId) => {
            if (!isAuthReady) return;
            try {
                const docRef = doc(db, getCollectionRef(window.currentView).path, docId);
                await deleteDoc(docRef);
            } catch (error) {
                console.error("Error deleting document: ", error);
            }
        };

        // Get Tailwind classes for priority badge
        const getPriorityClasses = (priority) => {
            switch (priority) {
                case 'High':
                    return 'bg-red-100 text-red-800 border-red-400';
                case 'Medium':
                    return 'bg-amber-100 text-amber-800 border-amber-400';
                case 'Low':
                default:
                    return 'bg-green-100 text-green-800 border-green-400';
            }
        };

        // Render the list of tasks to the UI
        const renderTasks = (tasks) => {
            const listContainer = document.getElementById('todo-list');
            listContainer.innerHTML = '';
            document.getElementById('loading-state').classList.add('hidden');
            
            const priorityMap = { 'High': 3, 'Medium': 2, 'Low': 1 };

            if (tasks.length === 0) {
                listContainer.innerHTML = `
                    <div class="text-center py-12 text-gray-500">
                        <svg xmlns="http://www.w3.org/2000/svg" class="h-10 w-10 mx-auto mb-2" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                            <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M9 5H7a2 2 0 00-2 2v12a2 2 0 002 2h10a2 2 0 002-2V7a2 2 0 00-2-2h-2M9 5a2 2 0 002 2h2a2 2 0 002-2M9 5a2 2 0 002 2h2M9 5h6" />
                        </svg>
                        <p class="text-lg font-semibold">No Tasks Yet</p>
                        <p class="text-sm">Add a new task below!</p>
                    </div>
                `;
                return;
            }

            // Sort tasks: 1. Incomplete first, 2. Priority (High first), 3. Timestamp (latest first)
            tasks.sort((a, b) => {
                if (a.completed !== b.completed) {
                    return a.completed ? 1 : -1;
                }
                const priorityA = priorityMap[a.priority] || 1;
                const priorityB = priorityMap[b.priority] || 1;
                if (priorityA !== priorityB) {
                    return priorityB - priorityA;
                }
                const timeA = a.timestamp?.toMillis() || 0;
                const timeB = b.timestamp?.toMillis() || 0;
                return timeB - timeA;
            });

            tasks.forEach(task => {
                const isEditable = (task.ownerId === userId) || (window.currentView === 'private');
                const isBeingEdited = window.editingTaskId === task.id;
                
                const taskElement = document.createElement('div');
                const completedClass = task.completed ? 'opacity-60 bg-gray-50 border-gray-300' : 'bg-white shadow-md border-blue-500';
                const textClass = task.completed ? 'line-through text-gray-500' : 'text-gray-800';
                const priorityClasses = getPriorityClasses(task.priority);

                taskElement.className = `flex items-start justify-between p-4 mb-3 rounded-xl border-l-4 transition duration-300 ease-in-out ${completedClass}`;

                let taskContentHTML = '';
                
                if (isBeingEdited) {
                    // Editing Mode UI
                    taskContentHTML = `
                        <div class="flex-1 min-w-0">
                            <textarea id="edit-input-${task.id}" class="w-full text-base font-medium p-2 border border-blue-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500 mb-2">${task.text}</textarea>
                            <div class="flex space-x-2 mt-1">
                                <button onclick="saveEdit('${task.id}', document.getElementById('edit-input-${task.id}').value)"
                                    class="text-sm bg-green-500 hover:bg-green-600 text-white px-3 py-1 rounded-md transition duration-150">
                                    Save
                                </button>
                                <button onclick="cancelEdit()"
                                    class="text-sm bg-gray-300 hover:bg-gray-400 text-gray-800 px-3 py-1 rounded-md transition duration-150">
                                    Cancel
                                </button>
                            </div>
                        </div>
                    `;
                } else {
                    // Display Mode UI
                    taskContentHTML = `
                        <div class="flex-1 min-w-0">
                            <span class="text-base break-words min-w-0 font-medium ${textClass} ${isEditable ? 'cursor-pointer hover:underline' : ''}" 
                                onclick="startEdit('${task.id}', ${isEditable})">
                                ${task.text}
                            </span>
                            
                            <!-- Priority & Owner Info -->
                            <div class="mt-1 flex items-center space-x-3 text-xs">
                                <span class="inline-block font-semibold px-2 py-0.5 rounded-full border ${priorityClasses}">
                                    ${task.priority || 'Low'}
                                </span>
                                ${window.currentView === 'public' ? `
                                    <span class="text-gray-500">
                                        Owner: ${task.ownerId === userId ? 'You' : task.ownerId.substring(0, 8) + '...'}
                                    </span>
                                ` : ''}
                            </div>
                        </div>
                    `;
                }
                
                // Construct the full item HTML
                taskElement.innerHTML = `
                    <div class="flex items-start flex-1 min-w-0 mr-4">
                        <!-- Checkbox -->
                        <button onclick="toggleTask('${task.id}', ${task.completed})"
                            class="flex-shrink-0 mt-1 w-6 h-6 rounded-full border-2 transition-all duration-200 focus:outline-none 
                                ${task.completed ? 'bg-green-500 border-green-500' : 'border-gray-300 hover:bg-gray-100'}"
                            aria-label="${task.completed ? 'Mark as incomplete' : 'Mark as complete'}"
                            ${isBeingEdited ? 'disabled' : ''}>
                            ${task.completed ? '<svg class="w-4 h-4 text-white mx-auto" fill="none" viewBox="0 0 24 24" stroke="currentColor"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="3" d="M5 13l4 4L19 7" /></svg>' : ''}
                        </button>
                        
                        <div class="ml-4 flex-1 min-w-0">
                            ${taskContentHTML}
                        </div>
                    </div>

                    <!-- Delete Button (Only visible if editable and not currently editing) -->
                    ${isEditable && !isBeingEdited ? `
                    <button onclick="deleteTask('${task.id}')"
                        class="text-red-400 hover:text-red-600 p-2 rounded-full transition duration-150 focus:outline-none hover:bg-red-50 flex-shrink-0"
                        aria-label="Delete task">
                        <svg class="w-5 h-5" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                            <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M19 7l-.867 12.142A2 2 0 0116.138 21H7.862a2 2 0 01-1.995-1.858L5 7m5 4v6m4-6v6m1-10V4a1 1 0 00-1-1h-4a1 1 0 00-1 1v3M4 7h16" />
                        </svg>
                    </button>
                    ` : ''}
                `;
                listContainer.appendChild(taskElement);
            });
        };

        let unsubscribeListener = null;

        // Setup real-time listener
        const setupRealtimeListener = () => {
            if (!isAuthReady || !db) {
                console.warn("Database not ready for listener setup.");
                return;
            }

            // Unsubscribe from previous listener if one exists
            if (unsubscribeListener) {
                unsubscribeListener();
                unsubscribeListener = null;
            }
            
            document.getElementById('loading-state').classList.remove('hidden');

            try {
                const q = query(getCollectionRef(window.currentView));
                
                // onSnapshot listens for real-time changes
                unsubscribeListener = onSnapshot(q, (snapshot) => {
                    const tasks = [];
                    snapshot.forEach((doc) => {
                        tasks.push({ id: doc.id, ...doc.data() });
                    });
                    console.log(`Real-time update received for ${window.currentView}. Tasks count:`, tasks.length);
                    renderTasks(tasks);
                }, (error) => {
                    console.error("Error setting up real-time listener:", error);
                    document.getElementById('loading-state').textContent = 'Error: Could not load tasks.';
                });

            } catch (error) {
                console.error("Error setting up Firestore query:", error);
            }
        };

        // Start the application setup
        window.onload = () => {
            initFirebase();
            switchView('private'); // Initialize view to Private
        }
    </script>
</head>
<body class="min-h-screen p-4 sm:p-6">

    <div class="max-w-xl mx-auto">
        <!-- Header -->
        <header class="text-center mb-6">
            <h1 class="text-3xl font-extrabold text-gray-900 leading-tight">
                SMRK To-Do List
            </h1>
            <p id="user-id-display" class="text-xs mt-1 text-gray-500 font-mono">Loading User ID...</p>
        </header>
        
        <!-- View Switcher (Public/Private) -->
        <div class="flex justify-center mb-6 bg-white p-1 rounded-xl shadow-inner">
            <button id="private-btn" onclick="switchView('private')"
                class="flex-1 font-semibold py-2 px-4 rounded-lg
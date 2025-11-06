# Smrk-to-do-list-
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Smrk</title>
    <!-- Tailwind CSS CDN -->
    <script src="https://cdn.tailwindcss.com"></script>
    <!-- Inter Font -->
    <link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;600;700&display=swap" rel="stylesheet">
    <!-- Firebase SDKs -->
    <script type="module">
        import { initializeApp } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-app.js";
        import { getAuth, signInAnonymously, signInWithCustomToken } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-auth.js";
        import { getFirestore, collection, query, orderBy, onSnapshot, addDoc, serverTimestamp, deleteDoc, doc, updateDoc, getDoc } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-firestore.js";
        import { setLogLevel } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-firestore.js";

        setLogLevel('Debug');

        let db, auth, userId;
        const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';
        const firebaseConfig = typeof __firebase_config !== 'undefined' ? JSON.parse(__firebase_config) : {};
        
        let currentSortOrder = 'newest'; // 'newest' or 'likes'

        // 1. Firebase and Authentication Setup
        try {
            const app = initializeApp(firebaseConfig);
            db = getFirestore(app);
            auth = getAuth(app);

            // Use Custom Token or Anonymous Sign-in
            if (typeof __initial_auth_token !== 'undefined' && __initial_auth_token) {
                signInWithCustomToken(auth, __initial_auth_token)
                    .then((userCredential) => {
                        userId = userCredential.user.uid;
                        document.getElementById('user-info').textContent = `User ID: ${userId}`;
                        loadPhotos();
                        console.log("Custom token sign-in successful.");
                    })
                    .catch((error) => {
                        console.error("Custom token sign-in failed. Falling back to anonymous.", error);
                        signInAnonymously(auth).then(userCredential => {
                            userId = userCredential.user.uid;
                            document.getElementById('user-info').textContent = `User ID: ${userId}`;
                            loadPhotos();
                        });
                    });
            } else {
                signInAnonymously(auth).then(userCredential => {
                    userId = userCredential.user.uid;
                    document.getElementById('user-info').textContent = `User ID: ${userId}`;
                    loadPhotos();
                }).catch(error => {
                    console.error("Anonymous sign-in failed:", error);
                    loadPhotos();
                });
            }

        } catch (error) {
            console.error("Firebase Initialization Error:", error);
            showMessage('Sorry, there was an issue initializing Firebase. Check the console.', 'bg-red-500');
        }
        
        // Public Collection Path
        const PHOTOS_COLLECTION_PATH = `artifacts/${appId}/public/data/photos`;

        // 2. Helper function to show messages
        function showMessage(text, colorClass) {
            const messageBox = document.getElementById('message-box');
            messageBox.textContent = text;
            messageBox.className = `p-3 mb-4 rounded-lg text-white font-medium ${colorClass} transition-all duration-300`;
            setTimeout(() => {
                messageBox.className = 'p-0 mb-0 rounded-lg text-white font-medium transition-all duration-300 h-0 overflow-hidden';
            }, 5000);
        }

        // 3. Convert file to Base64 (for small files)
        function fileToBase64(file) {
            return new Promise((resolve, reject) => {
                if (!file) {
                    reject(new Error("No file selected."));
                    return;
                }
                // Check file size (Firestore limit is 1MB, keeping buffer lower for safety)
                const MAX_FILE_SIZE = 300 * 1024; // 300 KB
                if (file.size > MAX_FILE_SIZE) {
                    reject(new Error(`File size (${(file.size / 1024).toFixed(0)} KB) exceeds the safe limit (300 KB) for direct storage. Please use a smaller image or URL.`));
                    return;
                }

                const reader = new FileReader();
                reader.onload = () => resolve(reader.result);
                reader.onerror = error => reject(error);
                reader.readAsDataURL(file);
            });
        }


        // 4. Save Photo to Firestore (Updated to handle File Input and optional Caption)
        window.savePhoto = async function() {
            const fileInput = document.getElementById('photo-file');
            const urlInput = document.getElementById('photo-url');
            const captionInput = document.getElementById('photo-caption');
            
            const caption = captionInput.value.trim();
            const file = fileInput.files[0];
            const url = urlInput.value.trim();

            let finalUrl = '';
            let isBase64 = false;

            if (file) {
                // Handle file upload (Base64)
                if (url) {
                    showMessage('Please upload a file OR use a URL, not both.', 'bg-yellow-500');
                    return;
                }
                
                try {
                    finalUrl = await fileToBase64(file);
                    isBase64 = true;
                } catch (error) {
                    console.error("Base64 Conversion Error:", error);
                    showMessage(`Image error: ${error.message}`, 'bg-red-500');
                    return;
                }

            } else if (url) {
                // Handle URL input
                finalUrl = url;
                isBase64 = false;
            } else {
                // No file and no URL
                showMessage('Please select an image file or provide an image URL.', 'bg-yellow-500');
                return;
            }

            if (!db) {
                showMessage('Database is not available.', 'bg-red-500');
                return;
            }
            
            const saveBtn = document.getElementById('save-button');
            saveBtn.disabled = true;
            saveBtn.textContent = 'Saving...';

            try {
                await addDoc(collection(db, PHOTOS_COLLECTION_PATH), {
                    url: finalUrl, 
                    caption: caption, // Caption is now optional (can be empty string)
                    timestamp: serverTimestamp(),
                    uploaderId: userId || 'anonymous',
                    likes: {}, 
                    comments: [],
                    isBase64: isBase64 
                });

                showMessage('Photo successfully added to the gallery!', 'bg-green-500');
                fileInput.value = ''; 
                urlInput.value = '';
                captionInput.value = '';

            } catch (error) {
                console.error("Error adding document: ", error);
                showMessage('Error saving photo: ' + error.message, 'bg-red-500');
            } finally {
                saveBtn.disabled = false;
                saveBtn.textContent = 'Add Photo';
            }
        }
        
        // 5. Edit Photo - Show Modal/Form
        window.startEditPhoto = function(docId, currentCaption, currentUrl, isBase64) {
            const modal = document.getElementById('edit-modal');
            document.getElementById('edit-doc-id').value = docId;
            document.getElementById('edit-caption').value = currentCaption;
            
            // Hide URL field for Base64, show error message instead
            const urlGroup = document.getElementById('edit-url-group');
            const base64Message = document.getElementById('edit-base64-message');
            
            if (isBase64) {
                urlGroup.classList.add('hidden');
                base64Message.classList.remove('hidden');
                // For base64, only caption can be edited.
            } else {
                urlGroup.classList.remove('hidden');
                base64Message.classList.add('hidden');
                // Set URL value 
                document.getElementById('edit-url').value = currentUrl;
            }

            modal.classList.remove('hidden');
        }

        // 6. Close Edit Modal
        window.closeEditModal = function() {
            document.getElementById('edit-modal').classList.add('hidden');
        }

        // 7. Update Photo in Firestore (Updated for optional Caption)
        window.updatePhoto = async function() {
            const docId = document.getElementById('edit-doc-id').value;
            const newCaption = document.getElementById('edit-caption').value.trim(); // Can be empty
            const photoRef = doc(db, PHOTOS_COLLECTION_PATH, docId);
            
            if (!db) {
                showMessage('Database is not available.', 'bg-red-500');
                return;
            }
            
            const docSnap = await getDoc(photoRef);
            const photoData = docSnap.data();
            
            // If it's Base64, we don't allow URL change here. Otherwise, get from input.
            const newUrl = photoData.isBase64 ? photoData.url : document.getElementById('edit-url').value.trim();
            
            if (!newUrl) {
                showMessage('URL/Image data cannot be empty.', 'bg-yellow-500');
                return;
            }
            
            const updateBtn = document.getElementById('update-button');
            updateBtn.disabled = true;
            updateBtn.textContent = 'Updating...';

            try {
                // Only allow uploader to update
                if (!docSnap.exists() || photoData.uploaderId !== userId) {
                    showMessage('You do not have permission to edit this photo.', 'bg-red-500');
                    return;
                }
                
                await updateDoc(photoRef, {
                    caption: newCaption, // Caption can be empty string
                    url: newUrl 
                });

                showMessage('Photo successfully updated!', 'bg-green-500');
                closeEditModal();

            } catch (error) {
                console.error("Error updating document: ", error);
                showMessage('Error updating photo: ' + error.message, 'bg-red-500');
            } finally {
                updateBtn.disabled = false;
                updateBtn.textContent = 'Save Changes';
            }
        }
        
        // 8. Toggle Like (Same as before)
        window.toggleLike = async function(docId) {
            if (!db || !userId) {
                showMessage('Database is not available for liking.', 'bg-red-500');
                return;
            }
            
            const likeButton = document.querySelector(`button[onclick="event.stopPropagation(); toggleLike('${docId}')"]`);
            if (likeButton) likeButton.disabled = true;

            try {
                const photoRef = doc(db, PHOTOS_COLLECTION_PATH, docId);
                const docSnap = await getDoc(photoRef); 

                if (!docSnap.exists()) {
                    showMessage('Photo not found.', 'bg-red-500');
                    return;
                }

                const photoData = docSnap.data();
                const currentLikes = photoData.likes || {};
                const isLiked = !!currentLikes[userId];
                
                const newLikes = { ...currentLikes };

                if (isLiked) {
                    delete newLikes[userId];
                    showMessage('Like removed!', 'bg-yellow-500');
                } else {
                    newLikes[userId] = true;
                    showMessage('Photo liked!', 'bg-green-500');
                }

                await updateDoc(photoRef, {
                    likes: newLikes
                });

            } catch (error) {
                console.error("Error toggling like: ", error);
                showMessage('Error toggling like: ' + error.message, 'bg-red-500');
            } finally {
                if (likeButton) likeButton.disabled = false;
            }
        }
        
        // 9. Add Comment (Same as before)
        window.addComment = async function(docId) {
            const input = document.getElementById(`comment-input-${docId}`);
            const commentText = input.value.trim();

            if (!db || !userId) {
                showMessage('Database is not available for commenting.', 'bg-red-500');
                return;
            }

            if (!commentText) {
                showMessage('Please write a comment.', 'bg-yellow-500');
                return;
            }

            try {
                const photoRef = doc(db, PHOTOS_COLLECTION_PATH, docId);
                const docSnap = await getDoc(photoRef); 

                if (!docSnap.exists()) {
                    showMessage('Photo not found.', 'bg-red-500');
                    return;
                }
                
                const photoData = docSnap.data();
                const currentComments = photoData.comments || [];
                
                const newComment = {
                    text: commentText,
                    userId: userId,
                    // Use ISO string timestamp for unique comment identification
                    timestamp: new Date().toISOString() 
                };

                await updateDoc(photoRef, {
                    comments: [...currentComments, newComment]
                });

                showMessage('Comment successfully added!', 'bg-green-500');
                input.value = '';

            } catch (error) {
                console.error("Error adding comment: ", error);
                showMessage('Error adding comment: ' + error.message, 'bg-red-500');
            }
        }
        
        // 10. Delete Comment (Same as before)
        window.deleteComment = async function(docId, commentTimestamp, event) {
            if (event) event.stopPropagation();

            if (!db || !userId) {
                showMessage('Database is not available for deleting comment.', 'bg-red-500');
                return;
            }

            try {
                const photoRef = doc(db, PHOTOS_COLLECTION_PATH, docId);
                const docSnap = await getDoc(photoRef); 

                if (!docSnap.exists()) {
                    showMessage('Photo not found.', 'bg-red-500');
                    return;
                }
                
                const photoData = docSnap.data();
                const currentComments = photoData.comments || [];
                
                let commentToDeleteIndex = -1;
                
                // Find the comment by timestamp
                for (let i = 0; i < currentComments.length; i++) {
                    const comment = currentComments[i];
                    if (comment.timestamp === commentTimestamp) {
                        // Check deletion permissions: commenter or photo uploader
                        const isCommenter = comment.userId === userId;
                        const isUploader = photoData.uploaderId === userId;

                        if (isCommenter || isUploader) {
                            commentToDeleteIndex = i;
                            break;
                        } else {
                            // Comment found, but user lacks permission
                            showMessage('You do not have permission to delete this comment.', 'bg-red-500');
                            return;
                        }
                    }
                }

                if (commentToDeleteIndex !== -1) {
                    // Remove comment from the Array
                    const newComments = [...currentComments];
                    newComments.splice(commentToDeleteIndex, 1);
                    
                    await updateDoc(photoRef, {
                        comments: newComments
                    });

                    showMessage('Comment successfully deleted!', 'bg-green-500');
                } else {
                    showMessage('Comment not found.', 'bg-yellow-500');
                }

            } catch (error) {
                console.error("Error deleting comment: ", error);
                showMessage('Error deleting comment: ' + error.message, 'bg-red-500');
            }
        }

        // 11. Delete Photo from Firestore (Same as before)
        window.deletePhoto = async function(docId) {
            if (!db) {
                showMessage('Database is not available.', 'bg-red-500');
                return;
            }

            try {
                const photoRef = doc(db, PHOTOS_COLLECTION_PATH, docId);
                const docSnap = await getDoc(photoRef); 

                // Only allow uploader to delete
                if (!docSnap.exists() || docSnap.data().uploaderId !== userId) {
                    showMessage('This photo does not exist or you do not have permission to delete it.', 'bg-red-500');
                    return;
                }

                await deleteDoc(photoRef);
                showMessage('Photo successfully deleted!', 'bg-green-500');
            } catch (error) {
                console.error("Error deleting document: ", error);
                showMessage('Error deleting photo: ' + error.message, 'bg-red-500');
            }
        }

        // 12. Change Sorting Order (Same as before)
        window.changeSort = function(order) {
            currentSortOrder = order;
            // Update button states in UI
            document.getElementById('sort-newest').classList.remove('bg-indigo-600', 'text-white');
            document.getElementById('sort-newest').classList.add('bg-gray-200', 'text-gray-700');
            document.getElementById('sort-likes').classList.remove('bg-indigo-600', 'text-white');
            document.getElementById('sort-likes').classList.add('bg-gray-200', 'text-gray-700');
            
            document.getElementById(`sort-${order}`).classList.remove('bg-gray-200', 'text-gray-700');
            document.getElementById(`sort-${order}`).classList.add('bg-indigo-600', 'text-white');
        }

        // 13. Load Photo Gallery in Real-time (Same as before)
        function loadPhotos() {
            if (!db) return;

            const gallery = document.getElementById('photo-gallery');
            const photosRef = collection(db, PHOTOS_COLLECTION_PATH);

            const q = query(photosRef, orderBy('timestamp', 'desc'));

            onSnapshot(q, (snapshot) => {
                
                const photoArray = [];
                snapshot.forEach((doc) => {
                    photoArray.push({
                        id: doc.id,
                        ...doc.data()
                    });
                });
                
                // Client-side Sorting
                const sortedPhotos = photoArray.sort((a, b) => {
                    if (currentSortOrder === 'likes') {
                        const likesA = a.likes ? Object.keys(a.likes).length : 0;
                        const likesB = b.likes ? Object.keys(b.likes).length : 0;
                        return likesB - likesA;
                    } else { // 'newest'
                        const timeA = a.timestamp ? a.timestamp.toDate().getTime() : 0;
                        const timeB = b.timestamp ? b.timestamp.toDate().getTime() : 0;
                        return timeB - timeA;
                    }
                });
                
                gallery.innerHTML = ''; 

                if (sortedPhotos.length === 0) {
                    gallery.innerHTML = '<p class="text-center text-gray-500 col-span-full py-8">The gallery is currently empty. Add a photo using the form above.</p>';
                    return;
                }

                sortedPhotos.forEach((photoData) => {
                    const photoCard = createPhotoCard(photoData, photoData.id);
                    gallery.appendChild(photoCard);
                });
                
                window.changeSort(currentSortOrder);

            }, (error) => {
                console.error("Error listening to photos:", error);
                showMessage('Error loading photos. Please check the console.', 'bg-red-500');
            });
        }

        // 14. HTML Element creation for a Photo Card (Updated for optional Caption display)
        function createPhotoCard(photo, docId) {
            // Delete/Access privileges
            const isUploader = photo.uploaderId === userId;
            
            const likes = photo.likes || {};
            const likeCount = Object.keys(likes).length;
            const isLikedByCurrentUser = !!likes[userId];
            const comments = photo.comments || [];
            
            const card = document.createElement('div');
            // Added pointer-events-none by default, removed by isUploader to enable click for editing
            card.className = 'bg-white rounded-xl shadow-lg overflow-hidden transition-transform duration-300 hover:scale-[1.02] flex flex-col cursor-pointer';
            
            // Edit feature: If it's the uploader, allow click on the card to open edit modal
            if(isUploader) {
                // Pass isBase64 flag to edit function
                card.setAttribute('onclick', `startEditPhoto('${docId}', \`${(photo.caption || '').replace(/`/g, '\\`')}\`, \`${photo.url.replace(/`/g, '\\`')}\`, ${!!photo.isBase64})`);
                card.classList.add('ring-2', 'ring-indigo-300/50', 'hover:ring-indigo-500/80'); // Visual hint for editability
            }

            const likeButtonClass = isLikedByCurrentUser 
                ? 'text-red-600 bg-red-100/70 hover:bg-red-200' 
                : 'text-gray-500 bg-gray-100 hover:bg-red-50 hover:text-red-500';
            
            const likeIcon = isLikedByCurrentUser 
                ? `<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 20 20" fill="currentColor" class="w-5 h-5">
                     <path fill-rule="evenodd" d="M3.172 5.172a4.5 4.5 0 0 1 6.364 0L10 5.636l.464-.464a4.5 4.5 0 0 1 6.364 0 4.5 4.5 0 0 1 0 6.364L10 17.636l-6.828-6.828a4.5 4.5 0 0 1 0-6.364Z" clip-rule="evenodd" />
                   </svg>`
                : `<svg xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke-width="1.5" stroke="currentColor" class="w-5 h-5">
                     <path stroke-linecap="round" stroke-linejoin="round" d="M21 8.25c0-2.485-2.094-4.5-4.687-4.5H16.5A4.687 4.687 0 0 0 12 5.062 4.687 4.687 0 0 0 7.187 3.75H7.125C4.53 3.75 2.438 5.765 2.438 8.25c0 7.218 8.783 11.25 9.562 11.25.779 0 9.562-4.032 9.562-11.25Z" />
                   </svg>`;
            
            const placeholderImageUrl = `https://placehold.co/600x400/1E40AF/ffffff?text=Image+Not+Found`;

            const deleteButtonHtml = isUploader ? `
                <button 
                    onclick="event.stopPropagation(); deletePhoto('${docId}')" 
                    class="mt-2 w-full py-2 bg-red-500 text-white font-semibold rounded-lg hover:bg-red-600 transition duration-150 text-sm flex items-center justify-center space-x-2">
                    <!-- Trash icon -->
                    <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 20 20" fill="currentColor" class="w-5 h-5">
                        <path fill-rule="evenodd" d="M8.75 1A2.75 2.75 0 0 0 6 3.75v.5H2.75a.75.75 0 0 0 0 1.5h.76l.73 9.25A2.75 2.75 0 0 0 7 18.75h6A2.75 2.75 0 0 0 16.76 15l.73-9.25h.76a.75.75 0 0 0 0-1.5H14v-.5A2.75 2.75 0 0 0 11.25 1h-2.5ZM10 4.25a.75.75 0 0 0-.75.75v8.5a.75.75 0 0 0 1.5 0v-8.5a.75.75 0 0 0-.75-.75ZM7.75 5a.75.75 0 0 0-.75.75v7.5a.75.75 0 0 0 1.5 0v-7.5A.75.75 0 0 0 7.75 5Zm4.5 0a.75.75 0 0 0-.75.75v7.5a.75.75 0 0 0 1.5 0v-7.5a.75.75 0 0 0-.75-.75Z" clip-rule="evenodd" />
                    </svg>
                    <span>Delete Photo</span>
                </button>
            ` : `<p class="text-xs text-gray-400 mt-2 text-center border-t pt-2">Uploader ID: ${photo.uploaderId ? photo.uploaderId.substring(0, 8) + '...' : 'Unknown'}</p>`;

            // Render comments
            const commentsHtml = comments.map(comment => {
                const isCommenter = comment.userId === userId;
                const canDelete = isCommenter || isUploader;
                
                const deleteButton = canDelete ? `
                    <button 
                        onclick="deleteComment('${docId}', '${comment.timestamp}', event)" 
                        title="Delete Comment"
                        class="ml-2 text-red-400 hover:text-red-600 transition duration-150 p-1 rounded-full absolute right-1 top-1/2 transform -translate-y-1/2 opacity-0 group-hover:opacity-100"
                    >
                        <!-- Trash Icon -->
                        <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 20 20" fill="currentColor" class="w-3 h-3">
                            <path fill-rule="evenodd" d="M8.75 1A2.75 2.75 0 0 0 6 3.75v.5h-.75a.75.75 0 0 0 0 1.5h.76l.73 9.25A2.75 2.75 0 0 0 7 18.75h6A2.75 2.75 0 0 0 16.76 15l.73-9.25h.76a.75.75 0 0 0 0-1.5h-.75v-.5A2.75 2.75 0 0 0 11.25 1h-2.5ZM10 4.25a.75.75 0 0 0-.75.75v8.5a.75.75 0 0 0 1.5 0v-8.5a.75.75 0 0 0-.75-.75Z" clip-rule="evenodd" />
                        </svg>
                    </button>
                ` : '';

                return `
                    <div class="text-sm p-2 border-b border-gray-100 last:border-b-0 relative group">
                        <span class="font-bold text-indigo-500 text-xs">${comment.userId ? comment.userId.substring(0, 4) + '...' : 'Unknown'}</span>:
                        <span class="text-gray-700">${comment.text}</span>
                        ${deleteButton}
                    </div>
                `;
            }).join('');

            // Use photo.url directly, which is either a regular URL or Base64 data
            const imageUrl = photo.url || placeholderImageUrl;
            const imageErrorHtml = photo.isBase64 ? 
                '<div class=\'w-full h-full flex items-center justify-center text-center text-sm p-4 text-red-600\'>Base64 image is too large or corrupted.</div>' : 
                `<div class='w-full h-full flex items-center justify-center text-center text-sm p-4 text-red-600'>URL is invalid or image failed to load.</div>`;

            // Display "No Caption" if empty
            const captionDisplay = photo.caption && photo.caption.length > 0
                ? `<p class="font-semibold text-lg text-gray-800">${photo.caption}</p>`
                : `<p class="font-medium text-base italic text-gray-400">No Caption</p>`;


            card.innerHTML = `
                <!-- Photo Image -->
                <div class="aspect-video bg-gray-200 overflow-hidden">
                    <img 
                        src="${imageUrl}" 
                        alt="${photo.caption || 'No Caption'}" 
                        class="w-full h-full object-cover transition-opacity duration-500"
                        onerror="this.onerror=null;this.src='${placeholderImageUrl}';this.parentElement.innerHTML = '${imageErrorHtml}';"
                    >
                </div>
                <!-- Photo Details and Actions -->
                <div class="p-4 flex-grow flex flex-col justify-between">
                    <div>
                        <div class="flex justify-between items-start mb-2">
                           ${captionDisplay}
                            ${isUploader ? `
                                <div class="text-xs text-indigo-500 font-bold border border-indigo-500 rounded-full px-2 py-0.5 ml-2">EDIT</div>
                            ` : ''}
                        </div>

                        <!-- Action Row: Likes and Date -->
                        <div class="flex justify-between items-center mt-2 mb-3">
                            <button 
                                onclick="event.stopPropagation(); toggleLike('${docId}')" 
                                class="flex items-center space-x-1 p-2 rounded-lg transition duration-150 ${likeButtonClass}">
                                ${likeIcon}
                                <span class="font-bold text-sm">${likeCount}</span>
                                <span class="text-sm font-medium hidden sm:inline">Like</span>
                            </button>
                            <p class="text-xs text-gray-400">
                                ${photo.timestamp ? new Date(photo.timestamp.toDate()).toLocaleDateString('en-US') : 'Date Unknown'}
                            </p>
                        </div>

                    </div>
                    ${deleteButtonHtml}
                </div>
                
                <!-- Comments Section -->
                <div class="p-4 bg-gray-50 border-t border-gray-200" onclick="event.stopPropagation();">
                    <h4 class="font-semibold text-gray-700 mb-2 text-sm">Comments (${comments.length})</h4>
                    <div class="max-h-32 overflow-y-auto mb-3 border border-gray-200 rounded-lg bg-white">
                        ${commentsHtml.length > 0 ? commentsHtml : '<p class="text-xs text-gray-400 p-2 text-center">No comments yet.</p>'}
                    </div>
                    
                    <!-- Comment Input -->
                    <div class="flex space-x-2">
                        <input 
                            type="text" 
                            id="comment-input-${docId}" 
                            placeholder="Write your comment..."
                            class="flex-grow p-2 border border-gray-300 rounded-lg text-sm focus:ring-indigo-500 focus:border-indigo-500"
                        >
                        <button 
                            onclick="addComment('${docId}')" 
                            class="p-2 bg-indigo-500 text-white rounded-lg text-sm font-semibold hover:bg-indigo-600 transition duration-150"
                        >
                            Send
                        </button>
                    </div>
                </div>
            `;
            return card;
        }

        // Start load function on window load
        window.onload = function() {
            // Firebase setup starts above
        }
    </script>

    <style>
        body {
            font-family: 'Inter', sans-serif;
            background-color: #f3f4f6;
        }
        /* Mobile friendly layout */
        #main-container {
            max-width: 100%;
            padding: 0.5rem; 
        }
        @media (min-width: 640px) {
            #main-container {
                max-width: 768px;
                padding: 1.5rem;
            }
        }
        .sort-button {
            padding: 0.5rem 1rem;
            font-weight: 600;
            border-radius: 0.5rem;
            transition: all 0.15s;
        }
        .sort-button:hover {
            opacity: 0.9;
        }
        /* Show comment delete button on hover */
        .group:hover .opacity-0 {
            opacity: 100;
        }
    </style>
</head>
<body class="min-h-screen">
    
    <!-- Edit Modal (Hidden by Default) -->
    <div id="edit-modal" class="hidden fixed inset-0 bg-gray-900 bg-opacity-75 z-50 flex items-center justify-center p-4">
        <div class="bg-white rounded-xl shadow-2xl w-full max-w-md p-6 sm:p-8 transform transition-all duration-300">
            <h3 class="text-2xl font-bold text-indigo-700 mb-4">Edit Photo Details</h3>
            
            <input type="hidden" id="edit-doc-id">
            
            <div class="space-y-4">
                <div id="edit-url-group">
                    <label for="edit-url" class="block text-sm font-medium text-gray-700 mb-1">Image URL Link</label>
                    <input 
                        type="url" 
                        id="edit-url" 
                        placeholder="https://example.com/new-photo.jpg"
                        class="w-full p-3 border border-gray-300 rounded-lg focus:ring-indigo-500 focus:border-indigo-500"
                        required
                    >
                </div>
                <p id="edit-base64-message" class="hidden text-sm text-yellow-600 bg-yellow-100 p-3 rounded-lg border border-yellow-300">
                    *This is a direct upload. For security reasons, the image file cannot be re-uploaded or changed here. You can only update the caption.*
                </p>
                <div>
                    <label for="edit-caption" class="block text-sm font-medium text-gray-700 mb-1">Caption (Optional)</label>
                    <input 
                        type="text" 
                        id="edit-caption" 
                        placeholder="Updated description"
                        class="w-full p-3 border border-gray-300 rounded-lg focus:ring-indigo-500 focus:border-indigo-500"
                    >
                </div>
                
                <!-- Action Buttons -->
                <div class="flex justify-end space-x-3 mt-6">
                    <button 
                        onclick="closeEditModal()" 
                        class="px-4 py-2 text-gray-600 font-semibold rounded-lg border border-gray-300 hover:bg-gray-100 transition duration-150"
                    >
                        Cancel
                    </button>
                    <button 
                        onclick="updatePhoto()" 
                        id="update-button"
                        class="px-4 py-2 bg-indigo-600 text-white font-bold rounded-lg hover:bg-indigo-700 transition duration-150 shadow-md shadow-indigo-300/50"
                    >
                        Save Changes
                    </button>
                </div>
            </div>
        </div>
    </div>


    <div id="main-container" class="mx-auto mt-4 sm:mt-8">
        
        <!-- Header Section -->
        <header class="text-center p-4 bg-white rounded-2xl shadow-md mb-6">
            <div class="flex items-center justify-center">
                <!-- Custom Photo Icon with SMRK -->
                <div class="w-10 h-10 bg-indigo-600 rounded-lg shadow-xl relative flex items-center justify-center mr-3">
                    <span class="text-white text-sm font-black tracking-tighter">SMRK</span>
                </div>
                <h1 class="text-3xl font-bold text-indigo-700">Smrk</h1>
            </div>
            
            <p class="text-gray-500 mt-2">Upload small images directly or use image URLs! **(Uploader can click photo to Edit)**</p>
            <!-- User ID displayed here -->
            <p id="user-info" class="text-xs text-gray-400 mt-2"></p>
        </header>

        <!-- Message Box -->
        <div id="message-box" class="h-0 overflow-hidden transition-all duration-300"></div>

        <!-- Add Photo Form (File Input) -->
        <div class="bg-white p-6 sm:p-8 rounded-2xl shadow-lg mb-8">
            <h2 class="text-2xl font-semibold text-gray-700 mb-4">Add a New Photo</h2>
            <div class="space-y-4">
                <!-- File Input (New) -->
                <div>
                    <label for="photo-file" class="block text-sm font-medium text-gray-700 mb-1">Upload Image File (Max 300 KB)</label>
                    <input 
                        type="file" 
                        accept="image/*"
                        id="photo-file" 
                        class="w-full p-3 border border-gray-300 rounded-lg file:mr-4 file:py-2 file:px-4 file:rounded-lg file:border-0 file:text-sm file:font-semibold file:bg-indigo-50 file:text-indigo-700 hover:file:bg-indigo-100"
                    >
                    <p class="text-xs text-gray-500 mt-1">**Important:** For large images, use a public URL instead.</p>
                </div>
                
                <div class="flex items-center">
                    <div class="flex-grow border-t border-gray-300"></div>
                    <span class="flex-shrink mx-4 text-gray-400 text-sm">OR</span>
                    <div class="flex-grow border-t border-gray-300"></div>
                </div>

                <!-- URL Input (Optional fallback) -->
                <div>
                    <label for="photo-url" class="block text-sm font-medium text-gray-700 mb-1">Use Image URL Link</label>
                    <input 
                        type="url" 
                        id="photo-url" 
                        placeholder="https://example.com/my-photo.jpg"
                        class="w-full p-3 border border-gray-300 rounded-lg focus:ring-indigo-500 focus:border-indigo-500"
                    >
                </div>
                
                <div>
                    <label for="photo-caption" class="block text-sm font-medium text-gray-700 mb-1">Caption (Optional)</label>
                    <input 
                        type="text" 
                        id="photo-caption" 
                        placeholder="My mountain trip (Leave blank if you want no caption)"
                        class="w-full p-3 border border-gray-300 rounded-lg focus:ring-indigo-500 focus:border-indigo-500"
                    >
                </div>
                <button 
                    onclick="savePhoto()" 
                    id="save-button"
                    class="w-full py-3 px-4 bg-indigo-600 text-white font-bold rounded-lg hover:bg-indigo-700 transition duration-150 shadow-md shadow-indigo-300/50"
                >
                    Add Photo
                </button>
            </div>
        </div>

        <!-- Photo Gallery Display -->
        <div class="mt-8">
            <div class="flex justify-between items-center mb-4">
                <h2 class="text-2xl font-semibold text-gray-700">Gallery (Public View)</h2>
                
                <!-- Sorting Buttons -->
                <div class="flex space-x-2 bg-gray-100 p-1 rounded-lg shadow-inner">
                    <button 
                        id="sort-newest" 
                        onclick="changeSort('newest')" 
                        class="sort-button bg-indigo-600 text-white text-sm"
                    >
                        Newest
                    </button>
                    <button 
                        id="sort-likes" 
                        onclick="changeSort('likes')" 
                        class="sort-button bg-gray-200 text-gray-700 text-sm"
                    >
                        Most Popular
                    </button>
                </div>
            </div>
            
            <div 
                id="photo-gallery" 
                class="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 gap-6"
            >
                <!-- Photo cards load in real-time here -->
                <div class="col-span-full text-center py-10 text-gray-400">
                    <svg class="mx-auto h-12 w-12" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke-width="1.5" stroke="currentColor">
                        <path stroke-linecap="round" stroke-linejoin="round" d="m2.25 15.75 5.159-5.159a2.25 2.25 0 0 1 3.182 0l5.159 5.159m-1.581-1.581 1.581 1.581" />
                        <path stroke-linecap="round" stroke-linejoin="round" d="M12 21.75c-2.485 0-4.5-2.015-4.5-4.5S9.515 12.75 12 12.75s4.5 2.015 4.5 4.5S14.485 21.75 12 21.75zm0 0v-4.5" />
                        <path stroke-linecap="round" stroke-linejoin="round" d="M15 12.75l-4.5-4.5-4.5-4.5" />
                    </svg>
                    <p class="mt-2 text-sm font-medium">Loading photos...</p>
                </div>
            </div>
        </div>
    </div>
</body>
</html>


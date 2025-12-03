# SVstudentcpointtracker
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Real-Time Point Lookup</title>
    <!-- Load Tailwind CSS for styling -->
    <script src="https://cdn.tailwindcss.com"></script>
    <style>
        body {
            font-family: 'Inter', sans-serif;
            background-color: #f7fafc;
        }
        .card {
            box-shadow: 0 0 20px rgba(0, 0, 0, 0.08);
        }
        .text-shadow {
            text-shadow: 1px 1px 2px rgba(0, 0, 0, 0.1);
        }
    </style>
</head>
<body class="min-h-screen flex items-center justify-center p-4">

    <div id="app" class="w-full max-w-md bg-white p-8 rounded-xl card">
        <h1 class="text-3xl font-extrabold text-center text-indigo-700 mb-6 text-shadow">Real-Time Point Lookup</h1>

        <!-- Connection Status -->
        <p id="status" class="text-center text-sm mb-4 text-orange-500 font-medium">Connecting to secure database...</p>

        <!-- Input Field and Button -->
        <div class="space-y-4">
            <input 
                type="text" 
                id="studentInput" 
                placeholder="Enter Student ID or Full Name"
                class="w-full p-3 border border-gray-300 rounded-lg focus:ring-indigo-500 focus:border-indigo-500 text-lg transition duration-150"
                disabled
            >
            <button 
                onclick="lookupPoints()" 
                id="searchButton"
                class="w-full bg-indigo-600 text-white font-semibold py-3 rounded-lg shadow-lg hover:bg-indigo-700 transition duration-200 shadow-md transform hover:scale-[1.01] disabled:opacity-50 disabled:cursor-not-allowed"
                disabled
            >
                Search Points
            </button>
        </div>

        <!-- Result Display Area -->
        <div id="resultDisplay" class="mt-8 p-6 rounded-xl border-2 border-dashed border-gray-200 min-h-[8rem] flex items-center justify-center bg-gray-50">
            <p id="resultMessage" class="text-gray-500 text-center transition-opacity duration-300">
                Data will load automatically once connected.
            </p>
        </div>

        <!-- Hidden Admin Tool (Populate data) -->
        <button 
            onclick="addSampleData()" 
            id="adminDataButton" 
            class="hidden w-full mt-4 text-xs text-gray-400 hover:text-gray-600"
        >
            [Admin] Click here to initialize your spreadsheet with sample data.
        </button>
    </div>

    <!-- Firebase Setup and Logic -->
    <script type="module">
        import { initializeApp } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-app.js";
        import { getAuth, signInAnonymously, signInWithCustomToken, setLogLevel } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-auth.js";
        import { getFirestore, doc, setDoc, onSnapshot, collection } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-firestore.js";

        // Global variables provided by the environment
        const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';
        const firebaseConfig = JSON.parse(typeof __firebase_config !== 'undefined' ? __firebase_config : '{}');
        const initialAuthToken = typeof __initial_auth_token !== 'undefined' ? __initial_auth_token : null;

        // Constants
        // The data is stored in a public collection so students can look up their own data securely.
        const COLLECTION_PATH = `artifacts/${appId}/public/data/student_points`;
        
        // App State
        let db = null;
        let studentData = []; // This array holds the real-time "spreadsheet" data

        // DOM Elements
        const statusElement = document.getElementById('status');
        const inputElement = document.getElementById('studentInput');
        const searchButton = document.getElementById('searchButton');
        const adminButton = document.getElementById('adminDataButton');
        const resultMessageElement = document.getElementById('resultMessage');
        const resultDisplayElement = document.getElementById('resultDisplay');

        /**
         * Initializes Firebase and authenticates the user.
         */
        async function initializeFirebase() {
            try {
                // setLogLevel('debug'); // Uncomment for debugging
                const app = initializeApp(firebaseConfig);
                db = getFirestore(app);
                
                const auth = getAuth(app);
                if (initialAuthToken) {
                    await signInWithCustomToken(auth, initialAuthToken);
                } else {
                    await signInAnonymously(auth);
                }
                
                statusElement.textContent = `Connected! Searching ${COLLECTION_PATH.split('/').pop()}...`;
                statusElement.classList.remove('text-orange-500');
                statusElement.classList.add('text-green-600', 'font-semibold');
                
                // Enable UI elements
                inputElement.disabled = false;
                searchButton.disabled = false;
                searchButton.classList.remove('opacity-50', 'cursor-not-allowed');
                adminButton.classList.remove('hidden');

                // Start listening for real-time data
                listenForStudentData();

            } catch (error) {
                console.error("Firebase initialization failed:", error);
                statusElement.textContent = `Error connecting. Check console.`;
                statusElement.classList.remove('text-orange-500');
                statusElement.classList.add('text-red-600', 'font-semibold');
            }
        }

        /**
         * Sets up a real-time listener to sync the 'spreadsheet' data.
         */
        function listenForStudentData() {
            if (!db) return;
            const q = collection(db, COLLECTION_PATH);
            
            // onSnapshot provides the real-time update functionality!
            onSnapshot(q, (snapshot) => {
                const newData = [];
                snapshot.forEach((doc) => {
                    // Each document is a student record (a "row" in your spreadsheet)
                    newData.push(doc.data());
                });
                
                studentData = newData;
                
                // Display how many records were loaded
                if (inputElement.value === "") {
                    displayResult(`Database Live! ${studentData.length} student records loaded.`, 'border-gray-300 bg-gray-50');
                }
            }, (error) => {
                console.error("Error fetching student data:", error);
                // Fail silently in the UI but log the error
            });
        }

        // Expose lookupPoints and addSampleData to the global scope for onclick events
        window.lookupPoints = () => {
            const rawInput = inputElement.value;
            // Normalize input: trim whitespace and convert to lowercase for case-insensitive searching
            const searchTerm = rawInput.trim().toLowerCase(); 
            
            if (!searchTerm) {
                displayResult('Please enter a Student ID or Name to search.', 'border-red-400 bg-red-50');
                return;
            }

            // Search the LIVE data array
            const foundStudent = studentData.find(student => {
                // Check if ID matches exactly (case-insensitive)
                const idMatch = student.id && student.id.toLowerCase() === searchTerm;
                // Check if Name contains the search term (partial match, e.g., "liam" matches "Liam O'Connell")
                const nameMatch = student.name && student.name.toLowerCase().includes(searchTerm);
                return idMatch || nameMatch;
            });

            // Display the result
            if (foundStudent) {
                const message = `
                    <span class="block text-4xl font-extrabold text-green-700">${foundStudent.points || 0}</span>
                    <span class="block text-lg font-medium text-gray-700 mt-2">${foundStudent.name || foundStudent.id}</span>
                    <span class="block text-xs text-gray-500 mt-1">ID: ${foundStudent.id}</span>
                `;
                displayResult(message, 'border-green-400 bg-green-50');
            } else {
                const message = `
                    <span class="text-xl font-semibold text-red-600">No Match Found</span>
                    <span class="block text-sm text-gray-500 mt-1">Check the ID or name spelling.</span>
                `;
                displayResult(message, 'border-red-400 bg-red-50');
            }
        };


        window.addSampleData = async () => {
            if (!db) return;
            adminButton.textContent = "Adding data...";

            const sampleStudents = [
                { id: "S1001", name: "Liam O'Connell", points: 85, status: "Active" },
                { id: "S1002", name: "Zoe Chen", points: 120, status: "Active" },
                { id: "S1003", name: "Kai Rodriguez", points: 55, status: "Inactive" },
                { id: "S1004", name: "Irene Lee", points: 150, status: "Active" },
                { id: "S1005", name: "Marcus Johnson", points: 90, status: "Active" },
                { id: "S1006", name: "Elara Velez", points: 110, status: "Active" },
            ];

            try {
                for (const student of sampleStudents) {
                    // The student's ID (S1001) is used as the Firestore Document ID.
                    const docRef = doc(db, COLLECTION_PATH, student.id);
                    await setDoc(docRef, student);
                }
                adminButton.textContent = "Sample Data Added! (Look in Firestore console to edit)";
                adminButton.classList.add('text-green-500');
            } catch (e) {
                console.error("Error adding sample data: ", e);
                adminButton.textContent = "Error adding data. See console.";
                adminButton.classList.add('text-red-500');
            }
        };


        /**
         * Updates the result display with the message and styling.
         */
        function displayResult(messageHTML, classNames) {
            resultMessageElement.innerHTML = messageHTML;
            resultDisplayElement.className = `mt-8 p-6 rounded-xl border-2 ${classNames} min-h-[8rem] flex flex-col items-center justify-center text-center transition-all duration-300`;
        }

        // Optional: Allow search on Enter key press
        inputElement.addEventListener('keypress', function(event) {
            if (event.key === 'Enter' && !searchButton.disabled) {
                window.lookupPoints();
            }
        });
        
        // Start the application
        initializeFirebase();

    </script>
</body>
</html>

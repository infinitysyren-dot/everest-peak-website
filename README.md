<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>EVEREST PEAK | High-Altitude Solutions</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <script src="https://unpkg.com/lucide@latest"></script>
    <script src="https://cdn.jsdelivr.net/npm/chart.js@4.4.2/dist/chart.umd.min.js"></script>

    <style>
        /* Custom Styles for Enhanced Design and UX */
        .gradient-bg-v1 { background: linear-gradient(135deg, #4f46e5, #6366f1); }
        .gradient-bg-v2 { background: linear-gradient(135deg, #4338ca, #4f46e5); }
        .focus-ring { @apply focus:ring-4 focus:ring-indigo-300/70 focus:outline-none; }
        .main-container {
            max-width: 1400px;
            margin: 0 auto;
        }
        /* Skeleton Loader Styles */
        .skeleton-box {
            background-color: #e2e8f0; /* slate-200 */
            border-radius: 8px;
            animation: pulse 1.5s infinite ease-in-out;
        }
        @keyframes pulse {
            0%, 100% { opacity: 1; }
            50% { opacity: 0.5; }
        }
        /* Modal Transition */
        .modal-backdrop.open { opacity: 1; }
        .modal-content { transform: translateY(20px); transition: transform 0.3s ease; }
        .modal-backdrop.open .modal-content { transform: translateY(0); }
    </style>

    <script type="module">
        // Firebase Imports
        import { initializeApp, setLogLevel } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-app.js";
        import { getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged, createUserWithEmailAndPassword, signInWithEmailAndPassword, signOut } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-auth.js";
        import { getFirestore, doc, addDoc, onSnapshot, collection, query, where, serverTimestamp, getDocs, writeBatch } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-firestore.js";

        // Global Firebase and App Variables
        const firebaseConfig = {
            // Replace with your actual configuration
            apiKey: "YOUR_FIREBASE_API_KEY",
            authDomain: "YOUR_AUTH_DOMAIN",
            projectId: "YOUR_PROJECT_ID",
            storageBucket: "YOUR_STORAGE_BUCKET",
            messagingSenderId: "YOUR_MESSAGING_SENDER_ID",
            appId: "YOUR_APP_ID"
        };
        const appId = firebaseConfig.appId || 'default-app-id'; // Use project ID as fallback
        
        // Subscriptions
        let authSubscription = null;
        let productsSubscription = null;
        let callLogsSubscription = null;
        let financeSubscription = null;
        
        // Simulated Data (From Everest Website.html)
        const appDataSimulated = {
            rankings: [
                { id: 1, name: "GeoRank Pro", score: 9.8, trend: "up" },
                { id: 2, name: "CallLog Defender", score: 9.2, trend: "stable" },
                { id: 3, name: "Design Hub Elite", score: 8.5, trend: "down" },
                { id: 4, name: "Data Vault Secure", score: 9.5, trend: "up" },
            ],
            popularProducts: [
                { id: 'p001', name: "A.I. GeoRanker v2", category: "Software", rating: 4.9, designScore: 95, popular: true },
                { id: 'p002', name: "Secure Data Vault API", category: "Data", rating: 4.7, designScore: 88, popular: true },
                { id: 'p003', name: "Quantum-Proof Hardware Key", category: "Hardware", rating: 4.8, designScore: 92, popular: false },
                { id: 'p004', name: "Managed Scaling Service", category: "Service", rating: 4.6, designScore: 85, popular: true },
            ],
            salesmenData: [
                { category: "Software", name: "Elara Vance", product: "A.I. GeoRanker v2", rating: 4.9, phone: "555-SW-001", social: "@Elara_Code", avatarColor: "bg-blue-100" },
                { category: "Hardware", name: "Ryu Chen", product: "Quantum-Proof Hardware Key", rating: 4.8, phone: "555-HW-002", social: "@Ryu_Mech", avatarColor: "bg-red-100" },
                { category: "Data", name: "Maya Singh", product: "Secure Data Vault API", rating: 4.7, phone: "555-DT-003", social: "@MayaDataSci", avatarColor: "bg-green-100" },
                { category: "Service", name: "Jaxon Reed", product: "Managed Scaling Service", rating: 4.6, phone: "555-SV-004", social: "@Jaxon_Consult", avatarColor: "bg-yellow-100" },
            ]
        };


        // =========================================================================
        // STATE MANAGEMENT PATTERN (From index.html)
        // =========================================================================
        class StateController {
            constructor() {
                this.state = {
                    user: null,
                    isAuthReady: false,
                    products: [],
                    callLogs: [],
                    // Finance structure merged from both files
                    finance: { 
                        transactions: [], 
                        geoRank: 0, 
                        totalDeposit: 0, 
                        totalWithdrawal: 0, 
                        balance: 0, 
                        hasCreditCard: false // Added from File 2
                    },
                    isDataLoading: true 
                };
                this.subscribers = {};
            }

            subscribe(key, callback) {
                if (!this.subscribers[key]) {
                    this.subscribers[key] = [];
                }
                this.subscribers[key].push(callback);
                callback(this.state[key]);
                return () => {
                    this.subscribers[key] = this.subscribers[key].filter(sub => sub !== callback);
                };
            }

            setState(newState) {
                let changedKeys = [];
                for (const key in newState) {
                    if (JSON.stringify(this.state[key]) !== JSON.stringify(newState[key])) {
                        this.state[key] = newState[key];
                        changedKeys.push(key);
                    }
                }
                changedKeys.forEach(key => {
                    if (this.subscribers[key]) {
                        this.subscribers[key].forEach(callback => callback(this.state[key]));
                    }
                });
            }
        }
        window.stateController = new StateController();


        // =========================================================================
        // UTILITY CLASSES (From index.html)
        // =========================================================================

        class Util {
            static showValidationMessage(message, type = 'success', duration = 4000) {
                const container = document.getElementById('validation-message-container');
                if (!container) return; // Prevent errors if container is not in DOM
                
                const alertDiv = document.createElement('div');
                let bgColor = '', icon = '', iconColor = '';

                switch (type) {
                    case 'error':
                        bgColor = 'bg-red-500'; icon = 'alert-triangle'; iconColor = 'text-red-100'; break;
                    case 'info':
                        bgColor = 'bg-blue-500'; icon = 'info'; iconColor = 'text-blue-100'; break;
                    case 'warning':
                        bgColor = 'bg-yellow-500'; icon = 'bell'; iconColor = 'text-yellow-100'; break;
                    case 'success':
                    default:
                        bgColor = 'bg-green-500'; icon = 'check-circle'; iconColor = 'text-green-100'; break;
                }

                alertDiv.className = `flex items-center p-4 rounded-lg text-white shadow-xl mb-3 transition-opacity duration-500 ease-in-out ${bgColor}`;
                alertDiv.innerHTML = `
                    <i data-lucide="${icon}" class="w-6 h-6 mr-3 ${iconColor}"></i>
                    <span class="font-medium">${message}</span>
                `;
                
                container.appendChild(alertDiv);
                lucide.createIcons();

                setTimeout(() => {
                    alertDiv.style.opacity = '0';
                    setTimeout(() => alertDiv.remove(), 500);
                }, duration);
            }

            static formatCurrency(amount) {
                return new Intl.NumberFormat('en-US', { style: 'currency', currency: 'USD' }).format(amount);
            }
            
            static formatTimestamp(timestamp) {
                 return timestamp ? new Date(timestamp.seconds * 1000).toLocaleString() : 'Processing...';
            }
            
            // For form messages in Handlers (from Everest Website.html)
            static showFormMessage(element, message, isSuccess) { 
                element.textContent = message; 
                element.classList.remove('hidden', 'bg-red-100', 'bg-green-100', 'text-red-800', 'text-green-800'); 
                element.classList.add('p-3', 'rounded-lg'); 
                if (isSuccess) { 
                    element.classList.add('bg-green-100', 'text-green-800'); 
                } else { 
                    element.classList.add('bg-red-100', 'text-red-800'); 
                }
            }
        }

        class Router {
            constructor() {
                this.routes = {};
                window.onpopstate = () => this.navigate(document.location.pathname);
            }

            addRoute(path, elementId) {
                this.routes[path] = elementId;
            }

            navigate(path) {
                const elementId = this.routes[path];
                if (elementId) {
                    history.pushState(null, null, path);
                    document.querySelectorAll('.page-content').forEach(el => el.classList.add('hidden'));
                    document.getElementById(elementId).classList.remove('hidden');

                    document.querySelectorAll('nav a').forEach(a => {
                        if (a.getAttribute('href') === path) {
                            a.setAttribute('aria-current', 'page');
                            a.classList.add('bg-indigo-600/10', 'text-indigo-700', 'font-semibold');
                        } else {
                            a.removeAttribute('aria-current');
                            a.classList.remove('bg-indigo-600/10', 'text-indigo-700', 'font-semibold');
                        }
                    });
                    
                    document.getElementById('mobile-menu').classList.add('hidden');
                    DataRenderer.renderAll(); // Re-render data on navigation
                } else {
                    this.navigate('/dashboard'); // Changed default route to dashboard
                }
            }

            start() {
                // Find all links with data-route attribute to wire up navigation
                document.querySelectorAll('[data-route]').forEach(link => {
                    // Check if it's already a standard anchor tag with a href
                    if (link.tagName === 'A' && link.hasAttribute('href')) {
                         link.addEventListener('click', (e) => {
                            e.preventDefault();
                            this.navigate(link.getAttribute('href'));
                        });
                    }
                });
                // Initialize routes based on IDs in the HTML body
                document.querySelectorAll('.page-content').forEach(el => {
                    const id = el.id;
                    const path = `/${id.replace('-page', '').replace('-view', '')}`;
                    this.addRoute(path, id);
                });

                // Manually add home/default route
                this.addRoute('/', 'dashboard-page');

                this.navigate(document.location.pathname);
            }
        }
        window.router = new Router();


        // =========================================================================
        // FIREBASE AND AUTH SERVICES (Merged)
        // =========================================================================

        class FirebaseService {
            constructor() {
                this.app = initializeApp(firebaseConfig);
                this.auth = getAuth(this.app);
                this.db = getFirestore(this.app);
            }
            
            // Auth Functions (From Everest Website.html, adapted)
            async signUp(email, password) {
                try {
                    await createUserWithEmailAndPassword(this.auth, email, password);
                    return { success: true, message: "Registration successful! You are now signed in." };
                } catch (error) {
                    return { success: false, message: error.message };
                }
            }

            async signIn(email, password) {
                try {
                    await signInWithEmailAndPassword(this.auth, email, password);
                    return { success: true, message: "Sign-in successful!" };
                } catch (error) {
                    return { success: false, message: error.message };
                }
            }
            
            async signOutUser() {
                try {
                    await signOut(this.auth);
                    // Re-authenticate anonymously to keep the session alive with a new ID
                    await signInAnonymously(this.auth); 
                    return { success: true, message: "Successfully signed out." };
                } catch (error) {
                    console.error("Sign Out Error:", error);
                    return { success: false, message: error.message };
                }
            }

            // Centralized function to set up real-time data listeners (From index.html)
            setupDataListeners(userId) {
                this.unsubscribeAll();
                const db = this.db;

                // 1. Products Listener
                productsSubscription = onSnapshot(
                    query(collection(db, 'products'), where('userId', '==', userId)),
                    (snapshot) => {
                        const products = snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
                        window.stateController.setState({ products: products, isDataLoading: false });
                        DataRenderer.renderProductList(); // Update specific view
                        DataRenderer.updateDashboardMetrics(); // Update dashboard
                        // Util.showValidationMessage(`Products data updated (${products.length} items).`, 'info', 2000);
                    },
                    (error) => {
                        console.error("Error fetching products: ", error);
                        Util.showValidationMessage("Failed to load products data.", 'error');
                        window.stateController.setState({ isDataLoading: false });
                    }
                );
                
                // 2. Finance Listener (From index.html with File 2's structure)
                financeSubscription = onSnapshot(
                    doc(db, 'finance', userId), 
                    (docSnapshot) => {
                        let financeData = { transactions: [], geoRank: 0, totalDeposit: 0, totalWithdrawal: 0, balance: 0, hasCreditCard: false };

                        if (docSnapshot.exists()) {
                            financeData = docSnapshot.data();
                        }
                        
                        // Fetch sub-collection transactions as well
                        onSnapshot(
                            query(collection(db, `finance/${userId}/transactions`)),
                            (txSnapshot) => {
                                const transactions = txSnapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
                                
                                // Recalculate 'stored' balance/card status based on transactions (like File 2 logic)
                                let currentBalance = 0;
                                let hasCard = financeData.hasCreditCard || false; // Keep the stored hasCreditCard flag

                                transactions.forEach(t => {
                                    if (t.type === 'deposit' && t.status !== 'TRANSFERRED') {
                                        currentBalance += t.amount;
                                    } else if (t.type === 'withdrawal') {
                                        currentBalance -= t.amount;
                                    }
                                });
                                
                                const financeState = {
                                    ...financeData,
                                    transactions: transactions.sort((a, b) => b.timestamp - a.timestamp), 
                                    balance: currentBalance,
                                    hasCreditCard: hasCard // Use the state from the main doc or assume false
                                };
                                
                                window.stateController.setState({ finance: financeState, isDataLoading: false });
                                DataRenderer.renderFinanceView(); // Update specific view
                                DataRenderer.updateDashboardMetrics(); // Update dashboard
                            }
                        );
                    },
                    (error) => {
                        console.error("Error fetching finance data: ", error);
                        Util.showValidationMessage("Failed to load finance data.", 'error');
                        window.stateController.setState({ isDataLoading: false });
                    }
                );

                // 3. Call Logs Listener
                callLogsSubscription = onSnapshot(
                    query(collection(db, 'calllogs'), where('userId', '==', userId)),
                    (snapshot) => {
                        const callLogs = snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
                        window.stateController.setState({ callLogs: callLogs.sort((a, b) => (b.timestamp?.seconds || 0) - (a.timestamp?.seconds || 0)), isDataLoading: false });
                        DataRenderer.renderSocialHubView(); // Update specific view
                        DataRenderer.updateDashboardMetrics(); // Update dashboard
                    },
                    (error) => {
                        console.error("Error fetching call logs: ", error);
                        Util.showValidationMessage("Failed to load call logs.", 'error');
                        window.stateController.setState({ isDataLoading: false });
                    }
                );
            }

            unsubscribeAll() {
                if (productsSubscription) productsSubscription();
                if (callLogsSubscription) callLogsSubscription();
                if (financeSubscription) financeSubscription();
                productsSubscription = null;
                callLogsSubscription = null;
                financeSubscription = null;
            }
        }
        window.firebaseService = new FirebaseService();


        class DataService {
            constructor(db) {
                this.db = db;
            }
            
            // Helper for collection paths (From Everest Website.html, adapted for File 1 structure)
            getProductCollectionRef() {
                const userId = window.stateController.state.user?.uid;
                return collection(this.db, 'products'); // File 1 uses a simpler root collection
            }
            
            getCallLogCollectionRef() {
                const userId = window.stateController.state.user?.uid;
                return collection(this.db, 'calllogs'); // File 1 uses a simpler root collection
            }

            // Product Creation (From index.html)
            async createProduct(productData) {
                const user = window.stateController.state.user;
                if (!user) return { success: false, message: "Authentication required to create a product." };

                try {
                    await addDoc(this.getProductCollectionRef(), {
                        ...productData,
                        userId: user.uid,
                        createdAt: serverTimestamp(),
                        geoRank: Math.floor(Math.random() * 100)
                    });
                    return { success: true, message: "Product created successfully!" };
                } catch (error) {
                    const message = "Failed to create product. Please check your network or try again.";
                    console.error("Error creating product: ", error);
                    return { success: false, message: message };
                }
            }

            // Log Call (From index.html)
            async logCall(callData) {
                const user = window.stateController.state.user;
                if (!user) return { success: false, message: "Authentication required to log a call." };

                try {
                    await addDoc(this.getCallLogCollectionRef(), {
                        ...callData,
                        userId: user.uid,
                        timestamp: serverTimestamp(),
                    });
                    return { success: true, message: "Call logged successfully!" };
                } catch (error) {
                    const message = "Failed to log call. Please check your network or try again.";
                    console.error("Error logging call: ", error);
                    return { success: false, message: message };
                }
            }
            
            // Finance Update (From index.html, simplified for state use)
            async updateFinance(amount, type) {
                const user = window.stateController.state.user;
                if (!user) return { success: false, message: "Authentication required." };
                
                const financeDocRef = doc(this.db, 'finance', user.uid);
                const transactionRef = collection(financeDocRef, 'transactions');
                const isDeposit = type === 'deposit';

                try {
                    const batch = writeBatch(this.db);
                    
                    // 1. Add transaction
                    batch.set(doc(transactionRef), {
                        amount: amount,
                        type: type,
                        status: 'STORED', // All transactions start as stored until 'transferred'
                        timestamp: serverTimestamp(),
                        description: isDeposit ? 'User Deposit' : 'User Withdrawal'
                    });
                    
                    // 2. Update Finance Aggregates (Simulated. Will be updated by listener, but we ensure doc exists)
                    const currentFinance = window.stateController.state.finance;
                    const updateData = { 
                        geoRank: currentFinance.geoRank + (isDeposit ? 10 : -5), // Dummy GeoRank change
                        updatedAt: serverTimestamp(),
                        totalDeposit: currentFinance.totalDeposit + (isDeposit ? amount : 0),
                        totalWithdrawal: currentFinance.totalWithdrawal + (isDeposit ? 0 : amount),
                    };
                    
                    batch.set(financeDocRef, updateData, { merge: true });

                    await batch.commit();

                    return { success: true, message: `Successfully logged ${type} of ${Util.formatCurrency(amount)}!` };
                } catch (error) {
                    const message = "Failed to update finance. Please check your network or try again.";
                    console.error("Error updating finance: ", error);
                    return { success: false, message: message };
                }
            }
            
            // Simulate Credit Card Connection / Fund Transfer (From Everest Website.html, adapted)
            async transferFunds() {
                const user = window.stateController.state.user;
                if (!user) return { success: false, message: "Authentication required." };
                
                const currentFinance = window.stateController.state.finance;
                if (currentFinance.balance <= 0) {
                    return { success: false, message: "No stored funds to transfer." };
                }
                
                try {
                    const storedBalance = currentFinance.balance;
                    const financeDocRef = doc(this.db, 'finance', user.uid);
                    const transactionRef = collection(financeDocRef, 'transactions');
                    const batch = writeBatch(this.db);

                    // 1. Mark all previous 'STORED' deposits as 'TRANSFERRED'
                    // Note: This relies on the transaction listener logic for the actual balance calculation
                    const q = query(transactionRef, where('status', '==', 'STORED'));
                    const snapshot = await getDocs(q);
                    
                    snapshot.forEach((doc) => {
                        batch.update(doc.ref, { status: 'TRANSFERRED' });
                    });
                    
                    // 2. Update the main finance document to mark the card as connected
                    batch.set(financeDocRef, { hasCreditCard: true, updatedAt: serverTimestamp() }, { merge: true });
                    
                    await batch.commit();

                    return { success: true, message: `Successfully connected credit card and transferred ${Util.formatCurrency(storedBalance)} to your account!` };

                } catch (e) {
                    console.error("Error transferring funds: ", e);
                    return { success: false, message: "Failed to process transfer." };
                }
            }
        }
        window.dataService = new DataService(window.firebaseService.db);


        // =========================================================================
        // UI RENDERING UTILITIES (From Everest Website.html, adapted for StateController)
        // =========================================================================
        const DataRenderer = {
            // Renders all views that need initial data
            renderAll() {
                this.updateDashboardMetrics();
                this.renderDashboardChart();
                this.renderProductList();
                this.renderRankingsTable();
                this.renderFinanceView();
                this.renderSocialHubView();
                this.renderDisplayProducts();
                this.renderSalesmanView();
            },
            
            // Updates total counts on the dashboard
            updateDashboardMetrics() {
                const state = window.stateController.state;
                document.getElementById('dashboard-total-products').textContent = state.products.length;
                document.getElementById('dashboard-balance').textContent = Util.formatCurrency(state.finance.balance);
                document.getElementById('dashboard-geo-rank').textContent = state.finance.geoRank;
                document.getElementById('total-call-logs').textContent = state.callLogs.length; // From social hub
            },

            // Render the small dashboard chart
            renderDashboardChart() {
                const ctx = document.getElementById('geoRankChart');
                if (!ctx) return;
                
                if (window.dashboardChart) window.dashboardChart.destroy();

                const rankingsData = appDataSimulated.rankings; // Use simulated data for the static ranking chart
                
                window.dashboardChart = new Chart(ctx, {
                    type: 'line',
                    data: {
                        labels: rankingsData.map(r => r.name),
                        datasets: [{
                            label: 'Current Ranking Score',
                            data: rankingsData.map(r => r.score),
                            borderColor: 'rgb(75, 192, 192)',
                            tension: 0.4,
                            fill: true,
                            backgroundColor: 'rgba(75, 192, 192, 0.1)',
                            pointBackgroundColor: 'rgb(75, 192, 192)',
                            pointBorderColor: '#fff',
                            pointHoverBackgroundColor: '#fff',
                            pointHoverBorderColor: 'rgb(75, 192, 192)',
                        }]
                    },
                    options: {
                        responsive: true,
                        maintainAspectRatio: false,
                        plugins: { legend: { display: false }, tooltip: { mode: 'index', intersect: false } },
                        scales: { y: { beginAtZero: false, min: 8.0, max: 10.0, grid: { color: 'rgba(200, 200, 200, 0.2)' } }, x: { grid: { display: false } } }
                    }
                });
            },
            
            // Render list of created products in the Products view
            renderProductList() {
                const container = document.getElementById('product-list-container');
                const products = window.stateController.state.products;
                const isLoading = window.stateController.state.isDataLoading;
                if (!container) return;

                if (isLoading) {
                    container.innerHTML = Array(4).fill(0).map((_, i) => `
                        <div class="bg-white p-6 rounded-xl shadow-md">
                            <div class="skeleton-box h-6 w-3/4 mb-2"></div>
                            <div class="skeleton-box h-4 w-1/2 mb-4"></div>
                            <div class="flex justify-between items-center pt-2 border-t border-gray-100">
                                <div class="skeleton-box h-6 w-1/4"></div>
                                <div class="skeleton-box h-6 w-1/4"></div>
                            </div>
                        </div>
                    `).join('');
                    return;
                }
                
                if (products.length === 0) {
                    container.innerHTML = `
                        <div class="col-span-full p-10 text-center text-gray-500 bg-gray-50 rounded-xl border border-dashed border-indigo-200">
                            <i data-lucide="package-open" class="w-16 h-16 mx-auto mb-4 text-indigo-400"></i>
                            <p class="text-xl font-bold text-gray-700">No Products Created Yet</p>
                            <p class="text-sm">Use the form to start building your solution!</p>
                        </div>
                    `;
                    lucide.createIcons();
                    return;
                }

                container.innerHTML = products.map(product => {
                    // Use a geoRank property for visual feedback
                    const geoRank = product.geoRank || 50;
                    const color = geoRank > 75 ? 'bg-green-100 text-green-800' : geoRank > 50 ? 'bg-yellow-100 text-yellow-800' : 'bg-red-100 text-red-800';

                    return `
                        <div class="bg-white p-6 border border-gray-100 rounded-xl shadow-md hover:shadow-lg transition">
                            <h3 class="text-xl font-bold text-gray-800 mb-2">${product.name}</h3>
                            <p class="text-sm text-gray-500 mb-4">${product.category}</p>
                            <div class="flex justify-between items-center pt-2 border-t border-gray-100">
                                <span class="text-lg font-semibold text-indigo-600">${Util.formatCurrency(product.price || 99.99)}</span>
                                <span class="px-3 py-1 text-sm font-medium ${color} rounded-full">GeoRank: ${geoRank}</span>
                            </div>
                        </div>
                    `;
                }).join('');
                
                lucide.createIcons();
            },
            
            // Render the detailed Rankings table
            renderRankingsTable() {
                const tableBody = document.getElementById('rankings-table-body');
                if (!tableBody) return;
                
                tableBody.innerHTML = appDataSimulated.rankings.map(r => {
                    const trendIcon = r.trend === 'up' ? 'trending-up' : r.trend === 'down' ? 'trending-down' : 'minus';
                    const trendColor = r.trend === 'up' ? 'text-green-500' : r.trend === 'down' ? 'text-red-500' : 'text-yellow-500';
                    const trendBg = r.trend === 'up' ? 'bg-green-50' : r.trend === 'down' ? 'bg-red-50' : 'bg-yellow-50';

                    return `
                        <tr class="border-b hover:bg-indigo-50/50 transition">
                            <td class="px-6 py-4 whitespace-nowrap text-lg font-extrabold text-indigo-600">${r.id}</td>
                            <td class="px-6 py-4 whitespace-nowrap text-base font-semibold text-gray-800">${r.name}</td>
                            <td class="px-6 py-4 whitespace-nowrap text-2xl font-black">${r.score}</td>
                            <td class="px-6 py-4 whitespace-nowrap text-sm">
                                <span class="inline-flex items-center space-x-1 px-3 py-1 rounded-full font-medium ${trendColor} ${trendBg}">
                                    <i data-lucide="${trendIcon}" class="w-4 h-4"></i>
                                    <span>${r.trend.charAt(0).toUpperCase() + r.trend.slice(1)}</span>
                                </span>
                            </td>
                        </tr>
                    `;
                }).join('');
                lucide.createIcons();
            },

            // Render the Finance View
            renderFinanceView() {
                const container = document.getElementById('finance-view');
                const finance = window.stateController.state.finance;
                if (!container || !finance.transactions) return;

                // Card Status Section
                const hasCard = finance.hasCreditCard;
                document.getElementById('card-status').innerHTML = hasCard ? 
                    `<div class="inline-flex items-center px-4 py-1 rounded-full text-sm font-medium bg-green-100 text-green-800"><i data-lucide="credit-card" class="w-4 h-4 mr-2"></i> Connected</div>` :
                    `<div class="inline-flex items-center px-4 py-1 rounded-full text-sm font-medium bg-yellow-100 text-yellow-800"><i data-lucide="alert-triangle" class="w-4 h-4 mr-2"></i> Funds Stored</div>`;
                
                // Balance Display
                document.getElementById('finance-balance').textContent = Util.formatCurrency(finance.balance);
                
                // Transfer button visibility
                const transferBtn = document.getElementById('transfer-funds-btn');
                if (transferBtn) {
                    transferBtn.classList.toggle('hidden', hasCard || finance.balance <= 0);
                }

                // Transactions List
                const transactionList = document.getElementById('transaction-list');
                if (finance.transactions.length === 0) {
                    transactionList.innerHTML = `<p class="text-center text-gray-500 py-8 italic">No transactions recorded yet.</p>`;
                } else {
                    transactionList.innerHTML = finance.transactions.map(t => {
                        const isDeposit = t.type === 'deposit';
                        const isWithdrawal = t.type === 'withdrawal';
                        const sign = isDeposit ? '+' : isWithdrawal ? '-' : '';
                        const colorClass = isDeposit ? 'text-green-600' : 'text-red-600';
                        const statusClass = t.status === 'STORED' ? 'bg-yellow-100 text-yellow-800' : t.status === 'TRANSFERRED' ? 'bg-blue-100 text-blue-800' : 'bg-gray-100 text-gray-800';

                        return `
                            <li class="flex justify-between items-center py-3 border-b border-gray-100">
                                <div class="flex items-center">
                                    <i data-lucide="${isDeposit ? 'arrow-down-left' : 'arrow-up-right'}" class="w-5 h-5 mr-3 ${colorClass}"></i>
                                    <div>
                                        <p class="text-sm font-semibold text-gray-800">${t.description || t.type.toUpperCase()}</p>
                                        <p class="text-xs text-gray-400">${Util.formatTimestamp(t.timestamp)}</p>
                                    </div>
                                </div>
                                <div class="text-right">
                                    <div class="text-lg font-bold ${colorClass}">${sign}${Util.formatCurrency(Math.abs(t.amount))}</div>
                                    <span class="inline-flex items-center px-2 py-0.5 rounded-full text-xs font-medium ${statusClass}">
                                        ${t.status || 'PROCESSED'}
                                    </span>
                                </div>
                            </li>
                        `;
                    }).join('');
                }
                lucide.createIcons();
            },
            
            // Render the Social Hub View (Call Logs)
            renderSocialHubView() {
                const container = document.getElementById('call-log-container');
                const callLogs = window.stateController.state.callLogs;
                if (!container) return;
                
                if (callLogs.length === 0) {
                    container.innerHTML = `<p class="text-center text-gray-500 py-8 italic">No call logs recorded yet.</p>`;
                } else {
                    container.innerHTML = callLogs.map(log => {
                        const isOutgoing = log.type && log.type.includes('Outgoing');
                        const isMissed = log.type && log.type.includes('Missed');
                        const icon = isOutgoing ? 'phone-outgoing' : isMissed ? 'phone-missed' : 'phone-incoming';
                        const colorClass = isOutgoing ? 'text-indigo-600' : isMissed ? 'text-red-600' : 'text-green-600';
                        const date = Util.formatTimestamp(log.timestamp);
                        const durationText = isMissed ? 'Missed Call' : `Duration: ${log.duration} mins`;

                        return `
                            <div class="flex justify-between items-start p-4 border-b border-gray-100 last:border-b-0 hover:bg-gray-50 transition">
                                <div class="flex items-center">
                                    <i data-lucide="${icon}" class="w-5 h-5 mr-3 ${colorClass}"></i>
                                    <div>
                                        <p class="text-base font-semibold text-gray-800">${log.contactName || 'Unknown Contact'}</p>
                                        <p class="text-sm text-gray-500">${log.type} Call - ${durationText}</p>
                                    </div>
                                </div>
                                <span class="text-xs text-gray-400">${date}</span>
                            </div>
                        `;
                    }).join('');
                }
                lucide.createIcons();
            },

            // Render Display (Popular Products) View
            renderDisplayProducts() {
                const container = document.getElementById('display-product-list');
                if (!container) return;

                container.innerHTML = appDataSimulated.popularProducts.map(product => `
                    <div class="bg-white p-6 shadow-xl rounded-xl border border-gray-100 ${product.popular ? 'border-violet-500 ring-2 ring-violet-200' : ''} hover:shadow-2xl transition duration-300">
                        <div class="flex items-center justify-between mb-3">
                            <h3 class="text-xl font-bold text-gray-800">${product.name}</h3>
                            ${product.popular ? '<span class="px-3 py-1 text-xs font-semibold bg-violet-100 text-violet-800 rounded-full">POPULAR</span>' : ''}
                        </div>
                        <p class="text-sm font-semibold text-indigo-500 mb-4">${product.category}</p>
                        <div class="flex justify-between items-center pt-3 border-t border-gray-100">
                            <div class="text-center">
                                <p class="text-xs text-gray-500">Design Score</p>
                                <p class="text-lg font-bold text-indigo-600">${product.designScore}%</p>
                            </div>
                            <div class="text-center">
                                <p class="text-xs text-gray-500">Rating</p>
                                <p class="text-lg font-bold text-yellow-500 flex items-center">
                                    <i data-lucide="star" class="w-4 h-4 mr-1 fill-yellow-500"></i> ${product.rating.toFixed(1)}
                                </p>
                            </div>
                        </div>
                    </div>
                `).join('');
                lucide.createIcons();
            },

            // Render Salesman (Top Users) View
            renderSalesmanView() {
                const container = document.getElementById('salesman-list-container');
                if (!container) return;

                const salesmen = appDataSimulated.salesmenData;
                container.innerHTML = salesmen.map(salesman => `
                    <div class="bg-white p-6 shadow-xl rounded-xl border border-gray-100 flex flex-col items-center text-center hover:ring-4 hover:ring-violet-300 transition duration-300">
                        <div class="p-4 rounded-full ${salesman.avatarColor} ring-4 ring-white shadow-lg mb-4">
                            <i data-lucide="user" class="w-10 h-10 text-gray-800"></i>
                        </div>
                        <h3 class="text-2xl font-bold text-gray-900">${salesman.name}</h3>
                        <p class="text-sm font-semibold text-violet-600 mb-4">${salesman.category} Category Leader</p>
                        <div class="text-left w-full space-y-2">
                            <p class="text-sm text-gray-700"><span class="font-bold">Top Product:</span> ${salesman.product}</p>
                            <p class="text-sm text-gray-700 flex items-center"><i data-lucide="star" class="w-4 h-4 mr-2 fill-yellow-500 text-yellow-500"></i> <span class="font-bold">Rating:</span> ${salesman.rating.toFixed(1)}</p>
                            <p class="text-sm text-gray-700 flex items-center"><i data-lucide="phone" class="w-4 h-4 mr-2 text-indigo-500"></i> <span class="font-bold">Phone:</span> ${salesman.phone}</p>
                            <p class="text-sm text-gray-700 flex items-center"><i data-lucide="at-sign" class="w-4 h-4 mr-2 text-indigo-500"></i> <span class="font-bold">Social:</span> ${salesman.social}</p>
                        </div>
                    </div>
                `).join('');
                lucide.createIcons();
            }
        };
        window.dataRenderer = DataRenderer;


        // =========================================================================
        // MODAL MANAGER (From index.html)
        // =========================================================================
        const ModalManager = {
            init() {
                // Attach close listeners to all modal close buttons
                document.querySelectorAll('.modal-close').forEach(button => {
                    button.addEventListener('click', () => {
                        const modalId = button.closest('.modal-backdrop').id;
                        this.close(modalId);
                    });
                });
            },
            
            open(modalId) {
                const modal = document.getElementById(modalId);
                if (modal) {
                    modal.classList.remove('hidden');
                    // Force reflow to ensure transition works
                    modal.offsetHeight;
                    modal.classList.add('open');
                }
            },
            
            close(modalId) {
                const modal = document.getElementById(modalId);
                if (modal) {
                    modal.classList.remove('open');
                    setTimeout(() => {
                        modal.classList.add('hidden');
                    }, 300); // Match transition duration
                }
            }
        };


        // =========================================================================
        // UI HANDLERS (From Everest Website.html, adapted)
        // =========================================================================

        const AuthHandler = {
            authContainer: null,
            form: null,
            isSignIn: true, 

            init() {
                this.authContainer = document.getElementById('auth-content');
                if (!this.authContainer) return;

                // Initial render
                this.renderForm();
                
                // Event listener for mode switch button
                document.getElementById('auth-content').addEventListener('click', (e) => {
                    if (e.target.id === 'mode-switch-btn') {
                        this.isSignIn = !this.isSignIn;
                        this.renderForm();
                    }
                });
            },

            renderForm() {
                const isLogin = this.isSignIn;
                const formTitle = isLogin ? 'Sign In' : 'Create Account';
                const buttonText = isLogin ? 'Sign In' : 'Sign Up';
                const switchText = isLogin ? 'Need an account? Sign Up' : 'Already have an account? Sign In';

                this.authContainer.innerHTML = `
                    <h2 class="text-3xl font-extrabold text-gray-900 mb-2">${formTitle}</h2>
                    <p class="text-gray-500 mb-6">Access your high-altitude solutions for your business.</p>
                    <div id="auth-message" class="mb-4 text-sm text-center hidden font-medium"></div>
                    <form id="auth-form" class="space-y-4">
                        <div>
                            <label for="email" class="block text-sm font-medium text-gray-700">Email Address</label>
                            <input type="email" id="email" name="email" required class="mt-1 block w-full px-4 py-2 border border-gray-300 rounded-lg shadow-sm focus-ring" placeholder="you@example.com">
                        </div>
                        <div>
                            <label for="password" class="block text-sm font-medium text-gray-700">Password</label>
                            <input type="password" id="password" name="password" required class="mt-1 block w-full px-4 py-2 border border-gray-300 rounded-lg shadow-sm focus-ring" placeholder="Minimum 6 characters">
                        </div>
                        <button type="submit" class="w-full py-3 mt-4 rounded-lg gradient-bg-v1 text-white text-lg font-semibold hover:opacity-90 transition shadow-lg focus-ring">${buttonText}</button>
                    </form>
                    <div class="mt-6 text-center">
                        <button id="mode-switch-btn" type="button" class="text-sm font-semibold text-indigo-600 hover:text-indigo-500 transition" aria-label="Switch to ${isLogin ? 'Sign Up' : 'Sign In'} mode">
                            ${switchText}
                        </button>
                    </div>
                `;
                this.form = document.getElementById('auth-form');
                this.form.addEventListener('submit', this.handleAuthSubmit.bind(this));
            },

            async handleAuthSubmit(e) {
                e.preventDefault();
                const email = e.target.email.value;
                const password = e.target.password.value;
                const messageEl = document.getElementById('auth-message');
                const submitBtn = e.target.querySelector('button[type="submit"]');

                submitBtn.disabled = true;
                submitBtn.textContent = this.isSignIn ? 'Signing In...' : 'Registering...';
                messageEl.classList.add('hidden');

                let result;
                if (this.isSignIn) {
                    result = await window.firebaseService.signIn(email, password);
                } else {
                    result = await window.firebaseService.signUp(email, password);
                }

                Util.showFormMessage(messageEl, result.message, result.success);

                if (result.success) {
                    // Firebase listener handles the UI updates and modal close
                    e.target.reset();
                } else {
                    submitBtn.disabled = false;
                    submitBtn.textContent = this.isSignIn ? 'Sign In' : 'Sign Up';
                }
                setTimeout(() => messageEl.classList.add('hidden'), 5000);
            }
        };

        const DesignHub = {
            init() {
                const form = document.getElementById('product-creation-form');
                if (form) {
                    form.addEventListener('submit', this.handleProductCreation.bind(this));
                }
            },

            async handleProductCreation(e) {
                e.preventDefault();
                const name = e.target.productName.value.trim();
                const description = e.target.productDescription.value.trim();
                const category = e.target.productCategory.value;
                const messageEl = document.getElementById('design-hub-message');
                const submitBtn = e.target.querySelector('button[type="submit"]');

                // Client-side Input Validation
                if (name.length < 3) {
                    Util.showFormMessage(messageEl, "Product name must be at least 3 characters long.", false);
                    setTimeout(() => messageEl.classList.add('hidden'), 5000);
                    return;
                }
                if (description.length < 10) {
                    Util.showFormMessage(messageEl, "Product description must be at least 10 characters long.", false);
                    setTimeout(() => messageEl.classList.add('hidden'), 5000);
                    return;
                }

                submitBtn.disabled = true;
                submitBtn.textContent = 'Creating...';
                messageEl.classList.add('hidden');

                const productData = { name, description, category, price: 99.99 }; // Dummy price
                const result = await window.dataService.createProduct(productData);

                Util.showFormMessage(messageEl, result.message, result.success);
                if (result.success) {
                    e.target.reset();
                }

                submitBtn.disabled = false;
                submitBtn.textContent = 'Create Product';
                setTimeout(() => messageEl.classList.add('hidden'), 5000);
            }
        };
        
        const FinanceHandler = {
            init() {
                const depositForm = document.getElementById('deposit-form');
                const transferBtn = document.getElementById('transfer-funds-btn');

                if (depositForm) {
                    depositForm.addEventListener('submit', this.handleDeposit.bind(this));
                }
                if (transferBtn) {
                    transferBtn.addEventListener('click', this.handleTransfer.bind(this));
                }
            },

            async handleDeposit(e) {
                e.preventDefault();
                const amount = parseFloat(e.target.amount.value);
                const type = e.target.transactionType.value; // 'deposit' or 'withdrawal'
                const messageEl = document.getElementById('finance-message');
                const submitBtn = e.target.querySelector('button[type="submit"]');

                if (isNaN(amount) || amount <= 0) {
                    Util.showFormMessage(messageEl, "Please enter a valid amount greater than zero.", false);
                    setTimeout(() => messageEl.classList.add('hidden'), 3000);
                    return;
                }

                submitBtn.disabled = true;
                submitBtn.textContent = type === 'deposit' ? 'Processing Deposit...' : 'Processing Withdrawal...';
                messageEl.classList.add('hidden');

                const result = await window.dataService.updateFinance(amount, type);

                Util.showFormMessage(messageEl, result.message, result.success);
                if (result.success) {
                    e.target.reset();
                }

                submitBtn.disabled = false;
                submitBtn.textContent = 'Execute Transaction';
                setTimeout(() => messageEl.classList.add('hidden'), 5000);
            },
            
            async handleTransfer() {
                const messageEl = document.getElementById('finance-message');
                const transferBtn = document.getElementById('transfer-funds-btn');

                transferBtn.disabled = true;
                transferBtn.textContent = 'Connecting & Transferring...';
                messageEl.classList.add('hidden');
                
                const result = await window.dataService.transferFunds();

                Util.showFormMessage(messageEl, result.message, result.success);
                
                transferBtn.disabled = false;
                transferBtn.textContent = 'Transfer Stored Funds';
                setTimeout(() => messageEl.classList.add('hidden'), 5000);
            }
        };

        const SocialHubHandler = {
            init() {
                const form = document.getElementById('call-log-form');
                if (form) {
                    form.addEventListener('submit', this.handleLogCall.bind(this));
                }
            },

            async handleLogCall(e) {
                e.preventDefault();
                const contactName = e.target.contactName.value.trim();
                const number = e.target.phoneNumber.value.trim();
                const type = e.target.callType.value;
                const duration = parseFloat(e.target.duration.value);
                const messageEl = document.getElementById('call-log-message');
                const submitBtn = e.target.querySelector('button[type="submit"]');

                // Client-side Input Validation
                if (contactName.length < 3) {
                    Util.showFormMessage(messageEl, "Please enter a valid contact name (at least 3 characters).", false);
                    setTimeout(() => messageEl.classList.add('hidden'), 3000);
                    return;
                }
                if (number.length < 5) {
                    Util.showFormMessage(messageEl, "Please enter a valid phone number (at least 5 characters).", false);
                    setTimeout(() => messageEl.classList.add('hidden'), 3000);
                    return;
                }
                if (type !== 'Missed' && (isNaN(duration) || duration <= 0)) {
                    Util.showFormMessage(messageEl, "Please enter a valid call duration greater than zero.", false);
                    setTimeout(() => messageEl.classList.add('hidden'), 3000);
                    return;
                }

                submitBtn.disabled = true;
                submitBtn.textContent = 'Logging Call...';
                messageEl.classList.add('hidden');

                // Pass 0.01 duration for missed calls to ensure a number is logged
                const callDuration = type === 'Missed' ? 0.01 : duration;
                
                const callData = { contactName, phoneNumber: number, type: type, duration: callDuration };
                const result = await window.dataService.logCall(callData);

                Util.showFormMessage(messageEl, result.message, result.success);
                if (result.success) {
                    e.target.reset();
                }

                submitBtn.disabled = false;
                submitBtn.textContent = 'Log Call';
                setTimeout(() => messageEl.classList.add('hidden'), 5000);
            }
        };


        // =========================================================================
        // AI CHAT (From Everest Website.html, adapted for class structure of index.html)
        // =========================================================================
        class AIChat {
            constructor() {
                this.chatWindow = document.getElementById('ai-chat-window');
                this.chatForm = document.getElementById('chat-form');
                this.chatInput = document.getElementById('chat-input');
                this.messagesContainer = document.getElementById('messages-container');
                this.chatHistory = [ 
                    { role: "model", parts: [{ text: "Hello! I am your AI assistant for EVEREST PEAK. I can help you with app features, data insights (GeoRank/Call Logs), or general knowledge. **I use Google Search for real-time information.** What can I help you with?" }] }
                ];

                this.chatForm.addEventListener('submit', this.handleChatSubmit.bind(this));
                document.getElementById('ai-chat-btn').addEventListener('click', () => {
                    document.getElementById('aiChatModal').classList.toggle('hidden');
                });
                document.getElementById('chat-close-btn').addEventListener('click', () => {
                    document.getElementById('aiChatModal').classList.add('hidden');
                });
                
                this.renderChat();
                this.renderSuggestedPrompts();
            }
            
            scrollToBottom() {
                this.messagesContainer.scrollTop = this.messagesContainer.scrollHeight;
            }

            // New: Suggested Prompts Implementation
            renderSuggestedPrompts() {
                const prompts = [
                    "What is my current GeoRank score?", 
                    "Summarize my recent call log activity.", 
                    "What is a quantum-proof hardware key?",
                    "How can I improve my product's ranking?"
                ];
                const container = document.getElementById('suggested-prompts');
                if (!container) return;
                
                container.innerHTML = prompts.map(prompt => 
                    `<button type="button" class="text-xs px-3 py-1 bg-gray-100 text-gray-700 rounded-full hover:bg-indigo-100 transition duration-150 ease-in-out whitespace-nowrap">${prompt}</button>`
                ).join('');
                
                container.querySelectorAll('button').forEach(btn => {
                    btn.addEventListener('click', (e) => {
                        this.chatInput.value = e.target.textContent;
                        this.handleChatSubmit(null, e.target.textContent);
                    });
                });
            }

            renderChat(isLoading = false) {
                this.messagesContainer.innerHTML = this.chatHistory.map(message => {
                    const isUser = message.role === 'user';
                    const text = message.parts[0].text;
                    
                    // Simple logic for message rendering from index.html snippet
                    const messageDiv = document.createElement('div');
                    messageDiv.className = `flex mb-4 ${isUser ? 'justify-end' : 'justify-start'}`;
                    const contentClass = isUser ? 'bg-indigo-600 text-white rounded-t-xl rounded-bl-xl p-3 max-w-xs shadow-md' : 'bg-gray-100 text-gray-800 rounded-t-xl rounded-br-xl p-3 max-w-xs shadow-sm';
                    const icon = isUser ? 'user' : 'bot';
                    const iconColor = isUser ? 'text-indigo-600' : 'text-gray-500';
                    const avatar = `<i data-lucide="${icon}" class="w-5 h-5 ${isUser ? 'ml-3' : 'mr-3'} ${iconColor} flex-shrink-0"></i>`;
                    
                    let textContent = text;
                    
                    if (isUser) {
                        messageDiv.innerHTML = `<div class="${contentClass} flex items-end">${textContent}</div> ${avatar}`;
                    } else {
                        // Check for sources (From File 2)
                        let sourceHTML = '';
                        if (message.parts[0].groundingMetadata && message.parts[0].groundingMetadata.groundingChunks) {
                            const sources = message.parts[0].groundingMetadata.groundingChunks;
                            sourceHTML = sources.length > 0 ? `
                                <div class="mt-2 pt-2 border-t border-gray-200 text-xs text-gray-500">
                                    <p class="font-semibold">Sources:</p>
                                    ${sources.map((s, i) => `<a href="${s.web.uri}" target="_blank" class="block hover:text-indigo-600 truncate">${i+1}. ${s.web.title || 'Link'}</a>`).join('')}
                                </div>
                            ` : '';
                        }
                        
                        messageDiv.innerHTML = `${avatar} <div class="${contentClass} flex items-start flex-col">${textContent}${sourceHTML}</div>`;
                    }
                    return messageDiv.outerHTML;
                }).join('');
                
                // Add typing indicator
                if (isLoading) {
                    const typingIndicator = document.createElement('div');
                    typingIndicator.className = 'flex justify-start mb-4';
                    typingIndicator.innerHTML = `
                        <i data-lucide="bot" class="w-5 h-5 mr-3 text-gray-500 flex-shrink-0"></i>
                        <div class="bg-gray-100 text-gray-800 rounded-t-xl rounded-br-xl p-3 max-w-xs shadow-sm flex items-center space-x-2">
                            <div class="h-2 w-2 rounded-full bg-gray-500 animate-bounce" style="animation-delay: -0.32s"></div>
                            <div class="h-2 w-2 rounded-full bg-gray-500 animate-bounce" style="animation-delay: -0.16s"></div>
                            <div class="h-2 w-2 rounded-full bg-gray-500 animate-bounce"></div>
                        </div>
                    `;
                    this.messagesContainer.appendChild(typingIndicator);
                }
                
                lucide.createIcons();
                this.scrollToBottom();
            }

            async handleChatSubmit(e, promptOverride = null) {
                if (e) e.preventDefault();
                
                const prompt = promptOverride || this.chatInput.value.trim();
                if (!prompt) return;

                this.chatInput.value = '';

                // Add user message to history
                this.chatHistory.push({ role: 'user', parts: [{ text: prompt }] });
                this.renderChat();

                // Add model loading message
                this.renderChat(true);

                // Construct system prompt and context (from File 2, enhanced with current state)
                const state = window.stateController.state;
                const userProducts = state.products.map(p => `- ${p.name} (GeoRank: ${p.geoRank})`).join('\n');
                const callLogSummary = state.callLogs.length > 0 ? `Total Calls: ${state.callLogs.length}, Most Recent: ${state.callLogs[0].type} with ${state.callLogs[0].contactName} for ${state.callLogs[0].duration} mins.` : "No call logs available.";
                const financeSummary = `Balance: ${Util.formatCurrency(state.finance.balance)}, Total Deposit: ${Util.formatCurrency(state.finance.totalDeposit)}, Card Connected: ${state.finance.hasCreditCard ? 'Yes' : 'No'}.`;
                
                const systemPrompt = `You are the EVEREST PEAK AI Assistant, designed to help users with their dashboard data and general queries.
Available Context:
- User Products: \n${userProducts}
- User Finance: ${financeSummary}
- User Call Logs: ${callLogSummary}

Instructions:
1. When answering questions related to GeoRank, Call Logs, or Finance, use the provided user context.
2. For all other questions (e.g., about hardware keys, general knowledge), use the Google Search tool.
3. Keep your responses concise and format key data points clearly using **markdown for emphasis**.`;

                const apiKey = "YOUR_GEMINI_API_KEY"; 
                const apiUrl = `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-preview-09-2025:generateContent?key=${apiKey}`; 

                const payload = {
                    contents: [{ role: "user", parts: [{ text: prompt }] }],
                    tools: [{ "google_search": {} }],
                    systemInstruction: { parts: [{ text: systemPrompt }] },
                };

                let responseText = "Sorry, I couldn't get a response. Please try again.";
                let resultCandidate = null;

                try {
                    const response = await fetch(apiUrl, {
                        method: 'POST',
                        headers: { 'Content-Type': 'application/json' },
                        body: JSON.stringify(payload)
                    });

                    if (response.ok) {
                        const result = await response.json();
                        resultCandidate = result.candidates?.[0];
                        responseText = resultCandidate?.content?.parts?.[0]?.text || "I apologize, the response was empty.";
                    }
                } catch (error) {
                    console.error("Gemini API Error:", error);
                }
                
                // Remove loading indicator
                this.chatHistory.pop(); 
                
                // Add the actual response to history
                this.chatHistory.push({ 
                    role: 'model', 
                    parts: [{ text: responseText, groundingMetadata: resultCandidate?.groundingMetadata }] 
                });
                
                this.renderChat();
            }
        }
        window.aiclass = new AIChat();


        // =========================================================================
        // APP INITIALIZATION
        // =========================================================================
        window.addEventListener('DOMContentLoaded', () => {
            // 1. Initialize Core Services
            const appContainer = document.getElementById('app-container');
            const authModal = document.getElementById('authModal');
            const globalSkeleton = document.getElementById('global-skeleton');

            // Show loading spinner while Auth is in progress
            globalSkeleton.classList.remove('hidden');
            appContainer.classList.add('hidden');

            window.firebaseService.auth.onAuthStateChanged((user) => {
                const isFirstAuthCheck = !window.stateController.state.isAuthReady;
                
                if (user) {
                    window.stateController.setState({ user: user, isAuthReady: true });
                    
                    // Show main app content
                    appContainer.classList.remove('hidden');
                    globalSkeleton.classList.add('hidden');
                    ModalManager.close('authModal');
                    
                    // Setup listeners only once on successful auth
                    if (isFirstAuthCheck) {
                         window.firebaseService.setupDataListeners(user.uid);
                    }
                    
                    document.getElementById('user-id-display').textContent = `User ID: ${user.uid.substring(0, 10)}...`;
                    document.getElementById('auth-status').textContent = 'Signed In';
                    document.getElementById('logout-btn').classList.remove('hidden');
                    document.getElementById('sign-in-prompt-btn').classList.add('hidden');

                } else {
                    // Anonymous/Logged out state
                    window.stateController.setState({ user: null, isAuthReady: true });
                    
                    // Fallback UI for logged out state
                    appContainer.classList.add('hidden');
                    globalSkeleton.classList.add('hidden');

                    document.getElementById('user-id-display').textContent = 'User ID: Anonymous';
                    document.getElementById('auth-status').textContent = 'Signed Out';
                    document.getElementById('logout-btn').classList.add('hidden');
                    document.getElementById('sign-in-prompt-btn').classList.remove('hidden');
                    
                    // Force show login modal if not authenticated
                    if (isFirstAuthCheck) {
                       ModalManager.open('authModal');
                       // Initialize anonymous session for viewing, if allowed
                       signInAnonymously(window.firebaseService.auth).then(cred => {
                            window.firebaseService.setupDataListeners(cred.user.uid);
                       });
                    } else if (authModal.classList.contains('open') && !user) {
                        // Keep modal open if it was manually closed on the first check and we are still unauthenticated
                    }
                }
                
                if (isFirstAuthCheck) {
                    window.router.start(); 
                    DataRenderer.renderAll();
                }
            });
            
            // 2. Initialize UI Components
            ModalManager.init();
            AuthHandler.init();
            DesignHub.init();
            FinanceHandler.init();
            SocialHubHandler.init();
            
            // Logout Handler
            document.getElementById('logout-btn').addEventListener('click', () => {
                window.firebaseService.signOutUser();
            });

            // Mobile Menu Toggle
            const menuBtn = document.getElementById('mobile-menu-btn');
            const mobileMenu = document.getElementById('mobile-menu');
            menuBtn.addEventListener('click', () => {
                mobileMenu.classList.toggle('hidden');
                menuBtn.querySelector('[data-lucide="menu"]').classList.toggle('hidden');
                menuBtn.querySelector('[data-lucide="x"]').classList.toggle('hidden');
            });
        });
    </script>
</head>
<body class="bg-gray-50 min-h-screen">
    <div id="validation-message-container" class="fixed top-4 right-4 z-50 w-full max-w-sm"></div>

    <div id="authModal" class="modal-backdrop fixed inset-0 bg-gray-900 bg-opacity-80 flex items-center justify-center p-4 hidden opacity-0 transition-opacity duration-300 z-[100]">
        <div id="auth-content" class="modal-content bg-white w-full max-w-md p-8 rounded-xl shadow-2xl relative">
            </div>
    </div>

    <div id="aiChatModal" class="modal-backdrop fixed inset-0 bg-gray-900 bg-opacity-80 flex items-center justify-center p-4 hidden opacity-0 transition-opacity duration-300 z-[100]">
        <div id="ai-chat-window" class="bg-white w-full max-w-lg h-full max-h-[80vh] flex flex-col rounded-xl shadow-2xl relative">
            <div class="flex justify-between items-center p-4 border-b border-gray-200 bg-indigo-600 rounded-t-xl">
                <h3 class="text-xl font-bold text-white flex items-center">
                    <i data-lucide="bot" class="w-6 h-6 mr-2"></i> EVEREST AI Assistant
                </h3>
                <button id="chat-close-btn" class="modal-close text-indigo-100 hover:text-white transition">
                    <i data-lucide="x" class="w-6 h-6"></i>
                </button>
            </div>
            
            <div id="messages-container" class="flex-grow p-4 overflow-y-auto space-y-4">
                </div>
            
            <div class="p-4 border-t border-gray-200 bg-white">
                <div id="suggested-prompts" class="flex flex-wrap gap-2 mb-3">
                    </div>
                <form id="chat-form" class="flex space-x-2">
                    <input type="text" id="chat-input" class="flex-grow px-4 py-3 border border-gray-300 rounded-lg focus-ring text-base shadow-sm" placeholder="Ask about GeoRank, Call Logs, or Data Protection..." required>
                    <button id="chat-send-btn" type="submit" class="p-3 rounded-lg gradient-bg-v2 text-white hover:opacity-90 transition shadow-md focus-ring" aria-label="Send message">
                        <i data-lucide="send" class="w-6 h-6"></i>
                    </button>
                </form>
            </div>
        </div>
    </div>

    <div id="app-container" class="hidden min-h-screen">
        <header class="h-20 bg-white border-b border-gray-100 sticky top-0 z-30 shadow-sm">
            <div class="main-container h-full flex items-center justify-between px-4">
                <div class="flex items-center space-x-4">
                    <button id="mobile-menu-btn" class="lg:hidden p-2 rounded-lg text-indigo-600 hover:bg-indigo-50 transition">
                        <i data-lucide="menu" class="w-6 h-6"></i>
                        <i data-lucide="x" class="w-6 h-6 hidden"></i>
                    </button>
                    <a href="/" class="text-2xl font-black text-indigo-700 flex items-center space-x-2">
                        <i data-lucide="mountain" class="w-6 h-6"></i>
                        <span>EVEREST PEAK</span>
                    </a>
                </div>
                
                <div class="flex items-center space-x-4">
                    <button id="ai-chat-btn" class="p-3 rounded-full bg-indigo-50 text-indigo-600 hover:bg-indigo-100 transition shadow-md focus-ring" aria-label="Open AI Chat">
                        <i data-lucide="message-square-text" class="w-6 h-6"></i>
                    </button>

                    <div class="flex items-center space-x-2">
                        <i data-lucide="user-circle" class="w-8 h-8 text-gray-400"></i>
                        <div class="hidden md:block">
                            <p id="user-id-display" class="text-xs text-gray-500 font-medium truncate">User ID: Loading...</p>
                            <p id="auth-status" class="text-sm font-semibold text-gray-700">Loading...</p>
                        </div>
                    </div>
                    
                    <button id="logout-btn" class="hidden py-2 px-4 rounded-lg bg-red-500 text-white text-sm font-semibold hover:bg-red-600 transition shadow-md focus-ring">
                        <i data-lucide="log-out" class="w-5 h-5 inline-block mr-1"></i> Sign Out
                    </button>
                    <button id="sign-in-prompt-btn" class="py-2 px-4 rounded-lg gradient-bg-v1 text-white text-sm font-semibold hover:opacity-90 transition shadow-md focus-ring" onclick="ModalManager.open('authModal')">
                        <i data-lucide="log-in" class="w-5 h-5 inline-block mr-1"></i> Sign In
                    </button>
                </div>
            </div>
        </header>

        <div class="flex main-container">
            <aside id="mobile-menu" class="fixed inset-0 lg:static lg:w-64 bg-indigo-700 p-6 lg:p-4 hidden lg:block z-20">
                <nav class="space-y-2 pt-10 lg:pt-0">
                    <a href="/dashboard" data-route="dashboard" class="flex items-center space-x-3 p-3 rounded-xl text-white bg-indigo-600 font-semibold transition duration-150 ease-in-out" aria-current="page">
                        <i data-lucide="layout-dashboard" class="w-5 h-5"></i> <span class="font-semibold">Dashboard</span>
                    </a>
                    <a href="/products" data-route="products" class="flex items-center space-x-3 p-3 rounded-xl text-indigo-200 hover:bg-indigo-600 transition duration-150 ease-in-out">
                        <i data-lucide="package" class="w-5 h-5"></i> <span class="font-semibold">Products</span>
                    </a>
                    <a href="/rankings" data-route="rankings" class="flex items-center space-x-3 p-3 rounded-xl text-indigo-200 hover:bg-indigo-600 transition duration-150 ease-in-out">
                        <i data-lucide="bar-chart-3" class="w-5 h-5"></i> <span class="font-semibold">Rankings</span>
                    </a>
                    <a href="/finance" data-route="finance" class="flex items-center space-x-3 p-3 rounded-xl text-indigo-200 hover:bg-indigo-600 transition duration-150 ease-in-out">
                        <i data-lucide="credit-card" class="w-5 h-5"></i> <span class="font-semibold">Finance</span>
                    </a>
                    <a href="/social-hub" data-route="social-hub" class="flex items-center space-x-3 p-3 rounded-xl text-indigo-200 hover:bg-indigo-600 transition duration-150 ease-in-out">
                        <i data-lucide="message-square-text" class="w-5 h-5"></i> <span class="font-semibold">Social Hub</span>
                    </a>
                    <a href="/display" data-route="display" class="flex items-center space-x-3 p-3 rounded-xl text-indigo-200 hover:bg-indigo-600 transition duration-150 ease-in-out">
                        <i data-lucide="monitor-dot" class="w-5 h-5"></i> <span class="font-semibold">Display (Popular)</span>
                    </a>
                    <a href="/salesman" data-route="salesman" class="flex items-center space-x-3 p-3 rounded-xl text-indigo-200 hover:bg-indigo-600 transition duration-150 ease-in-out">
                        <i data-lucide="users" class="w-5 h-5"></i> <span class="font-semibold">Salesman (Top Users)</span>
                    </a>
                    <a href="/design-hub" data-route="design-hub" class="flex items-center space-x-3 p-3 rounded-xl text-indigo-200 hover:bg-indigo-600 transition duration-150 ease-in-out">
                        <i data-lucide="pencil-ruler" class="w-5 h-5"></i> <span class="font-semibold">Design Hub</span>
                    </a>
                </nav>
            </aside>

            <main class="flex-grow p-4 lg:p-8 min-h-screen">
                
                <div id="global-skeleton" class="hidden p-8 text-center text-indigo-500">
                    <i data-lucide="loader-circle" class="w-12 h-12 animate-spin mx-auto mb-4"></i>
                    <p class="text-lg font-semibold">Loading Everest Peak data...</p>
                </div>
                
                <div id="dashboard-page" class="page-content data-container grid grid-cols-1 lg:grid-cols-3 gap-8">
                    <div class="lg:col-span-2">
                        <h1 class="text-3xl font-bold text-gray-900 mb-6">Welcome to Your Dashboard</h1>
                        
                        <div class="grid grid-cols-1 md:grid-cols-3 gap-6 mb-8">
                            <div class="bg-white p-6 rounded-xl shadow-lg border border-gray-100">
                                <i data-lucide="box" class="w-8 h-8 text-indigo-500 mb-3"></i>
                                <p class="text-sm font-medium text-gray-500">Total Products</p>
                                <p id="dashboard-total-products" class="text-3xl font-bold text-gray-900 skeleton-box h-8 w-1/2"></p>
                            </div>
                            <div class="bg-white p-6 rounded-xl shadow-lg border border-gray-100">
                                <i data-lucide="trending-up" class="w-8 h-8 text-green-500 mb-3"></i>
                                <p class="text-sm font-medium text-gray-500">Current GeoRank</p>
                                <p id="dashboard-geo-rank" class="text-3xl font-bold text-gray-900 skeleton-box h-8 w-1/2"></p>
                            </div>
                            <div class="bg-white p-6 rounded-xl shadow-lg border border-gray-100">
                                <i data-lucide="wallet" class="w-8 h-8 text-yellow-500 mb-3"></i>
                                <p class="text-sm font-medium text-gray-500">Account Balance</p>
                                <p id="dashboard-balance" class="text-3xl font-bold text-gray-900 skeleton-box h-8 w-1/2"></p>
                            </div>
                        </div>

                        <div class="bg-white p-6 rounded-xl shadow-lg border border-gray-100">
                            <h2 class="text-2xl font-semibold text-gray-800 mb-4 flex items-center justify-between">
                                Recent Products
                                <a href="/products" class="text-indigo-600 text-sm hover:text-indigo-500">View All</a>
                            </h2>
                            <div id="product-list-container" class="grid grid-cols-1 md:grid-cols-2 gap-4">
                                </div>
                        </div>
                    </div>
                    
                    <div class="space-y-8">
                         <div class="bg-white p-6 rounded-xl shadow-lg border border-gray-100">
                            <h2 class="text-xl font-semibold text-gray-800 mb-4">GeoRank Trend</h2>
                            <div class="h-40">
                                <canvas id="geoRankChart"></canvas>
                            </div>
                            <p class="text-sm text-gray-500 mt-4">Simulated ranking trends for core solutions.</p>
                        </div>

                        <div class="bg-white p-6 rounded-xl shadow-lg border border-gray-100">
                            <h2 class="text-xl font-semibold text-gray-800 mb-4 flex items-center justify-between">
                                Recent Call Logs
                                <a href="/social-hub" class="text-indigo-600 text-sm hover:text-indigo-500">Full Log</a>
                            </h2>
                            <div id="call-log-container" class="divide-y divide-gray-100">
                                </div>
                        </div>
                    </div>
                </div>

                <div id="products-page" class="page-content data-container hidden space-y-8">
                    <h1 class="text-4xl font-extrabold text-gray-900 border-b pb-4">Product Management</h1>
                    <p class="text-lg text-gray-600">All the high-altitude solutions you have created in the Design Hub, loaded in real-time from your private data vault.</p>
                    <div id="product-list-container" class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 xl:grid-cols-4 gap-6">
                        </div>
                </div>

                <div id="rankings-view" class="page-content data-container hidden space-y-8">
                    <h1 class="text-4xl font-extrabold text-gray-900 border-b pb-4">System Ratings & Ranking</h1>
                    <p class="text-lg text-gray-600">Detailed breakdown of the core EVEREST PEAK solutions' performance metrics.</p>
                    <div class="bg-white overflow-hidden shadow-2xl rounded-xl border border-gray-100">
                        <table class="min-w-full divide-y divide-gray-200">
                            <thead class="bg-indigo-50">
                                <tr>
                                    <th scope="col" class="px-6 py-4 text-left text-sm font-bold text-indigo-800 uppercase tracking-wider">Rank</th>
                                    <th scope="col" class="px-6 py-4 text-left text-sm font-bold text-indigo-800 uppercase tracking-wider">Solution Name</th>
                                    <th scope="col" class="px-6 py-4 text-left text-sm font-bold text-indigo-800 uppercase tracking-wider">Rating Score (10)</th>
                                    <th scope="col" class="px-6 py-4 text-left text-sm font-bold text-indigo-800 uppercase tracking-wider">Trend</th>
                                </tr>
                            </thead>
                            <tbody id="rankings-table-body" class="bg-white divide-y divide-gray-200">
                                </tbody>
                        </table>
                    </div>
                </div>

                <div id="finance-page" class="page-content data-container hidden space-y-8">
                    <h1 class="text-4xl font-extrabold text-gray-900 border-b pb-4">Financial Overview</h1>
                    <p class="text-lg text-gray-600">Manage your deposits, withdrawals, and track transactions securely.</p>
                    <div class="grid grid-cols-1 lg:grid-cols-3 gap-8">
                        
                        <div class="lg:col-span-1 space-y-6">
                            <div class="bg-white p-6 rounded-xl shadow-2xl border-t-8 border-indigo-500">
                                <div class="flex justify-between items-center">
                                    <h2 class="text-xl font-bold text-gray-700">Account Balance</h2>
                                    <div id="card-status"></div>
                                </div>
                                <p id="finance-balance" class="text-6xl font-extrabold text-gray-900 mt-2 skeleton-box h-12 w-3/4"></p>
                                <p class="text-sm text-gray-500 mt-2">Funds available for transfer/withdrawal.</p>
                                <button id="transfer-funds-btn" class="w-full py-2 mt-4 rounded-lg bg-yellow-500 text-white font-semibold hover:bg-yellow-600 transition shadow-md focus-ring hidden">
                                    <i data-lucide="zap" class="w-5 h-5 inline-block mr-1"></i> Transfer Stored Funds
                                </button>
                            </div>

                            <div class="bg-white p-6 rounded-xl shadow-xl border border-gray-100">
                                <h3 class="text-2xl font-bold mb-5 text-gray-800 border-b pb-3">Execute Transaction</h3>
                                <div id="finance-message" class="mb-4 text-sm text-center hidden font-medium"></div>
                                <form id="deposit-form" class="space-y-4">
                                    <div>
                                        <label for="amount" class="block text-sm font-medium text-gray-700">Amount (USD)</label>
                                        <input type="number" id="amount" name="amount" step="0.01" required min="0.01" class="mt-1 block w-full px-3 py-2 border border-gray-300 rounded-lg shadow-sm focus-ring" placeholder="50.00">
                                    </div>
                                    <div>
                                        <label for="transactionType" class="block text-sm font-medium text-gray-700">Type</label>
                                        <select id="transactionType" name="transactionType" required class="mt-1 block w-full px-3 py-2 border border-gray-300 rounded-lg shadow-sm focus-ring">
                                            <option value="deposit">Deposit</option>
                                            <option value="withdrawal">Withdrawal</option>
                                        </select>
                                    </div>
                                    <button type="submit" class="w-full py-2 rounded-lg gradient-bg-v2 text-white font-semibold hover:opacity-90 transition shadow-md focus-ring">
                                        Execute Transaction
                                    </button>
                                </form>
                            </div>
                        </div>

                        <div class="lg:col-span-2 bg-white p-6 rounded-xl shadow-lg border border-gray-100">
                            <h2 class="text-2xl font-semibold text-gray-800 mb-4">Transaction History</h2>
                            <ul id="transaction-list" class="divide-y divide-gray-50">
                                </ul>
                        </div>
                    </div>
                </div>

                <div id="social-hub-page" class="page-content data-container hidden space-y-8">
                    <h1 class="text-4xl font-extrabold text-gray-900 border-b pb-4">Call Logs & Activity</h1>
                    <p class="text-lg text-gray-600">Simulate and track your call logs and social media engagement for analysis.</p>
                    <div class="grid grid-cols-1 lg:grid-cols-3 gap-8">
                        
                        <div class="lg:col-span-1 space-y-6">
                            <div class="bg-white p-6 rounded-xl shadow-2xl border-t-8 border-indigo-500">
                                <div class="flex justify-between items-center">
                                    <h2 class="text-xl font-bold text-gray-700">Total Call Logs</h2>
                                    <i data-lucide="phone" class="w-8 h-8 text-indigo-500"></i>
                                </div>
                                <p id="total-call-logs" class="text-6xl font-extrabold text-gray-900 mt-2 skeleton-box h-12 w-1/4"></p>
                                <p class="text-sm text-gray-500 mt-2">Recorded items in your private log.</p>
                            </div>

                            <div class="bg-white p-6 rounded-xl shadow-xl border border-gray-100">
                                <h3 class="text-2xl font-bold mb-5 text-gray-800 border-b pb-3">Simulate Call Log</h3>
                                <div id="call-log-message" class="mb-4 text-sm text-center hidden font-medium"></div>
                                <form id="call-log-form" class="space-y-4">
                                    <div>
                                        <label for="contactName" class="block text-sm font-medium text-gray-700">Contact Name</label>
                                        <input type="text" id="contactName" name="contactName" required class="mt-1 block w-full px-3 py-2 border border-gray-300 rounded-lg shadow-sm focus-ring" placeholder="Client X">
                                    </div>
                                    <div>
                                        <label for="phoneNumber" class="block text-sm font-medium text-gray-700">Phone Number</label>
                                        <input type="text" id="phoneNumber" name="phoneNumber" required class="mt-1 block w-full px-3 py-2 border border-gray-300 rounded-lg shadow-sm focus-ring" placeholder="555-1234">
                                    </div>
                                    <div>
                                        <label for="callType" class="block text-sm font-medium text-gray-700">Call Type</label>
                                        <select id="callType" name="callType" required class="mt-1 block w-full px-3 py-2 border border-gray-300 rounded-lg shadow-sm focus-ring">
                                            <option value="Incoming">Incoming</option>
                                            <option value="Outgoing">Outgoing</option>
                                            <option value="Missed">Missed</option>
                                        </select>
                                    </div>
                                    <div>
                                        <label for="duration" class="block text-sm font-medium text-gray-700">Duration (Minutes)</label>
                                        <input type="number" id="duration" name="duration" step="0.01" required min="0.01" class="mt-1 block w-full px-3 py-2 border border-gray-300 rounded-lg shadow-sm focus-ring" placeholder="5.5">
                                        <p class="text-xs text-gray-500 mt-1">Enter 0.01 for missed calls.</p>
                                    </div>
                                    <button type="submit" class="w-full py-2 rounded-lg gradient-bg-v2 text-white font-semibold hover:opacity-90 transition shadow-md focus-ring">
                                        <i data-lucide="phone-call" class="w-5 h-5 inline-block mr-1"></i> Log Call
                                    </button>
                                </form>
                            </div>
                        </div>

                        <div class="lg:col-span-2 bg-white p-6 rounded-xl shadow-lg border border-gray-100">
                            <h2 class="text-2xl font-semibold text-gray-800 mb-4">Recent Call Logs</h2>
                            <div id="call-log-container" class="divide-y divide-gray-100">
                                </div>
                        </div>
                    </div>
                </div>

                <div id="display-view" class="page-content data-container hidden space-y-8">
                    <h1 class="text-4xl font-extrabold text-gray-900 border-b pb-4">Popular Products & Design</h1>
                    <p class="text-lg text-gray-600">Explore the highest-rated products and their design quality scores across various categories.</p>
                    <div id="display-product-list" class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-4 gap-6">
                        </div>
                    <div class="bg-indigo-50 p-8 rounded-xl shadow-inner border border-indigo-200">
                        <h2 class="text-2xl font-bold text-indigo-900 mb-4 flex items-center">
                            <i data-lucide="lightbulb" class="w-6 h-6 mr-3"></i> Design Philosophy
                        </h2>
                        <p class="text-indigo-800">EVEREST PEAK prioritizes **Minimalist Functionality** and **Data-Driven Aesthetics**. All popular products exhibit a design score of 85% or higher, reflecting superior user experience and engineering rigor.</p>
                    </div>
                </div>

                <div id="salesman-view" class="page-content data-container hidden space-y-8">
                    <h1 class="text-4xl font-extrabold text-gray-900 border-b pb-4">Top Salesmen & Contributors</h1>
                    <p class="text-lg text-gray-600">Highlighting the top creators and category leaders driving innovation within the platform.</p>
                    <div id="salesman-list-container" class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-4 gap-6">
                        </div>
                </div>

                <div id="design-hub-page" class="page-content data-container hidden space-y-8">
                    <h1 class="text-4xl font-extrabold text-gray-900 border-b pb-4">Design & Creation Hub</h1>
                    <p class="text-lg text-gray-600">Use this hub to securely launch new high-altitude solutions for your business.</p>
                    
                    <div class="lg:w-1/2 bg-white p-8 rounded-xl shadow-2xl border border-gray-100 mx-auto">
                        <h2 class="text-3xl font-bold mb-6 text-gray-900">Create New Product</h2>
                        <div id="design-hub-message" class="mb-4 text-sm text-center hidden font-medium"></div>
                        <form id="product-creation-form" class="space-y-6">
                            <div>
                                <label for="productName" class="block text-sm font-medium text-gray-700">Product Name</label>
                                <input type="text" id="productName" name="productName" required class="mt-1 block w-full px-4 py-2 border border-gray-300 rounded-lg shadow-sm focus-ring" placeholder="Quantum Data Vault">
                            </div>
                            <div>
                                <label for="productCategory" class="block text-sm font-medium text-gray-700">Category</label>
                                <select id="productCategory" name="productCategory" required class="mt-1 block w-full px-4 py-2 border border-gray-300 rounded-lg shadow-sm focus-ring">
                                    <option value="Software">Software</option>
                                    <option value="Hardware">Hardware</option>
                                    <option value="Data">Data</option>
                                    <option value="Service">Service</option>
                                </select>
                            </div>
                            <div>
                                <label for="productDescription" class="block text-sm font-medium text-gray-700">Description</label>
                                <textarea id="productDescription" name="productDescription" rows="4" required class="mt-1 block w-full px-4 py-2 border border-gray-300 rounded-lg shadow-sm focus-ring" placeholder="A brief summary of the product's function and benefits (min 10 characters)."></textarea>
                            </div>
                            <button type="submit" class="w-full py-3 mt-4 rounded-lg gradient-bg-v1 text-white text-lg font-semibold hover:opacity-90 transition shadow-lg focus-ring">Create Product</button>
                        </form>
                    </div>
                </div>

            </main>
        </div>
    </div>
</body>
</html>

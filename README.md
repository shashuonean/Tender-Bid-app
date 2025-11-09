<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>TenderBid: India's Premier Tender & BBMP Bidding Platform</title>
    <meta name="description" content="TenderBid connects contractors and clients for BBMP-certified infrastructure projects. Post your project or place a bid and collect a 5% commission.">
    <meta name="robots" content="index, follow">

    <!-- Load Tailwind CSS -->
    <script src="https://cdn.tailwindcss.com"></script>
    <!-- Load React and ReactDOM (Stable Production Versions) -->
    <script src="https://unpkg.com/react@18/umd/react.production.min.js"></script>
    <script src="https://unpkg.com/react-dom@18/umd/react-dom.production.min.js"></script>
    <!-- Load Babel for JSX/ES6 support -->
    <script src="https://unpkg.com/@babel/standalone/babel.min.js"></script>

    <!-- 
        CRITICAL FIX: Load Firebase Dependencies from single file 
        This is much more reliable than separate imports in this environment.
    -->
    <script src="https://www.gstatic.com/firebasejs/10.12.2/firebase-app-compat.js"></script>
    <script src="https://www.gstatic.com/firebasejs/10.12.2/firebase-auth-compat.js"></script>
    <script src="https://www.gstatic.com/firebasejs/10.12.2/firebase-firestore-compat.js"></script>

    <script>
        // Global variables to hold initialized services
        window.dbInstance = null;
        window.authInstance = null;
        window.isFirebaseReady = false;
        
        // This function attempts to safely initialize Firebase
        const initializeFirebase = () => {
            if (window.isFirebaseReady || typeof firebase === 'undefined') return;

            try {
                const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';
                // Note: __firebase_config is a string and must be parsed
                const firebaseConfig = typeof __firebase_config !== 'undefined' ? JSON.parse(__firebase_config) : {};
                const initialAuthToken = typeof __initial_auth_token !== 'undefined' ? __initial_auth_token : null;

                if (Object.keys(firebaseConfig).length > 0) {
                    const app = firebase.initializeApp(firebaseConfig);
                    window.dbInstance = app.firestore(); // Using compat API style
                    window.authInstance = app.auth();     // Using compat API style
                    window.appId = appId;
                    window.initialAuthToken = initialAuthToken;
                    window.isFirebaseReady = true;

                    // This calls the function inside the React component to trigger a re-render
                    if (window.triggerFirebaseReady) {
                        window.triggerFirebaseReady();
                    }
                } else {
                    console.warn("Firebase config missing. Running without database persistence.");
                }
            } catch (e) {
                console.error("Firebase Initialization Error (Fatal):", e);
            }
        };

        // Initialize immediately after the DOM loads
        document.addEventListener('DOMContentLoaded', initializeFirebase);
        // Also try to initialize when the window finishes loading all scripts
        window.addEventListener('load', initializeFirebase);
    </script>
    <style>
        /* Custom scrollbar for better visual integration with the dark theme */
        ::-webkit-scrollbar {
            width: 8px;
        }
        ::-webkit-scrollbar-thumb {
            background-color: #4f46e5; /* Indigo-600 */
            border-radius: 4px;
        }
        ::-webkit-scrollbar-track {
            background: #1f2937; /* Gray-800 */
        }
    </style>
</head>
<body class="bg-gray-900">

    <div id="root"></div>

    <script type="text/babel">
        // Helper to mimic React imports from global scope
        const { useState, useEffect, useMemo, useCallback } = React;

        // --- Utility Hooks for State Management ---

        const useAlert = () => {
            const [alert, setAlert] = useState(null);
            const showAlert = (message, type = 'info') => {
                setAlert({ message, type });
                setTimeout(() => setAlert(null), 4000);
            };
            return { alert, showAlert };
        };

        const useFirebase = (isLoaded) => {
            const [userId, setUserId] = useState(null);
            const [loading, setLoading] = useState(true);
            
            // Access global compat instances
            const db = window.dbInstance;
            const auth = window.authInstance;
            const appId = window.appId;
            const initialAuthToken = window.initialAuthToken;

            useEffect(() => {
                if (!isLoaded || !auth || !db) {
                    // Only run if the Firebase global setup is complete
                    if (isLoaded) setLoading(false);
                    return;
                }

                const signIn = async () => {
                    try {
                        let userCredential;
                        if (initialAuthToken) {
                            userCredential = await auth.signInWithCustomToken(initialAuthToken);
                        } else {
                            userCredential = await auth.signInAnonymously();
                        }
                        setUserId(userCredential.user.uid);
                    } catch (error) {
                        console.error("Firebase Auth Error (Falling back to anonymous):", error);
                        try {
                            const userCredential = await auth.signInAnonymously();
                            setUserId(userCredential.user.uid);
                        } catch (e) {
                            console.error("Final Auth Failure:", e);
                        }
                    } finally {
                        setLoading(false);
                    }
                };
                
                // Use onAuthStateChanged for real-time auth status
                const unsubscribe = auth.onAuthStateChanged((user) => {
                    if (!user) {
                        signIn(); // Sign in if no user found
                    } else {
                        setUserId(user.uid);
                        setLoading(false);
                    }
                });

                return () => unsubscribe();
            }, [isLoaded]); // Depend only on the isLoaded flag

            // Firestore helpers using compat API
            const firestore = {
                collection: (path, ...subpaths) => db.collection(path).doc(appId).collection('public').doc('data').collection(...subpaths),
                batch: () => db.batch(),
                get: (ref) => ref.get(),
                onSnapshot: (queryRef, callback) => queryRef.onSnapshot(callback),
                query: (collectionRef, ...constraints) => collectionRef.where(...constraints),
            };


            return { db: firestore, userId, loading };
        };
        
        // --- Custom Components ---

        const Header = ({ currentView, setView, userId, currentUsername }) => {
            const isClient = currentView === 'client';
            return (
                <header className="flex justify-between items-center p-4 bg-gray-900 shadow-xl border-b border-indigo-700">
                    <div className="flex items-center space-x-2">
                        <svg className="w-8 h-8 text-orange-500" fill="currentColor" viewBox="0 0 24 24">
                            <path d="M12 2L2 7V17L12 22L22 17V7L12 2ZM12 19.3L4 15.1V8.9L12 13.1V19.3ZM20 15.1L12 19.3V13.1L20 8.9V15.1Z"/>
                        </svg>
                        <h1 className="text-3xl font-extrabold text-white tracking-wider">
                            Tender<span className="text-indigo-400">Bid</span>
                        </h1>
                    </div>
                    <div className="flex items-center space-x-4">
                        <button
                            onClick={() => setView('contractor')}
                            className={`px-4 py-2 text-sm font-semibold rounded-full transition duration-300 ${!isClient ? 'bg-indigo-600 text-white shadow-lg' : 'bg-gray-700 text-gray-300 hover:bg-gray-600'}`}
                        >
                            Contractor View
                        </button>
                        <button
                            onClick={() => setView('client')}
                            className={`px-4 py-2 text-sm font-semibold rounded-full transition duration-300 ${isClient ? 'bg-indigo-600 text-white shadow-lg' : 'bg-gray-700 text-gray-300 hover:bg-gray-600'}`}
                        >
                            Client View
                        </button>
                        <div className="flex items-center space-x-2 text-gray-400 text-sm">
                            <span className="hidden sm:inline">User:</span>
                            <span className="font-mono text-indigo-400">{currentUsername || userId?.substring(0, 8) + '...'}</span>
                        </div>
                    </div>
                </header>
            );
        };

        const CustomModal = ({ isOpen, onClose, title, children }) => {
            if (!isOpen) return null;
            return (
                <div className="fixed inset-0 z-50 flex items-center justify-center bg-gray-900 bg-opacity-75 p-4 transition-opacity">
                    <div className="bg-gray-800 rounded-xl shadow-2xl w-full max-w-lg overflow-hidden transform transition-all border border-indigo-700">
                        <div className="p-4 border-b border-gray-700 flex justify-between items-center">
                            <h3 className="text-xl font-bold text-white">{title}</h3>
                            <button onClick={onClose} className="text-gray-400 hover:text-white transition">
                                <svg className="w-6 h-6" fill="none" stroke="currentColor" viewBox="0 0 24 24" xmlns="http://www.w3.org/2000/svg"><path strokeLinecap="round" strokeLinejoin="round" strokeWidth="2" d="M6 18L18 6M6 6l12 12"></path></svg>
                            </button>
                        </div>
                        <div className="p-6 text-gray-200">
                            {children}
                        </div>
                    </div>
                </div>
            );
        };

        const ChatModal = ({ isOpen, onClose, tender, userId, currentUsername, db }) => {
            const [messages, setMessages] = useState([]);
            const [newMessage, setNewMessage] = useState('');
            const [chatLoading, setChatLoading] = useState(true);

            const chatCollectionRef = useMemo(() => db.collection('chats', 'messages'), [db]);

            useEffect(() => {
                if (!isOpen || !db || !tender.id) return;

                const chatQuery = db.query(chatCollectionRef, 'tenderId', '==', tender.id);
                setChatLoading(true);

                const unsubscribe = db.onSnapshot(chatQuery, (snapshot) => {
                    const msgs = snapshot.docs.map(doc => ({
                        id: doc.id,
                        ...doc.data(),
                        timestamp: doc.data().timestamp ? doc.data().timestamp.toDate() : new Date(0) // Convert Timestamp to Date
                    })).sort((a, b) => a.timestamp.getTime() - b.timestamp.getTime());
                    setMessages(msgs);
                    setChatLoading(false);
                }, (error) => {
                    console.error("Chat Listener Error:", error);
                    setChatLoading(false);
                });

                return () => unsubscribe();
            }, [isOpen, db, tender.id, chatCollectionRef]);

            const handleSendMessage = useCallback(async () => {
                if (newMessage.trim() === '' || !db || !userId) return;

                try {
                    await chatCollectionRef.add({
                        senderId: userId,
                        senderName: currentUsername,
                        text: newMessage.trim(),
                        timestamp: firebase.firestore.FieldValue.serverTimestamp(),
                        tenderId: tender.id,
                    });
                    setNewMessage('');
                } catch (error) {
                    console.error("Error sending message:", error);
                }
            }, [newMessage, userId, chatCollectionRef, tender.id, db, currentUsername]);

            const isMyMessage = (msgSenderId) => msgSenderId === userId;

            return (
                <CustomModal isOpen={isOpen} onClose={onClose} title={`Chat: ${tender.title}`}>
                    <div className="h-80 flex flex-col justify-between">
                        <div className="flex-grow overflow-y-auto p-2 space-y-3 bg-gray-700 rounded-lg max-h-[20rem]">
                            {chatLoading ? (
                                <p className="text-indigo-400">Loading messages...</p>
                            ) : messages.length === 0 ? (
                                <p className="text-gray-400 text-center py-10">Start the conversation!</p>
                            ) : (
                                messages.map((msg) => (
                                    <div key={msg.id} className={`flex ${isMyMessage(msg.senderId) ? 'justify-end' : 'justify-start'}`}>
                                        <div className={`max-w-xs px-4 py-2 rounded-xl shadow-md ${isMyMessage(msg.senderId) ? 'bg-indigo-600 text-white rounded-br-none' : 'bg-gray-600 text-gray-100 rounded-tl-none'}`}>
                                            <p className="text-xs font-semibold mb-1 opacity-75">{isMyMessage(msg.senderId) ? 'You' : msg.senderName}</p>
                                            <p className="text-sm break-words">{msg.text}</p>
                                        </div>
                                    </div>
                                ))
                            )}
                        </div>
                        <div className="pt-4 flex space-x-2">
                            <input
                                type="text"
                                value={newMessage}
                                onChange={(e) => setNewMessage(e.target.value)}
                                onKeyPress={(e) => e.key === 'Enter' && handleSendMessage()}
                                placeholder="Type your message..."
                                className="flex-grow p-3 bg-gray-600 border border-gray-500 rounded-lg text-white focus:border-indigo-500 focus:ring-1 focus:ring-indigo-500"
                            />
                            <button
                                onClick={handleSendMessage}
                                className="bg-indigo-600 hover:bg-indigo-700 text-white font-semibold py-2 px-4 rounded-lg transition duration-300"
                            >
                                Send
                            </button>
                        </div>
                    </div>
                </CustomModal>
            );
        };

        const PlaceBidModal = ({ isOpen, onClose, tender, userId, db, showAlert, currentUsername }) => {
            const [bidAmount, setBidAmount] = useState('');
            const [durationDays, setDurationDays] = useState('');
            const [loading, setLoading] = useState(false);

            const bidsCollectionRef = useMemo(() => db.collection('bids'), [db]);

            const handlePlaceBid = async () => {
                const amount = parseFloat(bidAmount);
                const duration = parseInt(durationDays);

                if (isNaN(amount) || amount <= 0 || isNaN(duration) || duration <= 0) {
                    showAlert('Please enter valid positive numbers for Bid Amount and Duration.', 'warning');
                    return;
                }

                setLoading(true);
                try {
                    await bidsCollectionRef.add({
                        tenderId: tender.id,
                        contractorId: userId,
                        contractorName: currentUsername,
                        amount: amount,
                        durationDays: duration,
                        platformFee: amount * 0.05, // 5% Commission
                        netEarnings: amount * 0.95,
                        status: 'pending',
                        timestamp: firebase.firestore.FieldValue.serverTimestamp(),
                    });
                    showAlert(`Bid of ₹${amount.toFixed(2)} placed successfully!`, 'success');
                    onClose();
                } catch (error) {
                    console.error("Error placing bid:", error);
                    showAlert('Failed to place bid. Please check your network.', 'error');
                } finally {
                    setLoading(false);
                }
            };

            return (
                <CustomModal isOpen={isOpen} onClose={onClose} title={`Place Bid for ${tender.title}`}>
                    <p className="mb-4 text-indigo-400">Platform fee: 5% of the total bid amount will be deducted.</p>
                    <div className="space-y-4">
                        <div>
                            <label className="block text-sm font-medium mb-1">Bid Amount (₹)</label>
                            <input
                                type="number"
                                value={bidAmount}
                                onChange={(e) => setBidAmount(e.target.value)}
                                placeholder="e.g., 50000"
                                className="w-full p-3 bg-gray-700 border border-gray-600 rounded-lg text-white focus:border-indigo-500"
                                min="1"
                            />
                        </div>
                        <div>
                            <label className="block text-sm font-medium mb-1">Estimated Duration (Days)</label>
                            <input
                                type="number"
                                value={durationDays}
                                onChange={(e) => setDurationDays(e.target.value)}
                                placeholder="e.g., 7"
                                className="w-full p-3 bg-gray-700 border border-gray-600 rounded-lg text-white focus:border-indigo-500"
                                min="1"
                            />
                        </div>
                        <div className="pt-4">
                            <button
                                onClick={handlePlaceBid}
                                disabled={loading}
                                className="w-full bg-orange-600 hover:bg-orange-700 text-white font-bold py-3 rounded-lg shadow-lg transition duration-300 disabled:opacity-50"
                            >
                                {loading ? 'Placing Bid...' : 'Submit Bid'}
                            </button>
                        </div>
                    </div>
                </CustomModal>
            );
        };

        const PostTenderForm = ({ userId, db, showAlert }) => {
            const [formData, setFormData] = useState({ title: '', description: '', location: '', bbmpId: '' });
            const [agreeToTerms, setAgreeToTerms] = useState(false);
            const [loading, setLoading] = useState(false);

            const tendersCollectionRef = useMemo(() => db.collection('tenders'), [db]);

            const handleChange = (e) => {
                setFormData({ ...formData, [e.target.name]: e.target.value });
            };

            const handlePostTender = async () => {
                if (!userId || loading) return;
                if (!formData.title || !formData.description || !formData.location || !formData.bbmpId) {
                    showAlert('Please fill in all required fields.', 'warning');
                    return;
                }
                if (!agreeToTerms) {
                    showAlert('You must agree to the Platform Terms before posting.', 'warning');
                    return;
                }

                setLoading(true);
                try {
                    await tendersCollectionRef.add({
                        ...formData,
                        clientId: userId,
                        status: 'Open',
                        createdAt: firebase.firestore.FieldValue.serverTimestamp(),
                        deadline: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000)
                    });
                    showAlert('New Tender Posted Successfully!', 'success');
                    setFormData({ title: '', description: '', location: '', bbmpId: '' });
                    setAgreeToTerms(false);
                } catch (error) {
                    console.error("Error posting tender:", error);
                    showAlert('Failed to post tender. Please try again.', 'error');
                } finally {
                    setLoading(false);
                }
            };

            return (
                <div className="bg-gray-800 p-6 rounded-xl shadow-2xl border border-indigo-700/50">
                    <h2 className="text-2xl font-bold text-white mb-6 border-b border-gray-700 pb-3">Post a New Tender</h2>
                    <div className="space-y-4">
                        <div>
                            <label className="block text-sm font-medium text-gray-300 mb-1">Tender Title</label>
                            <input type="text" name="title" value={formData.title} onChange={handleChange} placeholder="e.g., HVAC Repair for Office" className="w-full p-3 bg-gray-700 border border-gray-600 rounded-lg text-white focus:border-indigo-500" required />
                        </div>
                        <div>
                            <label className="block text-sm font-medium text-gray-300 mb-1">Description (Scope of Work)</label>
                            <textarea name="description" value={formData.description} onChange={handleChange} rows="3" placeholder="Detailed scope of work and materials needed..." className="w-full p-3 bg-gray-700 border border-gray-600 rounded-lg text-white focus:border-indigo-500" required></textarea>
                        </div>
                        <div className="grid grid-cols-1 sm:grid-cols-2 gap-4">
                            <div>
                                <label className="block text-sm font-medium text-gray-300 mb-1">Location / Area (Bengaluru)</label>
                                <input type="text" name="location" value={formData.location} onChange={handleChange} placeholder="e.g., Jayanagar" className="w-full p-3 bg-gray-700 border border-gray-600 rounded-lg text-white focus:border-indigo-500" required />
                            </div>
                            <div>
                                <label className="block text-sm font-medium text-gray-300 mb-1">BBMP Work ID (Mandatory for Compliance)</label>
                                <input type="text" name="bbmpId" value={formData.bbmpId} onChange={handleChange} placeholder="e.g., BBMP/HVAC/3001" className="w-full p-3 bg-gray-700 border border-gray-600 rounded-lg text-white focus:border-indigo-500" required />
                            </div>
                        </div>
                        <div className="pt-4 border-t border-gray-700">
                            <h3 className="text-lg font-semibold text-indigo-400 mb-2">Platform Terms & Liability Disclaimer</h3>
                            <p className="text-xs text-gray-400 mb-3 p-3 bg-gray-700 rounded-lg border border-red-500/30">
                                By proceeding, you agree that **TenderBid is solely a connection platform** and is NOT responsible for: (1) The quality of work performed by the contractor. (2) The contractor's payment or tax compliance (including GST). (3) Final project arbitration or liability after project handover.
                            </p>
                            <div className="flex items-center">
                                <input
                                    type="checkbox"
                                    checked={agreeToTerms}
                                    onChange={(e) => setAgreeToTerms(e.target.checked)}
                                    id="agreeToTerms"
                                    className="h-4 w-4 text-indigo-600 bg-gray-700 border-gray-500 rounded focus:ring-indigo-500"
                                />
                                <label htmlFor="agreeToTerms" className="ml-2 block text-sm font-medium text-white">
                                    I agree to the TenderBid Platform Terms & Disclaimer.
                                </label>
                            </div>
                        </div>
                        <button
                            onClick={handlePostTender}
                            disabled={loading || !agreeToTerms}
                            className="w-full bg-indigo-600 hover:bg-indigo-700 text-white font-bold py-3 rounded-lg shadow-lg transition duration-300 disabled:opacity-50 mt-6"
                        >
                            {loading ? 'Posting...' : 'Post Tender'}
                        </button>
                    </div>
                </div>
            );
        };

        const TenderCard = ({ tender, bids, isClient, userId, onChatClick, onBidClick, onAwardClick, onPaymentClick }) => {
            const tenderBids = bids.filter(bid => bid.tenderId === tender.id).sort((a, b) => a.amount - b.amount);
            const hasClientAwarded = tender.status === 'Awarded' || tender.status === 'Paid';
            const lowestBid = tenderBids.length > 0 ? tenderBids[0] : null;

            const getStatusStyle = (status) => {
                switch (status) {
                    case 'Open': return 'bg-green-600';
                    case 'Awarded': return 'bg-yellow-600';
                    case 'Paid': return 'bg-indigo-600';
                    default: return 'bg-gray-500';
                }
            };

            const ActionButton = () => {
                if (isClient) {
                    if (tender.status === 'Open') {
                        return (
                            <button onClick={() => onAwardClick(tender, tenderBids)} className="w-full bg-orange-600 hover:bg-orange-700 text-white font-semibold py-2 rounded-lg transition duration-300 mt-2">
                                Review Bids ({tenderBids.length})
                            </button>
                        );
                    } else if (tender.status === 'Awarded' && lowestBid) {
                        return (
                            <button onClick={() => onPaymentClick(tender, lowestBid)} className="w-full bg-indigo-600 hover:bg-indigo-700 text-white font-semibold py-2 rounded-lg transition duration-300 mt-2">
                                Simulate Payment
                            </button>
                        );
                    } else if (tender.status === 'Paid') {
                        return <span className="w-full text-center py-2 mt-2 font-bold text-green-400">Payment Complete!</span>;
                    }
                } else {
                    const hasContractorBid = tenderBids.some(bid => bid.contractorId === userId);
                    if (hasContractorBid) {
                        const myBid = tenderBids.find(bid => bid.contractorId === userId);
                        let message = 'Bid Submitted';
                        let style = 'bg-gray-600 text-gray-300';
                        if (hasClientAwarded && lowestBid?.contractorId === userId) {
                            message = 'AWARDED! (Proceed to work)';
                            style = 'bg-green-700 text-white animate-pulse';
                        } else if (hasClientAwarded) {
                            message = 'Tender Awarded to Another Bidder';
                            style = 'bg-red-700 text-white';
                        }

                        return <span className={`w-full text-center py-2 mt-2 font-bold rounded-lg ${style}`}>{message}</span>;

                    } else if (tender.status === 'Open') {
                        return (
                            <button onClick={() => onBidClick(tender)} className="w-full bg-teal-600 hover:bg-teal-700 text-white font-semibold py-2 rounded-lg transition duration-300 mt-2">
                                Place Bid
                            </button>
                        );
                    }
                }
                return null;
            };

            const isMyTender = tender.clientId === userId;
            const cardBgColor = isClient ? (isMyTender ? 'bg-gray-800' : 'bg-gray-700') : 'bg-gray-700';

            return (
                <div className={`p-5 rounded-xl shadow-lg border ${cardBgColor} transition duration-300 hover:shadow-2xl`}>
                    <div className="flex justify-between items-start mb-3 border-b border-gray-600 pb-2">
                        <h3 className="text-xl font-extrabold text-white">{tender.title}</h3>
                        <span className={`px-3 py-1 text-xs font-bold rounded-full text-white ${getStatusStyle(tender.status)}`}>
                            {tender.status}
                        </span>
                    </div>
                    <p className="text-sm text-gray-400 mb-3">{tender.description.substring(0, 100)}...</p>

                    <div className="space-y-2 text-sm">
                        <p className="text-gray-300"><span className="font-semibold text-indigo-400">ID:</span> {tender.bbmpId}</p>
                        <p className="text-gray-300"><span className="font-semibold text-indigo-400">Location:</span> {tender.location}</p>
                        {lowestBid && (
                            <p className="text-gray-300 font-semibold">
                                <span className="text-teal-400">Lowest Bid:</span> ₹{lowestBid.amount.toLocaleString('en-IN')} ({lowestBid.durationDays} Days)
                            </p>
                        )}
                    </div>

                    <div className="mt-4 pt-4 border-t border-gray-700">
                        <button
                            onClick={() => onChatClick(tender)}
                            className="w-full bg-gray-600 hover:bg-gray-500 text-white font-semibold py-2 rounded-lg transition duration-300"
                        >
                            Start Chat (Email/Chat Mode)
                        </button>
                        {ActionButton()}
                    </div>
                </div>
            );
        };

        const ReviewAwardModal = ({ isOpen, onClose, tender, bids, onAwardConfirm, showAlert }) => {
            const eligibleBids = bids.filter(b => b.tenderId === tender.id && b.status === 'pending').sort((a, b) => a.amount - b.amount);
            const hasBeenAwarded = tender.status === 'Awarded' || tender.status === 'Paid';

            const handleAward = async (bid) => {
                const confirmation = window.prompt('Type "CONFIRM" to award this tender.', '');
                if (confirmation !== 'CONFIRM') {
                    showAlert('Award action cancelled.', 'warning');
                    return;
                }

                onAwardConfirm(tender, bid);
                onClose();
            };

            return (
                <CustomModal isOpen={isOpen} onClose={onClose} title={`Review Bids for ${tender.title}`}>
                    <p className="mb-4 text-gray-300">Total Bids Received: <span className="font-bold text-indigo-400">{eligibleBids.length}</span></p>

                    {hasBeenAwarded ? (
                        <p className="text-center p-4 bg-green-900/50 text-green-400 rounded-lg font-bold">This tender has already been {tender.status}!</p>
                    ) : eligibleBids.length === 0 ? (
                        <p className="text-center p-4 bg-gray-700 rounded-lg text-gray-400">No active bids yet. Please wait.</p>
                    ) : (
                        <div className="space-y-3 max-h-80 overflow-y-auto">
                            {eligibleBids.map((bid, index) => (
                                <div key={bid.id} className={`p-4 rounded-xl shadow-md ${index === 0 ? 'bg-teal-900/50 border-2 border-teal-500' : 'bg-gray-700 border border-gray-600'}`}>
                                    <div className="flex justify-between items-center">
                                        <div>
                                            <p className="font-extrabold text-white text-lg">₹{bid.amount.toLocaleString('en-IN')}</p>
                                            <p className="text-sm text-gray-400">by {bid.contractorName} ({bid.durationDays} Days)</p>
                                        </div>
                                        <button
                                            onClick={() => handleAward(bid)}
                                            className="bg-indigo-600 hover:bg-indigo-700 text-white font-semibold py-2 px-4 rounded-lg transition duration-300"
                                        >
                                            {index === 0 ? 'Award (Lowest)' : 'Award'}
                                        </button>
                                    </div>
                                    <div className="mt-2 text-xs border-t border-gray-600 pt-2">
                                        <p className="text-red-300">Platform Fee (5%): ₹{bid.platformFee.toFixed(2)}</p>
                                        <p className="text-green-300 font-bold">Contractor Net Earning: ₹{bid.netEarnings.toFixed(2)}</p>
                                    </div>
                                </div>
                            ))}
                        </div>
                    )}
                </CustomModal>
            );
        };

        const PaymentModal = ({ isOpen, onClose, tender, bid, onPaymentConfirm }) => {
            const [loading, setLoading] = useState(false);

            const handleConfirm = () => {
                setLoading(true);
                onPaymentConfirm(tender, bid);
                setLoading(false);
                onClose();
            };

            if (!bid) return null;

            return (
                <CustomModal isOpen={isOpen} onClose={onClose} title={`Payment Simulation: ${tender.title}`}>
                    <p className="mb-6 text-gray-300 text-center">Simulate the payment process. TenderBid releases the funds to the contractor once the client confirms payment.</p>

                    <div className="bg-gray-700 p-5 rounded-lg space-y-3">
                        <div className="flex justify-between text-white font-bold text-lg">
                            <span>Total Tender Value:</span>
                            <span className="text-orange-400">₹{bid.amount.toLocaleString('en-IN')}</span>
                        </div>
                        <div className="flex justify-between text-sm text-gray-400 border-t border-gray-600 pt-3">
                            <span>-5% Platform Fee (TenderBid Commission):</span>
                            <span className="text-red-400">₹{bid.platformFee.toFixed(2)}</span>
                        </div>
                        <div className="flex justify-between text-white font-extrabold text-xl pt-3 border-t border-gray-600">
                            <span>Contractor Receives:</span>
                            <span className="text-green-400">₹{bid.netEarnings.toFixed(2)}</span>
                        </div>
                    </div>

                    <div className="mt-8">
                        <button
                            onClick={handleConfirm}
                            disabled={loading}
                            className="w-full bg-indigo-600 hover:bg-indigo-700 text-white font-bold py-3 rounded-lg shadow-lg transition duration-300 disabled:opacity-50"
                        >
                            {loading ? 'Processing...' : 'Confirm Payment'}
                        </button>
                    </div>
                </CustomModal>
            );
        };

        const AppContent = () => {
            const { db, userId, loading } = useFirebase(window.isFirebaseReady); // Pass the ready state
            const { alert, showAlert } = useAlert();
            const [currentView, setCurrentView] = useState('client');
            const [tenders, setTenders] = useState([]);
            const [bids, setBids] = useState([]);

            const [isChatOpen, setIsChatOpen] = useState(false);
            const [isBidOpen, setIsBidOpen] = useState(false);
            const [isAwardOpen, setIsAwardOpen] = useState(false);
            const [isPaymentOpen, setIsPaymentOpen] = useState(false);
            const [selectedTender, setSelectedTender] = useState(null);
            const [selectedBid, setSelectedBid] = useState(null);

            const tendersCollectionRef = useMemo(() => db.collection('tenders'), [db]);
            const bidsCollectionRef = useMemo(() => db.collection('bids'), [db]);

            const currentUsername = useMemo(() => {
                return currentView === 'client' ? 'Client: John D.' : 'Contractor: Sam T.';
            }, [currentView]);

            useEffect(() => {
                if (!db || !userId) return;

                const unsubscribeTenders = db.onSnapshot(tendersCollectionRef, (snapshot) => {
                    const tenderList = snapshot.docs.map(doc => ({
                        id: doc.id,
                        ...doc.data()
                    }));
                    setTenders(tenderList);
                }, (error) => console.error("Tenders Listener Error:", error));

                const unsubscribeBids = db.onSnapshot(bidsCollectionRef, (snapshot) => {
                    const bidList = snapshot.docs.map(doc => ({
                        id: doc.id,
                        ...doc.data()
                    }));
                    setBids(bidList);
                }, (error) => console.error("Bids Listener Error:", error));

                return () => {
                    unsubscribeTenders();
                    unsubscribeBids();
                };
            }, [db, userId]);

            const handleChatClick = (tender) => { setSelectedTender(tender); setIsChatOpen(true); };
            const handleBidClick = (tender) => { setSelectedTender(tender); setIsBidOpen(true); };
            const handleAwardClick = (tender, bids) => { setSelectedTender(tender); setIsAwardOpen(true); };
            const handlePaymentClick = (tender, bid) => { setSelectedTender(tender); setSelectedBid(bid); setIsPaymentOpen(true); };

            const handleAwardConfirm = async (tender, bidToAward) => {
                if (!db || !tender || !bidToAward) return;
                try {
                    const batch = db.batch();
                    const tenderDocRef = tendersCollectionRef.doc(tender.id);
                    batch.update(tenderDocRef, {
                        status: 'Awarded',
                        awardedBidId: bidToAward.id,
                        awardedContractorId: bidToAward.contractorId,
                        awardedAmount: bidToAward.amount,
                        platformFee: bidToAward.platformFee,
                        awardedAt: firebase.firestore.FieldValue.serverTimestamp(),
                    });
                    const winningBidDocRef = bidsCollectionRef.doc(bidToAward.id);
                    batch.update(winningBidDocRef, { status: 'Awarded' });
                    bids.filter(b => b.tenderId === tender.id && b.id !== bidToAward.id && b.status === 'pending').forEach(otherBid => {
                        const otherBidDocRef = bidsCollectionRef.doc(otherBid.id);
                        batch.update(otherBidDocRef, { status: 'Rejected' });
                    });
                    await batch.commit();
                    showAlert(`Tender "${tender.title}" successfully awarded!`, 'success');
                } catch (error) {
                    console.error("Error awarding tender:", error);
                    showAlert('Failed to award tender. Please try again.', 'error');
                }
            };

            const handlePaymentConfirm = async (tender, bid) => {
                if (!db || !tender || !bid) return;
                try {
                    const batch = db.batch();
                    const tenderDocRef = tendersCollectionRef.doc(tender.id);
                    batch.update(tenderDocRef, { status: 'Paid', paymentDate: firebase.firestore.FieldValue.serverTimestamp() });
                    const winningBidDocRef = bidsCollectionRef.doc(bid.id);
                    batch.update(winningBidDocRef, { status: 'Paid' });
                    await batch.commit();
                    showAlert(`Payment of ₹${bid.netEarnings.toFixed(2)} simulated and released!`, 'success');
                } catch (error) {
                    console.error("Error confirming payment:", error);
                    showAlert('Failed to confirm payment. Please try again.', 'error');
                }
            };

            if (loading) {
                return (
                    <div className="flex items-center justify-center h-screen bg-gray-900 text-indigo-400 text-xl">
                        <svg className="animate-spin -ml-1 mr-3 h-5 w-5" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24"><circle className="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" strokeWidth="4"></circle><path className="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"></path></svg>
                        Loading TenderBid...
                    </div>
                );
            }

            if (!db || !userId) {
                return (
                    <div className="flex items-center justify-center h-screen bg-gray-900 p-6">
                        <div className="text-center bg-red-900/50 p-6 rounded-lg border border-red-500">
                            <h1 className="text-2xl font-bold text-red-400">Initialization Failed</h1>
                            <p className="text-gray-300 mt-3">Could not connect to the database. Please reload the application.</p>
                            <p className="text-xs text-gray-400 mt-2">Error: No database or user connection established.</p>
                        </div>
                    </div>
                );
            }

            const clientTenders = tenders.filter(t => t.clientId === userId);
            const contractorTenders = tenders.filter(t => t.clientId !== userId);
            const availableTenders = currentView === 'client' ? clientTenders : contractorTenders;

            return (
                <div className="min-h-screen bg-gray-900 font-sans">
                    <Header currentView={currentView} setView={setCurrentView} userId={userId} currentUsername={currentUsername} />
                    {alert && (
                        <div className={`fixed top-4 right-4 z-50 p-4 rounded-lg shadow-xl text-white transition-opacity ${alert.type === 'success' ? 'bg-green-600' : alert.type === 'warning' ? 'bg-yellow-600' : 'bg-red-600'}`}>
                            <p className="font-semibold">{alert.message}</p>
                        </div>
                    )}
                    <main className="p-6 pt-10">
                        <h2 className="text-3xl font-bold text-white mb-6">
                            {currentView === 'client' ? 'Your Posted Tenders' : 'Available Bidding Opportunities'}
                        </h2>
                        <div className="grid grid-cols-1 lg:grid-cols-3 gap-8">
                            <div className="lg:col-span-1">
                                {currentView === 'client' ? (
                                    <PostTenderForm userId={userId} db={db} showAlert={showAlert} />
                                ) : (
                                    <div className="bg-gray-800 p-6 rounded-xl shadow-2xl border border-teal-700/50">
                                        <h2 className="text-2xl font-bold text-white mb-6 border-b border-gray-700 pb-3">Verified Contractor CRM</h2>
                                        <p className='text-gray-400 mb-4'>The best talent for TenderBid is listed here. (Mock Data)</p>
                                        <div className='space-y-3'>
                                            <div className='p-3 bg-gray-700 rounded-lg text-white'><span className='text-teal-400'>Contractor ID: </span>102 - Sharma Construction</div>
                                            <div className='p-3 bg-gray-700 rounded-lg text-white'><span className='text-teal-400'>Contractor ID: </span>103 - Bangalore Electric</div>
                                            <div className='p-3 bg-gray-700 rounded-lg text-white'><span className='text-teal-400'>Contractor ID: </span>104 - City HVAC Services</div>
                                        </div>
                                    </div>
                                )}
                            </div>
                            <div className="lg:col-span-2 space-y-6">
                                {availableTenders.length === 0 ? (
                                    <div className="p-10 text-center bg-gray-800 rounded-xl border border-indigo-700/50">
                                        <p className="text-xl text-gray-400">
                                            {currentView === 'client' ? "You haven't posted any tenders yet." : "No open tenders available right now."}
                                        </p>
                                    </div>
                                ) : (
                                    <div className="grid grid-cols-1 md:grid-cols-2 gap-6">
                                        {availableTenders.map(tender => (
                                            <TenderCard
                                                key={tender.id}
                                                tender={tender}
                                                bids={bids}
                                                isClient={currentView === 'client'}
                                                userId={userId}
                                                onChatClick={handleChatClick}
                                                onBidClick={handleBidClick}
                                                onAwardClick={handleAwardClick}
                                                onPaymentClick={handlePaymentClick}
                                            />
                                        ))}
                                    </div>
                                )}
                            </div>
                        </div>
                    </main>
                    {selectedTender && (
                        <ChatModal
                            isOpen={isChatOpen}
                            onClose={() => setIsChatOpen(false)}
                            tender={selectedTender}
                            userId={userId}
                            currentUsername={currentUsername}
                            db={db}
                        />
                    )}
                    {selectedTender && currentView === 'contractor' && (
                        <PlaceBidModal
                            isOpen={isBidOpen}
                            onClose={() => setIsBidOpen(false)}
                            tender={selectedTender}
                            userId={userId}
                            db={db}
                            showAlert={showAlert}
                            currentUsername={currentUsername}
                        />
                    )}
                    {selectedTender && currentView === 'client' && (
                        <ReviewAwardModal
                            isOpen={isAwardOpen}
                            onClose={() => setIsAwardOpen(false)}
                            tender={selectedTender}
                            bids={bids}
                            onAwardConfirm={handleAwardConfirm}
                            showAlert={showAlert}
                        />
                    )}
                    {selectedTender && selectedBid && currentView === 'client' && (
                        <PaymentModal
                            isOpen={isPaymentOpen}
                            onClose={() => setIsPaymentOpen(false)}
                            tender={selectedTender}
                            bid={selectedBid}
                            onPaymentConfirm={handlePaymentConfirm}
                        />
                    )}
                </div>
            );
        };

        const App = () => {
             const [isLoaded, setIsLoaded] = useState(window.isFirebaseReady);

             useEffect(() => {
                 // Register the global function that the script can call once firebase is initialized
                 window.triggerFirebaseReady = () => {
                    setIsLoaded(true);
                    console.log("Firebase Global Ready Flag Set.");
                 };

                 // Check once more after a small delay in case the script loaded very quickly
                 const timer = setTimeout(() => {
                     if (window.isFirebaseReady) {
                         setIsLoaded(true);
                     }
                 }, 500);

                 return () => {
                     clearTimeout(timer);
                     delete window.triggerFirebaseReady; // Clean up
                 }
             }, []);

             if (!isLoaded) {
                 return (
                     <div className="flex items-center justify-center h-screen bg-gray-900 text-indigo-400 text-xl">
                         <svg className="animate-spin -ml-1 mr-3 h-5 w-5" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24"><circle className="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" strokeWidth="4"></circle><path className="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"></path></svg>
                         Preparing Application Environment...
                     </div>
                 );
             }

             return <AppContent />;
        }

        const container = document.getElementById('root');
        const root = ReactDOM.createRoot(container);
        root.render(<App />);
    </script>
</body>
</html>


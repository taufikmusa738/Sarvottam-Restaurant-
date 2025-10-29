<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Sarvottam Restaurant Delivery App</title>
    <!-- Tailwind CSS CDN for styling -->
    <script src="https://cdn.tailwindcss.com"></script>
    <script>
        tailwind.config = {
            theme: {
                extend: {
                    colors: {
                        // Sarvottam के लोगो के रंग (नारंगी/भूरा और हरा) से मेल खाने के लिए अपडेट किया गया
                        'primary': '#F47E3C', // Sarvottam Orange/Brown
                        'secondary': '#FEF3C7', // कंट्रास्ट के लिए हल्का बैकग्राउंड
                        'accent': '#367C33', // Sarvattam Green
                    },
                    fontFamily: {
                        sans: ['Inter', 'sans-serif'],
                    },
                }
            }
        }
    </script>
    <!-- Firebase SDKs -->
    <script type="module">
        // आवश्यक Firebase मॉड्यूल को इंपोर्ट करें
        import { initializeApp } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-app.js";
        import { getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged, signOut } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-auth.js";
        import { getFirestore, doc, getDoc, addDoc, setDoc, updateDoc, onSnapshot, collection, query, where, serverTimestamp, setLogLevel } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-firestore.js";

        // Firebase Canvas एनवायरनमेंट से उपलब्ध कराए गए ग्लोबल वेरिएबल्स (इस्तेमाल करना अनिवार्य है)
        const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';
        const firebaseConfig = JSON.parse(typeof __firebase_config !== 'undefined' ? __firebase_config : '{}');
        const initialAuthToken = typeof __initial_auth_token !== 'undefined' ? __initial_auth_token : null;

        let app, db, auth, userId = null;
        let isAuthReady = false;
        let userName = ''; // यूज़र का नाम स्टोर करने के लिए वेरिएबल

        // --- ऐप स्टेट और मेनू डेटा ---
        let currentView = 'home'; // 'home', 'customer-menu', 'customer-orders', 'admin'
        let menuItems = [];
        let cart = {};
        let restaurantOrders = [];
        let customerOrders = [];
        
        // ऑर्डर स्टेटस की संभावित कुंजियाँ (Keys for Order Status)
        const orderStatusKeys = ['New Order', 'Preparing', 'Out for Delivery', 'Delivered', 'Cancelled'];
        
        // Firestore कलेक्शन पाथ
        const menuItemCollectionPath = `artifacts/${appId}/public/data/menu_items`;
        const orderCollectionPath = `artifacts/${appId}/public/data/orders`;

        // लॉगिंग के लिए यूटिलिटी फ़ंक्शन
        const log = (message, data = '') => console.log(`[APP LOG] ${message}`, data);

        // --- मुख्य Firebase इनिशियलाइज़ेशन और ऑथेंटिकेशन ---
        const initializeFirebase = async () => {
            try {
                // setLogLevel('debug'); 
                app = initializeApp(firebaseConfig);
                db = getFirestore(app);
                auth = getAuth(app);
                log("Firebase इनिशियलाइज़ हो गया।");

                // ऑथेंटिकेशन स्थिति श्रोता (Auth state listener)
                onAuthStateChanged(auth, (user) => {
                    if (user) {
                        userId = user.uid;
                        log("यूज़र प्रमाणीकृत (Authenticated) है। UID:", userId);
                        // साइन इन होने पर रियल-टाइम श्रोताओं (listeners) को सेट करें
                        setupOrderListeners(); 
                        // यदि यूज़र होम स्क्रीन पर है, तो उसे मेनू पर स्विच करें
                        if (currentView === 'home') {
                            switchView('customer-menu');
                        }
                    } else {
                        userId = null;
                        log("यूज़र साइन आउट या प्रमाणीकृत नहीं है।");
                    }
                    isAuthReady = true;
                    // होम स्क्रीन का प्रारंभिक रेंडर (Initial render)
                    renderApp(); 
                });

            } catch (error) {
                console.error("Firebase इनिशियलाइज़ेशन या ऑथेंटिकेशन विफल रहा:", error);
                document.getElementById('app-root').innerHTML = showMessage('Firebase सेटअप में कोई समस्या आई।', 'bg-red-500');
            }
        };
        
        // एनोनिमस साइन-इन प्रक्रिया को संभालने के लिए फ़ंक्शन
        const startAppSession = async (username) => {
            try {
                // ग्लोबल यूज़रनाम सेट करें
                userName = username; 
                log("यूज़र के रूप में साइन इन करने का प्रयास:", userName);

                // ऑर्डरों को ट्रैक करने के लिए एक यूनीक यूज़र आईडी प्राप्त करने हेतु एनोनिमस रूप से साइन इन करें
                if (initialAuthToken) {
                    await signInWithCustomToken(auth, initialAuthToken);
                } else {
                    await signInAnonymously(auth);
                }
                log("यूज़र ने सेशन शुरू किया और साइन इन हुआ।", userId);
                // onAuthStateChanged श्रोता (listener) फायर होगा और स्टेट को अपडेट करेगा, फिर switchView को कॉल करेगा
            } catch (error) {
                console.error("एनोनिमस रूप से साइन इन करने में त्रुटि:", error);
                document.getElementById('home-message').innerHTML = showMessage('लॉगिन विफल रहा। कृपया नेटवर्क जांचें।', 'bg-red-500');
                 // यदि ऑथेंटिकेशन विफल हो, तो यूज़रनाम क्लियर करें
                userName = '';
            }
        };

        // लॉगिन बटन क्लिक हैंडलर
        const handleLoginClick = () => {
            const inputElement = document.getElementById('username-input');
            const username = inputElement ? inputElement.value.trim() : '';

            if (username.length < 3) {
                document.getElementById('home-message').innerHTML = showMessage('कृपया एक वैध यूज़रनाम (कम से कम 3 वर्ण) दर्ज करें।', 'bg-red-500');
                return;
            }

            document.getElementById('home-message').innerHTML = '';
            window.appFunctions.startAppSession(username);
        };

        // --- लॉगआउट फ़ंक्शन ---
        const handleLogout = async () => {
            try {
                await signOut(auth); 
                log("यूज़र सफलतापूर्वक लॉगआउट हुआ।");
                
                // लोकल स्टेट क्लियर करें
                userName = '';
                cart = {};
                
                // लॉगिन स्क्रीन पर वापस स्विच करें
                switchView('home'); 
            } catch (error) {
                console.error("साइन आउट करने में त्रुटि:", error);
                // यदि साइन आउट विफल हो, तो भी लोकल स्टेट क्लियर करें और व्यू स्विच करें
                userName = '';
                cart = {};
                switchView('home'); 
            }
        };


        // --- UI रेंडरिंग फ़ंक्शन ---

        const showMessage = (text, bgColor = 'bg-primary') => `
            <div class="p-4 rounded-lg text-white ${bgColor} shadow-lg mt-4 max-w-sm mx-auto">
                ${text}
            </div>
        `;

        const renderHome = () => {
            // Sarvottam ब्रांडिंग के साथ सुरुचिपूर्ण मोबाइल-फ़र्स्ट होम स्क्रीन
            return `
                <div class="flex flex-col items-center justify-center min-h-screen p-6 bg-secondary text-gray-800">
                    
                    <!-- लोगो टेक्स्ट -->
                    <h2 class="text-6xl font-extrabold mb-0 text-primary mt-12">Sarvottam</h2>
                    <p class="text-xl font-semibold mb-4 text-accent">Restaurant</p>
                    <p class="text-gray-600 text-lg mb-8 border-b-2 border-accent pb-2">INDIAN VEG CUISINE</p>

                    <div class="w-full max-w-xs p-6 bg-white rounded-xl shadow-2xl">
                        <h3 class="text-2xl font-bold text-gray-800 mb-4 text-center">लॉगिन</h3>
                        <input id="username-input" type="text" placeholder="अपना यूज़रनाम दर्ज करें" value="${userName}"
                            class="w-full px-4 py-3 mb-4 border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-primary focus:border-primary transition duration-200">

                        <button onclick="window.appFunctions.handleLoginClick()" 
                                class="w-full bg-accent text-white text-xl font-bold py-3 rounded-xl shadow-lg transition transform hover:scale-[1.02] duration-300">
                            मेनू पर जारी रखें
                        </button>
                        <div id="home-message" class="mt-4 text-center"></div>
                    </div>
                    
                    <p class="text-xs text-gray-500 mt-8">ऑर्डर के लिए, हम एक सुरक्षित, अस्थायी यूज़र सेशन बनाते हैं।</p>
                </div>
            `;
        }

        const renderNavBar = () => {
            // होम व्यू पर नेविगेशन बार छिपाएँ
            if (currentView === 'home') return '';
            
            const getButtonClass = (view) => 
                currentView === view 
                ? 'bg-white text-primary font-bold shadow-md' 
                : 'text-white hover:bg-primary-dark';

            return `
                <nav class="bg-primary p-4 shadow-xl fixed top-0 w-full z-10">
                    <div class="flex justify-between items-center max-w-4xl mx-auto">
                        <h1 class="text-2xl font-extrabold text-white tracking-wider">Sarvottam</h1>
                        <div class="flex items-center space-x-2">
                            <button onclick="window.appFunctions.switchView('customer-menu')" class="px-3 py-1 text-sm rounded-full transition-colors ${getButtonClass('customer-menu')}">
                                मेनू
                            </button>
                            <button onclick="window.appFunctions.switchView('customer-orders')" class="px-3 py-1 text-sm rounded-full transition-colors ${getButtonClass('customer-orders')}">
                                मेरे ऑर्डर
                            </button>
                            <button onclick="window.appFunctions.switchView('admin')" class="px-3 py-1 text-sm rounded-full transition-colors ${getButtonClass('admin')}">
                                रेस्टोरेंट
                            </button>
                            <!-- लॉगआउट बटन: यूज़र के साइन इन होने पर दिखाई देता है -->
                            ${userId ? `<button onclick="window.appFunctions.handleLogout()" class="px-3 py-1 text-sm rounded-full transition-colors bg-red-500 text-white font-bold hover:bg-red-600">
                                लॉगआउट
                            </button>` : ''}
                        </div>
                    </div>
                </nav>
            `;
        };

        const renderCustomerMenu = () => {
            if (menuItems.length === 0) return showMessage('मेनू लोड हो रहा है या कोई आइटम उपलब्ध नहीं है।', 'bg-gray-500');

            const totalItems = Object.values(cart).reduce((sum, item) => sum + item.quantity, 0);
            const totalPrice = Object.values(cart).reduce((sum, item) => sum + (item.price * item.quantity), 0);
            
            // --- मेनू आइटम को कैटेगरी द्वारा ग्रुप करने का लॉजिक ---
            const groupedMenu = menuItems.reduce((acc, item) => {
                const category = item.category || 'Other'; // यदि 'category' मिसिंग है तो डिफ़ॉल्ट कैटेगरी
                if (!acc[category]) {
                    acc[category] = [];
                }
                acc[category].push(item);
                return acc;
            }, {});

            const categories = Object.keys(groupedMenu).sort(); // कैटेगरी के नाम प्राप्त करें और सॉर्ट करें
            
            let menuHtml = '';

            // कैटेगरी और उनके आइटम रेंडर करें
            categories.forEach(category => {
                // कैटेगरी हेडर
                menuHtml += `
                    <h3 class="text-2xl font-extrabold text-accent mt-6 mb-4 border-b-2 border-primary pb-1">${category}</h3>
                `;

                // कैटेगरी के अंदर आइटम
                menuHtml += groupedMenu[category].map(item => `
                    <div class="bg-white p-4 rounded-xl shadow-lg flex justify-between items-center mb-4 border border-gray-100 transition-all hover:shadow-xl">
                        <div class="flex-1">
                            <h4 class="text-lg font-semibold text-gray-800">${item.name}</h4>
                            <p class="text-primary font-bold">₹${item.price}</p>
                            <p class="text-sm text-gray-500">${item.description}</p>
                        </div>
                        <div class="flex items-center space-x-2 ml-4">
                            <button onclick="window.appFunctions.updateCart('${item.id}', -1)" class="bg-red-100 text-primary font-bold text-lg w-8 h-8 rounded-full flex items-center justify-center hover:bg-primary hover:text-white transition duration-200">
                                -
                            </button>
                            <span class="font-bold text-gray-700 w-6 text-center">
                                ${cart[item.id] ? cart[item.id].quantity : 0}
                            </span>
                            <button onclick="window.appFunctions.updateCart('${item.id}', 1, '${item.name}', ${item.price})" class="bg-accent text-white font-bold text-lg w-8 h-8 rounded-full flex items-center justify-center hover:bg-green-700 transition duration-200">
                                +
                            </button>
                        </div>
                    </div>
                `).join('');
            });
            // --- कैटेगरी द्वारा ग्रुप करने का लॉजिक समाप्त ---


            return `
                <div class="max-w-xl mx-auto p-4 pt-16">
                    <h2 class="text-3xl font-bold text-gray-800 mb-6 border-b pb-2">मेनू</h2>

                    <div>${menuHtml}</div>

                    <!-- कार्ट सारांश/चेकआउट -->
                    <div id="cart-summary" class="fixed bottom-0 left-0 right-0 p-4 bg-primary text-white shadow-2xl transition-transform duration-300 ${totalItems > 0 ? '' : 'translate-y-full'}">
                        <div class="flex justify-between items-center max-w-xl mx-auto">
                            <div class="text-sm">
                                <span class="font-bold">${totalItems} आइटम्स</span> | कुल: <span class="text-xl font-extrabold">₹${totalPrice}</span>
                            </div>
                            <!-- 'placeOrder' अब मोडल दिखाएगा -->
                            <button onclick="window.appFunctions.placeOrder()" class="bg-white text-primary px-5 py-2 font-bold rounded-lg shadow-xl hover:bg-secondary transition duration-200">
                                ऑर्डर प्लेस करें (COD)
                            </button>
                        </div>
                    </div>
                </div>
                <div class="h-20"></div> <!-- फिक्स्ड कार्ट के लिए स्पेसर -->
            `;
        };

        const renderCustomerOrders = () => {
            if (!userId) return showMessage('कृपया प्रतीक्षा करें, प्रमाणीकरण (authentication) प्रगति पर है...', 'bg-gray-500');
            if (customerOrders.length === 0) return showMessage('आपके पास कोई पिछला ऑर्डर नहीं है।', 'bg-accent');

            const ordersHtml = customerOrders.map(order => `
                <div class="bg-white p-4 rounded-xl shadow-lg mb-4 border-l-4 ${order.status === 'Delivered' ? 'border-green-500' : 'border-primary'}">
                    <div class="flex justify-between items-center mb-2">
                        <h3 class="text-lg font-bold text-gray-800">ऑर्डर #${order.id.substring(0, 8)}</h3>
                        <span class="px-3 py-1 text-xs font-semibold rounded-full ${order.status === 'Delivered' ? 'bg-green-100 text-green-700' : 'bg-primary text-white'}">
                            ${order.status}
                        </span>
                    </div>
                    <p class="text-sm text-gray-600 mb-2">समय: ${new Date(order.createdAt.seconds * 1000).toLocaleString('en-US')}</p>
                    <p class="text-base font-semibold text-primary">कुल: ₹${order.totalPrice}</p>
                    <p class="text-sm font-semibold text-accent mt-1">भुगतान विधि: ${order.paymentMethod || 'N/A'}</p>
                    <ul class="list-disc list-inside mt-2 text-sm text-gray-700">
                        ${order.items.map(item => `<li>${item.name} x ${item.quantity}</li>`).join('')}
                    </ul>
                    <p class="text-xs mt-2 text-gray-500">पता: ${order.address || 'पता प्रदान नहीं किया गया'}</p>
                </div>
            `).join('');

            return `
                <div class="max-w-xl mx-auto p-4 pt-16">
                    <h2 class="text-3xl font-bold text-gray-800 mb-6 border-b pb-2">मेरे ऑर्डर</h2>
                    <div>${ordersHtml}</div>
                </div>
            `;
        };

        const renderAdminView = () => {
            if (restaurantOrders.length === 0) return showMessage('कोई नया ऑर्डर नहीं है! 🎉', 'bg-accent');
            
            // स्टेटस और फिर समय (सबसे नया पहले) के अनुसार सॉर्ट करें
            const sortedOrders = [...restaurantOrders].sort((a, b) => {
                const statusOrder = orderStatusKeys;
                const statusDiff = statusOrder.indexOf(a.status) - statusOrder.indexOf(b.status);
                if (statusDiff !== 0) return statusDiff;
                // यदि स्टेटस समान है, तो सबसे नया पहले सॉर्ट करें
                return (b.createdAt?.seconds || 0) - (a.createdAt?.seconds || 0);
            });

            const ordersHtml = sortedOrders.map(order => `
                <div class="bg-white p-4 rounded-xl shadow-lg mb-4 border-l-8 ${order.status === 'New Order' ? 'border-red-500' : 'border-blue-500'}">
                    <div class="flex justify-between items-start mb-3">
                        <div>
                            <h3 class="text-xl font-extrabold text-gray-900">ऑर्डर #${order.id.substring(0, 8)}</h3>
                            <!-- ग्राहक का नाम प्रदर्शित करें -->
                            <p class="text-sm text-gray-600 font-medium">ग्राहक: ${order.customerName || 'N/A'}</p>
                            <p class="text-sm text-gray-500">समय: ${new Date(order.createdAt.seconds * 1000).toLocaleString('en-US')}</p>
                        </div>
                        <span class="px-3 py-1 text-xs font-semibold rounded-full ${order.status === 'New Order' ? 'bg-red-500 text-white animate-pulse' : 'bg-gray-200 text-gray-700'}">
                            ${order.status}
                        </span>
                    </div>

                    <ul class="list-disc list-inside mt-2 text-base text-gray-700 font-medium">
                        ${order.items.map(item => `<li>${item.name} x ${item.quantity}</li>`).join('')}
                    </ul>

                    <div class="mt-4 border-t pt-3">
                        <p class="text-base font-semibold text-primary">कुल राशि: ₹${order.totalPrice}</p>
                        <p class="text-sm text-gray-600 font-bold mt-1">भुगतान विधि: ${order.paymentMethod || 'N/A'}</p>
                        <p class="text-sm text-gray-600 font-bold mt-1">पता: ${order.address || 'ग्राहक का पता नहीं दिया गया'}</p>
                    </div>

                    <!-- स्टेटस अपडेट ड्रॉपडाउन -->
                    <div class="mt-4 flex items-center space-x-2">
                        <label for="status-select-${order.id}" class="text-sm font-medium text-gray-700">स्टेटस बदलें:</label>
                        <select id="status-select-${order.id}" class="block w-full p-2 border border-gray-300 bg-white rounded-md shadow-sm focus:ring-accent focus:border-accent"
                            onchange="window.appFunctions.updateOrderStatus('${order.id}', this.value)">
                            ${orderStatusKeys.map(status => `
                                <option value="${status}" ${order.status === status ? 'selected' : ''}>
                                    ${status}
                                </option>
                            `).join('')}
                        </select>
                    </div>
                </div>
            `).join('');

            const pendingCount = restaurantOrders.filter(o => o.status !== 'Delivered' && o.status !== 'Cancelled').length;

            return `
                <div class="max-w-xl mx-auto p-4 pt-16">
                    <h2 class="text-3xl font-bold text-primary mb-6 border-b-4 border-primary pb-2">सभी ऑर्डर (रेस्टोरेंट डैशबोर्ड)</h2>
                    <p class="text-gray-600 mb-4">कुल लंबित ऑर्डर: ${pendingCount}</p>
                    <div>${ordersHtml}</div>
                </div>
            `;
        };
        
        const renderApp = () => {
            const root = document.getElementById('app-root');
            if (!isAuthReady) {
                root.innerHTML = showMessage('एप्लिकेशन लोड हो रहा है, कृपया प्रतीक्षा करें...', 'bg-gray-500');
                return;
            }

            let content = '';
            switch (currentView) {
                case 'home': // नया होम व्यू
                    content = renderHome();
                    break;
                case 'customer-menu':
                    content = renderCustomerMenu();
                    break;
                case 'customer-orders':
                    content = renderCustomerOrders();
                    break;
                case 'admin':
                    content = renderAdminView();
                    break;
                default:
                    content = renderHome();
            }

            root.innerHTML = renderNavBar() + content;
        };

        // --- GPS और मोडल लॉजिक ---

        const showAddressModal = () => {
            const modal = document.getElementById('address-modal');
            const addressInput = document.getElementById('manual-address-input');
            const gpsStatus = document.getElementById('gps-status');
            
            // मोडल दिखाने पर इनपुट को क्लियर करें और स्टेटस रीसेट करें
            addressInput.value = '';
            gpsStatus.innerHTML = '';
            
            if (modal) {
                modal.classList.remove('hidden');
                document.body.classList.add('overflow-hidden');
            }
        };

        const hideAddressModal = () => {
            const modal = document.getElementById('address-modal');
            if (modal) {
                modal.classList.add('hidden');
                document.body.classList.remove('overflow-hidden');
            }
        };

        const attemptGPSLocation = () => {
            const gpsStatus = document.getElementById('gps-status');
            const addressInput = document.getElementById('manual-address-input');
            
            if (!navigator.geolocation) {
                gpsStatus.innerHTML = '<span class="text-red-500">माफ़ करना, आपका ब्राउज़र GPS/जियोलोकेशन का समर्थन नहीं करता है।</span>';
                return;
            }

            gpsStatus.innerHTML = '<span class="text-blue-500">GPS लोकेशन प्राप्त कर रहा है...</span>';

            navigator.geolocation.getCurrentPosition((position) => {
                const lat = position.coords.latitude;
                const lon = position.coords.longitude;
                const accuracy = position.coords.accuracy;

                // पते के रूप में Lat/Lon को टेक्स्टएरिया में भरें
                addressInput.value = `GPS लोकेशन: Lat: ${lat.toFixed(6)}, Lon: ${lon.toFixed(6)}. (सटीकता: ${accuracy} मीटर)`;
                gpsStatus.innerHTML = `<span class="text-accent font-bold">सफलता!</span> GPS डेटा प्राप्त हुआ। (${lat.toFixed(2)}, ${lon.toFixed(2)})`;
                
                // यूज़र को पता दें कि वे अभी भी मैन्युअल रूप से सुधार कर सकते हैं
                document.getElementById('address-message').innerHTML = 'GPS डेटा भर दिया गया है। आप पुष्टि कर सकते हैं या मैन्युअल रूप से पता बदल सकते हैं।';

            }, (error) => {
                // GPS त्रुटि हैंडलिंग
                let errorMessage = 'GPS का उपयोग करने में विफल रहा।';
                switch(error.code) {
                    case error.PERMISSION_DENIED:
                        errorMessage = 'GPS एक्सेस अस्वीकृत। कृपया ब्राउज़र सेटिंग्स में अनुमति दें।';
                        break;
                    case error.POSITION_UNAVAILABLE:
                        errorMessage = 'GPS लोकेशन अनुपलब्ध है।';
                        break;
                    case error.TIMEOUT:
                        errorMessage = 'GPS अनुरोध का समय समाप्त हो गया।';
                        break;
                    default:
                        errorMessage = 'GPS त्रुटि: ' + error.message;
                }
                gpsStatus.innerHTML = `<span class="text-red-500">${errorMessage}</span>`;
            }, {
                enableHighAccuracy: true,
                timeout: 5000,
                maximumAge: 0
            });
        };

        // --- डेटा इंटरेक्शन (Firestore) फ़ंक्शन ---

        const fetchMenu = () => {
            if (!db) return;
            const q = collection(db, menuItemCollectionPath);
            // मेनू परिवर्तनों के लिए रियल-टाइम श्रोता
            onSnapshot(q, (snapshot) => {
                menuItems = snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
                log("मेनू रियल-टाइम में फ़ेच किया गया।", menuItems);
                // केवल तभी पुनः रेंडर करें जब हम मेनू-संबंधित व्यू पर हों
                if (currentView === 'customer-menu') renderApp();
            }, (error) => {
                console.error("मेनू फ़ेच करने में त्रुटि: ", error);
            });
        };

        const setupOrderListeners = () => {
            if (!db || !userId) return;

            // 1. रेस्टोरेंट (एडमिन) ऑर्डर श्रोता (सभी ऑर्डर)
            const restaurantQ = query(collection(db, orderCollectionPath));
            onSnapshot(restaurantQ, (snapshot) => {
                restaurantOrders = snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
                log("एडमिन के लिए सभी रेस्टोरेंट ऑर्डर फ़ेच किए गए।", restaurantOrders.length);
                if (currentView === 'admin') renderApp();
            }, (error) => {
                console.error("एडमिन के लिए सभी ऑर्डर फ़ेच करने में त्रुटि: ", error);
            });

            // 2. ग्राहक के अपने ऑर्डर श्रोता
            const customerQ = query(collection(db, orderCollectionPath), where("customerId", "==", userId));
            onSnapshot(customerQ, (snapshot) => {
                customerOrders = snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
                // निर्माण समय (नया पहले) के अनुसार सॉर्ट करें
                customerOrders.sort((a, b) => (b.createdAt?.seconds || 0) - (a.createdAt?.seconds || 0));
                log("ग्राहक के ऑर्डर फ़ेच किए गए।", customerOrders.length);
                if (currentView === 'customer-orders') renderApp();
            }, (error) => {
                console.error("ग्राहक के ऑर्डर फ़ेच करने में त्रुटि: ", error);
            });
        };

        const updateCart = (itemId, change, name = '', price = 0) => {
            const currentQuantity = cart[itemId] ? cart[itemId].quantity : 0;
            const newQuantity = currentQuantity + change;

            if (newQuantity <= 0) {
                delete cart[itemId];
            } else {
                cart[itemId] = {
                    name: name,
                    price: price,
                    quantity: newQuantity
                };
            }
            renderApp();
        };

        // यह फ़ंक्शन अब केवल मोडल दिखाता है
        const placeOrder = () => {
            const itemsInCart = Object.values(cart).filter(item => item.quantity > 0);
            if (itemsInCart.length === 0) {
                // मोडल दिखाने के बजाय एक इन-ऐप मैसेज दिखा सकते हैं, लेकिन फ़िलहाल केवल वार्निंग लॉग करते हैं।
                console.warn('कार्ट खाली है, ऑर्डर प्लेस नहीं किया जा सकता।'); 
                return;
            }
            // मोडल दिखाएं
            showAddressModal();
        };

        // मोडल से ऑर्डर डेटा प्राप्त करने और Firestore में सहेजने के लिए नया फ़ंक्शन
        const finalizeOrder = async () => {
            const itemsInCart = Object.values(cart).filter(item => item.quantity > 0);
            const addressInput = document.getElementById('manual-address-input');
            const address = addressInput.value.trim();

            if (!address) {
                document.getElementById('gps-status').innerHTML = '<span class="text-red-500">कृपया डिलीवरी पता दर्ज करें या GPS का उपयोग करें।</span>';
                return;
            }

            hideAddressModal(); // ऑर्डर शुरू करने से पहले मोडल छिपाएँ

            const paymentMethod = 'Cash on Delivery (COD)'; // COD को स्पष्ट रूप से सेट करें
            const totalPrice = itemsInCart.reduce((sum, item) => sum + (item.price * item.quantity), 0);
            
            const newOrder = {
                customerId: userId,
                customerName: userName, // कैप्चर किया गया यूज़रनाम शामिल है
                items: itemsInCart,
                totalPrice: totalPrice,
                address: address, // मैन्युअल या GPS डेटा
                status: 'New Order', // प्रारंभिक स्टेटस
                paymentMethod: paymentMethod, // COD जोड़ा गया
                createdAt: serverTimestamp()
            };

            try {
                await addDoc(collection(db, orderCollectionPath), newOrder);
                log("नया ऑर्डर सफलतापूर्वक प्लेस किया गया।");
                cart = {}; // ऑर्डर प्लेस करने के बाद कार्ट क्लियर करें
                currentView = 'customer-orders'; // ऑर्डर ट्रैकिंग व्यू पर स्विच करें
                renderApp();
                console.log('ऑर्डर सफलतापूर्वक प्लेस किया गया।');
            } catch (error) {
                console.error("ऑर्डर प्लेस करने में त्रुटि: ", error);
                console.error('ऑर्डर प्लेस करने में विफल रहा।');
            }
        };
        
        const updateOrderStatus = async (orderId, newStatus) => {
            if (!orderStatusKeys.includes(newStatus)) {
                log('अवैध स्टेटस का प्रयास किया गया।', newStatus);
                return;
            }
            try {
                const orderRef = doc(db, orderCollectionPath, orderId);
                await updateDoc(orderRef, { status: newStatus });
                log(`ऑर्डर ${orderId} स्टेटस अपडेट होकर ${newStatus} हो गया`);
                // onSnapshot श्रोता (listener) स्वचालित रूप से व्यू को पुनः रेंडर करेगा
            } catch (error) {
                console.error("ऑर्डर स्टेटस अपडेट करने में त्रुटि: ", error);
                console.error('ऑर्डर स्टेटस अपडेट करने में विफल रहा।');
            }
        };

        const switchView = (view) => {
            currentView = view;
            renderApp();
        };

        // --- प्रारंभिक मेनू सेटअप (पहले रन के लिए) ---
        const setupInitialMenu = async () => {
            if (!db) return;
            // जांचें कि क्या मेनू कलेक्शन खाली है
            const menuRef = collection(db, menuItemCollectionPath);
            const docRef = doc(menuRef, "burger-1");
            const docSnap = await getDoc(docRef);
            
            if (!docSnap.exists()) {
                log("प्रारंभिक मेनू आइटम सेट कर रहा है...");
                try {
                    // कैटेगरी के साथ मेनू आइटम
                    await setDoc(docRef, { name: "Aloo Tikki Burger", description: "Delicious potato patty and fresh vegetables.", price: 99, category: "Burgers & Sandwiches" });
                    await setDoc(doc(menuRef, "pizza-2"), { name: "Paneer Tikka Pizza", description: "Crispy pizza with paneer tikka topping.", price: 249, category: "Pizzas" });
                    await setDoc(doc(menuRef, "fries-3"), { name: "Peri-Peri Fries", description: "Tangy and spicy flavored fries.", price: 79, category: "Sides" });
                    await setDoc(doc(menuRef, "shake-4"), { name: "Chocolate Shake", description: "Cold and thick chocolate shake.", price: 129, category: "Beverages" });
                    await setDoc(doc(menuRef, "vadapav-5"), { name: "Mumbai Vada Pav", description: "Spicy potato fritter in a soft bun.", price: 49, category: "Burgers & Sandwiches" });
                    await setDoc(doc(menuRef, "soda-6"), { name: "Masala Soda", description: "Refreshing Indian spiced soda.", price: 69, category: "Beverages" });
                    await setDoc(doc(menuRef, "samosa-7"), { name: "Veg Samosa (2 Pcs)", description: "Crispy fried pastry with spiced potatoes.", price: 60, category: "Sides" });
                    await setDoc(doc(menuRef, "margherita-8"), { name: "Classic Margherita", description: "Traditional pizza with mozzarella and basil.", price: 199, category: "Pizzas" });
                    
                    log("प्रारंभिक मेनू सेटअप पूरा हुआ।");
                } catch (e) {
                    console.error("प्रारंभिक मेनू सेट करने में त्रुटि: ", e);
                }
            } else {
                 log("प्रारंभिक मेनू पहले से मौजूद है।");
            }
        };
        
        // --- मुख्य निष्पादन (Main Execution) ---
        window.onload = () => {
            initializeFirebase().then(() => {
                // सभी यूज़र्स के लिए मेनू तुरंत फ़ेच करें
                fetchMenu();
                // प्रारंभिक मेनू सेटअप चलाएँ
                setupInitialMenu();
                // यूज़र के साइन इन होने पर ऑर्डर श्रोता (listeners) चलेंगे
            });
        };

        // HTML/इनलाइन इवेंट्स के लिए फ़ंक्शन को वैश्विक स्तर पर उजागर करें
        window.appFunctions = {
            switchView,
            updateCart,
            placeOrder,
            finalizeOrder, // नया ऑर्डर फाइनल करने का फ़ंक्शन
            updateOrderStatus,
            handleLoginClick,
            startAppSession,
            handleLogout,
            showAddressModal,
            hideAddressModal,
            attemptGPSLocation // नया GPS फ़ंक्शन
        };
    </script>
</head>
<body class="bg-gray-50 font-sans min-h-screen">
    <div id="app-root">
        <!-- सामग्री (Content) यहाँ JavaScript द्वारा रेंडर की जाएगी -->
    </div>

    <!-- डिलीवरी एड्रेस और GPS इनपुट के लिए कस्टम मोडल (Address Modal) -->
    <div id="address-modal" class="fixed inset-0 z-50 hidden bg-black bg-opacity-50 flex items-center justify-center p-4">
        <div class="bg-white rounded-xl shadow-2xl p-6 w-full max-w-sm">
            <h3 class="text-xl font-bold text-gray-800 mb-4">डिलीवरी पता</h3>
            <p class="text-sm text-gray-600 mb-4" id="address-message">कृपया अपना पता मैन्युअल रूप से दर्ज करें, या GPS का उपयोग करें। (केवल COD)</p>

            <!-- Address Input -->
            <textarea id="manual-address-input" rows="3" placeholder="पूरा पता दर्ज करें..."
                class="w-full px-3 py-2 border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-accent mb-4"></textarea>

            <!-- GPS Status/Button -->
            <button id="gps-button" onclick="window.appFunctions.attemptGPSLocation()"
                    class="w-full bg-blue-500 text-white py-2 rounded-lg font-bold shadow-md hover:bg-blue-600 transition duration-200 mb-3 flex items-center justify-center">
                <svg class="w-5 h-5 mr-2" fill="none" stroke="currentColor" viewBox="0 0 24 24" xmlns="http://www.w3.org/2000/svg"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M17.657 16.657L13.414 20.899a1.002 1.002 0 01-1.414 0l-4.243-4.242m9.192-8.192a5 5 0 00-7.07 0c-1.56 1.56-1.56 4.09 0 5.65l7.07 7.07 7.07-7.07c1.56-1.56 1.56-4.09 0-5.65zM12 11a1 1 0 100-2 1 1 0 000 2z"></path></svg>
                GPS लोकेशन का उपयोग करें
            </button>
            <div id="gps-status" class="text-xs text-center text-gray-500 mb-4"></div>

            <!-- Action Buttons -->
            <div class="flex space-x-3">
                <button onclick="window.appFunctions.hideAddressModal()"
                        class="flex-1 bg-gray-300 text-gray-800 py-2 rounded-lg font-bold hover:bg-gray-400 transition duration-200">
                    कैंसिल करें
                </button>
                <button id="confirm-order-button" onclick="window.appFunctions.finalizeOrder()"
                        class="flex-1 bg-accent text-white py-2 rounded-lg font-bold hover:bg-green-700 transition duration-200">
                    ऑर्डर की पुष्टि करें
                </button>
            </div>
        </div>
    </div>
    
    <!-- यूज़र/मालिक के लिए निर्देश -->
    <div class="fixed bottom-0 left-0 right-0 bg-yellow-100 p-3 text-xs text-center text-gray-700 border-t border-yellow-300 z-50">
        <p class="font-bold">

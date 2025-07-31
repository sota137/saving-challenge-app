import React, { useState, useEffect, createContext, useContext } from 'react';
import { initializeApp } from 'firebase/app';
import { getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged } from 'firebase/auth';
import { getFirestore, doc, setDoc, onSnapshot, collection, getDoc } from 'firebase/firestore';
import { LineChart, Line, XAxis, YAxis, CartesianGrid, Tooltip, Legend, ResponsiveContainer } from 'recharts';

// Firebase configuration and app ID are provided globally in the Canvas environment
// Firebase設定とアプリIDはCanvas環境でグローバルに提供されます
const firebaseConfig = typeof __firebase_config !== 'undefined' ? JSON.parse(__firebase_config) : {};
const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';
const initialAuthToken = typeof __initial_auth_token !== 'undefined' ? __initial_auth_token : null;

// Create a context for Firebase services
// Firebaseサービスのためのコンテキストを作成
const FirebaseContext = createContext(null);

// Custom hook to use Firebase services
// Firebaseサービスを使用するためのカスタムフック
const useFirebase = () => useContext(FirebaseContext);

function App() {
    const [db, setDb] = useState(null);
    const [auth, setAuth] = useState(null);
    const [userId, setUserId] = useState(null);
    const [isAuthReady, setIsAuthReady] = useState(false);
    // Retrieve selected player from localStorage for persistence across sessions
    // セッション間で選択されたプレイヤーを保持するため、localStorageから取得
    const [selectedPlayer, setSelectedPlayer] = useState(localStorage.getItem('selectedPlayer') || null); // 'sota' or 'renma'
    // Stores daily expenses: { 'YYYY-MM-DD': { sotaExpenses: [], renmaExpenses: [] } }
    // 日々の支出を格納: { 'YYYY-MM-DD': { sotaExpenses: [], renmaExpenses: [] } }
    const [expenses, setExpenses] = useState({});
    const [loading, setLoading] = useState(true);
    const [error, setError] = useState(null);
    const [currentExpenseAmount, setCurrentExpenseAmount] = useState('');
    // Changed from category to description for free-form input
    // 自由入力のためにカテゴリから説明に変更
    const [currentExpenseDescription, setCurrentExpenseDescription] = useState('');
    // Default date to today, formatted as YYYY-MM-DD
    // デフォルトの日付を今日に設定し、YYYY-MM-DD形式でフォーマット
    const [selectedDate, setSelectedDate] = useState(new Date().toISOString().split('T')[0]);

    // Monthly goals
    // 月間目標
    const sotaMonthlyGoal = 80000;
    const renmaMonthlyGoal = 120000;

    // Initialize Firebase and handle authentication
    // Firebaseを初期化し、認証を処理
    useEffect(() => {
        const initializeFirebase = async () => {
            try {
                const app = initializeApp(firebaseConfig);
                const firestore = getFirestore(app);
                const authService = getAuth(app);

                setDb(firestore);
                setAuth(authService);

                // Sign in with custom token or anonymously
                // カスタムトークンまたは匿名でサインイン
                if (initialAuthToken) {
                    await signInWithCustomToken(authService, initialAuthToken);
                } else {
                    await signInAnonymously(authService);
                }

                // Listen for auth state changes to get the user ID
                // ユーザーIDを取得するために認証状態の変化をリッスン
                const unsubscribe = onAuthStateChanged(authService, (user) => {
                    if (user) {
                        setUserId(user.uid);
                        setIsAuthReady(true);
                    } else {
                        setUserId(null);
                        setIsAuthReady(true); // Still ready, but no user
                    }
                    setLoading(false);
                });

                return () => unsubscribe(); // Cleanup auth listener
            } catch (err) {
                console.error("Firebase initialization or authentication error:", err);
                setError("Firebaseの初期化または認証に失敗しました。");
                setLoading(false);
            }
        };

        initializeFirebase();
    }, []); // Run only once on component mount

    // Listen for expense data changes from Firestore
    // Firestoreからの支出データの変更をリッスン
    useEffect(() => {
        if (!db || !isAuthReady) return; // Ensure Firebase is initialized and auth is ready

        // Reference to the public expenses collection
        // 公開支出コレクションへの参照
        const expensesCollectionRef = collection(db, `artifacts/${appId}/public/data/savingChallenge`);

        // Set up a real-time listener for the expenses collection
        // 支出コレクションのリアルタイムリスナーを設定
        const unsubscribe = onSnapshot(expensesCollectionRef, (snapshot) => {
            const newExpenses = {};
            snapshot.forEach(doc => {
                newExpenses[doc.id] = doc.data(); // Store data with document ID as key (date)
            });
            setExpenses(newExpenses); // Update state with fetched expenses
        }, (err) => {
            console.error("Error fetching expenses:", err);
            setError("支出データの取得に失敗しました。");
        });

        return () => unsubscribe(); // Cleanup snapshot listener on component unmount
    }, [db, isAuthReady]); // Re-run if db or isAuthReady changes

    // Handle player selection and store it in localStorage
    // プレイヤーの選択を処理し、localStorageに保存
    const handlePlayerSelect = (player) => {
        setSelectedPlayer(player);
        localStorage.setItem('selectedPlayer', player); // Persist selection
    };

    // Handle expense submission to Firestore
    // Firestoreへの支出の送信を処理
    const handleAddExpense = async () => {
        if (!db || !userId || !selectedPlayer) {
            setError("データベースまたはユーザー情報が利用できません。");
            return;
        }

        const amount = parseFloat(currentExpenseAmount);
        // Validate input: must be a non-negative number
        // 入力検証: 負でない数値であること
        if (isNaN(amount) || amount < 0) {
            setError("有効な数値を入力してください。");
            return;
        }
        // Validate description: must not be empty
        // 説明の検証: 空であってはならない
        if (!currentExpenseDescription.trim()) {
            setError("支出内容を入力してください。");
            return;
        }

        // Document reference for the selected date
        // 選択された日付のドキュメント参照
        const docRef = doc(db, `artifacts/${appId}/public/data/savingChallenge`, selectedDate);

        try {
            // Fetch current document to get existing expenses for the day
            // 既存のその日の支出を取得するために現在のドキュメントをフェッチ
            const docSnap = await getDoc(docRef);
            let currentDailyData = docSnap.exists() ? docSnap.data() : {};

            // Get the player's expense array for the selected date
            // 選択された日付のプレイヤーの支出配列を取得
            const playerExpensesKey = `${selectedPlayer}Expenses`; // e.g., 'sotaExpenses' or 'renmaExpenses'
            const existingPlayerExpenses = currentDailyData[playerExpensesKey] || [];

            // Add new expense to the array, using description instead of category
            // 配列に新しい支出を追加、カテゴリの代わりに説明を使用
            const newPlayerExpenses = [...existingPlayerExpenses, { amount, description: currentExpenseDescription.trim(), timestamp: Date.now() }];

            // Update the document with the new expense array
            // 新しい支出配列でドキュメントを更新
            await setDoc(docRef, {
                date: selectedDate,
                [playerExpensesKey]: newPlayerExpenses,
                [`${selectedPlayer}UserId`]: userId, // Store user ID for the player who made the entry
            }, { merge: true }); // Merge to update specific fields without overwriting others

            setCurrentExpenseAmount(''); // Clear amount input field after successful submission
            setCurrentExpenseDescription(''); // Clear description input field after successful submission
            setError(null); // Clear any previous errors
        } catch (e) {
            console.error("Error adding expense: ", e);
            setError("支出の追加に失敗しました。");
        }
    };

    // Helper to get total daily expense for a player
    // プレイヤーの1日の合計支出を取得するヘルパー
    const getDailyTotalExpense = (player, date) => {
        const dailyData = expenses[date];
        if (!dailyData) return 0;
        const playerExpensesKey = `${player}Expenses`;
        const playerExpenses = dailyData[playerExpensesKey] || [];
        return playerExpenses.reduce((sum, item) => sum + (item.amount || 0), 0);
    };

    // Calculate daily winner based on the rules
    // ルールに基づいて日ごとの勝者を計算
    const getDailyWinner = (sotaTotalExp, renmaTotalExp) => {
        // If data is missing for either player, indicate insufficient data
        // いずれかのプレイヤーのデータが不足している場合、データ不足を示す
        if (sotaTotalExp === undefined || renmaTotalExp === undefined) return 'データ不足';

        const sotaAdjusted = sotaTotalExp * 1.5; // Sota's expense adjusted by 1.5 times

        if (renmaTotalExp > sotaAdjusted) {
            return '新井颯太の勝ち';
        } else if (renmaTotalExp === sotaAdjusted) {
            return '引き分け';
        } else { // renmaTotalExp < sotaAdjusted
            return '高橋蓮馬の勝ち';
        }
    };

    // Calculate cumulative results (total wins and draws for daily)
    // 累計結果（日ごとの合計勝利数と引き分け数）を計算
    const calculateCumulativeDailyResults = () => {
        let sotaWins = 0;
        let renmaWins = 0;
        let draws = 0;

        // Sort dates to ensure consistent cumulative calculation
        // 累計計算の一貫性を保つため、日付をソート
        const sortedDates = Object.keys(expenses).sort();

        sortedDates.forEach(date => {
            const sotaExp = getDailyTotalExpense('sota', date);
            const renmaExp = getDailyTotalExpense('renma', date);

            const winner = getDailyWinner(sotaExp, renmaExp);
            if (winner === '新井颯太の勝ち') {
                sotaWins++;
            } else if (winner === '高橋蓮馬の勝ち') {
                renmaWins++;
            } else if (winner === '引き分け') {
                draws++;
            }
        });

        return { sotaWins, renmaWins, draws };
    };

    const cumulativeDailyResults = calculateCumulativeDailyResults();

    // Calculate overall total expenses for each player
    // 各プレイヤーの全体総支出を計算
    const calculateOverallTotalExpenses = () => {
        let sotaOverallTotal = 0;
        let renmaOverallTotal = 0;

        Object.keys(expenses).forEach(date => {
            sotaOverallTotal += getDailyTotalExpense('sota', date);
            renmaOverallTotal += getDailyTotalExpense('renma', date);
        });

        return { sotaOverallTotal, renmaOverallTotal };
    };

    const { sotaOverallTotal, renmaOverallTotal } = calculateOverallTotalExpenses();

    // Determine overall winner
    // 全体の勝者を決定
    const getOverallWinner = () => {
        if (sotaOverallTotal === 0 && renmaOverallTotal === 0) return 'データ不足';

        const sotaAdjustedOverall = sotaOverallTotal * 1.5;

        if (renmaOverallTotal > sotaAdjustedOverall) {
            return '新井颯太の勝ち';
        } else if (renmaOverallTotal === sotaAdjustedOverall) {
            return '引き分け';
        } else { // renmaOverallTotal < sotaAdjustedOverall
            return '高橋蓮馬の勝ち';
        }
    };

    const overallWinner = getOverallWinner();

    // Prepare data for the cumulative expense graph
    // 累積支出グラフのデータを準備
    const getChartData = () => {
        const sortedDates = Object.keys(expenses).sort();
        const chartData = [];
        let sotaCumulative = 0;
        let renmaCumulative = 0;

        sortedDates.forEach(date => {
            const sotaDaily = getDailyTotalExpense('sota', date);
            const renmaDaily = getDailyTotalExpense('renma', date);

            sotaCumulative += sotaDaily;
            renmaCumulative += renmaDaily;

            chartData.push({
                date: date.substring(5), // Show MM-DD
                新井颯太: sotaCumulative,
                高橋蓮馬: renmaCumulative,
                '新井颯太の1.5倍': sotaCumulative * 1.5,
            });
        });
        return chartData;
    };

    const chartData = getChartData();

    // Display loading state
    // ローディング状態を表示
    if (loading) {
        return (
            <div className="flex items-center justify-center min-h-screen bg-gray-100">
                <div className="text-xl font-semibold text-gray-700">読み込み中...</div>
            </div>
        );
    }

    // Display error message
    // エラーメッセージを表示
    if (error && !loading) { // Show error only if not loading
        return (
            <div className="flex items-center justify-center min-h-screen bg-red-100 text-red-800 p-4 rounded-lg">
                エラー: {error}
            </div>
        );
    }

    return (
        <FirebaseContext.Provider value={{ db, auth, userId, isAuthReady }}>
            <div className="min-h-screen bg-gradient-to-br from-blue-100 to-purple-100 p-4 sm:p-6 lg:p-8 font-inter antialiased">
                <div className="max-w-4xl mx-auto bg-white rounded-2xl shadow-xl p-6 sm:p-8">
                    <h1 className="text-3xl sm:text-4xl font-extrabold text-center text-gray-800 mb-6">
                        💰 2025年8月度 節約対決 💰
                    </h1>
                    <p className="text-center text-gray-600 mb-8 text-lg">
                        より少ない支出で効率的な節約を競います！
                    </p>

                    {/* Player Selection Section */}
                    {/* プレイヤー選択セクション */}
                    {!selectedPlayer && (
                        <div className="text-center mb-8">
                            <h2 className="text-2xl font-bold text-gray-700 mb-4">プレイヤーを選択してください</h2>
                            <div className="flex flex-col sm:flex-row justify-center gap-4">
                                <button
                                    onClick={() => handlePlayerSelect('sota')}
                                    className="bg-blue-600 hover:bg-blue-700 text-white font-bold py-3 px-6 rounded-xl shadow-lg transform transition duration-300 hover:scale-105"
                                >
                                    新井颯太
                                </button>
                                <button
                                    onClick={() => handlePlayerSelect('renma')}
                                    className="bg-purple-600 hover:bg-purple-700 text-white font-bold py-3 px-6 rounded-xl shadow-lg transform transition duration-300 hover:scale-105"
                                >
                                    高橋蓮馬
                                </button>
                            </div>
                        </div>
                    )}

                    {/* Main Application Content after Player Selection */}
                    {/* プレイヤー選択後のメインアプリケーションコンテンツ */}
                    {selectedPlayer && (
                        <>
                            <div className="text-center mb-6">
                                <p className="text-xl font-semibold text-gray-700">
                                    あなたは: <span className="text-blue-600">{selectedPlayer === 'sota' ? '新井颯太' : '高橋蓮馬'}</span>
                                </p>
                                <p className="text-sm text-gray-500">
                                    ユーザーID: <span className="break-all">{userId}</span>
                                </p>
                                <button
                                    onClick={() => handlePlayerSelect(null)}
                                    className="mt-2 text-sm text-gray-500 hover:text-gray-700 underline"
                                >
                                    プレイヤーを変更
                                </button>
                            </div>

                            {/* Overall Challenge Status Section - Moved to top */}
                            {/* 全体の対決状況セクション - 上部に移動 */}
                            <div className="bg-gray-50 p-6 rounded-xl shadow-inner mb-8">
                                <h2 className="text-2xl font-bold text-gray-700 mb-4">総支出での勝敗</h2>
                                <div className="text-center text-xl font-bold mb-4">
                                    <p className="text-gray-700">総支出:</p>
                                    <p className="text-blue-600">新井颯太: {sotaOverallTotal.toLocaleString()} 円</p>
                                    <p className="text-purple-600">高橋蓮馬: {renmaOverallTotal.toLocaleString()} 円</p>
                                </div>
                                <div className="text-center text-2xl font-extrabold p-4 rounded-lg bg-indigo-100 text-indigo-800">
                                    現在の勝者: {overallWinner}
                                </div>
                            </div>

                            {/* Expense Input Section */}
                            {/* 支出入力セクション */}
                            <div className="bg-gray-50 p-6 rounded-xl shadow-inner mb-8">
                                <h2 className="text-2xl font-bold text-gray-700 mb-4">今日の支出を記録</h2>
                                <div className="flex flex-col sm:flex-row items-center gap-4">
                                    <input
                                        type="date"
                                        value={selectedDate}
                                        onChange={(e) => setSelectedDate(e.target.value)}
                                        className="p-3 border border-gray-300 rounded-lg w-full sm:w-auto focus:ring-blue-500 focus:border-blue-500"
                                    />
                                    {/* Changed from select to text input for custom description */}
                                    {/* カスタム説明のために選択からテキスト入力に変更 */}
                                    <input
                                        type="text"
                                        placeholder="支出内容"
                                        value={currentExpenseDescription}
                                        onChange={(e) => setCurrentExpenseDescription(e.target.value)}
                                        className="p-3 border border-gray-300 rounded-lg w-full focus:ring-blue-500 focus:border-blue-500"
                                    />
                                    <input
                                        type="number"
                                        placeholder="支出額 (円)"
                                        value={currentExpenseAmount}
                                        onChange={(e) => setCurrentExpenseAmount(e.target.value)}
                                        className="p-3 border border-gray-300 rounded-lg w-full focus:ring-blue-500 focus:border-blue-500"
                                    />
                                    <button
                                        onClick={handleAddExpense}
                                        className="bg-green-500 hover:bg-green-600 text-white font-bold py-3 px-6 rounded-xl shadow-md transform transition duration-300 hover:scale-105 w-full sm:w-auto"
                                    >
                                        記録する
                                    </button>
                                </div>
                                {error && <p className="text-red-500 text-sm mt-2">{error}</p>}
                            </div>

                            {/* Current Challenge Status Section */}
                            {/* 現在の対決状況セクション */}
                            <div className="bg-gray-50 p-6 rounded-xl shadow-inner mb-8">
                                <h2 className="text-2xl font-bold text-gray-700 mb-4">現在の対決状況</h2>
                                <div className="grid grid-cols-1 sm:grid-cols-3 gap-4 text-center text-lg font-semibold">
                                    <div className="p-4 bg-blue-100 rounded-lg shadow-sm">
                                        <p className="text-blue-800">新井颯太の勝ち (日別)</p>
                                        <p className="text-3xl text-blue-900">{cumulativeDailyResults.sotaWins}</p>
                                    </div>
                                    <div className="p-4 bg-yellow-100 rounded-lg shadow-sm">
                                        <p className="text-yellow-800">引き分け (日別)</p>
                                        <p className="text-3xl text-yellow-900">{cumulativeDailyResults.draws}</p>
                                    </div>
                                    <div className="p-4 bg-purple-100 rounded-lg shadow-sm">
                                        <p className="text-purple-800">高橋蓮馬の勝ち (日別)</p>
                                        <p className="text-3xl text-purple-900">{cumulativeDailyResults.renmaWins}</p>
                                    </div>
                                </div>
                            </div>

                            {/* Monthly Goals Section */}
                            {/* 月間目標セクション */}
                            <div className="bg-gray-50 p-6 rounded-xl shadow-inner mb-8">
                                <h2 className="text-2xl font-bold text-gray-700 mb-4">月間目標</h2>
                                <div className="grid grid-cols-1 sm:grid-cols-2 gap-4">
                                    <div className="p-4 bg-blue-100 rounded-lg shadow-sm">
                                        <p className="text-blue-800 font-semibold">新井颯太</p>
                                        <p className="text-xl text-blue-900">目標: {sotaMonthlyGoal.toLocaleString()} 円</p>
                                        <p className="text-lg text-blue-700">現在: {sotaOverallTotal.toLocaleString()} 円</p>
                                        <div className="w-full bg-gray-200 rounded-full h-2.5 mt-2">
                                            <div
                                                className="bg-blue-600 h-2.5 rounded-full"
                                                style={{ width: `${Math.min(100, (sotaOverallTotal / sotaMonthlyGoal) * 100)}%` }}
                                            ></div>
                                        </div>
                                        <p className="text-sm text-gray-600 mt-1">
                                            達成率: {((sotaOverallTotal / sotaMonthlyGoal) * 100).toFixed(1)}%
                                        </p>
                                    </div>
                                    <div className="p-4 bg-purple-100 rounded-lg shadow-sm">
                                        <p className="text-purple-800 font-semibold">高橋蓮馬</p>
                                        <p className="text-xl text-purple-900">目標: {renmaMonthlyGoal.toLocaleString()} 円</p>
                                        <p className="text-lg text-purple-700">現在: {renmaOverallTotal.toLocaleString()} 円</p>
                                        <div className="w-full bg-gray-200 rounded-full h-2.5 mt-2">
                                            <div
                                                className="bg-purple-600 h-2.5 rounded-full"
                                                style={{ width: `${Math.min(100, (renmaOverallTotal / renmaMonthlyGoal) * 100)}%` }}
                                            ></div>
                                        </div>
                                        <p className="text-sm text-gray-600 mt-1">
                                            達成率: {((renmaOverallTotal / renmaMonthlyGoal) * 100).toFixed(1)}%
                                        </p>
                                    </div>
                                </div>
                            </div>

                            {/* Cumulative Expense Graph */}
                            {/* 累積支出グラフ */}
                            <div className="bg-gray-50 p-6 rounded-xl shadow-inner mb-8">
                                <h2 className="text-2xl font-bold text-gray-700 mb-4">累積支出グラフ</h2>
                                {chartData.length > 0 ? (
                                    <ResponsiveContainer width="100%" height={300}>
                                        <LineChart
                                            data={chartData}
                                            margin={{ top: 5, right: 30, left: 20, bottom: 5 }}
                                        >
                                            <CartesianGrid strokeDasharray="3 3" />
                                            <XAxis dataKey="date" />
                                            <YAxis />
                                            <Tooltip formatter={(value) => `${value.toLocaleString()} 円`} />
                                            <Legend />
                                            <Line type="monotone" dataKey="新井颯太" stroke="#3b82f6" activeDot={{ r: 8 }} />
                                            <Line type="monotone" dataKey="高橋蓮馬" stroke="#9333ea" activeDot={{ r: 8 }} />
                                            <Line type="monotone" dataKey="新井颯太の1.5倍" stroke="#f59e0b" strokeDasharray="5 5" />
                                        </LineChart>
                                    </ResponsiveContainer>
                                ) : (
                                    <p className="text-center text-gray-500">
                                        グラフを表示するには支出を記録してください。
                                    </p>
                                )}
                            </div>

                            {/* Daily Expenses and Results Table */}
                            {/* 日別支出と結果の表 */}
                            <div className="bg-gray-50 p-6 rounded-xl shadow-inner">
                                <h2 className="text-2xl font-bold text-gray-700 mb-4">日別支出と勝敗</h2>
                                <div className="overflow-x-auto">
                                    <table className="min-w-full bg-white rounded-lg shadow-md">
                                        <thead>
                                            <tr className="bg-gray-200 text-gray-700 uppercase text-sm leading-normal">
                                                <th className="py-3 px-6 text-left rounded-tl-lg">日付</th>
                                                <th className="py-3 px-6 text-right">新井颯太 (円)</th>
                                                <th className="py-3 px-6 text-right">高橋蓮馬 (円)</th>
                                                <th className="py-3 px-6 text-left rounded-tr-lg">勝敗 (日別)</th>
                                            </tr>
                                        </thead>
                                        <tbody className="text-gray-600 text-sm font-light">
                                            {Object.keys(expenses).sort().map(date => {
                                                const sotaExp = getDailyTotalExpense('sota', date);
                                                const renmaExp = getDailyTotalExpense('renma', date);
                                                const winner = getDailyWinner(sotaExp, renmaExp);
                                                return (
                                                    <tr key={date} className="border-b border-gray-200 hover:bg-gray-100">
                                                        <td className="py-3 px-6 text-left whitespace-nowrap">{date}</td>
                                                        <td className="py-3 px-6 text-right">{sotaExp.toLocaleString()}</td>
                                                        <td className="py-3 px-6 text-right">{renmaExp.toLocaleString()}</td>
                                                        <td className="py-3 px-6 text-left">
                                                            <span className={`py-1 px-3 rounded-full text-xs font-semibold ${
                                                                winner === '新井颯太の勝ち' ? 'bg-blue-200 text-blue-800' :
                                                                winner === '高橋蓮馬の勝ち' ? 'bg-purple-200 text-purple-800' :
                                                                winner === '引き分け' ? 'bg-yellow-200 text-yellow-800' : 'bg-gray-200 text-gray-800'
                                                            }`}>
                                                                {winner}
                                                            </span>
                                                        </td>
                                                    </tr>
                                                );
                                            })}
                                            {Object.keys(expenses).length === 0 && (
                                                <tr>
                                                    <td colSpan="4" className="py-4 text-center text-gray-500">
                                                        まだ支出が記録されていません。
                                                    </td>
                                                </tr>
                                            )}
                                        </tbody>
                                    </table>
                                </div>
                            </div>

                            {/* Rules Section */}
                            {/* ルールセクション */}
                            <div className="mt-8 p-6 bg-yellow-50 rounded-xl shadow-md border border-yellow-200">
                                <h2 className="text-2xl font-bold text-yellow-800 mb-4">ルール</h2>
                                <ul className="list-disc list-inside text-gray-700 space-y-2">
                                    <li>各プレイヤーは自分の支出のみを記録できます。</li>
                                    <li>日別（0:00〜23:59）の支出合計で勝敗を自動判定します。</li>
                                    <li>高橋蓮馬は新井颯太より1.5倍多く使う想定で調整します。</li>
                                    <li>高橋蓮馬の支出 &gt; 新井颯太の支出 × 1.5 の場合、<span className="font-bold text-blue-700">新井颯太の勝ち</span>です。</li>
                                    <li>高橋蓮馬の支出 = 新井颯太の支出 × 1.5 の場合、<span className="font-bold text-yellow-700">引き分け</span>です。</li>
                                    <li>高橋蓮馬の支出 &lt; 新井颯太の支出 × 1.5 の場合、<span className="font-bold text-purple-700">高橋蓮馬の勝ち</span>です。</li>
                                    <li>より少ない支出で効率的な節約を競います！</li>
                                    <li>🍽️ 負けた方は2回一般的な価格帯のランチをおごります。</li>
                                </ul>
                            </div>
                        </>
                    )}
                </div>
            </div>
        </FirebaseContext.Provider>
    );
}

export default App;


import React, { useState, useEffect } from 'react';
import { 
  Wallet, 
  LogIn, 
  UserPlus, 
  LogOut, 
  PlusCircle, 
  TrendingUp, 
  TrendingDown, 
  CreditCard,
  AlertCircle
} from 'lucide-react';

export default function App() {
  // ==========================================
  // State Management (จัดการสถานะของแอป)
  // ==========================================
  const [view, setView] = useState('login'); // 'login', 'register', 'dashboard'
  const [currentUser, setCurrentUser] = useState(null);
  
  // จำลองฐานข้อมูลด้วย LocalStorage
  const [users, setUsers] = useState(() => {
    const saved = localStorage.getItem('cashbook_users');
    return saved ? JSON.parse(saved) : [];
  });
  
  const [transactions, setTransactions] = useState(() => {
    const saved = localStorage.getItem('cashbook_transactions');
    return saved ? JSON.parse(saved) : [];
  });

  const [errorMsg, setErrorMsg] = useState('');
  const [successMsg, setSuccessMsg] = useState('');

  // บันทึกข้อมูลลง LocalStorage อัตโนมัติเมื่อมีการเปลี่ยนแปลง
  useEffect(() => {
    localStorage.setItem('cashbook_users', JSON.stringify(users));
  }, [users]);

  useEffect(() => {
    localStorage.setItem('cashbook_transactions', JSON.stringify(transactions));
  }, [transactions]);

  // ซ่อนข้อความแจ้งเตือนอัตโนมัติ
  useEffect(() => {
    if (errorMsg || successMsg) {
      const timer = setTimeout(() => {
        setErrorMsg('');
        setSuccessMsg('');
      }, 3000);
      return () => clearTimeout(timer);
    }
  }, [errorMsg, successMsg]);

  // ==========================================
  // Auth Controllers (ระบบจัดการผู้ใช้)
  // ==========================================
  const handleRegister = (e) => {
    e.preventDefault();
    const username = e.target.username.value.trim();
    const password = e.target.password.value;

    if (!username || !password) {
      setErrorMsg('กรุณากรอกข้อมูลให้ครบถ้วน');
      return;
    }

    if (users.find(u => u.username === username)) {
      setErrorMsg('ชื่อผู้ใช้นี้มีในระบบแล้ว');
      return;
    }

    const newUser = { id: Date.now(), username, password }; // ในระบบจริง password ต้องถูก hash
    setUsers([...users, newUser]);
    setSuccessMsg('สมัครสมาชิกสำเร็จ! กรุณาเข้าสู่ระบบ');
    setView('login');
  };

  const handleLogin = (e) => {
    e.preventDefault();
    const username = e.target.username.value.trim();
    const password = e.target.password.value;

    const user = users.find(u => u.username === username && u.password === password);
    if (!user) {
      setErrorMsg('ชื่อผู้ใช้หรือรหัสผ่านไม่ถูกต้อง');
      return;
    }

    setCurrentUser(user);
    setSuccessMsg('เข้าสู่ระบบสำเร็จ!');
    setView('dashboard');
  };

  const handleLogout = () => {
    setCurrentUser(null);
    setView('login');
  };

  // ==========================================
  // Transaction Controllers (ระบบจัดการรายรับ-รายจ่าย)
  // ==========================================
  const handleAddTransaction = (e) => {
    e.preventDefault();
    const date = e.target.date.value;
    const abstract = e.target.abstract.value.trim();
    const type = e.target.type.value;
    const amount = Number(e.target.amount.value);

    if (!date || !abstract || !amount || amount <= 0) {
      setErrorMsg('กรุณากรอกข้อมูลให้ครบถ้วนและจำนวนเงินต้องมากกว่า 0');
      return;
    }

    // กรองเฉพาะรายการของ user ปัจจุบัน เพื่อคำนวณ balance
    const userTransactions = transactions.filter(t => t.userId === currentUser.id);
    
    // ดึง balance ล่าสุด
    let currentBalance = 0;
    if (userTransactions.length > 0) {
      currentBalance = userTransactions[userTransactions.length - 1].balance;
    }

    // คำนวณยอดคงเหลือใหม่
    if (type === 'income') {
      currentBalance += amount;
    } else {
      currentBalance -= amount;
    }

    const newTx = {
      id: Date.now(),
      userId: currentUser.id,
      date,
      abstract,
      type,
      amount,
      balance: currentBalance
    };

    setTransactions([...transactions, newTx]);
    setSuccessMsg('บันทึกรายการสำเร็จ');
    e.target.reset();
    
    // ตั้งค่าวันที่เริ่มต้นเป็นวันนี้
    const today = new Date().toISOString().split('T')[0];
    e.target.date.value = today;
  };

  // กรองข้อมูลเฉพาะของผู้ใช้ปัจจุบันสำหรับการแสดงผล
  const myTransactions = currentUser 
    ? transactions.filter(t => t.userId === currentUser.id) 
    : [];
  
  // เรียงลำดับจากใหม่ไปเก่า (DESC) ตามโค้ด API
  const displayTransactions = [...myTransactions].sort((a, b) => b.id - a.id);

  // คำนวณยอดรวมต่างๆ
  const totalIncome = myTransactions.filter(t => t.type === 'income').reduce((sum, t) => sum + t.amount, 0);
  const totalExpense = myTransactions.filter(t => t.type === 'expense').reduce((sum, t) => sum + t.amount, 0);
  const currentTotalBalance = myTransactions.length > 0 ? myTransactions[myTransactions.length - 1].balance : 0;

  // ==========================================
  // UI Components
  // ==========================================

  // Component แจ้งเตือน
  const AlertMessage = () => {
    if (!errorMsg && !successMsg) return null;
    return (
      <div className={`p-4 mb-4 rounded-lg flex items-center ${errorMsg ? 'bg-red-100 text-red-700' : 'bg-green-100 text-green-700'}`}>
        <AlertCircle className="w-5 h-5 mr-2" />
        <span>{errorMsg || successMsg}</span>
      </div>
    );
  };

  // หน้า Login / Register
  if (view === 'login' || view === 'register') {
    return (
      <div className="min-h-screen bg-gray-50 flex flex-col justify-center py-12 sm:px-6 lg:px-8">
        <div className="sm:mx-auto sm:w-full sm:max-w-md">
          <div className="flex justify-center text-blue-600">
            <Wallet className="w-16 h-16" />
          </div>
          <h2 className="mt-6 text-center text-3xl font-extrabold text-gray-900">
            Cashbook App
          </h2>
          <p className="mt-2 text-center text-sm text-gray-600">
            {view === 'login' ? 'เข้าสู่ระบบเพื่อจัดการบัญชีของคุณ' : 'สมัครสมาชิกเพื่อเริ่มต้นใช้งาน'}
          </p>
        </div>

        <div className="mt-8 sm:mx-auto sm:w-full sm:max-w-md">
          <div className="bg-white py-8 px-4 shadow sm:rounded-lg sm:px-10">
            <AlertMessage />
            
            <form className="space-y-6" onSubmit={view === 'login' ? handleLogin : handleRegister}>
              <div>
                <label className="block text-sm font-medium text-gray-700">ชื่อผู้ใช้ (Username)</label>
                <div className="mt-1">
                  <input name="username" type="text" required className="appearance-none block w-full px-3 py-2 border border-gray-300 rounded-md shadow-sm focus:outline-none focus:ring-blue-500 focus:border-blue-500 sm:text-sm" />
                </div>
              </div>

              <div>
                <label className="block text-sm font-medium text-gray-700">รหัสผ่าน (Password)</label>
                <div className="mt-1">
                  <input name="password" type="password" required className="appearance-none block w-full px-3 py-2 border border-gray-300 rounded-md shadow-sm focus:outline-none focus:ring-blue-500 focus:border-blue-500 sm:text-sm" />
                </div>
              </div>

              <div>
                <button type="submit" className="w-full flex justify-center py-2 px-4 border border-transparent rounded-md shadow-sm text-sm font-medium text-white bg-blue-600 hover:bg-blue-700 focus:outline-none focus:ring-2 focus:ring-offset-2 focus:ring-blue-500">
                  {view === 'login' ? (
                    <><LogIn className="w-5 h-5 mr-2" /> เข้าสู่ระบบ</>
                  ) : (
                    <><UserPlus className="w-5 h-5 mr-2" /> สมัครสมาชิก</>
                  )}
                </button>
              </div>
            </form>

            <div className="mt-6 text-center">
              <button 
                onClick={() => setView(view === 'login' ? 'register' : 'login')}
                className="text-sm text-blue-600 hover:text-blue-500 font-medium"
              >
                {view === 'login' ? 'ยังไม่มีบัญชี? สมัครสมาชิกที่นี่' : 'มีบัญชีอยู่แล้ว? เข้าสู่ระบบ'}
              </button>
            </div>
          </div>
        </div>
      </div>
    );
  }

  // หน้า Dashboard (หลังเข้าสู่ระบบ)
  return (
    <div className="min-h-screen bg-gray-100 pb-12">
      {/* Navigation */}
      <nav className="bg-blue-600 shadow-lg">
        <div className="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8">
          <div className="flex justify-between h-16">
            <div className="flex items-center text-white">
              <Wallet className="w-8 h-8 mr-2" />
              <span className="font-bold text-xl tracking-tight">Cashbook</span>
            </div>
            <div className="flex items-center">
              <span className="text-blue-100 mr-4 text-sm hidden sm:block">
                สวัสดี, <strong>{currentUser.username}</strong>
              </span>
              <button 
                onClick={handleLogout}
                className="flex items-center px-3 py-2 rounded-md text-sm font-medium text-white bg-blue-700 hover:bg-blue-800 focus:outline-none focus:ring-2 focus:ring-offset-2 focus:ring-offset-blue-600 focus:ring-white transition-colors"
              >
                <LogOut className="w-4 h-4 mr-1 sm:mr-2" />
                <span className="hidden sm:inline">ออกจากระบบ</span>
              </button>
            </div>
          </div>
        </div>
      </nav>

      <div className="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8 mt-8">
        <AlertMessage />

        {/* Summary Cards */}
        <div className="grid grid-cols-1 gap-5 sm:grid-cols-3 mb-8">
          <div className="bg-white overflow-hidden shadow rounded-lg border-l-4 border-blue-500">
            <div className="p-5">
              <div className="flex items-center">
                <div className="flex-shrink-0 bg-blue-100 rounded-md p-3">
                  <CreditCard className="h-6 w-6 text-blue-600" />
                </div>
                <div className="ml-5 w-0 flex-1">
                  <dl>
                    <dt className="text-sm font-medium text-gray-500 truncate">ยอดคงเหลือ (Balance)</dt>
                    <dd className="text-2xl font-semibold text-gray-900">฿{currentTotalBalance.toLocaleString(undefined, {minimumFractionDigits: 2})}</dd>
                  </dl>
                </div>
              </div>
            </div>
          </div>

          <div className="bg-white overflow-hidden shadow rounded-lg border-l-4 border-green-500">
            <div className="p-5">
              <div className="flex items-center">
                <div className="flex-shrink-0 bg-green-100 rounded-md p-3">
                  <TrendingUp className="h-6 w-6 text-green-600" />
                </div>
                <div className="ml-5 w-0 flex-1">
                  <dl>
                    <dt className="text-sm font-medium text-gray-500 truncate">รายรับรวม (Total Income)</dt>
                    <dd className="text-xl font-semibold text-green-600">฿{totalIncome.toLocaleString(undefined, {minimumFractionDigits: 2})}</dd>
                  </dl>
                </div>
              </div>
            </div>
          </div>

          <div className="bg-white overflow-hidden shadow rounded-lg border-l-4 border-red-500">
            <div className="p-5">
              <div className="flex items-center">
                <div className="flex-shrink-0 bg-red-100 rounded-md p-3">
                  <TrendingDown className="h-6 w-6 text-red-600" />
                </div>
                <div className="ml-5 w-0 flex-1">
                  <dl>
                    <dt className="text-sm font-medium text-gray-500 truncate">รายจ่ายรวม (Total Expense)</dt>
                    <dd className="text-xl font-semibold text-red-600">฿{totalExpense.toLocaleString(undefined, {minimumFractionDigits: 2})}</dd>
                  </dl>
                </div>
              </div>
            </div>
          </div>
        </div>

        <div className="grid grid-cols-1 lg:grid-cols-3 gap-8">
          {/* Form - Add Transaction */}
          <div className="lg:col-span-1">
            <div className="bg-white shadow sm:rounded-lg">
              <div className="px-4 py-5 border-b border-gray-200 sm:px-6">
                <h3 className="text-lg leading-6 font-medium text-gray-900 flex items-center">
                  <PlusCircle className="w-5 h-5 mr-2 text-blue-600" />
                  เพิ่มรายการใหม่
                </h3>
              </div>
              <div className="px-4 py-5 sm:p-6">
                <form onSubmit={handleAddTransaction} className="space-y-4">
                  <div>
                    <label className="block text-sm font-medium text-gray-700">วันที่</label>
                    <input 
                      type="date" 
                      name="date" 
                      required 
                      defaultValue={new Date().toISOString().split('T')[0]}
                      className="mt-1 block w-full border border-gray-300 rounded-md shadow-sm py-2 px-3 focus:outline-none focus:ring-blue-500 focus:border-blue-500 sm:text-sm" 
                    />
                  </div>
                  <div>
                    <label className="block text-sm font-medium text-gray-700">ชื่อรายการ</label>
                    <input 
                      type="text" 
                      name="abstract" 
                      placeholder="เช่น เงินเดือน, ค่าอาหาร" 
                      required 
                      className="mt-1 block w-full border border-gray-300 rounded-md shadow-sm py-2 px-3 focus:outline-none focus:ring-blue-500 focus:border-blue-500 sm:text-sm" 
                    />
                  </div>
                  <div>
                    <label className="block text-sm font-medium text-gray-700">ประเภท</label>
                    <select 
                      name="type" 
                      className="mt-1 block w-full bg-white border border-gray-300 rounded-md shadow-sm py-2 px-3 focus:outline-none focus:ring-blue-500 focus:border-blue-500 sm:text-sm"
                    >
                      <option value="expense">รายจ่าย</option>
                      <option value="income">รายรับ</option>
                    </select>
                  </div>
                  <div>
                    <label className="block text-sm font-medium text-gray-700">จำนวนเงิน</label>
                    <div className="mt-1 relative rounded-md shadow-sm">
                      <div className="absolute inset-y-0 left-0 pl-3 flex items-center pointer-events-none">
                        <span className="text-gray-500 sm:text-sm">฿</span>
                      </div>
                      <input 
                        type="number" 
                        name="amount" 
                        min="0.01" 
                        step="0.01" 
                        placeholder="0.00" 
                        required 
                        className="mt-1 block w-full border border-gray-300 rounded-md shadow-sm py-2 pl-8 pr-3 focus:outline-none focus:ring-blue-500 focus:border-blue-500 sm:text-sm" 
                      />
                    </div>
                  </div>
                  <button 
                    type="submit" 
                    className="w-full flex justify-center py-2 px-4 border border-transparent rounded-md shadow-sm text-sm font-medium text-white bg-blue-600 hover:bg-blue-700 focus:outline-none focus:ring-2 focus:ring-offset-2 focus:ring-blue-500"
                  >
                    บันทึกข้อมูล
                  </button>
                </form>
              </div>
            </div>
          </div>

          {/* Table - Transactions List */}
          <div className="lg:col-span-2">
            <div className="bg-white shadow sm:rounded-lg overflow-hidden">
              <div className="px-4 py-5 border-b border-gray-200 sm:px-6 flex justify-between items-center bg-gray-50">
                <h3 className="text-lg leading-6 font-medium text-gray-900">
                  ประวัติรายการ (ล่าสุด)
                </h3>
              </div>
              
              <div className="overflow-x-auto">
                {displayTransactions.length === 0 ? (
                  <div className="p-8 text-center text-gray-500">
                    <Wallet className="mx-auto h-12 w-12 text-gray-300 mb-3" />
                    <p>ยังไม่มีข้อมูลรายการ เริ่มต้นบันทึกรายรับ-รายจ่ายของคุณได้เลย!</p>
                  </div>
                ) : (
                  <table className="min-w-full divide-y divide-gray-200">
                    <thead className="bg-gray-50">
                      <tr>
                        <th scope="col" className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">วันที่</th>
                        <th scope="col" className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">รายการ</th>
                        <th scope="col" className="px-6 py-3 text-right text-xs font-medium text-gray-500 uppercase tracking-wider">รายรับ</th>
                        <th scope="col" className="px-6 py-3 text-right text-xs font-medium text-gray-500 uppercase tracking-wider">รายจ่าย</th>
                        <th scope="col" className="px-6 py-3 text-right text-xs font-medium text-gray-500 uppercase tracking-wider">คงเหลือ</th>
                      </tr>
                    </thead>
                    <tbody className="bg-white divide-y divide-gray-200">
                      {displayTransactions.map((tx) => (
                        <tr key={tx.id} className="hover:bg-gray-50 transition-colors">
                          <td className="px-6 py-4 whitespace-nowrap text-sm text-gray-500">
                            {new Date(tx.date).toLocaleDateString('th-TH', { year: 'numeric', month: 'short', day: 'numeric' })}
                          </td>
                          <td className="px-6 py-4 text-sm font-medium text-gray-900">
                            <div className="flex items-center">
                              <span className={`w-2 h-2 rounded-full mr-2 ${tx.type === 'income' ? 'bg-green-500' : 'bg-red-500'}`}></span>
                              {tx.abstract}
                            </div>
                          </td>
                          <td className="px-6 py-4 whitespace-nowrap text-sm text-right text-green-600 font-medium">
                            {tx.type === 'income' ? `+฿${tx.amount.toLocaleString(undefined, {minimumFractionDigits: 2})}` : '-'}
                          </td>
                          <td className="px-6 py-4 whitespace-nowrap text-sm text-right text-red-600 font-medium">
                            {tx.type === 'expense' ? `-฿${tx.amount.toLocaleString(undefined, {minimumFractionDigits: 2})}` : '-'}
                          </td>
                          <td className="px-6 py-4 whitespace-nowrap text-sm text-right font-bold text-gray-900">
                            ฿{tx.balance.toLocaleString(undefined, {minimumFractionDigits: 2})}
                          </td>
                        </tr>
                      ))}
                    </tbody>
                  </table>
                )}
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>
  );
}

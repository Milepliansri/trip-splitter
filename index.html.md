# ```react  
import React, { useState, useMemo, useRef, useEffect } from 'react';  
import { Wallet, Users, Receipt, ArrowRight, Plus, Trash2, X, Check, Calculator, Edit2, ChevronRight, Info, Cloud, CloudOff, Loader2 } from 'lucide-react';  
  
// --- Firebase Imports ---  
import { initializeApp } from 'firebase/app';  
import { getAuth, signInWithCustomToken, signInAnonymously, onAuthStateChanged } from 'firebase/auth';  
import { getFirestore, doc, setDoc, onSnapshot } from 'firebase/firestore';  
  
// --- Firebase Initialization ---  
const firebaseConfig = typeof __firebase_config !== 'undefined' ? JSON.parse(__firebase_config) : {};  
const app = Object.keys(firebaseConfig).length ? initializeApp(firebaseConfig) : null;  
const auth = app ? getAuth(app) : null;  
const db = app ? getFirestore(app) : null;  
const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';  
  
export default function App() {  
  // สถานะแอปพลิเคชัน  
  const [activeTab, setActiveTab] = useState('members'); // เปิดมาหน้าสมาชิกก่อนเพื่อให้เพิ่มคน  
  const [tripName, setTripName] = useState('ทริปใหม่ของฉัน');  
  const [isEditingTripName, setIsEditingTripName] = useState(false);  
  const tripNameInputRef = useRef(null);  
  
  // สถานะ Cloud & Auth  
  const [user, setUser] = useState(null);  
  const [isSyncing, setIsSyncing] = useState(true);  
  const [cloudStatus, setCloudStatus] = useState('connecting'); // connecting, synced, error  
  
  // Modals  
  const [isExpenseModalOpen, setIsExpenseModalOpen] = useState(false);  
  const [selectedMemberDetail, setSelectedMemberDetail] = useState(null);  
  const [confirmDialog, setConfirmDialog] = useState({ isOpen: false, type: '', id: '', message: '' });  
  
  // ข้อมูลตั้งต้น (ว่างเปล่า)  
  const [members, setMembers] = useState([]);  
  const [expenses, setExpenses] = useState([]);  
  
  // ฟอร์ม  
  const [newExpense, setNewExpense] = useState({ title: '', amount: '', payerId: '', splitBetween: [] });  
  const [newMemberName, setNewMemberName] = useState('');  
  
  // --------------------------------------------------  
  // Firebase Auth & Real-time Database  
  // --------------------------------------------------  
  useEffect(() => {  
    if (!auth) {  
      setCloudStatus('error');  
      setIsSyncing(false);  
      return;  
    }  
  
    const initAuth = async () => {  
      try {  
        if (typeof __initial_auth_token !== 'undefined' && __initial_auth_token) {  
          await signInWithCustomToken(auth, __initial_auth_token);  
        } else {  
          await signInAnonymously(auth);  
        }  
      } catch (error) {  
        console.error("Auth failed:", error);  
        setCloudStatus('error');  
      }  
    };  
  
    initAuth();  
    const unsubscribe = onAuthStateChanged(auth, setUser);  
    return () => unsubscribe();  
  }, []);  
  
  useEffect(() => {  
    if (!user || !db) return;  
      
    // ชี้เป้าไปที่เอกสาร Database หลักของทริปนี้  
    const docRef = doc(db, 'artifacts', appId, 'public', 'data', 'trip_state', 'main');  
  
    const unsubscribe = onSnapshot(docRef, (docSnap) => {  
      if (docSnap.exists()) {  
        const data = docSnap.data();  
        setMembers(data.members || []);  
        setExpenses(data.expenses || []);  
        if (data.tripName) setTripName(data.tripName);  
      } else {  
        // หากยังไม่เคยมีข้อมูลในระบบคลาวด์ ให้เอาข้อมูลตั้งต้น(ที่ว่างเปล่า)ใส่เข้าไป  
        setDoc(docRef, { members, expenses, tripName });  
      }  
      setCloudStatus('synced');  
      setIsSyncing(false);  
    }, (error) => {  
      console.error("Database connection error: ", error);  
      setCloudStatus('error');  
      setIsSyncing(false);  
    });  
  
    return () => unsubscribe();  
  }, [user]);  
  
  // ฟังก์ชันสำหรับเซฟข้อมูลเข้า Cloud  
  const saveStateToFirebase = async (newMembers, newExpenses, newTripName) => {  
    if (!user || !db) return;  
    setCloudStatus('syncing');  
    const docRef = doc(db, 'artifacts', appId, 'public', 'data', 'trip_state', 'main');  
    try {  
      await setDoc(docRef, {   
        members: newMembers,   
        expenses: newExpenses,   
        tripName: newTripName   
      });  
      setCloudStatus('synced');  
    } catch (error) {  
      console.error("Error saving data:", error);  
      setCloudStatus('error');  
    }  
  };  
  
  // โฟกัส Input เมื่อกดแก้ไขชื่อทริป  
  useEffect(() => {  
    if (isEditingTripName && tripNameInputRef.current) {  
      tripNameInputRef.current.focus();  
    }  
  }, [isEditingTripName]);  
  
  // --------------------------------------------------  
  // คำนวณยอดเงิน (Engine)  
  // --------------------------------------------------  
  const { totalTripCost, balances, settlements, memberDetails } = useMemo(() => {  
    let total = 0;  
    const memberStats = {};  
    const details = {};  
  
    members.forEach(m => {  
      memberStats[m.id] = { id: m.id, name: m.name, paid: 0, share: 0, balance: 0 };  
      details[m.id] = { paidItems: [], sharedItems: [] };  
    });  
  
    expenses.forEach(exp => {  
      total += exp.amount;  
        
      if (memberStats[exp.payerId]) {  
        memberStats[exp.payerId].paid += exp.amount;  
        details[exp.payerId].paidItems.push({ title: exp.title, amount: exp.amount });  
      }  
        
      const splitCount = exp.splitBetween.length;  
      if (splitCount > 0) {  
        const shareAmount = exp.amount / splitCount;  
        exp.splitBetween.forEach(userId => {  
          if (memberStats[userId]) {  
            memberStats[userId].share += shareAmount;  
            details[userId].sharedItems.push({ title: exp.title, totalAmount: exp.amount, splitCount, shareAmount });  
          }  
        });  
      }  
    });  
  
    const balanceArray = Object.values(memberStats).map(stat => ({  
      ...stat,  
      balance: stat.paid - stat.share  
    }));  
  
    const debtors = balanceArray.filter(b => b.balance < -0.01).map(b => ({ ...b, amount: Math.abs(b.balance) })).sort((a, b) => b.amount - a.amount);  
    const creditors = balanceArray.filter(b => b.balance > 0.01).map(b => ({ ...b, amount: b.balance })).sort((a, b) => b.amount - a.amount);  
      
    const transfers = [];  
    let i = 0, j = 0;  
  
    while (i < debtors.length && j < creditors.length) {  
      const debtor = debtors[i];  
      const creditor = creditors[j];  
      const amount = Math.min(debtor.amount, creditor.amount);  
  
      transfers.push({  
        fromId: debtor.id, fromName: debtor.name,  
        toId: creditor.id, toName: creditor.name,  
        amount: amount  
      });  
  
      debtor.amount -= amount;  
      creditor.amount -= amount;  
  
      if (debtor.amount < 0.01) i++;  
      if (creditor.amount < 0.01) j++;  
    }  
  
    return { totalTripCost: total, balances: balanceArray, settlements: transfers, memberDetails: details };  
  }, [expenses, members]);  
  
  // --------------------------------------------------  
  // Handlers (แก้ไขข้อมูล & เซฟลงคลาวด์)  
  // --------------------------------------------------  
  const handleSaveTripName = () => {  
    let nameToSave = tripName;  
    if (!tripName.trim()) {  
      nameToSave = 'ทริปไม่ระบุชื่อ';  
      setTripName(nameToSave);  
    }  
    setIsEditingTripName(false);  
    saveStateToFirebase(members, expenses, nameToSave); // เซฟ  
  };  
  
  const handleOpenExpenseModal = () => {  
    if (members.length === 0) {  
        alert("กรุณาเพิ่มสมาชิกในหน้า 'สมาชิก' ก่อนเพิ่มรายการค่าใช้จ่าย");  
        setActiveTab('members');  
        return;  
    }  
    setNewExpense({  
        title: '',  
        amount: '',  
        payerId: members[0]?.id || '',  
        splitBetween: members.map(m => m.id)  
    });  
    setIsExpenseModalOpen(true);  
  };  
  
  const handleAddExpense = (e) => {  
    e.preventDefault();  
    if (!newExpense.title || !newExpense.amount || newExpense.splitBetween.length === 0) return;  
      
    const newExpenseItem = {  
      id: Date.now().toString(),  
      title: newExpense.title,  
      amount: parseFloat(newExpense.amount),  
      payerId: newExpense.payerId,  
      splitBetween: newExpense.splitBetween,  
      date: new Date().toISOString()  
    };  
      
    const newExpenses = [...expenses, newExpenseItem];  
    setExpenses(newExpenses);  
    saveStateToFirebase(members, newExpenses, tripName); // เซฟ  
      
    setNewExpense({ title: '', amount: '', payerId: members[0]?.id || '', splitBetween: members.map(m => m.id) });  
    setIsExpenseModalOpen(false);  
  };  
  
  const handleDeleteExpense = (id) => {  
    setConfirmDialog({  
      isOpen: true,  
      type: 'expense',  
      id: id,  
      message: 'ต้องการลบรายการค่าใช้จ่ายนี้ใช่หรือไม่? ยอดเงินจะถูกคำนวณใหม่และซิงค์ทันที'  
    });  
  };  
  
  const handleAddMember = (e) => {  
    e.preventDefault();  
    if (!newMemberName.trim()) return;  
    const newId = Date.now().toString();  
      
    const newMembers = [...members, { id: newId, name: newMemberName.trim() }];  
    setMembers(newMembers);  
      
    // ถ้ากำลังเปิดฟอร์มเพิ่มค่าใช้จ่ายอยู่ ให้เลือกคนใหม่เข้าร่วมหารด้วยอัตโนมัติ  
    setNewExpense(prev => ({   
        ...prev,   
        splitBetween: prev.splitBetween ? [...prev.splitBetween, newId] : [newId]   
    }));  
      
    setNewMemberName('');  
    saveStateToFirebase(newMembers, expenses, tripName); // เซฟ  
  };  
  
  const handleDeleteMember = (id) => {  
    setConfirmDialog({  
      isOpen: true,  
      type: 'member',  
      id: id,  
      message: 'ต้องการลบสมาชิกคนนี้ใช่หรือไม่? รายการที่เกี่ยวข้องจะถูกอัปเดตอัตโนมัติ'  
    });  
  };  
  
  const executeDelete = () => {  
    if (confirmDialog.type === 'expense') {  
      const newExpenses = expenses.filter(e => e.id !== confirmDialog.id);  
      setExpenses(newExpenses);  
      saveStateToFirebase(members, newExpenses, tripName); // เซฟ  
    } else if (confirmDialog.type === 'member') {  
      const newMembers = members.filter(m => m.id !== confirmDialog.id);  
      const newExpenses = expenses.map(e => ({  
        ...e,  
        splitBetween: e.splitBetween.filter(mid => mid !== confirmDialog.id)  
      }));  
      setMembers(newMembers);  
      setExpenses(newExpenses);  
      saveStateToFirebase(newMembers, newExpenses, tripName); // เซฟ  
    }  
    setConfirmDialog({ isOpen: false, type: '', id: '', message: '' });  
  };  
  
  const toggleSplitMember = (memberId) => {  
    setNewExpense(prev => {  
      const isIncluded = prev.splitBetween.includes(memberId);  
      return {  
        ...prev,  
        splitBetween: isIncluded ? prev.splitBetween.filter(id => id !== memberId) : [...prev.splitBetween, memberId]  
      };  
    });  
  };  
  
  const formatMoney = (amount) => {  
    return new Intl.NumberFormat('th-TH', { style: 'decimal', minimumFractionDigits: 2, maximumFractionDigits: 2 }).format(amount);  
  };  
  
  // --------------------------------------------------  
  // UI Components  
  // --------------------------------------------------  
  const MemberDetailModal = () => {  
    if (!selectedMemberDetail) return null;  
    const member = members.find(m => m.id === selectedMemberDetail);  
    const stat = balances.find(b => b.id === selectedMemberDetail);  
    const detail = memberDetails[selectedMemberDetail];  
  
    if (!member || !stat || !detail) return null;  
  
    return (  
      <div className="fixed inset-0 bg-slate-900/60 backdrop-blur-sm flex items-end md:items-center justify-center z-50 animate-in fade-in duration-200">  
        <div className="bg-white w-full max-w-lg md:rounded-2xl rounded-t-2xl shadow-2xl overflow-hidden animate-in slide-in-from-bottom-10 md:zoom-in-95 duration-300 flex flex-col max-h-[90vh]">  
          <div className="bg-slate-50 border-b border-slate-100 p-4 flex justify-between items-center sticky top-0 z-10">  
            <div>  
              <h3 className="text-xl font-bold text-slate-800">สรุปของ {member.name}</h3>  
              <p className={`text-sm font-medium ${stat.balance > 0 ? 'text-emerald-600' : stat.balance < 0 ? 'text-rose-600' : 'text-slate-500'}`}>  
                {stat.balance > 0 ? `ได้เงินคืนรวม ฿${formatMoney(stat.balance)}` : stat.balance < 0 ? `ต้องจ่ายเพิ่มรวม ฿${formatMoney(Math.abs(stat.balance))}` : 'เคลียร์บิลแล้ว ไม่มียอดค้าง'}  
              </p>  
            </div>  
            <button onClick={() => setSelectedMemberDetail(null)} className="p-2 bg-slate-200/50 rounded-full text-slate-500 hover:bg-slate-200 transition-colors">  
              <X className="w-5 h-5" />  
            </button>  
          </div>  
  
          <div className="overflow-y-auto p-4 space-y-6">  
            <section>  
              <h4 className="font-bold text-emerald-600 flex items-center gap-2 mb-3">  
                <ArrowRight className="w-4 h-4 rotate-45" /> จ่ายไปก่อน (รวม ฿{formatMoney(stat.paid)})  
              </h4>  
              {detail.paidItems.length === 0 ? (  
                <p className="text-sm text-slate-400 bg-slate-50 p-3 rounded-lg border border-slate-100">ไม่ได้จ่ายรายการใดๆ ไว้ล่วงหน้า</p>  
              ) : (  
                <div className="space-y-2">  
                  {detail.paidItems.map((item, idx) => (  
                    <div key={idx} className="flex justify-between items-center text-sm p-3 bg-emerald-50/50 rounded-lg border border-emerald-100">  
                      <span className="text-slate-700">{item.title}</span>  
                      <span className="font-semibold text-emerald-700">฿{formatMoney(item.amount)}</span>  
                    </div>  
                  ))}  
                </div>  
              )}  
            </section>  
  
            <section>  
              <h4 className="font-bold text-rose-600 flex items-center gap-2 mb-3">  
                <ArrowRight className="w-4 h-4 -rotate-45" /> ส่วนแบ่งที่ต้องจ่าย (รวม ฿{formatMoney(stat.share)})  
              </h4>  
              {detail.sharedItems.length === 0 ? (  
                <p className="text-sm text-slate-400 bg-slate-50 p-3 rounded-lg border border-slate-100">ไม่มีส่วนแบ่งในรายการใดๆ</p>  
              ) : (  
                <div className="space-y-2">  
                  {detail.sharedItems.map((item, idx) => (  
                    <div key={idx} className="flex justify-between items-start text-sm p-3 bg-rose-50/50 rounded-lg border border-rose-100">  
                      <div>  
                        <div className="text-slate-700 font-medium">{item.title}</div>  
                        <div className="text-xs text-slate-500 mt-1">ยอดเต็ม ฿{formatMoney(item.totalAmount)} (หาร {item.splitCount})</div>  
                      </div>  
                      <span className="font-semibold text-rose-700">฿{formatMoney(item.shareAmount)}</span>  
                    </div>  
                  ))}  
                </div>  
              )}  
            </section>  
          </div>  
        </div>  
      </div>  
    );  
  };  
  
  // Screen Loading ระหว่างต่อฐานข้อมูล  
  if (isSyncing && cloudStatus === 'connecting') {  
    return (  
      <div className="min-h-screen bg-slate-50 flex flex-col items-center justify-center space-y-4">  
        <Loader2 className="w-10 h-10 text-indigo-500 animate-spin" />  
        <p className="text-slate-500 font-medium animate-pulse">กำลังซิงค์ข้อมูลกับคลาวด์...</p>  
      </div>  
    );  
  }  
  
  return (  
    <div className="min-h-screen bg-slate-100/50 text-slate-800 font-sans pb-20 md:pb-6">  
      {/* Header Area */}  
      <header className="bg-gradient-to-r from-blue-600 to-indigo-600 text-white shadow-lg sticky top-0 z-30">  
        <div className="max-w-3xl mx-auto px-4 py-4 md:py-6">  
          <div className="flex items-center justify-between mb-4">  
            <div className="flex items-center gap-3">  
              <div className="p-2 bg-white/20 rounded-xl backdrop-blur-sm">  
                <Calculator className="w-6 h-6" />  
              </div>  
              <div>  
                <div className="flex items-center gap-2 mb-0.5">  
                  <p className="text-blue-100 text-xs font-medium uppercase tracking-wider">แชร์เงินไปเที่ยวกัน</p>  
                  {/* Cloud Indicator */}  
                  {cloudStatus === 'synced' ? (  
                    <span className="flex items-center gap-1 text-[10px] bg-emerald-500/20 text-emerald-100 px-1.5 py-0.5 rounded-full" title="ซิงค์ข้อมูลแล้ว">  
                      <Cloud className="w-3 h-3" /> ออนไลน์  
                    </span>  
                  ) : cloudStatus === 'error' ? (  
                    <span className="flex items-center gap-1 text-[10px] bg-rose-500/20 text-rose-100 px-1.5 py-0.5 rounded-full" title="ไม่ได้เชื่อมต่อคลาวด์ (บันทึกแค่ในเครื่อง)">  
                      <CloudOff className="w-3 h-3" /> ออฟไลน์  
                    </span>  
                  ) : (  
                    <span className="flex items-center gap-1 text-[10px] bg-amber-500/20 text-amber-100 px-1.5 py-0.5 rounded-full">  
                      <Loader2 className="w-3 h-3 animate-spin" /> กำลังซิงค์  
                    </span>  
                  )}  
                </div>  
  
                {isEditingTripName ? (  
                  <input  
                    ref={tripNameInputRef}  
                    type="text"  
                    value={tripName}  
                    onChange={(e) => setTripName(e.target.value)}  
                    onBlur={handleSaveTripName}  
                    onKeyDown={(e) => e.key === 'Enter' && handleSaveTripName()}  
                    className="bg-white/20 text-white border-b-2 border-white/50 focus:border-white focus:outline-none px-1 py-0.5 font-bold text-xl rounded-t-sm w-full max-w-[200px]"  
                    placeholder="ตั้งชื่อทริป..."  
                  />  
                ) : (  
                  <h1   
                    onClick={() => setIsEditingTripName(true)}   
                    className="text-xl md:text-2xl font-bold flex items-center gap-2 cursor-pointer group hover:text-blue-50 transition-colors"  
                    title="คลิกเพื่อแก้ไขชื่อทริป"  
                  >  
                    {tripName}  
                    <Edit2 className="w-4 h-4 opacity-0 group-hover:opacity-100 transition-opacity" />  
                  </h1>  
                )}  
              </div>  
            </div>  
              
            <div className="text-right">  
              <p className="text-blue-100 text-xs">ยอดรวมทั้งทริป</p>  
              <p className="text-2xl font-black tracking-tight">฿{formatMoney(totalTripCost)}</p>  
            </div>  
          </div>  
  
          <div className="hidden md:flex gap-1 bg-white/10 p-1 rounded-xl backdrop-blur-md">  
            {[  
              { id: 'summary', icon: Wallet, label: 'สรุปยอด & โอนเงิน' },  
              { id: 'expenses', icon: Receipt, label: `รายการทั้งหมด (${expenses.length})` },  
              { id: 'members', icon: Users, label: `สมาชิก (${members.length})` }  
            ].map(tab => (  
              <button  
                key={tab.id}  
                onClick={() => setActiveTab(tab.id)}  
                className={`flex-1 py-2 px-4 rounded-lg text-sm font-medium transition-all flex items-center justify-center gap-2 ${  
                  activeTab === tab.id   
                    ? 'bg-white text-indigo-700 shadow-sm'   
                    : 'text-white/80 hover:bg-white/20 hover:text-white'  
                }`}  
              >  
                <tab.icon className="w-4 h-4" />  
                {tab.label}  
              </button>  
            ))}  
          </div>  
        </div>  
      </header>  
  
      <main className="max-w-3xl mx-auto px-4 py-6">  
          
        {/* --- TAB: SUMMARY --- */}  
        {activeTab === 'summary' && (  
          <div className="space-y-6 animate-in fade-in slide-in-from-bottom-2 duration-300">  
            <div className="bg-white rounded-3xl shadow-sm border border-slate-200 overflow-hidden">  
              <div className="bg-slate-50/50 p-5 border-b border-slate-100 flex items-center justify-between">  
                <div className="flex items-center gap-2">  
                  <Wallet className="w-5 h-5 text-indigo-500" />  
                  <h3 className="text-lg font-bold text-slate-800">สรุปการโอนเงิน</h3>  
                </div>  
                <span className="text-xs font-medium bg-indigo-100 text-indigo-700 px-2 py-1 rounded-full">  
                  {settlements.length} รายการโอน  
                </span>  
              </div>  
                
              <div className="p-5">  
                {members.length === 0 ? (  
                  <div className="text-center py-8 text-slate-400 flex flex-col items-center">  
                    <Users className="w-12 h-12 text-slate-200 mb-3" />  
                    <p className="font-medium text-slate-600">ยังไม่มีสมาชิก</p>  
                    <p className="text-sm mt-1">กรุณาไปที่แท็บ 'สมาชิก' เพื่อเริ่มใช้งาน</p>  
                    <button onClick={() => setActiveTab('members')} className="mt-4 text-indigo-600 font-medium hover:underline text-sm">ไปเพิ่มสมาชิก →</button>  
                  </div>  
                ) : settlements.length === 0 ? (  
                  <div className="text-center py-8 text-slate-400 flex flex-col items-center">  
                    <Check className="w-12 h-12 text-emerald-400 mb-3 bg-emerald-50 rounded-full p-2" />  
                    <p className="font-medium text-slate-600">ไม่มีรายการค้างชำระ</p>  
                    <p className="text-sm mt-1">ทุกคนเคลียร์กันลงตัวแล้ว 🎉</p>  
                  </div>  
                ) : (  
                  <div className="space-y-3">  
                    {settlements.map((s, idx) => (  
                      <div key={idx} className="flex items-center justify-between p-4 bg-white rounded-2xl border border-slate-200 shadow-sm hover:border-indigo-200 transition-colors">  
                        <div className="flex items-center gap-3 md:gap-4 flex-1 overflow-hidden">  
                          <div className="flex flex-col items-end min-w-[70px]">  
                            <span className="text-xs text-slate-400 mb-0.5">จาก</span>  
                            <span className="font-bold text-rose-600 truncate w-full text-right">{s.fromName}</span>  
                          </div>  
                          <div className="flex-shrink-0 bg-slate-100 p-2 rounded-full relative">  
                            <ArrowRight className="w-4 h-4 text-slate-500" />  
                          </div>  
                          <div className="flex flex-col items-start min-w-[70px]">  
                            <span className="text-xs text-slate-400 mb-0.5">ให้</span>  
                            <span className="font-bold text-emerald-600 truncate w-full">{s.toName}</span>  
                          </div>  
                        </div>  
                        <div className="text-right pl-4 border-l border-slate-100 ml-2">  
                          <span className="text-xs text-slate-400 block mb-0.5">ยอดโอน</span>  
                          <span className="font-black text-slate-800 text-lg">฿{formatMoney(s.amount)}</span>  
                        </div>  
                      </div>  
                    ))}  
                  </div>  
                )}  
              </div>  
            </div>  
  
            {members.length > 0 && (  
                <div className="bg-white rounded-3xl shadow-sm border border-slate-200 overflow-hidden">  
                <div className="bg-slate-50/50 p-5 border-b border-slate-100 flex items-center justify-between">  
                    <h3 className="text-lg font-bold text-slate-800">สถานะของทุกคน</h3>  
                    <p className="text-xs text-slate-500 flex items-center gap-1">  
                    <Info className="w-4 h-4" /> แตะที่ชื่อเพื่อดูรายละเอียด  
                    </p>  
                </div>  
                  
                <div className="divide-y divide-slate-100">  
                    {balances.map(b => (  
                    <div   
                        key={b.id}   
                        onClick={() => setSelectedMemberDetail(b.id)}  
                        className="p-5 hover:bg-slate-50 cursor-pointer transition-colors group flex flex-col md:flex-row md:items-center gap-4"  
                    >  
                        <div className="flex justify-between items-center md:w-1/3">  
                        <div className="flex items-center gap-3">  
                            <div className={`w-10 h-10 rounded-full flex items-center justify-center font-bold text-white shadow-inner ${  
                            b.balance > 0 ? 'bg-emerald-400' : b.balance < 0 ? 'bg-rose-400' : 'bg-slate-300'  
                            }`}>  
                            {b.name.charAt(0)}  
                            </div>  
                            <span className="font-bold text-slate-800 text-lg group-hover:text-indigo-600 transition-colors">{b.name}</span>  
                        </div>  
                        <ChevronRight className="w-5 h-5 text-slate-300 md:hidden" />  
                        </div>  
  
                        <div className="flex-1 space-y-2">  
                        <div className="flex justify-between text-sm">  
                            <span className="text-slate-500">จ่ายไป: <span className="font-semibold text-slate-700">฿{formatMoney(b.paid)}</span></span>  
                            <span className="text-slate-500">ส่วนแบ่ง: <span className="font-semibold text-slate-700">฿{formatMoney(b.share)}</span></span>  
                        </div>  
                          
                        <div className="h-2 w-full bg-slate-100 rounded-full overflow-hidden flex">  
                            {b.paid > 0 && (  
                            <div   
                                className={`h-full ${b.balance >= 0 ? 'bg-emerald-400' : 'bg-slate-300'}`}   
                                style={{ width: `${Math.min(100, (b.paid / Math.max(b.paid, b.share)) * 100)}%` }}   
                            />  
                            )}  
                            {b.share > b.paid && (  
                            <div   
                                className="h-full bg-rose-400 opacity-80"   
                                style={{ width: `${((b.share - b.paid) / b.share) * 100}%` }}   
                            />  
                            )}  
                        </div>  
  
                        <div className="flex justify-between items-center pt-1">  
                            <span className={`text-sm font-bold px-2 py-1 rounded-md ${  
                            b.balance > 0 ? 'bg-emerald-50 text-emerald-600' :   
                            b.balance < 0 ? 'bg-rose-50 text-rose-600' : 'bg-slate-100 text-slate-500'  
                            }`}>  
                            {b.balance > 0 ? `+ ฿${formatMoney(b.balance)} (ได้คืน)` :   
                            b.balance < 0 ? `- ฿${formatMoney(Math.abs(b.balance))} (ต้องจ่ายเพิ่ม)` :   
                            'พอดี'}  
                            </span>  
                            <ChevronRight className="w-5 h-5 text-slate-300 hidden md:block group-hover:translate-x-1 transition-transform" />  
                        </div>  
                        </div>  
                    </div>  
                    ))}  
                </div>  
                </div>  
            )}  
          </div>  
        )}  
  
        {/* --- TAB: EXPENSES --- */}  
        {activeTab === 'expenses' && (  
          <div className="space-y-4 animate-in fade-in slide-in-from-bottom-2 duration-300">  
            <div className="flex justify-between items-center mb-4 px-1">  
              <h2 className="text-lg font-bold flex items-center gap-2 text-slate-800">  
                <Receipt className="w-5 h-5 text-indigo-500" />  
                รายการทั้งหมด  
              </h2>  
              <button   
                onClick={handleOpenExpenseModal}  
                className="hidden md:flex bg-indigo-600 hover:bg-indigo-700 text-white px-4 py-2 rounded-xl text-sm font-medium items-center gap-1 transition-colors shadow-sm"  
              >  
                <Plus className="w-4 h-4" /> เพิ่มรายการใหม่  
              </button>  
            </div>  
  
            {expenses.length === 0 ? (  
               <div className="text-center py-16 text-slate-500 bg-white rounded-3xl border border-slate-200 shadow-sm flex flex-col items-center">  
                 <Receipt className="w-16 h-16 text-slate-200 mb-4" />  
                 <p className="text-lg font-medium">ยังไม่มีรายการค่าใช้จ่าย</p>  
                 <p className="text-sm text-slate-400 mt-1">กดปุ่ม + เพื่อเพิ่มรายการแรกของคุณ</p>  
               </div>  
            ) : (  
              <div className="grid gap-3 md:grid-cols-2">  
                {expenses.map((exp, index) => {  
                  const payer = members.find(m => m.id === exp.payerId);  
                  const splitAmount = exp.amount / (exp.splitBetween.length || 1);  
                    
                  return (  
                    <div key={exp.id} className="bg-white p-5 rounded-2xl shadow-sm border border-slate-200 relative group overflow-hidden">  
                      <div className="absolute -top-4 -right-4 text-6xl font-black text-slate-50/50 pointer-events-none">  
                        {index + 1}  
                      </div>  
  
                      <div className="relative z-10 flex flex-col h-full">  
                        <div className="flex justify-between items-start mb-3">  
                          <div>  
                            <h4 className="font-bold text-slate-800 text-lg mb-1 leading-tight">{exp.title}</h4>  
                            <div className="flex flex-wrap gap-2 text-xs">  
                              <span className="bg-slate-100 text-slate-600 px-2 py-1 rounded-md flex items-center gap-1 font-medium">  
                                จ่ายโดย: <span className="text-indigo-600">{payer?.name || 'ไม่ทราบ'}</span>  
                              </span>  
                            </div>  
                          </div>  
                          <div className="text-right">  
                            <div className="font-black text-indigo-600 text-xl tracking-tight">฿{formatMoney(exp.amount)}</div>  
                          </div>  
                        </div>  
  
                        <div className="mt-auto pt-4 border-t border-slate-100 flex justify-between items-end">  
                          <div className="text-sm">  
                            <p className="text-slate-500 mb-1">  
                              หาร <span className="font-bold text-slate-700">{exp.splitBetween.length}</span> คน   
                              <span className="text-slate-400 ml-1 text-xs">  
                                ({exp.splitBetween.map(id => members.find(m => m.id === id)?.name).filter(Boolean).join(', ')})  
                              </span>  
                            </p>  
                            <p className="font-medium text-rose-500 bg-rose-50 px-2 py-0.5 rounded inline-block text-xs">  
                              ตกคนละ ฿{formatMoney(splitAmount)}  
                            </p>  
                          </div>  
                          <button   
                            onClick={() => handleDeleteExpense(exp.id)}   
                            className="w-8 h-8 rounded-full bg-slate-50 text-slate-400 hover:text-rose-500 hover:bg-rose-50 flex items-center justify-center transition-colors"  
                            title="ลบรายการ"  
                          >  
                            <Trash2 className="w-4 h-4" />  
                          </button>  
                        </div>  
                      </div>  
                    </div>  
                  );  
                })}  
              </div>  
            )}  
            <div className="h-16 md:hidden"></div>  
          </div>  
        )}  
  
        {/* --- TAB: MEMBERS --- */}  
        {activeTab === 'members' && (  
          <div className="space-y-6 animate-in fade-in slide-in-from-bottom-2 duration-300">  
            <div className="bg-white rounded-3xl shadow-sm border border-slate-200 overflow-hidden">  
              <div className="bg-slate-50/50 p-5 border-b border-slate-100">  
                <h2 className="text-lg font-bold flex items-center gap-2 text-slate-800">  
                  <Users className="w-5 h-5 text-indigo-500" />  
                  จัดการสมาชิก ({members.length})  
                </h2>  
                <p className="text-xs text-slate-500 mt-1">พิมพ์ชื่อแล้วกดเพิ่มได้เลย หรือแตะที่ชื่อเพื่อนเพื่อดูรายงานส่วนตัว</p>  
              </div>  
                
              <div className="p-5">  
                <form onSubmit={handleAddMember} className="flex gap-2 mb-6">  
                  <input   
                    type="text"   
                    placeholder="พิมพ์ชื่อเพื่อน..."   
                    value={newMemberName}  
                    onChange={(e) => setNewMemberName(e.target.value)}  
                    className="flex-1 bg-white border border-slate-300 rounded-xl px-4 py-3 text-sm focus:outline-none focus:border-indigo-500 focus:ring-1 focus:ring-indigo-500 transition-shadow shadow-sm"  
                  />  
                  <button type="submit" disabled={!newMemberName.trim()} className="bg-indigo-600 disabled:bg-slate-300 text-white px-6 py-3 rounded-xl hover:bg-indigo-700 transition-colors font-medium shadow-sm flex items-center gap-2">  
                    <Plus className="w-4 h-4 hidden sm:block" /> เพิ่ม  
                  </button>  
                </form>  
  
                {members.length === 0 ? (  
                    <div className="text-center py-8 text-slate-400 bg-slate-50 rounded-2xl border border-dashed border-slate-200">  
                        <p>ยังไม่มีสมาชิก</p>  
                        <p className="text-sm mt-1">พิมพ์ชื่อด้านบนแล้วกดเพิ่มเพื่อเริ่มต้น</p>  
                    </div>  
                ) : (  
                    <div className="grid sm:grid-cols-2 gap-3">  
                    {members.map(m => (  
                        <div key={m.id} className="flex justify-between items-center p-4 bg-slate-50 hover:bg-indigo-50/50 rounded-2xl border border-slate-200 transition-colors group">  
                        <div   
                            className="flex-1 cursor-pointer flex items-center gap-3"  
                            onClick={() => setSelectedMemberDetail(m.id)}  
                        >  
                            <div className="w-8 h-8 rounded-full bg-indigo-100 text-indigo-600 flex items-center justify-center font-bold text-sm">  
                            {m.name.charAt(0)}  
                            </div>  
                            <span className="font-bold text-slate-700 group-hover:text-indigo-700 transition-colors">{m.name}</span>  
                        </div>  
                          
                        <button   
                            onClick={() => handleDeleteMember(m.id)}  
                            className="text-slate-300 hover:text-rose-500 bg-white p-2 rounded-full shadow-sm hover:shadow transition-all"  
                            title="ลบสมาชิก"  
                        >  
                            <Trash2 className="w-4 h-4" />  
                        </button>  
                        </div>  
                    ))}  
                    </div>  
                )}  
              </div>  
            </div>  
          </div>  
        )}  
  
      </main>  
  
      {/* Floating Action Button */}  
      {activeTab === 'expenses' && (  
        <button   
          onClick={handleOpenExpenseModal}  
          className="md:hidden fixed bottom-24 right-4 w-14 h-14 bg-indigo-600 text-white rounded-full shadow-lg shadow-indigo-200 flex items-center justify-center hover:bg-indigo-700 active:scale-95 transition-all z-40"  
        >  
          <Plus className="w-6 h-6" />  
        </button>  
      )}  
  
      {/* Bottom Navigation */}  
      <nav className="md:hidden fixed bottom-0 w-full bg-white border-t border-slate-200 pb-safe z-40 shadow-[0_-4px_20px_rgba(0,0,0,0.05)]">  
        <div className="flex justify-around items-center h-16 px-2">  
          {[  
            { id: 'summary', icon: Wallet, label: 'สรุปยอด' },  
            { id: 'expenses', icon: Receipt, label: 'รายการ' },  
            { id: 'members', icon: Users, label: 'สมาชิก' }  
          ].map(tab => (  
            <button   
              key={tab.id}  
              onClick={() => setActiveTab(tab.id)}   
              className={`flex flex-col items-center justify-center w-full h-full space-y-1 relative ${activeTab === tab.id ? 'text-indigo-600' : 'text-slate-400'}`}  
            >  
              {activeTab === tab.id && <div className="absolute top-0 w-1/2 h-0.5 bg-indigo-600 rounded-b-full"></div>}  
              <tab.icon className={`w-5 h-5 ${activeTab === tab.id ? 'fill-indigo-50 text-indigo-600' : ''}`} />  
              <span className="text-[10px] font-bold">{tab.label}</span>  
            </button>  
          ))}  
        </div>  
      </nav>  
  
      {/* Add Expense Modal */}  
      {isExpenseModalOpen && (  
        <div className="fixed inset-0 bg-slate-900/60 backdrop-blur-sm flex items-end md:items-center justify-center z-50 animate-in fade-in duration-200">  
          <div className="bg-white w-full max-w-md md:rounded-3xl rounded-t-3xl shadow-2xl overflow-hidden animate-in slide-in-from-bottom-10 md:zoom-in-95 duration-300 flex flex-col max-h-[90vh]">  
            <div className="bg-indigo-600 text-white p-5 flex justify-between items-center sticky top-0 z-10 shadow-sm">  
              <h3 className="text-xl font-bold flex items-center gap-2">  
                <Receipt className="w-5 h-5" /> เพิ่มรายการใหม่  
              </h3>  
              <button onClick={() => setIsExpenseModalOpen(false)} className="text-white/80 hover:text-white bg-white/10 p-1.5 rounded-full transition-colors">  
                <X className="w-5 h-5" />  
              </button>  
            </div>  
              
            <form onSubmit={handleAddExpense} className="p-5 space-y-5 overflow-y-auto">  
              <div>  
                <label className="block text-sm font-bold text-slate-700 mb-1.5">ชื่อรายการ</label>  
                <input   
                  type="text" required autoFocus  
                  value={newExpense.title}  
                  onChange={(e) => setNewExpense({...newExpense, title: e.target.value})}  
                  placeholder="เช่น ค่าอาหารเย็น, ค่าแท็กซี่"  
                  className="w-full bg-white border border-slate-300 rounded-xl px-4 py-3 text-slate-800 focus:outline-none focus:border-indigo-500 focus:ring-1 focus:ring-indigo-500 shadow-sm"  
                />  
              </div>  
  
              <div>  
                <label className="block text-sm font-bold text-slate-700 mb-1.5">จำนวนเงิน (บาท)</label>  
                <div className="relative">  
                  <span className="absolute left-4 top-1/2 -translate-y-1/2 text-slate-400 font-bold">฿</span>  
                  <input   
                    type="number" required min="1" step="0.01"  
                    value={newExpense.amount}  
                    onChange={(e) => setNewExpense({...newExpense, amount: e.target.value})}  
                    placeholder="0.00"  
                    className="w-full bg-white border border-slate-300 rounded-xl pl-9 pr-4 py-3 font-bold text-lg text-slate-800 focus:outline-none focus:border-indigo-500 focus:ring-1 focus:ring-indigo-500 shadow-sm"  
                  />  
                </div>  
              </div>  
  
              <div>  
                <label className="block text-sm font-bold text-slate-700 mb-1.5">ใครเป็นคนสำรองจ่าย?</label>  
                <div className="relative">  
                  <select   
                    value={newExpense.payerId}  
                    onChange={(e) => setNewExpense({...newExpense, payerId: e.target.value})}  
                    className="w-full bg-white border border-slate-300 rounded-xl px-4 py-3 text-slate-800 focus:outline-none focus:border-indigo-500 focus:ring-1 focus:ring-indigo-500 shadow-sm appearance-none font-medium"  
                  >  
                    {members.map(m => (  
                      <option key={m.id} value={m.id}>{m.name}</option>  
                    ))}  
                  </select>  
                  <div className="pointer-events-none absolute inset-y-0 right-0 flex items-center px-4 text-slate-500">  
                    <ChevronRight className="w-4 h-4 rotate-90" />  
                  </div>  
                </div>  
              </div>  
  
              <div className="pt-2">  
                <div className="flex justify-between items-end mb-3">  
                  <label className="block text-sm font-bold text-slate-700">หารกันกี่คน?</label>  
                  <span className="text-xs font-medium text-indigo-600 bg-indigo-50 px-2 py-1 rounded-md">  
                    เลือก {newExpense.splitBetween.length} คน  
                  </span>  
                </div>  
                  
                <div className="grid grid-cols-2 gap-3">  
                  {members.map(m => {  
                    const isSelected = newExpense.splitBetween.includes(m.id);  
                    return (  
                      <button  
                        type="button"  
                        key={m.id}  
                        onClick={() => toggleSplitMember(m.id)}  
                        className={`flex items-center gap-3 p-3.5 rounded-xl border-2 text-left transition-all ${  
                          isSelected   
                            ? 'bg-indigo-50 border-indigo-500 text-indigo-700 shadow-sm'   
                            : 'bg-white border-slate-200 text-slate-500 hover:border-indigo-200'  
                        }`}  
                      >  
                        <div className={`w-5 h-5 rounded flex-shrink-0 flex items-center justify-center border transition-colors ${isSelected ? 'bg-indigo-500 border-indigo-500 text-white' : 'border-slate-300 bg-slate-50'}`}>  
                          {isSelected && <Check className="w-3.5 h-3.5" />}  
                        </div>  
                        <span className="font-bold text-sm truncate">{m.name}</span>  
                      </button>  
                    );  
                  })}  
                </div>  
              </div>  
  
              <div className="pt-6 pb-2 sticky bottom-0 bg-white">  
                <button   
                  type="submit"   
                  disabled={!newExpense.title || !newExpense.amount || newExpense.splitBetween.length === 0}  
                  className="w-full bg-indigo-600 disabled:bg-slate-300 text-white font-bold rounded-xl px-4 py-4 hover:bg-indigo-700 transition-all shadow-md active:scale-[0.98] flex justify-center items-center gap-2"  
                >  
                  <Check className="w-5 h-5" />  
                  บันทึกรายการ  
                </button>  
              </div>  
            </form>  
          </div>  
        </div>  
      )}  
  
      {/* Detail Modal & Delete Dialog */}  
      <MemberDetailModal />  
        
      {confirmDialog.isOpen && (  
        <div className="fixed inset-0 bg-slate-900/60 backdrop-blur-sm flex items-center justify-center z-[60] animate-in fade-in duration-200 px-4">  
          <div className="bg-white rounded-3xl p-6 w-full max-w-sm shadow-2xl text-center animate-in zoom-in-95 duration-200">  
            <div className="w-16 h-16 bg-rose-100 text-rose-600 rounded-full flex items-center justify-center mx-auto mb-4">  
              <Trash2 className="w-8 h-8" />  
            </div>  
            <h3 className="text-xl font-bold text-slate-800 mb-2">ยืนยันการลบ</h3>  
            <p className="text-slate-500 mb-6 text-sm">{confirmDialog.message}</p>  
            <div className="flex gap-3">  
              <button   
                onClick={() => setConfirmDialog({ isOpen: false, type: '', id: '', message: '' })}  
                className="flex-1 bg-slate-100 hover:bg-slate-200 text-slate-700 font-bold py-3 rounded-xl transition-colors"  
              >  
                ยกเลิก  
              </button>  
              <button   
                onClick={executeDelete}  
                className="flex-1 bg-rose-600 hover:bg-rose-700 text-white font-bold py-3 rounded-xl transition-colors shadow-sm"  
              >  
                ลบเลย  
              </button>  
            </div>  
          </div>  
        </div>  
      )}  
    </div>  
  );  
}  
  
```  
